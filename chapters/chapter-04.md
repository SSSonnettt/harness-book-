# 第4章：工具系统 — Agent 如何"做事"

> **核心概念：** ToolRegistry，工具注册/发现/执行，混合确定性+概率性  
> **难度：** ★★☆☆☆  
> **预计阅读时间：** 45 分钟  
> **学习目标：** 读完本章后，你将能够：
> 1. 判断一个任务应该用工具还是 LLM
> 2. 设计和实现符合 ToolRegistry 规范的工具函数
> 3. 理解自定义 ToolRegistry 与原生 Function Calling 的选型差异  

---

## 4.1 为什么需要工具系统？

### LLM 的"软肋"

LLM 在推理和生成方面很强，但在以下方面很弱：

| 任务 | LLM 的弱点 | 代码的优势 |
|------|-----------|-----------|
| 模式匹配 | 会出错（概率性） | 100% 确定 |
| 数据验证 | 可能漏检 | Schema 强约束 |
| 精确计算 | 数学可能算错 | 精确无误 |
| 外部调用 | 无法直接操作 | fetch、fs、DB |

### 混合策略

最佳实践是**混合策略**：LLM 做决策 + 工具做执行：

```
用户输入 → LLM 决定"需要做什么" → 工具执行"怎么做"
                                     ↓
                               确定性结果返回给 LLM
                                     ↓
                               LLM 基于结果继续推理
```

就像人类一样：大脑（LLM）决定"我需要一杯咖啡"，手脚（工具）去泡咖啡。大脑不需要精确控制每根手指的肌肉运动——那是工具的职责。

---

## 4.2 什么是 ToolRegistry？

### 核心设计

resumate 的 ToolRegistry 是一个极简的名称→函数映射（`packages/agent-harness/src/tool-registry.ts`）：

```typescript
interface ToolFn {
  (args: Record<string, unknown>): Promise<Record<string, unknown>>;
}

class ToolRegistry {
  private tools = new Map<string, ToolFn>();

  register(name: string, fn: ToolFn): void {
    this.tools.set(name, fn);
  }

  has(name: string): boolean {
    return this.tools.has(name);
  }

  async execute(name: string, args: Record<string, unknown>):
    Promise<Record<string, unknown>> {
    const tool = this.tools.get(name);
    if (!tool) throw new Error(`Tool not found: ${name}`);
    return tool(args);
  }
}
```

总共 20 行有效代码。但它的设计蕴含了几个重要原则：

**原则 1：统一的函数签名。** 所有工具都是 `(args) => Promise<result>`。调用方不需要知道工具内部做什么，只需要知道它接收什么参数、返回什么结果。

**原则 2：名字即契约。** `registry.execute("validateResume", args)`——工具通过名字调用，名字是唯一的。这意味着不能有两个同名工具，这是合理的设计约束。

**原则 3：错误明确。** 如果调用的工具不存在，立即抛出错误而非静默失败。在 Agent 场景中，静默失败比明确错误更危险——你可能以为简历已经验证过了，但实际上验证工具根本没运行。

### resumate 的内置工具

resumate 有两个内置工具：

**1. classifyIntent — 意图分类**

```typescript
registry.register("classifyIntent", async (args) => {
  const input = (args.input as string) || "";

  // 确定性规则：正则匹配，不用 LLM
  const hasJD = /(职位描述|岗位职责|任职要求|job description|jd)/i.test(input);
  const hasPersonalInfo = [
    /(我叫|我是|我的|我毕业于|我在|我做|我负责)/i,
    /(20\d{2}[\.\-年]\d{1,2}).*(20\d{2}|至今|现在)/,  // 时间段
  ].some(p => p.test(input));

  return {
    intent: hasJD ? "jd_optimize"
          : hasPersonalInfo ? "resume_enhance"
          : "blank_slate",
    hasJD,
    hasResume: hasPersonalInfo,
  };
});
```

这个工具的设计哲学：**意图分类是规则问题，不是推理问题**。用三个正则 + 清晰的优先级就能 100% 准确，效率比 LLM 高几百倍。

**2. validateResume — 简历验证**

```typescript
registry.register("validateResume", async (args) => {
  const resume = resumeSchema.parse(args.resume);  // Zod 验证
  const issues: string[] = [];

  if (resume.modules.length === 0) issues.push("简历没有任何模块");
  const header = resume.modules.find(m => m.type === "header");
  if (!header) issues.push("缺少头部模块（姓名、联系方式）");
  else if (!header.data.name) issues.push("姓名未填写");

  return { valid: issues.length === 0, issues };
});
```

这个工具的价值在于：**它是 LLM 生成后的最后一道防线**。LLM 生成的 JSON 可能在语法上合法但在语义上有问题（如缺少姓名）——Zod parse 只检查语法，真正的语义验证由这个工具完成。

---

## 4.3 动手实践：为 resumate 添加新工具

让我们为 resumate 添加一个 JD 解析工具，展示工具设计的核心思路：

```typescript
registry.register("parseJobDescription", async (args) => {
  const jd = (args.jdText as string) || "";

  // 提取技术栈关键词（正则匹配，100% 确定性）
  const skills = extractTechKeywords(jd);  // ["React", "TypeScript", ...]

  // 提取年限和学历要求
  const yearsRequired = extractYearsRequirement(jd);    // 3
  const educationRequired = extractEducationRequirement(jd);  // "本科"

  return { skills, yearsRequired, educationRequired };
});
```

注意这个工具的设计要点：**输入是原始 JD 文本，输出是结构化的关键信息**。它把"从非结构化文本中提取结构化数据"这件事用确定性代码完成了，不需要 LLM 参与。类似地，`matchSkills` 工具将简历技能与 JD 技能做集合运算，返回匹配率和缺失项（完整实现见 [resumate 源码](https://github.com/SSSonnettt/resumate)）。

### 在 Plan 中使用新工具

```typescript
const jdOptimizePlan: Plan = {
  id: "jd-optimize",
  steps: [
    { id: "parseJD", type: "tool", description: "解析职位描述",
      tool: "parseJobDescription",
      toolArgs: (runtime) => ({ jdText: runtime.context.jdText }),
    },
    { id: "collect", type: "chat", description: "收集用户职业信息",
      dependsOn: ["parseJD"],
      userPromptTemplate: (runtime) => {
        const jd = runtime.stepResults.parseJD as { skills: string[] };
        return `用户想找的工作需要这些技能：${jd.skills.join("、")}。请帮用户...`;
      },
    },
    { id: "generate", type: "structured", dependsOn: ["collect"] },
    { id: "validate", type: "tool", dependsOn: ["generate"],
      tool: "validateResume" },
    { id: "present", type: "compose",
      dependsOn: ["validate"] },
  ],
};
```

关键观察：`parseJD` 的结果通过 `RuntimeValue` 动态注入 `collect` 的 prompt 中——这正是 Ch2 讲的"工具结果 → stepResults → 动态 prompt"数据链路。

---

## 4.4 代码解析：工具系统的三个决策

### 决策 1：工具 vs LLM 的边界在哪？

判断一个任务应该用工具还是 LLM 的简单规则：

| 任务特征 | 用工具 | 用 LLM |
|---------|--------|--------|
| 答案唯一确定 | ✅ | ❌ |
| 需要模式匹配/正则 | ✅ | ❌ |
| 涉及大量上下文推理 | ❌ | ✅ |
| 需要创造性输出 | ❌ | ✅ |
| 调用外部 API/数据库 | ✅ | ❌ |

一个实用测试：**如果你能写出测试用例来精确判断输出是否正确，就用工具。**

### 决策 2：工具结果如何传递给 LLM？

工具执行结果（`stepResults[step.id]`）存在 `StepRuntime` 中。后续步骤的 `userPromptTemplate` 是动态函数，可以从 `runtime.stepResults` 中读取工具结果并嵌入 prompt。

这形成了一条清晰的"工具 → LLM"数据链路：

```
Tool 输出确定性数据 → stepResults 结构化存储
    → RuntimeValue 动态读取 → 注入 LLM Prompt
    → LLM 基于精确数据推理
```

### 决策 3：工具应该多"大"？

resumate 的工具都很小（10-30 行），每个只做一件事。就像 Unix 哲学：**一个工具做一件事，做好它**。

大工具（如"生成完整简历"）的问题是：
- 难以测试
- 难以组合
- 出错时难以定位

小工具的好处：
- 可以独立测试
- 可以自由组合到不同 Plan 中
- 出错时明确知道是哪个工具

### 决策 4：自定义 ToolRegistry vs 原生 Function Calling？

主流 LLM 平台都提供了原生的工具调用功能：Anthropic 的 `tool_use`、OpenAI 的 `function calling`。它们让模型自己决定"何时调用哪个工具"，并以结构化格式返回工具调用参数。

| 维度 | 自定义 ToolRegistry | 原生 Function Calling |
|------|-------------------|---------------------|
| 工具调用决策 | Plan 中预定义（确定性） | 模型自主决定（概率性） |
| 通用性 | 任何 LLM | 仅支持 function calling 的模型 |
| 可控性 | 完全可控，执行顺序可预测 | 模型可能调用错误的工具或不调用 |
| 适用场景 | 步骤确定的 pipeline | 开放式对话、工具选择不确定时 |
| 外部工具标准 | 可通过 MCP（Model Context Protocol）扩展为标准化工具服务 | 平台绑定，工具定义随模型 API 变化 |

> 📌 **关于 MCP（Model Context Protocol）**：MCP 是 Anthropic 于 2024 年底提出概念、2025 年正式发布规范的开放协议，用于标准化 Agent 与外部工具/数据源的连接。如果你的工具需要被多个 Agent 共享、或需要跨语言/跨平台调用，MCP Server 是比自定义 ToolRegistry 更好的选择。本书的 ToolRegistry 是教学级的最小实现——理解它之后，迁移到 MCP 只是将 `Map<string, ToolFn>` 替换为 MCP Client 的问题。核心设计原则（统一签名、错误明确、小工具）在 MCP 场景下同样适用。

resumate 选择自定义 ToolRegistry 的原因是**可预测性**——简历生成的步骤是固定的（classify → collect → generate → validate → present），不需要模型自主决定"要不要验证"。在步骤确定的 pipeline 中，预定义的工具调用比模型自主决策更可靠、更高效。

如果你的 Agent 场景是开放式的（如"帮我规划一次旅行"，不确定需要调哪些工具），原生 Function Calling 是更好的选择。

### 设计好的工具接口：通用原则

无论是自定义 ToolRegistry 还是原生 Function Calling，工具设计的质量直接影响 Agent 的可靠性。以下是几条通用原则：

**1. 输入输出规范化。** 工具的参数和返回值应该有明确的类型约束。`parseJobDescription` 接收 `string`，返回 `{ skills: string[], yearsRequired: number | null }`——调用方不需要猜测数据格式。

**2. 错误处理要显式。** 工具失败时应该返回结构化的错误信息（`{ error: "JD 文本为空", code: "EMPTY_INPUT" }`），而不是抛出未捕获的异常。AgentRunner 可以据此决定是重试还是中止。

**3. 工具组合策略。** 多个小工具优于一个大工具。`parseJobDescription`（提取关键词）和 `matchSkills`（技能匹配）分开设计，可以独立测试、自由组合到不同 Plan 中。

**4. 工具粒度选择。** 一个实用的判断标准：**如果一个工具需要超过 50 行代码，考虑拆分**。大工具（如"端到端生成简历"）难以测试和调试；过小的工具（如"格式化一个日期"）则增加了 Plan 的复杂度。

---

## 4.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "所有步骤都应该让 LLM 做" | 反过来才对：能用代码确定的事情，绝不用 LLM |
| "工具越多越好" | 不是。每个工具增加维护成本。只在 LLM 做不好或效率低的地方加工具 |
| "工具应该返回自然语言给 LLM 读" | 不。工具应返回结构化数据。LLM 对结构化数据的理解优于自然语言描述 |

---

## 本章小结

- **工具系统的本质**：用确定性代码补 LLM 概率性的短板
- **ToolRegistry**：名称→函数的极简映射，20 行代码
- **混合策略**：LLM 决策 + 工具执行 = Agent 的完整能力

下一章：**结构化输出**——如何让 LLM 生成类型安全的 JSON。

### 自检问题

1. 什么任务应该用工具（确定性代码），什么任务应该用 LLM？给出你的判断标准。
2. ToolRegistry 的 `(args) => Promise<result>` 统一签名设计有什么好处？如果不同工具有不同签名会怎样？
3. 自定义 ToolRegistry 和原生 Function Calling 的核心区别是什么？在什么场景下你会选择哪一个？

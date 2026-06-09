# 第4章：工具系统 — Agent 如何"做事"

> **核心概念：** ToolRegistry，工具注册/发现/执行，混合确定性+概率性  
> **难度：** ★★☆☆☆  
> **预计阅读时间：** 45 分钟  

## 开篇故事

resumate 的 AgentRunner 执行 Plan 时，有两个步骤是 `type: "tool"` 的：`classify` 和 `validate`。

早期的实现中，这两步也是调 LLM：

```typescript
// 用 LLM 做意图分类
const classifyPrompt = `判断以下用户输入的意图：${userInput}
返回 JSON: { intent: "new_resume" | "jd_optimize" | "enhance" }`;
const result = await llm.chat(classifyPrompt);
```

结果呢？有时对，有时错。用户说"帮我看看这个 JD 怎么改简历"，LLM 偶尔返回 `{ intent: "new_resume" }`——把"改简历"误判为"新建简历"。

更糟糕的是：每次调 LLM 分类都花 1-2 秒 + 几百 Token 成本。而意图分类本质上是个**确定性问题**——检测输入中是否包含"JD/职位描述/岗位职责"关键词。

所以我把分类逻辑从 LLM 改成了正则匹配：

```typescript
function classifyIntent(input: string) {
  const hasJD = /(职位描述|岗位职责|JD|job description)/i.test(input);
  if (hasJD) return "jd_optimize";
  // ...
}
```

分类正确率从"有时错"变成了"100% 正确"（对本场景而言）。速度从 1-2 秒变成了瞬时。

这揭示了一个重要原则：**能用代码确定的事情，不应该交给 LLM**。

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

## 4.3 动手实践：为 resumate 添加新工具

让我们为 resumate 添加两个新工具。

### 工具 1：parseJobDescription — JD 关键词提取

```typescript
registry.register("parseJobDescription", async (args) => {
  const jd = (args.jdText as string) || "";

  // 提取技术栈关键词
  const techPatterns = [
    /React|Vue|Angular|Next\.js|TypeScript|JavaScript|Python|Go|Rust|Java/gi,
    /AWS|Azure|GCP|Docker|Kubernetes|CI\/CD/gi,
    /MySQL|PostgreSQL|MongoDB|Redis|GraphQL/gi,
  ];

  const skills = new Set<string>();
  for (const pattern of techPatterns) {
    const matches = jd.match(pattern);
    if (matches) matches.forEach(m => skills.add(m));
  }

  // 提取年限要求
  const yearMatch = jd.match(/(\d+)[\s\-]*年(以上)?(工作经验|相关经验)/);
  const yearsRequired = yearMatch ? parseInt(yearMatch[1]) : null;

  // 提取学历要求
  const educationMatch = jd.match(/(本科|硕士|博士|大专)(及以上)?(学历)?/);
  const educationRequired = educationMatch ? educationMatch[1] : null;

  return {
    skills: Array.from(skills),
    yearsRequired,
    educationRequired,
    keywordDensity: skills.size / (jd.length / 100), // 每百字关键词密度
  };
});
```

### 工具 2：matchSkills — 技能匹配打分

```typescript
registry.register("matchSkills", async (args) => {
  const resumeSkills = (args.resumeSkills as string[]) || [];
  const jdSkills = (args.jdSkills as string[]) || [];

  const matched: string[] = [];
  const missing: string[] = [];

  for (const skill of jdSkills) {
    const found = resumeSkills.some(
      rs => rs.toLowerCase() === skill.toLowerCase()
    );
    if (found) matched.push(skill);
    else missing.push(skill);
  }

  const matchRate = jdSkills.length > 0
    ? matched.length / jdSkills.length
    : 0;

  return {
    matched,
    missing,
    matchRate,
    grade: matchRate >= 0.8 ? "A" : matchRate >= 0.5 ? "B" : "C",
    suggestion: missing.length > 0
      ? `建议补充以下技能：${missing.join("、")}`
      : "技能匹配度良好",
  };
});
```

### 在 Plan 中使用新工具

```typescript
const jdOptimizePlan: Plan = {
  id: "jd-optimize",
  steps: [
    // Step 1: 用工具解析 JD
    { id: "parseJD", type: "tool", description: "解析职位描述",
      tool: "parseJobDescription",
      toolArgs: (runtime) => ({ jdText: runtime.context.jdText }),
    },
    // Step 2: 收集用户信息
    { id: "collect", type: "chat", description: "收集用户职业信息",
      dependsOn: ["parseJD"],
      userPromptTemplate: (runtime) => {
        const jd = runtime.stepResults.parseJD as { skills: string[] };
        return `用户想找的工作需要这些技能：${jd.skills.join("、")}。请帮用户...`;
      },
    },
    // Step 3: 生成简历 → Step 4: 验证 → Step 5: 技能匹配打分
    { id: "generate", type: "structured", dependsOn: ["collect"], /* ... */ },
    { id: "validate", type: "tool", dependsOn: ["generate"],
      tool: "validateResume" },
    { id: "matchSkills", type: "tool", dependsOn: ["generate", "parseJD"],
      tool: "matchSkills",
      toolArgs: (runtime) => {
        const resume = runtime.stepResults.generate as Resume;
        const jd = runtime.stepResults.parseJD as { skills: string[] };
        const resumeSkills = resume.modules
          .find(m => m.type === "skills")?.data?.categories
          ?.flatMap((c: { items: string[] }) => c.items) ?? [];
        return { resumeSkills, jdSkills: jd.skills };
      },
    },
    { id: "present", type: "compose", dependsOn: ["validate", "matchSkills"] },
  ],
};
```

执行结果示例：

```
✅ parseJD → { skills: ["React","TypeScript","Docker"], yearsRequired: 3, ... }
✅ collect → { text: "用户有5年React经验，熟悉TypeScript..." }
✅ generate → { modules: [{ type: "header", ... }, { type: "skills", ... }] }
✅ validate → { valid: true, issues: [] }
✅ matchSkills → { matched: ["React","TypeScript"], missing: ["Docker"],
                    matchRate: 0.67, grade: "B" }
✅ present → 最终简历（附带技能匹配报告）
```

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

## 4.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "所有步骤都应该让 LLM 做" | 反过来才对：能用代码确定的事情，绝不用 LLM |
| "工具越多越好" | 不是。每个工具增加维护成本。只在 LLM 做不好或效率低的地方加工具 |
| "工具应该返回自然语言给 LLM 读" | 不。工具应返回结构化数据。LLM 对结构化数据的理解优于自然语言描述 |

### 练习

1. **（★☆☆）** 为 resumate 实现一个 `formatDate` 工具：输入日期字符串，返回标准化的日期格式。

2. **（★★☆）** 为 `validateResume` 工具添加更多检查规则：联系方式的格式验证、工作经历的日期连续性检查。

3. **（★★★）** 实现一个"工具调用日志"：每个工具被调用时，记录工具名、参数、结果和执行时间。用于调试和性能分析。

## 本章小结

- **工具系统的本质**：用确定性代码补 LLM 概率性的短板
- **ToolRegistry**：名称→函数的极简映射，20 行代码
- **混合策略**：LLM 决策 + 工具执行 = Agent 的完整能力

下一章：**结构化输出**——如何让 LLM 生成类型安全的 JSON。

# 第2章：执行循环 — Agent 如何"观察-思考-行动"

> **核心概念：** 执行循环 (Execution Loop)，Plan/Step 模型  
> **难度：** ★★☆☆☆  
> **预计阅读时间：** 50 分钟  
> **本章代码量：** 从 ~30 行扩展到 ~110 行  
> **学习目标：** 读完本章后，你将能够：
> 1. 用 Plan/Step 模型将复杂任务分解为有序步骤
> 2. 区分四种 Step 类型及其适用场景
> 3. 使用 AsyncGenerator 实现事件驱动的执行循环
> 4. 理解 RuntimeValue 动态参数的工作原理  

---

## 2.1 为什么需要执行循环？

### 单次调用的局限

回顾第 1 章的最简 Agent：

```typescript
class SimpleAgent {
  async run(userInput: string): Promise<string> {
    const messages = [
      { role: "system", content: "你是一位专业的中文求职顾问。" },
      { role: "user", content: userInput }
    ];
    return await this.model.chat(messages);
  }
}
```

当你只有一个"调用→返回"时，你面临三个根本问题：

**问题 1：无法分解复杂任务。** "帮我写一份针对这个 JD 的 ATS 友好简历"——这至少需要：解析 JD → 提取关键词 → 生成简历 → 检查 ATS 兼容性。如果把所有指令塞进一个 prompt，LLM 容易遗漏步骤或中途迷失。

**问题 2：没有中间验证。** 生成完就结束了。如果 LLM 生成的 JSON 缺了 `name` 字段，你只能等用户抱怨后才能修复。

**问题 3：无法复用中间结果。** 第一步"解析 JD"的结果（如关键词列表）可能需要在后续步骤中多次使用。单次调用时，这些中间结果只能存在 prompt 文本中，无法结构化地管理和引用。

### 解决方案：多步骤执行循环

解决这三个问题的方法就是**执行循环（Execution Loop）**：

```
while (not done) {
  observe();   // 观察当前状态
  think();     // 决定下一步做什么
  act();       // 执行下一步
}
```

一个最简单的多步骤 Agent 可以这样写：

```typescript
class MultiStepAgent {
  async run(userInput: string): Promise<Resume> {
    // Step 1: 分析意图
    const intent = await this.classify(userInput);  // "new_resume" | "jd_optimize" | "enhance"

    // Step 2: 收集信息
    const info = await this.collectInfo(userInput, intent);

    // Step 3: 生成简历
    const resume = await this.generate(info);

    // Step 4: 验证
    const validation = this.validate(resume);
    if (!validation.valid) {
      // 出错？修复它
      return await this.fixAndRegenerate(resume, validation.issues);
    }

    // Step 5: 完成
    return resume;
  }
}
```

这就是 resumate 中 AgentRunner 的设计原型。

---

## 2.2 什么是执行循环？

### Plan/Step 模型

在 resumate 的 agent-harness 中，执行循环被抽象为 **Plan/Step 模型**：

- **Plan（计划）**：一个有序的步骤序列，描述"要完成什么"
- **Step（步骤）**：Plan 中的单个执行单元，有明确的类型、依赖和输入输出

```typescript
interface Plan {
  id: string;           // 计划标识，如 "resume-generation"
  steps: PlanStep[];    // 有序步骤列表
}

interface PlanStep {
  id: string;                              // 步骤标识，如 "classify"
  type: "tool" | "chat" | "structured" | "compose";  // 步骤类型
  description?: string;                    // 人类可读的描述
  dependsOn?: string[];                    // 依赖的前置步骤 ID
  // 以下按步骤类型选择填写
  tool?: string;                           // tool 类型：工具名
  toolArgs?: RuntimeValue<Record<string, unknown>>;  // 工具参数
  systemPrompt?: RuntimeValue<string>;     // chat/structured：系统提示词
  userPromptTemplate?: RuntimeValue<string>; // chat/structured：用户提示词
  schema?: z.ZodType<unknown>;            // structured 类型：输出 Schema
  compose?: RuntimeValue<Resume>;         // compose 类型：组装逻辑
}
```

这里有四个关键设计决策，值得逐一展开。

#### 决策 1：四种 Step 类型

为什么需要四种类型，而不是一种通用的"调用 LLM"？

| 类型 | 用途 | 确定性 | 示例 |
|------|------|--------|------|
| `tool` | 执行确定性函数 | 100% 确定 | `classifyIntent`（正则匹配）、`validateResume`（Schema 验证） |
| `chat` | 流式 LLM 对话 | 概率性 | 与用户多轮对话，收集职业信息 |
| `structured` | LLM 输出结构化 JSON | 概率性但格式约束 | 生成 `Resume` JSON（由 Zod Schema 约束） |
| `compose` | 组装最终结果 | 100% 确定 | 将各步骤的输出合并为最终简历 |

核心理念：**混合确定性（tool/compose）和概率性（chat/structured）**。能用代码确定的事情（正则匹配、Schema 验证）就不该交给 LLM。

#### 决策 2：dependsOn — 声明式依赖

`dependsOn: ["classify", "collect"]` 意味着当前步骤必须在 `classify` 和 `collect` 都完成后才能运行。

这不是运行时动态决定的，而是**在定义 Plan 时就声明好的**。好处：
- 执行顺序清晰可预测
- AgentRunner 可以在执行前做静态检查
- 调试时可以精确知道"卡在哪一步"

#### 决策 3：RuntimeValue — 参数动态计算

```typescript
type RuntimeValue<T> = T | ((runtime: StepRuntime) => T);
```

一个参数可以是：
- 静态值：`"你是一位专业的求职顾问"`
- 动态计算函数：`(runtime) => \`用户的意图是 ${runtime.stepResults.classify.intent}\``

为什么需要动态计算？因为前一步的输出是后一步的输入。`classify` 的结果（用户意图）决定了 `collect` 用什么 prompt。如果所有参数都是静态的，你就没法根据上下文调整。

#### 决策 4：AsyncGenerator — 事件驱动

`AgentRunner.execute()` 返回的是 `AsyncGenerator<HarnessEvent>`，不是 `Promise<Resume>`。

这意味着每一步的执行状态都实时对外播报：

```typescript
for await (const event of runner.execute(plan)) {
  switch (event.type) {
    case "plan:start":    /* 计划开始 */     break;
    case "step:start":    /* 步骤开始 */     break;
    case "step:chunk":    /* LLM 流式文本 */  showText(event.text); break;
    case "step:tool_call": /* 工具调用 */    showToolUse(event.tool); break;
    case "step:done":     /* 步骤完成 */     break;
    case "plan:done":     /* 计划完成 */     showResume(event.resume); break;
    case "plan:error":    /* 错误 */        showError(event.error); break;
  }
}
```

这个设计让 UI 层可以实时响应 Agent 状态，而不需要轮询或回调。

### 完整的事件生命周期

resumate 中完整的一次 Plan 执行产生的事件序列：

```
plan:start (planId: "resume-generation")
  │
  ├── step:start (stepId: "classify")
  │   └── step:done  (stepId: "classify", result: {intent, hasJD, hasResume})
  │
  ├── step:start (stepId: "collect")
  │   ├── step:chunk (stepId: "collect", text: "你好！为了更好地帮你...")
  │   ├── step:chunk (stepId: "collect", text: "请问你之前在哪家公司...")
  │   └── step:done  (stepId: "collect", result: {text: "...", reasoning: "..."})
  │
  ├── step:start (stepId: "generate")
  │   └── step:done  (stepId: "generate", result: Resume)
  │
  ├── step:start (stepId: "validate")
  │   ├── step:tool_call (stepId: "validate", tool: "validateResume", args: {...})
  │   └── step:done  (stepId: "validate", result: {valid: true, issues: []})
  │
  └── step:start (stepId: "present")
      └── step:done  (stepId: "present", result: Resume)
      └── plan:done  (resume: Resume)
```

如果把整个执行看作一条生产线，每个事件就是生产线上的一盏指示灯——告诉你当前物料在哪、加工到哪一步、有没有异常。

---

## 2.3 动手实践：为最简 Agent 添加执行循环

让我们基于第 1 章的 `SimpleAgent`，逐步构建出 resumate 风格的执行循环。

### Step 1：定义 Plan 和 Step

```typescript
// 从第1章的 SimpleAgent 演变而来

type StepType = "tool" | "chat" | "structured" | "compose";

interface PlanStep {
  id: string;
  type: StepType;
  description: string;
  dependsOn?: string[];
}

interface Plan {
  id: string;
  steps: PlanStep[];
}

// resumate 的简历生成 Plan
const resumeGenerationPlan: Plan = {
  id: "resume-generation",
  steps: [
    { id: "classify",   type: "tool",       description: "分析用户意图" },
    { id: "collect",    type: "chat",       description: "收集用户的职业信息",
      dependsOn: ["classify"] },
    { id: "generate",   type: "structured", description: "生成结构化简历 JSON",
      dependsOn: ["collect"] },
    { id: "validate",   type: "tool",       description: "验证简历数据完整性",
      dependsOn: ["generate"] },
    { id: "present",    type: "compose",    description: "输出最终简历",
      dependsOn: ["validate"] },
  ],
};
```

### Step 2：实现事件类型

```typescript
// HarnessEvent: Agent 运行时的所有事件（8 种）
type HarnessEvent =
  | { type: "plan:start";  planId: string }
  | { type: "step:start";  stepId: string; description: string }
  | { type: "step:chunk";  stepId: string; text: string }
  | { type: "step:tool_call"; stepId: string; tool: string; args: unknown }
  | { type: "step:done";   stepId: string; result: unknown }
  | { type: "plan:done";   resume: unknown }
  | { type: "plan:error";  stepId: string; error: string }
  | { type: "reasoning:chunk"; stepId: string; text: string };  // 思考过程（Ch6 详述）
```

### Step 3：实现 AgentRunner

```typescript
class AgentRunner {
  constructor(
    private provider: LLMProvider,
    private tools: Map<string, Function>
  ) {}

  async *execute(plan: Plan): AsyncGenerator<HarnessEvent> {
    yield { type: "plan:start", planId: plan.id };
    const stepResults: Record<string, unknown> = {};

    for (const step of plan.steps) {
      // ① 依赖检查：前置步骤是否都已完成？
      const missing = step.dependsOn?.find(dep => !(dep in stepResults));
      if (missing) {
        yield { type: "plan:error", stepId: step.id,
                error: `依赖步骤 "${missing}" 未完成` };
        return;
      }

      // ② 播报：步骤开始
      yield { type: "step:start", stepId: step.id,
              description: step.description };

      try {
        // ③ 按类型执行（核心逻辑，细节见下方讨论）
        let result = await this.executeStep(step, stepResults);

        // ④ 保存结果，播报完成
        stepResults[step.id] = result;
        yield { type: "step:done", stepId: step.id, result };
      } catch (err) {
        // ⑤ 错误处理：快速失败
        yield { type: "plan:error", stepId: step.id,
                error: err instanceof Error ? err.message : String(err) };
        return;
      }
    }

    // ⑥ 计划完成
    yield { type: "plan:done", resume: stepResults["present"] };
  }
}
```

注意这个骨架只有 ~40 行，但已经包含了执行循环的全部核心逻辑：**依赖检查 → 事件播报 → 按类型执行 → 错误处理**。`executeStep` 方法内部根据 `step.type` 分发到不同的处理分支（tool/chat/structured/compose），每种分支的实现细节在前面的章节中已经分别涉及。完整的 ~110 行实现可以在 [resumate 源码](https://github.com/SSSonnettt/resumate) 的 `packages/agent-harness/src/runner.ts` 中找到。

### Step 4：对比第 1 章的 SimpleAgent

| 维度 | SimpleAgent (Ch1) | AgentRunner (Ch2) |
|------|-------------------|-------------------|
| 步骤数 | 1 | N（由 Plan 定义） |
| 执行方式 | `async run()` 返回 Promise | `async *execute()` 返回 AsyncGenerator |
| 状态可见性 | 黑盒（只有输入输出） | 白盒（每个步骤都有事件播报） |
| 错误处理 | try-catch 包裹整体 | 每步独立错误，精确知道哪步失败 |
| 中间结果 | 不可访问 | `stepResults` 结构化存储，后续步骤可用 |
| 依赖管理 | 无 | `dependsOn` 声明式依赖 |

---

## 2.4 代码解析：AgentRunner 的三层循环

现在让我们深入 resumate 的真正源码，理解 AgentRunner 的设计精要。

### 外层循环：Step 遍历

```typescript
for (const step of plan.steps) {
  // ...
}
```

这是最外层的控制流：按 Plan 中的声明顺序逐个执行 Step。注意这里的 `for...of` 是有顺序保证的——Plan 的设计假设步骤是**有序的**，`dependsOn` 只是增强校验，不是调度机制。

### 中层循环：事件产出

```typescript
yield { type: "step:start", ... };
// ... 执行 ...
yield { type: "step:done", ... };
```

每个 Step 在执行前后都产出事件。这些事件通过 `AsyncGenerator` 向外传输，UI 层可以用 `for await...of` 消费：

```typescript
// 在 Next.js API Route 中
const encoder = new TextEncoder();
for await (const event of runner.execute(plan, context)) {
  controller.enqueue(
    encoder.encode(`data: ${JSON.stringify(event)}\n\n`)
  );
}
```

### 内层循环（LLM streaming）：逐 Token 产出

```typescript
// 回调收集文本，完成后统一产出事件
let fullText = "";
await this.provider.streamChat({ messages }, (chunk) => {
  if (chunk.type === "text") {
    fullText += chunk.content;
  }
});
yield { type: "step:chunk", stepId: step.id, text: fullText };
```

对于 `chat` 类型的步骤，`streamChat` 使用回调模式逐 token 接收 LLM 输出。注意：回调函数内不能使用 `yield`（回调不是 generator），所以 AgentRunner 在回调中收集文本，流式结束后再通过外层 generator 产出 `step:chunk` 事件。实际的 resumate 中，Provider 层（Ch3）使用回调，AgentRunner 层使用 AsyncGenerator——两层各选适合的模式。

### RuntimeValue：动态参数解析

```typescript
function resolveRuntimeValue<T>(
  value: T | ((runtime: StepRuntime) => T) | undefined,
  runtime: StepRuntime,
): T | undefined {
  return typeof value === "function"
    ? (value as (runtime: StepRuntime) => T)(runtime)
    : value;
}
```

这 5 行代码是 AgentRunner 中最重要的设计之一。它让 Step 的参数可以：
- 是静态值（适用于不变的 system prompt）
- 是动态函数（适用于依赖前序步骤结果的 prompt）

例如：

```typescript
// collect 步骤的 userPromptTemplate 根据 classify 的结果动态生成
{
  id: "collect",
  type: "chat",
  userPromptTemplate: (runtime) => {
    const intent = runtime.stepResults.classify?.intent;
    return intent === "jd_optimize"
      ? "用户想针对 JD 优化简历，请收集以下信息..."
      : "用户想创建新简历，请按以下步骤收集信息...";
  }
}
```

---

## 2.5 业界执行循环模式对比

resumate 的 Plan/Step 模型只是执行循环的一种实现。了解业界主流模式有助于你在新项目中做出更好的选型：

| 模式 | 核心思想 | 代表 | 适用场景 |
|------|---------|------|---------|
| **ReAct**（Reasoning + Acting） | 模型交替"推理"和"行动"，每步动态决定下一步 | LangChain AgentExecutor | 开放式任务，步骤不确定 |
| **Plan-and-Execute** | 先制定完整计划，再按序执行 | 本书的 AgentRunner | 步骤确定的 pipeline |
| **Tree-of-Thought** | 探索多条推理路径，选择最优 | 研究场景 | 需要探索的推理问题 |
| **Reflexion** | 执行 → 反思 → 修正，循环迭代 | Ch9 的自修正循环 | 质量敏感的任务 |

resumate 选择 Plan-and-Execute 的原因是**可预测性**——简历生成的步骤是固定的，不需要模型动态探索。但 Plan-and-Execute 不排斥 ReAct：在 `collect` 步骤（开放式对话）中，模型实际上在做 ReAct 式的动态推理。

关键洞察：**执行循环的模式不是二选一的，而是可以嵌套的**。外层用 Plan-and-Execute 保证整体流程可控，内层某些步骤用 ReAct 处理开放式子任务。这种"确定性骨架 + 概率性血肉"的混合模式，是 Harness 设计中最实用的策略之一。

---

## 2.6 复盘与延伸

### 本章要点回顾

1. **执行循环**是 Agent 区别于"单次 API 调用"的核心。Agent 通过分解任务、多步执行、中间验证来完成复杂工作。

2. **Plan/Step 模型**：Plan 定义"做什么"，Step 定义"每一步怎么做"。四种 Step 类型（tool/chat/structured/compose）覆盖了确定性操作和概率性生成。

3. **事件驱动**：`AsyncGenerator<HarnessEvent>` 让每一步的状态都可见、可追踪、可中断。

4. **声明式依赖**：`dependsOn` 让步骤之间的依赖关系显式化，方便静态检查和调试。

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "执行循环就是 while(true) 调 LLM" | 循环的边界由 Plan 定义，不是无限循环。Plan 完成即退出 |
| "步骤数越多越好" | 并非如此。每增加一个步骤，就增加了一轮 LLM 调用的延迟和 Token 成本。够用即可 |
| "dependsOn 可以处理所有依赖" | dependsOn 只表达"A 必须在 B 之前完成"，不表达"B 的输出类型必须符合某 Schema"（那是 `compose` 步骤的职责） |

### Harness vs 流行框架

读者可能会问：既然 LangChain、CrewAI、AutoGen 这些框架已经提供了 Agent 能力，为什么还需要理解 AgentRunner 这种"自己写的执行循环"？

| 维度 | LangChain/CrewAI | 本书的 AgentRunner |
|------|-----------------|-------------------|
| 定位 | 通用 Agent 框架 | 教学级 Harness 实现 |
| 代码量 | 数万行 | ~200 行 |
| 学习曲线 | 陡峭（需要理解 Chain/Agent/Tool 等抽象） | 平缓（只理解 Plan/Step/Event 三个概念） |
| 灵活性 | 受限于框架设计 | 完全可控 |
| 适用场景 | 生产级复杂 Agent 系统 | 理解 Agent 原理 + 中小型项目 |

本质区别：框架是"实现"（Implementation），Harness 是"概念体系"（Conceptual Framework）。

LangChain 的 `AgentExecutor`、CrewAI 的 `Crew`、AutoGen 的 `ConversableAgent`——它们都是 Plan/Step 模型在不同场景下的具体实现。理解 Harness 概念之后，学任何框架都只是换个 API 调而已。

本书选择从零构建 AgentRunner，不是要重复造轮子。亲手造过之后，那些框架的设计决策就不再神秘了。

### 进阶话题

- **条件分支**：当前 AgentRunner 是线性执行 + dependsOn 约束。resumate 的 `classify` 步骤通过结果影响后续步骤的 prompt，实现"软分支"。如果你需要"硬分支"（跳过某些步骤），可以在 Plan 中引入 `condition` 字段。
- **重试机制**：当前错误处理是"出错即中止"。你可以在 `catch` 块中加入重试逻辑，对 transient error（如 API 限流）重试，对 logical error（如 Schema 不匹配）中止。

---

## 本章小结

这一章，我们从单次 API 调用走到了多步骤执行循环：

- **为什么**：复杂任务需要分解、验证、编排
- **是什么**：Plan/Step 模型 + AsyncGenerator 事件系统 + RuntimeValue 动态参数
- **怎么做**：AgentRunner 的三层循环（Step 遍历 → 事件产出 → Token 流式）

下一章，我们深入 AgentRunner 依赖的核心抽象——LLMProvider，理解如何让 Agent 与不同模型对话而无需修改业务逻辑。

### 自检问题

1. Plan/Step 模型中的四种 Step 类型（tool/chat/structured/compose）各适用于什么场景？
2. RuntimeValue 的类型签名 `type RuntimeValue<T> = T | ((runtime: StepRuntime) => T)` 解决了什么问题？为什么不用纯静态参数？
3. 业界执行循环模式（ReAct / Plan-and-Execute / Reflexion）中，resumate 选择 Plan-and-Execute 的原因是什么？

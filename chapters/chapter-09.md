# 第9章：编排与 Hooks — 从单兵到团队

> **核心概念：** Orchestration，Dependency Graph，Hooks，质量门  
> **难度：** ★★★★☆  
> **预计阅读时间：** 50 分钟  

---

## 开篇故事

resumate 的第 2 章执行循环是线性的：`classify → collect → generate → validate → present`。

但真实需求不总是线性的。有一天用户反馈：**"生成完你让我看看质量怎么样啊"**。

我想了一下，需要加一个**自我修正循环**：

```
generate → validate → (score < 70?) → enhance → validate → present
```

这不是简单的 5 步序列，而是一个带条件分支和循环的 DAG（有向无环图）。

更进一步，有些质量检查不是"步骤"而是"横切关注点"——无论哪个步骤，在输出给用户之前都必须通过某些检查（如：是否包含敏感信息？Token 预算是否超限？）。

这些横切检查就是 **Hooks**。它们不参与主线执行，而是挂在关键节点上的"守护者"。

---

## 9.1 为什么需要编排与 Hooks？

线性执行只能处理最简单的情况。一旦需要：
- 条件分支（"如果 JD 优化模式，跳过信息收集"）
- 循环（"生成 → 验证 → 失败 → 增强 → 再验证"）
- 横切关注点（"每个步骤完成后记录日志"）

线性代码就会变成意大利面条。

编排解决"步骤如何组合"，Hooks 解决"横切关注点如何管理"。

---

## 9.2 编排：从线性到 DAG

### resumate 当前的编排模型

```typescript
// 线性 Plan（当前实现）
const linearPlan: Plan = {
  id: "resume-generation",
  steps: [
    { id: "classify",  type: "tool",       dependsOn: [] },
    { id: "collect",   type: "chat",       dependsOn: ["classify"] },
    { id: "generate",  type: "structured", dependsOn: ["collect"] },
    { id: "validate",  type: "tool",       dependsOn: ["generate"] },
    { id: "present",   type: "compose",    dependsOn: ["validate"] },
  ],
};
```

`dependsOn` 定义了 DAG 的边。但它只做了"前置检查"——A 完成了吗？没有做"条件分支"——"如果 A 的结果是 X，跳过 B"。

### 增强：条件步骤和自修正循环

把 Plan 扩展为支持条件：

```typescript
interface PlanStep {
  // ... 原有字段
  condition?: RuntimeValue<boolean>;   // 条件：true 才执行此步骤
  onFail?: string[];                   // 失败时跳转到哪些步骤
}

// 带自修正循环的 Plan
const selfRevisePlan: Plan = {
  id: "resume-generation-v2",
  steps: [
    { id: "classify",  type: "tool" },
    { id: "collect",   type: "chat",       dependsOn: ["classify"],
      condition: (r) => r.stepResults.classify?.intent !== "blank_slate" },
    { id: "generate",  type: "structured", dependsOn: ["collect"] },
    { id: "validate",  type: "tool",       dependsOn: ["generate"] },
    // 新增：如果 validate 评分 < 70，进入 enhance 然后重新 validate
    { id: "enhance",   type: "chat",       dependsOn: ["validate"],
      condition: (r) => {
        const v = r.stepResults.validate as { score: number };
        return v.score < 70;
      },
      onFail: ["present"],
      userPromptTemplate: (r) => {
        const issues = (r.stepResults.validate as { issues: string[] }).issues;
        return `请修复以下问题：${issues.join("\n")}`;
      },
    },
    // enhance 后会重新 validate
    { id: "revalidate", type: "tool", dependsOn: ["enhance"],
      tool: "validateResume" },
    { id: "present",   type: "compose",
      dependsOn: [],  // 由 onFail 或正常流程到达
    },
  ],
};
```

### Hooks：横切关注点的守护者

Hooks 是不参与主线执行的拦截器。在 resumate 中，最简单的 Hook 实现：

```typescript
type HookPoint = "beforeStep" | "afterStep" | "beforePlan" | "afterPlan";

interface Hook {
  point: HookPoint;
  priority: number;  // 数字越小越先执行
  handler: (context: HookContext) => Promise<HookResult>;
}

type HookResult =
  | { action: "continue" }
  | { action: "block"; reason: string }
  | { action: "modify"; changes: Partial<PlanStep> };

class HookManager {
  private hooks: Hook[] = [];

  register(hook: Hook): void {
    this.hooks.push(hook);
    this.hooks.sort((a, b) => a.priority - b.priority);
  }

  async executeHooks(
    point: HookPoint,
    context: HookContext
  ): Promise<HookResult> {
    for (const hook of this.hooks.filter(h => h.point === point)) {
      const result = await hook.handler(context);
      if (result.action === "block") return result; // 阻塞：停止执行
    }
    return { action: "continue" };
  }
}
```

### resumate 中的典型 Hooks

```typescript
const hookManager = new HookManager();

// Hook 1: Token 预算检查（每个 Step 执行前）
hookManager.register({
  point: "beforeStep",
  priority: 10,
  async handler(ctx) {
    const estimatedTokens = estimateTokens(ctx.step);
    if (estimatedTokens > ctx.budget.remaining) {
      return {
        action: "block",
        reason: `Token 预算不足：需要 ${estimatedTokens}，剩余 ${ctx.budget.remaining}`,
      };
    }
    return { action: "continue" };
  },
});

// Hook 2: 安全过滤（每个 Step 执行后）
hookManager.register({
  point: "afterStep",
  priority: 20,
  async handler(ctx) {
    const result = ctx.stepResult;
    if (containsPII(result)) {
      return {
        action: "block",
        reason: "输出包含疑似个人敏感信息（身份证号、银行卡号等）",
      };
    }
    return { action: "continue" };
  },
});

// Hook 3: 日志记录（非阻塞，afterStep 和 afterPlan）
hookManager.register({
  point: "afterStep",
  priority: 100,  // 最后执行
  async handler(ctx) {
    await logToConsole({
      stepId: ctx.step.id,
      duration: ctx.duration,
      tokensUsed: ctx.tokensUsed,
    });
    return { action: "continue" };
  },
});
```

### 集成到 AgentRunner

```typescript
class AgentRunner {
  async *execute(plan: Plan): AsyncGenerator<HarnessEvent> {
    // Plan 执行前 Hook
    const beforeHook = await this.hooks.executeHooks("beforePlan", { plan });
    if (beforeHook.action === "block") {
      yield { type: "plan:error", error: beforeHook.reason };
      return;
    }

    for (const step of plan.steps) {
      // Step 执行前 Hook
      const beforeStep = await this.hooks.executeHooks("beforeStep", { step });
      if (beforeStep.action === "block") {
        yield { type: "plan:error", stepId: step.id, error: beforeStep.reason };
        return;
      }

      // 执行 Step...

      // Step 执行后 Hook
      await this.hooks.executeHooks("afterStep", {
        step, stepResult, duration,
      });
    }

    // Plan 执行后 Hook
    await this.hooks.executeHooks("afterPlan", { plan, stepResults });
  }
}
```

---

## 9.3 代码解析

### 编排的演进路线

```
线性管道（v1）：          A → B → C → D
条件分支（v2）：          A → B → (C or D)
循环自检（v3）：          A → B → C ⇄ D
层级编排（v4）：          Leader Agent → [Worker A, Worker B, Worker C]
```

resumate 目前处于 v1→v2 之间。v3（循环自检）通过 `condition` + `onFail` 实现。v4（多 Agent 协作）超出了单 Plan 模型的范畴——那是 LangGraph/CrewAI 等框架的领域。

### Hooks 与 Step 的区别

| | Step | Hook |
|------|------|------|
| 参与主线执行？ | ✅ 是 Plan 的一部分 | ❌ 透明的横切关注点 |
| 可被跳过？ | 可通过 `condition` 跳过 | 不能跳过（保证执行） |
| 影响 Plan 结果？ | 产出 `stepResults` | 只能 `block` 或 `continue` |
| 典型用途 | 生成内容、验证数据 | 安全检查、日志记录、预算控制 |

---

## 9.4 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "编排就是写 workflow" | 编排包括 workflow（执行流）+ 决策（条件）+ 质量控制（Hooks） |
| "Hooks 可以修改 Plan 结果" | 最好不要。Hooks 的语义是"守护"，修改结果会引入副作用的调试噩梦 |
| "循环越多越好" | 循环增加 Token 消耗和延迟。收敛条件要明确（如：score < 70 或 max 2 rounds） |

### 练习

1. **（★☆☆）** 为 resumate 的 AgentRunner 添加 `condition` 支持：如果 `classify` 输出 `intent: "blank_slate"`，跳过 `collect` 步骤。

2. **（★★☆）** 实现一个最大重试 Hook：限制 `generate → validate → enhance` 循环不超过 3 轮。

3. **（★★★）** 设计一个多 Agent Plan：简历生成 Agent + JD 分析 Agent + 排版 Agent 通过协作完成最终简历。

---

---

> 📍 **进度检查点**：至此，你已完成前三部分（基础篇 + 能力篇 + 品质篇）。回顾你的学习轨迹：
> - 基础篇 (Ch1-3)：Agent 的定义、执行循环、LLM 抽象层
> - 能力篇 (Ch4-6)：工具系统、结构化输出、流式响应
> - 品质篇 (Ch7-9)：上下文工程、提示词工程、编排与Hooks
>
> resumate 现在已经是一个功能完整的 Agent 系统。剩下的第四部分将带你将它从"开发环境"推入"生产世界"。

## 本章小结

- **编排**：从线性到 DAG，`dependsOn` + `condition` + `onFail` 构成完整的状态机
- **Hooks**：透明的横切关注点，守护关键节点
- **演进路线**：线性 → 条件 → 循环 → 层级编排

下一章：**安全与沙箱**——给 Agent 戴上"缰绳"。

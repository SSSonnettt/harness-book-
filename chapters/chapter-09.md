# 第9章：编排与 Hooks — 从单兵到团队

> **核心概念：** Orchestration，Dependency Graph，Hooks，质量门  
> **难度：** ★★★★☆  
> **预计阅读时间：** 50 分钟  
> **学习目标：** 读完本章后，你将能够：
> 1. 用 condition 和 onFail 实现条件分支和自修正循环
> 2. 设计 HookManager 管理横切关注点
> 3. 区分 Step 和 Hook 的职责边界  

---

## 9.1 为什么需要编排与 Hooks？

### 线性执行的三个局限

Ch2 的 AgentRunner 是线性执行：按 Plan 中的步骤顺序逐个执行，出错即中止。这对简单的 pipeline 够用，但随着 Agent 能力增长，你会遇到三类问题：

**1. 条件分支。** 用户说"帮我针对这个 JD 优化简历"——此时 `collect`（信息收集）步骤可以跳过，因为 JD 已经提供了足够的上下文。线性执行做不到"跳过某一步"。

**2. 自我修正。** `generate` 步骤生成的简历可能质量不达标（STAR 覆盖率低、缺少量化指标）。理想情况下，Agent 应该检测到这个问题并自动修正——`generate → validate → (score < 70?) → enhance → validate → present`。这需要一个循环结构，而线性管道不支持循环。

**3. 横切关注点。** 有些检查不属于任何特定步骤，而是每个步骤都需要：Token 预算是否超限？输出是否包含敏感信息？执行时间是否过长？如果把这些检查嵌入每个 Step 的代码中，业务逻辑就被"安全代码"污染了。

### 编排 vs Hooks：两个不同的问题

这两类问题需要两种不同的解法：

- **编排（Orchestration）**解决"步骤如何组合"：线性 → 条件分支 → 循环 → 层级
- **Hooks** 解决"横切关注点如何管理"：不嵌入 Step 代码，而是挂在关键节点上的独立拦截器

这与软件工程中的 **AOP（面向切面编程）** 思想一脉相承。在传统的 OOP 中，日志记录、权限检查、事务管理等"横切关注点"会散布在各个业务方法中。AOP 的解决方案是将它们提取为独立的"切面"，声明式地织入业务代码。Agent 系统中的 Hooks 扮演着同样的角色：

```
传统 AOP:           Agent Hooks:
┌──────────┐       ┌──────────────┐
│ 业务方法  │       │ Agent Step   │
│  + 日志   │  →    │  + beforeHook │
│  + 权限   │       │  + afterHook  │
│  + 事务   │       │  (安全检查)   │
└──────────┘       └──────────────┘
```

**"守护者"模式**是 Hooks 最核心的应用：Hook 不参与主线执行，不修改步骤的输出，但可以在关键节点上"拦截"——如果检查不通过，直接 `block` 阻止执行继续。这比在每个 Step 内部手动检查要干净得多，也安全得多（你不可能"忘记"添加 Hook，就像你不可能绕过 Spring 的 `@Transactional` 注解）。

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

### Hook 误拦截的调试技巧

Hook 最常见的故障模式是**误拦截**——本该通过的操作被 Hook 判定为 block。调试时关注三个断层：

**断层 1：校验逻辑过于严格。** `safetyHook` 的正则 `/delete|drop|rm\s+-rf/` 会把 "delete old resume" 这种合法操作拦截。修复：用允许列表（allowlist）辅助判断，而非纯拒绝列表。

**断层 2：上下文不足导致误判。** Hook 只看到当前 Step 的输入输出，看不到之前的上下文。如果 Step 3 的输出是「删除」，但 Step 1 已确认这是用户请求的数据清理操作，Hook 应该引入 `harnessState` 查看上游确认。

**断层 3：未暴露拦截原因给用户。** Hook 返回 `{ action: "block", reason: "..." }` 时，错误信息必须对用户可读。`"Regex /bad_pattern/ matched"` 没有帮助，`"您的简历包含不被支持的符号，请改用标准 Unicode 字符"` 才有意义。

> 💡 **调试口诀**：Hook 报 block → 查看 reason → 如果 reason 对不上实际意图，问题在 Hook 逻辑本身，不在 LLM 输出。不要调 prompt，调 Hook。

---

> 📍 **进度检查点**：至此，你已完成前三部分（基础篇 + 能力篇 + 品质篇）。回顾你的学习轨迹：
> - 基础篇 (Ch1-3)：Agent 的定义、执行循环、LLM 抽象层
> - 能力篇 (Ch4-6)：工具系统、结构化输出、流式响应
> - 品质篇 (Ch7-9)：上下文工程、提示词工程、编排与Hooks
>
> resumate 现在已经是一个功能完整的 Agent 系统。剩下第四部分把它从开发环境推进生产世界。

## 本章小结

- **编排**：从线性到 DAG，`dependsOn` + `condition` + `onFail` 构成完整的状态机
- **Hooks**：透明的横切关注点，守护关键节点
- **演进路线**：线性 → 条件 → 循环 → 层级编排

下一章：安全与沙箱——给 Agent 戴上缰绳。

### 自检问题

1. 编排（Orchestration）和 Hooks 分别解决什么问题？为什么不能用一个替代另一个？
2. `condition` + `onFail` 如何实现自修正循环？画出 `generate → validate → (score<70?) → enhance → revalidate → present` 的状态转换图。
3. Hooks 的 `action: "block"` 和 `action: "continue"` 设计如何体现"守护者"模式？为什么 Hooks 不应该直接修改 Step 的输出？

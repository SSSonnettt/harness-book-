# 第11章：评估与可观测性 — AI 系统的质量度量

> **核心概念：** 评估接口，可观测性，质量评分，日志追踪  
> **难度：** ★★★★☆  
> **预计阅读时间：** 45 分钟  

---

## 开篇故事

resumate 的简历质量如何评估？早期我的方法很简单：**自己看**。

但这不可持续。每改一次提示词就要手动测 20 个 case？做不到。

我需要一套**自动化的质量评估体系**。不需要人工判断每一份简历，而是用指标来量化：
- ATS 兼容性评分：关键词匹配率？
- 内容质量评分：STAR 法则覆盖率？量化指标密度？
- 完整性评分：所有模块都填了吗？

这就是"评估接口"——Harness 的第六个核心组件。它把 Agent 的输出质量从"我觉得还行"变成可度量的数字。

---

## 11.1 为什么需要评估？

AI 系统的特殊性：**同样的输入可能产生不同的输出**。你改了提示词，你以为改好了，但也许在 30% 的 case 上变差了。

没有评估就没有迭代方向。

三个关键问题：
1. 质量的基线是什么？（benchmark）
2. 这次改动是提升还是降低了质量？（regression detection）
3. 哪个步骤是瓶颈？（bottleneck analysis）

---

## 11.2 什么是可观测性？

### 三层观测

```
┌──────────────────────────────────┐
│ L3: 业务指标 (Business Metrics)  │  ← "简历质量评分 85/100"
├──────────────────────────────────┤
│ L2: Agent 行为 (Agent Behavior)  │  ← "classify 准确率 98%"
├──────────────────────────────────┤
│ L1: 系统指标 (System Metrics)    │  ← "平均延迟 12s, Token 消耗 3200"
└──────────────────────────────────┘
```

### resumate 的评估体系

**L1 系统指标：**

```typescript
interface ExecutionTrace {
  planId: string;
  startTime: number;
  endTime: number;
  steps: StepTrace[];
  totalTokens: number;
  totalCost: number;
}

interface StepTrace {
  stepId: string;
  type: StepType;
  startTime: number;
  endTime: number;
  tokensUsed: number;
  success: boolean;
  errorMessage?: string;
}
```

通过解析 HarnessEvent 流自动生成执行追踪：

```typescript
function buildTrace(events: HarnessEvent[]): ExecutionTrace {
  const trace: ExecutionTrace = {
    planId: "",
    startTime: 0,
    endTime: 0,
    steps: [],
    totalTokens: 0,
    totalCost: 0,
  };

  for (const event of events) {
    switch (event.type) {
      case "plan:start":
        trace.planId = event.planId;
        trace.startTime = Date.now();
        break;
      case "step:start":
        trace.steps.push({
          stepId: event.stepId,
          type: "chat", // 从 Plan 中获取
          startTime: Date.now(),
          endTime: 0,
          tokensUsed: 0,
          success: true,
        });
        break;
      case "step:done":
        const step = trace.steps.find(s => s.stepId === event.stepId);
        if (step) step.endTime = Date.now();
        break;
      case "plan:error":
        const errStep = trace.steps.find(s => s.stepId === event.stepId);
        if (errStep) { errStep.success = false;
          errStep.errorMessage = event.error; }
        break;
      case "plan:done":
        trace.endTime = Date.now();
        break;
    }
  }

  return trace;
}
```

**L2 Agent 行为指标：**

```typescript
function evaluateAgentBehavior(trace: ExecutionTrace): {
  classifyAccuracy: number;
  validateCatchRate: number;  // 验证步骤发现了多少问题
  averageStepLatency: number;
} {
  const classifyStep = trace.steps.find(s => s.stepId === "classify");
  const validateStep = trace.steps.find(s => s.stepId === "validate");

  return {
    classifyAccuracy: classifyStep?.success ? 1 : 0,  // 简化版
    validateCatchRate: validateStep?.success ? 1 : 0,
    averageStepLatency:
      trace.steps.reduce((sum, s) => sum + (s.endTime - s.startTime), 0)
      / trace.steps.length,
  };
}
```

**L3 业务指标 — 简历质量评分：**

```typescript
interface ResumeScore {
  atsCompatibility: number;   // 0-100: ATS 关键词覆盖率
  contentQuality: number;     // 0-100: STAR 覆盖率、量化指标密度
  completeness: number;       // 0-100: 必填模块完成度
  overall: number;            // 加权总分
}

function scoreResume(resume: Resume, jdKeywords?: string[]): ResumeScore {
  // ATS 兼容性
  const atsScore = jdKeywords
    ? calculateKeywordMatch(resume, jdKeywords)
    : 100;  // 无 JD 时满分

  // 内容质量：检查 STAR 要素和量化指标
  const workModules = resume.modules.filter(
    m => m.type === "work-experience"
  );
  const starScore = workModules.reduce((score, mod) => {
    const desc = mod.data.description as string || "";
    const highlights = (mod.data.highlights as string[]) || [];
    const allText = [desc, ...highlights].join(" ");

    let starPoints = 0;
    if (/(背景|项目|挑战|负责|团队)/.test(allText)) starPoints++;  // Situation
    if (/(负责|任务|目标|职责|需要)/.test(allText)) starPoints++;  // Task
    if (/(主导|设计|开发|优化|重构|实施|领导)/.test(allText))
      starPoints++;  // Action
    if (/(\d+%|提升|降低|节省|增长|减少)/.test(allText)) starPoints++; // Result
    return score + (starPoints / 4) * 100;
  }, 0);
  const contentScore = workModules.length > 0
    ? starScore / workModules.length
    : 0;

  // 完整性
  const requiredModules = ["header", "work-experience", "education", "skills"];
  const presentModules = requiredModules.filter(type =>
    resume.modules.some(m => m.type === type)
  );
  const completenessScore =
    (presentModules.length / requiredModules.length) * 100;

  return {
    atsCompatibility: Math.round(atsScore),
    contentQuality: Math.round(contentScore),
    completeness: Math.round(completenessScore),
    overall: Math.round(
      atsScore * 0.3 + contentScore * 0.4 + completenessScore * 0.3
    ),
  };
}
```

---

## 11.3 动手实践：在 resumate 中集成质量评分

在 `validate` 步骤中集成评分：

```typescript
registry.register("validateResume", async (args) => {
  const resume = resumeSchema.parse(args.resume);
  const jdKeywords = args.jdKeywords as string[] | undefined;
  const issues: string[] = [];

  // 结构验证
  if (resume.modules.length === 0) issues.push("简历没有任何模块");
  const header = resume.modules.find(m => m.type === "header");
  if (!header) issues.push("缺少头部模块");
  else if (!header.data.name) issues.push("姓名未填写");

  // 质量评分
  const score = scoreResume(resume, jdKeywords);

  // 质量告警
  if (score.contentQuality < 60) {
    issues.push(`内容质量偏低 (${score.contentQuality}/100)：建议加强 STAR 法则和量化指标`);
  }
  if (score.atsCompatibility < 50) {
    issues.push(`ATS 兼容性偏低 (${score.atsCompatibility}/100)：建议增加 JD 关键词`);
  }

  return {
    valid: score.overall >= 50,
    issues,
    score,
  };
});
```

---

## 11.4 代码解析

### 评估与 Step 类型的关系

| Step 类型 | 评估方式 | 指标 |
|-----------|---------|------|
| `tool` | 单元测试 | 正确率、执行时间 |
| `chat` | A/B 对比 + 人工抽查 | 对话质量、信息收集完整性 |
| `structured` | Schema 验证 + 内容评分 | Zod pass rate、质量维度评分 |
| `compose` | 集成测试 | 端到端成功率 |

### 可观测性金字塔

不要一开始就建全套监控。按阶段建设：

```
阶段 1: 基础日志      → console.log Trace，手动查看
阶段 2: 结构化追踪    → buildTrace(events)，存储在日志文件
阶段 3: 聚合分析      → 多轮运行的统计（平均延迟、Token 趋势）
阶段 4: 自动告警      → score < 50 自动通知
```

resumate 目前处于阶段 1-2，作为教学项目已经足够。

---

## 11.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "评分能完全替代人工判断" | 不能。评分给你方向，但最终需要人工抽查验证 |
| "指标越多越好" | 3-5 个关键指标 > 20 个次要指标 |
| "上线后就不用评估了" | 持续评估。提示词改动、模型升级都可能改变质量 |

### 练习

1. **（★☆☆）** 为 resumate 实现执行追踪的持久化存储（localStorage）。

2. **（★★☆）** 扩展 `scoreResume`：加入可读性评分（Flesch 指数）和行业关键词密度分析。

3. **（★★★）** 实现 regression test：收集 5 份基准简历，每次修改提示词后自动跑分对比。

---

## 本章小结

- **评估**：把"感觉"变成"数据"
- **三层观测**：系统指标（延迟/Token）→ Agent 行为（准确率）→ 业务指标（简历质量）
- **简历质量评分**：ATS 兼容性 + 内容质量 + 完整性

下一章（最后一章）：**从 Demo 到产品**——resumate 的完整发布。

# 第11章：评估与可观测性 — AI 系统的质量度量

> **核心概念：** 评估接口，可观测性，质量评分，日志追踪  
> **难度：** ★★★★☆  
> **预计阅读时间：** 45 分钟  
> **学习目标：** 读完本章后，你将能够：
> 1. 建立三层观测体系（系统/Agent 行为/业务）
> 2. 实现简历质量自动评分
> 3. 使用 LLM-as-Judge 评估主观质量维度  

---

## 11.1 为什么需要评估？

### "盲飞"的代价

假设你花了一下午优化了 resumate 的 STAR 法则 prompt，让它更强调"量化指标"。你感觉效果变好了——生成的简历描述看起来更具体了。但你不知道的是：

- 在 70% 的 case 上，内容质量确实提升了（从 72 分升到 85 分）
- 在 20% 的 case 上，没有明显变化
- 在 10% 的 case 上，**质量反而下降了**（从 78 分降到 61 分）——因为 LLM 为了"堆数字"编造了虚假数据

没有评估体系，你只看到了那 70%，根本不知道那 10% 的存在。这就是**"盲飞"（Flying Blind）**——凭感觉迭代，不知道自己是在提升产品还是在制造新 bug。

### AI 系统评估的特殊挑战

AI 系统的评估比传统软件困难得多，根本原因在于**非确定性**：

**1. 同样的输入，不同的输出。** 传统软件的 `f(x)` 永远返回同一个值。LLM 的 `generate(prompt)` 每次可能不同。你改了提示词，在 70% 的 case 上变好了，但可能在 30% 的 case 上变差了——没有评估体系，你根本不知道。

**2. "正确"的标准是主观的。** 传统软件的测试用例有明确的 expected output。但"一份好的简历"没有唯一标准——什么算"专业"？什么算"有影响力"？这些主观维度需要特殊的评估方法（如 LLM-as-Judge）。

**3. Regression 检测困难。** 传统软件用单元测试检测 regression：改了代码，跑一遍测试就知道有没有破坏已有功能。Agent 的"测试"需要统计方法——不是"这次通过/失败"，而是"100 次的平均质量是否下降"。

> ⚠️ **核心洞察**：Agent 开发中，"感觉变好了"是最大的陷阱。没有量化的评估体系，你就是在黑暗中掷骰子。你不评估，就不知道自己在往哪个方向走。

### 评估的三个关键问题

没有评估就没有迭代方向。每次修改 prompt 或模型时，你需要回答：

1. **质量的基线是什么？**（benchmark）→ 当前版本的平均评分、各维度分布
2. **这次改动是提升还是降低了质量？**（regression detection）→ 对比改动前后的评分，特别注意**退化案例**（以前好的现在坏了）
3. **哪个步骤是瓶颈？**（bottleneck analysis）→ classify 准确率？generate 质量？validate 覆盖率？

这三个问题对应三层观测体系——从系统指标到 Agent 行为到业务质量。

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

通过解析 HarnessEvent 流自动生成执行追踪。核心思路很简洁：遍历事件流，`plan:start` 记录开始时间，`step:start`/`step:done` 记录每个步骤的起止时间，`plan:error` 标记失败步骤：

```typescript
function buildTrace(events: HarnessEvent[]): ExecutionTrace {
  const trace: ExecutionTrace = { planId: "", startTime: 0, endTime: 0,
    steps: [], totalTokens: 0, totalCost: 0 };

  for (const event of events) {
    // 根据事件类型更新 trace（完整 switch/case 见 resumate 源码）
    if (event.type === "plan:start") trace.startTime = Date.now();
    if (event.type === "step:start") trace.steps.push({ /* 记录步骤开始 */ });
    if (event.type === "step:done")  /* 记录步骤结束和耗时 */;
    if (event.type === "plan:error") /* 标记失败步骤和错误信息 */;
  }
  return trace;
}
```

这个函数的价值在于：**从已有的事件流中自动提取执行指标，不需要任何额外的 instrumentation**。AgentRunner 产出 HarnessEvent 流，`buildTrace` 将其转化为可分析的追踪数据——完整实现约 40 行，见 [resumate 源码](https://github.com/SSSonnettt/resumate)。

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

简历质量评分从三个维度量化输出质量：

```typescript
interface ResumeScore {
  atsCompatibility: number;   // 0-100: ATS 关键词覆盖率
  contentQuality: number;     // 0-100: STAR 覆盖率、量化指标密度
  completeness: number;       // 0-100: 必填模块完成度
  overall: number;            // 加权总分（ATS×0.3 + Content×0.4 + Completeness×0.3）
}
```

| 评分维度 | 检查方式 | 示例规则 |
|---------|---------|---------|
| ATS 兼容性 | JD 关键词匹配率 | 简历中包含 JD 中 `React`、`TypeScript` 等关键词的比例 |
| 内容质量 | STAR 要素正则检测 | 每条工作经历是否包含情境/任务/行动/结果四要素 |
| 完整性 | 必填模块存在性检查 | 是否包含 header、work-experience、education、skills 四个模块 |

这种"多维度打分"的思路是通用的——无论是评估简历、评估文章还是评估代码，核心都是**将主观的"好不好"分解为多个可量化的客观维度**。`scoreResume` 的完整实现（~50 行正则匹配 + 评分逻辑）见 [resumate 源码](https://github.com/SSSonnettt/resumate)。

---

## 11.3 动手实践：在 resumate 中集成质量评分

将质量评分集成到 Ch4 的 `validateResume` 工具中，只需在结构验证后添加评分调用：

```typescript
registry.register("validateResume", async (args) => {
  const resume = resumeSchema.parse(args.resume);
  const issues: string[] = [];

  // 结构验证（Ch4 已有）
  if (!resume.modules.find(m => m.type === "header")) issues.push("缺少头部模块");

  // 质量评分（新增）
  const score = scoreResume(resume, args.jdKeywords as string[]);
  if (score.contentQuality < 60) issues.push(`内容质量偏低：建议加强 STAR 法则`);
  if (score.atsCompatibility < 50) issues.push(`ATS 兼容性偏低：建议增加 JD 关键词`);

  return { valid: score.overall >= 50, issues, score };
});
```

注意这里的设计：**评分不是替代验证，而是增强验证**。结构验证检查"有没有"（有没有姓名、有没有模块），质量评分检查"好不好"（STAR 覆盖率、关键词匹配）。两者结合，形成完整的质量防线。

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

### LLM-as-Judge：用模型评估模型

2024-2025 年兴起的一种评估方法：**用更强的 LLM 来评估 Agent 的输出质量**。这对于规则难以覆盖的维度（如"简历描述是否专业"、"语言是否自然"）特别有效。

```typescript
async function llmJudgeResume(
  resume: Resume,
  judgeProvider: LLMProvider,  // 用更强的模型做评判
): Promise<{ score: number; feedback: string }> {
  const prompt = `你是一位资深 HR 和简历评审专家。请评估以下简历：

${JSON.stringify(resume, null, 2)}

评估维度：
1. 专业性：描述是否使用专业术语和行业标准表达
2. 影响力：是否突出了可量化的成果和影响
3. 完整性：各模块信息是否充分
4. ATS 友好度：格式和关键词是否适合自动筛选

请输出 JSON：{ "score": 0-100, "feedback": "具体改进建议" }`;

  const result = await judgeProvider.generateStructured({
    messages: [{ role: "user", content: prompt }],
    schema: z.object({
      score: z.number().min(0).max(100),
      feedback: z.string(),
    }),
  });
  return result;
}
```

LLM-as-Judge 的优势和局限：

| | 优势 | 局限 |
|--|------|------|
| 覆盖面 | 能评估"专业性"等主观维度 | 评估本身也有偏差 |
| 成本 | 比人工评审便宜得多 | 每次评估仍需 API 调用 |
| 一致性 | 比人工评审更一致 | 不同模型给出不同评分 |

### 规则评分 vs LLM-as-Judge：什么时候用哪个？

一个实用的决策框架：

| 评估维度 | 推荐方法 | 原因 |
|---------|---------|------|
| 字段是否存在（姓名、邮箱） | 规则 | 确定性问题，Zod/正则 100% 准确 |
| ATS 关键词覆盖率 | 规则 | 集合运算，代码精确无歧义 |
| STAR 要素检测 | 规则（正则） | 模式匹配问题，正则比 LLM 更快更准 |
| "描述是否专业" | LLM-as-Judge | 主观维度，需要语义理解 |
| "语言是否自然流畅" | LLM-as-Judge | 没有客观标准，需要语言直觉 |
| "整体印象" | LLM-as-Judge | 综合主观判断 |

> ⚠️ **混合策略是黄金法则**：规则评分覆盖确定性维度，LLM-as-Judge 覆盖主观维度，人工抽查验证评估体系本身的可靠性。不要把鸡蛋放在一个篮子里——如果只用规则，你会漏掉"虽然格式正确但写得不好"的简历；如果只用 LLM-as-Judge，你会得到看似合理但不一致的评分。

### LLM-as-Judge 的校准策略

LLM-as-Judge 本身需要"被评估"——你怎么知道评判模型给出的 85 分是合理的？校准方法：

1. **人工标注基准集**：取 10-20 份简历，由 2-3 位人工评审员独立打分，取平均值作为 ground truth
2. **计算 Judge 偏差**：对比 Judge 评分与人工评分，计算 MAE（平均绝对误差）和相关性
3. **设定偏差阈值**：如果 MAE > 10 分，Judge 不够可靠——换更强的模型或调整 prompt
4. **定期复校**：每季度用新样本重新校准，检测模型版本升级是否改变了 Judge 行为

```typescript
// 校准函数示例
function calibrateJudge(
  judgeScores: number[],
  humanScores: number[],
): { mae: number; correlation: number; reliable: boolean } {
  const mae = judgeScores.reduce((sum, js, i) =>
    sum + Math.abs(js - humanScores[i]), 0
  ) / judgeScores.length;
  
  return {
    mae: Math.round(mae * 10) / 10,
    correlation: pearsonCorrelation(judgeScores, humanScores),
    // 注：pearsonCorrelation 可用 simple-statistics 库的 sampleCorrelation()
    // 或自行实现：npm install simple-statistics
    reliable: mae < 10,  // 阈值：平均误差不超过 10 分
  };
}
```

最佳实践：将 LLM-as-Judge 与规则评分结合——规则评分覆盖确定性维度（完整性、关键词匹配），LLM-as-Judge 覆盖主观维度（专业性、表达质量），人工抽查验证评估体系本身的可靠性。通过定期校准，确保 Judge 不会"漂移"。

### Agent 评估的通用框架

resumate 的三层评估体系（系统指标 → Agent 行为 → 业务质量）可以泛化到任何 Agent 项目：

1. **确定评估维度**：先回答"什么是好的输出"，将其分解为 3-5 个可量化的维度
2. **选择评估方法**：确定性维度用规则（正则、Schema），主观维度用 LLM-as-Judge，关键维度用人工抽查
3. **建立基准数据集**：收集 5-10 个有代表性的输入→输出对，用当前系统评分作为基线
4. **持续监控**：每次改动后自动跑基准数据集，对比评分变化（regression detection）

这套框架不限于简历场景。旅行规划 Agent 的评估维度可能是"行程合理性、预算准确性、景点覆盖率"；代码审查 Agent 的维度可能是"bug 检出率、误报率、建议可操作性"。思路是一样的：**将模糊的质量判断转化为可重复的量化指标**。

---

## 11.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "评分能完全替代人工判断" | 不能。评分给你方向，但最终需要人工抽查验证 |
| "指标越多越好" | 3-5 个关键指标 > 20 个次要指标 |
| "上线后就不用评估了" | 持续评估。提示词改动、模型升级都可能改变质量 |

---

## 本章小结

- **评估**：把"感觉"变成"数据"
- **三层观测**：系统指标（延迟/Token）→ Agent 行为（准确率）→ 业务指标（简历质量）
- **简历质量评分**：ATS 兼容性 + 内容质量 + 完整性

下一章（最后一章）：**从 Demo 到产品**——resumate 的完整发布。

### 自检问题

1. "盲飞"（Flying Blind）是什么意思？为什么"感觉变好了"是 Agent 开发中最大的陷阱？
2. resumate 的三层观测体系（L1/L2/L3）各度量什么？如何从 L1 的延迟数据推导出 L3 的质量瓶颈？
3. LLM-as-Judge 的校准流程是什么？为什么即使 Judge 很准，也要定期复校？

# 第8章：提示词工程 — 与模型高效对话

> **核心概念：** Prompt Engineering，System Prompt 设计，指令分层  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 40 分钟  
> **学习目标：** 读完本章后，你将能够：
> 1. 设计三层 Prompt 架构（Plan/Step/Context）
> 2. 应用三条经验法则优化提示词质量
> 3. 用 A/B 测试框架量化提示词效果  

---

## 8.1 为什么需要提示词工程？

LLM 是概率性系统，它根据 prompt 中的模式和指令来预测下一个 token。你的 prompt 质量直接决定输出质量。

但 Agent 的提示词工程比普通 chatbot 复杂得多，原因有三：

**1. Agent 的 prompt 是多层的。** 普通 chatbot 只有一个 system prompt。Agent 有 Plan 层（角色定义）、Step 层（步骤目标）、Context 层（动态数据注入）——三层 prompt 相互叠加，任何一层的歧义都可能导致输出偏移。

**2. 错误的代价被放大。** 普通 chatbot 说错一句话，用户重新问就行。Agent 的 prompt 错误会被执行循环放大——一个 `collect` 步骤的 prompt 措辞不当，可能导致收集到的信息不完整，进而影响后续所有步骤。

**3. prompt 影响 attention 分配。** LLM 的 attention 机制会根据 prompt 中的位置和措辞分配"注意力"。放在 prompt 开头的指令权重更高；用"必须"比"建议"更能引起模型的重视；给出具体的正例和反例比抽象规则更有效——因为模型本质上是模式匹配器，它对示例的模仿能力远强于对规则的遵循（参见 Brown et al. 2020 的 GPT-3 论文 *"Language Models are Few-Shot Learners"*）。

**好 prompt 的特征：**

1. **角色先行**：先告诉模型"你是谁"（定义专业边界）
2. **约束明确**：告诉模型"你不能做什么"（防止自由发挥）
3. **输出格式精确**：示例 > 描述（LLM 对示例的模仿能力远强于对规则的遵循）
4. **上下文精简**：只给需要的，不给全部（这是第 7 章的内容）

---

## 8.2 什么是 Harness 中的提示词体系？

### 三层 Prompt 架构

resumate 的 prompt 体系分为三层：

```
Layer 1: Plan-level System Prompt
   ↓ "你是专业的求职顾问..."
Layer 2: Step-level System Prompt  
   ↓ "现在你需要收集用户的职业信息..."
Layer 3: Runtime Context Injection
   ↓ "用户意图：jd_optimize, JD 关键词：React, TypeScript"
```

每一层的职责不同：

| 层 | 职责 | 变化频率 | 示例 |
|----|------|---------|------|
| Plan 层 | Agent 的角色与边界 | 极少变 | "你是求职顾问，只回答简历相关问题" |
| Step 层 | 当前步骤的目标与方法 | 每步不同 | "收集用户的工作经历" vs "验证简历完整性" |
| Context 层 | 当前步骤需要的上下文数据 | 每次运行不同 | 用户数据、前序步骤结果 |

### resumate 中的实际 prompt

**Plan 层的 system prompt（整个 Plan 的角色定义）：**

```
你是一位专业的中文求职顾问。你的职责是：
1. 引导用户提供完整的职业信息
2. 生成专业的、ATS 友好的简历
3. 每条工作经历须使用 STAR 法则
4. 每个描述须包含至少一个量化指标
5. 只回答简历相关的问题，礼貌拒绝无关话题
```

**Step 层的 system prompt（collect 步骤）：**

```
现在你需要收集用户的职业信息。请按以下顺序引导：
1. 基本信息：姓名、求职方向、联系方式
2. 工作经历：每段工作的时间、公司、职位、STAR 描述
3. 教育背景：学校、学位、专业
4. 技能：技术栈、语言、证书
每个类别至少询问一轮，在用户确认满意后进入下一类别。
```

---

## 8.3 动手实践：resumate 的提示词优化

### STAR 法则提示词

resumate 生成工作经历时的核心提示词片段：

```
对于每段工作经历，你必须使用 STAR 法则：
- 情境 (Situation)：项目背景或面临的挑战
- 任务 (Task)：你承担的具体职责
- 行动 (Action)：你采取的具体措施
- 结果 (Result)：可量化的成果

要求：
- 每个 bullet point 至少包含一个数字指标
- 使用主动动词开头（主导、设计、优化、重构）
- 避免模糊描述（"负责日常工作"→ 不合格）

示例：
✅ "主导电商平台前端重构，将页面加载时间从 4.2s 优化至 1.1s（提升 74%），
   支撑 GMV 增长 120%"
❌ "负责前端开发工作"
```

### ATS 兼容性提示词

```
你必须确保简历对 ATS（Applicant Tracking System）友好：
1. 使用标准章节标题（Work Experience 而非 My Journey）
2. 包含 JD 中提到的精确技能名（JD 写 React.js 就用 React.js，不要简写 React）
3. 避免表格、图片、特殊符号
4. 使用标准日期格式（YYYY.MM - YYYY.MM）
```

### 多模板提示词切换

resumate 通过动态 `systemPrompt` 支持不同简历风格：

```typescript
const promptByStyle: Record<string, string> = {
  professional: "使用正式、专业的商务语言。避免口语化表达。",
  creative: "在保持专业性的前提下，允许创意的排版和表达方式。",
  concise: "极致精简。每个 bullet 不超过 15 个词。使用电报式的简洁风格。",
};

{
  id: "generate",
  type: "structured",
  systemPrompt: (runtime) => {
    const style = runtime.context.style as string;
    return `${basePrompt}\n\n风格要求：${promptByStyle[style]}`;
  },
}
```

---

## 8.4 代码解析

### 提示词工程的三条经验法则

**法则 1：示例优于描述**

不要只说"用 STAR 法则"，要给出具体的正确示例和错误示例。LLM 是模式匹配器，它对示例的模仿远比遵循抽象规则准确（这一现象在 Brown et al. 2020 的 GPT-3 论文 *"Language Models are Few-Shot Learners"* 中得到系统验证）。

**法则 2：约束优于自由**

不要只说"生成好的简历"，要明确：
- ✅ 做什么 → "每个 bullet 包含数字指标"
- ❌ 不能做什么 → "不要使用被动语态"
- 格式约束 → "日期格式 YYYY.MM"

**法则 3：角色决定语气**

- "你是求职顾问" → 专业、引导式
- "你是简历审核员" → 批判性、检查式
- "你是排版专家" → 关注格式和可读性

角色定义了输出的行为边界。不要在同一个 prompt 中混合多个角色。

### Think Mode 的价值

resumate 的 AnthropicProvider 和 OpenAICompatProvider 都支持 `thinking: { type: "enabled" }`。对于复杂任务（如从长段描述中提取 STAR 要素），开启 think mode 可以显著提升质量。但代价是输出速度降低（模型需要先"思考"再输出）。在 resumate 中，`generate` 步骤（需要复杂推理）开启 thinking，`collect` 步骤（对话式，速度优先）关闭。

### 系统化改进 prompt 的通用方法论

不只是 resumate，任何 Agent 项目都可以用以下流程系统化地改进提示词：

1. **建立基线**：收集 5-10 个有代表性的测试用例，用当前 prompt 跑一遍，记录评分
2. **识别瓶颈**：分析低分案例——是角色定义不清？约束缺失？还是示例不够？
3. **单变量实验**：每次只改一个维度（如只改 STAR 要求），对比基线评分
4. **验证无回退**：确保改动在提升目标维度的同时，没有降低其他维度
5. **版本化记录**：每次成功的改动记录版本号和变更原因，支持回滚

这个流程的本质是**把软件工程中的"单元测试 + 回归测试 + 版本控制"应用到提示词管理中**。prompt 不是写一次就完事的——它和代码一样，需要持续迭代和维护。

### 动手实验：A/B 测试提示词

提示词的改进不能靠感觉。核心方法论是：**定义指标 → 跑测试用例 → 对比数据**。

```typescript
// 定义提示词变体
const promptVariants = {
  baseline: `你是简历专家。请根据用户信息生成简历。`,
  starEnforced: `你是简历专家。每条工作经历必须使用 STAR 法则，
每个 bullet 包含至少一个数字指标。参考示例：...`,
  atsOptimized: `${starEnforced}\n此外，确保简历对 ATS 友好：...`,
};

// 对每个变体，用同一组测试用例生成简历并评分
for (const [name, prompt] of Object.entries(promptVariants)) {
  const scores = testCases.map(tc => {
    const resume = await generateWithPrompt(prompt, tc);
    return scoreResume(resume, tc.jd);  // 自动评分（Ch11 详述）
  });
  results[name] = average(scores);
}
```

实验结果可能像这样：

| 提示词变体 | 平均分 | 最低分 | 最高分 |
|-----------|--------|--------|--------|
| baseline | 62 | 45 | 78 |
| starEnforced | 78 | 65 | 88 |
| atsOptimized | 85 | 72 | 93 |

效果一目了然：光是加上 STAR 法则要求就提升了 16 分。加上 ATS 优化又提升了 7 分。用数据迭代提示词，不凭感觉改，用指标说话。

---

## 8.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "提示词越长越好" | 不是。越长的提示词信噪比越低，关键指令可能被淹没 |
| "一个 prompt 解决所有" | 不要。分层 prompt：不同步骤用不同的 system prompt |
| "提示词写好了就完了" | 持续测试、A/B 对比、迭代优化 |

---

## 本章小结

- **三层 prompt 架构**：Plan 层（角色）+ Step 层（目标）+ Context 层（数据）
- **三条法则**：示例 > 描述，约束 > 自由，角色决定语气
- **Think Mode**：复杂推理开启，简单对话关闭

下一章：编排与 Hooks——从单兵到团队。

### 自检问题

1. Agent 的提示词工程为什么比普通 chatbot 复杂？三个原因中哪个影响最大？
2. 三层 Prompt 架构（Plan/Step/Context）中，每一层的职责和变化频率有何不同？
3. "示例优于描述"这条法则的底层原理是什么？为什么 LLM 对示例的模仿远强于对抽象规则的遵循？

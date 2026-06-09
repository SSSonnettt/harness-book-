# 第7章：上下文工程 — 信息管理的艺术

> **核心概念：** Context Engineering，RuntimeValue，Context Rot，渐进式加载  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 45 分钟  

## 开篇故事

resumate 上线一个月后的某个深夜，我盯着监控面板上的 Token 消耗曲线，发现了一个令人不安的趋势。

用户平均对话轮次是 8 轮。第 1 轮 Agent 的回复质量很高——准确、相关、有洞察。到第 5 轮，质量还行。到第 8 轮——用一个流行的术语——Agent 开始"犯傻"了。

最经典的一个 bug report：用户在第 1 轮就说了"我叫陈小明，5年前端经验"。到了第 8 轮，Agent 突然问："请问你怎么称呼？"

用户回复了三个字：**"我叫陈小明。"**

然后他另外开了一个新对话。

这是典型的 **Context Rot（上下文腐烂）**。问题不在于模型不够聪明——Claude Sonnet 4 足够聪明。问题在于我喂给它的信息：我把全部 8 轮对话历史不加筛选地塞给了 LLM。到第 8 轮时，prompt 中有 ~2400 Token，但有效的上下文不到 30%——其余全是陈旧的、过时的、已经不需要的对话残余。

病因很清楚：我把全部对话历史不加筛选地塞给了 LLM：

```typescript
// 坏做法
const messages = [
  { role: "system", content: "你是求职顾问..." },
  ...allConversationHistory,  // 8 轮对话，可能几千 token
  { role: "user", content: "继续..." },
];
```

8 轮对话中，前 3 轮是"收集基本信息"，中间 3 轮是"细化工作经历"，最后 2 轮是"确认"。但到第 8 轮时，前 3 轮的细节信息不再需要反复注入——它们已经在结构化数据中保存了。**重复注入浪费 Token，稀释关键信息，降低 LLM 的注意力质量。**

## 7.1 为什么需要上下文工程？

Context Engineering 是 Harness 的"元能力"——它不直接执行操作，而是管理**进入 LLM 的信息**。

核心公式：

> **Context Window Management = Information Diet（信息节食）**

就像人吃太多垃圾食品会变迟钝，LLM 被喂入过多无用信息也会变笨。

三个核心策略：

| 策略 | 解决的问题 |
|------|-----------|
| **压缩 (Compression)** | 长对话摘要，减少 Token |
| **卸载 (Offloading)** | 把结构化数据存到外部，只传引用 |
| **渐进加载 (Progressive Loading)** | 按需注入，不一次性塞入 |

## 7.2 什么是上下文工程？

### RuntimeValue：按需计算而非预计算

resumate 的 AgentRunner 中，Step 的参数是 `RuntimeValue<T>`：

```typescript
type RuntimeValue<T> = T | ((runtime: StepRuntime) => T);
```

这不是一个语法糖——它是上下文工程的核心机制。

**静态值**适用于不变的信息：
```typescript
systemPrompt: "你是一位专业的求职顾问。",  // 永远不变
```

**动态函数**适用于需要从上下文中提取的信息：
```typescript
userPromptTemplate: (runtime) => {
  // 只注入当前步骤需要的上下文片段
  const intent = runtime.stepResults.classify?.intent;
  const userInfo = runtime.stepResults.collect?.text;
  // 不注入 validate 的结果（当前步骤不需要它）
  return `意图：${intent}\n用户信息：${userInfo}`;
}
```

关键原则：**只注入当前步骤需要的信息，不注入"全部历史"**。

### Context Rot 的四种表现

| 症状 | 原因 | 对策 |
|------|------|------|
| LLM "忘记"早期信息 | 窗口前半部分被后文挤压 | 结构化存储早期信息，显式注入 |
| LLM 重复提问 | 早期答案被淹没 | 对话摘要 + 已收集信息清单 |
| LLM 回答质量下降 | 信噪比太低 | 压缩旧轮次 + 清理无效消息 |
| Token 浪费 | 冗余信息反复传输 | 摘要替代全文 |

### 实际的上下文管理策略

在 resumate 中，我们采用了一个三层策略：

```
层1: 结构化状态（localStorage / Zustand）
     ↓ 存储精确数据
层2: 对话摘要（每次 Agent 运行前生成）
     ↓ 提供上下文
层3: 最近 N 轮对话（N=2-3）
     ↓ 保持对话连贯性
```

LLM 只收到层 2（摘要）和层 3（最近对话）。历史全貌存在层 1 中，通过 prompt 模板注入特定需要的片段。

## 7.3 动手实践：为 resumate 添加对话摘要

```typescript
// 对话摘要器
function summarizeConversation(
  messages: ChatMessage[],
  collectedInfo: Record<string, unknown>
): string {
  const parts: string[] = [];

  // 已收集的结构化信息
  if (Object.keys(collectedInfo).length > 0) {
    parts.push("## 已收集的用户信息");
    for (const [key, value] of Object.entries(collectedInfo)) {
      parts.push(`- ${key}: ${JSON.stringify(value)}`);
    }
  }

  // 只保留最近 3 轮对话的精简版本
  const recentMessages = messages.slice(-6); // 3 轮 = 6 条消息
  if (recentMessages.length > 0) {
    parts.push("## 最近对话");
    for (const msg of recentMessages) {
      const role = msg.role === "user" ? "用户" : "助手";
      parts.push(`${role}: ${msg.content.slice(0, 100)}...`); // 截断长消息
    }
  }

  return parts.join("\n\n");
}

// 在 Step 中使用摘要
{
  id: "collect",
  type: "chat",
  userPromptTemplate: (runtime) => {
    const summary = summarizeConversation(
      runtime.context.messages as ChatMessage[],
      runtime.stepResults,
    );
    return `对话摘要：\n${summary}\n\n请基于以上信息继续与用户对话，收集缺失的简历信息。`;
  },
}
```

### 关键指标：Token 效率

优化前后对比：

| 场景 | 优化前 Token | 优化后 Token | 节省 |
|------|-------------|-------------|------|
| 第 1 轮 | 200 | 200 | 0% |
| 第 4 轮 | 800 | 450 | 44% |
| 第 8 轮 | 1600 | 600 | 63% |
| 第 12 轮 | 2400 | 700 | 71% |

随着对话延长，节省效果越来越明显。

## 7.4 代码解析：上下文工程的三个支柱

### 支柱 1：信息过滤

不是"把所有东西都给 LLM"，而是"LLM 需要什么给什么"。判断标准：

- 当前步骤需要这个信息来完成职责吗？→ 是 → 注入
- 不需要 → 不注入

`classify` 步骤只需要用户最后一条输入，不需要全部对话历史。
`generate` 步骤需要已收集的结构化信息，不需要每轮对话的原文。

### 支柱 2：格式选择

信息以什么格式注入 LLM？

| 格式 | 适用场景 |
|------|---------|
| 自然语言摘要 | LLM 需要"理解上下文氛围"时 |
| JSON 结构化数据 | LLM 需要精确使用字段值时 |
| 原文引用 | LLM 需要分析原文措辞时 |

在 resumate 中，`collect` 步骤用自然语言摘要，`generate` 步骤用 JSON 结构化数据。

### 支柱 3：时机策略

上下文在何时注入？

- **预加载（Pre-load）**：在 Plan 开始前一次性注入所有上下文 → 简单但浪费
- **渐进加载（Progressive）**：每个 Step 只注入它需要的上下文 → resumate 采用的方式
- **按需加载（On-demand）**：Step 执行过程中，LLM 通过工具调用来检索 → 最灵活但需要模型支持

## 7.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "给 LLM 越多上下文越好" | 更多上下文 ≠ 更好结果。信噪比下降，LLM 注意力被稀释 |
| "上下文工程就是截断历史" | 截断是最粗糙的手段。好的上下文工程是信息重组和结构化 |
| "RuntimeValue 只是延迟求值" | 它是上下文注入的架构机制，影响整个 Plan 的执行策略 |

### 练习

1. **（★☆☆）** 为 resumate 实现一个 `contextWindowBudget` 工具：在注入前计算当前 prompt 的预估 token 数，如果超过阈值则触发摘要。

2. **（★★☆）** 实现分层上下文：将用户信息按"核心/重要/补充"分级。每一步只注入所需级别。

3. **（★★★）** 研究 Anthropic 的 prompt caching 机制。设计一个利用缓存的上下文策略：不变的 system prompt 和变化的 step prompt 如何分离以最大化缓存命中。

## 本章小结

- **Context Engineering** = Information Diet（信息节食）
- **RuntimeValue** 是上下文注入的架构机制：按需计算，不预计算
- **三层策略**：结构化状态 + 对话摘要 + 最近 N 轮 = 对抗 Context Rot

下一章：**提示词工程**——如何与模型高效对话。

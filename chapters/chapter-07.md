# 第7章：上下文工程 — 信息管理的艺术

> **核心概念：** Context Engineering，RuntimeValue，Context Rot，渐进式加载  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 45 分钟  
> **学习目标：** 读完本章后，你将能够：
> 1. 识别 Context Rot 的四种症状和对策
> 2. 实现对话摘要和分层上下文管理
> 3. 利用 Prompt Caching 优化 Token 成本  

---

## 7.1 为什么需要上下文工程？

Context Engineering 是 Harness 的"元能力"——它不直接执行操作，而是管理**进入 LLM 的信息**。（这一概念在 2025 年由多位 AI 工程师独立提出，参见 Andrej Karpathy、Swyx 等人的讨论。）

核心公式：

> **Context Window Management = Information Diet（信息节食）**

就像人吃太多垃圾食品会变迟钝，LLM 被喂入过多无用信息也会变笨。

核心策略：

| 策略 | 解决的问题 |
|------|-----------|
| **压缩 (Compression)** | 长对话摘要，减少 Token |
| **卸载 (Offloading)** | 把结构化数据存到外部，只传引用 |
| **渐进加载 (Progressive Loading)** | 按需注入，不一次性塞入 |

---

## 7.2 什么是上下文工程？

### RuntimeValue：按需注入的上下文机制

第 2 章介绍了 `RuntimeValue<T>`——Step 的参数可以是静态值或动态函数。在上下文工程的语境下，这个机制有了更深的意义：**它是控制"什么信息在什么时机进入 LLM"的架构手段**。

```typescript
// 第 2 章的类型定义回顾
type RuntimeValue<T> = T | ((runtime: StepRuntime) => T);
```

上下文工程的关键原则：**只注入当前步骤需要的信息，不注入"全部历史"**。

```typescript
userPromptTemplate: (runtime) => {
  // 只注入当前步骤需要的上下文片段
  const intent = runtime.stepResults.classify?.intent;
  const userInfo = runtime.stepResults.collect?.text;
  // 不注入 validate 的结果（当前步骤不需要它）
  return `意图：${intent}\n用户信息：${userInfo}`;
}
```

`classify` 步骤只需要用户最后一条输入，不需要全部对话历史。`generate` 步骤需要已收集的结构化信息，不需要每轮对话的原文。RuntimeValue 让这个"按需注入"在架构层面成为可能。

### Context Rot 的四种表现

| 症状 | 原因 | 对策 |
|------|------|------|
| LLM "忘记"早期信息 | 窗口前半部分被后文挤压 | 结构化存储早期信息，显式注入 |
| LLM 重复提问 | 早期答案被淹没 | 对话摘要 + 已收集信息清单 |
| LLM 回答质量下降 | 信噪比太低 | 压缩旧轮次 + 清理无效消息 |
| Token 浪费 | 冗余信息反复传输 | 摘要替代全文 |

### ❌ 失败案例：把所有历史塞给 LLM 的代价

```typescript
// ❌ 反模式：不加过滤，把所有对话历史一次塞给 LLM
async function runAgent(userInput: string) {
  const allMessages = chatHistory.getAll(); // 50 轮对话，~12000 tokens
  const prompt = `
你是求职顾问。

对话历史：
${allMessages.map(m => `${m.role}: ${m.content}`).join("\n")}

用户新输入：${userInput}
`;
  // 问题：
  // 1. 信噪比极低——LLM 要在 50 轮历史中找相关信息
  // 2. 早期信息被淹没——第 1 轮用户说的"我是前端工程师"在第 50 轮几乎不可见
  // 3. Token 浪费——12000 tokens 中 80% 是无关历史
  // 4. Context Rot——LLM 开始"忘记"早期关键信息，回答质量下降

  return await llm.chat([{ role: "user", content: prompt }]);
}
```

对比正例（使用三层策略）：

```typescript
// ✅ 正例：只注入当前步骤需要的信息
async function runAgent(userInput: string) {
  const summary = summarizeConversation(chatHistory.getRecent(6));  // 最近 3 轮
  const structuredState = extractCollectedInfo(chatHistory.getAll()); // 结构化提取

  const prompt = `
你是求职顾问。

已收集的用户信息（结构化）：
${JSON.stringify(structuredState)}

最近对话摘要：
${summary}

用户新输入：${userInput}
`;
  // 优势：
  // 1. Token 从 12000 降至 ~600
  // 2. 关键信息以结构化形式显式注入，不会被淹没
  // 3. 最近对话保持连贯性，不丢失上下文

  return await llm.chat([{ role: "user", content: prompt }]);
}
```

这个正反对比揭示了上下文工程的本质：**不是"给 LLM 越多越好"，而是"精确控制 LLM 看到什么"。**

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

---

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

> *数据来源：resumate 开发过程中，使用 Claude API 在 5 组模拟对话上实测的平均值，Token 计数口径为 API 返回的 `usage.input_tokens`。*

随着对话延长，节省效果越来越明显。

---

## 7.4 代码解析：上下文工程的实施维度

### 维度 1：信息过滤

不是"把所有东西都给 LLM"，而是"LLM 需要什么给什么"。判断标准：

- 当前步骤需要这个信息来完成职责吗？→ 是 → 注入
- 不需要 → 不注入

`classify` 步骤只需要用户最后一条输入，不需要全部对话历史。
`generate` 步骤需要已收集的结构化信息，不需要每轮对话的原文。

### 维度 2：格式选择

信息以什么格式注入 LLM？

| 格式 | 适用场景 |
|------|---------|
| 自然语言摘要 | LLM 需要"理解上下文氛围"时 |
| JSON 结构化数据 | LLM 需要精确使用字段值时 |
| 原文引用 | LLM 需要分析原文措辞时 |

在 resumate 中，`collect` 步骤用自然语言摘要，`generate` 步骤用 JSON 结构化数据。

### 维度 3：时机策略

上下文在何时注入？

- **预加载（Pre-load）**：在 Plan 开始前一次性注入所有上下文 → 简单但浪费
- **渐进加载（Progressive）**：每个 Step 只注入它需要的上下文 → resumate 采用的方式
- **按需加载（On-demand）**：Step 执行过程中，LLM 通过工具调用来检索 → 最灵活但需要模型支持

### Prompt Caching：减少重复计算的成本

Anthropic 在 2024 年推出了 **Prompt Caching** 功能：如果多次请求中 system prompt 的前缀相同，API 会缓存这部分的计算结果，后续请求可以跳过这部分的 token 处理，大幅降低成本和延迟。

这对上下文工程有直接的架构影响。设计 prompt 时的原则：**不变的内容放前面（被缓存），变化的内容放后面（每次不同）**。

```typescript
// 好的 prompt 结构（最大化缓存命中）
const systemPrompt = [
  // 不变的部分（被缓存）
  "你是求职顾问...（角色定义）",
  "你必须遵守以下规则...（通用约束）",
  // 变化的部分（每次不同，不缓存）
  `当前步骤：${stepId}`,
  `用户数据：${JSON.stringify(context)}`,
].join("\n\n");
```

在 resumate 的 Plan 中，Plan-level 的 system prompt 在所有 Step 间共享，天然适合被缓存。Step-level 的 prompt 因步骤而异，放在末尾。这个分层设计在第 8 章会进一步展开。

---

## 7.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "给 LLM 越多上下文越好" | 更多上下文 ≠ 更好结果。信噪比下降，LLM 注意力被稀释 |
| "上下文工程就是截断历史" | 截断是最粗糙的手段。好的上下文工程是信息重组和结构化 |
| "RuntimeValue 只是延迟求值" | 它是上下文注入的架构机制，影响整个 Plan 的执行策略 |

---

## 本章小结

- **Context Engineering** = Information Diet（信息节食）
- **RuntimeValue** 是上下文注入的架构机制：按需计算，不预计算
- **三层策略**：结构化状态 + 对话摘要 + 最近 N 轮 = 对抗 Context Rot

下一章：**提示词工程**——如何与模型高效对话。

### 自检问题

1. Context Rot 的四种症状是什么？各用什么策略应对？
2. resumate 的三层上下文策略（结构化状态 + 对话摘要 + 最近 N 轮）中，每一层各自解决什么问题？
3. Prompt Caching 对上下文工程的设计有什么影响？为什么"不变的内容放前面，变化的内容放后面"？

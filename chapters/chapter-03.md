# 第3章：LLM 抽象层 — 为什么需要 Provider 接口

> **核心概念：** LLMProvider 接口，依赖反转，模型无关性  
> **难度：** ★★☆☆☆  
> **预计阅读时间：** 40 分钟  

## 开篇故事

resumate 刚发布时，我只支持 Anthropic Claude。用户在 GitHub 提了个 issue：

> "能支持 DeepSeek 吗？我在国内用 Claude 不太方便。"

我想了想，好像不难——把 Anthropic SDK 换成 OpenAI 兼容的 SDK 就行了？等等，如果又一个用户要用 Ollama 本地模型呢？

如果我把业务逻辑直接绑在某个模型上：

```typescript
// 坏做法：业务逻辑和模型强耦合
async function generateResume(userInput: string): Promise<Resume> {
  const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    messages: [{ role: "user", content: userInput }],
    // ... Anthropic 特有的参数
  });
  // 用 Anthropic 特有格式解析
  return parseAnthropicResponse(response);
}
```

每加一个新模型，就要改 `generateResume` 的代码。如果有 10 个函数都在调 LLM，就要在 10 个地方写 if-else 判断模型类型。这就是**紧耦合的代价**。

解决方案是引入一层抽象——**LLMProvider 接口**。

## 3.1 为什么需要 LLM 抽象？

### 三个促使你抽象的信号

当你看到以下任何一种情况，就该抽象模型调用了：

**信号 1：if-else 判断模型类型**

```typescript
if (model === "claude") {
  response = await callAnthropic(prompt);
} else if (model === "deepseek") {
  response = await callOpenAICompatible(prompt, "https://api.deepseek.com");
} else if (model === "ollama") {
  response = await callOllama(prompt);
}
```

**信号 2：硬编码模型特有参数**

```typescript
// Claude 的 system prompt 是顶级参数
const claudeRes = await anthropic.messages.create({
  system: "你是求职顾问",
  messages: [...],
});

// OpenAI 的 system prompt 在 messages 数组里
const gptRes = await openai.chat.completions.create({
  messages: [{ role: "system", content: "你是求职顾问" }, ...],
});
```

**信号 3：测试困难**

如果你想测试简历生成逻辑，但每次都要真的调 LLM API（耗时、花钱、不可重复）。

### 解决方案：依赖反转

依赖反转原则（Dependency Inversion Principle）：**高层模块不应该依赖低层模块，两者都应该依赖抽象**。

在这里：
- 高层模块 = Agent 业务逻辑（AgentRunner）
- 低层模块 = 具体 LLM 实现（Anthropic SDK、OpenAI SDK）
- 抽象 = LLMProvider 接口

```
AgentRunner → 依赖 → LLMProvider（接口）
                         ↑
           ┌─────────────┼─────────────┐
           │             │             │
    AnthropicProvider  OpenAICompat  OllamaProvider
```

AgentRunner 只知道它有一个能"流式对话"和"生成结构化输出"的对象，完全不需要知道这个对象背后是 Claude 还是 DeepSeek 还是 Ollama。

## 3.2 什么是 LLMProvider 接口？

### 接口定义

resumate 的 LLMProvider 接口只有两个方法（`packages/agent-harness/src/llm/types.ts`）：

```typescript
export interface LLMProvider {
  /** 流式对话：逐 token 回调，适合聊天场景 */
  streamChat(params: ChatParams, onChunk: StreamingCallback): Promise<string>;

  /** 结构化输出：根据 Zod Schema 生成类型安全的 JSON */
  generateStructured<T>(params: StructuredParams): Promise<T>;
}
```

为什么只有两个方法？因为 Agent 对 LLM 的需求其实只有两种：

1. **聊天**：流式文本输出 → `streamChat`
2. **结构化数据**：符合 Schema 的 JSON → `generateStructured`

这体现了接口设计的**最小化原则**：不暴露模型特有的功能（如 Anthropic 的 tool_use、OpenAI 的 function calling），只暴露 Agent 业务逻辑真正需要的。

### 输入输出类型

```typescript
interface ChatMessage {
  role: "system" | "user" | "assistant";
  content: string;
}

interface StreamChunk {
  type: "text" | "reasoning";  // "reasoning" 用于 DeepSeek/Claude 的思考模式
  content: string;
}

interface ChatParams {
  messages: ChatMessage[];
  temperature?: number;
  thinking?: { type: "enabled" | "disabled" };  // 思考模式开关
}

interface StructuredParams {
  messages: ChatMessage[];
  schema: z.ZodType<unknown>;  // Zod Schema → JSON Schema → 注入 prompt
  temperature?: number;
  thinking?: { type: "enabled" | "disabled" };
}

type StreamingCallback = (chunk: StreamChunk) => void;
```

### 设计决策分析

**决策 1：为什么统一用 `ChatMessage[]` 而非各模型的原生消息格式？**

因为 Anthropic 的 system prompt 是顶级参数，OpenAI 的 system prompt 在 messages 数组里。如果接口暴露这个差异，调用方就要做适配。LLMProvider 接口使用统一格式，由各实现内部处理差异。

**决策 2：为什么 `streamChat` 用回调而非返回 `AsyncGenerator`？**

这是一个有趣的权衡。resumate 的选择是回调模式：

```typescript
// 回调模式：Provider 内部处理流式细节
let fullText = "";
await provider.streamChat({ messages }, (chunk) => {
  if (chunk.type === "text") {
    fullText += chunk.content;
    yield { type: "step:chunk", stepId: step.id, text: chunk.content };
  }
});
```

vs AsyncGenerator 模式：

```typescript
// Generator 模式：调用方自己迭代
for await (const chunk of provider.streamChat({ messages })) {
  yield { type: "step:chunk", text: chunk.content };
}
```

回调模式的优势是：Provider 实现可以在回调内做后处理（如过滤空 chunk、合并相邻 reasoning），调用方只是被动接收处理后的数据。但这两种模式各有优劣，你可以根据自己的偏好调整。

**决策 3：为什么 `thinking` 字段用 `{ type: "enabled" }` 而非 `boolean`？**

为了未来扩展。`{ type: "disabled" }` 明确禁用；`{ type: "enabled" }` 启用；未来可能有 `{ type: "auto", budget: 2048 }` 这样的细粒度控制。

## 3.3 动手实践：实现多个 Provider

### AnthropicProvider

resumate 的 Anthropic 实现（`packages/agent-harness/src/llm/anthropic.ts`）：

```typescript
import Anthropic from "@anthropic-ai/sdk";
import type { LLMProvider, ChatParams, StructuredParams, StreamingCallback } from "./types";

export function createAnthropicProvider(apiKey: string): LLMProvider {
  const client = new Anthropic({ apiKey });

  return {
    async streamChat(params: ChatParams, onChunk: StreamingCallback): Promise<string> {
      const stream = client.messages.stream({
        model: "claude-sonnet-4-6",
        max_tokens: 4096,
        system: extractSystem(params.messages),  // 取出 system 消息
        messages: extractNonSystem(params.messages),
      });

      let fullText = "";
      for await (const event of stream) {
        if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
          fullText += event.delta.text;
          onChunk({ type: "text", content: event.delta.text });
        }
      }
      return fullText;
    },

    async generateStructured<T>(params: StructuredParams): Promise<T> {
      // 将 Zod Schema 注入 system prompt
      const schemaDescription = JSON.stringify(
        zodToJsonSchema(params.schema), null, 2
      );
      const systemMsg = params.messages.find(m => m.role === "system");
      const schemaPrompt = `${systemMsg?.content ?? ""}\n\n
你必须严格按照以下 JSON Schema 输出（只输出 JSON，不要其他内容）：\n${schemaDescription}`;

      const response = await client.messages.create({
        model: "claude-sonnet-4-6",
        max_tokens: 4096,
        system: schemaPrompt,
        messages: extractNonSystem(params.messages),
      });

      // 解析 JSON → Zod 验证 → 类型安全返回
      const text = response.content[0].type === "text"
        ? response.content[0].text : "";
      const json = JSON.parse(extractJSON(text));
      return params.schema.parse(json) as T;
    },
  };
}
```

### OpenAICompatProvider — 支持 DeepSeek + Ollama

```typescript
export function createOpenAICompatProvider(
  apiKey: string,
  baseURL: string = "https://api.deepseek.com",
  model: string = "deepseek-chat",
): LLMProvider {
  return {
    async streamChat(params, onChunk) {
      const response = await fetch(`${baseURL}/v1/chat/completions`, {
        headers: {
          "Authorization": `Bearer ${apiKey}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          model,
          messages: params.messages,  // 注意：不需要拆分 system
          stream: true,
          ...(params.thinking?.type === "enabled" && {
            // DeepSeek 特有的 thinking 参数
            extra_body: { thinking: { type: "enabled" } }
          }),
        }),
      });

      let fullText = "";
      for await (const chunk of parseSSEStream(response)) {
        const delta = chunk.choices?.[0]?.delta;
        if (delta?.content) {
          fullText += delta.content;
          onChunk({ type: "text", content: delta.content });
        }
        // 处理 reasoning（DeepSeek 的思考过程）
        if (delta?.reasoning_content) {
          onChunk({ type: "reasoning", content: delta.reasoning_content });
        }
      }
      return fullText;
    },

    async generateStructured<T>(params): Promise<T> {
      // 类似 streamChat，但非流式 + JSON 解析
      const schemaJSON = zodToJsonSchema(params.schema);
      const response = await fetch(`${baseURL}/v1/chat/completions`, {
        headers: { /* ... */ },
        body: JSON.stringify({
          model,
          messages: [
            ...params.messages,
            { role: "system", content: `输出格式：\n${JSON.stringify(schemaJSON)}` }
          ],
          response_format: { type: "json_object" },  // OpenAI 兼容的 JSON 模式
        }),
      });
      const data = await response.json();
      const json = JSON.parse(data.choices[0].message.content);
      return params.schema.parse(json) as T;
    },
  };
}
```

### 使用：AgentRunner 不感知模型差异

```typescript
// 用户选择 Claude
const claudeRunner = new AgentRunner(
  createAnthropicProvider("sk-ant-..."),
  createBuiltInTools()
);

// 用户选择 DeepSeek
const deepseekRunner = new AgentRunner(
  createOpenAICompatProvider("sk-ds-...", "https://api.deepseek.com", "deepseek-chat"),
  createBuiltInTools()
);

// 用户选择本地 Ollama
const ollamaRunner = new AgentRunner(
  createOpenAICompatProvider("ollama", "http://localhost:11434", "llama3.1"),
  createBuiltInTools()
);

// 三者使用完全相同的 Plan 和 Tool，AgentRunner 不需要任何修改！
const plan = resumeGenerationPlan;
for await (const event of claudeRunner.execute(plan)) { /* ... */ }
for await (const event of deepseekRunner.execute(plan)) { /* ... */ }
for await (const event of ollamaRunner.execute(plan)) { /* ... */ }
```

这就是 LLM 抽象层的价值：**加一个新模型，只写一个 Provider 实现，不需要改任何业务代码**。

## 3.4 代码解析：四层架构

resumate 的 LLM 层整体架构：

```
┌─────────────────────────────────────┐
│ AgentRunner                         │  ← 业务层：只知道 LLMProvider 接口
│  runner.execute(plan)               │
├─────────────────────────────────────┤
│ LLMProvider (interface)             │  ← 抽象层：两个方法
│  streamChat() / generateStructured()│
├─────────────────────────────────────┤
│ AnthropicProvider | OpenAICompat     │  ← 适配层：把接口翻译成具体 API 调用
│  调用 Anthropic SDK | 调用 fetch()  │
├─────────────────────────────────────┤
│ Anthropic API | DeepSeek | Ollama   │  ← 模型层：真正执行推理的地方
└─────────────────────────────────────┘
```

关键设计原则：

1. **接口最小化**：只有 Agent 真正需要的方法
2. **类型安全**：`generateStructured<T>` 用泛型和 Zod 保证输出类型
3. **实现细节内聚**：Anthropic 的 system prompt 独立参数 vs OpenAI 的 messages 内 system——差异被封装在适配层内部
4. **Think Mode 统一化**：`{ type: "enabled" }` 统一了不同模型的"思考"功能

## 3.5 复盘与延伸

### 本章要点回顾

1. **LLMProvider 接口**让 Agent 与具体模型解耦，实现模型无关性。

2. **两个方法**覆盖了 Agent 的全部 LLM 需求：`streamChat`（流式对话）和 `generateStructured`（结构化输出）。

3. **适配层模式**：每个模型一个 Provider 实现，封装 API 差异。

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "接口要设计成能覆盖所有模型的所有功能" | 不要。接口只暴露业务需要的功能。模型特有的高级功能（如 tool_use）由 Provider 内部自行处理 |
| "多 Provider 就得处理所有差异" | 不需要。80% 的差异在于请求/响应的序列化格式。适配层就是做这件事的 |
| "Zod Schema 注入 prompt 不够精确" | 对复杂度不高的 Schema，prompt 注入是足够的。如果你需要严格的 constrained decoding（如 JSON 语法级约束），那是模型层面的功能，不是 Provider 接口的职责 |

### 练习

1. **（★☆☆）** 为 resumate 添加一个新的 Provider 实现（如 Google Gemini）。不需要完整实现，写出类结构和方法签名即可。

2. **（★★☆）** 在 `generateStructured` 中添加自动重试：如果 Zod parse 失败，把 parse error 反馈给 LLM 再试一次。

3. **（★★★）** 实现一个 `MockProvider`：不调真实 API，返回预定义数据。用于 AgentRunner 的单元测试。

---

> 📍 **进度检查点**：至此，你已经完成了第一部分（基础篇）的全部内容。回顾一下你学到了什么：
> - Ch1：Agent = Model + Harness 公式和六大组件全景
> - Ch2：AgentRunner 执行循环——Plan/Step 模型、四种步骤类型、事件驱动
> - Ch3：LLMProvider 接口——模型无关性、依赖反转、多 Provider 适配
>
> 你现在可以从零构建一个具有多步骤执行能力和多模型支持的最简 Agent。在第二部分，我们将为它装上"手脚"。

## 本章小结

这一章我们学习了 LLM 抽象层：

- **为什么**：避免业务代码与模型 SDK 紧耦合
- **是什么**：LLMProvider 接口（两个方法）+ 适配层模式
- **怎么做**：AnthropicProvider + OpenAICompatProvider 的实现

下一章，我们进入 Agent 的"手脚"——**工具系统**，理解 Agent 如何执行确定性操作。

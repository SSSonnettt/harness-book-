# 第6章：流式响应 — SSE 与实时反馈

> **核心概念：** SSE (Server-Sent Events)，AsyncGenerator，HarnessEvent 生命周期  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 40 分钟  
> **学习目标：** 读完本章后，你将能够：
> 1. 判断何时使用 SSE 而非 WebSocket
> 2. 将 AsyncGenerator 事件流转换为 HTTP SSE 响应
> 3. 实现前端 SSE 事件消费和进度展示  

---

## 6.1 为什么需要流式响应？

### 阻塞 API 的三个问题

| 问题 | 影响 |
|------|------|
| **用户感知延迟** | 30 秒白屏 → 用户以为坏了 → 离开页面 |
| **无法提前纠错** | 如果第 2 步的意图分类就错了，要等 30 秒才知道 |
| **资源浪费** | 即使用户关闭了页面，后端仍在执行完整的 5 步 pipeline |

### SSE vs WebSocket

既然要实时推送，为什么选 SSE 而不是 WebSocket？

| 特性 | SSE | WebSocket |
|------|-----|-----------|
| 方向 | 单向（服务器→客户端） | 双向 |
| 协议 | HTTP/1.1，无需升级 | 需要协议升级 |
| 断线重连 | 浏览器自动重连 | 需要手动实现 |
| 复杂度 | 极简（`text/event-stream`） | 需要心跳、重连逻辑 |
| Agent 场景适合？ | ✅ Agent 只需推送状态 | ❌ 不需要客户端主动发消息 |

对于 Agent 的"执行状态推送"场景，SSE 是更简单的选择。

---

## 6.2 什么是 Harness 的事件系统？

### HarnessEvent 类型

resumate 定义了 8 种事件（`packages/shared/src/resume.ts`）：

```typescript
type HarnessEvent =
  | { type: "plan:start";      planId: string }
  | { type: "step:start";      stepId: string; description: string }
  | { type: "step:chunk";      stepId: string; text: string }
  | { type: "step:tool_call";  stepId: string; tool: string; args: unknown }
  | { type: "step:done";       stepId: string; result: unknown }
  | { type: "plan:done";       resume: Resume }
  | { type: "plan:error";      stepId: string; error: string }
  | { type: "reasoning:chunk"; stepId: string; text: string };  // 思考过程
```

这些事件可以分为三层：

```
计划层级  → plan:start / plan:done / plan:error
步骤层级  → step:start / step:done
内容层级  → step:chunk / step:tool_call / reasoning:chunk
```

### AsyncGenerator → SSE

AgentRunner 用 AsyncGenerator 产出事件，Next.js Route Handler 将其转为 SSE：

```typescript
// packages/web/app/api/agent/run/route.ts
export async function POST(request: Request) {
  const { messages, apiKey } = await request.json();

  const provider = createAnthropicProvider(apiKey);
  const runner = new AgentRunner(provider, createBuiltInTools());

  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder();

      try {
        for await (const event of runner.execute(resumeGenerationPlan, {
          messages,
        })) {
          // 每个 HarnessEvent → SSE data 行
          const data = `data: ${JSON.stringify(event)}\n\n`;
          controller.enqueue(encoder.encode(data));
        }
      } catch (err) {
        controller.enqueue(encoder.encode(
          `data: ${JSON.stringify({
            type: "plan:error",
            error: err instanceof Error ? err.message : String(err),
          })}\n\n`
        ));
      }
      controller.close();
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      "Connection": "keep-alive",
    },
  });
}
```

### 客户端消费

客户端的核心逻辑很简洁：打开 SSE 流 → 逐行解析 → 按事件类型更新 UI：

```typescript
async function runAgent(messages: ChatMessage[]) {
  const response = await fetch("/api/agent/run", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ messages, apiKey }),
  });

  // 核心：逐行解析 SSE 事件
  for await (const event of parseSSEEvents(response.body)) {
    handleEvent(event);  // 根据 event.type 分发到不同的 UI 更新
  }
}
```

`parseSSEEvents` 是一个通用的 SSE 流解析器，负责处理 `ReadableStream` 的分块读取和 `data:` 行解析。`handleEvent` 是一个 switch/case，根据事件类型调用对应的 UI 更新函数（如 `showProgress`、`appendToChat`、`showFinalResume`）。这两部分都是标准的 Web API 用法，完整实现见 [resumate 源码](https://github.com/SSSonnettt/resumate)。

---

## 6.3 动手实践：在 resumate 中添加进度指示

resumate 的 `chat-store.ts` 存储了所有 `harnessEvents`。基于这些数据，前端可以根据事件类型渲染不同的状态图标：

- `step:start` 且尚未收到 `step:done` → 显示旋转加载图标 + 步骤描述
- `step:done` → 显示绿色对勾
- `plan:error` → 显示红色叉号

这是一个纯 React 组件的工作——遍历 `harnessEvents` 数组，用 Map 记录每个步骤的最新状态，然后渲染列表。核心逻辑不到 20 行，不涉及任何 Harness 概念，因此这里不展开代码（完整实现见 [resumate 源码](https://github.com/SSSonnettt/resumate) 的 `AgentProgressBar` 组件）。

值得注意的**设计思路**：进度指示的数据来源是 HarnessEvent 流，而不是 AgentRunner 的直接 API。前端不需要知道 AgentRunner 内部怎么工作，只需要消费事件流就能还原出完整的执行状态。这是事件驱动架构的优势——**生产者与消费者完全解耦**。

---

## 6.4 代码解析：事件驱动架构在 Agent 系统中的价值

### 事件驱动的通用价值

HarnessEvent 系统不只是"给前端推状态"——它体现了**事件驱动架构（Event-Driven Architecture）**的核心思想：**通过事件流解耦系统的各个组件**。

在 Agent 系统中，事件驱动的价值远不止 UI 展示：

| 消费者 | 用途 |
|--------|------|
| **前端 UI** | 实时进度条、流式文本显示 |
| **日志系统** | 持久化事件流 → 事后分析和调试 |
| **监控系统** | 聚合多轮运行的延迟、错误率、Token 趋势 |
| **回放系统** | 重播事件流 → 复现 bug、对比不同版本 |
| **审计系统** | 记录 Agent 的每一次决策和操作 |

所有这些都是事件的**消费者**，它们不需要修改 AgentRunner 的代码，也不需要了解 Agent 的内部实现。只要 HarnessEvent 的格式保持稳定，你可以随时添加新的消费者。

这种"生产一次，消费多次"的模式，在 Kafka、RabbitMQ 等消息系统中很常见。Agent 的 AsyncGenerator 本质上就是一个轻量的事件总线。

### "push" 优于 "poll"

传统方案：前端每 2 秒 `GET /api/status?jobId=xxx` 查询状态。问题是：
- 客户端不知道什么时候有新状态，轮询间隔太短浪费资源，太长响应慢
- 服务端需要维护 job 状态（Redis、数据库），增加复杂度

SSE 方案：服务端主动推送，客户端被动接收。零额外状态维护。

### 事件粒度选择

为什么选这些事件而不是更多或更少？

- **太少**：如果只有 `plan:start` 和 `plan:done` → 回到黑盒问题
- **太多**：如果每个 token 都发独立事件类型 → 前端处理复杂，网络开销大

当前粒度刚好：1 个 plan 级 + N 个 step 级。每个 step 内部可以有多个 chunk。

### 错误传播

`plan:error` 在两种情况下触发：
1. `dependsOn` 检查失败（依赖步骤未完成）
2. Step 执行中抛出异常

错误事件一旦发送，AgentRunner 就停止执行。这是一种"快速失败"策略——一个步骤错了，继续执行没有意义。

---

## 6.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "SSE 和 WebSocket 差不多" | SSE 是单向推送且简单，WebSocket 是双向且有更多复杂性 |
| "流式响应用于所有 API" | 不。只有长时间运行、需要实时反馈的操作才需要流式 |
| "事件越多越详细越好" | 每个事件类型都增加客户端的处理复杂度，够用即可 |

---

## 本章小结

- **SSE** 是 Agent 的"指示灯面板"——每一步状态实时可见
- **HarnessEvent** 8 种事件类型，计划层/步骤层/内容层三层结构
- **AsyncGenerator → SSE**：一个 `for await...of` + `ReadableStream` 实现

---

> 📍 **进度检查点**：至此，你已完成本书第一部分和第二部分。resumate 已经从最简 30 行 Agent 演进到一个具有 Plan/Step 执行循环、LLM 抽象层、工具系统和流式响应的完整 Agent。下一章起，我们进入第三部分——让 Agent 更"聪明"。

### 自检问题

1. Agent 的"执行状态推送"场景，为什么 SSE 比 WebSocket 更适合？
2. HarnessEvent 的 8 种事件类型分为哪三层？每层的职责是什么？
3. 事件驱动架构在 Agent 系统中的五个典型消费者是什么？为什么"生产一次，消费多次"是比"为每个消费者单独 API"更好的设计？

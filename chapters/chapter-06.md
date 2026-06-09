# 第6章：流式响应 — SSE 与实时反馈

> **核心概念：** SSE (Server-Sent Events)，AsyncGenerator，HarnessEvent 生命周期  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 40 分钟  

## 开篇故事

resumate 的第一个版本，我把整个 Agent 运行放在一个 API 端点里：

```typescript
// POST /api/generate — 阻塞 30 秒后一次性返回
app.post("/api/generate", async (req, res) => {
  const result = await agentRunner.runSync(req.body.input);
  res.json(result);
});
```

用户在浏览器里点"生成"，然后看 30 秒白屏。不知道 Agent 在干什么、进行到哪一步、是不是卡死了。30 秒后突然弹出一份简历。

这体验？糟糕透了。

修改方式：把 AgentRunner 的执行过程改为**实时流式传输**——每一步都通过 SSE (Server-Sent Events) 推送给前端。用户看到：
- "正在分析你的意图..."
- "正在收集职业信息..."（对话逐字显示）
- "正在生成简历..."
- "正在验证简历..." ✅ 姓名：张三 ✅ 模块：6 个
- "简历生成完毕！"

从"黑盒 30 秒"变成了"透明过程"。这就是 SSE 的力量。

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

## 6.2 什么是 Harness 的事件系统？

### HarnessEvent 类型

resumate 定义了 7 种事件（`packages/shared/src/resume.ts`）：

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

```typescript
// 前端：消费 SSE 事件
async function runAgent(messages: ChatMessage[]) {
  const response = await fetch("/api/agent/run", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ messages, apiKey }),
  });

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split("\n");
    buffer = lines.pop() || "";

    for (const line of lines) {
      if (line.startsWith("data: ")) {
        const event: HarnessEvent = JSON.parse(line.slice(6));

        // 根据事件类型更新 UI
        switch (event.type) {
          case "step:start":
            showProgress(event.description);
            break;
          case "step:chunk":
            appendToChat(event.text);
            break;
          case "step:tool_call":
            showToolUse(event.tool);
            break;
          case "step:done":
            markStepComplete(event.stepId);
            break;
          case "plan:done":
            showFinalResume(event.resume);
            break;
          case "plan:error":
            showError(event.error);
            break;
        }
      }
    }
  }
}
```

## 6.3 动手实践：在 resumate 中添加进度指示

resumate 的 `chat-store.ts` 已经存储了 `harnessEvents`。我们可以基于它实现步骤进度条：

```typescript
// 组件：AgentProgressBar
function AgentProgressBar({ events }: { events: HarnessEvent[] }) {
  const steps = useMemo(() => {
    const stepMap = new Map<string, {
      id: string; description: string;
      status: "pending" | "running" | "done" | "error";
    }>();

    for (const event of events) {
      switch (event.type) {
        case "step:start":
          stepMap.set(event.stepId, {
            id: event.stepId,
            description: event.description,
            status: "running",
          });
          break;
        case "step:done":
          stepMap.set(event.stepId, {
            ...stepMap.get(event.stepId)!,
            status: "done",
          });
          break;
        case "plan:error":
          stepMap.set(event.stepId, {
            ...stepMap.get(event.stepId)!,
            status: "error",
          });
          break;
      }
    }
    return Array.from(stepMap.values());
  }, [events]);

  return (
    <div className="space-y-2 p-4">
      {steps.map((step, i) => (
        <div key={step.id} className="flex items-center gap-2">
          {step.status === "done" && <CheckCircle className="text-green-500" />}
          {step.status === "running" && <Loader className="animate-spin text-blue-500" />}
          {step.status === "error" && <XCircle className="text-red-500" />}
          {step.status === "pending" && <Circle className="text-gray-300" />}
          <span className={
            step.status === "running" ? "font-semibold" : ""
          }>
            {step.description}
          </span>
        </div>
      ))}
    </div>
  );
}
```

## 6.4 代码解析：事件系统的设计哲学

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

## 6.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "SSE 和 WebSocket 差不多" | SSE 是单向推送且简单，WebSocket 是双向且有更多复杂性 |
| "流式响应用于所有 API" | 不。只有长时间运行、需要实时反馈的操作才需要流式 |
| "事件越多越详细越好" | 每个事件类型都增加客户端的处理复杂度，够用即可 |

### 练习

1. **（★☆☆）** 在 resumate 中添加一个新事件类型 `step:progress`，用于报告子步骤（如"正在生成工作经历..."）。

2. **（★★☆）** 修改客户端 SSE reader，在连接断开时自动重连。

3. **（★★★）** 实现事件持久化：将所有 SSE 事件存入 IndexedDB，刷新页面后可回放执行过程。

## 本章小结

- **SSE** 是 Agent 的"指示灯面板"——每一步状态实时可见
- **HarnessEvent** 7 种事件类型，计划层/步骤层/内容层三层结构
- **AsyncGenerator → SSE**：一个 `for await...of` + `ReadableStream` 实现

---

> 📍 **进度检查点**：至此，你已完成本书第一部分和第二部分。resumate 已经从最简 30 行 Agent 演进到一个具有 Plan/Step 执行循环、LLM 抽象层、工具系统和流式响应的完整 Agent。下一章起，我们进入第三部分——让 Agent 更"聪明"。

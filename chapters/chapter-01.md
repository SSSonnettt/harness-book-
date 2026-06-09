# 第1章：你好，Agent — 从"调用 API"到"构建 Agent"

> **核心概念：** Agent = Model + Harness  
> **难度：** ★☆☆☆☆  
> **预计阅读时间：** 45 分钟  

---

> 📚 **本书技术栈**  
> 本书使用 **Next.js 15 + TypeScript 5.7 + Anthropic SDK** 作为技术栈，以 **resumate**（一个真实的开源 AI 简历生成器）为贯穿案例。选择这个技术栈的原因是：
> - **Next.js**：全栈框架，前后端一体化，读者不需要同时学 Express + React
> - **TypeScript**：类型安全，LLM 的结构化输出天然需要强类型约束
> - **Anthropic SDK**：支持 reasoning/thinking 模式，对复杂任务有显著提升
> - **pnpm + Turborepo**：monorepo 管理，真实项目标配
>
> 读者需要：TypeScript/React 基础（1-2年经验）、Git 基础、命令行基础。不需要：Agent 框架经验、机器学习知识。
>
> 如果你主要使用 Python，本书的概念和架构设计依然适用——Harness 是语言无关的。所有代码示例都可以用 Python/Go/Rust 等价实现。

## 开篇故事

2025 年秋天，我接了一个"简单"的外包项目：做一个 AI 简历生成器。需求很明确——用户输入经历，AI 生成专业简历，导出 PDF。

"太简单了，"我想，"不就是调个 Anthropic API 嘛。"

30 分钟后，我写出了第一版：

```typescript
const response = await fetch("https://api.anthropic.com/v1/messages", {
  headers: { "x-api-key": apiKey, "anthropic-version": "2023-06-01" },
  body: JSON.stringify({
    model: "claude-sonnet-4-6",
    max_tokens: 4096,
    messages: [{ role: "user", content: "帮我写一份软件工程师的简历" }]
  })
});
const data = await response.json();
console.log(data.content[0].text);
```

确实生成了简历。看起来也不错。我得意地发给朋友测试。

5 分钟后他回消息了：**"我想把工作经历改成 STAR 格式，怎么改？"**

我愣住了。这需要重新调一次 API，但上下文里没有之前的简历——因为每次调用都是全新的、无状态的。

然后是第二个问题：**"我想导出成 PDF。"**

第三个：**"能不能针对这个 JD 优化一下？"**

第四个：**"我刚才生成的简历呢？刷新页面不见了。"**

我突然意识到：**"调 API"和"做产品"之间有一条巨大的鸿沟**。

这条鸿沟，就是 resumate 这个项目要跨越的。这条鸿沟，就是本书要带你理解的 Harness。

## 1.1 为什么需要 Harness？

### 裸模型的四大硬伤

让我们做一个思想实验：如果你只有 `fetch("/v1/messages", ...)`，没有任何其他基础设施，你能做一个 AI 产品吗？

答案是可以——但仅限于"输入文本，输出文本"的一次性场景。一旦你需要：

- **记住上下文**（上一轮用户说了什么？刚才生成的简历在哪？）
- **执行操作**（生成 PDF、读取文件、搜索职位描述）
- **获取实时信息**（今天的职位需求是什么？）
- **多步骤协作**（先分析 JD → 再生成简历 → 再验证 → 再导出）

你会发现，裸模型做不到。具体来说，裸模型有四大硬伤：

| 硬伤 | 表现 | 影响 |
|------|------|------|
| **无法维持跨会话状态** | 每次 API 调用都是全新的，不记得上一轮说了什么 | 没法做多轮对话产品 |
| **无法执行代码** | 只能输出文本，不能运行程序 | 不能生成 PDF、不能读文件、不能调 API |
| **无法获取实时知识** | 知识截止于训练日期 | 不知道今天的 JD、最新的技术栈 |
| **无法搭建工作环境** | 没有工作目录、没有工具链 | 不能管理中间文件、不能版本控制 |

回到我的简历生成器。这四个硬伤分别对应四个真实需求：

1. **跨会话状态** → 用户刷新页面后，简历还在（localStorage 持久化）
2. **执行代码** → 把 Markdown 简历编译成 PDF（html2canvas / 浏览器 print）
3. **实时知识** → 读取用户粘贴的 JD 内容（文本解析 + 关键词提取）
4. **工作环境** → 管理多份简历、支持撤销重做（Immer Patch 状态管理）

这不是简历生成器特有的问题——**任何 AI 产品都面临同样的挑战**。聊天机器人需要记忆对话历史，代码助手需要执行编译命令，数据分析助手需要读写文件。所有这些问题，都指向同一个解决方案。

### 一个公式

2026 年，AI 工程领域形成了一个核心共识。这个共识只有五个字，但它统一了所有 AI 产品的架构设计思想：

> **Agent = Model + Harness**

- **Model（模型）** 是"大脑"——负责理解和推理。它是智能的来源，但不具备行动能力。
- **Harness（驾驭层/挽具）** 是"操作系统"——包裹在模型外面，提供记忆、工具执行、安全护栏、上下文管理、编排调度。

正如 CPU 需要操作系统才能发挥作用，裸模型需要 Harness 才能变成可用的智能体。

> ⚠️ **术语说明**："Harness"一词借自马具中的"挽具"——马本身有力量，但需要挽具来引导方向、控制速度、承载负载。AI 模型也是如此：模型提供智能，Harness 提供工程控制。

从 Prompt Engineering 到 Context Engineering 再到 Harness Engineering，AI 工程的演进史可以概括为：

```
2022-2024: Prompt Engineering     → 优化单次对话的质量
2025:      Context Engineering    → 管理上下文窗口内的信息
2026+:     Harness Engineering    → 构建完整的 Agent 运行时环境
```

你正在学习的，就是这个领域最新的范式。

## 1.2 什么是 Harness？

### 六大核心组件

如果把 Harness 拆开看，它由六个核心组件构成。用一个表格来直观理解：

| 组件 | 一句话概括 | 解决的问题 |
|------|-----------|-----------|
| **文件系统** | Agent 的"工作台" | 中间结果存哪？代码写在哪？ |
| **Bash + 沙箱** | Agent 的"手脚" | 怎么运行代码？怎么编译 PDF？ |
| **记忆系统** | Agent 的"长期记忆" | 刷新页面后数据还在吗？ |
| **搜索 + MCP** | Agent 的"眼睛和耳朵" | 怎么读取外部数据？怎么连接 API？ |
| **上下文工程** | Agent 的"信息过滤器" | 对话长了怎么办？Token 超了怎么办？ |
| **编排 + Hooks** | Agent 的"指挥系统" | 多个步骤怎么协调？质量怎么保证？ |

这六个组件不是孤立的——它们相互依赖，构成一个完整的运行时环境。本书的第 3-11 章会逐一深入讲解每个组件。但现在，让我们先通过 resumate 看到它们的全貌。

### 三层架构

换一个视角，Harness 也可以理解为一个三层架构：

```
┌──────────────────────────────────────┐
│  控制层 (Control)                     │
│  AGENTS.md · 代码规范 · 规则权限      │  ← 静态约束："什么是允许的"
├──────────────────────────────────────┤
│  代理层 (Agency)                      │
│  工具/API · 浏览器 · 多Agent角色      │  ← 行动界面："能做什么"
├──────────────────────────────────────┤
│  运行层 (Runtime)                     │
│  内存管理 · 上下文压缩 · 重试回滚     │  ← 动态管理："如何高效运行"
└──────────────────────────────────────┘
```

- **控制层**：定义 Agent "什么是允许的"。在 resumate 中，这就是 `AGENTS.md` 和 `CLAUDE.md`——它们定义了项目架构原则和代码规范。
- **代理层**：定义 Agent "能做什么"。在 resumate 中，这就是 `ToolRegistry` 中注册的工具函数和 `AgentRunner` 管理的 Plan/Step。
- **运行层**：定义 Agent "如何高效运行"。在 resumate 中，这就是 SSE 流式传输、Zustand 状态管理、Immer 撤销重做。

### resumate 中的 Harness 全景

resumate 是一个真实的、可用的开源 AI 简历生成器。它的 monorepo 结构本身就映射了 Harness 的三层架构：

```
resumate/
├── packages/
│   ├── shared/              # 共享类型（纯类型定义）
│   ├── agent-harness/       # 🤖 Harness 引擎 —— 代理层 + 运行层
│   │   ├── src/
│   │   │   ├── runner.ts         # AgentRunner：执行循环
│   │   │   ├── tool-registry.ts  # ToolRegistry：工具系统
│   │   │   └── llm/              # LLMProvider：模型抽象层
│   │   │       ├── types.ts      #   接口定义
│   │   │       ├── anthropic.ts  #   Anthropic 适配器
│   │   │       └── openai-compat.ts # OpenAI兼容适配器
│   └── web/                 # Next.js 全栈应用 —— 控制层 + 用户界面
│       ├── app/
│       │   ├── page.tsx          # 首页：AI 聊天 + 简历预览
│       │   ├── editor/page.tsx   # 编辑器：三栏布局拖拽编辑
│       │   ├── preview/page.tsx  # 预览导出：PDF/PNG
│       │   └── api/agent/run/    # SSE Agent API
│       ├── lib/
│       │   ├── stores/           # Zustand 状态管理（内存/持久化）
│       │   └── templates/        # 简历模板（JSON → CSS）
│       └── components/
│           ├── chat/             # 聊天面板
│           ├── editor/           # 编辑器组件
│           └── renderers/        # 简历模块渲染器
├── AGENTS.md                # 🎯 控制层：Agent 行为约束合约
├── CLAUDE.md                # 🎯 控制层：项目架构指引
└── turbo.json               # 构建编排
```

来看看 resumate 中 Harness 六大组件的对应关系：

| Harness 组件 | resumate 中的实现 |
|-------------|-----------------|
| **文件系统** | monorepo 工作空间、模板 JSON 文件、localStorage 持久化 |
| **Bash + 沙箱** | `pnpm dev/build` 开发命令、Docker 容器化部署 |
| **记忆系统** | Zustand + Immer 状态管理（undoStack/redoStack）、localStorage 持久化 |
| **搜索 + MCP** | JD 文本解析、Anthropic SDK（作为"外部模型服务"）、未来可扩展 MCP 工具 |
| **上下文工程** | SSE 流式传输、Plan/Step 渐进上下文注入、RuntimeValue 动态计算 |
| **编排 + Hooks** | AgentRunner Plan/Step 执行、SSE 事件生命周期、Turbo 构建编排 |

在接下来的章节中，我们会逐一展开每个组件的实现细节。你会发现：**resumate 的 agent-harness 包本身就是一个微型 Harness 实现**——它只有几百行代码，但完整覆盖了 Harness 的全部核心概念。

## 1.3 动手实践：从零体验 resumate

### 启动项目

确保你的环境满足以下条件：
- Node.js 18+
- pnpm 10+（安装：`npm install -g pnpm`）
- Anthropic API Key（获取：https://console.anthropic.com/）

```bash
cd /path/to/resumate
pnpm install
pnpm dev
```

浏览器访问 `http://localhost:5001`。你会看到 resumate 的主界面——左边是 AI 聊天面板，右边是简历实时预览。

### 完整流程体验

让我们追踪一次完整的简历生成流程：

1. **输入 API Key**：首次使用会弹出 API Key 对话框。Key 存储在浏览器 localStorage 中，不会上传服务器。

2. **开始对话**：输入"你好，我想生成一份前端工程师的简历"。

3. **观察 SSE 事件流**：打开浏览器开发者工具 → Network 标签 → 找到 `/api/agent/run` 请求 → 查看 EventStream。你会看到这样的事件序列：

```
event: plan:start
data: {"planId":"resume-generation"}

event: step:start
data: {"stepId":"classify","description":"分析用户意图"}

event: step:done
data: {"stepId":"classify","result":{"intent":"resume_enhance"}}

event: step:start
data: {"stepId":"collect","description":"收集用户的职业信息"}

event: step:chunk
data: {"stepId":"collect","text":"你好！为了更好地帮你..."}

event: step:done
data: {"stepId":"collect","result":{"text":"..."}}

event: step:start
data: {"stepId":"generate","description":"生成结构化简历"}

event: step:done
data: {"stepId":"generate","result":{...}} // 结构化 JSON

event: step:start
data: {"stepId":"validate","description":"验证简历完整性"}

event: step:tool_call
data: {"stepId":"validate","tool":"validateResume","args":{...}}

event: step:done
data: {"stepId":"validate","result":{"valid":true,"issues":[]}}

event: step:start
data: {"stepId":"present","description":"呈现最终简历"}

event: plan:done
data: {"resume":{...}} // 最终简历数据
```

这就是一个完整的 Agent 执行流程！5 个步骤，每个步骤都有明确的职责和产出。右栏的简历预览会随着事件流实时更新。

4. **进入编辑器**：点击顶部"编辑器"链接，进入三栏布局编辑器——左侧模块面板、中间拖拽画布、右侧样式面板。试试切换模板（经典黑/优雅衬线/蓝色简约）、调整主题色。

5. **导出**：进入"预览导出"页面，点击"导出 PDF"或"导出图片"。

### 数据流全景

把刚才的体验抽象成数据流：

```
用户输入（自然语言）
    │
    ▼
┌─────────────┐    POST /api/agent/run    ┌──────────────────┐
│  ChatPanel  │ ──────────────────────────→│  AgentRunner      │
│  (React)    │ ←── SSE stream ───────────│  .execute(plan)   │
└─────────────┘                           │                   │
      │                                    │  Plan:            │
      │  HarnessEvent                      │    classify       │
      ▼                                    │    → collect      │
┌─────────────┐                           │    → generate     │
│ chat-store  │                           │    → validate     │
│ (Zustand)   │                           │    → present      │
└─────────────┘                           └──────────────────┘
      │                                              │
      │  applyAIResult                               │ LLMProvider
      ▼                                              ▼
┌─────────────┐                           ┌──────────────────┐
│ resume-store│ ←─────────────────────────│ Anthropic /       │
│ (Zustand)   │    structured JSON        │ OpenAI Compat     │
└─────────────┘                           └──────────────────┘
      │
      ▼
┌─────────────┐
│ ResumePreview│  ← 实时渲染简历
│ (React)     │
└─────────────┘
```

这就是 Agent = Model + Harness 在 resumate 中的完整体现：
- **Model** = Anthropic Claude（通过 `AnthropicProvider`）
- **Harness** = AgentRunner + ToolRegistry + Zustand Stores + SSE + React UI + localStorage

## 1.4 代码解析：30 行理解 Agent = Model + Harness

让我们写一个最简 Agent，用 30 行代码体现 Agent = Model + Harness：

```typescript
// 最简 Agent：输入文本 → 调 LLM → 输出文本
// 这已经包含了 Agent 的两个基本要素

// Part 1: Model（模型——"大脑"）
interface LLMProvider {
  chat(messages: { role: string; content: string }[]): Promise<string>;
}

// Part 2: Harness（驾驭层——"操作系统"）
class SimpleAgent {
  constructor(private model: LLMProvider) {}

  async run(userInput: string): Promise<string> {
    // 执行循环（Execution Loop）
    const messages = [
      { role: "system", content: "你是一位专业的中文求职顾问。" },
      { role: "user", content: userInput }
    ];

    // 观察（Observe）：从上下文获取信息 → 思考+行动（Think+Act）：调模型
    const response = await this.model.chat(messages);

    // 返回结果
    return response;
  }
}

// 使用
const anthropicProvider: LLMProvider = {
  async chat(messages) {
    // 这里调用 Anthropic API
    const res = await fetch("https://api.anthropic.com/v1/messages", {
      headers: { "x-api-key": apiKey, "anthropic-version": "2023-06-01" },
      body: JSON.stringify({ model: "claude-sonnet-4-6", max_tokens: 4096, messages })
    });
    const data = await res.json();
    return data.content[0].text;
  }
};

const agent = new SimpleAgent(anthropicProvider);
const result = await agent.run("帮我写一份前端工程师的简历");
console.log(result);
```

这 30 行代码包含了 Agent 的两个基本要素：

1. **Model（LLMProvider）**：提供智能。你可以在不修改 `SimpleAgent` 的前提下，把 `anthropicProvider` 替换成 `openaiProvider` 或 `ollamaProvider`。这就是 Harness 的**模型无关性**——将在第 3 章展开。

2. **Harness（SimpleAgent）**：提供 engineering infrastructure。目前它只有一个执行循环（`run` 方法），但你可以逐步扩展它：
   - 加 `memory: Map<string, string>` → 记忆系统
   - 加 `tools: Map<string, Function>` → 工具系统
   - 加 `async *stream()` → 流式响应
   - 加 `history: Message[]` → 多轮对话

这就是全书的核心方法论：**每一章，我们都在这个最简 Agent 的基础上添加一个新的 Harness 组件**，直到它变成一个完整的、生产可用的 resumate。

### resumate 的真实 AgentRunner

来看看 resumate 中 `AgentRunner` 的实际实现（`packages/agent-harness/src/runner.ts`，核心代码约 160 行）：

```typescript
export class AgentRunner {
  private registry: ToolRegistry;
  private provider: LLMProvider;

  async *execute(
    plan: Plan,
    context: Record<string, unknown> = {},
  ): AsyncGenerator<HarnessEvent> {
    yield { type: "plan:start", planId: plan.id };

    const stepResults: Record<string, unknown> = {};

    for (const step of plan.steps) {
      // 检查依赖
      const missingDep = step.dependsOn?.find((dep) => !(dep in stepResults));
      if (missingDep) {
        yield { type: "plan:error", stepId: step.id,
                error: `依赖步骤 ${missingDep} 未完成` };
        return;
      }

      yield { type: "step:start", stepId: step.id,
              description: step.description ?? `执行: ${step.id}` };

      const runtime = { context, stepResults };

      try {
        switch (step.type) {
          case "tool":    /* 执行工具函数 */ break;
          case "chat":    /* 流式 LLM 对话 */  break;
          case "structured": /* 结构化 JSON 输出 */ break;
          case "compose": /* 组装最终结果 */  break;
        }
      } catch (err) {
        yield { type: "plan:error", stepId: step.id,
                error: err instanceof Error ? err.message : String(err) };
        return;
      }
    }
  }
}
```

对比最简 Agent，你会发现 AgentRunner 多了三个关键设计：

1. **Plan/Step 模型**：不是一次调用，而是一个有序的步骤序列
2. **依赖管理（dependsOn）**：步骤之间有依赖关系，后一步依赖前一步的结果
3. **事件系统（HarnessEvent）**：每一步都通过 AsyncGenerator 向外播报状态

这三个设计，将在第 2 章（执行循环）中深入讲解。

## 1.5 复盘与延伸

### 本章要点回顾

1. **"调 API" ≠ "做产品"**。裸模型的四大硬伤（无状态、无执行、无感知、无环境）是所有 AI 产品必须解决的工程问题。

2. **Agent = Model + Harness**。模型提供智能，Harness 提供工程基础设施。

3. **Harness 六大组件**：文件系统、Bash+沙箱、记忆系统、搜索+MCP、上下文工程、编排+Hooks。

4. **resumate 的 agent-harness 包**就是一个微型但完整的 Harness 实现——几百行 TypeScript，覆盖了全部核心概念。

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "Harness 就是 LangChain/CrewAI" | 不对。Harness 是**概念体系**，LangChain 是**具体实现**。理解 Harness 之后，用什么框架是次要选择 |
| "我需要一个很复杂的框架才能做 Agent" | 不对。30 行代码就能写出一个最简 Agent，resumate 的 agent-harness 也只有几百行 |
| "模型越强，Agent 越好" | 部分正确。模型决定能力的**下限**，但 Harness 决定能力的**上限** |
| "Harness 是后端概念" | 不对。resumate 的 localStorage 持久化、浏览器 print PDF、React 状态管理都是 Harness 的组成部分 |

### 进阶话题

本章只展示了 Harness 的全景图。如果你迫不及待想深入了解某个方向：

- **执行循环的细节** → 直接跳到第 2 章
- **LLM 抽象层的设计** → 直接跳到第 3 章
- **工具系统的实现** → 直接跳到第 4 章

### 练习

1. **（★☆☆）** 用你熟悉的语言（Python/TypeScript/Go）写一个最简 Agent：输入一句话，调用任意 LLM API，输出回答。对比你的实现和本章的 `SimpleAgent`。

2. **（★★☆）** 在 `SimpleAgent` 的基础上添加 `history: Message[]`，使其支持多轮对话。用户第二轮输入时，Agent 能"记得"第一轮说了什么。

3. **（★★★）** 研究 resumate 源码中 `packages/agent-harness/src/runner.ts` 的完整代码。找出 `AgentRunner.execute()` 方法中的四种 Step 类型，理解它们各处理什么场景。

## 本章小结

这一章，我们站在 Harness Engineering 的大门前，看到了全景：

- **裸模型做不到什么**（四大硬伤）
- **Harness 是什么**（六大组件 + 三层架构）
- **resumate 如何实现 Harness**（agent-harness 包 + web 应用）
- **30 行最简 Agent** 如何体现 Agent = Model + Harness

下一章，我们将进入 Harness 的第一个核心组件——**执行循环**，理解 Agent 如何"观察-思考-行动"。

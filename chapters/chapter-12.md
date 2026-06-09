# 第12章：从 Demo 到产品 — 打包、部署与治理

> **核心概念：** Monorepo，Docker 部署，AGENTS.md 治理合约  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 40 分钟  

## 开篇故事

resumate 在 `localhost:5001` 跑得很好。但要让别人用，需要：

- 一键启动（`docker compose up -d`）
- 构建可靠（`pnpm build` 通过才能上线）
- 行为可预测（AI 编码助手遵循相同的规范）

这三件事对应了 Harness 在"生产环境"这一层的三个核心关注点：部署、构建、治理。

resumate 的 `AGENTS.md` 和 `CLAUDE.md` 本身就是 Harness 治理层的实例。你写的 System Prompt、定义的规则、配置的工具链，都是在构建 Harness。

## 12.1 Monorepo：多包工程的 Harness

resumate 使用 **Turborepo + pnpm workspace** 管理三个包：

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
```

```json
// turbo.json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],  // 先构建依赖包
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "cache": false,           // 开发模式不缓存
      "persistent": true         // 持续运行
    },
    "check-types": {
      "dependsOn": ["^check-types"]
    },
    "test": {}
  }
}
```

Turborepo 的关键能力：
- **依赖拓扑排序**：`^build` 确保 `shared` → `agent-harness` → `web` 的构建顺序
- **并行执行**：无依赖的包同时构建
- **缓存**：未变更的包跳过构建

这就是 Harness 文件系统组件在构建层面的体现。

## 12.2 Docker：一键部署

```dockerfile
# 阶段 1: 构建
FROM node:22-alpine AS builder
RUN corepack enable
WORKDIR /app
COPY . .
RUN pnpm install --frozen-lockfile
RUN pnpm build

# 阶段 2: 运行
FROM node:22-alpine AS runner
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs
USER nextjs
WORKDIR /app
COPY --from=builder /app/packages/web/.next/standalone ./
COPY --from=builder /app/packages/web/public ./packages/web/public
EXPOSE 3000
ENV PORT=3000
CMD ["node", "packages/web/server.js"]
```

```yaml
# docker-compose.yml
services:
  resumate:
    build: .
    ports:
      - "3000:3000"
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
```

`docker compose up -d` → 服务运行在 `http://localhost:3000`。

## 12.3 AGENTS.md：Agent 的"宪法"

resumate 的 `AGENTS.md`（180 行）定义了 AI 编码助手（Claude Code、Codex CLI 等）如何理解和修改这个项目：

```
# AGENTS.md

## Commands
pnpm dev          # 开发模式
pnpm build        # 构建
pnpm lint         # 代码检查

## Architecture
packages/
├── shared/              # 纯类型定义
├── agent-harness/       # Agent 编排引擎
└── web/                 # Next.js 应用

## State management
- resume-store.ts: Zustand + Immer undo/redo...
- chat-store.ts: 聊天 UI 状态...
```

这就是 Harness 的**控制层**。它不运行任何代码，但决定了 Agent（AI 编码助手）在操作代码时的行为边界。

### 治理的核心原理

每当你发现 AI 编码助手犯了一个错误，你就把修复方案写入 AGENTS.md：

```
# 案例：Agent 错误"修复"
Agent 将 localStorage key 从 "resumate-data" 改为 "resume-data"
→ 旧用户数据无法读取
→ 在 AGENTS.md 中添加规则：
  "Never rename localStorage keys without migration logic"
```

这就是 Mitchell Hashimoto 的核心理念：

> 每当 Agent 犯了一个错误，你就工程化一个解决方案，让它永远不再犯同样的错误。

`AGENTS.md` 就是这个"工程化解决方案"的载体。它从空白开始，随着项目的演进不断增长，最终变成项目最可靠的"长时记忆"。

## 12.4 全书回顾：你构建的每一个约束，都在定义 Harness

让我们从第 1 章出发，遍历 12 章的旅程：

```
Ch1:  为什么裸模型不够？           → Agent = Model + Harness
Ch2:  如何组织多步骤任务？          → 执行循环 (AgentRunner, Plan/Step)
Ch3:  如何支持多种模型？            → LLMProvider 接口
Ch4:  如何执行确定性操作？          → ToolRegistry
Ch5:  如何得到结构化数据？          → Zod Schema + Structured Output
Ch6:  如何实时反馈状态？            → SSE + HarnessEvent
Ch7:  如何防止上下文腐烂？          → Context Engineering + RuntimeValue
Ch8:  如何与模型高效对话？          → 三层 Prompt 架构
Ch9:  如何协调多个步骤？            → Orchestration + Hooks
Ch10: 如何保护用户？               → PII 过滤 + 本地模式
Ch11: 如何度量质量？               → 评估体系 + 可观测性
Ch12: 如何走向生产？               → Docker + Monorepo + AGENTS.md
```

这 12 章覆盖了 Harness 的全部六大组件。resumate 从 30 行 `SimpleAgent` 演进到一个完整的、可部署的 AI 产品。

但更重要的是你学到了一种**思维方式**：

1. 遇到问题 → 先问"这是模型的问题还是 Harness 的问题？"
2. 如果是 Harness 的问题 → 是六大组件中的哪一个？
3. 定位组件 → 找到对应的设计模式 → 用最简代码实现

掌握了这个思维路径，遇到新的 Agent 问题就不会无从下手。

### 再谈 Agent = Model + Harness

回到第 1 章的公式。现在你应该能感受到，这五个字承载了多少工程智慧：

- **Model** 是智能的来源。它像一匹千里马。
- **Harness** 是驾驭马的挽具。没有挽具，再好的马也只能在原地打转。

你写的 LLMProvider 接口（Ch3）、ToolRegistry（Ch4）、Zod Schema（Ch5）、SSE 路由（Ch6）、上下文摘要（Ch7）、三层 Prompt（Ch8）、Hook 管理器（Ch9）、PII 过滤器（Ch10）、质量评分（Ch11）、AGENTS.md（Ch12）——**这些全都是 Harness**。

而最重要的是：你现在知道**为什么**需要它们，**什么**时候添加它们，以及**如何**用最简代码实现它们。

## 12.5 复盘与延伸

### 下一步去哪里？

| 方向 | 路径 |
|------|------|
| 深入学习 Agent 框架 | LangGraph（状态机编排）、CrewAI（多 Agent 协作） |
| 模型底层 | Transformer 架构、Fine-tuning、RLHF |
| 生产实践 | MCP Server 开发、A2A 协议、Agent 监控平台 |
| 继续开发 resumate | 添加多语言支持、AI 面试模拟、LinkedIn 导入 |

### 最后的练习

**（全书大作业）** 选择另一个场景（如：旅行规划 Agent、代码审查 Agent、学习助手 Agent），从头搭建它的 Harness。不参考 resumate 的代码，只使用书中学到的概念和模式。

你会发现：场景不同，但 Harness 的架构骨架惊人地一致。好的工程模式是通用的。

## 本章小结

- **Monorepo**：Turborepo 管理多包依赖和构建
- **Docker**：多阶段构建 + 非 root 运行
- **AGENTS.md**：Agent 的治理合约，项目的"长时记忆"
- **元闭环**：你构建 resumate 的过程本身就是在构建 Harness

## 全书后记

感谢你读到这里。

写这本书的动机，源于我自己在构建 resumate 过程中的困惑。市面上有很多"用 LangChain 搭建 Agent"的教程，但几乎没有书告诉你：**为什么要这样设计？** 框架会迭代，API 会变化，但设计原则和思维方式是稳定的。

Harness Engineering 是一个年轻的领域。2026 年我们才开始为它命名。本书的定位不是"权威参考"——这个领域还没有权威——而是"一位同伴的实践笔记"。我和你一样，在这个领域探索。这本书记录了我到目前为止的理解。

如果你在阅读过程中有任何疑问、批评或新的想法，欢迎在 resumate 的 GitHub Issues 中提出。这本书和 resumate 项目一样，是活的。

> 如果你不是模型本身，那你就是 Harness 的一部分。
> — 2026 AI 工程共识

**你写的 System Prompt、你配置的工具链、你设计的记忆机制、你优化的上下文策略、你编写的 Hook 规则——这些都是在构建 Harness。**

现在，去构建吧。

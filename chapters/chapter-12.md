# 第12章：从 Demo 到产品 — 打包、部署与治理

> **核心概念：** Monorepo，Docker 部署，AGENTS.md 治理合约  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 40 分钟  
> **学习目标：** 读完本章后，你将能够：
> 1. 用 Turborepo 管理多包构建依赖
> 2. 编写 Docker 多阶段构建并一键部署
> 3. 用 AGENTS.md 定义 AI 编码助手的行为规范  

---

## 12.1 为什么 Agent 系统需要治理层？

### 确定性系统 vs 概率性系统

在传统 Web 应用中，"部署"意味着构建 + 容器 + CI/CD。部署完成后，行为是确定的——相同的输入永远产生相同的输出。你不需要担心你的 Express 服务器"今天心情不好"返回了不同的结果。

Agent 系统完全不同。它的行为取决于 LLM 的推理，而 LLM 的推理受 prompt、上下文、模型版本等多因素影响。这意味着：

- **AI 编码助手**（Claude Code、Codex CLI）可能在重构时破坏已有功能——上次它正确处理了 Zustand 的 `persist` middleware，这次它可能会"自作主张"地把 `localStorage` key 改了，导致所有老用户数据丢失
- **Agent 本身**可能因为 prompt 改动而产生质量回退——你优化了"专业性"的表达，却意外降低了 ATS 关键词覆盖率
- **多 Agent 协作**时，各 Agent 的行为边界需要明确定义——"这个文件归 Agent A 管还是 Agent B 管？"

### 一个真实的教训

在 resumate 的开发过程中，我们遇到过这样的事：AI 编码助手在重构 `resume-store.ts` 时，把 `localStorage` 的 key 从 `"resumate-data"` 改成了 `"resume-data"`。代码逻辑完全正确，TypeScript 编译通过，所有测试通过——但这个改动导致所有已有用户的简历数据无法读取。

这个 bug 在传统软件中几乎不会发生——没有人会无理由地改一个 key 名。但 AI 编码助手会——它不理解 `localStorage` key 的"向后兼容"语义。它只是在做模式匹配：`resumate-data` 看起来有点长，`resume-data` 更简洁。

这就是为什么 Agent 系统需要**治理层（Governance）**：不是通过代码约束（你不可能为每个 `localStorage` key 写单元测试），而是通过**声明式规则**——在问题出现之前就告诉 Agent "什么不能做"。

### AGENTS.md：声明式治理

在 Harness 的概念体系中，治理层的载体是 **AGENTS.md**——一个声明式文件，定义 Agent（AI 编码助手或运行时 Agent）在操作项目时的行为边界：

- 它不运行任何代码
- 它不参与编译或部署
- 但它决定了 Agent 的行为上限和下限

AGENTS.md 的核心原理（最早由 Mitchell Hashimoto 系统阐述）是：**每当 Agent 犯了一个错误，你就工程化一个解决方案，让它永远不再犯同样的错误。** AGENTS.md 就是这个"工程化解决方案"的载体——它从空白开始，随着项目的演进不断增长，最终变成项目最可靠的"长时记忆"。

从 Harness 三层架构的视角看，AGENTS.md 属于**控制层（Control Layer）**——它不参与执行（那是代理层的事），也不管理运行时状态（那是运行层的事），但它定义了"什么是允许的"、"什么是禁止的"、"什么是推荐的"。本书第 1 章介绍的三层架构（控制层→代理层→运行层），AGENTS.md 正是控制层的完整体现。

---

## 12.2 Monorepo：多包工程的 Harness

resumate 使用 **Turborepo + pnpm workspace** 管理三个包（`shared`、`agent-harness`、`web`）：

```yaml
# pnpm-workspace.yaml — 声明工作空间
packages:
  - "packages/*"
```

Turborepo 的关键能力：
- **依赖拓扑排序**：`^build` 确保 `shared` → `agent-harness` → `web` 的构建顺序
- **并行执行 + 缓存**：无依赖的包同时构建，未变更的包跳过

这是 Harness 文件系统组件在构建层面的体现——管理代码组织、依赖关系和构建流程。

---

## 12.3 Docker：一键部署

resumate 使用 Docker 多阶段构建，核心思路：**构建阶段安装全部依赖并编译，运行阶段只拷贝产物**（Ch10 已讨论过沙箱隔离的安全考量）：

```dockerfile
# 阶段 1: 构建（安装依赖 + pnpm build）
FROM node:22-alpine AS builder
RUN corepack enable
WORKDIR /app
COPY . .
RUN pnpm install --frozen-lockfile && pnpm build

# 阶段 2: 运行（只拷贝 standalone 产物，非 root 用户）
FROM node:22-alpine AS runner
USER nextjs  # 非 root 运行，安全最佳实践
COPY --from=builder /app/packages/web/.next/standalone ./
CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml — 一行命令启动
services:
  resumate:
    build: .
    ports: ["3000:3000"]
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
```

`docker compose up -d` → 服务运行在 `http://localhost:3000`。多阶段构建的优势是最终镜像很小（只包含运行时需要的文件），且安全（非 root 用户、无开发依赖）。

---

## 12.4 AGENTS.md：Agent 的"宪法"

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

每当你发现 AI 编码助手犯了一个错误，你就把修复方案写入 AGENTS.md。以下是 resumate 实际开发中积累的几条真实规则：

```
# 案例 1：localStorage Key 兼容性
Agent 将 localStorage key 从 "resumate-data" 改为 "resume-data"
→ 旧用户数据无法读取
→ 规则：Never rename localStorage keys without migration logic.
        Always use constants for storage keys, not inline strings.

# 案例 2：Zustand Store 结构变更
Agent 在 resume-store 中新增字段时未考虑 Immer patch 兼容性
→ 旧版本的 patch 回放失败，撤销功能崩溃
→ 规则：When modifying Zustand store shape, check Immer patch compatibility.
        New optional fields must have defaults; never remove existing fields
        without a migration version bump.

# 案例 3：API Route 响应格式
Agent 在 /api/agent/run 返回了非 SSE 格式的 JSON 响应
→ 前端 SSE 解析器报错，用户看到白屏
→ 规则：The /api/agent/run route MUST ALWAYS return SSE (text/event-stream).
        Never return plain JSON from this endpoint. Use a separate route
        for non-streaming API calls.

# 案例 4：pnpm Workspace 依赖
Agent 在 web/package.json 直接添加了 shared 的 npm 包依赖
→ pnpm workspace 协议被绕过，构建失败
→ 规则：Internal package dependencies MUST use "workspace:*" protocol.
        Example: "@resumate/shared": "workspace:*" not "^1.0.0"

# 案例 5：TypeScript 类型导出
Agent 重构时删除了 shared/src/index.ts 中"未使用"的类型导出
→ web 包构建失败（通过 barrel export 引用）
→ 规则：Do not remove exports from shared/src/index.ts without checking
        all consumers. Run `pnpm build` before committing.
```

注意这 5 条规则的模式：**每条都对应一个真实踩过的坑，每条都用祈使句写清楚"做什么/不做什么"**。这不是理论推演，而是工程经验的固化。

这就是 Mitchell Hashimoto 的核心理念（参见其推文 *"Whenever an AI agent makes a mistake, engineer a solution so it never makes the same mistake again"*）：

> 每当 Agent 犯了一个错误，你就工程化一个解决方案，让它永远不再犯同样的错误。

`AGENTS.md` 就是这个"工程化解决方案"的载体。它从空白开始，随着项目的演进不断增长，最终变成项目最可靠的"长时记忆"。

---

## 12.5 全书回顾：你构建的每一个约束，都在定义 Harness

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

这 12 章覆盖了 Harness 的核心组件。resumate 从 30 行 `SimpleAgent` 演进到一个完整的、可部署的 AI 产品。

但更重要的是你学到了一种**思维方式**：

1. 遇到问题 → 先问"这是模型的问题还是 Harness 的问题？"
2. 如果是 Harness 的问题 → 是哪个核心组件的问题？
3. 定位组件 → 找到对应的设计模式 → 用最简代码实现

掌握了这个思维路径，遇到新的 Agent 问题就不会无从下手。

### 再谈 Agent = Model + Harness

回到第 1 章的公式。现在你应该能感受到这个公式的分量：

- **Model** 是智能的来源。它像一匹千里马。
- **Harness** 是驾驭马的挽具。没有挽具，再好的马也只能在原地打转。

你写的 LLMProvider 接口（Ch3）、ToolRegistry（Ch4）、Zod Schema（Ch5）、SSE 路由（Ch6）、上下文摘要（Ch7）、三层 Prompt（Ch8）、Hook 管理器（Ch9）、PII 过滤器（Ch10）、质量评分（Ch11）、AGENTS.md（Ch12）——**这些全都是 Harness**。

而最重要的是：你现在知道**为什么**需要它们，**什么**时候添加它们，以及**如何**用最简代码实现它们。

### 能力自检：你掌握了什么？

读完本书，你可以在以下维度上自我评估。每项用 1-5 分打分（1=完全不会，5=可以独立实现）：

| 能力维度 | 检验方式 | 对应章节 |
|---------|---------|---------|
| **解释 Harness 概念** | 能否向同事讲清楚 Agent = Model + Harness？ | Ch1 |
| **设计执行循环** | 能否为一个新场景设计 Plan/Step 序列？ | Ch2 |
| **适配多种模型** | 能否在不改业务代码的前提下切换到新的 LLM 提供商？ | Ch3 |
| **实现工具系统** | 能否判断一个任务应该用工具还是 LLM？ | Ch4 |
| **约束结构化输出** | 能否用 Zod Schema 定义并验证 LLM 的输出格式？ | Ch5 |
| **构建流式响应** | 能否将 AsyncGenerator 事件流转化为 SSE？ | Ch6 |
| **管理上下文窗口** | 能否设计对话摘要 + 近轮保留的上下文策略？ | Ch7 |
| **设计分层 Prompt** | 能否为 Agent 设计 Plan/Step/Context 三层 prompt？ | Ch8 |
| **编排多步骤** | 能否在 Plan 中实现条件分支和自修正循环？ | Ch9 |
| **实施安全护栏** | 能否实现输入过滤 + 输出 PII 检测？ | Ch10 |
| **建立评估体系** | 能否为 Agent 输出设计多维度评分 + LLM-as-Judge？ | Ch11 |
| **治理与部署** | 能否编写 AGENTS.md 约束 AI 编码助手的行为？ | Ch12 |

> 如果你在某个维度上打了 3 分或以下，回到对应章节重读关键部分。这套能力清单也是你未来在 Agent 项目中做技术决策时的"思维检查表"。

### 从三层架构到 AGENTS.md：一个完整的圆

还记得第 1 章的三层架构吗？

```
控制层 (Control)  → AGENTS.md · 代码规范 · 规则权限
代理层 (Agency)   → 工具/API · 浏览器 · 多Agent角色
运行层 (Runtime)  → 内存管理 · 上下文压缩 · 重试回滚
```

现在你应该能看到这个架构的完整闭环：

- **第 2-6 章** 构建了**运行层**和**代理层**：AgentRunner（执行循环）、LLMProvider（模型抽象）、ToolRegistry（工具执行）、Zod Schema（结构化输出）、SSE（事件传输）
- **第 7-9 章** 优化了**运行层**：上下文工程（信息管理）、提示词工程（对话质量）、编排+Hooks（流程控制）
- **第 10-11 章** 为所有层添加了**安全护栏**和**质量度量**
- **第 12 章** 回到了**控制层**：AGENTS.md 是控制层的声明式载体，它不参与执行，但定义了执行的上限和下限

这形成了一个完整的圆：从控制层（定义规则）出发，经过代理层（构建能力）和运行层（优化执行），最终回到控制层（AGENTS.md 记录经验、固化规则）。而 AGENTS.md 的独特之处在于——**你此刻正在阅读的这本书，也是通过 AGENTS.md 治理的 Agent（AI 编码助手）帮助你完成的。** 你就是 Harness 的一部分。

---

## 12.6 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "AGENTS.md 写一次就够了" | AGENTS.md 是**活的文档**——每当你发现 AI 编码助手犯了一个新错误，就应该把规则加入 AGENTS.md。它随项目一起成长 |
| "AGENTS.md 越长越好" | 并非如此。太长的 AGENTS.md 会导致 AI 编码助手"看不到"关键规则。定期清理过时规则，保持文件在 200-300 行以内 |
| "有了 AGENTS.md 就不需要 Code Review" | AGENTS.md 是**约束**，不是**验证**。它告诉 Agent "不要改 localStorage key"，但不能保证 Agent 一定遵守。关键变更仍需人工审查 |

### 下一步去哪里？

| 方向 | 路径 |
|------|------|
| 深入学习 Agent 框架 | LangGraph（状态机编排）、CrewAI（多 Agent 协作） |
| 模型底层 | Transformer 架构、Fine-tuning、RLHF |
| 生产实践 | MCP Server 开发、A2A 协议、Agent 监控平台 |
| 继续开发 resumate | 添加多语言支持、AI 面试模拟、LinkedIn 导入 |

### 从 700 行到 10,000 行：扩展指南

本书的 agent-harness 约 700 行，覆盖了 Harness 的核心概念。当你面对一个 10,000 行的真实 Agent 系统时，以下差异值得关注：

**1. Plan 管理从硬编码到配置驱动。** 本书的 Plan 是 TypeScript 对象字面量。中型项目（5+ 个 Agent、20+ 个 Plan）应该将 Plan 定义移到配置文件（JSON/YAML），让非开发人员也能调整步骤顺序和条件。

**2. ToolRegistry 从 Map 到服务发现。** 本书的 20 行 `Map<string, ToolFn>` 适用于 5-10 个工具。当工具数超过 50 个时，考虑按领域分组（`FileTools`、`APITools`、`DBTools`）或迁移到 MCP Server 架构。

**3. 状态管理从内存到持久化。** `stepResults: Record<string, unknown>` 在进程重启后丢失。中型系统需要将执行状态持久化到数据库，支持断点续跑和跨会话恢复。

**4. 错误处理从 fail-fast 到 graceful degradation。** 本书的 `plan:error → return` 策略适用于教学场景。生产系统需要：区分可重试错误（API 限流）和不可重试错误（Schema 不匹配），对非关键步骤的失败允许降级继续。

**5. AGENTS.md 从单文件到治理体系。** 当项目有 5+ 个 AI 编码助手同时协作时，单一的 AGENTS.md 不够。考虑分层治理：`AGENTS.md`（全局规则）+ `packages/*/AGENTS.md`（包级规则）+ `.github/copilot-instructions.md`（GitHub Copilot 专用）。

> 💡 **核心理念不变**：无论系统多大，Agent = Model + Harness 的公式不变，六大核心组件的设计原则不变。规模增长改变的是**实现复杂度**，不是**设计理念**。

---

## 本章小结

- **Monorepo**：Turborepo 管理多包依赖和构建
- **Docker**：多阶段构建 + 非 root 运行
- **AGENTS.md**：Agent 的治理合约，项目的"长时记忆"
- **元闭环**：你构建 resumate 的过程本身就是在构建 Harness

### 自检问题

1. 为什么 Agent 系统需要治理层（Governance）？传统软件的配置管理为什么不足以约束 AI 编码助手的行为？
2. AGENTS.md 的核心原理（"每当 Agent 犯错，就工程化一个解决方案"）如何体现在 resumate 的实际开发中？
3. 从 Ch1 的三层架构出发，AgentRunner（Ch2-6）、上下文/提示词工程（Ch7-8）、Hooks（Ch9）、安全/评估（Ch10-11）、AGENTS.md（Ch12）分别属于哪一层？画出完整的映射关系。

---

<p align="center">✦ ✦ ✦</p>

---

---

## 全书后记

感谢你读到这里。

写这本书的动机，源于我自己在构建 resumate 过程中的困惑。市面上有很多"用 LangChain 搭建 Agent"的教程，但几乎没有书告诉你：**为什么要这样设计？** 框架会迭代，API 会变化，但设计原则和思维方式是稳定的。

Harness Engineering 是一个年轻的领域。2026 年我们才开始为它命名。本书的定位不是"权威参考"——这个领域还没有权威——而是"一位同伴的实践笔记"。我和你一样，在这个领域探索。这本书记录了我到目前为止的理解。

如果你在阅读过程中有任何疑问、批评或新的想法，欢迎在 resumate 的 GitHub Issues 中提出。这本书和 resumate 项目一样，是活的。

> 如果你不是模型本身，那你就是 Harness 的一部分。
> — 2026 AI 工程共识

**你写的 System Prompt、你配置的工具链、你设计的记忆机制、你优化的上下文策略、你编写的 Hook 规则——这些都是在构建 Harness。**

现在，去构建吧。

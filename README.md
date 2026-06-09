# Harness 理论与实战

> **从零构建简历生成 Agent，理解 AI 系统设计思维**

---

## 关于本书

本书以 **resumate**（一个真实的开源 AI 简历生成器）为贯穿案例，带你从零理解 Agent 系统的核心概念与工程实践。

**技术栈：** Next.js 15 + TypeScript 5.7 + Anthropic SDK + Zustand + Immer

**目标读者：** 入门级 AI 开发者（有 TypeScript/React 基础，不熟悉 Agent 框架）

**教学方法：** Project-Based Learning + why→what→how 三段式

**案例项目：** [resumate](https://github.com/SSSonnettt/resumate)

---

## 全书结构

```
第一部分：基础篇 (Ch 1-3)    理解 Agent 的"大脑"         ~30,000 字
第二部分：能力篇 (Ch 4-6)    给 Agent 装上"手脚"         ~30,000 字
第三部分：品质篇 (Ch 7-9)    让 Agent 更"聪明"           ~30,000 字
第四部分：进阶篇 (Ch 10-12)  走向生产环境                ~30,000 字
                                           总计：      ~120,000 字
```

### 第一部分：基础篇 — 理解 Agent 的"大脑"

| 章节 | 核心概念 |
|------|---------|
| 第1章：你好，Agent — 从"调用 API"到"构建 Agent" | Agent = Model + Harness |
| 第2章：执行循环 — Agent 如何"观察-思考-行动" | Plan/Step 模型，AgentRunner |
| 第3章：LLM 抽象层 — 为什么需要 Provider 接口 | LLMProvider，模型无关性 |

### 第二部分：能力篇 — 给 Agent 装上"手脚"

| 章节 | 核心概念 |
|------|---------|
| 第4章：工具系统 — Agent 如何"做事" | ToolRegistry，混合策略 |
| 第5章：结构化输出 — 让 LLM 生成确定性 JSON | Zod Schema，类型安全 |
| 第6章：流式响应 — SSE 与实时反馈 | HarnessEvent，AsyncGenerator |

### 第三部分：品质篇 — 让 Agent 更"聪明"

| 章节 | 核心概念 |
|------|---------|
| 第7章：上下文工程 — 信息管理的艺术 | Context Rot，RuntimeValue |
| 第8章：提示词工程 — 与模型高效对话 | 三层 Prompt 架构 |
| 第9章：编排与 Hooks — 从单兵到团队 | Dependency Graph，HookManager |

### 第四部分：进阶篇 — 走向生产环境

| 章节 | 核心概念 |
|------|---------|
| 第10章：安全与沙箱 — 给 Agent 戴上"缰绳" | PII 过滤，本地模式 |
| 第11章：评估与可观测性 — AI 系统的质量度量 | 质量评分，执行追踪 |
| 第12章：从 Demo 到产品 — 打包、部署与治理 | Docker，AGENTS.md |

---

## 核心公式

> **Agent = Model + Harness**

- **Model（模型）** 是"大脑"——负责理解和推理。它是智能的来源，但不具备行动能力。
- **Harness（驾驭层/挽具）** 是"操作系统"——包裹在模型外面，提供记忆、工具执行、安全护栏、上下文管理、编排调度。

---

## 阅读须知

每章遵循统一的教学结构：

1. **开篇故事** — 用真实场景引入问题
2. **为什么需要** — 理解动机和背景
3. **概念讲解** — 核心原理和设计思想
4. **动手实践** — 在 resumate 中实现功能
5. **代码解析** — 逐行分析关键实现
6. **复盘与延伸** — 回顾要点，拓展思考
7. **本章小结** — 提炼核心收获

---

> 如果你不是模型本身，那你就是 Harness 的一部分。
> — 2026 AI 工程共识

# Harness 理论与实战

> 从零构建简历生成 Agent，理解 AI 系统设计思维

[在线阅读](https://sssonnettt.github.io/harness-book-/) · [案例项目 resumate](https://github.com/SSSonnettt/resumate) · [Issues](https://github.com/SSSonnettt/harness-book-/issues)

---

## 这是什么

一本关于 AI Agent 工程实践的技术书籍，共 12 章，约 12 万字。以 **resumate**（一个开源 AI 简历生成器）为贯穿案例，从零讲解 Agent 系统的核心概念与工程实现。

**核心公式：**

> **Agent = Model + Harness**

- **Model** 负责理解和推理，是智能的来源
- **Harness** 提供记忆、工具执行、安全护栏、上下文管理、编排调度等工程基础设施

**技术栈：** Next.js 15 + TypeScript 5.7 + Anthropic SDK + Zustand + Immer

**适合谁：** 有 TypeScript/React 基础、想理解 Agent 系统设计的开发者。不需要 Agent 框架经验或机器学习知识。

## 全书结构

```
第一部分：基础篇 (Ch 1-3)    理解 Agent 的"大脑"         ~30,000 字
第二部分：能力篇 (Ch 4-6)    给 Agent 装上"手脚"         ~30,000 字
第三部分：品质篇 (Ch 7-9)    让 Agent 更"聪明"           ~30,000 字
第四部分：进阶篇 (Ch 10-12)  走向生产环境                ~30,000 字
                                           总计：      ~120,000 字
```

| 部分 | 章节 | 核心概念 |
|------|------|---------|
| **基础篇** | 第1章：你好，Agent | Agent = Model + Harness |
| | 第2章：执行循环 | Plan/Step 模型，AgentRunner |
| | 第3章：LLM 抽象层 | LLMProvider，模型无关性 |
| **能力篇** | 第4章：工具系统 | ToolRegistry，混合策略 |
| | 第5章：结构化输出 | Zod Schema，类型安全 |
| | 第6章：流式响应 | HarnessEvent，AsyncGenerator |
| **品质篇** | 第7章：上下文工程 | Context Rot，RuntimeValue |
| | 第8章：提示词工程 | 三层 Prompt 架构 |
| | 第9章：编排与 Hooks | Dependency Graph，HookManager |
| **进阶篇** | 第10章：安全与沙箱 | PII 过滤，本地模式 |
| | 第11章：评估与可观测性 | 质量评分，执行追踪 |
| | 第12章：从 Demo 到产品 | Docker，AGENTS.md |

每章遵循统一结构：开篇故事 → 为什么需要 → 概念讲解 → 动手实践 → 代码解析 → 复盘与延伸。

## 快速开始

本项目使用 [HonKit](https://github.com/honkit/honkit) 构建静态站点。

**环境要求：** Node.js 18+、pnpm 或 npm

```bash
# 克隆仓库
git clone https://github.com/SSSonnettt/harness-book-.git
cd harness-book-

# 安装依赖
npm install

# 本地预览（默认 http://localhost:4000）
npm run serve

# 构建静态站点（输出到 _book/）
npm run build
```

## 项目结构

```
harness-book/
├── chapters/              # 12 章 markdown 源文件
│   ├── chapter-01.md
│   ├── ...
│   └── chapter-12.md
├── _book/                 # 构建产物（git 忽略）
├── .github/
│   └── workflows/
│       └── deploy.yml     # GitHub Actions 自动部署
├── book.json              # HonKit 配置
├── SUMMARY.md             # 侧边栏目录
├── README.md              # 本书前言（本文件）
└── package.json
```

## 在线阅读

本书通过 GitHub Pages 自动部署：

**https://sssonnettt.github.io/harness-book-/**

每次推送到 `main` 分支，GitHub Actions 会自动构建并发布。

## 参与贡献

欢迎通过以下方式参与：

- **发现错误：** 提交 [Issue](https://github.com/SSSonnettt/harness-book-/issues) 说明章节和内容
- **内容改进：** Fork → 修改 → 提交 Pull Request
- **建议新内容：** 在 Issue 中讨论后再动手

修改章节内容时请注意：
- 保持每章的统一教学结构（开篇故事 → ... → 本章小结）
- 代码示例使用 TypeScript，与 resumate 项目一致
- 遵循现有的 Markdown 格式风格

## 许可证

本书内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 许可。你可以自由分享和改编，但需署名、非商业使用、并以相同方式共享。

案例项目 [resumate](https://github.com/SSSonnettt/resumate) 的源代码许可证请参阅其仓库说明。

---

> 如果你不是模型本身，那你就是 Harness 的一部分。

# 第10章：安全与沙箱 — 给 Agent 戴上"缰绳"

> **核心概念：** 安全护栏，沙箱隔离，API Key 管理，PII 过滤  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 40 分钟  

---

## 开篇故事

resumate 的一个早期版本中，用户问 Agent："我的身份证号是 310xxx...，帮我填到简历里"。

Agent 照做了。

虽然这是用户主动提供的，但最佳实践是：**简历中不应该包含身份证号、银行卡号等敏感信息**。即使生成了，导出时也应该过滤掉。

另一个问题是 API Key 管理。resumate 最初要求用户在前端输入 API Key 并存储在 `localStorage`。有用户担心："万一有 XSS 攻击偷走我的 Key 怎么办？"

安全不是 Agent 的"附加功能"——它是 Agent 能否被信任的基础。

---

## 10.1 为什么需要安全？

Agent 系统的安全有三个层面：

| 层面 | 威胁 | 对策 |
|------|------|------|
| **输入安全** | Prompt 注入、恶意输入 | 输入过滤、意图分类 |
| **执行安全** | Agent 执行危险操作 | 沙箱隔离、权限控制 |
| **输出安全** | 敏感信息泄露、不当内容 | 输出过滤、PII 检测 |

resumate 作为简历生成器，最核心的安全需求是：保护用户隐私（数据不出浏览器）和过滤敏感输出。

---

## 10.2 resumate 的安全架构

### 1. 数据不出浏览器

resumate 的隐私策略：**所有用户数据存储在浏览器 `localStorage`，从未上传到服务端**。

```
┌────── 浏览器 ──────────────────────────┐
│                                          │
│  React App                                │
│  ├─ resume-store  → localStorage          │
│  ├─ chat-store    → 内存                   │
│  └─ apiKey        → localStorage          │
│       │                                    │
│       │ POST /api/agent/run {apiKey}       │
│       ▼                                    │
│  Next.js API Route                         │
│  ├─ 使用 apiKey 调 Anthropic API           │
│  └─ SSE 流式返回简历 JSON                   │
│       │                                    │
└───────┼────────────────────────────────────┘
        │ HTTPS
        ▼
   Anthropic API
   (接收：prompt + 用户信息)
   (返回：简历 JSON)
```

数据流分析：
- **用户履历数据**：仅存在于浏览器 `localStorage`，通过 prompt 传给 Anthropic API
- **API Key**：浏览器 `localStorage` → 每次请求时作为参数传给后端 API Route → 后端用完后立即丢弃（不存数据库、不写日志）
- **简历结果**：Agent 返回后存入 `localStorage`，不经过服务端持久化

### 2. PII 过滤器

```typescript
// PII 检测工具
function containsPII(text: string): { found: boolean; types: string[] } {
  const patterns: Record<string, RegExp> = {
    "身份证号": /[1-9]\d{5}(19|20)\d{2}(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])\d{3}[\dXx]/,
    "手机号": /1[3-9]\d{9}/,
    "银行卡号": /\d{16,19}/,
    "邮箱": /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/,
  };

  const found: string[] = [];
  for (const [type, pattern] of Object.entries(patterns)) {
    if (pattern.test(text)) found.push(type);
  }

  return { found: found.length > 0, types: found };
}
```

注意：邮箱在简历中是正常的。PII 过滤器需要有**上下文感知**——在 Header 模块中的 `email` 字段是合法的，但出现在不应该出现的位置（如 Summary 文本中）才需告警。

### 3. 本地模式（Ollama）

resumate 支持通过 `OpenAICompatProvider` 连接本地 Ollama 模型。这是最强的隐私保护：**数据完全不出本机**。

```bash
# 启动本地 Ollama
ollama pull llama3.1
ollama serve

# resumate 连接本地模型
const provider = createOpenAICompatProvider(
  "ollama",                  // 任意字符串
  "http://localhost:11434",  // 本地 Ollama 地址
  "llama3.1"
);
```

本地模式的优势：零数据外泄风险、零 API 费用、离线可用。

---

## 10.3 动手实践：为 resumate 添加输出安全门

```typescript
// 添加安全验证到 compose 步骤
{
  id: "present",
  type: "compose",
  compose: (runtime) => {
    const resume = runtime.stepResults.generate as Resume;

    // 安全门 1: PII 检查
    for (const module of resume.modules) {
      const moduleText = JSON.stringify(module.data);
      const piiCheck = containsPII(moduleText);

      // 排除合法字段（header 中的 email、phone）
      if (module.type !== "header" && piiCheck.found) {
        throw new Error(
          `模块 "${module.type}" 包含疑似敏感信息：${piiCheck.types.join("、")}`
        );
      }
    }

    // 安全门 2: 内容审查（简单关键词检查）
    const forbiddenWords = ["赌博", "色情", /* ... */];
    const fullText = JSON.stringify(resume);
    for (const word of forbiddenWords) {
      if (fullText.includes(word)) {
        throw new Error(`简历含不当内容`);
      }
    }

    return resume;
  },
}
```

### Docker 沙箱部署

resumate 的 Docker 部署提供了一层额外的系统级隔离：

```dockerfile
# 多阶段构建
FROM node:22-alpine AS builder
WORKDIR /app
COPY . .
RUN corepack enable && pnpm install --frozen-lockfile
RUN pnpm build

# 最小化运行镜像
FROM node:22-alpine AS runner
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
USER nextjs  # 非 root 用户运行
WORKDIR /app
COPY --from=builder /app/packages/web/.next/standalone ./
EXPOSE 3000
CMD ["node", "server.js"]
```

安全要点：非 root 用户运行、最小化镜像、仅暴露必要端口。

---

## 10.4 代码解析

### 安全设计原则

| 原则 | resumate 中的实现 |
|------|-----------------|
| **最小权限** | Frontend 只调 `/api/agent/run`，不能直接访问文件系统 |
| **数据最小化** | API 请求只携带当前需要的 messages，不传全部历史 |
| **纵深防御** | 多层检查：输入过滤（classify） → 输出过滤（compose 安全门） |
| **默认安全** | 本地模式可选，PII 检查默认开启，不可通过配置关闭 |

### 安全 vs 便利性

安全设计的永恒张力：更安全通常意味着更不便。

resumate 的平衡策略：
- **默认安全**：PII 过滤默认开启
- **允许选择**：用户可以选择使用本地模型来获得最大隐私
- **透明**：API Key 的使用方式在 README 和 UI 中明确说明

---

## 10.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "Agent 不操作数据库，不需要安全考虑" | Agent 处理用户数据，隐私和安全永远是第一优先级 |
| "加了 PII 检测就安全了" | 上下文很重要。邮箱在 header 中是正常的，在 summary 中可能不是 |
| "本地模型绝对安全" | 本地模型处理的数据不出本机，但要考虑模型本身的安全漏洞和供应链攻击 |

### 练习

1. **（★☆☆）** 为 resumate 的 API Route 添加请求频率限制（rate limiting），防止滥用。

2. **（★★☆）** 实现一个更智能的 PII 检测：基于模块类型决定哪些字段可以包含邮箱/电话。

3. **（★★★）** 研究 Anthropic 的 safety classifier。考虑是否为 resumate 添加预过滤步骤。

---

## 本章小结

- **三层安全**：输入安全 + 执行安全 + 输出安全
- **隐私优先**：数据不出浏览器，API Key 不持久化在服务端
- **PII 过滤**：上下文感知的敏感信息检测

下一章：**评估与可观测性**——AI 系统的质量度量。

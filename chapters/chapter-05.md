# 第5章：结构化输出 — 让 LLM 生成确定性的 JSON

> **核心概念：** Structured Output，Zod Schema，类型安全  
> **难度：** ★★★☆☆  
> **预计阅读时间：** 45 分钟  
> **学习目标：** 读完本章后，你将能够：
> 1. 用 Zod Schema 定义类型安全的数据结构
> 2. 实现 Schema 注入 + Zod 验证的双重保证
> 3. 处理 Schema 演化和数据迁移  

---

## 5.1 为什么需要结构化输出？

### 自由文本的三个问题

| 问题 | 表现 |
|------|------|
| **解析不可靠** | 正则匹配依赖特定格式，LLM 换一种表达方式就失效 |
| **类型不安全** | `JSON.parse()` 返回 `any`，TypeScript 无法提供编译时检查 |
| **验证困难** | 你无法确定"LLM 是否真的生成了所有必需字段" |

### 解决方案

```
自由文本方案：LLM → 文本 → 正则解析 → 字段提取（脆弱）
结构化方案：LLM → JSON → Zod parse → 类型安全的对象（可靠）
```

### ❌ 失败案例：正则解析的脆弱性

假设你用正则从 LLM 的自由文本输出中提取姓名和邮箱：

```typescript
// ❌ 反模式：用正则解析 LLM 的自由文本输出
function parseResumeFromText(llmOutput: string) {
  const nameMatch = llmOutput.match(/姓名[：:]\s*(.+)/);
  const emailMatch = llmOutput.match(/邮箱[：:]\s*([\w.+-]+@[\w.-]+\.\w+)/);

  return {
    name: nameMatch?.[1] ?? "未知",
    email: emailMatch?.[1],
  };
}

// LLM 换了一种表达方式，解析就失败了
parseResumeFromText("我叫张三，联系邮箱是 zhangsan@example.com");
// → { name: "未知", email: undefined }  ← 全丢！
// 原因：LLM 用了 "我叫..." 和 "联系邮箱是..."，而不是正则期望的 "姓名: ..."
```

这个反例揭示了自由文本方案的根本问题：**正则依赖于 LLM 的输出格式，而 LLM 的输出格式是不确定的**。即使你明确要求"用姓名: 格式输出"，LLM 也可能在 5% 的情况下换一种说法。没有 Schema 约束，你永远在追着 LLM 的输出格式跑。

---

## 5.2 什么是结构化输出？

### Zod Schema 定义

resumate 的简历数据类型全部由 Zod 定义（`packages/shared/src/resume.ts`）：

```typescript
import { z } from "zod";

export const resumeSchema = z.object({
  id: z.string(),
  modules: z.array(z.object({
    id: z.string(),
    type: z.enum(["header", "summary", "work-experience",
                   "education", "skills", "projects", "custom"]),
    data: z.record(z.unknown()),
    visible: z.boolean().default(true),
  })),
  theme: z.object({
    templateId: z.string(),
    primaryColor: z.string(),
    fontFamily: z.string(),
    fontSize: z.enum(["small", "medium", "large"]),
    spacing: z.enum(["compact", "normal", "loose"]),
  }),
});
```

### Schema 注入

在 `generateStructured` 中，Zod Schema 被转为 JSON Schema 并注入 prompt：

```typescript
async generateStructured<T>(params: StructuredParams): Promise<T> {
  // 1. Zod Schema → JSON Schema
  const jsonSchema = zodToJsonSchema(params.schema);

  // 2. 注入到 system prompt
  const systemWithSchema = [
    ...params.messages,
    {
      role: "system",
      content: `你必须严格按以下 JSON Schema 输出（只输出 JSON）：
\`\`\`json
${JSON.stringify(jsonSchema, null, 2)}
\`\`\``
    }
  ];

  // 3. 调 LLM → 解析 JSON → Zod 验证 → 返回类型安全的 T
  const response = await callLLM(systemWithSchema);
  const json = JSON.parse(response);
  return params.schema.parse(json) as T;  // Zod parse 在运行时验证
}
```

这个流程形成了**双重验证**：
1. JSON Schema 在 prompt 中约束 LLM 的输出格式（软约束）
2. Zod parse 在运行时验证实际输出（硬约束）

### 与 Tool Use / Function Calling 的对比

许多 LLM 平台提供了原生的结构化输出功能。Schema 注入与它们有何不同？

| 特性 | Schema 注入（Prompt Injection） | 原生 Structured Outputs / Function Calling |
|------|------------|---------------------------|
| 原理 | 在 prompt 中描述格式，Zod 做最终验证 | 模型 API 层级保证 JSON 格式合规 |
| 模型依赖 | 任何 LLM | 仅支持的模型（OpenAI json_schema、Anthropic tool_use） |
| 格式保证 | 概率性（LLM 可能输出不合规 JSON） | API 层级保证格式正确 |
| 灵活性 | 完全自定义 Schema | 受限于模型 API 的功能 |
| 适用时机 | 2023-2024 年的主流方案 | 2025+ 的推荐方案（可靠度已大幅提升） |

resumate 选择 Schema 注入方案的原因是**通用性**——它在 Anthropic、OpenAI、DeepSeek、Ollama 上都能工作，不需要依赖特定模型的 function calling 功能。

不过需要注意的是：到 2025-2026 年，OpenAI 的 `response_format: { type: "json_schema" }` 和 Anthropic 的 tool use 已经大幅提升了原生结构化输出的可靠性。如果你的项目只使用单一模型（如只用 OpenAI），优先考虑原生 Structured Outputs——它在 API 层面就保证了格式正确，比 prompt 注入 + Zod 验证更简洁可靠。Schema 注入仍然适用于需要跨模型兼容的场景，或作为原生方案的 fallback。

---

## 5.3 动手实践：为 resumate 添加类型安全的模块生成

### 各模块的 Zod Schema

```typescript
// 头部模块
const headerSchema = z.object({
  type: z.literal("header"),
  data: z.object({
    name: z.string().min(1, "姓名不能为空"),
    title: z.string(),
    email: z.string().email().optional(),
    phone: z.string().optional(),
    location: z.string().optional(),
    links: z.array(z.object({
      label: z.string(),
      url: z.string().url(),
    })).optional(),
  }),
});

// 工作经历模块
const workExperienceSchema = z.object({
  type: z.literal("work-experience"),
  data: z.object({
    company: z.string().min(1),
    position: z.string().min(1),
    startDate: z.string(),
    endDate: z.string(),
    description: z.string(),
    highlights: z.array(z.string()),  // STAR 法则子弹点
  }),
});

// 技能模块
const skillsSchema = z.object({
  type: z.literal("skills"),
  data: z.object({
    categories: z.array(z.object({
      name: z.string(),
      items: z.array(z.string()),
    })),
  }),
});
```

### 类型安全的模块生成器

```typescript
async function generateModule<T extends z.ZodType>(
  schema: T,
  context: string,
  provider: LLMProvider,
): Promise<z.infer<T>> {
  return provider.generateStructured<T>({
    messages: [
      { role: "system", content: `根据用户信息生成简历模块。` },
      { role: "user", content: context },
    ],
    schema,
  });
}

// 使用：编译时就知道返回类型
const header = await generateModule(
  headerSchema,
  "张三，前端工程师，zhangsan@example.com",
  provider,
);
//    ^? { type: "header"; data: { name: string; title: string; ... } }
//       TypeScript 自动推断！无需 as HeaderData
```

### 容错：Zod parse 失败时

```typescript
async function generateModuleWithRetry<T extends z.ZodType>(
  schema: T, context: string, provider: LLMProvider, maxRetries = 2
): Promise<z.infer<T>> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await generateModule(schema, context, provider);
    } catch (err) {
      if (err instanceof z.ZodError && attempt < maxRetries) {
        // 把 Zod 的错误信息反馈给 LLM，让它修正
        context = `${context}\n\n之前的输出有格式错误，请修正：${err.message}`;
        continue;
      }
      throw err;
    }
  }
  throw new Error("结构化生成失败：超过最大重试次数");
}
```

### Schema 演化：处理版本升级

resumate 的简历 Schema 不是一成不变的。比如 v1.0 的 `skills` 模块只是一个字符串数组，v2.0 改成了分类结构 `categories: [{ name, items }]`。如果有用户用 v1.0 生成的旧简历数据存在 localStorage 中，升级到 v2.0 后会 Zod parse 失败。

解决方案是**数据迁移**：

```typescript
import { z } from "zod";

// v1 Schema
const skillsV1Schema = z.object({
  type: z.literal("skills"),
  data: z.object({ items: z.array(z.string()) }),
});

// v2 Schema（当前）
const skillsV2Schema = z.object({
  type: z.literal("skills"),
  data: z.object({
    categories: z.array(z.object({
      name: z.string(),
      items: z.array(z.string()),
    })),
  }),
});

// 迁移函数
function migrateSkillsV1toV2(oldData: z.infer<typeof skillsV1Schema>):
  z.infer<typeof skillsV2Schema> {
  return {
    type: "skills",
    data: {
      categories: [{ name: "技能", items: oldData.data.items }],
    },
  };
}

// 加载时自动检测版本并迁移
async function loadResume(raw: unknown): Promise<Resume> {
  // 先尝试 v2 parse
  const v2Result = resumeV2Schema.safeParse(raw);
  if (v2Result.success) return v2Result.data;

  // v2 失败，尝试 v1 然后迁移
  const v1Result = resumeV1Schema.safeParse(raw);
  if (v1Result.success) {
    console.warn("检测到旧版本简历数据，自动迁移中...");
    return migrateResumeV1toV2(v1Result.data);
  }

  throw new Error("无法解析简历数据");
}
```

这展示了 Zod 在 Schema 演化中的价值：类型安全的解析 + 失败时可以 fallback 到旧 Schema + 自动迁移。

### 常见 Schema 设计错误

| 错误 | 后果 | 正确做法 |
|------|------|---------|
| 使用 `z.string()` 而非 `z.enum()` | LLM 输出不一致的枚举值 | 明确用 `z.enum(["header", "summary", ...])` |
| 缺少 `.optional()` | LLM 不知道字段是可选的，硬填虚假数据 | 可选字段加 `.optional()` |
| 没有 `.min(1)` | 空字符串通过验证 | 必填字段加 `.min(1, "不能为空")` |
| 嵌套过深 | LLM 容易漏掉深层字段 | 最多3层嵌套 |
| Schema 描述太少 | LLM 不理解字段含义 | 用 `.describe()` 给每个字段加说明 |

---

## 5.4 代码解析：Schema 注入的完整链路

```
用户上下文（自然语言）
    │
    ▼
┌──────────────────────┐
│ Prompt 拼接           │
│ system: 角色 + Schema │
│ user: 用户数据        │
└──────────────────────┘
    │
    ▼
┌──────────────────────┐
│ LLM 调用              │  ← 第1层保证：prompt 中的 JSON Schema
│ 返回: JSON 字符串     │
└──────────────────────┘
    │
    ▼
┌──────────────────────┐
│ JSON.parse()         │  ← 语法验证
└──────────────────────┘
    │
    ▼
┌──────────────────────┐
│ Zod Schema.parse()   │  ← 第2层保证：运行时类型验证
│ 成功 → 类型安全的 T   │
│ 失败 → ZodError      │
└──────────────────────┘
    │
    ▼
┌──────────────────────┐
│ 返回: T (类型安全)    │  ← TypeScript 编译器可验证
└──────────────────────┘
```

三层保证：Prompt 约束（软）→ JSON.parse（语法）→ Zod parse（语义）。

---

## 5.5 复盘与延伸

### ⚠️ 常见误区

| 误区 | 真相 |
|------|------|
| "加上 Schema 就能 100% 正确输出" | Schema 是概率性约束，不是确定性保证。永远要有 Zod parse 做最后防线 |
| "JSON 模式只适合简单结构" | Zod 支持嵌套、联合类型、递归类型。简历这种复杂结构完全可行 |
| "用 TypeScript 类型就够了" | TS 类型只在编译时有效。运行时 LLM 的输出必须用 Zod 验证 |

---

## 本章小结

- **Schema 注入**：在 prompt 中描述 JSON Schema + Zod 运行时验证
- **双重保证**：LLM 格式约束（软）+ Zod parse（硬）
- **类型安全**：从 LLM 输出到 TypeScript 类型的端到端类型链

下一章：**SSE 流式响应**——Agent 如何实时反馈。

### 自检问题

1. Schema 注入 + Zod parse 的"双重验证"各自解决什么问题？如果只用其中一层会有什么风险？
2. 当 Zod parse 失败时，重试策略（将 ZodError 反馈给 LLM）的原理是什么？为什么通常有效？
3. 处理 Schema 演化时，为什么用 `safeParse` + fallback 模式，而不是直接修改 Schema 破坏旧数据兼容？

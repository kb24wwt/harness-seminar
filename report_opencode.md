# OpenCode Harness 深度分析报告

> 源码路径：`D:\opencode\packages\opencode\src\`
> 语言：TypeScript（Bun 运行时）
> 关键依赖：Effect 4.0、Vercel AI SDK（`ai` 6.0）、SQLite + Drizzle ORM

---

## 理解 OpenCode 的两个前置概念

### 1. Effect 框架是什么？

Effect 是 TypeScript 的一个函数式编程框架，把所有"可能有副作用的计算"（网络请求、文件读写、数据库操作）都包裹在一个叫 `Effect` 的容器里，统一管理错误、并发和依赖注入。

用最直白的话类比：
- **普通函数**：`function fetchUser(id)` — 直接执行，可能抛异常
- **Effect 函数**：`const fetchUser = Effect.gen(function* () {...})` — 描述"如何执行"，由框架统一控制执行时机、错误处理和资源清理

这样做的好处：
- 依赖注入变得简单（用 `Layer` 组织服务）
- 错误处理类型安全（编译时就知道会有哪些错误）
- 资源自动清理（类似 Java 的 `try-with-resources`）

**读代码时看到 Effect 语法怎么理解**：

```typescript
// 看起来复杂，其实就是：
Effect.gen(function* () {
  const db = yield* Database.Service    // 获取数据库服务（类似依赖注入）
  const user = yield* db.findUser(id)  // 等待异步操作（类似 await）
  return user
})

// 等价于普通 async/await（概念上）：
async function() {
  const db = inject(Database)     // 获取依赖
  const user = await db.findUser(id)
  return user
}
```

### 2. Vercel AI SDK 是什么？

Vercel AI SDK（包名 `ai`）是一个统一接口库，把 20+ 个 LLM 提供商（OpenAI、Anthropic、Google、Mistral 等）包装成同一套 API。

```typescript
// 无论后端是 Claude 还是 GPT-4，调用方式一样：
const result = streamText({
  model: anthropic("claude-opus-4"),  // 换成 openai("gpt-4o") 也行
  messages: [...],
  tools: {...}
})
```

---

## 总体架构一句话

OpenCode 以 **Effect 框架**驱动整个系统，通过 **Vercel AI SDK** 统一接入 20+ LLM 提供商，会话数据存入 **SQLite 数据库**，上下文压缩采用**两阶段策略**（先修剪工具输出，再生成摘要）。

---

## 维度一：Agent Loop（控制循环）

### 入口链路

```
用户输入（CLI / API）
       ↓
SessionRunState.ensureRunning()   ← 确保同一会话不重复运行
       ↓
SessionProcessor.create()         ← 创建本轮处理器
       ↓
processor.process(streamInput)    ← 启动 LLM 流式调用 + 事件处理
       ↓
检查结果：continue / stop / compact
```

### 并发控制：Runner 状态机

OpenCode 用一个 `Runner` 对象确保同一个会话同一时间只有一个活跃请求：

```
Runner 状态：
  Idle      → 空闲，可以接受新请求
  Running   → 正在处理前台任务（AI 请求）
  Shell     → 正在运行后台 shell 任务

状态转换：
  Idle + 新请求 → Running（开始处理）
  Running + 完成 → Idle
  Running + 中断信号 → 触发 onInterrupt 回调 → Idle
```

这就像一个互斥锁，但更智能——它知道"前台任务"和"后台任务"的区别。

### 核心处理循环（processor.process）

```
[processor.process() 的内部流程]

① 创建 User Message，写入数据库
② Permission.ask() 预检权限
   → 规则集匹配：是否有规则拒绝此操作？
   → 可能等待用户确认
   → 如果拒绝 → 抛出 Permission.RejectedError，结束
③ 构建系统提示：
   agent.prompt（代理的基本指令）
   + 用户自定义系统提示
   + 插件可以修改（plugin.trigger("experimental.chat.system.transform")）
④ 转换消息历史为 LLM 格式（MessageV2.toModelMessagesEffect）
⑤ 注册工具（每个工具都包装了权限检查）
⑥ 调用 LLM.stream()：

[事件处理循环]：
   ┌─────────────────────────────────────────────────────┐
   │ for await (事件 of LLM 流) {                        │
   │   match 事件类型：                                  │
   │                                                     │
   │   "text-start"  → 创建 TextPart，写入数据库         │
   │   "text-delta"  → 追加文本到 TextPart（增量更新）   │
   │   "text-end"    → 标记 TextPart 完成                │
   │                                                     │
   │   "tool-call"   → 执行工具：                        │
   │     ① Permission.ask()                             │
   │     ② Tool.execute(args, context)                  │
   │     ③ updatePart({ status: "completed" })          │
   │                                                     │
   │   "finish-step" → 更新 token 统计                  │
   │                                                     │
   │   "error"       → 记录错误，标记 stop              │
   │ }                                                   │
   │                                                     │
   │ [Stream.takeUntil(ctx.needsCompaction)]             │
   │   如果 token 数超限 → 立即中断流                    │
   └─────────────────────────────────────────────────────┘

⑦ 检查超载：
   isOverflow(tokens, model)？
   → Yes：返回 "compact"（触发压缩后重试）
   → No：返回 "continue"（等待下一条用户输入）
   
⑧ 如果有错误 → 返回 "stop"
```

### 关键设计：流式中断

OpenCode 使用 `Stream.takeUntil()` 实现了一个优雅的中断机制：

```typescript
yield* stream.pipe(
  Stream.tap((event) => handleEvent(event)),      // 处理每个事件
  Stream.takeUntil(() => ctx.needsCompaction),    // 一旦超载就停止
  Stream.runDrain,                                // 消耗完流
)
```

这意味着：**在 LLM 还在生成的过程中，如果检测到 token 超限，可以立即停止接收，不必等到模型说完**。

---

## 维度二：Tool System（工具系统）

### 内置工具列表（17 个）

**编辑类**：
- `bash`：执行 shell 命令（21KB 实现）
- `edit`：编辑文件（diff-based，精确字符串替换）
- `apply_patch`：应用 patch 文件
- `write`：创建新文件
- `read`：读取文件内容

**查询类**：
- `glob`：文件名模式匹配
- `grep`：代码内容搜索（使用 ripgrep）
- `lsp`：通过 LSP 协议查找定义/引用
- `webfetch`：获取网页内容
- `websearch`：网络搜索

**管理类**：
- `task`：任务列表 CRUD
- `skill`：加载技能脚本
- `question`：向用户提问（仅 app 模式）
- `plan`：进入/退出计划模式
- `invalid`：处理无效工具调用

### 工具接口定义

```typescript
// 每个工具实现这个接口
interface Tool.Def<Parameters, Metadata> {
  id: string                   // 工具 ID（如 "bash"）
  description: string          // 发给 LLM 的描述文本
  parameters: Zod.ZodType      // 参数结构（Zod Schema 自动验证）
  execute(args, ctx): Effect   // 执行函数，返回 Effect
}

// 执行上下文（ctx）包含：
type Context = {
  sessionID, messageID, agent
  abort: AbortSignal          // 中断信号
  messages: WithParts[]       // 完整消息历史（工具可以读历史）
  metadata(input): void       // 向 UI 汇报进度/标题
  ask(input): void            // 向用户请求权限
}
```

### 权限系统：基于规则集

OpenCode 的权限系统是一个**有序规则列表（Ruleset）**，采用首匹配原则：

```typescript
// 规则格式
Rule {
  permission: string   // 权限名（如 "bash", "edit", "external_directory"）
  pattern: string      // 通配符模式（如 "*.env", "src/**/*.ts", "*"）
  action: "allow" | "deny" | "ask"
}

// 评估逻辑：
// 从前往后遍历规则列表，返回第一个匹配的规则的 action
// 类似防火墙规则：顺序很重要
```

**代理的默认权限**（`src/agent/agent.ts`）：

```
"*"                 → allow    (默认允许所有操作)
"doom_loop"         → ask      (检测到重复调用同一工具 3 次，询问用户)
"external_directory/*" → ask   (访问工作目录之外的文件夹，询问)
"read/*.env"        → ask      (读取 .env 文件，询问)
"read/*.env.*"      → ask      (读取 .env.local 等，询问)
"question"          → deny     (禁用用户提问工具，仅 app 模式允许)
```

### 工具执行的完整流程

```
LLM 返回工具调用事件
       ↓
"tool-call" 事件处理：
  ① 解析参数（JSON → 对象）
  ② Permission.ask()：
     → 对每个 pattern 检查规则集
     → "deny" → 抛出 DeniedError，结束
     → "allow" → 继续
     → "ask" → 发布权限请求事件，等待用户确认
  ③ Zod 参数验证（自动类型检查）
  ④ tool.execute(args, ctx)：
     → 执行实际操作
     → 可以调用 ctx.metadata() 向 UI 汇报状态
  ⑤ 输出截断（Truncate.output()）：
     → 如果输出超过限制 → 保存到临时文件
     → 返回截断提示 + 文件路径
  ⑥ updatePart({ status: "completed", output: 结果 })
     → 写入数据库
```

---

## 维度三：Context Compaction（上下文压缩）★

**文件**：`src/session/compaction.ts`（约 400 行）

### 触发时机

```
每次 LLM 流处理完一步（finish-step 事件）时检查：

isOverflow({
  cfg: 用户配置,
  tokens: 当前 token 用量,
  model: 模型信息
})

触发条件（近似）：
  消息 tokens > 模型上下文 × 85% - 保留给输出的 tokens
  即：可用上下文 < 模型容量的 15%

→ 设置 ctx.needsCompaction = true
→ Stream.takeUntil() 立即中断 LLM 流
→ 返回 "compact" 触发压缩流程
```

### 两阶段压缩策略

这是 OpenCode 与其他工具最不同的地方：**先修剪，再摘要**。

#### 第一阶段：修剪旧工具输出（Prune）

```
目标：删除历史中已完成的工具输出（最"占地方"但最不重要的内容）

逻辑（从后往前遍历消息）：
  - 跳过最近 2 轮对话（保护最新的上下文）
  - 跳过被标记为 "已压缩" 的工具
  - 跳过受保护的工具（如 skill 工具）
  - 当累计"保护 token 数"超过 40K 后，开始删除：
    → 找到早期的 ToolPart（type="tool", status="completed"）
    → 清除其 output 字段，标记为 compacted=true

参数：
  PRUNE_MINIMUM = 20,000 tokens   至少要释放这么多才值得修剪
  PRUNE_PROTECT = 40,000 tokens   最新的这么多 token 不删
  保护工具列表 = ["skill"]
```

修剪后，工具调用结构还在（LLM 知道有这个操作），但输出内容消失（节省 token）。

#### 第二阶段：生成压缩摘要

```
如果修剪后 token 仍然超限，进行摘要压缩：

① 清理媒体文件：
   toModelMessagesEffect(messages, model, { stripMedia: true })
   → 移除图片、视频（这些会占大量 token）

② 创建特殊的 "compaction" 消息：
   Message {
     role: "assistant",
     mode: "compaction",
     summary: true           ← 标记这是摘要生成
   }

③ 用 PROMPT_COMPACTION 提示词召唤 LLM：

   提示词（翻译）：
   "提供一个详细的 prompt，用于继续我们的对话。
   重点关注有助于继续对话的信息，包括：
   - 我们做了什么
   - 我们在做什么
   - 我们正在处理哪些文件
   - 接下来要做什么
   
   摘要将被另一个 agent 读取并继续工作。
   不要调用任何工具，只返回摘要文本。
   
   摘要模板：
   ## Goal（目标）
   ## Instructions（指令）
   ## Discoveries（发现）
   ## Accomplished（已完成）
   ## Relevant files / directories（相关文件）"

④ LLM 生成摘要，保存为 TextPart
⑤ 压缩消息成为会话的一部分，后续对话从它之后继续
```

### 压缩后的自动继续

压缩完成后，OpenCode 可以自动继续：

```
if auto == true:
  if overflow == true:
    → 重放用户最后一条消息（排除与压缩无关的部分）
  else:
    → 自动发送内部消息（synthetic=true）：
      "Continue if you have next steps, or stop and ask for clarification..."
```

`synthetic=true` 表示这条消息是系统内部生成的，不计入用户输入，对用户不可见。

### 压缩前后示例

**压缩前**（假设 120K tokens）：
```
Message 1: User "帮我分析整个 src/ 目录"
Message 2: Assistant [tool: glob] + 文本 "我来扫描..."
  ToolPart { output: "找到 200 个文件：[完整列表...]" }   ← 大量 token
Message 3: Assistant [tool: read src/api.ts] + 文本
  ToolPart { output: "文件内容：[5000行代码]" }          ← 大量 token
...（共 50 条消息）
Message 50: User "关注错误处理部分"
```

**修剪后**（释放了 60K tokens）：
```
Message 1: User "帮我分析整个 src/ 目录"
Message 2: Assistant [tool: glob] + 文本 "我来扫描..."
  ToolPart { output: null, compacted: true }             ← 输出被清除
Message 3: Assistant [tool: read src/api.ts] + 文本
  ToolPart { output: null, compacted: true }             ← 输出被清除
...
Message 50: User "关注错误处理部分"
```

**再压缩后**（仍然超限时）：
```
Message 1: User "帮我分析整个 src/ 目录"
...（早期消息保留但工具输出已清除）
Message 49: User "...（其他消息）"
Message 50: User "关注错误处理部分"
Message 51: Assistant [compaction summary]:
  ## Goal
  分析 src/ 目录的错误处理机制
  ## Accomplished
  扫描了 200 个文件，读取了 api.ts 等核心文件
  发现 api.ts 中错误处理不一致...
  ## Relevant files
  src/api.ts, src/error.ts, src/middleware.ts
```

---

## 维度四：Memory System（记忆系统）★

OpenCode 的记忆系统完全基于 **SQLite 数据库**，这是三款工具中唯一采用关系型数据库的。

### 数据库核心表结构

#### session 表（会话）

```
列名                  类型     说明
------               ------   ------
id (PK)              TEXT     会话唯一 ID
project_id (FK)      TEXT     所属项目
parent_id (FK)       TEXT     父会话（子会话嵌套关系）
slug                 TEXT     URL 友好的短名称
directory            TEXT     工作目录路径
title                TEXT     会话标题
version              TEXT     版本号
permission           JSON     会话级权限规则集（覆盖全局）
time_created         INT      创建时间（Unix 时间戳）
time_updated         INT      最后更新时间
summary_additions    INT      代码变更统计：新增行数
summary_deletions    INT      代码变更统计：删除行数
summary_files        INT      代码变更统计：修改文件数
revert               JSON     恢复点信息（messageID + git 快照）
```

#### message 表（消息）

```
列名                  类型     说明
------               ------   ------
id (PK)              TEXT     消息唯一 ID
session_id (FK)      TEXT     所属会话（级联删除）
data                 JSON     完整消息元数据

data 字段包含：
  id, sessionID, role (user/assistant)
  mode: "normal" | "compaction"
  agent: 处理此消息的代理名称
  model: 使用的 LLM 模型
  format: "text" | "json_schema"（输出格式）
  cost: { input, output, cache }（费用）
  tokens: { input, output, reasoning, cache }（token 用量）
  finish: "stop" | "tool_calls" | "length"（结束原因）
  summary: boolean（是否为摘要消息）
  time: { created, completed }
```

#### part 表（消息部分）

这是最灵活的表，存储消息的实际内容：

```
列名                  类型     说明
------               ------   ------
id (PK)              TEXT     部分唯一 ID
message_id (FK)      TEXT     所属消息（级联删除）
session_id           TEXT     所属会话（冗余字段，方便查询）
data                 JSON     部分内容（根据 type 不同而变化）
```

**Part 的类型（data.type 字段）**：

```
TextPart：
  { type: "text", text: "...", synthetic?: true, ignored?: boolean }
  
ToolPart：
  { type: "tool", tool: "bash", callID: "call_1",
    state: {
      status: "pending" | "running" | "completed" | "error",
      input: { command: "ls -la" },
      output: "total 48\ndrwxr...",    ← 工具执行结果
      title: "Running bash command",
      time: { start: 1234, end: 1235, compacted?: 1240 }
    }
  }

FilePart：
  { type: "file", mime: "image/png", url: "blob:...", filename: "arch.png" }

ReasoningPart：
  { type: "reasoning", text: "让我先想想..." }  ← 扩展思考内容

CompactionPart：
  { type: "compaction", auto: true, overflow: true }  ← 压缩标记

SnapshotPart：
  { type: "snapshot", snapshot: "git_tree_hash" }  ← Git 快照
  
PatchPart：
  { type: "patch", hash: "abc123", files: ["src/api.ts"] }  ← 代码变更记录
```

### 记忆的写入流程

写入发生在 LLM 流式事件处理过程中，**实时写入数据库**：

```
LLM 流事件 → 立即写数据库：

"text-start"
  → db.insert(PartTable, { type: "text", text: "" })
  → 创建空的 TextPart

"text-delta"（增量文本）
  → db.update("UPDATE part SET data = json_set(data, '$.text', 
               json_extract(data, '$.text') || ?)")
  → 直接在数据库中追加文本，不需要读再写

"text-end"
  → db.update(part, { status: "completed" })

"tool-call"（工具执行完成）
  → db.update(ToolPart, { 
      status: "completed",
      output: 结果文本,
      time.end: 当前时间
    })

"finish-step"（LLM 一步结束）
  → db.update(MessageTable, {
      tokens: { input, output, cache },
      cost: { ... },
      finish: stop_reason
    })
```

**增量更新**的实现利用了 SQLite 的 `json_set()` 函数，直接修改 JSON 字段中的特定路径，而不需要读取整行、修改、再写回——这是一个重要的性能优化。

### 记忆的读取与注入

每次发起 AI 请求时，先从数据库读取完整历史，然后转换格式：

```
① 查询数据库：
   messages = SELECT * FROM message WHERE session_id = ?
              ORDER BY time_created
   
   parts = 对每条消息，SELECT * FROM part WHERE message_id = ?
           ORDER BY id

② 消息转换（MessageV2.toModelMessagesEffect）：
   把数据库中的 Message + Parts → LLM 格式的 ModelMessage[]
   
   TextPart → { role: "...", content: text }
   FilePart（图片）→ { type: "image", image: url }
   ToolPart（已完成）→ 作为对话历史包含
   CompactionPart → 跳过（压缩标记不传给 LLM）

③ 构建完整请求：
   {
     model: 选择的 LLM,
     messages: 转换后的历史,
     system: 系统提示文本,
     tools: 注册的工具列表
   }
```

### 三种记忆层级

OpenCode 的"记忆"实际上分三层：

| 层级 | 存储位置 | 内容 | 生命周期 |
|------|---------|------|---------|
| Part 级（最细） | SQLite part 表 | 每条消息的文本、工具调用等 | 按会话，cascade delete |
| 消息级 | SQLite message 表 | 元数据（token 用量、模型、时间） | 按会话 |
| 会话级（最粗） | SQLite session 表 | 项目、目录、权限、变更统计 | 跨会话持久 |

没有"跨会话文本摘要"的概念（这与 Claude Code 和 Codex 的区别），OpenCode 依靠完整的数据库历史来维持上下文，通过压缩来控制规模。

---

## 维度五：Provider / LLM 绑定

### 多 Provider 架构

OpenCode 最大的架构特色是**完全不绑定特定 LLM**，通过 Vercel AI SDK 支持 20+ 提供商：

```
Anthropic Claude    OpenAI GPT-4/o    Google Gemini
Azure OpenAI        Google Vertex AI  xAI Grok
Mistral             Groq              DeepInfra
Cerebras            Cohere            Together AI
Perplexity          AWS Bedrock       GitHub Copilot
GitLab AI           Venice AI         OpenRouter
Alibaba Qwen        ... 等 20+ 个
```

### Provider 接口（适配器模式）

每个提供商有一个"适配器"：

```typescript
// src/provider/provider.ts 中每个提供商的配置示例

anthropic: {
  autoload: false,      // 不自动加载模型列表
  options: {
    headers: {
      "anthropic-beta": "interleaved-thinking-2025-05-14"  // Anthropic 特定头
    }
  }
}

openai: {
  autoload: false,
  async getModel(sdk, modelID) {
    return sdk.responses(modelID)    // 使用 responses API（非 chat completions）
  }
}

"amazon-bedrock": {
  autoload: true,   // 自动从 AWS 加载可用模型列表
  options: { region, credentialProvider },
  async getModel(sdk, modelID) {
    return sdk.languageModel(modelID)
  }
}
```

适配器封装了每个提供商的特殊性（认证方式、API 版本、模型命名规范），调用方无需关心。

### 统一的调用接口

无论使用哪个 LLM，最终都通过 Vercel AI SDK 的 `streamText()` 函数：

```typescript
// src/session/llm.ts
streamText({
  model: language_model,          // Vercel AI SDK 的统一 model 对象
  messages: model_messages,       // 统一的 ModelMessage[] 格式
  tools: tools,                   // 统一的工具描述格式
  temperature, topP, maxOutputTokens,  // 通用参数
  headers: {
    "x-opencode-session": sessionID,
    "User-Agent": `opencode/${version}`
  },
  maxRetries: 0,                  // 重试在上层处理
})
```

### 模型能力声明

每个模型在配置中声明自己的能力，系统据此调整行为：

```typescript
ModelInfo {
  context_window: 200000,       // 上下文窗口大小
  supports_thinking: true,      // 是否支持扩展思考
  supports_vision: true,        // 是否支持图片输入
  supports_tool_call: true,     // 是否支持工具调用
  pricing: {                    // 定价（用于成本计算）
    input_per_million: 3,
    output_per_million: 15
  }
}
```

---

## 维度六：状态管理

### InstanceState：按工作目录隔离

这是 OpenCode 特有的状态管理方式：

```typescript
// 每个工作目录有独立的 InstanceState
const state = yield* InstanceState.make<AgentState>(
  Effect.fn("Agent.state")(function* (ctx) {
    // ctx.directory = 当前工作目录
    // 不同工作目录 → 创建不同的 state 实例
    return {
      runningMessages: new Map(),
      compactionSignals: new Map()
    }
  })
)
```

这意味着：同一进程可以服务多个工作目录（多个项目），各自的 Agent 状态完全隔离。

### 会话实时状态

```typescript
// src/session/status.ts
SessionStatus =
  | { type: "idle" }      // 空闲，等待用户输入
  | { type: "busy" }      // 正在处理
  | { type: "retry",      // 正在等待重试
      attempt: number,
      message: string,
      next: number        // 下次重试的时间（毫秒）
    }
```

### 会话的持久状态（数据库）

会话所有的持久状态都在 SQLite 数据库中，包括：
- 代码变更统计（添加/删除行数、修改文件）
- 恢复点（特定消息的 git 快照）
- 会话级权限覆盖
- 父子会话关系（对话分支）
- 共享链接

---

## 架构总结

| 维度 | OpenCode 的选择 |
|------|---------------|
| Agent Loop | Effect 框架驱动，Runner 并发控制，流式中断（takeUntil） |
| 工具系统 | 17 个内置工具，Wildcard 规则集权限，增量写数据库 |
| 上下文压缩 | 两阶段：先修剪旧工具输出（保留结构），再生成 Markdown 摘要 |
| 记忆系统 | SQLite + Drizzle ORM，Part 化消息，增量更新，无独立摘要记忆 |
| LLM 绑定 | 完全解耦（20+ 提供商），Vercel AI SDK 统一接口，Adapter 模式 |
| 状态管理 | 按工作目录隔离（InstanceState），实时写库，数据库即真相源 |

**设计哲学**：**面向多 LLM 生态**——最大的价值主张是提供商无关性。Effect 框架牺牲了初学门槛，换来了类型安全的依赖注入和精确的错误处理。SQLite 存储使得所有历史都可以精确查询和分析，而不只是简单的文本追加。

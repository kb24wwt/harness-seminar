# Claude Code Harness 深度分析报告

> 源码路径：`D:\claude-code-source-code\src\`
> 语言：TypeScript（Node.js / Bun 运行时）
> 版本：@anthropic-ai/claude-code-source 2.1.88

---

## 总体架构一句话

Claude Code 以 **QueryEngine** 为核心，用异步生成器（async generator）串联"会话管理 → LLM 调用 → 工具执行"三层，上下文压缩和记忆系统作为一等公民内置其中。

---

## 维度一：Agent Loop（控制循环）

### 层次结构

Claude Code 的主循环分为两层：

```
第一层：QueryEngine.submitMessage()
  会话级别，管理消息存储、压缩触发、记忆注入
       ↓
第二层：query.ts 的 query() 函数
  轮次级别，处理单次 API 请求、工具执行、流式解析
```

### 一次完整的交互流程

```
用户输入 "分析 src/main.ts"
         ↓
[QueryEngine.submitMessage()]
  ① 预处理：检测斜杠命令（/compact 等），无则跳过
  ② 记录：立即写入 transcript.jsonl（防止中断丢失）
  ③ 注入：从记忆文件加载相关记忆，拼入系统提示
  ④ 发出：系统初始化消息（工具列表、权限模式）
         ↓
[query() 主循环 - 第 N 轮]
  ⑤ 构建 API 请求（消息 + 系统提示 + 工具描述）
  ⑥ 流式调用 Claude API
  ⑦ 解析流：
       text_delta → 累积文本
       tool_use   → 收集工具名+参数
       end_turn   → 消息完成
  ⑧ 权限检查（并行检查所有工具调用）
  ⑨ 执行工具，收集结果
  ⑩ 将工具结果追加到消息列表，重回步骤 ⑤
         ↓
  当 stop_reason == "end_turn" 且无工具调用 → 退出循环
         ↓
[QueryEngine 后处理]
  ⑪ 异步写入完整 transcript
  ⑫ 检查是否触发自动压缩
  ⑬ 返回成功结果（文本 + 成本 + token 用量）
```

### 循环何时退出

| 终止原因 | 触发条件 |
|---------|---------|
| 正常结束 | `stop_reason == "end_turn"` 且无工具调用 |
| 达到轮数上限 | `max_turns` 参数耗尽 |
| 超出预算 | `total_cost >= max_budget_usd` |
| 执行错误 | API 错误或工具抛异常 |

### 关键设计：生成器（Generator）作为控制流

`submitMessage()` 是一个 `async *` 生成器函数，而不是普通函数。这意味着：
- **流式推送**：每产生一条消息/事件，立即 `yield` 给调用者，而不等全部完成
- **可取消**：调用者随时可以中断生成器
- **可组合**：`yield* query(...)` 把下层的结果直接转发出去

```
// 简化的结构
async *submitMessage(prompt) {
  yield 系统初始化消息          // 立刻让 UI 显示工具列表
  yield* query(...)            // 把 LLM 流式事件逐条转发
  yield 成功结果               // 最后发出总结
}
```

### 关键文件

| 文件 | 作用 |
|------|------|
| `src/entrypoints/cli.tsx` | CLI 入口，初始化 QueryEngine |
| `src/QueryEngine.ts`（1296 行） | 会话级控制循环 |
| `src/query.ts`（1730 行） | 轮次级 API 调用与工具执行 |

---

## 维度二：Tool System（工具系统）

### 工具接口定义

每个工具实现以下接口（`src/Tool.ts`）：

```
Tool {
  name: string                    // 工具唯一名称
  inputSchema: Zod Schema         // 参数的结构和验证规则
  call(args, context, ...)        // 核心执行函数
  description(input)              // 描述文本（会发给 LLM 作为工具说明）
  checkPermissions(input)         // 权限检查
  isReadOnly(input): boolean      // 是否只读（影响权限级别）
  isConcurrencySafe(input)        // 是否可并行执行
  maxResultSizeChars: number      // 结果超限时写文件而非内联
}
```

### 工具分层设计

Claude Code 有 55 个左右工具（`src/tools/` 下有 55 个工具目录），分为三类：

**原子工具（基础 I/O）**
- `BashTool`：执行 shell 命令
- `FileReadTool`：读取文件内容
- `FileWriteTool`：写入文件
- `GrepTool`：正则搜索文件内容
- `GlobTool`：文件名模式匹配

**组合工具（任务管理）**
- `TaskCreateTool`、`TaskUpdateTool`、`TaskListTool`：任务跟踪系统
- `ScheduleCronTool`：定时任务
- `EnterWorktreeTool`：切换 git worktree

**元工具（能力扩展）**
- `AgentTool`：在当前会话中启动子 Agent（嵌套执行）
- `SkillTool`：加载并执行预定义的技能脚本
- `MCPTool`：调用 MCP 协议注册的外部工具

### 工具执行流程

```
LLM 输出 tool_use 块（含 tool_name + input JSON）
         ↓
canUseTool() 权限检查：
  → 查配置规则：always_allow / always_deny?
  → 是 auto 模式？调用分类器自动判断
  → 否则：弹出 UI 对话框请用户确认
         ↓
Tool.call(args, context, canUseTool, parentMessage)
  → 执行实际操作
  → 如果结果 > maxResultSizeChars，写入临时文件
  → 返回 ToolResult { data, newMessages? }
         ↓
mapToolResultToToolResultBlockParam()
  → 转换为 API 格式的 tool_result 块
         ↓
追加到消息列表，下一轮继续发给 LLM
```

### 权限系统三层降级

```
第一层（最快）：配置规则
  → settings.json 中的 alwaysAllow / alwaysDeny 列表
  → 命中则立即 allow/deny，不再往下走

第二层（自动）：分类器
  → 仅在 --auto 模式下启用
  → yoloClassifier 分析对话上下文判断是否允许（src/utils/permissions/yoloClassifier.ts）
  → bashClassifier 专门处理 shell 命令安全性（src/utils/permissions/bashClassifier.ts，
    注意：在外部构建中为 stub，此功能仅 Anthropic 内部可用）

第三层（交互）：用户确认
  → 弹出 UI 对话框
  → 用户可选：允许一次 / 永久允许 / 拒绝
```

---

## 维度三：Context Compaction（上下文压缩）★

### 为什么需要压缩

LLM 有固定的上下文窗口（例如 200K tokens）。长对话的消息历史不断增长，最终会撑满窗口。压缩的目的是：**用一段结构化摘要替换早期对话，释放 token 空间，同时保留关键信息**。

### 触发时机

**自动触发（`src/services/compact/autoCompact.ts`）**：
```
触发阈值（基于绝对值 buffer，非百分比）：
  effectiveContextWindow = contextWindow - min(maxOutputTokens, 20_000)
  autoCompactThreshold   = effectiveContextWindow - 13_000  （AUTOCOMPACT_BUFFER_TOKENS）

每轮结束后独立检查一次：
  tokenUsage >= autoCompactThreshold  →  立即触发压缩

不存在"连续多轮 WARNING"才触发的逻辑，单次超阈值即触发。
```

WARNING 显示（UI 变色）：独立的视觉提示，阈值为 effectiveContextWindow - 20_000，
与压缩触发无因果关系。

**手动触发**：用户输入 `/compact` 命令

### 压缩流程（三阶段）

#### 阶段 1：预处理（`compact.ts`）

```
① 从消息列表中剥离图片
   原因：图片本身占用大量 token，会让压缩 API 调用也超限
   
② 按 API 轮次分组
   目的：识别哪些消息属于同一次"问答循环"，作为摘要边界
```

#### 阶段 2：调用 LLM 生成摘要（`prompt.ts`）

发一个专用的压缩请求给 Claude，请它生成结构化摘要（`src/services/compact/prompt.ts`）：

```
模板常量：BASE_COMPACT_PROMPT（约 80 行）
摘要格式：<analysis>（草稿思考区，最终会被剥离）+ <summary>（最终输出）

<summary> 包含以下 9 个章节：
  1. Primary Request and Intent    主要任务和目标
  2. Key Technical Concepts        关键技术概念
  3. Files and Code Sections       涉及的文件和代码段
  4. Errors and Fixes              出现的错误和修复方式
  5. Problem Solving               解题思路
  6. All User Messages             全部用户输入（精简）
  7. Pending Tasks                 待完成任务
  8. Current Work                  当前正在做的工作
  9. Optional Next Step            可选的下一步

max_tokens：20,000（MAX_OUTPUT_TOKENS_FOR_SUMMARY，而非 2,000）
```

注：不存在 `<user_context>` 和 `<project_context>` 标签，这两个名称在压缩 prompt 中从未出现。

#### 阶段 3：重建消息列表（`postCompactCleanup.ts`）

```
① 创建"压缩边界消息"（SystemCompactBoundaryMessage）：
   {
     type: "system",
     subtype: "compact_boundary",
     content: "[earlier conversation truncated]",
     compactMetadata: {
       summary: "...",
       userContext: "...",
       projectContext: "...",
       preservedSegment: {
         tailUuid: 最后一条旧消息的ID,
         headUuid: 第一条被保留消息的ID
       }
     }
   }

② 后压缩恢复（让 LLM 有足够上下文继续工作）：
   - 重新读取最近编辑的 5 个文件（恢复文件状态）
   - 重新注入最近调用的技能内容（最多 25K tokens）
   - 重新注入 MCP 工具说明、代理列表等

③ 释放旧消息：
   mutableMessages.splice(0, boundaryIdx)
   → 边界之前的所有消息从内存中删除（让垃圾回收器处理）
```

### 压缩前后对比

**压缩前**（100 条消息，约 180K tokens）：
```
User: "分析 main.ts"
Assistant: [tool_use: FileRead]
User: [tool_result: 文件内容...]
Assistant: "我读取了文件..."
...（90 条消息）...
Assistant: "以上是分析结果"
```

**压缩后**（约 30K tokens）：
```
[COMPACT BOUNDARY]
  summary: "分析了 main.ts，发现错误处理缺失..."
  userContext: "用户是 Go 专家，首次接触 React"
  projectContext: "TypeScript API 服务，需要 2026-05-20 前完成"

[POST-COMPACT 恢复]
  main.ts 最新文件内容（重新读取）
  api.ts 最新文件内容
  ...

User: "继续分析"  ← 从这里开始保留
```

### 关键设计亮点

- **不是简单截断**：保留结构化信息（用户上下文 + 项目背景），而不是随机丢弃
- **后压缩恢复**：压缩后自动重新注入关键文件状态，防止 LLM 忘记当前工作内容
- **边界即检查点**：压缩边界消息记录了分割点的 UUID，支持会话恢复

---

## 维度四：Memory System（记忆系统）★

### 两种记忆的区别

| 类型 | 生命周期 | 存储位置 | 用途 |
|------|---------|---------|------|
| 会话内记忆 | 当前对话结束即消失 | 内存 | 追踪本次对话发现的技能等临时信息 |
| 跨会话持久记忆 | 永久保存，下次对话可用 | 文件系统（MEMORY.md） | 用户偏好、项目背景、工作指导 |

本节重点介绍**跨会话持久记忆**。

### 四种记忆类型（`src/memdir/memoryTypes.ts`）

```
user（用户信息）
  存什么：用户的角色、技能背景、知识水平
  例子："用户是 Go 专家，首次接触 React"
  使用：调整解释深度和类比方式

feedback（反馈指导）
  存什么：用户给出的工作方式指导——要做什么、避免什么
  例子："集成测试必须用真实数据库，不要 mock"
  格式：规则 + Why（原因） + How to apply（何时适用）
  使用：指导未来的代码和测试决策

project（项目背景）
  存什么：当前进行中的工作、截止日期、约束
  例子："2026-05-20 前冻结合并（移动团队发布分支）"
  注意：保存时相对日期要转为绝对日期（"明天" → "2026-05-12"）
  使用：了解项目时间线和优先级

reference（外部指针）
  存什么：外部系统的位置信息
  例子："管道 bug 在 Linear 项目 'INGEST' 中追踪"
  使用：用户提及外部系统时知道去哪里找信息
```

### 记忆文件结构

记忆存储为 Markdown 文件，带 frontmatter 元数据：

```markdown
---
name: 用户技术背景
description: 用户有 Go 经验，React 新手 — 解释时用后端类比
type: user
---

用户有 10 年 Go 经验，但这是他第一次接触这个项目的 React 部分。
如果讨论前端，应该用后端概念做类比（比如把 useState 比作 Go 的局部变量）。

---
name: 测试策略：禁止 mock 数据库
description: 集成测试必须用真实数据库
type: feedback
---

集成测试必须命中真实数据库，不使用 mock。

**Why:** 去年 mock 和 prod 环境不一致，导致一个坏迁移通过了测试，直到线上才失败。

**How to apply:** 写任何涉及数据库的测试时，使用 test fixture，不要 stub。
```

### 记忆文件目录

```
~/.claude/
└── projects/
    └── <sanitized-project-path>/   ← 按项目根目录区分（git root）
        └── memory/
            ├── MEMORY.md           ← 记忆索引和内容文件
            └── logs/
                └── YYYY/MM/
                    └── YYYY-MM-DD.md  ← 每日操作日志
```

路径由 `src/memdir/paths.ts` 中的 `getAutoMemPath()` 生成：
`~/.claude/projects/<sanitized-git-root>/memory/`

另外，每个项目可以有 `CLAUDE.md` 文件，起到项目级记忆的作用（但属于静态配置，不由系统自动写入）。

### 记忆的写入时机

系统在对话过程中自动检测并保存记忆（`src/services/extractMemories/`）：

```
检测条件（触发保存）：
  ✓ 用户透露了角色或技能背景
  ✓ 用户纠正了某个做法（"不，不要这样，要那样"）
  ✓ 用户确认了某个非显而易见的做法
  ✓ 用户提到了截止日期、项目约束
  ✓ 用户说到了外部系统的位置

不保存的信息（`src/memdir/memoryTypes.ts` 明确列出）：
  ✗ 代码模式、架构（可以直接读代码推导）
  ✗ git 历史（可以 git log 查看）
  ✗ 当前对话的临时状态
  ✗ CLAUDE.md 中已有的内容
  ✗ 调试过程（只保存最终修复结论）
```

### 记忆注入上下文的流程

记忆注入分**两层**，机制不同：

#### 第一层：MEMORY.md 索引全量注入（system prompt）

`MEMORY.md` 索引文件不做相关性筛选，直接全量写入 system prompt（上限 200 行 / 25,000 字节）。

#### 第二层：具体记忆文件按需注入（sideQuery 预取）

在每个用户轮次开始时，与主模型推理**并行**启动一次异步预取（`query.ts:301`）：

```
① 扫描记忆目录中所有 .md 文件（排除 MEMORY.md）
   → 每个文件只读前 30 行提取 frontmatter（文件名、description、type、修改时间）
   → 最多处理 200 个文件（src/memdir/memoryScan.ts）

② 将清单 + 用户当前输入发给 Sonnet（LLM sideQuery）：
   → 发送格式示例：
       Query: <用户输入>
       Available memories:
       - [user] background.md (2026-05-10T...): 用户是 Go 专家，React 新手
       - [feedback] testing.md (2026-05-11T...): 集成测试必须用真实数据库
       ...
       Recently used tools: BashTool, FileReadTool
   → Sonnet 以 JSON Schema 约束格式返回最多 5 个文件名：
       { "selected_memories": ["testing.md", "background.md"] }
   → 判断逻辑由 Sonnet 自主完成（不存在关键词匹配或向量相似度）
   → 特殊规则：最近刚用过的工具，其 API 文档类记忆跳过，warning/gotcha 类保留

③ 读取选中文件内容（单文件上限 4096 字节 / 200 行）
   以 <system-reminder> 形式作为 isMeta 用户消息注入对话（messages.ts:3708）

注入量限制：
  单文件上限：4 KB
  每轮最多选：5 个文件（约 20 KB）
  会话累计上限：60 KB（超过后停止预取）
```

#### 记忆陈旧性标注（memoryAge.ts）

超过 1 天的记忆会自动附加提醒文字，告知该记忆是历史观测值，引用代码行号等信息可能已过时，建议对照当前代码核实。

### 记忆系统的设计哲学

MEMORY.md 全量注入保证索引始终可见，具体记忆文件通过 Sonnet sideQuery 按相关性筛选注入，避免每次对话塞入大量过时信息占用 token。四种记忆类型清晰区分了"用户是谁"（user）、"怎么工作"（feedback）、"做什么"（project）、"在哪里"（reference）四个维度。

---

## 维度五：Provider / LLM 绑定

### 硬绑定 Anthropic

Claude Code 只使用 Anthropic 的 Claude 模型，通过官方 SDK 调用。

### 系统提示的构建

系统提示由多个来源组合而成（`QueryEngine.ts`）：

```
最终系统提示 = [
  默认系统提示     ← 内置的 Claude Code 行为规范
  记忆摘要文本     ← 当前会话相关的持久记忆（可选）
  用户追加提示     ← --append-system-prompt 参数（可选）
]
```

### 高级特性支持

- **扩展思考（Extended Thinking）**：`thinking.type = "adaptive"` 让模型自主决定是否花时间深思
- **Prompt Caching**：在消息列表中插入 `cache_control: {type: "ephemeral"}` 标记，命中缓存可节省最多 90% 的输入 token 费用
- **结构化输出**：支持 JSON Schema 约束，确保模型输出可解析
- **流式处理**：通过 SSE 流式接收响应，逐块渲染给用户

---

## 维度六：状态管理

### 全局状态（单例）

`src/bootstrap/state.ts` 维护一个全局状态对象，包含：

```
State {
  totalCostUSD: number          // 本次进程的累计费用
  modelUsage: {...}             // 各模型的 token 用量统计
  sessionId: string             // 当前会话 ID
  invokedSkills: Map<...>       // 本次会话调用过的技能
  sessionCronTasks: [...]       // 已调度的定时任务
  isRemoteMode: boolean         // 是否远程模式
  lastAPIRequest: {...}         // 调试用：最后一次 API 请求内容
}
```

### 会话历史持久化

对话历史以 JSONL 格式写到磁盘（`~/.claude/projects/<sanitized-cwd-path>/<session-id>.jsonl`）：

```jsonl
{"type":"user","message":{"role":"user","content":"分析 main.ts"},"uuid":"...","timestamp":1234}
{"type":"assistant","message":{"role":"assistant","content":[...]},"uuid":"...","stop_reason":"end_turn"}
{"type":"system","subtype":"compact_boundary","compactMetadata":{...}}
```

**关键特性**：
- **尽早写入**：用户输入一到就写，防止崩溃丢失
- **追加写入**：不覆盖，每条消息独立一行，支持恢复
- **消息类型完整**：包含系统事件（压缩边界）和附件，不只是对话文本

### 消息数据结构

```
Message（联合类型）：
  UserMessage      - 用户输入或工具结果
  AssistantMessage - LLM 响应（含工具调用块）
  SystemMessage    - 系统事件（压缩边界、API 错误等）
  AttachmentMessage - 结构化数据（任务输出、进度信息等）

每条消息有：
  uuid: 全局唯一 ID（用于压缩边界的 preservedSegment）
  timestamp: 创建时间
  isMeta: 是否为内部合成消息（不展示给用户）
```

---

## 架构总结

| 维度 | Claude Code 的选择 |
|------|-------------------|
| Agent Loop | 两层生成器（QueryEngine + query.ts），会话级 + 轮次级分离 |
| 工具系统 | 40+ 工具，三类（原子/组合/元），三层权限降级 |
| 上下文压缩 | LLM 摘要 + 后压缩恢复，固定 buffer（13K tokens）触发，非百分比阈值 |
| 记忆系统 | 文件系统（MEMORY.md），四种类型，按相关性注入 |
| LLM 绑定 | 硬绑 Anthropic Claude，支持 thinking/caching/streaming |
| 状态管理 | 全局单例 + JSONL 追加写入，尽早持久化 |

**设计哲学**：面向"长对话持续工作"场景，上下文压缩和记忆系统不是附加功能，而是核心设计考量。权限系统通过分层降级在安全性和易用性之间取得平衡。

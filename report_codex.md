# Codex Harness 深度分析报告

> 源码路径：`D:\codex\codex-rs\`
> 语言：Rust（核心） + TypeScript（CLI 包装）
> 版本：@openai/codex 0.0.0-dev

> **说明**：本报告全程使用通俗语言解释 Rust 实现，不依赖 Rust 语法背景。遇到 Rust 特有概念会用中文类比解释。

---

## Rust 基础概念速查（读报告前先看这里）

| Rust 概念 | 中文理解 |
|-----------|---------|
| `async/await` | 和 JavaScript 一样，表示异步等待 |
| `Arc<T>` | 共享指针，多个地方可以同时持有同一个对象的引用 |
| `Mutex<T>` | 锁，同一时间只有一个人能读写这个值 |
| `RwLock<T>` | 读写锁，多人可同时读，但写时独占 |
| `enum` + `match` | 类似其他语言的联合类型，`match` 是模式匹配（类似 switch） |
| `trait` | 接口（interface），定义一组必须实现的方法 |
| `Result<T>` | 表示"可能成功（T）或失败（Error）"的返回值 |
| `Option<T>` | 表示"可能有值（T）或没有值（None）"，类似 null 的安全版本 |
| `tokio` | Rust 的异步运行时，类似 Node.js 的事件循环 |

---

## 总体架构一句话

Codex 以 **Session** 为核心容器，通过事件驱动的 Turn 循环管理 LLM 调用与工具执行，记忆系统采用"两阶段提取写入 + 启动时读入"模式，上下文压缩基于 Memento 模式（摘要替换历史）。

---

## 维度一：Agent Loop（控制循环）

### 整体架构：生产者 - 消费者模型

Codex 的通信模型不是直接函数调用，而是通过**消息通道**（channel）传递：

```
用户 / 外部代码
     │ 发送 Op（操作）
     ↓
  [通道入口 tx_sub]
          │
          ↓
     Codex 内部
     会话处理循环
          │
     [通道出口 rx_event]
          │
          ↓
   事件流（Event）
     → 推送给调用方
```

就像点菜（Op）和出菜（Event）分开两个窗口，互不阻塞。

### Session（会话）的数据结构

Session 是 Codex 中最大的状态容器，包含：

```
Session {
  conversation_id         // 会话唯一 ID（= ThreadId）
  
  model_client           // API 客户端（每个会话一个）
  skills_manager         // 技能管理器
  plugins_manager        // 插件管理器
  mcp_connection_manager // MCP 工具连接池
  
  context_manager        // 对话历史管理器（重要！）
    └─ items: 所有消息的有序列表
    └─ token_info: 当前 token 用量信息
    └─ reference_context_item: 上一轮注入的上下文快照（用于 diff）
  
  active_turn            // 当前活跃的 Turn 状态（如有）
  thread_config          // 当前配置（模型、权限、工作目录等）
  
  tx_event               // 事件推送通道
  thread_store           // 持久化存储接口
  pending_input          // 排队等待处理的用户输入
}
```

### Turn（轮次）的完整流程

**一个 Turn = 从用户提交一条消息到 LLM 完全停止回答的整个过程**（可能包含多次工具调用）。

```
用户提交 Op::UserMessage("帮我创建一个 Node.js 服务器")
                  ↓
            run_turn() 开始
                  ↓
[步骤 1] 预压缩检查：
  估算 token 数量
  如果 > 模型上下文限制的一定比例 → 先执行压缩
                  ↓
[步骤 2] 初始化：
  从 context_manager 克隆当前历史
  准备本轮的 model_client_session（可复用 WebSocket 连接）
                  ↓
[步骤 3] 进入采样循环（可能循环多次）：
  ┌─────────────────────────────────────────────────────────┐
  │ 拉取排队中的新消息（可能有用户中途追加的输入）          │
  │                                                         │
  │ 构建 Prompt：                                          │
  │   history + 系统指令 + 工具列表 + 元数据               │
  │                   ↓                                     │
  │ 调用 model.stream(prompt) → 流式接收响应                │
  │   ↓               ↓               ↓                    │
  │ 文本内容      工具调用块       推理内容                 │
  │ 实时推送给    收集并行执行     保存（若启用思考模式）    │
  │ 用户                                                    │
  │                   ↓                                     │
  │ 并行执行所有工具调用：                                  │
  │   ① 权限检查（可能等待用户审批）                       │
  │   ② 执行工具                                           │
  │   ③ 收集结果                                           │
  │                   ↓                                     │
  │ 把工具结果追加到历史                                    │
  │                   ↓                                     │
  │ 如果有工具调用 → 继续循环（重新发给模型）               │
  │ 如果 stop_reason == "end_turn" → 跳出循环              │
  └─────────────────────────────────────────────────────────┘
                  ↓
[步骤 4] Turn 完成：
  将本轮所有 Items 持久化到 thread_store
  更新 token 用量统计
  推送 TurnCompletedEvent 给调用方
```

### Active Turn 的状态机

Turn 的状态也是管理的：

```
初始状态：None（无活跃 Turn）

用户提交 → Running {
  turn_id: "turn-123",
  started_at: 时间戳,
  pending_requests: 等待审批的权限请求列表
}

Turn 完成 → None（再次回到空闲）
Turn 中断 → None + emit TurnAbortedEvent
```

### 关键设计：WebSocket 连接复用

Codex 对同一 Turn 内的多次 API 调用（每次工具结果后都要重新调用模型）复用同一个 WebSocket 连接：

```
Turn 内第一次调用：创建新 WebSocket 连接
Turn 内第二次调用（工具结果）：复用同一连接
Turn 内第三次调用（再次工具结果）：复用同一连接
Turn 结束后：缓存连接供下一 Turn 使用
```

**为什么这样做**：减少 TLS 握手开销，同时利用"sticky routing"——确保同一 Turn 的所有请求都被路由到同一台 GPU 服务器（服务器端有 KV cache，可节省 token 计算）。

---

## 维度二：Tool System（工具系统）

### 四类工具

| 类型 | 描述 | 例子 |
|------|------|------|
| Function Tools | 内置功能工具，编译进代码 | `shell`（执行命令）、`write_file` |
| MCP Tools | 通过 MCP 协议动态连接的外部服务 | GitHub、Jira、Slack |
| App Tools | 内置 Apps 服务（需 OAuth 认证） | Google Drive、Salesforce |
| Skill Tools | 用户/社区自定义的 YAML 脚本 | 自定义工作流程 |

### 工具接口（所有工具都要实现）

```
ToolHandler 接口（用中文描述）：
  tool_name()         → 工具唯一名称
  spec()              → 给 LLM 的说明（名称 + 描述 + 参数 Schema）
  is_mutating(args)   → 这次调用会修改文件/环境吗？（影响权限检查）
  handle(invocation)  → 实际执行工具，返回结果
```

### 工具注册与发现

```
系统启动时：
  ① 内置工具：直接注册到 ToolRegistry（HashMap 结构）
  ② MCP 工具：启动 MCP 进程 → 握手 → 调用 list_tools() → 注册到 Registry
  ③ Skills：扫描 ~/.codex/skills/ 目录 → 解析 YAML → 编译注册

Turn 执行时：
  工具注册表是只读的，直接查找即可
```

### 工具调用的完整生命周期

```
模型返回 tool_call JSON：
  { "id": "call_1", "name": "shell", "arguments": {"command": "ls"} }
                  ↓
[Tool Router 解析]
  识别工具名 → 查找 ToolRegistry → 获取 ToolHandler
  构造 ToolInvocation（包含 call_id、工具名、参数、Session 引用）
                  ↓
[权限检查阶段]
  查询 ExecPolicy（执行策略）和 SandboxPolicy（沙箱策略）
  判断是否需要用户审批：
    → 如果配置了 "always allow" → 直接通过
    → 如果需要审批 → 发送 ApprovalRequest 事件，等待用户响应
                  ↓
[Pre-Tool Hook]（可选）
  运行用户配置的钩子脚本
  钩子可以修改参数或阻止执行
                  ↓
[执行工具]
  handler.handle(invocation)
  工具在沙箱环境中运行（文件系统和网络都受限）
  返回 ToolOutput（字符串）
                  ↓
[Post-Tool Hook]（可选）
  运行执行后钩子（审计日志等）
                  ↓
[构建结果]
  成功 → FunctionCallOutput { call_id, content }
  失败 → 包含 error 的 FunctionCallOutput
  超时 → 超时标记
                  ↓
[追加到历史，模型继续采样]
```

### 并行工具执行

Codex 支持同时执行多个工具（当模型一次返回多个 tool_call 时）：

```
模型返回 [call_1: "read file A", call_2: "read file B", call_3: "grep C"]
                  ↓
并行启动：
  任务 1 → 读文件 A（同时执行）
  任务 2 → 读文件 B（同时执行）
  任务 3 → grep C（同时执行）
                  ↓
等待全部完成（或超时）
                  ↓
将三个结果打包，一起发回模型
```

最大并发数可配置（如最多同时 4 个工具），防止资源耗尽。

---

## 维度三：Context Compaction（上下文压缩）★

### 触发时机

```
Turn 开始时（run_turn 的第一步）：
  
  估算 token 数量：
    精确值：上一次 API 响应中返回的 usage.total_tokens
    估算增量：本轮新增消息的字符数 ÷ 4（近似 token 数）
  
  如果 估算总 token > 模型的 auto_compact_token_limit（如 150K）：
    trigger = Auto
    reason = PreSamplingFull
    执行压缩
```

### 压缩策略：Memento 模式

Codex 的压缩方式叫 **Memento 模式**，核心思想是：
- **保留**：所有用户消息的骨架（用户说了什么）
- **删除**：所有模型回复（模型之前说了什么被替换为摘要）
- **摘要**：把整段对话压缩成一条"摘要消息"，放在历史末尾

```
压缩流程（伪代码）：

① 准备压缩输入：
  把当前所有历史消息拿出来
  
② 构建压缩 Prompt：
  system: "你是压缩助手，总结对话历史"
  包含所有用户消息 + 要求生成摘要
  
③ 模型生成摘要：
  "在这个会话中，用户要求构建 Node.js 服务器。
   我们已经：
   1. 创建了 server.js（基础 HTTP 服务器）
   2. 添加了 logger.js（结构化日志）
   当前状态：服务器可以运行，日志记录到 logs/ 目录"
   
④ 构建新的历史：
  [initial_context]                          ← 完整系统指令重新注入
  [user_message_1: "帮我创建服务器"]          ← 保留所有用户消息
  [user_message_2: "添加日志"]               ← 保留
  [user_message_3: "Memento: [摘要内容]"]    ← 最后加入摘要消息
  
  注意：所有旧的模型回复都被删除！
  
⑤ 替换 context_manager 中的历史
⑥ 重新计算 token 用量
```

### 压缩前后对比

**压缩前**（假设 5000 tokens）：
```
历史：
  User: "帮我创建服务器"
  Assistant: "好的，我来创建..." + [工具调用: write_file server.js]
  User: [工具结果: 文件创建成功]
  Assistant: "文件创建完成..."
  User: "添加日志"
  Assistant: "好的..." + [工具调用: write_file logger.js]
  User: [工具结果: 成功]
  Assistant: "已添加日志系统..."
  
约 5000 tokens
```

**压缩后**（约 1500 tokens）：
```
历史：
  [初始上下文：系统指令 + 环境信息]
  User: "帮我创建服务器"
  User: "添加日志"
  User: "Memento: 我们创建了 server.js 和 logger.js..."
  
约 1500 tokens（节省 70%）
```

### Token 计数的两层策略

Codex 用了一个聪明的混合策略：

```
精确值（来自 API 响应）：
  每次 API 调用后，响应头包含 usage.total_tokens
  → 更新 last_api_response_total_tokens
  → 这是当前最精确的 token 用量基准

估算值（本地计算）：
  新增消息时，用 字符数 ÷ 4 估算 token 数
  → 累加到 estimated_tokens_since_last_api_response
  → 用于快速判断：继续对话是否有超限风险

总估算 = 精确值 + 估算增量
```

好处：不需要每条消息都调用 API 计数（太慢），用本地估算 + 周期校准，快速而足够准确。

### 远程压缩（特殊模式）

Codex 还支持调用专门的 `/responses/compact` API 端点进行压缩，这个端点使用更强的模型生成更好的摘要。本地压缩是 fallback。

---

## 维度四：Memory System（记忆系统）★

### 两个独立模块的职责分工

| 模块 | 路径 | 职责 |
|------|------|------|
| `thread-store` | `codex-rs/thread-store/` | 存储单个会话的完整对话历史（短期，按会话 ID 查询） |
| `memories` | `codex-rs/memories/` | 存储跨会话的用户知识摘要（长期，启动时注入） |

### Thread Store：会话历史持久化

```
存储位置：
  ~/.codex/threads/{thread_id}/
    ├── metadata.json     ← 元数据（会话 ID、创建时间等）
    └── rollout.msgpack   ← 二进制格式的所有 Items

存储的 Items 包括：
  TurnStartedEvent     ← 一次 Turn 开始
  UserMessageItem      ← 用户消息
  AgentMessageItem     ← 模型回复
  ToolCallItem         ← 工具调用
  ToolResultItem       ← 工具结果
  TurnCompletedEvent   ← Turn 结束（含 token 用量等）

写入时机：每次 Turn 完成后，追加所有本轮 Items
读取场景：用户恢复已有会话时，加载历史
```

msgpack 是二进制格式，比 JSON 更省空间，读写更快。

### Memories：跨会话长期记忆

这是 Codex 最有特色的记忆系统——**两阶段异步写入**。

#### 写入路径（后台异步，不阻塞用户）

**阶段 1：提取（快速粗糙）**

```
触发时机：会话结束后，后台静默运行

处理流程：
  ① 从 thread-store 读取当前会话的所有历史
  ② 使用轻量模型（如 gpt-mini）快速处理
  ③ 生成"初步提取（extraction）"
  ④ 保存到 ~/.codex/memories/rollout_summaries/{thread_id}.md

目的：快速、粗糙地抓取可能有价值的信息
```

**阶段 2：编辑整合（精细处理）**

```
触发时机：在阶段 1 之后，或定期运行

处理流程：
  ① 读取所有 rollout_summaries/ 下的文件
  ② 使用强力模型（如 gpt-5）深度处理
  ③ 消除重复、整理归类、优化表述
  ④ 覆盖写入 ~/.codex/memories/raw_memories.md

目的：生成高质量、有组织的长期记忆
```

#### 读取路径（Turn 开始时注入）

```
Turn 初始化时：
  ① 读取 ~/.codex/memories/raw_memories.md
  ② 追加到初始上下文（initial_context）中
  ③ 模型在生成回答时可以参考
  
注入格式（直接文本，无特殊结构）：
  # 用户信息
  - 姓名：Alice，公司：Acme Corp
  
  # 项目上下文
  - 当前项目：WebApp v2.0，技术栈：Node.js + React
  - 上次会话进度：已完成用户认证模块
  
  # 偏好设置
  - 更倾向于编写测试用例
  - 喜欢简洁的代码风格
```

### 记忆扩展（Extensions）

Codex 支持从外部系统（GitHub、Jira 等）补充记忆：

```
~/.codex/memories/extensions/
  github/
    instructions.md    ← 告诉模型如何解读这些数据
    recent_repos.md    ← 用户最近的 GitHub 仓库（外部脚本定期同步）
  jira/
    instructions.md
    open_issues.md    ← 当前未解决的问题列表

处理规则（来自 Phase 2）：
  → 根据 instructions.md 解释扩展数据
  → 融合到主记忆文件中
  → 如果扩展资源被删除，从记忆中也删除相关条目
```

### 两种记忆类型的对比

| 特性 | thread-store（会话历史） | memories（长期记忆） |
|------|------------------------|---------------------|
| 生命周期 | 按会话 ID 存储，永久保留 | 跨会话积累，持续更新 |
| 存储格式 | 二进制（msgpack） | Markdown 文本 |
| 写入时机 | 每个 Turn 完成后即时写入 | 异步后台写入（两阶段） |
| 读取时机 | 恢复旧会话时 | 每次 Turn 启动时 |
| 颗粒度 | 完整原始数据 | 提炼后的摘要 |
| 可被用户编辑 | 否（内部格式） | 是（纯文本 Markdown） |

---

## 维度五：Provider / LLM 绑定

### 硬绑定 OpenAI

Codex 使用 OpenAI 的 Responses API（而不是 Chat Completions API），通过 WebSocket 进行流式通信。

### 系统提示与消息格式

Codex 将所有内容组织成单个"用户消息"（而不是 system 字段）：

```json
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "【系统指令：你是 Codex，一个能读写代码的 AI 助手...】\n\n【当前环境：工作目录 /home/user/project，Shell: bash...】\n\n【可用工具：write_file, shell, read_file...】\n\n【用户记忆：\n上次进度：实现了路由...\n用户偏好：简洁代码风格...】"
    },
    {
      "type": "text",
      "text": "帮我创建一个 Node.js 服务器"
    }
  ]
}
```

注意：系统指令、环境信息、记忆都被**拼接在用户消息中**，而不是放在独立的 system 字段。

### 渐进式上下文注入（差异注入）

这是 Codex 的一个优化：

```
第一次 Turn：
  注入完整初始上下文（系统指令 + 环境 + 工具 + 记忆）
  保存快照 reference_context_item = 本轮上下文内容
  
后续 Turn：
  计算差异：当前配置 vs reference_context_item
  只注入变化的部分（如工作目录改变了 → 只发新工作目录）
  如果没变化 → 不重复注入
  更新 reference_context_item
  
效果：避免每轮重复发送相同的大段系统指令，节省 token
```

---

## 维度六：状态管理

### 配置的分层更新

Codex 的配置可以在多个层级设置，后面的层级覆盖前面的：

```
全局配置（~/.codex/settings.json）
       ↓ 覆盖
线程级配置（通过 SessionSettingsUpdate）
       ↓ 覆盖
Turn 级临时调整（用户在 UI 中修改）

最终有效配置 = 取最严格的那个（安全方面）
```

### 配置变更的追踪

每次配置发生变化时，Codex 会发出一个 `TurnDiffEvent`，记录变更前后的差异。这让整个配置历史可追溯。

### 权限的分层设计

```
全局沙箱级别：
  open     → 允许大部分操作
  moderate → 需要审批写操作
  strict   → 需要审批所有操作

工具级别策略（在沙箱基础上叠加）：
  Shell 命令还有额外的命令黑名单
  MCP 工具可能需要 OAuth 认证

最终策略 = 取交集（两个约束都满足才允许）
```

### 会话配置数据结构

```
ThreadConfigSnapshot {
  model: "gpt-5.4",
  permission_profile: PermissionProfile,
  cwd: "/home/user/project",
  shell: "/bin/bash",
  reasoning_effort: "medium",   // 推理深度
  personality: "concise",       // 回答风格
}
```

---

## 架构总结

| 维度 | Codex 的选择 |
|------|------------|
| Agent Loop | Session + Turn 两层，通道驱动的事件模型，WebSocket 连接复用 |
| 工具系统 | 四类（内置/MCP/Apps/Skills），并行执行，pre/post hook |
| 上下文压缩 | Memento 模式：保留用户消息骨架，删除模型回复，生成摘要 |
| 记忆系统 | 两模块：thread-store（精确历史）+ memories（异步提炼的长期知识） |
| LLM 绑定 | 硬绑 OpenAI Responses API，WebSocket 流式，差异注入上下文 |
| 状态管理 | 多层配置覆盖，Turn 状态机，配置变更追踪 |

**设计哲学**：强调**性能和安全**——WebSocket 连接复用、差异注入、并行工具执行都是性能优化；多层权限、沙箱政策、工具钩子则是安全保障。两阶段记忆提取体现了"质量比速度重要"的记忆设计思路。

# 三款 Harness 横向比较报告

> 比较对象：Claude Code · Codex · OpenCode
> 重点维度：上下文压缩 × 记忆系统
> 参考框架：Inside the Scaffold Taxonomy（12 维度 × 3 层）

---

## 一张表看全局

| 维度 | Claude Code | Codex | OpenCode |
|------|------------|-------|---------|
| **实现语言** | TypeScript | Rust + TypeScript | TypeScript（Effect 框架） |
| **LLM 绑定** | Anthropic 专属 | OpenAI 专属 | 20+ 提供商（Vercel AI SDK） |
| **控制循环** | 两层生成器（QueryEngine + query.ts） | Session + Turn（通道驱动） | Effect 流水线 + Runner |
| **工具数量** | 40+（三类：原子/组合/元） | 4 类（内置/MCP/Apps/Skills） | 17 个内置工具 |
| **工具权限** | 三层降级（配置→分类器→用户） | 三层叠加（全局→线程→工具级） | Wildcard 规则集（首匹配） |
| **上下文压缩** | LLM 摘要 + 后压缩恢复 | Memento 模式 | 两阶段（修剪工具输出 + 摘要） |
| **压缩触发** | 85% 阈值，多轮警告后自动 | Turn 启动前检测，预防式 | 每步检测，流式中断 |
| **记忆存储** | 文件系统（MEMORY.md） | 两模块：msgpack + Markdown | SQLite 数据库 |
| **记忆类型** | 4 类（user/feedback/project/reference） | 无类型，自由文本 | 无独立摘要记忆，完整历史 |
| **记忆写入** | 对话中检测自动写入 | 会话后台异步两阶段提取 | 实时流式写入数据库 |
| **跨会话记忆** | 是（MEMORY.md 持久化） | 是（raw_memories.md） | 是（SQLite 完整历史） |
| **执行隔离** | 权限三层降级 + hooks | 多层沙箱策略 + 审批流 | Wildcard 规则集 |
| **状态管理** | JSONL 追加写入 | msgpack 二进制写入 | SQLite 实时写入 |
| **MCP 支持** | 是（作为工具之一） | 是（一类工具） | 是（原生一等公民） |
| **插件/技能** | Skill + MCP | Plugins + Skills | Plugin + Skill 双层 |

---

## 重点维度一：上下文压缩

### 三者的触发时机对比

```
Claude Code：
  "我还有余量时就提前准备"
  
  实时监控 token 用量
  到达 85% → 进入 WARNING 状态
  连续多轮 WARNING → 自动触发压缩
  
  特点：有缓冲期，用户感知较少

─────────────────────────────────────────

Codex：
  "每次开始新轮次时先检查"
  
  Turn 启动时：
    估算 token = 精确历史 token + 本轮新增估算
    如果 > auto_compact_token_limit → 先压缩再开始
  
  特点：预防式，永不在处理中途压缩

─────────────────────────────────────────

OpenCode：
  "处理过程中随时检测，可以立即叫停"
  
  每次 LLM 生成一步（finish-step）后：
    检查 isOverflow()
    如果超载 → 设置 needsCompaction = true
    Stream.takeUntil() 立即中断 LLM 流
  
  特点：最细粒度，甚至可以在模型回答到一半时中断
```

### 三者的压缩策略对比（核心差异）

#### Claude Code：LLM 摘要 + 精心的后压缩恢复

```
策略哲学："全面摘要 + 确保模型有足够上下文继续工作"

压缩流程：
  1. 剥离图片（防止压缩请求本身超限）
  2. 发给 LLM，要求生成三段式摘要：
     <summary>     技术决策和操作记录
     <user_context>  用户背景和偏好
     <project_context> 项目约束和进度
  3. 创建"压缩边界消息"，保存摘要元数据
  4. 后压缩恢复（这是 Claude Code 独有的）：
     → 重新读取最近编辑的 5 个文件
     → 重新注入最近调用的技能内容
     → 重新注入 MCP 工具说明
  5. 释放边界之前的所有消息（GC）

摘要格式示例：
  <summary>
    分析了 main.ts 的错误处理，发现 try/catch 缺失
    完成了 api.ts 的重构（新增 3 个端点）
  </summary>
  <user_context>
    用户是 Go 专家，React 新手，偏好简洁解释
  </user_context>
  <project_context>
    TypeScript API 服务，需要 2026-05-20 前完成，不能引入新依赖
  </project_context>
```

**优点**：摘要结构清晰，区分了技术信息和用户上下文；后压缩恢复确保模型不"忘记"当前文件状态。

**缺点**：后压缩恢复需要重新读取文件，增加延迟；摘要中可能丢失细节。

---

#### Codex：Memento 模式（骨架保留）

```
策略哲学："保留用户说了什么，用摘要替换模型说了什么"

压缩流程：
  1. 把当前历史中的所有用户消息列出来
  2. 发给 LLM 生成一段摘要（自由文本，无固定结构）
  3. 重建历史：
     [initial_context]           ← 完整系统指令重新注入
     [user_message_1]            ← 保留所有用户原话
     [user_message_2]            ← 保留
     ...
     [user_message_N]            ← 保留
     [Memento: 摘要文本]          ← 追加在末尾

注意：所有旧的模型回复全部删除！

摘要格式（自由文本）：
  "用户要求构建 Node.js 服务器并添加日志系统。
   我们已经创建了 server.js（基础 HTTP）和 logger.js（Winston）。
   服务器运行在 3000 端口，日志写入 logs/ 目录。"
```

**优点**：保留用户意图（所有用户消息），摘要简洁；initial_context 重新注入确保指令完整。

**缺点**：模型的中间推理过程全部丢失；摘要无固定结构，依赖模型自由发挥质量。

---

#### OpenCode：两阶段（修剪 + 摘要）

```
策略哲学："先精准删除最没价值的部分，真撑不住了再摘要"

压缩流程（两阶段）：

第一阶段：修剪工具输出
  → 保留工具调用的结构（LLM 知道做过什么）
  → 删除工具的输出内容（大量 token 的来源）
  → 保护：最近 2 轮、skill 工具、最新 40K tokens
  → 只有释放量 ≥ 20K tokens 才值得修剪
  
  效果：工具骨架还在，但内容消失
    Before: ToolPart { output: "ls -la 的 2000 行输出..." }
    After:  ToolPart { output: null, compacted: true }

第二阶段：Markdown 结构化摘要（若第一阶段不够）
  → 移除图片/视频
  → 用固定模板生成摘要：
  
  ## Goal（目标）
  ## Instructions（指令）
  ## Discoveries（发现）
  ## Accomplished（已完成）
  ## Relevant files / directories（相关文件）

压缩后消息作为 Assistant 消息保留在历史中（mode="compaction"）
```

**优点**：两阶段策略精准——先删最没用的，再求助摘要；Markdown 模板结构清晰便于阅读；流式中断让压缩更及时。

**缺点**：两阶段增加了复杂度；修剪后工具输出消失，模型可能困惑"我做了什么操作但结果在哪"。

### 压缩策略的本质差异（一句话总结）

```
Claude Code = "摘要 + 文件恢复"
  核心：不信任摘要，所以额外恢复关键文件状态

Codex = "用户骨架 + 自由摘要"
  核心：用户意图最重要，保留所有用户消息

OpenCode = "精准修剪 + 结构摘要"
  核心：先精确删除没价值的部分，再结构化摘要
```

### 论文的 Prevention vs Cure 分类

根据论文（10_Inside_the_Scaffold_Taxonomy.pdf）的框架，三款工具都属于 **Cure（治疗）** 哲学——即允许上下文接近或达到限制，然后再处理，而非提前设计避免超限。

但三者的触发时机各有侧重：
- Claude Code：偏向 **缓冲式 Cure**（85% 时开始准备）
- Codex：偏向 **预防式 Cure**（每轮开始前检测，在超限之前处理）
- OpenCode：偏向 **即时 Cure**（实时检测，随时中断）

---

## 重点维度二：记忆系统

### 记忆存储介质对比

```
Claude Code → 文件系统（Markdown 文件）
─────────────────────────────────────────
~/.claude/.memdir/private-memory/MEMORY.md

格式：
---
name: 记忆名称
description: 描述
type: user | feedback | project | reference
---
记忆内容（自由文本 Markdown）

特点：
  ✓ 人类可读，用户可以手动编辑
  ✓ 结构化 frontmatter 支持类型过滤
  ✓ 支持团队共享（team-memory/ 目录）
  ✗ 无法做复杂查询（只能全文搜索）
  ✗ 并发写入需要文件锁
```

```
Codex → 两种格式（会话 msgpack + 记忆 Markdown）
─────────────────────────────────────────
会话历史：~/.codex/threads/{id}/rollout.msgpack（二进制）
长期记忆：~/.codex/memories/raw_memories.md（Markdown）

特点（会话历史）：
  ✓ 二进制格式，读写快，空间省
  ✗ 不可人工阅读，需要工具查看
  
特点（长期记忆）：
  ✓ 自由文本，无固定结构，灵活
  ✓ 用户可手动编辑
  ✓ 支持扩展（extensions/ 目录接入外部数据）
  ✗ 无类型区分（不如 Claude Code 的四象限分类）
```

```
OpenCode → SQLite 数据库
─────────────────────────────────────────
~/.local/share/opencode/data.db（或类似路径）

表结构：session / message / part 三张表
所有内容都以 JSON 字段存在 part 表的 data 列中

特点：
  ✓ 关系型查询（按时间、按类型、按会话过滤）
  ✓ 增量更新（不需要读整行再写回）
  ✓ 事务保证（不会出现写一半的情况）
  ✓ 级联删除（删会话自动删所有消息和部分）
  ✗ 不可人工阅读，需要 SQLite 工具查看
  ✗ 没有独立的"提炼摘要"记忆层
```

### 三者的记忆类型对比

**Claude Code：四象限分类系统**

```
user（用户维度）
  → 谁在用？有什么技能背景？什么偏好？
  → "用户是 Go 专家，React 新手"
  
feedback（工作方式维度）
  → 怎么工作？要做什么？要避免什么？
  → "测试必须用真实数据库，不要 mock"
  → 格式：规则 + Why + How to apply
  
project（项目维度）
  → 在做什么？什么时候交？有什么约束？
  → "2026-05-20 后冻结合并"
  
reference（定位维度）
  → 在哪里找信息？
  → "bug 在 Linear 项目 INGEST 中追踪"
```

这四种类型的划分，其实回答了四个问题：**Who / How / What / Where**。

**Codex：自由文本，无强制分类**

```
raw_memories.md 没有类型约束，自由文本：

# 用户信息
- 姓名：Alice，公司：Acme Corp

# 项目上下文
- 当前项目：WebApp v2.0
- 上次进度：已完成用户认证模块

# 偏好设置
- 喜欢简洁代码

优点：灵活，不被类型约束
缺点：结构依赖 LLM 自动整理，质量不稳定
```

**OpenCode：无独立摘要记忆**

```
OpenCode 没有"摘要性记忆文件"的概念。
记忆 = 完整的数据库历史。

"记忆"体现在：
  - 每条消息的完整 Parts（工具调用、文本、推理）
  - 会话元数据（代码变更统计、git 快照）
  - 会话间的父子关系

优点：精确，不丢失细节
缺点：随时间增长，依赖压缩机制控制规模
```

### 记忆写入时机对比

```
Claude Code：
  对话进行中，系统自动检测可保存信息
  
  检测规则：
    ✓ 用户透露了背景信息 → 保存到 user 类型
    ✓ 用户纠正了某个做法 → 保存到 feedback 类型
    ✓ 提到了截止日期 → 保存到 project 类型
    ✗ 代码模式、架构 → 不保存（可从代码推导）
    ✗ 当前临时任务状态 → 不保存
  
  写入方式：append 到 MEMORY.md 文件

─────────────────────────────────────────

Codex：
  会话结束后，后台异步运行（不阻塞用户）
  
  Phase 1（快，用轻量模型）：
    从历史中粗略提取候选记忆
    保存到 rollout_summaries/
    
  Phase 2（慢，用强力模型）：
    读取所有候选记忆
    整理、去重、优化
    覆盖写入 raw_memories.md
  
  写入方式：覆盖写 Markdown 文件

─────────────────────────────────────────

OpenCode：
  实时，每个 LLM 事件立即写数据库
  
  text-start → 创建 TextPart（空）
  text-delta → 数据库 json_set() 追加文本
  tool-call  → 更新 ToolPart 状态
  
  写入方式：SQLite 增量更新
```

### 记忆注入方式对比

```
Claude Code：
  按相关性过滤 → 组装 Markdown 文本 → 插入系统提示
  
  注入格式：
  # 相关记忆
  ## User memories
  - 用户是 Go 专家，React 新手
  ## Feedback memories
  - 集成测试用真实数据库
  
  特点：过滤后注入，token 效率高；相关性判断可能不准

─────────────────────────────────────────

Codex：
  全量注入 → 直接追加在初始上下文末尾
  
  注入格式：
  # 用户信息
  - 姓名：Alice
  # 项目上下文
  ...（raw_memories.md 全文）
  
  特点：全量注入，简单可靠；记忆越多占用越多 token

─────────────────────────────────────────

OpenCode：
  历史全量读取 → 转换为 ModelMessage[] → 直接作为对话历史传入
  
  不是在系统提示里插入摘要，而是把整段历史发给 LLM
  
  特点：精确（没有信息丢失）；token 消耗高（依赖压缩控制）
```

### 记忆系统的本质差异（一句话总结）

```
Claude Code = "有选择地记忆"
  挑出重要的、持久的信息，分类存储，按需注入

Codex = "事后提炼记忆"
  先完整记录，后台慢慢提炼，下次用摘要

OpenCode = "完整保存，压缩控制"
  什么都存，历史即记忆，用压缩保持规模
```

---

## 其他维度的对比

### 控制循环（Agent Loop）

三者都采用 ReAct 模式（推理 → 行动 → 观察），但实现机制不同：

| 方面 | Claude Code | Codex | OpenCode |
|------|------------|-------|---------|
| 循环机制 | async generator（yield） | Tokio async + 通道 | Effect Stream |
| 工具调用 | 顺序执行（默认） | 并行执行（可配置） | 顺序执行 |
| 循环终止 | stop_reason == end_turn | stop_reason == end_turn | finish-step + 无更多工具 |
| 中断支持 | 是（generator.return()） | 是（Abort 信号） | 是（Runner.cancel()） |

**最大差异**：Codex 支持**并行工具执行**（模型同时返回多个工具调用，Codex 并行跑），Claude Code 和 OpenCode 默认顺序执行。

### 工具集设计

| 方面 | Claude Code | Codex | OpenCode |
|------|------------|-------|---------|
| 内置工具数 | 40+ | ~10 个核心工具 | 17 个 |
| 组合工具 | 有（AgentTool 可嵌套 Agent） | 无 | 无 |
| 任务管理工具 | 有（Task CRUD） | 无 | 有（task 工具） |
| LSP 支持 | 无 | 无 | 有（lsp 工具） |
| MCP 集成 | 作为普通工具之一 | 作为一类工具 | 原生一等公民 |

**最大差异**：Claude Code 的 `AgentTool` 允许启动嵌套 Agent（子 Agent），这是三者中独有的。

### Provider 绑定

| 方面 | Claude Code | Codex | OpenCode |
|------|------------|-------|---------|
| 支持的 LLM | 仅 Anthropic | 仅 OpenAI | 20+ 提供商 |
| 切换 LLM | 不支持 | 不支持 | 配置即可切换 |
| 提供商抽象 | 无 | 无 | Vercel AI SDK 统一接口 |
| streaming | SSE | WebSocket | 统一（SDK 处理） |

**最大差异**：OpenCode 的多 Provider 支持是其最核心的差异化特性，Claude Code 和 Codex 都是各自 LLM 厂商的官方工具，天然绑定。

---

## 论文维度的映射

| 论文维度 | Claude Code | Codex | OpenCode |
|---------|------------|-------|---------|
| **控制循环拓扑** | ReAct（主） | ReAct（主） | ReAct（主） |
| **循环驱动者** | LLM-driven | LLM-driven | LLM-driven |
| **控制流实现** | TypeScript generator | Rust async + channel | Effect Stream |
| **工具集设计** | 40+（分三层） | 4 类（动态注册） | 17 个（模块化） |
| **编辑格式** | string replacement | string replacement | string replacement（edit 工具） |
| **工具发现策略** | 静态 + MCP 动态 | 静态 + MCP 动态 | 静态 + MCP 动态 |
| **上下文检索** | LLM 导航（grep/glob） | LLM 导航 | LLM 导航 + LSP |
| **执行隔离** | 三层权限降级 | 多层沙箱 + 审批 | Wildcard 规则集 |
| **状态管理** | JSONL + 单例状态 | msgpack + Turn 状态机 | SQLite + Effect 状态 |
| **上下文压缩** | LLM 摘要 + 后恢复 | Memento 骨架保留 | 两阶段（修剪+摘要） |
| **多模型路由** | 单模型（Claude） | 单模型（GPT） | 按配置路由（20+） |
| **持久记忆** | 文件 + 四类型 | 两阶段提炼 Markdown | SQLite 完整历史 |

三款工具在**工具发现策略**（静态+MCP动态）、**编辑格式**（字符串替换）、**上下文检索**（LLM导航）上**高度趋同**，体现了行业共识。

**分歧最大的维度**：持久记忆、上下文压缩、Provider 绑定。

---

## 选型视角：各自适合什么场景

```
Claude Code 适合：
  ✓ 重度依赖 Claude 的团队
  ✓ 需要长对话持续工作（压缩+后恢复设计精良）
  ✓ 重视记忆质量（四象限分类，有 Why/How to apply 结构）
  ✓ 需要子 Agent 嵌套执行复杂任务

Codex 适合：
  ✓ 重度依赖 OpenAI 模型的团队
  ✓ 重视性能（Rust、WebSocket 复用、并行工具执行）
  ✓ 有安全合规要求（最完整的沙箱设计）
  ✓ 希望记忆积累质量高（两阶段提炼）

OpenCode 适合：
  ✓ 不想绑定特定 LLM（可随时切换）
  ✓ 需要精确的历史查询和分析（SQLite）
  ✓ 在乎代码变更追踪（git 快照集成）
  ✓ 有多个项目/工作目录需要隔离管理
```

---

## 关键洞察：设计哲学的差异

**面对"如何保持跨会话的连贯性"这个核心问题，三者选择了不同答案**：

```
Claude Code：人工智能 + 精选存储
  → "让 AI 判断什么值得记住，分类保存，按需检索"
  → 核心挑战：AI 可能判断错误，重要信息可能被遗漏

Codex：后台提炼 + 质量优先
  → "先完整记录，事后用更好的模型提炼成精华"
  → 核心挑战：会话结束才提炼，当前会话感受不到

OpenCode：完整保存 + 压缩平衡
  → "什么都存，通过压缩控制规模，历史即记忆"
  → 核心挑战：随时间积累，压缩的代价越来越高
```

**面对"上下文窗口有限"这个工程约束，三者也做出了不同权衡**：

```
Claude Code：信息保留优先
  → 摘要时明确区分三类信息（技术/用户/项目），后压缩恢复文件状态
  → 愿意为质量多消耗几次 API 调用

Codex：用户意图优先
  → 保留所有用户消息骨架，牺牲模型的中间推理
  → 简洁高效，但可能丢失推理链

OpenCode：精确性优先
  → 先修剪"输出"（最占空间但最易再生），再整体摘要
  → 分两步走，每步都有明确的最小损失原则
```

---

## 总结

三款工具在同一领域（AI coding agent），解决同一问题（LLM 有限上下文 + 跨会话连贯性），但因为**价值观不同**（绑定单一 LLM vs 多 LLM、用户体验 vs 性能 vs 灵活性）和**技术选择不同**（TypeScript vs Rust vs Effect框架，文件 vs 二进制 vs 数据库），形成了显著不同的实现路径。

这正是 harness scaffold 研究的价值所在：**表面相似的 ReAct 循环背后，是完全不同的工程决策和设计哲学**。

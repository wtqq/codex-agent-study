# Codex Agent 架构介绍

> 团队内部分享 · 2026-08-04 · 基于 openai/codex 源码分析

---

## 1. 开场：Codex 是什么

Codex 是 OpenAI 的终端级 AI 编程 agent。和 Cursor/Copilot 不同，它不是编辑器的"补全插件"，而是一个**能独立规划、执行工具、读写文件、多轮推理的自治 agent 引擎**。

我们研究它的原因：它是目前架构最完整、代码最公开可读的 agent 系统之一。理解它的设计决策，对我们团队自己设计 agent 有直接的借鉴意义。

---

## 2. 总体架构

### 逻辑架构（分层视图）

```
+- 前端层 --------------+
| TUI | CLI | IDE | SDK |   ← 用户交互入口
+---------+-------------+
          | JSON-RPC 2.0
+- 协议网关层 ----------+
| codex app-server      |   ← 统一网关,所有前端走同一套 API
| Thread/Turn/Item 模型  |
+---------+-------------+
          |
+- 核心引擎层 ----------+
| codex-core            |   ← agent 血液循环系统
| ThreadManager/Session |
| ModelClient/ToolRouter|
| WorldState/Context     |
+---------+-------------+
          |
+- 执行层 --------------+
| ExecServer / Sandbox   |   ← 隔离执行
+------------------------+
```

### 技术栈

| 维度 | 选型 |
|------|------|
| 语言 | **Rust**（agent 引擎），TS/Python 仅为外部 SDK |
| 异步运行时 | Tokio（多线程 work-stealing） |
| 序列化协议 | serde + JSON-RPC 2.0 |
| 持久化 | SQLite + JSONL + Tantivy |
| 模型通信 | OpenAI Responses API（HTTP SSE 流式） |
| 沙箱 | macOS Seatbelt / Windows unelevated / Linux bubblewrap |

### 关键数据：80+ Rust crate，Cargo workspace 管理，入口是单一 `codex` 二进制。

---

## 3. 协议网关：codex app-server（协议网关服务）

**一句话：所有前端对话共享同一套 JSON-RPC 2.0 协议，app-server 是唯一的网关。**

### 三种接入形态

| 前端 | 连接方式 | 适用场景 |
|------|---------|---------|
| TUI / `codex exec` | 进程内 typed channel（不走网络） | 终端日常使用 / CI 自动化 |
| IDE 扩展（VS Code） | 独立进程 `codex app-server` + stdio | 编辑器内对话 |
| Desktop App / SDK | 独立进程 + websocket / unix socket | 桌面体验 / 嵌入应用 |

### 核心数据模型（所有对话都按这个结构走）

```
Thread（会话线程）
  +-- Turn（推理轮次） × N
        +-- Item（对话单元） × N
              |-- userMessage     用户消息
              |-- agentMessage    模型回复
              |-- reasoning       推理过程
              |-- commandExecution 命令执行
              |-- fileChange      文件变更
              |-- mcpToolCall     MCP 工具调用
              +-- ...
```

- **Thread（会话线程）**：一个用户对话的完整生命周期。可 create / resume（恢复）/ fork（分支）/ archive（归档）。
- **Turn（推理轮次）**：一次"用户输入 → 模型反复思考+执行工具 → 最终回复"的闭环。一个 Thread 包含多个 Turn。
- **Item（对话单元）**：可流式的持久化单元，每种类型有自己的 `started → delta → completed` 生命周期。

### 为什么这层重要

把网关和引擎解耦意味着：你换一个前端（比如手机 App），只要实现同一套 JSON-RPC，core 完全不用改。这是 agent 工程化最被低估的设计决策之一。

---

## 4. 多会话模型（多会话并发）

**Codex 最鲜明的设计：一个进程承载所有并发会话。**

### 对比 Claude Code

| | Codex | Claude Code |
|---|-------|-------------|
| 进程模型 | **一个进程多会话** | 一个会话一个进程 |
| 并发 | 同一 Tokio runtime 内协程并发 | 进程隔离,靠 OS 调度 |
| 上下文切换 | 异步协作,内存开销极低 | 每个会话独立内存空间 |
| 共享能力 | 插件/MCP/模型连接池共享 | 进程间无共享 |

### Codex 的多会话生命周期

```
用户打开一个新对话
  → ThreadManager（会话管理器）注册新 ThreadId
  → 分配 Session（会话）+ submission_loop（提交循环）
  → session 启动后持续等待用户输入
  → 每个 Turn 独立有 TurnContext（轮次快照）+ StepContext（步骤快照）
  → 用户切到另一个对话继续干活
  → 两个 session 在同一进程内并发运行,互不阻塞
  → 会话关闭 → ThreadManager 摘除 → 持久化到 JSONL
```

### 操作类型

| 操作 | 含义 |
|------|------|
| create / resume | 新建 / 恢复已有会话 |
| fork | 从历史某一点分支出新 Thread |
| archive | 归档冷会话,移入 `archived_sessions/` |

---

## 5. 核心引擎：一次 Turn（推理轮次）的端到端链路

**一个 Turn 不是一次模型调用,而是"模型 → 工具 → 结果 → 再模型"的循环,直到模型不再请求工具。**

```
turn/start (用户输入)
  |
  v
1. 计算所需 MCP server ← 解析 Skills/Plugins 的依赖
  |
  v
2. 冻结 StepContext（步骤快照） ← 固定环境、MCP 工具清单、审批策略
  |
  v
3. 组装 WorldState（世界状态） ← 系统提示 + Skills 目录 + Plugins 说明 +
  |                               权限说明 + 环境说明 + Hooks 上下文
  v
4. 注入点名 Skills 的 SKILL.md 全文 + 点名 Plugins 的 MCP/App 清单
  |
  v
5. 执行 Hooks: UserPromptSubmit / PreToolUse
  |
  v
6. build_model_request → ModelClient.stream → HTTP SSE 流
  |                                              |
  |   +-- agentMessage 回复文字 → 直接推前端      |
  |   |                                          |
  |   +-- tool_call 工具调用                      |
  |         |                                     |
  |         v                                     |
  |      ToolRouter.dispatch（工具路由分发）        |
  |         → ToolOrchestrator（工具编排器）        |
  |            → approval 审批管线                 |
  |            → sandbox 沙箱执行                  |
  |            → 结果结构化 → 写回上下文            |
  |                                               |
  +-- 回到第 6 步（模型继续推理） ← 循环 ← 直到模型不再调用工具
  |
  v
7. Hooks: PostToolUse / Stop
  |
  v
8. Rollout 持久化 → turn/completed
```

### 关键点

- **StepContext（步骤快照）每轮采样冻结一次**：MCP 工具列表、环境、审批策略在单次采样期间固定,避免"模型以为能用的工具"和"执行时实际能用的工具"不一致。
- **TurnContext（轮次快照）覆盖整个 Turn**：config、model info、skills 快照等在一个 Turn 内不变。

---

## 6. 上下文系统：WorldState（世界状态/上下文）组装

**模型收到的 prompt 不是一段固定模板,而是 12+ 个 WorldStateSection 增量拼装的结果。**

### 组装顺序

```
WorldState
  +-- Personality（人格设定）
  +-- ModelInstructions（模型指令）
  +-- CollaborationMode（协作模式）
  +-- MultiAgentMode（多 agent 模式）
  +-- Permissions（权限说明）
  +-- Environments（环境说明）
  +-- AvailableSkillsInstructions（可用技能目录）
  +-- AvailablePluginsInstructions（可用插件目录）
  +-- AppsInstructions（App 说明）
  +-- AgentsMd（AGENTS.md 项目规范）
  +-- ContextWindowGuidance（上下文窗口提示）
  +-- ToolsState（工具清单）
  +-- RealtimeState（实时状态）
  +-- ...Extensions 扩展注入...
```

### 两点关键设计

1. **增量 diff**：两次采样之间只渲染变化的部分（如技能目录在新旧 Turn 间不变就不重新注入）,控制上下文膨胀。
2. **Skills / Plugins 的 Progressive Disclosure（渐进加载）**：
   - 目录先给（一行 name + description）,正文按需注入（用户点名后才把 SKILL.md 全文塞进去）
   - 每个模块有独立的 token 预算（如 Skills 目录默认 8000 字符或上下文窗口 2%）

---

## 7. 工具执行与安全

**所有工具调用走同一条管线：Hook 拦截 → Guardian 审计 → 用户审批 → 沙箱执行 → 失败重试。**

### 审批管线（Approval Pipeline）

```
模型发出 tool_call
  → Hook: PreToolUse（可 Block / 改写入参 / 直接 Allow）
    → Guardian（自动审计员）：压缩 transcript → 交给 reviewer 模型 → 超时/失败 fail-closed
      → 用户审批: 可缓存（"always allow for this command"）
        → Sandbox（沙箱）执行
          → 成功 → Hook: PostToolUse
          → 失败 → 按策略升级重试（如只读→workspace-write→full-access→拒绝）
```

### 三层沙箱

| 级别 | 权限 | 默认？ |
|------|------|--------|
| `read-only` | 只能读文件 | ✅ 默认 |
| `workspace-write` | 可写工作区目录 | 需显式配置 |
| `danger-full-access` | 完全访问 | 需显式开启 |

每个工具执行在独立子进程中跑,平台相关实现（macOS Seatbelt / Windows unelevated / Linux bubblewrap）。

### 安全关键文件
- `codex-rs/core/src/tools/orchestrator.rs` — 工具编排器
- `codex-rs/core/src/guardian/` — 自动审计员
- `codex-rs/hooks/` — 生命周期钩子（11 个事件点）
- `codex-rs/sandboxing/` — 沙箱策略

---

## 8. 扩展体系

### 四种扩展方式一张表

| 扩展方式 | 本质 | 接入点 | 典型场景 |
|---------|------|--------|---------|
| **Skills（技能指令）** | 给模型的"说明书"（SKILL.md） | prompt 注入 | 告诉模型特定任务怎么做 |
| **Plugins（能力插件）** | 能力打包分发（skills + MCP + apps + hooks） | manifest + marketplace | 一次安装配齐浏览器操控/文档编辑等 |
| **MCP servers** | 外部工具进程（Model Context Protocol） | `[mcp_servers]` config | 数据库查询、API 调用等 |
| **Hooks（生命周期钩子）** | 11 个事件点的外部拦截命令 | config / plugin | 企业审计、命令改写、权限决策 |

### Skills vs Plugins 关系

```
Skills（技能指令）           Plugins（能力插件）
    |                            |
    |  可以独立存在              |  可以独立存在
    |  (放目录即生效)             |  (marketplace 安装)
    |                            |
    +---------- 交叉点 -----------+
                |
         插件可以包含 Skills（带 `plugin_name:` 前缀）
         但 Skills 不是 Plugins 的子集
```

---

## 9. 数据持久化

**JSONL 是 Source of Truth（唯一真相源）,SQLite 是索引/投影。**

### 存储全景

```
$CODEX_HOME/
+-- sessions/
|   +-- rollout-<timestamp>-<uuid>.jsonl      ← 会话回放数据
|   +-- rollout-<...>.jsonl.zst                ← 冷文件压缩
+-- state_5.sqlite                              ← 线程元数据
+-- thread_history_1.sqlite                     ← JSONL byte offset 投影
+-- logs_2.sqlite                               ← 事件日志
+-- goals_1.sqlite                              ← Goal 状态
+-- memories_1.sqlite                           ← 记忆产物
+-- session_index.jsonl                         ← 名称 → ID 映射
```

### 两条流

**写入流**：core → RolloutRecorder（后台 mpsc channel）→ JSONL 文件 append → SQLite 同步更新 byte offset + metadata

**读取流（会话恢复）**：ThreadStore::load_history → thread_history_1.sqlite 查 byte offset → JSONL 精确跳转读取 → 重建对话上下文

### 为什么这层重要

- JSONL 是纯文本,可用 `jq` / `fx` 直接查看,丢了 SQLite 也能恢复
- byte offset 投影避免了加载历史时扫描整个 JSONL（O(1) 跳转）
- ThreadStore trait 抽象了存储实现：未来换远程存储（如 S3）只需新增 impl

---

## 10. 对团队自己设计 agent 的 6 条启示

| # | 启示 | Codex 怎么做 |
|---|------|-------------|
| 1 | **协议与引擎解耦** | app-server 是统一网关,所有前端走同一套 JSON-RPC,引擎不感知前端形态 |
| 2 | **单进程多会话** | 一个 Tokio runtime 承载所有并发会话,内存/连接池共享,低成本扩展 |
| 3 | **Turn 是循环不是一次调用** | 模型→工具→结果→再模型的闭环,StepContext 每轮冻结防不一致 |
| 4 | **渐进式加载替代全量注入** | Skills/Plugins 目录先给、正文按需,每模块有独立 token 预算 |
| 5 | **统一审批管线** | 所有工具走一条链: Hook → Guardian → 用户审批 → 沙箱,不区分工具类型 |
| 6 | **JSONL 是 truth,SQLite 是索引** | 纯文本可恢复,byte offset 投影做 O(1) 定位,存储后端可替换 |

---

> 本文档基于 openai/codex (codex-rs) 源码分析 · 学习笔记持续更新在 [github.com/wtqq/codex-agent-study](https://github.com/wtqq/codex-agent-study)

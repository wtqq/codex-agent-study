# Codex Agent 学习计划

目标：从全局到模块完整理解 Codex agent 的设计，形成可分享给团队的架构知识，并为团队自研 agent 提供设计输入。

## 总进度

| 阶段 | 主题 | 状态 | 交付物 | 下一步 |
| --- | --- | --- | --- | --- |
| 0 | 全局架构速览 | 已完成 | 分层架构图 + 模块全景表 | 已完成 |
| 1 | 产品与运行模型 | 已完成 | 用户入口/配置/沙箱速查表 | 进入阶段 2 |
| 2 | 数据模型与协议层 | 已完成 | Thread/Turn/Item 时序图 | 进入阶段 3 |
| 3 | 核心 agent 循环 | 已完成 | 一次 turn 的完整调用链图 | 进入阶段 4 |
| 4 | 横向能力与安全 | 进行中（skills/plugins/hooks 已完成） | 扩展机制与安全边界表 | 读 MCP/Connectors |
| 5 | 输出分享 | 未开始 | 团队分享文档与讲稿 | 汇总知识库并组织交流 |

## 阶段 0：全局架构速览

目标：先建立全局地图，再深入模块；每看一个模块都能放回“这一层、这个职责、和谁交互”的位置。

阅读清单（先读文档，再读代码骨架）：
- `codex-rs/app-server/README.md`：Thread / Turn / Item、JSON-RPC 生命周期、API 全景
- `codex-rs/exec-server/README.md`：进程 / PTY / 文件系统执行协议
- `codex-rs/core/README.md` 与 `codex-rs/core/src/lib.rs`：核心引擎的模块清单
- `codex-rs/AGENTS.md`：代码库约定、上下文规则、测试规则
- `codex-rs/README.md`、`codex-rs/protocol/README.md`、`codex-rs/tools/README.md`

交付物：分层架构图 + 每层模块职责/场景/交互表（已写入 `KNOWLEDGE.md`）。

完成清单：
- [x] 能画出前端、协议、核心、执行、外部系统五层地图
- [x] 能说出每层关键模块的职责和主要使用场景
- [x] 能说明一次 turn 从前端到执行的完整链路
- [x] 能说出模块之间的主要交互边界（JSON-RPC / 内存调用 / 流式事件 / 持久化）
- [x] 本阶段已记入 `SESSION_LOG.md`

## 阶段 1：产品与运行模型

目标：知道 Codex 是什么、有哪些入口、配置和沙箱怎么工作。

阅读清单：
- `D:/ai-project/test-codex/codex/README.md`
- `D:/ai-project/test-codex/codex/docs/getting-started.md`
- `D:/ai-project/test-codex/codex/docs/exec.md`
- `D:/ai-project/test-codex/codex/docs/config.md`
- `D:/ai-project/test-codex/codex/docs/sandbox.md`

交付物：用户入口、常用命令、配置项、沙箱策略速查表。

完成清单：
- [x] 能列出 CLI、TUI、IDE、桌面 App、SDK 五种入口的区别
- [x] 能说明 `codex`、`codex exec`、`codex app-server` 各自场景
- [x] 能说明配置分层和沙箱策略
- [x] 速查表已写入 `KNOWLEDGE.md`
- [x] 本阶段已记入 `SESSION_LOG.md`

## 阶段 2：数据模型与协议层

目标：理解所有前端为什么共用同一套 agent 抽象。

阅读清单：
- `D:/ai-project/test-codex/codex/codex-rs/app-server/README.md`
- `D:/ai-project/test-codex/codex/codex-rs/app-server-client/src/lib.rs`
- `D:/ai-project/test-codex/codex/codex-rs/core/src/lib.rs`
- `D:/ai-project/test-codex/codex/codex-rs/app-server-protocol/`

交付物：Thread → Turn → Item → 事件流的时序图。

完成清单：
- [x] 能解释 Thread、Turn、Item 的边界
- [x] 能说明 app-server 的 JSON-RPC 生命周期
- [x] 能说明 in-process 与 remote client 的差异
- [x] 时序图已写入 `KNOWLEDGE.md`
- [x] 本阶段已记入 `SESSION_LOG.md`

## 阶段 3：核心 agent 循环

目标：追踪一次对话从输入到执行的完整链路。

建议阅读顺序：
- `codex-rs/core/src/thread_manager.rs`
- `codex-rs/core/src/codex_thread.rs`
- `codex-rs/core/src/session/mod.rs`
- `codex-rs/core/src/session/turn.rs`
- `codex-rs/core/src/tasks/mod.rs`
- `codex-rs/core/src/context/mod.rs`
- `codex-rs/core/src/tools/mod.rs`
- `codex-rs/core/src/tools/handlers/`
- `codex-rs/exec-server/README.md`

建议追踪链路：
`ThreadManager` → `Session` → `TurnContext` → `ModelClient` → `ToolRouter` → `Orchestrator` → `UnifiedExec` → `ExecServer`

交付物：一次 turn 的完整调用链图，标注数据流向和模块边界。

完成清单：
- [x] 能讲清 context 片段如何组装、如何限制大小
- [x] 能讲清工具调用的审批与执行流程
- [x] 能讲清本地进程与远程 exec-server 的区别
- [x] 调用链图已写入 `KNOWLEDGE.md`
- [x] 本阶段已记入 `SESSION_LOG.md`

## 阶段 4：横向能力与安全

目标：理解 Codex 的扩展机制与安全边界。

阅读范围：
- `codex-rs/skills/`：技能注入
- `codex-rs/plugin/`：插件
- `codex-rs/mcp-server/`、`codex-rs/ext/mcp/`：MCP
- `codex-rs/ext/connectors/`：连接器
- `codex-rs/hooks/`：钩子
- `codex-rs/memories/`：记忆
- `codex-rs/core/src/agent/`：多智能体
- `codex-rs/core/src/sandboxing/`：权限与沙箱
- `codex-rs/model-provider/`、`codex-rs/ollama/`、`codex-rs/lmstudio/`：模型接入

交付物：扩展机制与安全边界对照表。

当前进度：
- [x] Skills / Plugins / Hooks 已精读并沉淀到 `KNOWLEDGE.md`（2026-08-02）
- [ ] MCP / Connectors
- [ ] Memories
- [ ] Multi-agent
- [ ] Sandboxing
- [ ] Model provider

完成清单：
- [ ] 能区分 Skills、Plugins、MCP、Connectors、Hooks 的定位
- [ ] 能说明多 agent 的 spawn/control/status 机制
- [ ] 能说明沙箱在 mac/Linux/Windows 的实现差异
- [ ] 对照表已写入 `KNOWLEDGE.md`
- [ ] 本阶段已记入 `SESSION_LOG.md`

## 阶段 5：输出分享

目标：把知识转化为团队可讨论、可复用的材料。

交付物：
- 团队分享文档
- 30 分钟讲稿或大纲
- 与其他 agent 的对比维度表（Claude Code 初稿已写入 `CLAUDE_CODE_NOTES.md`）

完成清单：
- [ ] 架构总览图最终版已定稿
- [ ] 核心设计决策已总结为 3 到 5 条
- [ ] 对比维度：数据模型、agent 循环、工具执行、安全边界、扩展机制
- [ ] 已组织一次团队分享
- [ ] 分享反馈已记入 `SESSION_LOG.md`

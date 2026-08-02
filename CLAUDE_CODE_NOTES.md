# Claude Code 会话/线程模型笔记（claude-code-best）

## 源码来源与注意点

- 仓库：`https://github.com/claude-code-best/claude-code.git`
- 本地路径：`D:\ai-project\test-codex\claude-code`
- 版本：`package.json` 中 `2.8.4`，自述为 “Reverse-engineered Anthropic Claude Code CLI”。
- 运行时：Bun（`engines.bun >= 1.3.0`），同时提供 `cli-node.js` / `cli-bun.js` 两个入口。
- 重要：这是社区逆向实现（Claude Code Best / CCB），不是 Anthropic 官方闭源二进制；以下结论只代表这份源码，不能直接等同于官方发布版。

## 会话与 agent 循环

- 每个会话的“主循环”是一个 async generator：`src/query.ts` 的 `query()`，内部 `while (true)` 执行：`stream_request_start` → 模型采样 → `tool_use` → `tool_result` → 下一轮，直到本轮 turn 完成。
- `src/QueryEngine.ts` 负责按用户输入驱动 `query()`。
- 交互 UI 是 React/Ink（`src/main.tsx`、`src/screens/REPL.tsx`），和 agent 循环在同一个进程内。

## 会话 ↔ 进程模型

- 交互式 CLI：一个进程运行一个交互会话，UI 与 agent 循环共用该进程。
- bridge / remote-control：`src/bridge/sessionRunner.ts` 会为每个会话 `spawn` 一个子进程，命令类似 `claude --print --sdk-url ... --session-id ...`，通过 NDJSON over stdio 通信；子进程以 headless 模式运行。
- 多会话注册表：`src/utils/concurrentSessions.ts` 写 `~/.claude/sessions/<pid>.json`（pid、sessionId、cwd、kind：`interactive`/`bg`/`daemon`/`daemon-worker`），供 `claude ps` 和 peer discovery 使用。
- 子 agent / teammate / workflow：`packages/builtin-tools/src/tools/shared/spawnMultiAgent.ts` 和 `src/workflow/backends/claudeCodeBackend.ts` 会启动独立 agent（可配合 worktree 隔离）。

## 单线程还是多线程

- Bun/Node 的 JS 执行是单事件循环线程；源码中 `LocalMemoryRecallTool` 也明确注释了 “V8/Bun JavaScript runs JS on a single event-loop thread”。
- 因此“同一个进程内所有会话共用同一个主线程”只在“单进程内多个异步任务”这个层面成立。
- 但多个会话通常不共用一个进程：交互模式一个会话一个进程；bridge 模式下每个会话一个子进程。
- 它的并发能力来自“多进程 + 单进程内异步 I/O 交错”，而不是多线程 runtime。

## 与 Codex 的对比表

| 维度 | Codex（Rust / Tokio） | claude-code-best（Bun / Node） |
| --- | --- | --- |
| 会话进程模型 | 一个 app-server 进程可挂多个会话 | 通常一个进程一个会话；bridge 下每会话一个子进程 |
| 会话主循环 | `Session.submission_loop`（异步任务，等待 Op） | `query()` async generator（`while (true)` 每轮 turn） |
| 并发模型 | Tokio 多线程 runtime，多会话异步任务并行调度 | JS 单事件循环线程；并发靠异步 I/O + 多进程 |
| 多会话并行 | 同进程内多任务并行推进 | 多进程并行；单进程内事件循环串行调度 |
| turn 驱动 | `RegularTask` / `ReviewTask` 等 `SessionTask` | `query()` / `QueryEngine` 顺序驱动 |
| 子进程 | 工具命令、MCP server、远程 exec-server | Bash 等工具子进程；bridge 会话本身也是子进程 |
| 持久化 | rollout JSONL + SQLite state + thread-store | `~/.claude/sessions` PID 注册表 + transcript/NDJSON |
| 对外协议 | `codex app-server` JSON-RPC（Thread / Turn / Item） | ACP / SDK / bridge NDJSON |

## 权限管理与沙箱

- 权限模式：`default` / `acceptEdits` / `plan` / `bypassPermissions`（`src/utils/permissions/PermissionMode.ts`、`getNextPermissionMode.ts`）。
- 权限规则：`allow` / `deny` / `ask` 规则按工具和 shell 命令匹配（`PermissionRule.ts`、`shellRuleMatching.ts`、`permissionsLoader.ts`）。
- 审批/控制：bridge / ACP 通过 `can_use_tool` control_request 把工具调用交给外部宿主审批（`bridge/sessionRunner.ts`、`bridgeMessaging.ts`）。
- 静态命令安全：`BashTool/bashSecurity.ts`、`readOnlyValidation.ts` 做命令分析、危险模式检测、`/dev/tcp` 等网络设备检测、sandbox escape 检测。
- OS 级沙箱：`src/utils/sandbox/sandbox-adapter.ts` 包装 `@anthropic-ai/sandbox-runtime` 的 `SandboxManager`，配置 fs `allowRead`/`denyRead`/`allowWrite`/`denyWrite`、network `allowedHosts`/`deniedHosts`、unix socket、proxy；底层依赖 bwrap 等系统机制，bridge 也有 `--sandbox` / `--no-sandbox` 与 `CLAUDE_CODE_FORCE_SANDBOX`。

与 Codex 的相同点：权限审批 + 命令静态策略 + OS 级沙箱三层都有；沙箱都限制文件系统与网络，都有 violation 检测与 denyWrite。

与 Codex 的差异：Codex 沙箱是自研 Rust `codex-sandboxing` + `linux-sandbox` / `windows-sandbox-rs` / Seatbelt profile；CCB 包装 Anthropic 的 `sandbox-runtime` 并叠加 JS 静态校验。自动审批方面，Codex 有 Guardian 子 agent，CCB 有 auto/YOLO classifier。

## 一句话总结

用户问题“Claude Code 是不是所有会话共用同一个主线程”：在这份源码里不成立。它通常是“一个会话一个进程”，进程内才是单事件循环线程；而 Codex 是“一个进程多个会话”，进程内由 Tokio 多线程 runtime 调度多个异步任务。两者都是异步模型，但 Codex 偏向单进程多会话，CCB 偏向多进程多会话。

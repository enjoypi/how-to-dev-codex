# 01 系统架构与设计动机

核对基线：commit `430a16799d`。路径相对 `codex-rs/`，另行注明者除外。

## 1. 分层总览

```
┌ 入口层 ─────────────────────────────────────────────────────────────┐
│ cli（子命令总入口） arg0（多身份分发） tui（交互界面） exec（无人值守） │
│ app-server-daemon（常驻服务管理）                                    │
├ 协议层 ─────────────────────────────────────────────────────────────┤
│ app-server-client（进程内/远程门面） app-server（MessageProcessor）   │
│ app-server-transport（stdio/unix/ws） app-server-protocol（v2 JSON-RPC）│
│ protocol（Op/Event 核心类型）                                         │
├ 核心层 ─────────────────────────────────────────────────────────────┤
│ core（ThreadManager/Session/turn 循环） core-api                       │
│ thread-store（存储抽象） rollout（JSONL） state（SQLite）              │
├ 能力层 ─────────────────────────────────────────────────────────────┤
│ tools apply-patch shell-command exec-server codex-mcp rmcp-client      │
│ hooks skills plugin core-plugins sandboxing linux-sandbox execpolicy   │
├ 基础设施层 ─────────────────────────────────────────────────────────┤
│ model-provider codex-api codex-client http-client config login otel   │
│ features secrets keyring-store utils/*                                 │
└─────────────────────────────────────────────────────────────────────┘
```

分层是概念划分，不是强制约束：下层 crate 普遍依赖 `codex-protocol`，`core`、`thread-store` 还依赖 `codex-app-server-protocol`。CI 只强制一条：`tui` 不得直接依赖 `core`，由 `.github/scripts/verify_tui_core_boundary.py` 检查。

## 2. 为什么用 Rust

| 原因 | 证据 |
|---|---|
| Node.js 22+ 的运行时要求阻碍推广，需要零依赖的独立可执行文件 | `31d0d7a305`（#629）提交信息 |
| 沙箱需直接调用 seccomp、Landlock 等 Linux 系统 API | 同上 |
| 无 GC，内存占用与延迟可预测 | 同上 |

#1084（`63deb7c369`）起，npm 的 `@native` 发布默认使用 Rust 版本；`codex-cli/` 只剩启动器。

## 3. 入口：单二进制与 arg0 分发

**机制**：`cli/src/main.rs` 的 `main` 用 `arg0_dispatch_or_else` 包住全部逻辑。它先调用 `arg0/src/lib.rs` 的 `arg0_dispatch`，依次判断：

1. 可执行文件名为 `codex-linux-sandbox`：进入 Linux 沙箱。
2. 文件名为 `apply_patch`/`applypatch`：进入补丁工具。
3. argv1 为内部标记（Windows MXC 沙箱、exec 辅助进程、fs 辅助进程、Windows 沙箱 wrapper、core 内 apply_patch）：进入对应辅助入口。
4. 都不是：加载 `.env`，把别名目录注入 `PATH`。

之后 `arg0_dispatch_or_else` 新建 `codex-main` 线程，在其中启动 Tokio 运行时，交给 `cli_main`。

子命令：无子命令或 `resume`/`fork` 进入 `codex_tui::run_main`；`exec` 进入 `codex_exec::run_main`；`app-server` 进入 `codex_app_server::run_main_with_transport_options`。

**为什么**：

- #1086（`89ef4efdcf`）原文：“we still want to deploy the Rust version of Codex as a single, standalone binary rather than a CLI and a supporting sandboxing binary”。沙箱进程用 `current_exe` 重新执行自身，不需额外分发文件。
- 分发判断发生在 Tokio 启动之前，此时进程只有一个线程，`execvp` 与 seccomp 设置是安全的。
- `apply_patch` 以别名注入 `PATH`，模型生成的 `apply_patch` 命令可直接执行，无需安装独立工具。
- #1697（`9102255854`）把这部分逻辑抽成独立的 `arg0` crate。

## 4. 前端与核心之间：app-server 协议

**机制**：`tui` 与 `exec` 的会话与 turn 执行都经 `app-server-client` 走 JSON-RPC；`exec` 仍直接依赖 `codex-core`，用于加载配置、初始化 state DB、检查 execpolicy：

```
tui / exec ──► app-server-client ─┬─ InProcessAppServerClient（内存通道，默认）
                                  └─ RemoteAppServerClient（ws / unix socket）
                                              │
                                              ▼
                              app-server::MessageProcessor
                                              │  turn_processor：请求 → Op
                                              ▼
                                   core::ThreadManager / CodexThread
                                              │  bespoke_event_handling：EventMsg → v2 通知
                                              ▼
                                         回到客户端
```

`exec/src/lib.rs` 调用 `InProcessAppServerClient::start`；`tui/src/lib.rs` 可选 `RemoteAppServerClient::connect`。

**为什么**：

| 原因 | 证据 |
|---|---|
| app-server 最初为 VS Code 扩展而建：“used to power experiences such as the VS Code extension” | `d9dbf48828`（#4471） |
| 进程内模式保留 app-server 语义、省去进程边界，避免出现“第二套执行契约” | `app-server/src/in_process.rs` 模块文档 |
| exec 先迁移，TUI 分阶段迁移（曾并行维护 `tui_app_server`），后接入远程认证 | `da3689f0ef`（#14005）、`9dba7337f2`（#14512）、`db89b73a9c`（#14717）、`1ff39b6fa8`（#14853） |
| TUI 长期直接依赖 core 且无手段阻止恶化，于是引入边界检查，剩余调用收敛到 `legacy_core` 后逐步清空 | `66e13efd9c`（#17399） |

效果：CLI、IDE、桌面端、SDK 共用一套协议与一套测试，新前端无需理解 core 内部。桌面端与 SDK 作为动机属推断：根目录 `README.md` 描述了 IDE 与桌面端（`codex app`），SDK 的依据只有 `sdk/` 目录。

## 5. 核心运行时：会话与 turn 生命周期

**结构**：

- `core/src/thread_manager.rs` 的 `ThreadManager` 提供 `start_thread`、`resume_*`、`fork_*`，产出 `CodexThread`（`core/src/codex_thread.rs`，对外 `submit` 与 `next_event`）。
- 每个线程一对通道：有界提交通道与无界事件通道，在 `core/src/session/mod.rs` 中创建；`submission_loop` 负责消费提交，定义在 `core/src/session/handlers.rs`。

**一次 turn**：

```
Op::TurnInput
   │ session/handlers.rs 分发
   ▼
tasks/regular.rs ──► session/turn.rs::run_turn
                        │ 必要时自动压缩上下文
                        ▼
                  run_sampling_request（循环）
                        │ try_run_sampling_request
                        ▼
                  ModelClientSession.stream ──► ResponseEvent 流
                        │ OutputItemDone
                        ▼
                  stream_events_utils::handle_output_item_done
                        │ ToolRouter::build_tool_call 识别工具调用
                        ▼
                  ToolCallRuntime::handle_tool_call（并发，放入 in_flight）
                        │
                        ▼
                  drain_in_flight 收集结果，回填为下一轮输入
                        │ needs_follow_up ?
             ┌──────────┴──────────┐
            是：继续采样         否：执行 Stop hooks，TurnComplete
```

**模型调用链**：`core` 的 `ModelClient` → `model-provider`（鉴权、Bedrock、工作区路由）→ `codex-api`（`ResponsesClient` / `ResponsesWebsocketClient`）→ `codex-client`（HTTP、重试、SSE）。`responses-api-proxy` 是独立的反向代理（`codex responses-api-proxy`），用于注入 API Key，不在默认链路中。

**为什么用队列与事件**：`docs/protocol_v1.md`（随 #629 引入）说明，Codex 通过提交队列与事件队列通信，可以运行在后台线程或独立进程中，“intended to be operated by arbitrary UI implementations”。异步事件流天然适配流式输出、并发工具调用、中断与审批等待。与直接函数调用的取舍对比无直接证据，属推断。

## 6. 协议

| 层 | 内容 | 稳定性 |
|---|---|---|
| `protocol` | `Op`（提交，`non_exhaustive`）、`Event`/`EventMsg`（事件） | 内部协议，不承诺线上格式 |
| `app-server-protocol` | JSON-RPC 2.0，方法以宏声明于 `src/protocol/common.rs`，如 `thread/start`、`turn/start`；服务端请求如 `item/commandExecution/requestApproval`；通知如 `turn/completed` | 对外稳定，标记 `#[experimental(...)]` 的方法除外 |

`docs/protocol_v1.md` 定义的层级：Session → Task（同一时间至多一个）→ 多个 Turn，上一 Turn 的输出是下一 Turn 的输入。

**为什么预计算 schema**：`acd540f158`（#36212）原文：“Normal app-server protocol builds do not need the `ts-rs` and `schemars` implementations”。常规构建使用空操作的 derive（`app-server-protocol-noop-macros`），运行时 `generate_ts` 直接解压 `schema/precomputed/*.json.zst`，只有重新生成 fixture 和测试时才启用真生成器，以此缩短编译时间。`e1e2257e29`（#54113）进一步改为在 Bazel 中编译期生成并嵌入。

## 7. 工具体系

```
模型输出 function_call
   │
   ▼
           router.rs ──► registry.rs::dispatch_any_with_state
                           │ PreToolUse hooks
                           ▼
                         orchestrator.rs：审批 → 选沙箱 → 执行 → 被拒则升级重试
                           │
         ┌─────────────────┼────────────────────┬──────────────┐
         ▼                 ▼                    ▼              ▼
   unified_exec      apply-patch          mcp_tool_call     skills/plugin
   → exec-server     （进程内或 arg0     → codex-mcp
   （本地或远程）      自调用）            → rmcp-client（stdio/HTTP/OAuth）
                           │
                           ▼
                         PostToolUse hooks ──► 结果回填
```

`router.rs`、`registry.rs`、`orchestrator.rs` 均位于 `core/src/tools/`。

| crate | 职责 |
|---|---|
| `tools` | ToolSpec 与 Responses 工具基础类型 |
| `shell-command` | 命令解析与安全判定 |
| `exec-server` | JSON-RPC 子进程与文件系统服务，可远程部署 |
| `hooks` | SessionStart、UserPromptSubmit、Pre/PostToolUse、Stop 等事件 |
| `skills` | 加载 `SKILL.md`，处理提及 |
| `plugin` / `core-plugins` | 插件包模型，目录与启用策略 |

**为什么拆出 `tools`**：`44d28f500f`（#15923）指出 schema 解析已被 core 之外的代码调用，留在 core 会让 crate 边界难以理解；`be5afc65d3`（#15928）随后抽出 MCP schema 适配器。

## 8. 持久化

| 层 | 形式 | 作用 |
|---|---|---|
| `thread-store` | `ThreadStore` trait，Local、InMemory 实现 | 统一会话存取，core 不关心后端 |
| `rollout` | `$CODEX_HOME/sessions/YYYY/MM/DD/rollout-*.jsonl` | 逐行追加的完整会话记录，用于 resume/fork/回放 |
| `state` | SQLite | 元数据索引与日志库 |

**为什么**：`dae56994da`（#17659）引入 `ThreadStore`，统一中介对 JSONL 与 SQLite 的访问；`thread-store/src/lib.rs` 模块文档称其为 “Storage-neutral”。`6e72f0dbfd`（#17826）曾加入远程 gRPC 实现，`79ad209ce6`（#21596）将其移出本仓库。

## 9. 为什么拆成约 160 个小 crate

| 原因 | 证据 |
|---|---|
| 已删除的根 AGENTS.md 原文：codex-core 已经臃肿，“resist adding code to codex-core”，新概念优先进入已有或新建 crate | `git show 18131270fe^:AGENTS.md` |
| core 越来越单体，改无关代码的迭代成本上升 | `d8f9bb65e2`（#11361） |
| `codex-common` 用 features 把不相关工具捆在一起，因而被拆掉 | `8b7f8af343`（#11422） |
| 增量编译更快 | 无直接证据，属推断；#16630（`3c7f013f97`）将 core 编译时间降低 63%，但手段是去掉 `async_trait` |

## 10. 为什么禁止 crate features

| 原因 | 证据 |
|---|---|
| features 造成只在 Cargo 下存在的 feature 组合，团队正在消除，白名单只减不增 | `dc263f5926`（#16455） |
| 一个 feature 导致 codex-core 被重复编译 | `b68a84ee8e`（#11393） |
| `--all-features` 占用大量磁盘 | `264fc444b6`（#12429） |
| Bazel 构建不支持 crate features | `verify_cargo_workspace_manifests.py` 报错文本：“our Bazel build setup does not honor them today” |

由 `.github/scripts/verify_cargo_workspace_manifests.py` 强制。#17826 曾为绕开此规则把 proto 生成放进 example 目标。

## 11. 为什么 Cargo 与 Bazel 并存

- `codex-rs/docs/bazel.md`：Bazel 提供 hermetic 构建、工具链与跨平台产物；Cargo 仍是 crate 的唯一事实来源，`MODULE.bazel` 通过 `crate.from_cargo` 读取 `Cargo.toml` 与 `Cargo.lock`。
- `2a06d64bc9`（#8875）：用 BuildBuddy 远程构建，macOS 产物在 Linux 交叉编译、在本机运行测试。
- `.github/workflows/README.md`：`bazel.yml` 是 PR 主路径，保证反馈快；远程缓存见 #10079、#9616。
- 开发者日常用 Cargo。Bazel 是 PR 阶段 CI 的主路径；正式发布仍由 Cargo 构建，Bazel 发布产物并行生成、仅供参考（`rust-release.yml` 中的 `continue-on-error`）。

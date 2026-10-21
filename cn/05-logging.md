# 05 日志与记录格式

核对基线：commit `430a16799d`。路径相对 `codex-rs/`。示例行中尖括号内的值是占位符，字段名来自代码定义。

## 1. 总览

| 输出 | 格式 | 去向 | 用途 |
|---|---|---|---|
| tracing 文本 | `tracing_subscriber` 默认文本格式 | TUI 仅在显式配置 `log_dir` 时写文件；exec 和 app-server 写 stderr | 开发者排障 |
| tracing JSON | 每行一个 JSON 对象 | app-server 设置 `LOG_FORMAT=json` 时输出到 stderr | 机器采集 |
| SQLite 日志 | `LogEntry` 记录 | `$CODEX_HOME/logs_2.sqlite` | TUI 与 app-server 写入，exec 不写；`just log` 查看 |
| rollout | JSONL | `$CODEX_HOME/sessions/...` | 会话回放、resume、fork |
| exec 事件流 | JSONL | `codex exec --json` 输出到 stdout | 自动化集成 |
| OTEL | OTLP log 和 trace | 外部采集端 | 生产遥测 |

## 2. tracing 文本日志

| 入口 | 初始化位置 | 格式 | 默认过滤 |
|---|---|---|---|
| TUI | `tui/src/startup_orchestration.rs` 中的 `tui_file_layer` | 带时间戳和 target，不含 ANSI 颜色，记录 span 的 NEW/CLOSE；文件权限 0600，用 `non_blocking` 写入，不滚动 | `codex_core=info,codex_tui=info,codex_rmcp_client=info,codex_realtime_webrtc=warn` |
| exec | `exec/src/lib.rs` 中的 `exec_stderr_env_filter` | 文本，ANSI 跟随 `--color` 参数 | `error,opentelemetry_sdk=off,opentelemetry_otlp=off` |
| app-server | `app-server/src/lib.rs` 中的 `LogFormat` | 文本，或 `LOG_FORMAT=json` 时输出 JSON | `RUST_LOG` 未设置时只输出 error |
| login | `cli/src/login.rs` 中的 `init_login_file_logging` | 文本，写入 `<log_dir>/codex-login.log`，权限 0600 | `codex_cli=info,codex_core=info,codex_login=info,...` |

上表各入口都可以用 `RUST_LOG` 覆盖默认过滤；SQLite 日志的过滤固定为 `default_filter()`，不受 `RUST_LOG` 影响。

示例：

```
2026-10-21T08:15:30.123456Z  INFO codex_tui::app: <message>
```

由 daemon 托管的 app-server 与 updater，stderr 经过 `app-server-daemon/src/backend/stderr_log.rs` 的 `stderr_writer` 写入日志文件：队列容量 64 条，队列满时丢弃新日志，超过 16 KiB 的单条整条丢弃；文件超过 1 MiB 时裁剪，最近 256 KiB 保存到 `.log.previous`。直接运行 `codex app-server` 时仍写普通 stderr。

**为什么**：

- **TUI 不能写 stderr**：TUI 运行在 alternate screen 上，任何写到终端的输出都会弄花界面。`tui/src/lib.rs` 开头的注释写明 “Forbid accidental stdout/stderr writes”，并开启了 `deny(clippy::print_stderr)`。
- **TUI 默认不再写 `codex-tui.log`**：`tui/src/lib.rs` 中 `remove_legacy_tui_log_file` 的注释是 “Shared append-only TUI logs could grow without bound”，所以改为写入有容量上限的 SQLite，旧日志文件在启动时删除。
- **exec 的 stderr 关闭 OTEL 自诊断**：注释是 “OTEL export is best-effort; keep exporter self-diagnostics out of headless command output unless the caller opts in with RUST_LOG”。基线只输出 error、stdout 留给结果和 JSONL 事件流，属推断。
- **托管 app-server 的 stderr 有上限**：防止日志写爆宿主进程或磁盘（`6bae551093`，#55856）。

## 3. SQLite 日志库

- 位置：优先配置项 `sqlite_home`，其次环境变量 `CODEX_SQLITE_HOME`，都没有时用 `CODEX_HOME`。文件名 `logs_2.sqlite`，常量在 `state/src/sqlite.rs`。
- 记录：`LogEntry`，包含 ts、level、target、message、thread_id 等字段。
- 容量：每个 thread 或进程分区最多 10 MiB 或 1000 行，见 `state/src/runtime.rs`。
- 默认过滤：基线为 TRACE，另有按 target 的 DEBUG、INFO、WARN、OFF 覆盖，例如 `h2`、`hyper_util` 为 WARN，`log`、sqlx、`codex_otel.log_only`、`codex_otel.trace_safe` 关闭。完整列表见 `state/src/log_db.rs` 中的 `default_filter`。
- 查看：

```bash
just log --level debug --module codex_core --thread-id <thread_id>
```

**为什么**：按 thread 分区，排查某个会话时一次就能查全；有容量上限，不会无限增长；默认记录 TRACE 级别，出问题时现场信息已经在库里，不用加大日志级别再复现一遍。

## 4. rollout JSONL

- 路径：`$CODEX_HOME/sessions/YYYY/MM/DD/rollout-YYYY-MM-DDThh-mm-ss-<thread_id>.jsonl`；rollout_id 与 thread_id 不同时，文件名为 `rollout-<ts>-<thread_id>_<rollout_id>.jsonl`，见 `rollout/src/rollout_file_name.rs` 中的 `render`。目录和文件名用本地时间，文件可能压缩为 `.jsonl.zst`，归档会话放在 `archived_sessions/`。路径由 `rollout/src/recorder.rs` 中的 `precompute_new_rollout_path` 生成。
- 每行一个 `RolloutLine`（`history/src/lib.rs`），字段为 `timestamp`（UTC，精确到毫秒）、可选的 `ordinal`，再加展开后的 `RolloutItem`。
- `RolloutItem` 的序列化形式由 `history/src/rollout_payload.rs` 定义：`type` 是标签，内容放在 `payload` 里。常见的 `type` 有 `session_meta`、`response_item`、`turn_context`、`compacted`、`event_msg`、`token_usage_record`。
- 第一行固定是 `session_meta`，即展开后的 `SessionMeta` 再加上 `git` 信息。

示例：

```json
{"timestamp":"2026-10-21T08:15:30.123Z","type":"session_meta","payload":{"session_id":"<uuid>","id":"<uuid>","timestamp":"2026-10-21T08:15:30.123Z","cwd":"/repo","originator":"codex_exec","cli_version":"<ver>","source":"exec","model_provider":"openai","base_instructions":{"text":"…"},"history_mode":"legacy","git":{"commit_hash":"<sha>","branch":"main"}}}
```

**为什么用 JSONL**：

- 每次只追加一行，进程崩溃时最多丢最后一行，前面的记录不受影响。`rollout/src/recorder.rs` 的模块注释写明目的是 “so sessions can be replayed or inspected later”。
- paginated 模式 resume 时，`reverse_jsonl_scanner.rs` 从文件末尾倒着读，不必加载整个文件；legacy 模式仍完整读取。
- paginated 模式 fork 时通过 `forked_from_id` 和 `history_base` 引用原会话的前缀；legacy 模式 fork 会复制完整历史。
- 每行都是独立的 JSON，可以直接用 `jq` 或 `grep` 分析。

## 5. `codex exec --json` 事件流

事件定义在 `exec/src/exec_events.rs` 的 `ThreadEvent`，每个事件以一行 JSON 写到 stdout。

| type | 含义 |
|---|---|
| `thread.started` | 会话开始，带 `thread_id` |
| `turn.started` / `turn.completed` / `turn.failed` | 一轮对话的开始、完成、失败 |
| `item.started` / `item.updated` / `item.completed` | 消息、命令、文件改动等条目的生命周期 |
| `error` | 错误 |

条目的 `id` 形如 `item_N`，`type` 取值包括 `agent_message`、`command_execution`、`file_change` 等。

示例：

```json
{"type":"thread.started","thread_id":"<uuid>"}
{"type":"item.completed","item":{"id":"item_0","type":"command_execution","command":"ls","aggregated_output":"…","exit_code":0,"status":"completed"}}
```

**为什么**：这套格式比内部的 `EventMsg` 更扁平，稳定性承诺属推断；脚本和 TypeScript SDK 可以直接消费。TS SDK 就是解析这个事件流实现的，见仓库根目录的 `sdk/typescript/src/exec.ts`。

## 6. OTEL 事件

- 宏：`otel/src/events/shared.rs` 中的 `log_event!`、`trace_event!`、`log_and_trace_event!`。两个通道的 target 分别是 `codex_otel.log_only` 和 `codex_otel.trace_safe`。
- 每个事件都带的字段：`event.timestamp`、`conversation.id`、`app.version`、`auth_mode`、`originator`、`terminal.type`、`model`、`slug`。`user.account_id` 和 `user.email` 只出现在 log 通道。

| 事件 | 主要字段 |
|---|---|
| `codex.api_request` | `duration_ms`、`http.response.status_code`、`attempt`、`endpoint` |
| `codex.tool_result` | `tool_name`、`call_id`、`duration_ms`、`success`、`output_truncated`；log 通道另含参数和输出预览 |
| `codex.user_prompt`、`codex.tool_decision`、`codex.sse_event`、`codex.sandbox_outcome` | 用户输入、审批决策、流式事件、沙箱结果 |

`codex.tool_call` 不属于上述体系：它由 `core/src/tools/parallel.rs` 中的 `log_tool_call!` 用普通的 `info!` 发出，target 是模块路径，公共字段中只带 `conversation.id`，字段见 [03 §3](03-coding-practices.md)。

示例：

```
event.name="codex.tool_result" tool_result_seq=<n> tool_name=shell tool_namespace=<ns> call_id=<id> duration_ms=42 success=true output_truncated=false
```

**为什么分两个通道**：trace 通道可能被发往范围更广的追踪后端，只带不含隐私的字段；log 通道才带用户身份和内容预览。target 名 `trace_safe` 体现了这一意图，具体部署方式属推断。

## 7. 脱敏

- `secrets/src/sanitizer.rs` 中的 `redact_secrets` 用正则替换 `sk-`、`AKIA`、`Bearer`、`api_key=`、`token=` 等形式的密钥。调用方包括 `app-server-protocol/src/protocol/item_builders.rs`（构造命令条目）、memories 模块和 `login/src/oauth_session_token.rs`。
- 用户 prompt 默认不进入遥测。`[otel] log_user_prompt` 默认是 false：log 通道写 `"[REDACTED]"`，trace 通道只记录长度和计数。`log_agent_responses`、`log_guardian_assessments` 默认也是 false。

**为什么**：日志和遥测最终会离开用户机器，所以默认只上报可以安全公开的信息。要上报内容，必须显式打开开关。

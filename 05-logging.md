> Original (Chinese): [cn/05-logging.md](cn/05-logging.md)

# 05 Logging and Record Formats

Verified against: commit `430a16799d`. Paths are relative to `codex-rs/`. Values in angle brackets in the sample lines are placeholders; field names come from the code definitions.

## 1. Overview

| Output | Format | Destination | Purpose |
|---|---|---|---|
| tracing text | `tracing_subscriber` default text format | TUI writes to a file only when `log_dir` is explicitly configured; exec and app-server write to stderr | Developer troubleshooting |
| tracing JSON | One JSON object per line | Written to stderr when app-server sets `LOG_FORMAT=json` | Machine ingestion |
| SQLite logs | `LogEntry` records | `$CODEX_HOME/logs_2.sqlite` | Written by TUI and app-server, not by exec; view with `just log` |
| rollout | JSONL | `$CODEX_HOME/sessions/...` | Session replay, resume, fork |
| exec event stream | JSONL | `codex exec --json` writes to stdout | Automation integration |
| OTEL | OTLP logs and traces | External collector | Production telemetry |

## 2. tracing text logs

| Entry point | Initialized in | Format | Default filter |
|---|---|---|---|
| TUI | `tui_file_layer` in `tui/src/startup_orchestration.rs` | Includes timestamp and target, no ANSI colors, records span NEW/CLOSE; file mode 0600, written via `non_blocking`, no rotation | `codex_core=info,codex_tui=info,codex_rmcp_client=info,codex_realtime_webrtc=warn` |
| exec | `exec_stderr_env_filter` in `exec/src/lib.rs` | Text, ANSI follows the `--color` flag | `error,opentelemetry_sdk=off,opentelemetry_otlp=off` |
| app-server | `LogFormat` in `app-server/src/lib.rs` | Text, or JSON when `LOG_FORMAT=json` | Only errors when `RUST_LOG` is unset |
| login | `init_login_file_logging` in `cli/src/login.rs` | Text, written to `<log_dir>/codex-login.log`, mode 0600 | `codex_cli=info,codex_core=info,codex_login=info,...` |

Every entry point above can override its default filter with `RUST_LOG`; the SQLite log filter is fixed to `default_filter()` and is not affected by `RUST_LOG`.

Example:

```
2026-10-21T08:15:30.123456Z  INFO codex_tui::app: <message>
```

For app-server and updater hosted by the daemon, stderr is written to a log file through `stderr_writer` in `app-server-daemon/src/backend/stderr_log.rs`: the queue holds 64 entries, new entries are dropped when the queue is full, and any single entry over 16 KiB is dropped entirely; when the file exceeds 1 MiB it is truncated, and the most recent 256 KiB is kept in `.log.previous`. Running `codex app-server` directly still writes to plain stderr.

**Why**:

- **TUI must not write to stderr**: the TUI runs on the alternate screen, and any output written to the terminal corrupts the display. The comment at the top of `tui/src/lib.rs` says "Forbid accidental stdout/stderr writes", and `deny(clippy::print_stderr)` is enabled.
- **TUI no longer writes `codex-tui.log` by default**: the comment on `remove_legacy_tui_log_file` in `tui/src/lib.rs` is "Shared append-only TUI logs could grow without bound", so logs now go to a size-capped SQLite database, and the legacy log file is deleted at startup.
- **exec stderr suppresses OTEL self-diagnostics**: the comment is "OTEL export is best-effort; keep exporter self-diagnostics out of headless command output unless the caller opts in with RUST_LOG". That the baseline emits only errors so stdout is left for results and the JSONL event stream is inferred.
- **Hosted app-server stderr is capped**: prevents logs from overwhelming the host process or filling the disk (`6bae551093`, #55856).

## 3. SQLite log database

- Location: the `sqlite_home` config option takes precedence, then the `CODEX_SQLITE_HOME` environment variable, falling back to `CODEX_HOME` if neither is set. The file name is `logs_2.sqlite`; the constant is in `state/src/sqlite.rs`.
- Records: `LogEntry`, with fields such as ts, level, target, message, and thread_id.
- Capacity: at most 10 MiB or 1000 rows per thread or process partition, see `state/src/runtime.rs`.
- Default filter: baseline TRACE, with per-target DEBUG, INFO, WARN, and OFF overrides, e.g. `h2` and `hyper_util` at WARN, and `log`, sqlx, `codex_otel.log_only`, and `codex_otel.trace_safe` off. For the full list, see `default_filter` in `state/src/log_db.rs`.
- Viewing:

```bash
just log --level debug --module codex_core --thread-id <thread_id>
```

**Why**: partitioning by thread means one query retrieves everything for a session under investigation; the size cap prevents unbounded growth; recording at TRACE by default means the context is already in the database when a problem occurs, with no need to raise the log level and reproduce it.

## 4. rollout JSONL

- Path: `$CODEX_HOME/sessions/YYYY/MM/DD/rollout-YYYY-MM-DDThh-mm-ss-<thread_id>.jsonl`; when rollout_id differs from thread_id, the file name is `rollout-<ts>-<thread_id>_<rollout_id>.jsonl`, see `render` in `rollout/src/rollout_file_name.rs`. Directory and file names use local time, files may be compressed as `.jsonl.zst`, and archived sessions go in `archived_sessions/`. The path is generated by `precompute_new_rollout_path` in `rollout/src/recorder.rs`.
- Each line is a `RolloutLine` (`history/src/lib.rs`) with the fields `timestamp` (UTC, millisecond precision), an optional `ordinal`, plus the flattened `RolloutItem`.
- The serialized form of `RolloutItem` is defined in `history/src/rollout_payload.rs`: `type` is the tag and the content goes in `payload`. Common `type` values are `session_meta`, `response_item`, `turn_context`, `compacted`, `event_msg`, and `token_usage_record`.
- The first line is always `session_meta`: the flattened `SessionMeta` plus `git` info.

Example:

```json
{"timestamp":"2026-10-21T08:15:30.123Z","type":"session_meta","payload":{"session_id":"<uuid>","id":"<uuid>","timestamp":"2026-10-21T08:15:30.123Z","cwd":"/repo","originator":"codex_exec","cli_version":"<ver>","source":"exec","model_provider":"openai","base_instructions":{"text":"…"},"history_mode":"legacy","git":{"commit_hash":"<sha>","branch":"main"}}}
```

**Why JSONL**:

- Each write appends a single line, so a process crash loses at most the last line and earlier records are unaffected. The module comment in `rollout/src/recorder.rs` states the purpose is "so sessions can be replayed or inspected later".
- On resume in paginated mode, `reverse_jsonl_scanner.rs` reads backward from the end of the file without loading the whole file; legacy mode still reads it in full.
- On fork in paginated mode, the original session's prefix is referenced via `forked_from_id` and `history_base`; fork in legacy mode copies the full history.
- Each line is standalone JSON and can be analyzed directly with `jq` or `grep`.

## 5. `codex exec --json` event stream

Events are defined by `ThreadEvent` in `exec/src/exec_events.rs`; each event is written to stdout as one line of JSON.

| type | Meaning |
|---|---|
| `thread.started` | Session started, includes `thread_id` |
| `turn.started` / `turn.completed` / `turn.failed` | Start, completion, or failure of a turn |
| `item.started` / `item.updated` / `item.completed` | Lifecycle of items such as messages, commands, and file changes |
| `error` | Error |

An item's `id` has the form `item_N`, and `type` values include `agent_message`, `command_execution`, `file_change`, and others.

Example:

```json
{"type":"thread.started","thread_id":"<uuid>"}
{"type":"item.completed","item":{"id":"item_0","type":"command_execution","command":"ls","aggregated_output":"…","exit_code":0,"status":"completed"}}
```

**Why**: this format is flatter than the internal `EventMsg`; the stability guarantee is inferred. Scripts and the TypeScript SDK can consume it directly. The TS SDK is implemented by parsing this event stream, see `sdk/typescript/src/exec.ts` at the repository root.

## 6. OTEL events

- Macros: `log_event!`, `trace_event!`, and `log_and_trace_event!` in `otel/src/events/shared.rs`. The targets of the two channels are `codex_otel.log_only` and `codex_otel.trace_safe`, respectively.
- Fields on every event: `event.timestamp`, `conversation.id`, `app.version`, `auth_mode`, `originator`, `terminal.type`, `model`, `slug`. `user.account_id` and `user.email` appear only on the log channel.

| Event | Key fields |
|---|---|
| `codex.api_request` | `duration_ms`, `http.response.status_code`, `attempt`, `endpoint` |
| `codex.tool_result` | `tool_name`, `call_id`, `duration_ms`, `success`, `output_truncated`; the log channel also includes argument and output previews |
| `codex.user_prompt`, `codex.tool_decision`, `codex.sse_event`, `codex.sandbox_outcome` | User input, approval decisions, streaming events, sandbox outcomes |

`codex.tool_call` is outside this scheme: it is emitted with a plain `info!` by `log_tool_call!` in `core/src/tools/parallel.rs`, its target is the module path, and of the common fields it carries only `conversation.id`; for its fields, see [03 §3](03-coding-practices.md).

Example:

```
event.name="codex.tool_result" tool_result_seq=<n> tool_name=shell tool_namespace=<ns> call_id=<id> duration_ms=42 success=true output_truncated=false
```

**Why two channels**: the trace channel may be sent to tracing backends with wider access, so it carries only non-private fields; only the log channel carries user identity and content previews. The target name `trace_safe` reflects this intent; the actual deployment is inferred.

## 7. Redaction

- `redact_secrets` in `secrets/src/sanitizer.rs` uses regexes to replace secrets of forms such as `sk-`, `AKIA`, `Bearer`, `api_key=`, and `token=`. Callers include `app-server-protocol/src/protocol/item_builders.rs` (building command items), the memories module, and `login/src/oauth_session_token.rs`.
- User prompts are excluded from telemetry by default. `[otel] log_user_prompt` defaults to false: the log channel writes `"[REDACTED]"`, and the trace channel records only length and counts. `log_agent_responses` and `log_guardian_assessments` also default to false.

**Why**: logs and telemetry eventually leave the user's machine, so by default only information that is safe to disclose is reported. Reporting content requires explicitly enabling a switch.

> Original (Chinese): [cn/03-coding-practices.md](cn/03-coding-practices.md)

# 03 Coding Best Practices (with Repository Examples)

Verified against: commit `430a16799d`. Paths are relative to `codex-rs/`. All code is quoted verbatim from the repository; `...` marks omissions.

Rule sources: `[workspace.lints]` in `Cargo.toml`, `clippy.toml`, `rustfmt.toml`, `deny.toml`, `tools/argument-comment-lint/` (at the repository root), and `.github/scripts/verify_*.py`. Most rules are set to deny in CI, so any violation fails the build.

## 1. Error Handling

**Rules**: Library crates define error enums with `thiserror` and `#[source]`; the application layer uses `anyhow` with `.with_context()`. `unwrap`/`expect` are banned in production code: convert an `Option` into an error with `ok_or_else(...)?`, and use `let ... else` for early returns.

**Why**:
- `unwrap_used` and `expect_used` are set to deny in the workspace lints; `clippy.toml` allows them only in test code. If a long-running Agent process panics, the session state is lost.
- Typed errors at the library layer let callers `match` on them; context chains at the application layer pinpoint which step failed.
- `clippy.toml` raises `large-error-threshold` from the clippy default of 128 to 256, with a comment saying this accommodates more informative error variants; `result_large_err` is not listed in the workspace lints, so it stays at the clippy default level, warn.

Library layer, `ImageProcessingError` in `utils/image/src/error.rs`:

```rust
#[derive(Debug, Error)]
pub enum ImageProcessingError {
    #[error("failed to read image at {path}: {source}")]
    Read {
        path: PathBuf,
        #[source]
        source: std::io::Error,
    },
    #[error("failed to decode image at {path}: {source}")]
    Decode {
        path: PathBuf,
        #[source]
        source: image::ImageError,
    },
    ...
    #[error("unsupported image `{mime}`")]
    UnsupportedImageFormat { mime: String },
```

Application layer, the flow for adding an MCP server in `cli/src/mcp_cmd.rs`:

```rust
    let mut servers = load_global_mcp_servers(codex_home)
        .await
        .with_context(|| format!("failed to load MCP servers from {}", codex_home.display()))?;
    let credential_name = new_entry.oauth_credential_name(&name);
    servers.insert(name.clone(), new_entry);

    ConfigEditsBuilder::new(codex_home)
        .replace_mcp_servers(&servers)
        .apply()
        .await
        .with_context(|| format!("failed to write MCP servers to {}", codex_home.display()))?;
```

Replacing `unwrap`, `consume_output` in `core/src/exec.rs`:

```rust
    let stdout_reader = child.stdout.take().ok_or_else(|| {
        CodexErr::Io(io::Error::other(
            "stdout pipe was unexpectedly not available",
        ))
    })?;
```

`let-else`, `core/src/mcp_tool_exposure.rs`:

```rust
        let Some(connector_id) = tool.connector_id.as_deref() else {
            return false;
        };
```

## 2. Do Not Hold Locks Across await

**Rules**: Acquire the lock inside a block scope, copy out the needed data with `Arc::clone`, let the guard drop automatically at the end of the block, and only then `await`.

**Why**: Holding a lock across `await` can block other tasks for a long time, or even deadlock. `clippy.toml` uses `await-holding-invalid-types` to forbid holding tokio's `MutexGuard`, `RwLockReadGuard`, or `RwLockWriteGuard` across `await`.

`core/src/session/step_activation.rs`:

```rust
        // Capture the running task named by this update and its settings, then release the lock.
        // A task that starts during preparation is never a new target.
        let target = {
            let active = self.active_turn.lock().await;
            active.as_ref().and_then(|active| {
                active.task.as_ref().and_then(|task| {
                    (task.turn_context.sub_id == turn_id && !task.cancellation_token.is_cancelled())
                        .then(|| {
                            (
                                Arc::clone(&task.turn_context),
                                Arc::clone(&task.done),
                                task.turn_context.next_step_settings.load_full(),
                            )
                        })
                })
            })
        };
        let Some((turn_context, task_done, current)) = target else {
            return TurnSettingsUpdateOutcome::TargetUnavailable;
        };
```

For the explicit `drop(guard)` style, see `core/src/unified_exec/process.rs`. The `parallel_execution` RwLock in `core/src/tools/parallel.rs` is a parallelism gate intentionally held across `await`, and is an exception.

## 3. Structured Logging with tracing

**Rules**: Use `#[instrument(level = "trace", skip_all)]` for function-level spans; events use structured fields, with `%` for Display and `?` for Debug, and the message text last; do not use `println!`. A few low-level crates (such as `utils/pty`) still use `log`.

**Why**:
- Structured fields can be exported directly via OTEL and written to the SQLite log store for field-based queries (see [05](05-logging.md)).
- `skip_all` keeps large or sensitive arguments from being recorded in the span automatically.
- The TUI owns the terminal, so `deny(clippy::print_stdout, clippy::print_stderr)` prevents accidental writes to the screen.

`core/src/tools/parallel.rs`:

```rust
    #[instrument(level = "trace", skip_all)]
    pub(crate) fn handle_tool_call_with_source(
        self,
        step_context: Arc<StepContext>,
        call: ToolCall,
        source: ToolCallSource,
        cancellation_token: CancellationToken,
        call_state: Arc<ToolCallState>,
    ) -> impl std::future::Future<Output = Result<AnyToolResult, FunctionCallError>> {
```

The `log_tool_call!` macro in the same file:

```rust
                info!(
                    event.name = "codex.tool_call",
                    trace_id = %codex_otel::current_span_trace_id().unwrap_or_default(),
                    conversation.id = %self.conversation_id,
                    turn_id = %self.turn_id,
                    tool_name = %self.tool_name,
                    call_id = %self.call_id,
                    tool_source = "direct",
                    execution_started = execution_started_at.is_some(),
                    dispatch_duration_ms = $dispatch_duration_ms,
                    handler_duration_ms = $handler_duration_ms,
                    total_duration_ms = $total_duration_ms,
                    "tool call completed"
                );
```

`core/src/tasks/mod.rs`:

```rust
                warn!(%err, "session task returned an unexpected error");
```

## 4. One Item per use

**Rules**: `rustfmt.toml` sets `imports_granularity = "Item"`, so each `use` imports exactly one item.

**Why**: One import per line minimizes merge conflicts, keeps diffs obvious, and spares both AI and humans from reshuffling braces when adding or removing imports.

`core/src/tools/parallel.rs`:

```rust
use std::sync::Arc;
use std::sync::OnceLock;
use std::sync::atomic::Ordering;
use std::time::Instant;

use tokio::sync::RwLock;
use tokio::task::JoinError;
use tokio_util::either::Either;
use tokio_util::sync::CancellationToken;
use tokio_util::task::AbortOnDropHandle;
use tracing::Instrument;
use tracing::info;
use tracing::instrument;
use tracing::trace_span;
```

## 5. Inline format Arguments

**Rules**: Write `format!("{name}")`, not `format!("{}", name)`. Enforced by `uninlined_format_args`.

**Why**: The variable name sits inside the placeholder, so arguments cannot get out of order.

`core/src/compact.rs`:

```rust
    let summary_text = format!("{SUMMARY_PREFIX}\n{summary_suffix}");
```

## 6. Annotate Literal Arguments with `/*param*/` Comments

**Rules**: When passing an anonymous literal such as `true`, `false`, `None`, or a number, precede it with `/*param_name*/`, and the name must match the callee's parameter. String and char literals are exempt. Also exempt: a method with a single non-self parameter whose method name is the parameter name, e.g. `.enabled(false)`.

**Why**: `foo(true, None)` at a call site is unreadable. `tools/argument-comment-lint/README.md` holds that when an enum, newtype, or named helper can make the API self-explanatory, that approach should be preferred; argument comments are the lowest-cost compromise for compatibility. The lint `argument_comment_mismatch` checks that the comment matches the parameter name, and `uncommented_anonymous_literal_argument` checks for missing comments; both are raised to error in the repository.

`core/src/sandbox_tags.rs`:

```rust
        WindowsSandboxSelection::Disabled => {
            get_platform_sandbox(/*windows_sandbox_enabled*/ false)
                .map(SandboxType::as_metric_tag)
                .unwrap_or("none")
        }
```

`core/src/mcp_tool_call.rs` (also shows inline format arguments):

```rust
        let item_metadata =
            McpToolCallItemMetadata::from_tool_metadata(&server, /*metadata*/ None);
        let result = notify_mcp_tool_call_skip(
            sess.as_ref(),
            turn_context.as_ref(),
            &call_id,
            invocation,
            item_metadata,
            format!("MCP tool `{server}/{tool_name}` is not available to the model"),
            /*already_started*/ false,
        )
        .await;
```

## 7. Unit Tests in Separate `*_tests.rs` Files

**Rules**: At the end of the source file, pull in tests with `#[cfg(test)] #[path = "xxx_tests.rs"] mod tests;`; use `pretty_assertions::assert_eq` for assertions.

**Why**:
- Implementation files are shorter, reading code is not cluttered by tests, and AI uses fewer tokens when loading context.
- Tests remain a submodule and can access private items directly.
- `pretty_assertions` prints a colored diff on failure, making differences easy to spot.

Over a thousand source files under `codex-rs/` pull in tests via `#[path = "*_tests.rs"]`.

`core/src/exec_env.rs`:

```rust
#[cfg(test)]
#[path = "exec_env_tests.rs"]
mod tests;
```

`core/src/exec_env_tests.rs`:

```rust
use super::*;
use codex_protocol::config_types::ShellEnvironmentPolicyInherit;
use maplit::hashmap;
use pretty_assertions::assert_eq;
...
#[test]
fn inject_permission_profile_env_removes_stale_value_without_active_profile() {
    let mut env = HashMap::from([(
        CODEX_PERMISSION_PROFILE_ENV_VAR.to_string(),
        "stale-profile".to_string(),
    )]);

    inject_permission_profile_env(&mut env, /*active_permission_profile*/ None);

    assert_eq!(env.get(CODEX_PERMISSION_PROFILE_ENV_VAR), None);
}
```

## 8. Integration Tests Mock the Model with wiremock

**Rules**:
- Integration tests are organized as `tests/all.rs` plus `tests/suite/`, with most cases compiled into a single test binary; the few tests that need a separate process live in their own `tests/*.rs`.
- Start wiremock with `responses::start_mock_server()`, mount SSE responses with `mount_sse_once`, and build events with `sse(vec![ev_*])`.
- Configure and start Codex with the `test_codex()` builder, then assert on the outgoing request with `mock.single_request()`.

**Why**:
- Real model output is nondeterministic; with mocked SSE, tests are reproducible and cost nothing.
- Asserting on "what request the model received" brings prompt assembly under regression testing too.
- Fewer test binaries mean faster compilation and linking.

Shared helpers live in `core/tests/common/` (crate name `core_test_support`) and `app-server/tests/common/` (`app_test_support`).

`core/tests/suite/bedrock_multi_agent_tests.rs`, parameterized with `test_case`; `provider_id` and `model` come from the omitted `#[test_case::test_case(...)]` attributes:

```rust
...
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn bedrock_catalog_selects_v2_by_default(
    provider_id: &'static str,
    model: &'static str,
) -> anyhow::Result<()> {
    skip_if_no_network!(Ok(()));
    let server = responses::start_mock_server().await;
    let mock = responses::mount_sse_once(
        &server,
        responses::sse(vec![responses::ev_completed("done")]),
    )
    .await;
    let test = test_codex()
        .with_auth(CodexAuth::BedrockApiKey(BedrockApiKeyAuth {
            api_key: "dummy".to_string(),
            region: "us-east-1".to_string(),
        }))
        .with_model(model)
        ...
        .build_with_auto_env(&server)
        .await?;
    ...
    test.submit_text_turn("Hello").await?;
    ...
    let request = mock.single_request();
```

For SSE with multiple events (`ev_response_created`, `ev_function_call`, `ev_completed`), see `core/tests/suite/abort_lifecycle.rs`.

## 9. insta Snapshots for the TUI

**Rules**: Join the rendered output into a string and pass it to `insta::assert_snapshot!`. Snapshot files live in the `snapshots/` directory and are named `codex_tui__<module>__tests__<test_name>.snap`.

**Why**: Hand-written UI assertions are tedious and easy to get incomplete; snapshots pin down the entire UI, so any UI change shows up directly as a diff in review. The repository has about 1600 `.snap` files.

`tui/src/analytics_tests.rs`:

```rust
...
    .map(|section| {
        view.panel(section, /*width*/ 54, /*chart_height*/ 2)
            .lines
            .into_iter()
            .map(|line| line.to_string())
            .collect::<Vec<_>>()
            .join("\n")
    })
    .collect::<Vec<_>>()
    .join("\n\n");
    insta::assert_snapshot!(panels);
```

`tui/src/snapshots/codex_tui__analytics__tests__analytics_loading_errors_and_empty_reports_stay_independent.snap`:

```
---
source: tui/src/analytics_tests.rs
expression: panels
---
Loading…

No history has been reported.
...
```

## 10. app-server v2 Protocol Types

**Rules**:
- Define each method's Params and Response as a pair, deriving `Serialize, Deserialize, Debug, Clone, PartialEq, JsonSchema, TS`.
- Add `#[serde(rename_all = "camelCase")]` and `#[ts(export_to = "v2/")]`; enums also get `#[ts(rename_all = "camelCase")]`.
- Register methods in `common.rs`, with names of the form `resource/action`.

**Why**:
- The same Rust types generate both the JSON Schema and the TypeScript types, so client and server cannot drift apart.
- If protocol types change without regenerating the schema, the fixture tests fail.
- The `serialization` field declares a concurrency exclusion group: `global` requests run exclusively within the group, while `global_shared_read` requests can run concurrently with each other.

`app-server-protocol/src/protocol/v2/account.rs`:

```rust
#[derive(Serialize, Deserialize, Debug, Clone, PartialEq, JsonSchema, TS)]
#[serde(rename_all = "camelCase")]
#[ts(export_to = "v2/")]
pub struct CancelLoginAccountParams {
    pub login_id: String,
}

#[derive(Serialize, Deserialize, Debug, Clone, PartialEq, Eq, JsonSchema, TS)]
#[serde(rename_all = "camelCase")]
#[ts(rename_all = "camelCase")]
#[ts(export_to = "v2/")]
pub enum CancelLoginAccountStatus {
    Canceled,
    NotFound,
}
```

`app-server-protocol/src/protocol/common.rs`:

```rust
    CancelLoginAccount => "account/login/cancel" {
        params: v2::CancelLoginAccountParams,
        serialization: global("account-auth"),
        response: v2::CancelLoginAccountResponse,
    },
```

## 11. Adding a Feature Flag

**Rules**: First add a variant with a doc comment to `enum Feature`, then add a `FeatureSpec` to `FEATURES` specifying the key (snake_case), stage, and default value.

**Why**: All feature toggles live in one table, which makes them easy to review and retire uniformly; for why runtime toggles are used instead of Cargo features, see [01 §10](01-architecture.md).

`features/src/lib.rs`:

```rust
    /// Preview consumer five-hour and weekly allowance history.
    AnalyticsPlanHistory,
```

```rust
pub const FEATURES: &[FeatureSpec] = &[
    FeatureSpec {
        id: Feature::AnalyticsPlanHistory,
        key: "analytics_plan_history",
        stage: Stage::Experimental {
            name: "Analytics plan history",
            menu_description: "Preview five-hour and weekly allowance history for consumer accounts in /analytics.",
            announcement: "",
        },
        default_enabled: false,
    },
```

## 12. Config Fields

**Rules**: New fields get a `///` doc comment (a few older fields lack one; no tool enforces this), and add `#[schemars(...)]` when a constraint or type substitution is needed.

**Why**: Doc comments flow into `config.schema.json` and become the config descriptions users see in their editor; constraints such as `range` reject invalid values at the schema level.

`config/src/config_toml.rs`:

```rust
    /// Percentage of the usable context window that triggers compaction after a final
    /// response. Existing auto-compaction limits still apply. Omitted or zero disables
    /// turn-end compaction; valid values are 0–100.
    #[schemars(range(min = 0, max = 100))]
    pub model_post_turn_compact_threshold_percent: Option<u8>,

    /// Default approval policy for executing commands.
    #[schemars(with = "Option<crate::schema::ConfigAskForApproval>")]
    pub approval_policy: Option<AskForApproval>,
```

## 13. TUI Colors and Disallowed APIs

**Rules**: The TUI uses ANSI colors by default and does not construct RGB or Indexed colors directly (theme-derived colors are constructed centrally in `tui/src/terminal_palette.rs`), nor hardcode white, black, or yellow; sqlx must not open connections directly and must go through the sqlite wrapper in `codex-state`.

**Why**: Users' terminal themes vary widely, and ANSI colors adapt to the theme automatically. Funneling database connections through one place makes migrations and connection parameters easy to manage uniformly (inferred).

`clippy.toml`:

```toml
disallowed-methods = [
    { path = "ratatui::style::Color::Rgb", reason = "Use ANSI colors, which work better in various terminal themes." },
    { path = "ratatui::style::Color::Indexed", reason = "Use ANSI colors, which work better in various terminal themes." },
    { path = "ratatui::style::Stylize::white", reason = "Avoid hardcoding white; prefer default fg or dim/bold. Exception: Disable this rule if rendering over a hardcoded ANSI background." },
    { path = "ratatui::style::Stylize::black", reason = "Avoid hardcoding black; prefer default fg or dim/bold. Exception: Disable this rule if rendering over a hardcoded ANSI background." },
    { path = "ratatui::style::Stylize::yellow", reason = "Avoid yellow; prefer other colors in `tui/styles.md`." },
```

Key points from `tui/styles.md`:

- Use `bold` for headings, the default color for body text, and `style::secondary_text_style()` or `dim` for secondary text.
- Use `style::accent_style()` for emphasis and `style::status_style(StatusTone)` for status. Use green for success and additions, red for errors and deletions, and magenta for the Codex brand.
- Do not use blue or yellow. Custom colors must pass the `style::readable_color_on` readability check.

## 14. Other Engineering Constraints

| Constraint | Enforcement | Reason |
|---|---|---|
| No new crate `[features]`, no `optional = true` | `verify_cargo_workspace_manifests.py` | see [01 §10](01-architecture.md) |
| Manifests must inherit the workspace version, edition, license, and lints | Same as above | Centralized management of versions and lints |
| Package names are `codex-<dir>`, or `codex-utils-<dir>` under utils, with a few exceptions | Same as above | The package name tells you where the directory is |
| The TUI must not depend directly on codex-core | `verify_tui_core_boundary.py` | see [01 §4](01-architecture.md) |
| Bazel clippy flags match the Cargo lints | `verify_bazel_clippy_lints.py` | Both build systems reach the same verdict on the same code |
| License allowlist; reject unknown registries or git sources | `deny.toml` + cargo-deny | Supply-chain security |
| Unused dependencies | `cargo shear` | Shorter compile times |
| Update module docs alongside state machine changes | `tui/src/bottom_pane/AGENTS.md` | Keeps docs consistent with the implementation |
| Sandbox integrity checks only emit telemetry, never change behavior, and never report paths, environment variable values, or error text | `exec-server/src/sandbox_integrity/AGENTS.md` | Monitoring must neither affect security behavior nor leak private data |

Standard stack: tokio for async, tracing for logging, and for error handling, thiserror at the library layer and anyhow at the application layer.

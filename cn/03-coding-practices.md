# 03 编码最佳实践（附仓库实例）

核对基线：commit `430a16799d`。路径相对 `codex-rs/`。代码均原样摘自仓库，`...` 表示省略。

规则来源：`Cargo.toml` 中的 `[workspace.lints]`、`clippy.toml`、`rustfmt.toml`、`deny.toml`、`tools/argument-comment-lint/`（位于仓库根目录）、`.github/scripts/verify_*.py`。多数规则在 CI 中设为 deny，违反即失败。

## 1. 错误处理

**规则**：库 crate 用 `thiserror` 定义带 `#[source]` 的错误枚举；应用层用 `anyhow` 并加 `.with_context()`；生产代码禁用 `unwrap`/`expect`，`Option` 用 `ok_or_else(...)?` 转换为错误，提前返回用 `let ... else`。

**为什么**：
- `unwrap_used` 与 `expect_used` 在 workspace lints 中设为 deny，`clippy.toml` 只在测试代码中放开。长时间运行的 Agent 进程一旦 panic，会话状态就会丢失。
- 库层的类型化错误方便调用方 `match`；应用层的 context 链方便定位是哪一步失败。
- `clippy.toml` 把 `large-error-threshold` 从 clippy 默认的 128 放宽到 256，注释说明是为了容纳信息更丰富的错误变体；`result_large_err` 未列入 workspace lints，按 clippy 默认级别 warn。

库层，`utils/image/src/error.rs` 中的 `ImageProcessingError`：

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

应用层，`cli/src/mcp_cmd.rs` 中添加 MCP server 的流程：

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

替代 `unwrap`，`core/src/exec.rs` 中的 `consume_output`：

```rust
    let stdout_reader = child.stdout.take().ok_or_else(|| {
        CodexErr::Io(io::Error::other(
            "stdout pipe was unexpectedly not available",
        ))
    })?;
```

`let-else`，`core/src/mcp_tool_exposure.rs`：

```rust
        let Some(connector_id) = tool.connector_id.as_deref() else {
            return false;
        };
```

## 2. await 期间不持有锁

**规则**：在块作用域内取锁，用 `Arc::clone` 拷出所需数据，块结束时 guard 自动释放，之后再 `await`。

**为什么**：持锁 `await` 会让其他任务长时间阻塞，甚至死锁。`clippy.toml` 用 `await-holding-invalid-types` 禁止跨 `await` 持有 tokio 的 `MutexGuard`、`RwLockReadGuard`、`RwLockWriteGuard`。

`core/src/session/step_activation.rs`：

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

显式 `drop(guard)` 的写法见 `core/src/unified_exec/process.rs`。`core/src/tools/parallel.rs` 的 `parallel_execution` 读写锁是有意跨 `await` 持有的并行闸门，属于例外。

## 3. tracing 结构化日志

**规则**：函数级 span 用 `#[instrument(level = "trace", skip_all)]`；事件用结构化字段，`%` 表示 Display，`?` 表示 Debug，消息文本放在最后；不用 `println!`。少数底层 crate（如 `utils/pty`）仍用 `log`。

**为什么**：
- 结构化字段可以直接被 OTEL 导出，也能写入 SQLite 日志库按字段检索（见 [05](05-logging.md)）。
- `skip_all` 避免把大参数或敏感参数自动写进 span。
- TUI 占用终端，用 `deny(clippy::print_stdout, clippy::print_stderr)` 防止误写屏幕。

`core/src/tools/parallel.rs`：

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

同一文件里的 `log_tool_call!` 宏：

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

`core/src/tasks/mod.rs`：

```rust
                warn!(%err, "session task returned an unexpected error");
```

## 4. 每个 use 只导入一项

**规则**：`rustfmt.toml` 设置了 `imports_granularity = "Item"`，每个 `use` 只导入一项。

**为什么**：每行一个 import，合并冲突最少，diff 一目了然，AI 和人增删 import 时也不用重排花括号。

`core/src/tools/parallel.rs`：

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

## 5. 内联 format 参数

**规则**：写 `format!("{name}")`，不写 `format!("{}", name)`。由 `uninlined_format_args` 强制。

**为什么**：变量名就在占位符里，不会出现参数顺序错位。

`core/src/compact.rs`：

```rust
    let summary_text = format!("{SUMMARY_PREFIX}\n{summary_suffix}");
```

## 6. 字面量参数加 `/*param*/` 注释

**规则**：传 `true`、`false`、`None` 或数字这类匿名字面量时，前面写 `/*形参名*/`，名字必须与被调函数的形参一致。字符串和字符字面量不需要。方法只有一个非 self 参数、且方法名就是参数名时也不需要，例如 `.enabled(false)`。

**为什么**：调用点上的 `foo(true, None)` 读不出含义。`tools/argument-comment-lint/README.md` 认为能用枚举、newtype 或具名 helper 让 API 自己说清楚时，应优先那样做；参数注释是兼容成本最低的折中。lint `argument_comment_mismatch` 检查注释与形参名是否一致，`uncommented_anonymous_literal_argument` 检查是否漏写注释，在仓库中都提升为 error。

`core/src/sandbox_tags.rs`：

```rust
        WindowsSandboxSelection::Disabled => {
            get_platform_sandbox(/*windows_sandbox_enabled*/ false)
                .map(SandboxType::as_metric_tag)
                .unwrap_or("none")
        }
```

`core/src/mcp_tool_call.rs`（同时展示了内联 format 参数）：

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

## 7. 单元测试放在独立的 `*_tests.rs` 文件

**规则**：源文件末尾用 `#[cfg(test)] #[path = "xxx_tests.rs"] mod tests;` 引入测试；断言用 `pretty_assertions::assert_eq`。

**为什么**：
- 实现文件更短，读代码时不受测试干扰，AI 加载上下文时也更省 token。
- 测试仍是子模块，能直接访问私有项。
- `pretty_assertions` 失败时输出彩色 diff，便于定位差异。

`codex-rs/` 下上千个源文件用 `#[path = "*_tests.rs"]` 引入测试。

`core/src/exec_env.rs`：

```rust
#[cfg(test)]
#[path = "exec_env_tests.rs"]
mod tests;
```

`core/src/exec_env_tests.rs`：

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

## 8. 集成测试用 wiremock 模拟模型

**规则**：
- 集成测试按 `tests/all.rs` 加 `tests/suite/` 组织，多数用例编译进同一个测试二进制；少数需要独立进程的测试单独放在 `tests/*.rs`。
- 用 `responses::start_mock_server()` 启动 wiremock，用 `mount_sse_once` 挂载 SSE 响应，事件由 `sse(vec![ev_*])` 构造。
- 用 `test_codex()` builder 配置并启动 Codex，最后用 `mock.single_request()` 断言发出的请求。

**为什么**：
- 真实模型输出不确定，模拟 SSE 后，测试可以重复跑出同样结果，也不花钱。
- 断言模型“收到了什么请求”，就能把 prompt 拼装方式也纳入回归测试。
- 测试二进制少，编译和链接都更快。

共享 helper 在 `core/tests/common/`（crate 名 `core_test_support`）和 `app-server/tests/common/`（`app_test_support`）。

`core/tests/suite/bedrock_multi_agent_tests.rs`，用 `test_case` 参数化，`provider_id` 和 `model` 来自省略的 `#[test_case::test_case(...)]` 属性：

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

包含多个事件的 SSE（`ev_response_created`、`ev_function_call`、`ev_completed`）见 `core/tests/suite/abort_lifecycle.rs`。

## 9. TUI 用 insta 快照

**规则**：把渲染结果拼成字符串，交给 `insta::assert_snapshot!`。快照文件放在 `snapshots/` 目录，命名为 `codex_tui__<模块>__tests__<测试名>.snap`。

**为什么**：手写界面断言既繁琐又容易漏；快照把整个界面固定下来，界面一改，review 时就能直接看到 diff。全仓库约 1600 个 `.snap` 文件。

`tui/src/analytics_tests.rs`：

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

`tui/src/snapshots/codex_tui__analytics__tests__analytics_loading_errors_and_empty_reports_stay_independent.snap`：

```
---
source: tui/src/analytics_tests.rs
expression: panels
---
Loading…

No history has been reported.
...
```

## 10. app-server v2 协议类型

**规则**：
- 方法的 Params 和 Response 成对定义，derive `Serialize, Deserialize, Debug, Clone, PartialEq, JsonSchema, TS`。
- 加 `#[serde(rename_all = "camelCase")]` 和 `#[ts(export_to = "v2/")]`，枚举再加 `#[ts(rename_all = "camelCase")]`。
- 方法在 `common.rs` 中注册，名字形如 `资源/动作`。

**为什么**：
- 同一份 Rust 类型同时生成 JSON Schema 和 TypeScript 类型，客户端与服务端不会对不上。
- 改了协议类型却没重新生成 schema 时，fixture 测试会失败。
- `serialization` 字段声明并发互斥组：`global` 请求在组内独占执行，`global_shared_read` 请求之间可以并发。

`app-server-protocol/src/protocol/v2/account.rs`：

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

`app-server-protocol/src/protocol/common.rs`：

```rust
    CancelLoginAccount => "account/login/cancel" {
        params: v2::CancelLoginAccountParams,
        serialization: global("account-auth"),
        response: v2::CancelLoginAccountResponse,
    },
```

## 11. 新增 Feature flag

**规则**：先在 `enum Feature` 中加一个带 doc 注释的变体，再在 `FEATURES` 中加一个 `FeatureSpec`，写明 key（snake_case）、stage 和默认值。

**为什么**：所有功能开关集中在一张表里，便于审查和统一下线；用运行时开关而不是 Cargo features，原因见 [01 §10](01-architecture.md)。

`features/src/lib.rs`：

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

## 12. 配置字段

**规则**：新增字段写 `///` doc 注释（少数旧字段缺失，无工具强制），需要约束或换类型时加 `#[schemars(...)]`。

**为什么**：doc 注释会进入 `config.schema.json`，成为用户在编辑器里看到的配置说明；`range` 之类的约束在 schema 层面就能拦住非法值。

`config/src/config_toml.rs`：

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

## 13. TUI 颜色与禁用的 API

**规则**：TUI 默认用 ANSI 颜色，不直接构造 RGB 和 Indexed（主题派生色集中在 `tui/src/terminal_palette.rs` 构造），也不硬编码 white、black、yellow；sqlx 不能直接建立连接，必须通过 `codex-state` 的 sqlite 封装。

**为什么**：用户的终端主题千差万别，ANSI 颜色会随主题自动适配。数据库连接收敛到一处，便于统一管理迁移和连接参数，此条属推断。

`clippy.toml`：

```toml
disallowed-methods = [
    { path = "ratatui::style::Color::Rgb", reason = "Use ANSI colors, which work better in various terminal themes." },
    { path = "ratatui::style::Color::Indexed", reason = "Use ANSI colors, which work better in various terminal themes." },
    { path = "ratatui::style::Stylize::white", reason = "Avoid hardcoding white; prefer default fg or dim/bold. Exception: Disable this rule if rendering over a hardcoded ANSI background." },
    { path = "ratatui::style::Stylize::black", reason = "Avoid hardcoding black; prefer default fg or dim/bold. Exception: Disable this rule if rendering over a hardcoded ANSI background." },
    { path = "ratatui::style::Stylize::yellow", reason = "Avoid yellow; prefer other colors in `tui/styles.md`." },
```

`tui/styles.md` 的要点：

- 标题用 `bold`，正文用默认颜色，次要文字用 `style::secondary_text_style()` 或 `dim`。
- 强调用 `style::accent_style()`，状态用 `style::status_style(StatusTone)`。成功和新增用 green，错误和删除用 red，Codex 品牌用 magenta。
- 不要用 blue 和 yellow。自定义颜色必须经过 `style::readable_color_on` 检查可读性。

## 14. 其他工程约束

| 约束 | 强制方式 | 原因 |
|---|---|---|
| 不新增 crate `[features]`，不写 `optional = true` | `verify_cargo_workspace_manifests.py` | 见 [01 §10](01-architecture.md) |
| manifest 必须继承 workspace 的 version、edition、license、lints | 同上 | 版本号和 lint 统一管理 |
| 包名为 `codex-<dir>`，utils 下为 `codex-utils-<dir>`，少数例外 | 同上 | 看到包名就能找到目录 |
| TUI 不得直接依赖 codex-core | `verify_tui_core_boundary.py` | 见 [01 §4](01-architecture.md) |
| Bazel 的 clippy 参数与 Cargo lints 一致 | `verify_bazel_clippy_lints.py` | 两套构建对同一段代码的判定结果相同 |
| 许可证白名单，拒绝未知 registry 或 git 源 | `deny.toml` + cargo-deny | 供应链安全 |
| 未使用的依赖 | `cargo shear` | 缩短编译时间 |
| 修改状态机时同步更新模块文档 | `tui/src/bottom_pane/AGENTS.md` | 文档与实现保持一致 |
| 沙箱完整性检查只发遥测，不改变行为，不上报路径、环境变量的值或错误文本 | `exec-server/src/sandbox_integrity/AGENTS.md` | 监控不能影响安全行为，也不能泄露隐私 |

惯用技术栈：异步统一用 tokio，日志统一用 tracing，错误处理库层用 thiserror、应用层用 anyhow。

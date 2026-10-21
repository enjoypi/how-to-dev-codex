# 04 环境、编译、运行与调试

核对基线：commit `430a16799d`。命令在仓库根目录执行，除非另行注明。justfile 设置了 `working-directory := "codex-rs"`，未标 `[no-cd]` 的配方在 `codex-rs/` 下运行；标了 `[no-cd]` 的配方（如 `bazel-lock-update`、`argument-comment-lint`、`write-hooks-schema`）在调用 `just` 的目录运行。

## 1. 环境准备

| 工具 | 安装或说明 | 依据 |
|---|---|---|
| Rust 1.99.1 | rustup 在 `codex-rs/` 下按 `rust-toolchain.toml` 自动切换，组件包括 clippy、rustfmt、rust-src | `codex-rs/rust-toolchain.toml` |
| just | `cargo install --locked just` | `docs/install.md` |
| cargo-nextest | `cargo install --locked cargo-nextest` | `just test` 依赖它 |
| dotslash | `cargo install --locked dotslash`，必须在 `PATH` 中 | 用于拉取固定版本的 buildifier、linter 等 |
| python3 | 必需 | justfile 的 shell 是 `scripts/just-shell.py` |
| uv | `just fmt` 调用 ruff，`write-app-server-schema` 更新 Python SDK 都要用 | `scripts/format.py` |
| Node 22+、pnpm 10.34.5+ | `pnpm install --frozen-lockfile` | `package.json` |
| bazelisk | 读取 `.bazelversion`（9.2.0） | `.github/actions/setup-bazel-ci/action.yml` |
| Nix（可选） | `nix develop` 提供 rust、openssl、cmake、clang | `flake.nix` |
| nightly 工具链 | 仅 argument-comment-lint 需要，版本见 `tools/argument-comment-lint/rust-toolchain` | 同目录 README |

装好后执行一次：

```bash
just install
pnpm install --frozen-lockfile
```

**为什么固定工具链**：本地、CI、Bazel 使用同一版本的 rustc 和 clippy，lint 结果才能一致。CI 用 `dtolnay/rust-toolchain` 预装同一版本，cargo 在 `codex-rs/` 下仍按 `rust-toolchain.toml` 选择；Bazel 在 `MODULE.bazel` 中固定同一版本；`flake.nix` 使用 `rust-bin.stable.latest`，不固定版本。

## 2. 编译

### 2.1 Cargo（日常开发）

```bash
cd codex-rs
cargo build
cargo run --bin codex -- "explain this codebase to me"
just codex
just exec "fix the failing test"
```

| profile | 设置 | 用途 |
|---|---|---|
| `dev` | `debug = "line-tables-only"` | 日常开发；需要完整符号时设 `CARGO_PROFILE_DEV_DEBUG=full` |
| `dev-small` | opt 0，无调试信息，strip | 减小体积 |
| `release` | thin LTO，`codegen-units = 4`，保留行号表 | 发布 |
| `profiling` | 继承 release，`debug = "full"`，关闭 LTO | 性能分析 |
| `ci-test` | 继承 test，opt 0 | CI 测试 |

**为什么 dev 只保留行号表**：保留源码位置，并与 Bazel 的调试信息默认值对齐（`a16381c445`，#47748）。减轻链接与磁盘负担属推断。

### 2.2 Bazel（CI 与发布）

```bash
just bazel-codex --help
just bazel-test
just bazel-clippy
just build-for-release
```

单个 crate 的测试目标名为 `//codex-rs/<dir>:<dir>-unit-tests`，按 `defs.bzl` 的命名规则推断。

可选的 BuildBuddy 远程缓存，在 `~/.bazelrc` 中写入：

```
common --remote_header=x-buildbuddy-api-key=<key>
```

在仓库根目录的 `user.bazelrc` 中写入：

```
common --config=buildbuddy-generic
```

API key 只能放在 `~/.bazelrc` 或已被 gitignore 的 `user.bazelrc` 中，不能提交。

### 2.3 musl 交叉编译

以 `.github/workflows/rust-release.yml` 为准：

1. 安装 Zig 0.14.0，执行 `TARGET=x86_64-unknown-linux-musl bash .github/scripts/install-musl-build-tools.sh`。
2. 设置 `AWS_LC_SYS_NO_JITTER_ENTROPY=1`。
3. 通过 `.github/actions/setup-rusty-v8` 设置 `RUSTY_V8_ARCHIVE`。
4. 先 `cargo build --target $T --release --bin bwrap`，strip 后计算 sha256，写入 `CODEX_BWRAP_SHA256`，再构建主二进制。

**为什么先构建 bwrap**：`rust-release.yml` 注释：“Codex embeds this digest at build time and verifies the bundled bwrap resource before use”。

## 3. 本地运行

| 场景 | 命令 |
|---|---|
| TUI | `just codex`，等价于 `cargo run --bin codex` |
| TUI 搭配独立的 exec-server | `just tui-with-exec-server`（仅 Unix） |
| 无人值守 | `just exec "<prompt>"`，参数见 [08](08-ai-sdlc.md) |
| app-server | `cargo run --bin codex -- app-server --listen stdio://`，可选 `unix://`、`ws://IP:PORT` |
| 生成协议类型 | `codex app-server generate-ts`、`codex app-server generate-json-schema` |
| 隔离的 home 目录 | `CODEX_HOME="$(mktemp -d)" just codex`，目录必须已存在 |
| 临时覆盖配置 | `-c model="o3" -c shell_environment_policy.inherit=all`，值按 TOML 解析 |
| 叠加 profile | `-p work`，叠加 `$CODEX_HOME/work.config.toml`；不是选择 `[profiles.work]` 表 |
| VS Code 扩展调试源码版 | 把 `chatgpt.cliExecutable` 指向 `scripts/debug-codex.sh` |

app-server 测试客户端（在 `codex-rs/` 下执行）：

```bash
cargo build -p codex-cli --bin codex
cargo run -p codex-app-server-test-client -- --codex-bin ./target/debug/codex serve --listen ws://127.0.0.1:4222 --kill
cargo run -p codex-app-server-test-client -- model-list
cargo run -p codex-app-server-test-client -- send-message-v2 "hello"
cargo run -p codex-app-server-test-client -- watch
```

app-server 的日志写在 `/tmp/codex-app-server-test-client/app-server.log`。

## 4. 调试

### 4.1 日志

| 入口 | 默认去向 | 打开或查看 |
|---|---|---|
| TUI | SQLite `$CODEX_HOME/logs_2.sqlite` | `just log --level debug --module codex_core --thread-id <id>` |
| TUI 文本文件 | 默认关闭 | `codex -c log_dir=./.codex-log`，再 `tail -F ./.codex-log/codex-tui.log` |
| exec | stderr，默认只输出 error | `RUST_LOG=codex_core=debug codex exec ...` |
| app-server | stderr，默认只输出 error；同时写入 SQLite `$CODEX_HOME/logs_2.sqlite` | `RUST_LOG=info LOG_FORMAT=json codex app-server` |
| TUI 会话录制 | 关闭 | `CODEX_TUI_RECORD_SESSION=1`；`CODEX_TUI_SESSION_LOG_PATH` 可指定路径，默认 `<log_dir>/session-<UTC 时间>.jsonl` |

格式细节见 [05](05-logging.md)。

### 4.2 诊断命令

| 命令 | 用途 |
|---|---|
| `codex doctor [--json] [--all]` | 检查安装、配置、认证和运行时状态 |
| `codex sandbox -- <cmd>` | 在当前平台的沙箱里执行命令，复现被拒问题 |
| `codex sandbox --log-denials -- <cmd>` | macOS 下收集 Seatbelt 的拒绝记录 |
| `codex debug models` | 输出模型目录 |
| `codex debug prompt-input "<prompt>"` | 输出模型实际收到的输入列表，用于排查 AGENTS.md 和上下文拼装 |
| `codex debug app-server send-message-v2 "<msg>"` | 直接向 app-server 发一条消息 |
| `codex execpolicy check --rules x.rules git status` | 验证 execpolicy 规则 |
| `codex mcp list|get|add|remove|login|logout` | 管理 MCP server |
| TUI 中输入 `/feedback` | 上传日志给维护者 |

### 4.3 回放与追踪

- **rollout**：每个会话对应一个 `$CODEX_HOME/sessions/YYYY/MM/DD/rollout-*.jsonl`，用 `codex resume <id>` 复现现场。
- **rollout trace**：设置 `CODEX_ROLLOUT_TRACE_ROOT=/dir` 后运行，得到 `manifest.json`、`trace.jsonl` 和 `payloads/`；再执行 `codex debug trace-reduce <bundle>` 生成 `state.json`。内容敏感，只保存在本地。
- **OTEL**：在 `config.toml` 中配置：

```toml
[otel]
exporter = { otlp-http = { endpoint = "http://localhost:4318", protocol = "json" } }
log_user_prompt = false
```

### 4.4 断点调试

`.vscode/launch.json` 提供两个配置：“Cargo launch” 构建 `codex-tui` 并启动，“Attach to running codex CLI” 附加到正在运行的进程。需要 vscode-lldb 插件，推荐插件列表见 `.vscode/extensions.json`。

### 4.5 MCP 一致性测试

```bash
pnpm install --frozen-lockfile
cargo build --locked --manifest-path codex-rs/Cargo.toml -p codex-cli --bin codex
python3 scripts/mcp_conformance/run_codex_compliance.py codex-rs/target/debug/codex \
  --conformance-cli node_modules/@modelcontextprotocol/conformance/dist/index.js \
  --baseline-report scripts/mcp_conformance/regression-baseline-v1.json \
  --report /tmp/mcp-report.json
```

`rmcp-client/src/bin/` 下有 stdio 和 streamable HTTP 两种测试用 MCP server。

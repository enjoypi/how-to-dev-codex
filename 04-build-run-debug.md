> Original (Chinese): [cn/04-build-run-debug.md](cn/04-build-run-debug.md)

# 04 Environment, Build, Run, and Debug

Verified against: commit `430a16799d`. Commands run from the repository root unless noted otherwise. The justfile sets `working-directory := "codex-rs"`, so recipes without `[no-cd]` run in `codex-rs/`; recipes marked `[no-cd]` (such as `bazel-lock-update`, `argument-comment-lint`, `write-hooks-schema`) run in the directory where `just` is invoked.

## 1. Environment Setup

| Tool | Installation or notes | Source |
|---|---|---|
| Rust 1.99.1 | rustup switches automatically under `codex-rs/` per `rust-toolchain.toml`; components include clippy, rustfmt, rust-src | `codex-rs/rust-toolchain.toml` |
| just | `cargo install --locked just` | `docs/install.md` |
| cargo-nextest | `cargo install --locked cargo-nextest` | Required by `just test` |
| dotslash | `cargo install --locked dotslash`; must be on `PATH` | Fetches pinned versions of buildifier, linters, etc. |
| python3 | Required | The justfile shell is `scripts/just-shell.py` |
| uv | Needed by `just fmt` (calls ruff) and by `write-app-server-schema` (updates the Python SDK) | `scripts/format.py` |
| Node 22+, pnpm 10.34.5+ | `pnpm install --frozen-lockfile` | `package.json` |
| bazelisk | Reads `.bazelversion` (9.2.0) | `.github/actions/setup-bazel-ci/action.yml` |
| Nix (optional) | `nix develop` provides rust, openssl, cmake, clang | `flake.nix` |
| nightly toolchain | Only needed for argument-comment-lint; version in `tools/argument-comment-lint/rust-toolchain` | README in the same directory |

After installing, run once:

```bash
just install
pnpm install --frozen-lockfile
```

**Why pin the toolchain**: local, CI, and Bazel builds use the same rustc and clippy versions, so lint results match. CI preinstalls the same version with `dtolnay/rust-toolchain`, and cargo under `codex-rs/` still selects it per `rust-toolchain.toml`; Bazel pins the same version in `MODULE.bazel`; `flake.nix` uses `rust-bin.stable.latest` and does not pin a version.

## 2. Build

### 2.1 Cargo (Daily Development)

```bash
cd codex-rs
cargo build
cargo run --bin codex -- "explain this codebase to me"
just codex
just exec "fix the failing test"
```

| profile | Settings | Purpose |
|---|---|---|
| `dev` | `debug = "line-tables-only"` | Daily development; set `CARGO_PROFILE_DEV_DEBUG=full` for full symbols |
| `dev-small` | opt 0, no debug info, strip | Smaller binaries |
| `release` | thin LTO, `codegen-units = 4`, keeps line tables | Release |
| `profiling` | Inherits release, `debug = "full"`, LTO off | Profiling |
| `ci-test` | Inherits test, opt 0 | CI tests |

**Why dev keeps only line tables**: it preserves source locations and aligns with Bazel's default debug info (`a16381c445`, #47748). Reducing link time and disk usage is inferred.

### 2.2 Bazel (CI and Release)

```bash
just bazel-codex --help
just bazel-test
just bazel-clippy
just build-for-release
```

The test target for a single crate is `//codex-rs/<dir>:<dir>-unit-tests`, inferred from the naming rules in `defs.bzl`.

For the optional BuildBuddy remote cache, add to `~/.bazelrc`:

```
common --remote_header=x-buildbuddy-api-key=<key>
```

Add to `user.bazelrc` at the repository root:

```
common --config=buildbuddy-generic
```

The API key belongs only in `~/.bazelrc` or the gitignored `user.bazelrc`; never commit it.

### 2.3 musl Cross-Compilation

Per `.github/workflows/rust-release.yml`:

1. Install Zig 0.14.0 and run `TARGET=x86_64-unknown-linux-musl bash .github/scripts/install-musl-build-tools.sh`.
2. Set `AWS_LC_SYS_NO_JITTER_ENTROPY=1`.
3. Set `RUSTY_V8_ARCHIVE` via `.github/actions/setup-rusty-v8`.
4. Run `cargo build --target $T --release --bin bwrap` first, strip it, compute its sha256, write it to `CODEX_BWRAP_SHA256`, then build the main binary.

**Why build bwrap first**: the `rust-release.yml` comment says: "Codex embeds this digest at build time and verifies the bundled bwrap resource before use".

## 3. Running Locally

| Scenario | Command |
|---|---|
| TUI | `just codex`, equivalent to `cargo run --bin codex` |
| TUI with a standalone exec-server | `just tui-with-exec-server` (Unix only) |
| Unattended | `just exec "<prompt>"`; for arguments, see [08](08-ai-sdlc.md) |
| app-server | `cargo run --bin codex -- app-server --listen stdio://`; `unix://` and `ws://IP:PORT` are also available |
| Generate protocol types | `codex app-server generate-ts`, `codex app-server generate-json-schema` |
| Isolated home directory | `CODEX_HOME="$(mktemp -d)" just codex`; the directory must already exist |
| Temporary config overrides | `-c model="o3" -c shell_environment_policy.inherit=all`; values are parsed as TOML |
| Layer a profile | `-p work` layers `$CODEX_HOME/work.config.toml`; it does not select a `[profiles.work]` table |
| Debug a source build from the VS Code extension | Point `chatgpt.cliExecutable` to `scripts/debug-codex.sh` |

app-server test client (run in `codex-rs/`):

```bash
cargo build -p codex-cli --bin codex
cargo run -p codex-app-server-test-client -- --codex-bin ./target/debug/codex serve --listen ws://127.0.0.1:4222 --kill
cargo run -p codex-app-server-test-client -- model-list
cargo run -p codex-app-server-test-client -- send-message-v2 "hello"
cargo run -p codex-app-server-test-client -- watch
```

The app-server log is written to `/tmp/codex-app-server-test-client/app-server.log`.

## 4. Debugging

### 4.1 Logs

| Entry point | Default destination | How to enable or view |
|---|---|---|
| TUI | SQLite `$CODEX_HOME/logs_2.sqlite` | `just log --level debug --module codex_core --thread-id <id>` |
| TUI text file | Off by default | `codex -c log_dir=./.codex-log`, then `tail -F ./.codex-log/codex-tui.log` |
| exec | stderr, errors only by default | `RUST_LOG=codex_core=debug codex exec ...` |
| app-server | stderr, errors only by default; also written to SQLite `$CODEX_HOME/logs_2.sqlite` | `RUST_LOG=info LOG_FORMAT=json codex app-server` |
| TUI session recording | Off | `CODEX_TUI_RECORD_SESSION=1`; `CODEX_TUI_SESSION_LOG_PATH` sets the path, default `<log_dir>/session-<UTC time>.jsonl` |

For format details, see [05](05-logging.md).

### 4.2 Diagnostic Commands

| Command | Purpose |
|---|---|
| `codex doctor [--json] [--all]` | Checks installation, config, auth, and runtime state |
| `codex sandbox -- <cmd>` | Runs a command in the current platform's sandbox to reproduce denials |
| `codex sandbox --log-denials -- <cmd>` | Collects Seatbelt denial records on macOS |
| `codex debug models` | Prints the model catalog |
| `codex debug prompt-input "<prompt>"` | Prints the input list the model actually receives, for debugging AGENTS.md and context assembly |
| `codex debug app-server send-message-v2 "<msg>"` | Sends a message directly to app-server |
| `codex execpolicy check --rules x.rules git status` | Validates execpolicy rules |
| `codex mcp list|get|add|remove|login|logout` | Manages MCP servers |
| Type `/feedback` in the TUI | Uploads logs to maintainers |

### 4.3 Replay and Tracing

- **rollout**: each session maps to one `$CODEX_HOME/sessions/YYYY/MM/DD/rollout-*.jsonl`; use `codex resume <id>` to reproduce it.
- **rollout trace**: run with `CODEX_ROLLOUT_TRACE_ROOT=/dir` set to get `manifest.json`, `trace.jsonl`, and `payloads/`; then run `codex debug trace-reduce <bundle>` to generate `state.json`. The contents are sensitive; keep them local.
- **OTEL**: configure in `config.toml`:

```toml
[otel]
exporter = { otlp-http = { endpoint = "http://localhost:4318", protocol = "json" } }
log_user_prompt = false
```

### 4.4 Breakpoint Debugging

`.vscode/launch.json` provides two configurations: "Cargo launch" builds and starts `codex-tui`, and "Attach to running codex CLI" attaches to a running process. Requires the vscode-lldb extension; recommended extensions are listed in `.vscode/extensions.json`.

### 4.5 MCP Conformance Tests

```bash
pnpm install --frozen-lockfile
cargo build --locked --manifest-path codex-rs/Cargo.toml -p codex-cli --bin codex
python3 scripts/mcp_conformance/run_codex_compliance.py codex-rs/target/debug/codex \
  --conformance-cli node_modules/@modelcontextprotocol/conformance/dist/index.js \
  --baseline-report scripts/mcp_conformance/regression-baseline-v1.json \
  --report /tmp/mcp-report.json
```

`rmcp-client/src/bin/` contains two test MCP servers: stdio and streamable HTTP.

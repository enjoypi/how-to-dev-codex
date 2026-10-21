# 06 软件开发生命周期：从改代码到发布

核对基线：commit `430a16799d`。

## 1. 本地开发循环

```
改代码
  │
  ▼
just fmt ─────────────► 格式化（justfile、cargo fmt、buildifier、ruff）
  │
  ▼
just fix -p <crate> ──► clippy --fix，处理剩余警告
  │
  ▼
just test -p <crate> ─► nextest，只跑受影响的 crate
  │  快照有变化？
  ├──是──► cargo insta review，逐条确认
  │  改了配置、协议或 hooks？
  ├──是──► just write-config-schema / write-app-server-schema / write-hooks-schema
  │  依赖有变化？
  ├──是──► just bazel-lock-update，与 Cargo.lock 一并提交
  ▼
提交前自查 ──► 提交 PR
```

| 步骤 | 命令 | 为什么 |
|---|---|---|
| 格式化 | `just fmt`，只检查用 `just fmt-check`；Markdown 和 JS 另用 `pnpm run format:fix`，`pnpm run format` 只检查 | 一条命令覆盖 justfile、Rust、Bazel、Python，避免 CI 卡在格式问题上 |
| 修复 lint | `just fix -p <crate>`，只检查用 `just clippy -p <crate>` | lint 规则多数为 deny；按 crate 运行比全量快得多 |
| 测试 | `just test -p <crate>`，全量用 `just test` | nextest 并行执行、隔离进程；`--no-fail-fast` 一次报出所有失败 |
| 快照 | `cargo insta review` 或 `cargo insta accept` | 界面变化必须逐条人工确认。仓库未写明这两条命令，属 cargo-insta 标准用法 |
| schema | `just write-*-schema` | 生成物有 fixture 测试守着，漏了重新生成就会失败 |
| 锁文件 | `just bazel-lock-update`，校验用 `just bazel-lock-check` | Bazel 从 `Cargo.lock` 派生依赖，两份锁文件必须同步 |

注意：

- 不要用 `--all-features`。workspace 禁止 crate features，`#12429` 记录过这个参数占用大量磁盘。
- `just test` 会设置 `RUST_MIN_STACK=8388608`，对应 `.cargo/config.toml` 中 Windows 的 8 MiB 栈。

## 2. 提交前自查

| 检查 | 命令 |
|---|---|
| 参数注释 | `just argument-comment-lint`（Bazel，全仓库）或 `just argument-comment-lint -p codex-core` |
| manifest 规则 | `python3 .github/scripts/verify_cargo_workspace_manifests.py` |
| TUI 边界 | `python3 .github/scripts/verify_tui_core_boundary.py` |
| Bazel clippy 一致性 | `python3 .github/scripts/verify_bazel_clippy_lints.py` |
| CI 脚本单测 | `just test-github-scripts` |
| 未用依赖 | `cd codex-rs && cargo shear --deny-warnings` |
| 依赖许可证 | `cd codex-rs && cargo deny check`（仓库未写明本地命令） |
| 拼写 | `codespell -I .codespellignore`（仓库未写明本地命令） |
| 文档格式 | `pnpm run format`、`./scripts/asciicheck.py README.md` |
| 基准冒烟 | `just bench-smoke` |

**为什么全部做成脚本**：每条规则都能在本地复现，开发者和 AI 不必等 CI 才发现问题，CI 失败时也能在本地一键重跑。

## 3. 新增 crate

1. 在 `codex-rs/Cargo.toml` 的 `[workspace] members` 中加入目录名；如有其他 crate 依赖它，再在 `[workspace.dependencies]` 中加 `codex-xxx = { path = "xxx" }`。
2. 新 crate 的 `Cargo.toml` 参照 `codex-rs/rollout-trace/Cargo.toml`：

```toml
[package]
edition.workspace = true
license.workspace = true
name = "codex-xxx"
version.workspace = true

[lib]
doctest = false
name = "codex_xxx"
path = "src/lib.rs"

[lints]
workspace = true

[dependencies]
anyhow = { workspace = true }
```

3. 满足 `verify_cargo_workspace_manifests.py` 的要求：
   - 包名为 `codex-<dir>`，`utils/` 下为 `codex-utils-<dir>`；目录名已以 `codex-` 开头时直接用目录名；例外见脚本中的 `TOP_LEVEL_NAME_EXCEPTIONS` 与 `UTILITY_NAME_EXCEPTIONS`，如 `windows-sandbox-rs` 对应 `codex-windows-sandbox`；
   - 不写 `[features]` 和 `optional = true`，`[features]` 的例外仅 `MANIFEST_FEATURE_EXCEPTIONS` 中 `code-mode`、`v8-poc` 的 `sandbox`；
   - workspace 内部依赖不写 `features` 或 `default-features = false`。
4. 新建 `BUILD.bazel`：

```
load("//:defs.bzl", "codex_rust_crate")

codex_rust_crate(
    name = "xxx",
    crate_name = "codex_xxx",
)
```

   按需添加参数：有快照的加 `test_data_extra = glob(["src/**/snapshots/**"])`；测试中需要嵌套沙箱的加 `test_tags = ["no-sandbox"]`。
5. 依次执行 `just bazel-lock-update`、`just fmt`、`cargo shear`、`just bazel-test`。

**为什么用 `codex_rust_crate` 宏**：一个宏同时生成 library、binary 和测试目标，命名和 Cargo 约定一致，新 crate 只要写两行。

## 4. 测试体系

| 层级 | 组织方式 | 工具 |
|---|---|---|
| 单元测试 | 同级的 `xxx_tests.rs`，通过 `#[path]` 引入 | `pretty_assertions`、`tempfile` |
| 集成测试 | `tests/all.rs` 加 `tests/suite/*.rs` | `core_test_support`、`app_test_support`、wiremock |
| 界面测试 | `snapshots/*.snap` | insta |
| CLI 测试 | 启动真实二进制 | `assert_cmd`、`test-binary-support`（arg0 分发） |
| 协议测试 | schema fixture 比对 | `app-server-protocol/src/schema_fixtures_tests.rs` |
| Windows | PR 阶段 `bazel.yml` 在自托管 Windows runner 上分 4 片运行 gnullvm 测试，有凭据时经 Linux RBE 交叉编译；合并到 main 后另跑原生 msvc 测试；少数场景（如 Linux 主机连 Windows exec-server）在 Linux 上用 Wine 运行 | `bazel/rules/testing/wine/wine.bzl` 中的 `wine_rust_test` |
| 互操作 | MCP 一致性测试 | `scripts/mcp_conformance/` |
| 性能 | `just bench`、`just bench-smoke`、`just bench-e2e` | cargo bench、Bazel |

**为什么另用 Wine**：跨系统场景需要 Linux 与 Windows 进程同时运行，Wine 让它在一台 Linux runner 上完成。此条属推断。

## 5. 持续集成

### 5.1 PR 阶段（阻塞合并）

入口 `.github/workflows/blocking-ci.yml` 汇总以下检查：

| workflow | 内容 |
|---|---|
| `bazel.yml` | 主要的测试和 clippy |
| `rust-ci.yml` | fmt、cargo shear、argument-comment-lint，按改动范围决定运行哪些 job |
| `cargo-deny.yml` | 安全公告、许可证、禁用依赖与依赖来源 |
| `codespell.yml` | 拼写 |
| `repo-checks.yml` | manifest 与边界校验、Python 脚本单测、prettier、ASCII 检查 |
| `sdk.yml` | SDK 测试 |
| blob-size-policy | 限制大文件进入仓库 |

最后由一个 `required` job 汇总结果，它用 `always()` 运行。注释说明了原因：不加 `always()` 时，依赖的 job 一失败，这个 job 就会被跳过，而跳过的必需检查在 GitHub 上会显示为通过。

### 5.2 合并后（不阻塞）

`postmerge-ci.yml` 调用 `rust-ci-full.yml`（全平台 clippy、分片 nextest、release 构建、remote-env 测试）和 `v8-canary.yml`。分支名包含 `full-ci` 时，可以在合并前提前跑一次全量检查。

**为什么这样分**：`.github/workflows/README.md` 写明，能用 Bazel 表达的检查放在 PR 阶段；重量级的 Cargo 原生检查放到合并后，压低 PR 的等待时间。

## 6. 发布

```
提交版本号（Cargo.toml 0.0.0 → x.y.z）
   │
   ▼
推送 tag rust-vX.Y.Z
   │
   ▼
rust-release.yml
   ├─ tag-check：tag 必须与 codex-rs/Cargo.toml 的 version 一致
   ├─ 各平台构建（含 musl、bwrap 哈希）
   ├─ macOS：Azure Key Vault + rcodesign 签名、公证；Linux：cosign 签名，生成 sigstore 产物
   ├─ GitHub Release：说明取自 tag 指向的 commit 信息，alpha、beta 版本标为 prerelease
   ├─ npm：x.y.z → latest；x.y.z-alpha.N 与 x.y.z-alpha.N.M → alpha；其他版本不发布
   ├─ r2-release.yml：上传 R2
   └─ dotslash、winget、latest-alpha-cli 分支
   ▼
python-sdk-cli-release.yml：Python SDK 跟进发布（见 sdk/python/RELEASING.md）
```

| 机制 | 为什么 |
|---|---|
| main 分支版本号固定为 `0.0.0`，发布时才写入真实版本 | 日常提交不会和版本号冲突；tag-check 保证 tag 与代码中的版本一致 |
| `.github/scripts/npm_alpha_tag.py` 保证 dist-tag 只前进不后退 | 防止较早的构建把 `alpha` 或 `latest` 指回旧版本；稳定版读不到元数据时直接失败 |
| `release`、`publish-r2`、`publish-npm`、`winget`、`update-branch` 共用 `rust-release` 并发锁 | 注释说明：防止构建乱序完成，导致各渠道的版本被回滚 |
| macOS 签名和公证 | 未签名的二进制会被 Gatekeeper 拦截 |
| sigstore | 用户可以验证产物的来源 |

Python SDK、rusty-v8 各有独立的发布 workflow；Windows 构建由 rust-release 调用 `rust-release-windows.yml` 完成。

## 7. 观测与反馈

| 渠道 | 内容 |
|---|---|
| OTEL | 请求、工具、审批、沙箱事件，见 [05](05-logging.md) |
| analytics | 产品事件批量上报 |
| `/feedback` | 内存中保留最多 4 MiB 的日志，连同 doctor 报告、rollout、Windows 沙箱日志一起上传到 Sentry |
| diagnostics | 进程级指标，快照中不含内容 |
| issue | 由 codex-action 自动打标签、去重、翻译，见 [08](08-ai-sdlc.md) |

## 8. 贡献规则

- `docs/contributing.md` 写明不接受外部代码 PR，只接受 issue、复现步骤和根因分析。CLA 由 `.github/workflows/cla.yml` 检查，见 `docs/CLA.md`；安全问题按 `SECURITY.md` 的流程披露。
- 提交信息：标题以 `(#PR号)` 结尾，正文用 `## Why` 和 `## What changed` 两节。近期提交多数包含 `## What changed`，约一半包含 `## Why`，无工具强制。

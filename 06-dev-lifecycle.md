> Original (Chinese): [cn/06-dev-lifecycle.md](cn/06-dev-lifecycle.md)

# 06 Software Development Lifecycle: From Code Change to Release

Verified against: commit `430a16799d`.

## 1. Local Development Loop

```
Edit code
  │
  ▼
just fmt ─────────────► Format (justfile, cargo fmt, buildifier, ruff)
  │
  ▼
just fix -p <crate> ──► clippy --fix, then handle remaining warnings
  │
  ▼
just test -p <crate> ─► nextest, only the affected crates
  │  Snapshots changed?
  ├──yes─► cargo insta review, confirm each one
  │  Changed config, protocol, or hooks?
  ├──yes─► just write-config-schema / write-app-server-schema / write-hooks-schema
  │  Dependencies changed?
  ├──yes─► just bazel-lock-update, commit together with Cargo.lock
  ▼
Pre-submit checks ──► Open PR
```

| Step | Command | Why |
|---|---|---|
| Format | `just fmt`; check only with `just fmt-check`. For Markdown and JS, use `pnpm run format:fix`; `pnpm run format` only checks | One command covers justfile, Rust, Bazel, and Python, so CI does not fail on formatting |
| Fix lints | `just fix -p <crate>`; check only with `just clippy -p <crate>` | Most lint rules are deny; running per crate is much faster than a full run |
| Test | `just test -p <crate>`; full run with `just test` | nextest runs in parallel with isolated processes; `--no-fail-fast` reports all failures at once |
| Snapshots | `cargo insta review` or `cargo insta accept` | UI changes must be reviewed one by one by a human. The repository does not document these two commands; they are standard cargo-insta usage |
| Schema | `just write-*-schema` | Generated files are guarded by fixture tests, which fail if you forget to regenerate |
| Lock files | `just bazel-lock-update`; verify with `just bazel-lock-check` | Bazel derives dependencies from `Cargo.lock`, so the two lock files must stay in sync |

Notes:

- Do not use `--all-features`. The workspace forbids crate features, and `#12429` records this flag consuming a lot of disk space.
- `just test` sets `RUST_MIN_STACK=8388608`, matching the 8 MiB Windows stack in `.cargo/config.toml`.

## 2. Pre-Submit Checks

| Check | Command |
|---|---|
| Argument comments | `just argument-comment-lint` (Bazel, whole repository) or `just argument-comment-lint -p codex-core` |
| Manifest rules | `python3 .github/scripts/verify_cargo_workspace_manifests.py` |
| TUI boundary | `python3 .github/scripts/verify_tui_core_boundary.py` |
| Bazel clippy consistency | `python3 .github/scripts/verify_bazel_clippy_lints.py` |
| CI script unit tests | `just test-github-scripts` |
| Unused dependencies | `cd codex-rs && cargo shear --deny-warnings` |
| Dependency licenses | `cd codex-rs && cargo deny check` (the repository does not document a local command) |
| Spelling | `codespell -I .codespellignore` (the repository does not document a local command) |
| Doc formatting | `pnpm run format`, `./scripts/asciicheck.py README.md` |
| Benchmark smoke test | `just bench-smoke` |

**Why everything is a script**: every rule is reproducible locally, so developers and AI need not wait for CI to find problems, and a CI failure can be rerun locally with one command.

## 3. Adding a Crate

1. Add the directory name to `[workspace] members` in `codex-rs/Cargo.toml`; if other crates depend on it, also add `codex-xxx = { path = "xxx" }` to `[workspace.dependencies]`.
2. Model the new crate's `Cargo.toml` on `codex-rs/rollout-trace/Cargo.toml`:

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

3. Satisfy the requirements of `verify_cargo_workspace_manifests.py`:
   - The package name is `codex-<dir>`, or `codex-utils-<dir>` under `utils/`; if the directory name already starts with `codex-`, use it as is. For exceptions, see `TOP_LEVEL_NAME_EXCEPTIONS` and `UTILITY_NAME_EXCEPTIONS` in the script, e.g. `windows-sandbox-rs` maps to `codex-windows-sandbox`;
   - Do not use `[features]` or `optional = true`; the only `[features]` exceptions are `sandbox` for `code-mode` and `v8-poc` in `MANIFEST_FEATURE_EXCEPTIONS`;
   - Internal workspace dependencies must not set `features` or `default-features = false`.
4. Create `BUILD.bazel`:

```
load("//:defs.bzl", "codex_rust_crate")

codex_rust_crate(
    name = "xxx",
    crate_name = "codex_xxx",
)
```

   Add arguments as needed: for snapshots, add `test_data_extra = glob(["src/**/snapshots/**"])`; for tests that need a nested sandbox, add `test_tags = ["no-sandbox"]`.
5. Run `just bazel-lock-update`, `just fmt`, `cargo shear`, and `just bazel-test` in order.

**Why use the `codex_rust_crate` macro**: one macro generates the library, binary, and test targets with names that follow Cargo conventions, so a new crate needs only two lines.

## 4. Testing

| Level | Organization | Tools |
|---|---|---|
| Unit tests | Sibling `xxx_tests.rs`, included via `#[path]` | `pretty_assertions`, `tempfile` |
| Integration tests | `tests/all.rs` plus `tests/suite/*.rs` | `core_test_support`, `app_test_support`, wiremock |
| UI tests | `snapshots/*.snap` | insta |
| CLI tests | Launch the real binary | `assert_cmd`, `test-binary-support` (arg0 dispatch) |
| Protocol tests | Schema fixture comparison | `app-server-protocol/src/schema_fixtures_tests.rs` |
| Windows | In the PR stage, `bazel.yml` runs gnullvm tests in 4 shards on self-hosted Windows runners, cross-compiled via Linux RBE when credentials are available; after merging to main, native msvc tests also run; a few scenarios (such as a Linux host connecting to a Windows exec-server) run under Wine on Linux | `wine_rust_test` in `bazel/rules/testing/wine/wine.bzl` |
| Interoperability | MCP conformance tests | `scripts/mcp_conformance/` |
| Performance | `just bench`, `just bench-smoke`, `just bench-e2e` | cargo bench, Bazel |

**Why also use Wine**: cross-OS scenarios need Linux and Windows processes running at the same time, and Wine lets this happen on a single Linux runner. This point is inferred.

## 5. Continuous Integration

### 5.1 PR Stage (Blocks Merge)

The entry point `.github/workflows/blocking-ci.yml` aggregates the following checks:

| workflow | Contents |
|---|---|
| `bazel.yml` | Main tests and clippy |
| `rust-ci.yml` | fmt, cargo shear, argument-comment-lint; which jobs run depends on the changed paths |
| `cargo-deny.yml` | Security advisories, licenses, banned dependencies, and dependency sources |
| `codespell.yml` | Spelling |
| `repo-checks.yml` | Manifest and boundary checks, Python script unit tests, prettier, ASCII check |
| `sdk.yml` | SDK tests |
| blob-size-policy | Keeps large files out of the repository |

A final `required` job aggregates the results and runs with `always()`. A comment explains why: without `always()`, the job is skipped when a dependency job fails, and a skipped required check shows as passed on GitHub.

### 5.2 Post-Merge (Non-Blocking)

`postmerge-ci.yml` calls `rust-ci-full.yml` (clippy on all platforms, sharded nextest, release builds, remote-env tests) and `v8-canary.yml`. A branch whose name contains `full-ci` runs the full checks before merging.

**Why split it this way**: `.github/workflows/README.md` states that checks expressible in Bazel run in the PR stage, while heavyweight native Cargo checks run after merge to keep PR wait times low.

## 6. Release

```
Commit the version (Cargo.toml 0.0.0 → x.y.z)
   │
   ▼
Push tag rust-vX.Y.Z
   │
   ▼
rust-release.yml
   ├─ tag-check: the tag must match the version in codex-rs/Cargo.toml
   ├─ Per-platform builds (including musl and the bwrap hash)
   ├─ macOS: Azure Key Vault + rcodesign signing and notarization; Linux: cosign signing, sigstore artifacts
   ├─ GitHub Release: notes come from the message of the commit the tag points to; alpha and beta versions are marked prerelease
   ├─ npm: x.y.z → latest; x.y.z-alpha.N and x.y.z-alpha.N.M → alpha; other versions are not published
   ├─ r2-release.yml: upload to R2
   └─ dotslash, winget, latest-alpha-cli branch
   ▼
python-sdk-cli-release.yml: Python SDK release follows (see sdk/python/RELEASING.md)
```

| Mechanism | Why |
|---|---|
| The main branch version is fixed at `0.0.0`; the real version is written only at release | Everyday commits never conflict over the version; tag-check ensures the tag matches the version in the code |
| `.github/scripts/npm_alpha_tag.py` ensures dist-tags only move forward | Prevents an older build from pointing `alpha` or `latest` back to an old version; a stable release fails outright if metadata cannot be read |
| `release`, `publish-r2`, `publish-npm`, `winget`, and `update-branch` share the `rust-release` concurrency lock | Per the comment: prevents builds from finishing out of order and rolling back versions on some channels |
| macOS signing and notarization | Gatekeeper blocks unsigned binaries |
| sigstore | Users can verify artifact provenance |

The Python SDK and rusty-v8 each have their own release workflow; Windows builds are done by rust-release calling `rust-release-windows.yml`.

## 7. Observability and Feedback

| Channel | Contents |
|---|---|
| OTEL | Request, tool, approval, and sandbox events; see [05](05-logging.md) |
| analytics | Batched product events |
| `/feedback` | Keeps up to 4 MiB of logs in memory and uploads them to Sentry along with the doctor report, rollout, and Windows sandbox logs |
| diagnostics | Process-level metrics; snapshots contain no content |
| issue | Auto-labeled, deduplicated, and translated by codex-action; see [08](08-ai-sdlc.md) |

## 8. Contribution Rules

- `docs/contributing.md` states that external code PRs are not accepted; only issues, reproduction steps, and root-cause analyses are. CLA is checked by `.github/workflows/cla.yml`, see `docs/CLA.md`; security issues are disclosed through the process in `SECURITY.md`.
- Commit messages: the title ends with `(#PR number)`, and the body has two sections, `## Why` and `## What changed`. Most recent commits include `## What changed` and about half include `## Why`; no tool enforces this.

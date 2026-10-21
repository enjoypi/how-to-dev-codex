> Original (Chinese): [cn/07-ai-friendly.md](cn/07-ai-friendly.md)

# 07 Why This Repository Is AI-Friendly

Verified against: commit `430a16799d`. Paths are relative to `codex-rs/`.

The biggest risk of AI-written code is that it "looks right but is actually wrong". This repository's approach is to have machines catch every category of error as early as possible, so AI can complete the "edit → verify → fix" loop on its own without waiting for human review.

## 1. Instruction Injection: AGENTS.md

**Mechanism** (`core/src/agents_md.rs`):

1. Search upward from the current directory until reaching a marker in `project_root_markers` (default `.git`).
2. In each directory level, take the first file that exists, in order: `AGENTS.override.md`, `AGENTS.md`, fallback filenames.
3. Concatenate from the root down to the current directory; content closer to the current directory comes later.
4. Global instructions live in `$CODEX_HOME/AGENTS.override.md` or `$CODEX_HOME/AGENTS.md` (`codex-home/src/instructions/mod.rs`). With a single environment, they are separated from project instructions by `--- project-doc ---`; with multiple environments, they are labeled per environment instead.
5. The budget for project-level docs is controlled by `project_doc_max_bytes` (default 32 KiB), and anything beyond it is truncated. Global instructions do not count toward it. Project docs are skipped when the project is not trusted.

**Why**:

- Rules apply per directory, so subdirectories can refine rules without polluting the global scope. The repository currently has two local rule files:
  - `tui/src/bottom_pane/AGENTS.md`: when modifying the composer or the paste-burst state machine, the module docs must be updated as well.
  - `exec-server/src/sandbox_integrity/AGENTS.md`: only telemetry may be emitted; sandbox behavior must not change, and paths, environment variable values, or error text must not be reported.
- The byte budget prevents instructions from crowding out the model's context window.
- `codex debug prompt-input` shows the final concatenated input, so you can see at a glance whether instructions took effect.

**Current state**: The root `AGENTS.md` (320 lines) and `.codex/skills` were deleted in #49713. The old content can be viewed with `git show 18131270fe^:AGENTS.md`. Its rules, such as "don't stuff code into codex-core" and using `just fix -p` and `just test -p` for local verification, are currently not machine-enforced; only the rule that the TUI must not depend on core is enforced, by `verify_tui_core_boundary.py`. There is no direct evidence for why it was deleted.

## 2. Strict Constraints Enable Self-Verification

| Constraint | Mistake AI makes | How it is caught |
|---|---|---|
| `[workspace.lints.clippy]` set to deny: `unwrap_used`, `expect_used`, `uninlined_format_args`, `redundant_clone`, etc. | Taking the shortcut of calling unwrap directly | `just clippy -p`, compile-time error |
| `await-holding-invalid-types` | Awaiting while holding a lock | Same as above |
| `disallowed-methods` | Using RGB colors in the TUI; bypassing the wrapper to access the database directly | Same as above |
| argument-comment-lint | Writing calls like `foo(true, None)` whose meaning is unclear | `just argument-comment-lint` |
| insta snapshots | Breaking the UI without noticing | Test fails and prints a diff |
| schema fixture tests | Changing the protocol or config but forgetting to regenerate | `config_schema_matches_fixture`, `schema_fixtures_tests.rs` |
| `verify_tui_core_boundary.py` | Letting the TUI call core directly for convenience | repo-checks |
| `verify_cargo_workspace_manifests.py` | Adding features; nonstandard package names | repo-checks |
| `cargo shear` | Leaving unused dependencies | rust-ci |

**Why it works**: Every rule can be reproduced locally with one command, and the error message points directly to the violated rule. AI can read the error and fix it on its own, without a human explaining the rules.

## 3. Small Crates Enable Local Verification

- The workspace has about 160 crates. When changing one crate, `just test -p <crate>` compiles and tests only that crate and its dependencies.
- **Why**: The faster AI's "edit → verify" loop runs, the more iterations it gets. A full build and test run takes tens of minutes, while running per crate takes only a few minutes; this timing comparison is inferred.

## 4. Tests Separated from Implementation

- Unit tests live in separate `*_tests.rs` files; see [03 §7](03-coding-practices.md).
- **Why**: When AI reads an implementation file, its context is not filled with large blocks of tests; when tests need to be added, the target file is obvious.

## 5. Deterministic Tests

| Technique | Location | Purpose |
|---|---|---|
| wiremock mocking Responses SSE | `core/tests/common/responses.rs` | Model behavior is controllable, results are reproducible, and tests cost nothing |
| Asserting outbound requests | `mock.single_request()` | Prompt assembly is also covered by regression tests |
| In-process arg0 dispatch for tests | `configure_test_binary_dispatch` in `test-binary-support` | In `InstallAliases` mode, installs helper aliases into a temp directory and restores the original `CODEX_HOME` afterward; config isolation is handled by each test's own temp directory |
| `skip_if_no_network!` | `core/tests/common/lib.rs` | Skips when `CODEX_SANDBOX_NETWORK_DISABLED` is detected; this variable is set inside the network-disabled Codex sandbox, so AI running tests in the sandbox does not get false failures from missing network access |

**Why it matters**: For AI, flaky tests are the most damaging: it cannot tell whether a failure was caused by its own change or by the environment.

## 6. Generated Artifacts Have a Single Source of Truth

- Protocol types are defined in Rust, and JSON Schema, TypeScript types, and Python SDK models are generated from them.
- `config.schema.json` is generated from the config structure `ConfigToml`.
- Hook inputs and outputs are generated into `hooks/schema/generated/`.

**Why**: AI only needs to change one place and then run `just write-*-schema` to generate the rest, so languages never end up half-updated.

## 7. Commit History Is Itself Training Material

- Nearly all commit titles end with `(#N)`.
- For the body format, see [06 §8](06-dev-lifecycle.md).

**Why it helps**: When AI wants to understand the origin of a design, `git log --grep` or `git log -S` finds the original commit containing `## Why`. Most of the "Why" explanations in this series were traced this way.

## 8. Applying This to Your Own Project

| Practice | Lowest-cost adoption |
|---|---|
| Per-directory AGENTS.md | Put general rules at the root; add separate rules for sensitive directories |
| Rules as lints | Wire clippy deny, `disallowed-methods`, and custom `verify_*.py` scripts into CI |
| Generated artifacts with a single source | Generate multi-language types from a schema, then guard them with fixture tests |
| Deterministic tests | Mock all external dependencies; lock outputs with snapshots |
| Local verification | Split into small modules; provide per-module entry points such as `just test -p` |
| Commits state the reason | Include `## Why` and `## What changed` sections in the PR template |

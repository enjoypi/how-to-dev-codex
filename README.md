> Original (Chinese): [cn/README.md](cn/README.md)

# How to develop the Codex 

*Codex Architecture and Engineering Practices*

Verified against: `openai/codex` commit `430a16799d` (2026-10-21). Paths are relative to the repository root. Conclusions marked "inferred" have no direct evidence.

## One-Line Overview

Codex is OpenAI's local coding agent. Its core is a Rust workspace of about 160 crates under `codex-rs/`, compiled into a single `codex` binary; `codex-cli/` handles npm distribution; `sdk/` provides the TypeScript and Python SDKs. All frontends (TUI, `codex exec`, IDE, SDK) share one app-server JSON-RPC protocol that drives the same core runtime.

## Document Index

| File | Question answered |
|---|---|
| [01-architecture.md](01-architecture.md) | How is the system layered? How does a conversation run? Why this design? |
| [02-security-config.md](02-security-config.md) | How do sandboxing, approvals, configuration, and authentication work? Why? |
| [03-coding-practices.md](03-coding-practices.md) | What are the coding conventions? What are real examples in the repository? |
| [04-build-run-debug.md](04-build-run-debug.md) | How to set up the environment, build, run, and debug? |
| [05-logging.md](05-logging.md) | What formats do logs and records use? Why? |
| [06-dev-lifecycle.md](06-dev-lifecycle.md) | What is the full workflow from code change to release? |
| [07-ai-friendly.md](07-ai-friendly.md) | Why is this repository easy for AI to develop? |
| [08-ai-sdlc.md](08-ai-sdlc.md) | How to have AI develop, test, and release automatically? |

## Repository Top Level

| Directory | Purpose |
|---|---|
| `codex-rs/` | Rust workspace, the core product |
| `codex-cli/` | npm wrapper: `bin/codex.js` launcher and packaging scripts |
| `sdk/` | `typescript/`, `python/`, `python-runtime/` |
| `bazel/`, `MODULE.bazel`, `defs.bzl`, `rbe.bzl` | Bazel rules, platforms, toolchains, remote execution |
| `patches/` | Patches for Bazel third-party dependencies (rules_rust, llvm, v8, etc.) |
| `third_party/` | powershell, v8, wine, wezterm, etc. |
| `tools/` | `argument-comment-lint` (Dylint rules), buildifier |
| `scripts/` | Formatting, npm staging, ASCII checks, MCP conformance tests, etc. |
| `.github/` | workflows, validation scripts, release scripts, Codex prompts |

## Lifecycle Overview

```
 Spec/Issue ──► Code ──► Verify ──► PR CI ──► Merge ──► Post-merge CI ──► Tag release ──► Observe
     │           │          │           │                    │              │              │
 codex-action  AGENTS.md  just fmt    blocking-ci        rust-ci-full   rust-release   otel/analytics
 label/dedupe  guardrails just fix    bazel tests        all-OS clippy  sign/notarize  /feedback
 translate     exec/SDK   just test   lint/deny/boundary nextest shards npm/R2/winget  rollout/logs
                          schema gen  required rollup    v8 canary      concurrency lock       │
     ▲                                                                                         │
     └────────────────────────────────── issues flow back ◄────────────────────────────────────┘
```

## Maintenance Conventions

- Reference only paths, symbols, and commit hashes, never line numbers.
- After code changes, run `grep -rhoE '[A-Za-z0-9_./-]+\.(rs|toml|py|yml|md|bzl|json|sh|ts|snap|sbpl|codexpolicy)' . | sort -u` in this directory to list referenced paths, and confirm each one still exists.
- When updating the baseline commit, update the first line of each file accordingly.

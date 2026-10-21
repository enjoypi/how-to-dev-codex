# How to develop the Codex 

*Codex 架构与工程实践*

核对基线：`openai/codex` commit `430a16799d`（2026-10-21）。路径均相对仓库根目录。标注“推断”的结论无直接证据。

## 一句话概览

Codex 是 OpenAI 的本地编码 Agent。主体是 `codex-rs/` 下约 160 个 crate 组成的 Rust workspace，编译为单个 `codex` 二进制；`codex-cli/` 负责 npm 分发；`sdk/` 提供 TypeScript 与 Python SDK。所有前端（TUI、`codex exec`、IDE、SDK）共用一套 app-server JSON-RPC 协议驱动同一个核心运行时。

## 文档索引

| 文件 | 回答的问题 |
|---|---|
| [01-architecture.md](01-architecture.md) | 系统怎么分层？一次对话如何运转？为什么这样设计？ |
| [02-security-config.md](02-security-config.md) | 沙箱、审批、配置、认证如何工作？为什么？ |
| [03-coding-practices.md](03-coding-practices.md) | 编码规范有哪些？仓库里的真实实例是什么？ |
| [04-build-run-debug.md](04-build-run-debug.md) | 如何搭环境、编译、运行、调试？ |
| [05-logging.md](05-logging.md) | 日志与记录是什么格式？为什么？ |
| [06-dev-lifecycle.md](06-dev-lifecycle.md) | 从改代码到发布的完整流程是什么？ |
| [07-ai-friendly.md](07-ai-friendly.md) | 为什么这个仓库方便 AI 开发？ |
| [08-ai-sdlc.md](08-ai-sdlc.md) | 如何让 AI 自动开发、测试、发布？ |

## 仓库顶层

| 目录 | 作用 |
|---|---|
| `codex-rs/` | Rust workspace，产品主体 |
| `codex-cli/` | npm 外壳：`bin/codex.js` 启动器与打包脚本 |
| `sdk/` | `typescript/`、`python/`、`python-runtime/` |
| `bazel/`、`MODULE.bazel`、`defs.bzl`、`rbe.bzl` | Bazel 规则、平台、工具链、远程执行 |
| `patches/` | Bazel 第三方依赖补丁（rules_rust、llvm、v8 等） |
| `third_party/` | powershell、v8、wine、wezterm 等 |
| `tools/` | `argument-comment-lint`（Dylint 规则）、buildifier |
| `scripts/` | 格式化、npm 暂存、ASCII 检查、MCP 一致性测试等 |
| `.github/` | workflows、校验脚本、发布脚本、Codex 提示词 |

## 生命周期总图

```
 需求/Issue ──► 编码 ──► 本地验证 ──► PR CI ──► 合并 ──► 合并后 CI ──► 打 tag 发布 ──► 观测与反馈
     │           │          │           │                    │              │              │
 codex-action  AGENTS.md  just fmt    blocking-ci        rust-ci-full   rust-release   otel/analytics
 打标签/去重   约束+规范   just fix     bazel 测试         全平台 clippy  签名/公证      /feedback
 翻译          exec/SDK    just test    lint/deny/边界    分片 nextest   npm/R2/winget  rollout/日志
                           schema 生成  required 汇总     v8 canary      并发锁                  │
     ▲                                                                                         │
     └──────────────────────────────────── 问题回流 ◄──────────────────────────────────────────┘
```

## 维护约定

- 只引用路径、符号、commit hash，不引用行号。
- 代码变更后，在本目录执行 `grep -rhoE '[A-Za-z0-9_./-]+\.(rs|toml|py|yml|md|bzl|json|sh|ts|snap|sbpl|codexpolicy)' . | sort -u` 列出引用路径，逐个确认仍存在。
- 更新基线 commit 时同步修改各文件首行。

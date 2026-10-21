# 07 为什么这个仓库方便 AI 开发

核对基线：commit `430a16799d`。路径相对 `codex-rs/`。

AI 写代码最大的风险是“看起来对，实际上错”。这个仓库的做法是让每一类错误都尽早被机器发现，AI 不用等人审查，自己就能完成“修改 → 验证 → 修正”的循环。

## 1. 指令注入：AGENTS.md

**机制**（`core/src/agents_md.rs`）：

1. 从当前目录向上查找，直到遇到 `project_root_markers` 中的标记（默认是 `.git`）。
2. 每一级目录按顺序取第一个存在的文件：`AGENTS.override.md`、`AGENTS.md`、fallback 文件名。
3. 从根目录到当前目录依次拼接，越靠近当前目录的内容越靠后。
4. 全局指令放在 `$CODEX_HOME/AGENTS.override.md` 或 `$CODEX_HOME/AGENTS.md`（`codex-home/src/instructions/mod.rs`），单环境时与项目指令之间用 `--- project-doc ---` 分隔，多环境时改为按环境加标签。
5. 项目级文档的预算由 `project_doc_max_bytes` 控制，默认 32 KiB，超出的部分截断；全局指令不计入；项目未被信任时跳过项目文档。

**为什么**：

- 规则按目录生效，子目录可以细化规则，而不必污染全局。仓库里正有两个局部规则文件：
  - `tui/src/bottom_pane/AGENTS.md`：修改 composer 或 paste-burst 状态机时，必须同步更新模块文档。
  - `exec-server/src/sandbox_integrity/AGENTS.md`：只允许发遥测，不得改变沙箱行为，不得上报路径、环境变量的值或错误文本。
- 设字节预算，是为了防止指令挤占模型的上下文窗口。
- 用 `codex debug prompt-input` 可以查看拼接后的最终输入，指令有没有生效一看便知。

**现状**：根目录的 `AGENTS.md`（320 行）和 `.codex/skills` 已在 #49713 删除。旧版内容可以用 `git show 18131270fe^:AGENTS.md` 查看，其中“不要往 codex-core 塞代码”、用 `just fix -p` 和 `just test -p` 做局部验证等规则，目前没有机器强制；仅 TUI 不得依赖 core 一条由 `verify_tui_core_boundary.py` 强制。删除原因无直接证据。

## 2. 强约束提供自验证

| 约束 | AI 会犯的错 | 发现手段 |
|---|---|---|
| `[workspace.lints.clippy]` 设为 deny：`unwrap_used`、`expect_used`、`uninlined_format_args`、`redundant_clone` 等 | 走捷径直接 unwrap | `just clippy -p`，编译期报错 |
| `await-holding-invalid-types` | 持锁 await | 同上 |
| `disallowed-methods` | TUI 中使用 RGB 颜色，绕过封装直连数据库 | 同上 |
| argument-comment-lint | 写出 `foo(true, None)` 这种看不出含义的调用 | `just argument-comment-lint` |
| insta 快照 | 改坏界面却没察觉 | 测试失败并输出 diff |
| schema fixture 测试 | 改了协议或配置却忘记重新生成 | `config_schema_matches_fixture`、`schema_fixtures_tests.rs` |
| `verify_tui_core_boundary.py` | 为了图省事让 TUI 直接调用 core | repo-checks |
| `verify_cargo_workspace_manifests.py` | 新增 feature、包名不规范 | repo-checks |
| `cargo shear` | 留下用不到的依赖 | rust-ci |

**为什么有效**：每条规则都能在本地用一条命令复现，错误信息直接指出违反了哪条规则。AI 读到报错就能自行修正，不需要人来解释规矩。

## 3. 小 crate 支持局部验证

- workspace 约 160 个 crate。改一个 crate 时，`just test -p <crate>` 只编译和测试它本身及其依赖。
- **为什么**：AI 的“修改 → 验证”循环跑得越快，迭代次数越多。全量编译和测试要几十分钟，按 crate 运行只要几分钟，这个耗时对比属推断。

## 4. 测试与实现分离

- 单元测试放在独立的 `*_tests.rs` 文件，见 [03 §7](03-coding-practices.md)。
- **为什么**：AI 读实现文件时不会被大段测试占满上下文；需要补测试时，目标文件是哪个也一目了然。

## 5. 确定性测试

| 手段 | 位置 | 作用 |
|---|---|---|
| wiremock 模拟 Responses SSE | `core/tests/common/responses.rs` | 模型行为可控，结果可复现，测试不花钱 |
| 断言出站请求 | `mock.single_request()` | prompt 拼装方式也纳入回归测试 |
| 测试进程内的 arg0 分发 | `test-binary-support` 中的 `configure_test_binary_dispatch` | 在 `InstallAliases` 模式下用临时目录安装 helper 别名，完成后恢复原 `CODEX_HOME`；配置隔离由各测试自建的临时目录完成 |
| `skip_if_no_network!` | `core/tests/common/lib.rs` | 检测到 `CODEX_SANDBOX_NETWORK_DISABLED` 时跳过；该变量在网络被禁用的 Codex 沙箱内设置，所以 AI 在沙箱中跑测试时不会因无网而误报失败 |

**为什么重要**：对 AI 来说，时好时坏的不稳定测试最致命，它分不清失败是自己改出来的，还是环境造成的。

## 6. 生成物有唯一来源

- 协议类型由 Rust 定义，再生成 JSON Schema、TypeScript 类型和 Python SDK 的模型。
- 配置结构由 `ConfigToml` 生成 `config.schema.json`。
- hooks 的输入输出生成到 `hooks/schema/generated/`。

**为什么**：AI 只需要改一个地方，再运行 `just write-*-schema` 生成其余部分，不会出现几种语言各改一半的情况。

## 7. 提交历史本身是训练材料

- 几乎所有提交标题都以 `(#N)` 结尾。
- 正文格式见 [06 §8](06-dev-lifecycle.md)。

**为什么有用**：AI 想弄清某个设计的来由时，用 `git log --grep` 或 `git log -S` 就能找到写有 `## Why` 的原始提交，本系列文档中的“为什么”大多就是这样考证出来的。

## 8. 在自己的项目中借鉴

| 做法 | 最低成本的落地方式 |
|---|---|
| 分目录的 AGENTS.md | 根目录写通用规则，敏感目录单独补充规则 |
| 规则写成 lint | 把 clippy deny、`disallowed-methods`、自定义的 `verify_*.py` 接入 CI |
| 生成物有唯一来源 | 用 schema 生成多语言类型，再用 fixture 测试守住 |
| 测试确定性 | 外部依赖一律 mock，用快照锁定输出 |
| 局部验证 | 模块拆小，提供按模块运行的 `just test -p` 一类入口 |
| 提交写明原因 | PR 模板中设 `## Why` 和 `## What changed` 两节 |

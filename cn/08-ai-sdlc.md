# 08 让 AI 自动开发、测试、发布

核对基线：commit `430a16799d`。路径相对仓库根目录。

## 1. 总体思路

```
 Issue ──► 分诊 ──► 编码 ──► 自测 ──► 自审 ──► PR CI ──► 人工合并 ──► 人工打 tag ──► 自动发布
          AI        AI       AI+hook   AI       机器       人            人             机器
```

| 阶段 | 执行者 | 关键机制 | 为什么这样分工 |
|---|---|---|---|
| 分诊 | AI | codex-action，只读沙箱，结构化输出 | 量大、规则明确、出错代价低 |
| 编码 | AI | `codex exec` + `workspace-write` 沙箱 | 修改范围限制在工作区内，不能联网 |
| 自测 | AI + hook | Stop hook 不让测试未通过的回合结束 | 用机器强制“先测试再交付”，不依赖模型自觉 |
| 自审 | AI | `codex exec review` | 换一个上下文复查，能发现编码时的盲点 |
| PR CI | 机器 | blocking-ci | AI 写的 PR 与人写的走同一套门禁 |
| 合并 | 人 | GitHub review | 对产品方向和安全负责的只能是人 |
| 发布 | 人触发，机器执行 | 推送 `rust-v*` tag 触发 rust-release | 发布不可撤回，触发权留给人 |

仓库自身目前只把 AI 用在分诊上，`.github/codex/labels/` 下的 4 个提示词（attempt、review、rust-review、triage）都没有被任何 workflow 引用，分诊 workflow 使用内联提示词。下文第 4、7 节的修复与 PR 流水线是按仓库现有模式推导出的示例，仓库本身没有这样做。

## 2. 无人值守执行：`codex exec`

参数定义在 `codex-rs/exec/src/cli.rs` 和 `codex-rs/utils/cli/src/shared_options.rs`。

| 类别 | 参数 |
|---|---|
| 输出 | `--json`（输出 JSONL 事件流）、`-o/--output-last-message FILE`、`--output-schema FILE`、`--color` |
| 环境 | `-C/--cd`、`--add-dir`、`--worktree`、`--skip-git-repo-check`、`--ephemeral`、`--ignore-user-config`、`--ignore-rules`、`--strict-config`、`-m`、`-p`、`-i/--image`、`-c key=value` |
| 权限 | `-s/--sandbox`、`--approve-for-me`、`--dangerously-bypass-approvals-and-sandbox`（别名 `--yolo`）、`--dangerously-bypass-hook-trust` |
| 子命令 | `resume [ID or --last]`、`fork ID`、`review --uncommitted / --base B / --commit SHA` |

行为：

- **审批**：exec 默认把 `approval_policy` 设为 `never`；审批人解析为 `auto_review`（如 `--approve-for-me`）时不强制。源码注释：“Default to never ask for approvals in headless mode. Rebuild below if the fully resolved reviewer is AutoReview.”需要审批的请求会被直接拒绝，进程不会卡住等待输入。
- **stdin**：没有给出 prompt 时，从 stdin 读取。同时给了 prompt 和管道输入时，stdin 的内容以 `<stdin>` 块的形式追加到 prompt 后面。prompt 写 `-` 表示强制从 stdin 读取。
- **退出码**：遇到不可重试的错误、turn 失败或被中断、参数或 schema 解析失败时，退出码为 1。源码注释：“exit with a non-zero status for automation-friendly signaling”。
- **已删除的参数**：`--full-auto`（`1c5f336c40`）。现在必须显式写 `-s workspace-write`，这样每次授予写权限都看得见。

示例：

```bash
codex exec -s workspace-write --json -o result.md \
  "修复 codex-rs/core 中失败的测试，完成后运行 just test -p codex-core" > events.jsonl

codex exec review --uncommitted

git diff | codex exec -s read-only --output-schema review.schema.json -o review.json \
  "审查以下 diff，列出正确性问题"
```

`review.schema.json`：

```json
{
  "type": "object",
  "properties": {
    "approved": { "type": "boolean" },
    "findings": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "file": { "type": "string" },
          "summary": { "type": "string" }
        },
        "required": ["file", "summary"],
        "additionalProperties": false
      }
    }
  },
  "required": ["approved", "findings"],
  "additionalProperties": false
}
```

**为什么用 `--output-schema`**：模型的自由文本没法稳定解析。约束成 JSON Schema 后，下游可以用 `jq` 校验，再决定下一步做什么。

## 3. 无人值守的安全组合

| 层 | 推荐配置 | 为什么 |
|---|---|---|
| 审批 | `approval_policy = "never"`（exec 默认值，`auto_review` 审批人除外） | 不会卡住；需要审批的操作一律拒绝，不会悄悄放行 |
| 沙箱 | `-s workspace-write` | 只能写工作区；默认不能联网 |
| 网络 | 必要时加 `-c sandbox_workspace_write.network_access=true` | 按需开放，默认关闭以防数据外泄 |
| 命令规则 | execpolicy 文件 | 在沙箱之外再禁止一批危险命令 |
| 企业锁定 | `requirements.toml` | 用户和项目配置都无法越过这条底线 |
| 禁止使用 | `danger-full-access` / `--yolo` | 帮助文本写明 “EXTREMELY DANGEROUS. Intended solely for running in environments that are externally sandboxed” |

execpolicy 规则的写法参照 `codex-rs/execpolicy/examples/example.codexpolicy`：

```
prefix_rule(pattern=["git", "reset", "--hard"], decision="forbidden", justification="destructive operation")
prefix_rule(pattern=["git", "push"], decision="forbidden", justification="push is done by the release pipeline")
prefix_rule(pattern=["cargo", "publish"], decision="forbidden", justification="publishing is human-gated")
```

验证规则：

```bash
codex execpolicy check --rules ci.rules git push origin main
```

`requirements.toml` 示例。位置在 `/etc/codex/requirements.toml`，Windows 下在 `%ProgramData%\OpenAI\Codex\`：

```toml
allowed_approval_policies = ["never"]
allowed_sandbox_modes = ["read-only", "workspace-write"]

[allowed_permission_profiles]
":danger-full-access" = false
```

## 4. hooks 闸门

hook 事件枚举 `HookEventName` 定义在 `codex-rs/protocol/src/protocol.rs`，配置结构在 `codex-rs/config/src/hook_config.rs`，输入输出的 schema 在 `codex-rs/hooks/schema/generated/`。

| 事件 | 能做什么 |
|---|---|
| `PreToolUse` | 输出 `permissionDecision: "deny"` 加非空的 `permissionDecisionReason`，或同步 hook 以退出码 2 结束并在 stderr 写明原因，可以阻断工具调用；`updatedInput` 只能与 `permissionDecision: "allow"` 同时使用，用来改写工具输入；不支持 `ask`；`additionalContext` 注入上下文 |
| `PostToolUse` | 检查工具结果，注入反馈 |
| `Stop` | 输出 `decision: "block"` 加非空的 `reason`，回合不会结束，`reason` 成为模型继续工作的提示 |
| `SessionStart`、`UserPromptSubmit` | 注入项目上下文，或拦截不合规的输入 |

配置写在 `.codex/config.toml`：

```toml
[[hooks.PreToolUse]]
matcher = "Bash"

[[hooks.PreToolUse.hooks]]
type = "command"
command = "sh .codex/hooks/gate.sh"

[[hooks.Stop]]

[[hooks.Stop.hooks]]
type = "command"
command = "sh .codex/hooks/require-tests.sh"
timeout = 1800
```

`.codex/hooks/gate.sh`：

```sh
#!/bin/sh
cmd=$(jq -r '.tool_input.command')
case "$cmd" in
  *"git push"*|*"npm publish"*|*"cargo publish"*)
    jq -n '{hookSpecificOutput: {hookEventName: "PreToolUse", permissionDecision: "deny", permissionDecisionReason: "发布操作只能由流水线执行"}}'
    ;;
esac
```

`.codex/hooks/require-tests.sh`：

```sh
#!/bin/sh
[ "$(jq -r '.stop_hook_active')" = "true" ] && exit 0
out=$(cd codex-rs && just test 2>&1) && exit 0
tail_out=$(printf '%s\n' "$out" | tail -n 60)
jq -n --arg r "测试未通过，修复后再结束：$tail_out" '{decision: "block", reason: $r}'
```

**为什么**：
- prompt 只能“请求”模型去做，hook 则由程序强制执行。
- `stop_hook_active` 为 true 表示本回合已被 Stop hook 拦过一次，此时直接放行，避免无限循环。
- 除内置与托管 hook 外，用户级和项目级 hook 都要信任哈希匹配才会执行，防止克隆来的仓库借 hook 执行任意命令。`--dangerously-bypass-hook-trust` 只用于已经审核过 hook 来源的自动化场景。

## 5. 编排接口

| 接口 | 底层 | 适用场景 |
|---|---|---|
| TypeScript SDK（`sdk/typescript`） | 启动 `codex exec --experimental-json`，解析 JSONL | Node 服务、脚本 |
| Python SDK（`sdk/python`） | 通过 stdio 与 `codex app-server` 交换 JSON-RPC | Python 流水线、评测 |
| app-server | JSON-RPC，可走 `stdio://`、`unix://`、`ws://` | IDE、自建平台、长期运行的服务 |

TypeScript，据 `sdk/typescript/README.md` 改写：

```ts
const thread = new Codex().startThread();
const turn = await thread.run("Diagnose the test failure and propose a fix");
const { events } = await thread.runStreamed("...");
await thread.run("Summarize", { outputSchema: schema });
```

Python，摘自 `sdk/python/docs/getting-started.md`：

```python
with Codex() as codex:
    thread = codex.thread_start(sandbox=Sandbox.workspace_write)
    thread.run("...")
```

**为什么有两种底层**：TS SDK 走 exec，部署最简单；Python SDK 走 app-server，可以用完整的协议能力，比如审批回调和线程管理。两者用的是同一个核心运行时（见 [01 §4](01-architecture.md)）。`codex mcp-server` 模式已删除（`531f3836a1`），要让其他 Agent 调用 Codex，请改用 app-server 或 SDK。

## 6. 仓库自身的 AI 自动化：codex-action

| workflow | 作用 |
|---|---|
| `.github/workflows/issue-labeler.yml` | 给新 issue 打标签 |
| `.github/workflows/issue-deduplicator.yml` | 查找重复 issue |
| `.github/workflows/issue-translator.yml` | 翻译非英文 issue |

三者采用同一种模式，取自 `issue-labeler.yml`：

```
job 1  gather-labels   permissions: contents: read
       codex-action    sandbox: read-only / safety-strategy: drop-sudo / output-schema
       输出 final-message（JSON）
          │
          ▼
job 2  apply-labels    permissions: issues: write
       jq 校验 .labels 是数组，拼成一条 gh issue edit 多个 --add-label，失败由 || true 忽略
```

**为什么**：
- 模型所在的 job 没有任何写权限。即使 issue 正文里藏有提示注入，最坏也只是输出一段 JSON。
- 写操作由普通脚本完成，脚本只接受 schema 约束过的字段。schema 只约束标签为字符串；只要有一个标签不存在，整条命令就会失败并被静默忽略，所以应在 jq 中先与允许的标签集合取交集。
- `drop-sudo` 去掉 runner 的 sudo 权限，`read-only` 沙箱禁止写入。
- 仅在 `github.repository == 'openai/codex'` 时运行，fork 仓库不会消耗 API Key。

## 7. 示例：AI 修复 issue 并提交 PR

按第 6 节的模式扩展，未在仓库中实际运行。仓库只用过 `sandbox: read-only`，`workspace-write` 取值是否被 codex-action 接受需对照其文档确认。

```yaml
name: Codex Fix

on:
  issues:
    types: [labeled]

jobs:
  propose:
    if: github.event.label.name == 'codex-fix'
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      - id: codex
        uses: openai/codex-action@5c3f4ccdb2b8790f73d6b21751ac00e602aa0c02
        with:
          openai-api-key: ${{ secrets.CODEX_OPENAI_API_KEY }}
          allow-users: "*"
          safety-strategy: drop-sudo
          sandbox: workspace-write
          prompt: |
            修复 issue #${{ github.event.issue.number }}：${{ github.event.issue.title }}
            ${{ github.event.issue.body }}
            只修改必要的文件；完成后运行 just fmt 与 just test -p <受影响的 crate>。
      - run: git add -A && git diff --cached --binary > codex.patch
      - uses: actions/upload-artifact@v4
        with:
          name: codex-patch
          path: codex.patch

  open-pr:
    needs: propose
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    env:
      GH_TOKEN: ${{ github.token }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: codex-patch
      - run: |
          test -s codex.patch
          git switch -c codex/issue-${{ github.event.issue.number }}
          git apply --index codex.patch
          git -c user.name=codex-bot -c user.email=codex-bot@users.noreply.github.com commit -m "Fix #${{ github.event.issue.number }}"
          git push origin HEAD
          gh pr create --fill --label codex-generated
```

| 设计点 | 为什么 |
|---|---|
| 由人打上 `codex-fix` 标签才触发 | 人来决定哪些 issue 值得交给 AI，控制成本和风险 |
| 模型在只读权限的 job 中运行，只产出 patch；checkout 设 `persist-credentials: false` | token 不写入 `.git/config`，模型无法自行推送或改写仓库 |
| `git add -A` 后导出 `--cached` diff | 包含模型新建的未跟踪文件 |
| `test -s` 检查 patch 非空，再 `git apply --index` | 空 patch 或无法应用时立即失败 |
| PR 走 blocking-ci | AI 写的代码与人写的经过同一套门禁 |
| 合并由人完成 | 见第 1 节 |

## 8. 发布自动化

完整流程见 [06 §6](06-dev-lifecycle.md)。可以交给 AI 的部分和必须留给人的部分：

| 步骤 | 可交给 AI | 为什么 |
|---|---|---|
| 汇总变更日志 | 是：`codex exec -s read-only "汇总 rust-v1.2.3..HEAD 的用户可见变更"` | 只读操作，有错可以由人修正 |
| 提交版本号 | 是，以 PR 形式提交 | tag-check 会校验 tag 与版本号是否一致 |
| 推送 tag | 否 | 推送即发布，无法撤回；需要访问签名与发布用的密钥 |
| 签名、公证、上传 npm 和 R2 | 由 rust-release 自动完成 | 并发锁和 `npm_alpha_tag.py` 防止版本回退 |

## 9. 其他自动化入口

| 入口 | 作用 |
|---|---|
| `codex review` | 与 `exec review` 参数相同，可以在本地审查未提交的改动、指定分支或 commit |
| `codex cloud exec --env ID --attempts 1-4` | 在云端运行任务，`--attempts` 指定同时尝试的次数，从中选最优；`status`、`list --json`、`diff` 查看进度和结果，`apply` 直接应用到本地，不做预检（预检只在 TUI 中进行） |
| `codex exec resume --last` | 续接上一次中断的无人值守任务 |
| `--ephemeral` | 不写 rollout，用于一次性的 CI 任务 |

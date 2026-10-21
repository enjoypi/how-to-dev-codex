> Original (Chinese): [cn/08-ai-sdlc.md](cn/08-ai-sdlc.md)

# 08 Letting AI Develop, Test, and Release Automatically

Verified against: commit `430a16799d`. Paths are relative to the repository root.

## 1. Overall Approach

```
 Issue ──► Triage ──► Code ──► Self-test ──► Self-review ──► PR CI ──► Human merge ──► Human tag ──► Auto release
           AI         AI       AI+hook       AI              Machine   Human           Human         Machine
```

| Stage | Executor | Key mechanism | Why this division of labor |
|---|---|---|---|
| Triage | AI | codex-action, read-only sandbox, structured output | High volume, clear rules, low cost of errors |
| Coding | AI | `codex exec` + `workspace-write` sandbox | Changes are confined to the workspace; no network access |
| Self-test | AI + hook | The Stop hook keeps a turn from ending while tests fail | The machine enforces "test before delivery" instead of relying on the model's discipline |
| Self-review | AI | `codex exec review` | A fresh context catches blind spots from the coding pass |
| PR CI | Machine | blocking-ci | AI-written PRs pass the same gates as human-written ones |
| Merge | Human | GitHub review | Only a human can be accountable for product direction and security |
| Release | Human-triggered, machine-executed | Pushing a `rust-v*` tag triggers rust-release | Releases are irreversible, so the trigger stays with a human |

The repository itself currently uses AI only for triage. None of the 4 prompts under `.github/codex/labels/` (attempt, review, rust-review, triage) is referenced by any workflow; the triage workflows use inline prompts. The fix and PR pipelines in Sections 4 and 7 below are examples inferred from the repository's existing patterns; the repository itself does not do this.

## 2. Unattended Execution: `codex exec`

Flags are defined in `codex-rs/exec/src/cli.rs` and `codex-rs/utils/cli/src/shared_options.rs`.

| Category | Flags |
|---|---|
| Output | `--json` (emits a JSONL event stream), `-o/--output-last-message FILE`, `--output-schema FILE`, `--color` |
| Environment | `-C/--cd`, `--add-dir`, `--worktree`, `--skip-git-repo-check`, `--ephemeral`, `--ignore-user-config`, `--ignore-rules`, `--strict-config`, `-m`, `-p`, `-i/--image`, `-c key=value` |
| Permissions | `-s/--sandbox`, `--approve-for-me`, `--dangerously-bypass-approvals-and-sandbox` (alias `--yolo`), `--dangerously-bypass-hook-trust` |
| Subcommands | `resume [ID or --last]`, `fork ID`, `review --uncommitted / --base B / --commit SHA` |

Behavior:

- **Approval**: exec sets `approval_policy` to `never` by default; this is not enforced when the reviewer resolves to `auto_review` (e.g. `--approve-for-me`). Source comment: "Default to never ask for approvals in headless mode. Rebuild below if the fully resolved reviewer is AutoReview." Requests that need approval are rejected outright, so the process never hangs waiting for input.
- **stdin**: When no prompt is given, the prompt is read from stdin. When both a prompt and piped input are given, the stdin content is appended to the prompt as a `<stdin>` block. A prompt of `-` forces reading from stdin.
- **Exit code**: The exit code is 1 on a non-retryable error, a failed or interrupted turn, or a flag or schema parsing failure. Source comment: "exit with a non-zero status for automation-friendly signaling".
- **Removed flags**: `--full-auto` (`1c5f336c40`). You must now write `-s workspace-write` explicitly, so every grant of write access is visible.

Examples:

```bash
codex exec -s workspace-write --json -o result.md \
  "Fix the failing tests in codex-rs/core, then run just test -p codex-core" > events.jsonl

codex exec review --uncommitted

git diff | codex exec -s read-only --output-schema review.schema.json -o review.json \
  "Review the following diff and list correctness issues"
```

`review.schema.json`:

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

**Why `--output-schema`**: The model's free text cannot be parsed reliably. Once it is constrained by a JSON Schema, downstream steps can validate it with `jq` and then decide what to do next.

## 3. A Safe Configuration for Unattended Runs

| Layer | Recommended setting | Why |
|---|---|---|
| Approval | `approval_policy = "never"` (the exec default, except with the `auto_review` reviewer) | Never hangs; operations that need approval are always rejected, never silently allowed |
| Sandbox | `-s workspace-write` | Writes only to the workspace; no network access by default |
| Network | Add `-c sandbox_workspace_write.network_access=true` when needed | Opened on demand; off by default to prevent data exfiltration |
| Command rules | execpolicy file | Forbids a set of dangerous commands on top of the sandbox |
| Enterprise lockdown | `requirements.toml` | A floor that neither user nor project config can override |
| Never use | `danger-full-access` / `--yolo` | The help text states "EXTREMELY DANGEROUS. Intended solely for running in environments that are externally sandboxed" |

For execpolicy rule syntax, see `codex-rs/execpolicy/examples/example.codexpolicy`:

```
prefix_rule(pattern=["git", "reset", "--hard"], decision="forbidden", justification="destructive operation")
prefix_rule(pattern=["git", "push"], decision="forbidden", justification="push is done by the release pipeline")
prefix_rule(pattern=["cargo", "publish"], decision="forbidden", justification="publishing is human-gated")
```

Verify the rules:

```bash
codex execpolicy check --rules ci.rules git push origin main
```

A `requirements.toml` example. It lives at `/etc/codex/requirements.toml`, or under `%ProgramData%\OpenAI\Codex\` on Windows:

```toml
allowed_approval_policies = ["never"]
allowed_sandbox_modes = ["read-only", "workspace-write"]

[allowed_permission_profiles]
":danger-full-access" = false
```

## 4. Hook Gates

The hook event enum `HookEventName` is defined in `codex-rs/protocol/src/protocol.rs`, the config structure in `codex-rs/config/src/hook_config.rs`, and the input/output schemas in `codex-rs/hooks/schema/generated/`.

| Event | What it can do |
|---|---|
| `PreToolUse` | Block a tool call by emitting `permissionDecision: "deny"` with a non-empty `permissionDecisionReason`, or by having a synchronous hook exit with code 2 and write the reason to stderr; `updatedInput` rewrites the tool input and is valid only together with `permissionDecision: "allow"`; `ask` is not supported; `additionalContext` injects context |
| `PostToolUse` | Inspect tool results and inject feedback |
| `Stop` | Emitting `decision: "block"` with a non-empty `reason` keeps the turn from ending; `reason` becomes the prompt for the model to keep working |
| `SessionStart`, `UserPromptSubmit` | Inject project context, or block non-compliant input |

Configure them in `.codex/config.toml`:

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

`.codex/hooks/gate.sh`:

```sh
#!/bin/sh
cmd=$(jq -r '.tool_input.command')
case "$cmd" in
  *"git push"*|*"npm publish"*|*"cargo publish"*)
    jq -n '{hookSpecificOutput: {hookEventName: "PreToolUse", permissionDecision: "deny", permissionDecisionReason: "Publishing may only be performed by the pipeline"}}'
    ;;
esac
```

`.codex/hooks/require-tests.sh`:

```sh
#!/bin/sh
[ "$(jq -r '.stop_hook_active')" = "true" ] && exit 0
out=$(cd codex-rs && just test 2>&1) && exit 0
tail_out=$(printf '%s\n' "$out" | tail -n 60)
jq -n --arg r "Tests failed; fix them before finishing: $tail_out" '{decision: "block", reason: $r}'
```

**Why**:
- A prompt can only "ask" the model to do something; a hook is enforced by the program.
- `stop_hook_active` being true means the Stop hook has already blocked this turn once, so the hook lets it through to avoid an infinite loop.
- Apart from built-in and managed hooks, user-level and project-level hooks run only if their trust hash matches, which prevents a cloned repository from running arbitrary commands through hooks. Use `--dangerously-bypass-hook-trust` only in automation where the hook source has already been audited.

## 5. Orchestration Interfaces

| Interface | Underlying mechanism | Use cases |
|---|---|---|
| TypeScript SDK (`sdk/typescript`) | Spawns `codex exec --experimental-json` and parses JSONL | Node services, scripts |
| Python SDK (`sdk/python`) | Exchanges JSON-RPC with `codex app-server` over stdio | Python pipelines, evals |
| app-server | JSON-RPC over `stdio://`, `unix://`, or `ws://` | IDEs, in-house platforms, long-running services |

TypeScript, adapted from `sdk/typescript/README.md`:

```ts
const thread = new Codex().startThread();
const turn = await thread.run("Diagnose the test failure and propose a fix");
const { events } = await thread.runStreamed("...");
await thread.run("Summarize", { outputSchema: schema });
```

Python, excerpted from `sdk/python/docs/getting-started.md`:

```python
with Codex() as codex:
    thread = codex.thread_start(sandbox=Sandbox.workspace_write)
    thread.run("...")
```

**Why two underlying mechanisms**: The TS SDK goes through exec, which is the simplest to deploy; the Python SDK goes through app-server, which exposes the full protocol, such as approval callbacks and thread management. Both use the same core runtime (see [01 §4](01-architecture.md)). The `codex mcp-server` mode has been removed (`531f3836a1`); to let other agents call Codex, use app-server or an SDK instead.

## 6. The Repository's Own AI Automation: codex-action

| workflow | Purpose |
|---|---|
| `.github/workflows/issue-labeler.yml` | Labels new issues |
| `.github/workflows/issue-deduplicator.yml` | Finds duplicate issues |
| `.github/workflows/issue-translator.yml` | Translates non-English issues |

All three follow the same pattern, taken from `issue-labeler.yml`:

```
job 1  gather-labels   permissions: contents: read
       codex-action    sandbox: read-only / safety-strategy: drop-sudo / output-schema
       outputs final-message (JSON)
          │
          ▼
job 2  apply-labels    permissions: issues: write
       jq checks that .labels is an array and builds one gh issue edit with multiple --add-label; failures are ignored via || true
```

**Why**:
- The job that runs the model has no write permissions. Even if the issue body hides a prompt injection, the worst outcome is a JSON output.
- Writes are done by a plain script that accepts only schema-constrained fields. The schema only requires labels to be strings; if any single label does not exist, the whole command fails and the failure is silently ignored, so intersect with the set of allowed labels in jq first.
- `drop-sudo` removes sudo privileges from the runner, and the `read-only` sandbox forbids writes.
- It runs only when `github.repository == 'openai/codex'`, so forks do not consume the API key.

## 7. Example: AI Fixes an Issue and Opens a PR

This extends the pattern from Section 6 and has not been run in the repository. The repository has only used `sandbox: read-only`; check the codex-action documentation to confirm whether it accepts the `workspace-write` value.

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
            Fix issue #${{ github.event.issue.number }}: ${{ github.event.issue.title }}
            ${{ github.event.issue.body }}
            Modify only the necessary files; when done, run just fmt and just test -p <affected crate>.
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

| Design point | Why |
|---|---|
| Triggered only when a human applies the `codex-fix` label | A human decides which issues are worth handing to AI, controlling cost and risk |
| The model runs in a read-only job and only produces a patch; checkout sets `persist-credentials: false` | The token is not written to `.git/config`, so the model cannot push or rewrite the repository on its own |
| Export the `--cached` diff after `git add -A` | Includes untracked files the model created |
| `test -s` checks that the patch is non-empty before `git apply --index` | Fails immediately on an empty patch or one that does not apply |
| The PR goes through blocking-ci | AI-written code passes the same gates as human-written code |
| A human performs the merge | See Section 1 |

## 8. Release Automation

For the full flow, see [06 §6](06-dev-lifecycle.md). What can be handed to AI and what must stay with humans:

| Step | Can be handed to AI | Why |
|---|---|---|
| Summarize the changelog | Yes: `codex exec -s read-only "Summarize user-visible changes in rust-v1.2.3..HEAD"` | Read-only; a human can correct mistakes |
| Commit the version bump | Yes, as a PR | tag-check verifies that the tag matches the version |
| Push the tag | No | Pushing is releasing and cannot be undone; it requires the signing and publishing secrets |
| Sign, notarize, upload to npm and R2 | Done automatically by rust-release | A concurrency lock and `npm_alpha_tag.py` prevent version rollback |

## 9. Other Automation Entry Points

| Entry point | Purpose |
|---|---|
| `codex review` | Same flags as `exec review`; reviews uncommitted changes, a given branch, or a commit locally |
| `codex cloud exec --env ID --attempts 1-4` | Runs a task in the cloud; `--attempts` sets the number of parallel attempts to pick the best from; `status`, `list --json`, and `diff` show progress and results; `apply` applies directly to the local tree without a preflight check (preflight runs only in the TUI) |
| `codex exec resume --last` | Resumes the last interrupted unattended task |
| `--ephemeral` | Does not write a rollout; for one-off CI tasks |

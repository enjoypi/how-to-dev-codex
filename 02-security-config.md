> Original (Chinese): [cn/02-security-config.md](cn/02-security-config.md)

# 02 Security, Configuration, and Authentication

Verified against: commit `430a16799d`. Paths are relative to `codex-rs/`.

## 1. Design Principles

Model output is untrusted, so every command must pass approval before it runs in a sandbox. Approval decides "whether to run"; the sandbox limits "what it can do while running". The two layers are independent: if one fails, the other still holds.

## 2. Sandbox

### 2.1 Platform Implementations

| Platform | Implementation | Key locations |
|---|---|---|
| macOS | Generates an SBPL policy and prefixes the command with `/usr/bin/sandbox-exec` | `sandboxing/src/seatbelt*.sbpl`, `sandboxing/src/manager.rs` |
| Linux | Rewrites into a `codex-linux-sandbox` call that runs, in order: bwrap builds the filesystem view → `no_new_privs` → seccomp → `execvp` | `linux-sandbox/src/linux_run_main.rs`, `linux-sandbox/src/seccomp.rs` |
| Windows | Restricted tokens, ACLs, WFP firewall, ConPTY; plus an MXC backend | `windows-sandbox-rs/`, `windows-sandbox-service/`, `mxc-sandbox/` |
| Network | Local HTTP and SOCKS5 proxies that allow or deny by domain | `network-proxy/src/network_policy.rs` |

The unified entry point is `SandboxManager` in `sandboxing/src/manager.rs`: `select_initial` picks the sandbox type and `transform` rewrites the command; the standalone function `get_platform_sandbox` in the same file handles platform mapping. seccomp blocks ptrace, io_uring, and network syscalls. The bubblewrap C source lives in `vendor/bubblewrap/` and is compiled by `bwrap/build.rs`.

### 2.2 Why Linux Moved from Landlock to bwrap + seccomp

| Reason | Evidence |
|---|---|
| bwrap has a more uniform filesystem isolation model: it can specify writable roots, make subdirectories read-only, and offers finer network control | `ae4de43ccc` (#9938) |
| Landlock cannot isolate the app-server's Unix socket | Inferred; `49305d74b4` (#45984) only states that filesystem-restricted execution must use bubblewrap |

Evolution: vendored source (#10413) → Experimental (#11381) → default (#13996) → `use_legacy_landlock` deprecated (#17971) → bwrap packaged separately (#21255).

### 2.3 Policy Data Flow

```
config.toml: sandbox_mode
   │  SandboxMode (protocol/src/config_types.rs)
   ▼
ConfigToml::derive_permission_profile (config/src/config_toml.rs)
   │  read-only          → PermissionProfile::read_only()
   │  workspace-write    → workspace_write(); workspace_write_with(...) when [sandbox_workspace_write] is set
   │  danger-full-access → Disabled
   ▼
PermissionProfile: Managed / Disabled / External (protocol/src/models.rs)
   ▼
core/src/tools/orchestrator.rs ──► SandboxManager::select_initial ──► SandboxType
   ▼
SandboxManager::transform ──► spawn process
```

Three exceptions:

- On Windows without the sandbox enabled, `workspace-write` is downgraded to `read-only`.
- When `sandbox_mode` is unset but the project already has a trust decision (trusted or untrusted), `workspace-write` is the default.
- When `sandbox_mode` is unset and the default is forbidden by requirements, it falls back to `read-only`.

## 3. Approval

### 3.1 Components

| Component | Role |
|---|---|
| `AskForApproval` (`protocol/src/protocol.rs`) | Approval policy: `untrusted`, `on-request`, `granular`, `never` |
| `execpolicy` | Starlark prefix rules yielding `Allow`, `Prompt`, or `Forbidden`; `load_exec_policy` in `core/src/exec_policy.rs` loads `rules/*.rules` from each config layer, with higher layers overriding lower ones |
| guardian | The reviewer is `ApprovalsReviewer::User` or `AutoReview`; `decide_approval` in `core/src/guardian/decision.rs` decides whether to hand off to a reviewer subagent; `guardian-context` assembles the review context |
| `user-verification` | Device credentials and biometric signatures, exposed via `userVerification/*` RPCs; not yet wired into command approval |

### 3.2 Decision Flow

```
Tool declares approval requirement (default_exec_approval_requirement if none)
   │  Skip / NeedsApproval / Forbidden
   ▼
execpolicy rules adjust it (exec_policy.rs)
   │  under the never policy, Prompt is always treated as denial
   ▼
guardian: user means a human confirms, auto_review means a subagent reviews
   ▼
Run inside the sandbox
   │  SandboxErr::Denied and the tool allows escalate_on_failure
   │  and the retry conditions are met (see below)
   ▼
Re-approve if needed, then retry in a looser sandbox or with no sandbox
```

Retry conditions: the tool's `wants_no_sandbox_approval` returns true for the current policy, or under `on-request` the denial was network-related and the default approval requirement is `NeedsApproval`; non-network denials additionally require that the filesystem policy has no read restrictions (`unsandboxed_execution_allowed`). The default implementation of `wants_no_sandbox_approval` returns true only for `untrusted` and for `granular` with `sandbox_approval` enabled; `apply_patch` also returns true for `on-request`; `never` is always false.

**Why**: The layered design lets enterprises, projects, and users each stack their own rules. Under `never`, Prompt is always denied, so unattended runs neither hang nor silently allow. Whether an escalated retry asks again depends on where the first approval came from: if a human or hook already approved, that approval is reused (except with `strict_auto_review` or when network is involved); see `should_bypass_approval` in `core/src/tools/orchestrator.rs` and `core/src/tools/sandboxing.rs`.

## 4. Configuration

### 4.1 Load Order

Defined in `config/src/loader/mod.rs`; priority values are in `config/src/config_layer_source.rs`. From lowest to highest:

| Priority | Layer | Location |
|---|---|---|
| -10 | packaged | Defaults shipped with the package |
| 0 | MDM | Device management |
| 10 | system | `/etc/codex/config.toml` |
| 15 | cloud | Pushed from the enterprise cloud |
| 20 | user | `$CODEX_HOME/config.toml` |
| 21 | profile | `$CODEX_HOME/<name>.config.toml`, selected with `-p <name>` |
| 25 | project | `.codex/`; loaded but not applied when the directory is untrusted |
| 30 | session | Command-line overrides such as `-c key=value` |
| 40, 50 | legacy managed_config | Legacy managed configuration |

### 4.2 Constraint Layer

`requirements.toml` comes from system, cloud, and MDM; on Unix, the legacy `/etc/codex/managed_config.toml` is also read as requirements. It is independent of the table above and restricts allowed values. `Constrained::can_set` can forbid specific `sandbox_mode` values, approval policies, or reviewers.

**Why**: Config layers answer "which value to use"; the constraint layer answers "which values are allowed". Even when an enterprise cannot control a user's `config.toml`, it can still enforce a baseline that cannot be bypassed.

### 4.3 features

- `features/src/lib.rs`: the registry is `FEATURES`; `Stage` values are UnderDevelopment, Experimental, Stable, Deprecated, Removed.
- `Features::from_sources` merges in the order defaults → base → profile → overrides, while still honoring legacy toggles.

**Why**: Runtime toggles replace Cargo features, so a single binary can roll out different features gradually and avoid a combinatorial explosion of builds (see [01 §10](01-architecture.md)).

### 4.4 schema

`config-schema` uses schemars to generate `core/config.schema.json`, and `core/src/config/schema_tests.rs` checks that the generated output matches the fixture. If you change a config field without regenerating the schema, the test fails.

### 4.5 Other

- `codex-home`: loads global instructions; if `AGENTS.override.md` is non-empty, only it is used, otherwise `AGENTS.md`.
- `utils/home-dir`: locates `CODEX_HOME`, defaulting to `~/.codex`.
- `cloud-config`: fetches and caches the cloud config bundle, with a 1-hour TTL and a refresh every 15 minutes.

## 5. Authentication and Secrets

| crate | Responsibility |
|---|---|
| `login` | `CodexAuth` (API Key, ChatGPT, AgentIdentity, PAT, Bedrock, etc.); `AuthManager` refreshes tokens; `create_auth_storage` supports File, Keyring, Auto, Ephemeral |
| `keyring-store` | System keychain abstraction `KeyringStore` |
| `secrets` | Encrypts local secrets with age/scrypt, with the passphrase stored in the keychain; `redact_secrets` redacts by regex |
| `aws-auth` | AWS profile and SigV4 for Bedrock |
| `agent-identity`, `workload-identity` | Agent identity JWTs, and exchanging workload assertions for authentication |
| `websocket-auth` | Validates WebSocket upgrade requests for app-server and exec-server; does not check revocation after the connection is established |

**Storage defaults** (`config/src/types.rs`):

| Enum | Default | Notes |
|---|---|---|
| `AuthCredentialsStoreMode` | `File` | Login credentials are written to `$CODEX_HOME/auth.json`; `Keyring` errors when the keychain is unavailable; `Auto` prefers the keychain and falls back to the file |
| `OAuthCredentialsStoreMode` | `Auto` | MCP OAuth credentials prefer the keychain; once a client picks a store it never switches, avoiding stale credentials on refresh |

**Why**: Login credentials default to a file for compatibility with headless servers and containers (inferred). The reason MCP credentials lock their store is given in that enum's comments.

## 6. Known Limitations

- `pre_main_hardening` in `process-hardening` (disables core dumps, blocks ptrace, clears `LD_*`/`DYLD_*`) is called only by `responses-api-proxy` and `voice-host`; the linux-sandbox proxy lifecycle calls only `disable_process_dumping`; the main CLI calls neither.
- `user-verification` is not yet wired into the approval chain.
- `websocket-auth` does not check expiry or revocation after the connection is established.

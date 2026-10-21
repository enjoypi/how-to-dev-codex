# 02 安全、配置与认证

核对基线：commit `430a16799d`。路径相对 `codex-rs/`。

## 1. 设计原则

模型输出不可信，所以每条命令都要先过审批，再放进沙箱执行。审批决定“要不要执行”，沙箱限制“执行时能做什么”。两层独立，任一层失效时另一层仍然有效。

## 2. 沙箱

### 2.1 平台实现

| 平台 | 实现 | 关键位置 |
|---|---|---|
| macOS | 生成 SBPL 策略，命令前加 `/usr/bin/sandbox-exec` | `sandboxing/src/seatbelt*.sbpl`、`sandboxing/src/manager.rs` |
| Linux | 改写为 `codex-linux-sandbox` 调用，依次执行 bwrap 构建文件系统视图 → `no_new_privs` → seccomp → `execvp` | `linux-sandbox/src/linux_run_main.rs`、`linux-sandbox/src/seccomp.rs` |
| Windows | 受限令牌、ACL、WFP 防火墙、ConPTY；另有 MXC 后端 | `windows-sandbox-rs/`、`windows-sandbox-service/`、`mxc-sandbox/` |
| 网络 | 本地 HTTP 与 SOCKS5 代理，按域名放行或拒绝 | `network-proxy/src/network_policy.rs` |

统一入口是 `sandboxing/src/manager.rs` 中的 `SandboxManager`：`select_initial` 选择沙箱类型，`transform` 改写命令；同文件的独立函数 `get_platform_sandbox` 负责平台映射。seccomp 禁用 ptrace、io_uring 和网络类 syscall。bubblewrap 的 C 源码在 `vendor/bubblewrap/`，由 `bwrap/build.rs` 编译。

### 2.2 为什么 Linux 从 Landlock 换成 bwrap + seccomp

| 原因 | 证据 |
|---|---|
| bwrap 的文件系统隔离模型更统一：可以指定可写根目录、把子目录设为只读，网络控制也更细 | `ae4de43ccc`（#9938） |
| Landlock 无法隔离 app-server 的 Unix socket | 推断；`49305d74b4`（#45984）只写明文件系统受限执行必须使用 bubblewrap |

演进过程：vendored 源码（#10413）→ Experimental（#11381）→ 默认（#13996）→ 弃用 `use_legacy_landlock`（#17971）→ 独立打包 bwrap（#21255）。

### 2.3 策略数据流

```
config.toml: sandbox_mode
   │  SandboxMode（protocol/src/config_types.rs）
   ▼
ConfigToml::derive_permission_profile（config/src/config_toml.rs）
   │  read-only          → PermissionProfile::read_only()
   │  workspace-write    → workspace_write()；有 [sandbox_workspace_write] 时为 workspace_write_with(...)
   │  danger-full-access → Disabled
   ▼
PermissionProfile：Managed / Disabled / External（protocol/src/models.rs）
   ▼
core/src/tools/orchestrator.rs ──► SandboxManager::select_initial ──► SandboxType
   ▼
SandboxManager::transform ──► 启动进程
```

三个例外：

- Windows 未启用沙箱时，`workspace-write` 会降级为 `read-only`。
- 未设置 `sandbox_mode` 但项目已有信任决定（trusted 或 untrusted）时，默认使用 `workspace-write`。
- 未设置 `sandbox_mode` 且默认值被 requirements 禁止时，回退为 `read-only`。

## 3. 审批

### 3.1 组成

| 组件 | 作用 |
|---|---|
| `AskForApproval`（`protocol/src/protocol.rs`） | 审批策略：`untrusted`、`on-request`、`granular`、`never` |
| `execpolicy` | Starlark 前缀规则，结果为 `Allow`、`Prompt` 或 `Forbidden`；`core/src/exec_policy.rs` 的 `load_exec_policy` 从各配置层加载 `rules/*.rules`，高层覆盖低层 |
| guardian | 审批人取 `ApprovalsReviewer::User` 或 `AutoReview`；`core/src/guardian/decision.rs` 的 `decide_approval` 决定是否交给审查子代理；`guardian-context` 负责组装审查上下文 |
| `user-verification` | 设备凭据与生物识别签名，通过 `userVerification/*` RPC 暴露；尚未接入命令审批 |

### 3.2 决策流程

```
工具声明审批需求（无声明时用 default_exec_approval_requirement）
   │  Skip / NeedsApproval / Forbidden
   ▼
execpolicy 规则修正（exec_policy.rs）
   │  never 策略下 Prompt 一律视为拒绝
   ▼
guardian：user 由人确认，auto_review 由子代理审查
   ▼
沙箱内执行
   │  SandboxErr::Denied 且工具允许 escalate_on_failure
   │  且满足重试条件（见下）
   ▼
按需再次审批，然后在更宽松的沙箱或无沙箱中重试
```

重试条件：工具的 `wants_no_sandbox_approval` 对当前策略返回 true，或 `on-request` 下因网络被拒且默认审批需求为 `NeedsApproval`；非网络拒绝还要求文件系统策略没有禁读限制（`unsandboxed_execution_allowed`）。`wants_no_sandbox_approval` 的默认实现只对 `untrusted` 和开启 `sandbox_approval` 的 `granular` 返回 true；`apply_patch` 另对 `on-request` 返回 true；`never` 一律为 false。

**为什么**：分层设计让企业、项目、用户各自叠加规则；`never` 时 Prompt 一律拒绝，保证无人值守不会卡住，也不会悄悄放行。升级重试是否再询问取决于首次批准的来源：人或 hook 已批准时沿用该批准（`strict_auto_review` 或涉及网络时除外），见 `core/src/tools/orchestrator.rs` 与 `core/src/tools/sandboxing.rs` 中的 `should_bypass_approval`。

## 4. 配置

### 4.1 加载顺序

定义在 `config/src/loader/mod.rs`，优先级数值在 `config/src/config_layer_source.rs`。从低到高：

| 优先级 | 层 | 位置 |
|---|---|---|
| -10 | packaged | 随包默认值 |
| 0 | MDM | 设备管理 |
| 10 | system | `/etc/codex/config.toml` |
| 15 | cloud | 企业云端下发 |
| 20 | user | `$CODEX_HOME/config.toml` |
| 21 | profile | `$CODEX_HOME/<name>.config.toml`，由 `-p <name>` 选择 |
| 25 | project | `.codex/`，目录未被信任时加载但不生效 |
| 30 | session | `-c key=value` 等命令行覆盖 |
| 40、50 | legacy managed_config | 旧式托管配置 |

### 4.2 约束层

`requirements.toml` 来自 system、cloud、MDM，Unix 上还会把旧式 `/etc/codex/managed_config.toml` 当作 requirements 读取，独立于上表，用来限制可选值。`Constrained::can_set` 可以禁止某些 `sandbox_mode`、审批策略或审批人。

**为什么**：配置层回答“取什么值”，约束层回答“允许取哪些值”。企业即使无法控制用户的 `config.toml`，也能确保底线不被突破。

### 4.3 features

- `features/src/lib.rs`：注册表是 `FEATURES`，阶段 `Stage` 分为 UnderDevelopment、Experimental、Stable、Deprecated、Removed。
- `Features::from_sources` 按 默认值 → base → profile → overrides 合并，同时兼容旧开关。

**为什么**：用运行时开关代替 Cargo features，同一个二进制就能灰度不同功能，还避免了编译组合爆炸（见 [01 §10](01-architecture.md)）。

### 4.4 schema

`config-schema` 用 schemars 生成 `core/config.schema.json`，`core/src/config/schema_tests.rs` 校验生成结果与 fixture 一致。改了配置字段却没重新生成 schema，测试就会失败。

### 4.5 其他

- `codex-home`：加载全局指令，`AGENTS.override.md` 非空时只用它，否则用 `AGENTS.md`。
- `utils/home-dir`：定位 `CODEX_HOME`，默认 `~/.codex`。
- `cloud-config`：拉取云端配置包并缓存，TTL 1 小时，每 15 分钟刷新。

## 5. 认证与密钥

| crate | 职责 |
|---|---|
| `login` | `CodexAuth`（API Key、ChatGPT、AgentIdentity、PAT、Bedrock 等）；`AuthManager` 刷新令牌；`create_auth_storage` 可选 File、Keyring、Auto、Ephemeral |
| `keyring-store` | 系统钥匙串抽象 `KeyringStore` |
| `secrets` | 用 age/scrypt 加密本地密钥，口令存放在钥匙串；`redact_secrets` 按正则脱敏 |
| `aws-auth` | Bedrock 的 AWS profile 与 SigV4 |
| `agent-identity`、`workload-identity` | Agent 身份 JWT，以及工作负载断言换取认证 |
| `websocket-auth` | 校验 app-server 与 exec-server 的 WebSocket 升级请求；不检查连接建立后的吊销 |

**存储默认值**（`config/src/types.rs`）：

| 枚举 | 默认 | 说明 |
|---|---|---|
| `AuthCredentialsStoreMode` | `File` | 登录凭据写入 `$CODEX_HOME/auth.json`；`Keyring` 在钥匙串不可用时报错；`Auto` 优先钥匙串，不可用时回退文件 |
| `OAuthCredentialsStoreMode` | `Auto` | MCP OAuth 凭据优先钥匙串；同一客户端一旦选定存储就不再切换，避免刷新时读到过期凭据 |

**为什么**：登录凭据默认用文件，兼容无图形界面的服务器与容器，属推断。MCP 凭据锁定存储的原因见该枚举的注释。

## 6. 已知边界

- `process-hardening` 的 `pre_main_hardening`（禁 core dump、禁 ptrace、清理 `LD_*`/`DYLD_*`）只被 `responses-api-proxy`、`voice-host` 调用；linux-sandbox 的代理生命周期只调用 `disable_process_dumping`；主 CLI 都不调用。
- `user-verification` 尚未接入审批链路。
- `websocket-auth` 不检查连接建立后的过期或吊销。

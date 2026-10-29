# 分层授权与凭据生命周期

[手册目录](../README.md)

适用范围：Maka 固定提交 `15d0a0dd19d39e2bc09ae35940e432c636982315`，核对日期 2026-10-24。
Pi 对照提交为 `482c1381a00d8d00bc81697e4a980c593db6803e`，本章引用的 Pi 文件无工作区差异。
本章只读源码与测试，不读取用户凭据，不调用模型或 OAuth 服务；运行轨迹是静态核对，不代表本次执行过测试。
Managed Secret 按已实现的独立组件介绍，不能据此认定默认 Host/CLI 已接入完整的云端秘密生命周期。

## 1. 问题：连接能力与执行授权的分离

假设同事通过共享 Session，委托 Agent 读取发布说明、从浏览器查询工单，并撰写一份报告。
此处至少存在四个独立问题：同事能否观察该 Session；能否请求启动一轮；Agent 能否操作已登录的浏览器；撰写报告的进程能否访问目标文件。
模型服务所需的 API key 是第五个独立问题。持有该 key 仅说明可向服务认证，不表示获准读取同事的个人目录。

认证是证明请求方持有可接受的身份凭据；授权是决定该主体可以执行哪些动作；秘密注入是将凭据交给实际调用外部服务的组件。
因此，将 token 写入配置、将工具列给模型、点击一次批准，不能概括为同一个“权限已打开”。

| 要检查的层 | Maka 中的对象 | 不能替代的检查 |
| --- | --- | --- |
| Host 连接身份 | `RuntimeHostConnectionAuthority` | 连接成功不等于所有 operation 获准 |
| Host 操作许可 | `operationGrants`、发布能力和 Host 路径开关 | 不直接批准 Bash 或浏览器动作 |
| 共享会话许可 | `SessionCollaborationGrant` | 观察 Session 不等于直接启动 Turn |
| Client Capability 使用许可 | 会话 grant 与 Host admission | 发布工具不等于每个会话都能使用 |
| 工具执行与系统访问 | `ToolRuntime`、`ExecutionBoundary`、OS sandbox | 工具可见性与实际进程权限不同 |
| 外部服务凭据 | 本地凭据库或 Managed Secret | 本地许可不能扩大服务端 token 的权限 |

上述分界可分别在[连接授权][m-connection]、[Guest 授权][m-guest]、[工具运行时][m-runtime]和[秘密接口][m-managed]核对。

## 2. Host 身份、operation grant 与 Guest

`FileRuntimeHostAccessAuthority.authenticate()` 对传入凭据计算 hash，与存储记录作 `timingSafeEqual()` 比较；只接受 active 或未过期的 pending 记录。
Host 访问记录保存 `credentialId/credentialHash/principalId/principalKind/status` 等字段，不能与后文保存模型 API key 的库混为一谈。[认证入口][m-authenticate]

认证结果的实际字段包括 `operationGrants`、`canPublishClientCapabilities`、`canUseHostPaths`，以及可选的 `clientInstanceId/capabilityOwner`。
主体种类是 `local_owner`、`remote_owner`、`capability_provider`、`session_guest`。本地 owner 常量使用 `local_os_user` 和全部 operation；远程主体不能仅凭名称自称本地 owner。
`authorizeRuntimeHostOperation()` 还检查 operation 是否允许 remote owner、是否在 grants 中，以及发布能力、Host 路径开关。[授权函数][m-connection]

持久记录内部字段名是 `grants`，磁盘编码名才是 `operationGrants`。`effectiveOperationGrants()` 按当前协议重新计算有效操作：未知旧键可保留在记录中，但不会因此获得执行权。[有效授权][m-effective]
`capability_provider` 当前只有 `host.status`、`client.capability.replace`、`client.capability.unregister` 三种发布侧操作，不能等同于远程控制员。
需要绑定客户端的 pending 凭据在 finalize 前只取得状态查询和 finalize 权限；完成配对后也不跳过后续 operation 检查。

Guest 的 operation 白名单包含共享查询、订阅、转录分页和 `collaboration.turn-request.*`，不包含直接 `turn.start`。
它还需要绑定具体 Session 的 grant：`session_observation` 或 `session_turn_request`。存储解码拒绝将同一 Guest 授予多个不同 Session。
`createTurnAccessRequest()` 检查目标 Session 的 `session_turn_request` grant；`decideTurnAccessRequest()` 再记录 Owner 决定，批准后的 admission 仍可能是 `started/blocked/failed`。[Guest 实现][m-guest]

这回答了示例中的第一处疑问：同事可以提出运行请求，但“已收到请求”“Owner 已批准”“运行已启动”是三个不同的状态。

## 3. 工具权限：模式制与分类弹窗制的差异

当前 `PERMISSION_MODES` 只有 `explore/ask/bypass`。`execute` 已废弃，仅 `decodePersistedPermissionMode()` 将旧记录中的该模式折叠为 `ask`；新输入的严格检查不接受该模式。[模式定义][m-mode]

| 模式 | 初始执行边界 | 阅读源码时应注意 |
| --- | --- | --- |
| `explore` | managed、只读 profile | 限制来自实际 profile 和执行路径，不是模式名称字符串 |
| `ask` | managed、workspace-write profile | 并非每次 Bash 都必然出现分类审批；越界扩张和 Client Capability 有各自流程 |
| `bypass` | bypass boundary | 跳过 Maka 本地沙箱限制，不增加 Host operation grant 或外部服务权限 |

映射入口是 `createGenesisExecutionBoundary()`；OS 后端与 Windows 限制见[执行环境与操作系统沙箱](sandbox.md)。[初始边界][m-genesis]
`ToolRuntime.executeTool()` 检查参数、step admission、工具是否已加载，并在执行时读取可信的 `readExecutionBoundary()` 和 `readPermissionMode()`。
`discovery/categoryHint/permissionArgs` 分别涉及发现、分类和审批／持久化参数投影；不能仅凭这些字段判断执行已获批准。[运行时实现][m-runtime]

尤其不应将 `permission.ts` 中保留的 legacy payload 或分类注释视为当前的统一审批引擎。
本地工具通过执行上下文使用边界；当前扩张入口是 `request_sandbox_boundary`，参数是 `expansion` 与 `justification`，且要求独占一个 assistant step。[扩张工具][m-boundary-tool]
`expansion.filesystem.entries[]` 使用 `path/access/scope`，其中 access 为 `read/write`，scope 为 `exact/subtree`；网络扩张是 `network: { enabled: true }`，没有“自动切换 bypass”字段。
请求保存 `baseRevision`，结算区分 `approved/denied/conflict`；这是会话边界扩张，不能解释为仅批准一次命令的执行。[边界数据][m-boundary]
`ToolRuntime` 在本轮明确拒绝后阻止再次申请扩张；缺少交互承载能力时直接失败，而非假定用户已批准。[扩张等待与结算][m-boundary-runtime]

## 4. Client Capability：发布、接受与获准执行的分离

Client Capability 是客户端提供、由客户端执行的能力，例如浏览器或桌面 MCP。Host 的本地 OS 沙箱不能限制客户端机器上的效果。
`ToolRuntime` 以 `hostAdmission: 'client_capability'` 识别这条路径，而不是相信客户端工具的类别标签；对应工具必须提供 `prepareExecution()`。[准入分支][m-client-admission]

此处存在一处文案与条件的差异：错误常量的文案称需要 Bypass，但实际分支允许“boundary 为 bypass，或者实时 permissionMode 为 ask”进入准备阶段。
因此不能表述为“所有 Client Capability 只能在 bypass 中使用”。进入准备阶段也不等于已经派发执行。

协议的完整时序（accepted → 会话审批 → T1 → admitted）、browser-call 等待与取消的真实测试顺序，以及会话 grant 的键结构与 scope 分类，在[客户端能力协议](mcp-and-client-capabilities.md)展开。
要点：`accepted` 不是执行许可，进入准备阶段也不等于已经派发；grant 的键包含 sessionId/providerId/contractId/serverId/toolName/capability/scope，迁移批准不能仅复制一个“allow”布尔值。

## 5. 本地凭据：凭据库与秘密流向

最新源码同时存在以下入口，文件名不是可互换的别名。

| 存储 | 实际记录与用途 | 当前保护边界 |
| --- | --- | --- |
| `runtime-host-access.json` | Host 接入凭据 hash、主体、操作与协作授权 | 认证／撤销状态，不是模型 token 库 |
| `credential-vault.json` | Runtime Policy 凭据；entry 带 locator、credentialId、revision、secret、updatedAt | 普通 JSON；快照不返回 secret，内部 material 才包含它 |
| `credentials.json` | `CredentialStore` 的 `slug + kind` 字符串值；仍供 MCP OAuth 等调用者使用 | 明文 JSON、文件权限、原子写与更新锁，不是 OS keychain |
| `managed-secrets.json` | 独立 Managed Secret 后端的密文记录与会话 grants | AES-256-GCM；密钥由外部 keyProvider 提供 |

前三项分别可查[Host 记录][m-effective]、[vault 实现][m-vault]、[本地 store][m-local-store]；第四项见[加密后端][m-encrypted]。
vault 的 connection locator 精确使用 `{ scope: 'connection', connectionId, kind }`，kind 为 `api_key/oauth_token/request_headers`；另有 web_search、network_proxy、jev scope。[locator 解码][m-locator]
`prepareSet()` 用 expected credentialId/revision 防止旧写覆盖新值，冲突返回 `credential_stale`；`vaultSnapshot()` 只投影配置状态和版本。
`writeJsonDocument()` 创建 `0600` 临时文件、同步并 rename；该操作不属于加密。Windows 实际文件访问取决于 ACL，不能将 POSIX mode 数字视为跨平台隔离承诺。[文档写入][m-vault-io]

模型凭据的消费示例是 `resolveExecutionTarget()`：

```text
Session 的 connection 标识与 model
  → runtimePolicy.operations.resolveExecutionConnection(...)
  → 检查 ready、provider 与允许使用的 model
  → secretMaterial.connection.secret → apiKey
  → secretMaterial.requestHeaders.secret → parseRequestHeaders()
  → OAuth 路径改为 oauthCredentials.bind(material)
runHostAuxiliaryModelCall()
  → OAuth 时 binding.resolve() 取得 access_token
  → getAIModel({ apiKey, requestHeaders, fetch, ... })
```

此处秘密作为模型适配器与传输参数注入，无需写入用户消息或工具参数。[执行目标与调用][m-model]
该链路与 Bash 的环境继承不同；本章未声称全部工具输出、扩展或日志都自动获得秘密隔离。

`createCredentialMcpOAuthStorage()` 将 MCP OAuth 整条记录存为 `mcp:<serverId>` 的 `oauth_token` 值，而非写入可分享的 MCP 配置。
它在底层支持时使用 compare-and-set；`FileCredentialStore.compareAndSetSecret()` 在锁内核对原值，区分并发更新与已删除，防止旧刷新结果覆盖新状态。[MCP 存储适配][m-mcp-store]
因此，不能仅依据旧 store 的注释，便宣称当前所有模型连接仍从 `credentials.json` 取值。

## 6. Managed Secret：引用与授权、清理与撤销的区分

`ManagedSecretReference` 只有 `{ schemaVersion: 1, secretId }`。metadata 另带 ownerPrincipalId、revision、status 和时间；可携带引用不代表可取得 value。[接口定义][m-managed]
可信调用者提供 `{ principalId, cloudSessionId, activationId }`；store 自身不认证这些字段，也不证明 Session 属于该 principal，控制平面必须先行建立这一关系。
`authorizeSession()` 保存指定秘密与目标 Cloud Session 的授权；`resolveForActivation()` 检查 owner、active 状态和 grant 后才交回材料。
测试明确断言：源 Session 已授权，复制引用到目标 Session 后仍是 `unauthorized`，直到目标也被授权。[授权与轮换测试][m-managed-test]

加密文件后端已实现，并非仅处于规划阶段：`sealSecret()` 使用 AES-256-GCM，附加认证数据绑定 secretId、owner 和 revision；keyProvider 负责提供 32 字节密钥，密钥不与密文一同存放。
但它是参考后端，不是已经接入的 KMS/HSM。当前检索的非测试 TypeScript 调用者中未见该 factory 或 injector 的生产装配调用，本章不将组件能力表述为默认产品保证。[加密实现][m-encrypted]

注入测试使用两个假值，以下顺序由 `ActivationSecretInjector.prepare()` 与测试共同确认：[注入实现][m-injector]、[注入测试][m-inject-test]

```text
createSecret → authorizeSession
prepare({ context, bindings, sink })
  → 校验 reference 与环境变量名；拒绝大小写折叠后的重复名
  → store.resolveForActivation()：整批解析，任一缺授权则不注入
  → 校验材料数量与引用顺序，建立私有快照
  → sink.injectEnvironmentVariables({ entries })：一次提交整批
  → 返回 references、redact()、release()，不公开 value 字段
release() → 恢复 FIRST_TOKEN 原值，删除新增 SECOND_TOKEN，保留无关变量
```

`ActivationEnvironmentSecretSink` 修改调用者传入的环境对象，不会替调用者创建进程或隔离环境；应为其提供新的 launch overlay，而非全局 `process.env`。
`redact()` 只按已知字面值替换，无法保证识别编码、拆分后的秘密；release 后清空此句柄保留的脱敏值，因此调用者应先排空相关诊断输出。
清理失败后可以再次 release；该操作不保证已复制的字符串从内存中彻底擦除，也不能收回已进入子进程的值。
rotate/revoke 使用 expectedRevision；撤销将阻止后续解析，但不会自动撤销外部服务已经签发的 token，更不会终止所有持有旧值的进程。

## 7. 与 Pi 对照：接口存在与默认统一 ACL 的差距

本表讨论 Pi 默认 `coding-agent/agent/ai` 的相关入口与可选扩展；实验 durable 是另一条执行路径，不能从本表推断其缺少恢复或编排能力。

| 机制问题 | Maka 本章已核对实现 | Pi 对照与范围 |
| --- | --- | --- |
| 谁能调用 Host operation | 连接主体、operationGrants、Guest Session grants 分层 | 本章的 AuthStorage、project trust 和工具钩子不构成等价的统一 Host ACL |
| 凭据保存 | Runtime Policy vault；另有本地 CredentialStore | `AuthStorage` 默认文件后端保存 auth.json 普通 JSON，可换内存后端 |
| 并发更新 | vault 版本 basis；本地 store 锁内 CAS | `modify(provider, fn)` 锁内重读并仅合并目标 provider |
| 项目内容能否进入进程 | 不应与执行边界混为一谈 | `resolveProjectTrusted()` 决定项目设置、资源、包及扩展的加载信任 |
| 工具执行前是否拦截 | ToolRuntime 准入；Client Capability 先准备再派发 | 扩展 `tool_call` 可返回 block；具体规则来自安装的扩展 |
| 实际系统资源限制 | managed profile 交给平台后端实施 | 可选 sandbox/SSH/env 路径另行配置，见[沙箱实现](sandbox.md)；信任或钩子不替代 OS 限制 |
| 按 Session 取用秘密 | Managed Secret 独立组件有显式 Session grant | 本章 AuthStorage 接口按 provider 存取，不应推断默认每租户秘密 ACL |

Pi `AuthStorage.list()` 只返回 providerId/type；`read()` 会解析 API key 的配置值，支持字面值、环境模板和 `!command`。
`!command` 在配置解析路径调用宿主 shell，不是一次模型工具调用，不能期望 permission-gate 拦截该调用。[AuthStorage][p-auth]、[配置解析][p-resolve]
文件后端新建文件请求 `0600`、新建目录请求 `0700`，已存在文件保留原有模式／ACL；其后端不提供默认加密存储。[文件后端][p-auth-file]

`resolveProjectTrusted()` 首先读取 override；无须信任的项目资源直接通过，否则再检查扩展决定、已记住的决定和默认策略，需要询问但无 UI 时返回 false。[项目 trust][p-trust]
`permission-gate.ts` 仅为可选示例：仅检查 bash 的少量危险正则，存在 UI 时才询问，无 UI 时阻止命中的命令；未覆盖所有 shell 语义、文件工具和 MCP。[拦截示例][p-gate]
加载扩展本身意味着运行宿主进程内代码。以上接口不能组合成“Pi 默认统一 ACL”，也不能据此表述为“Pi 无权限控制”。

## 8. 源码定位

| 入口 | 函数／类型与阅读问题 |
| --- | --- |
| [access-authority.ts][m-authenticate] | `authenticate()`：凭据有效与 operation 获准如何分开 |
| [connection-authority.ts][m-connection] | `authorizeRuntimeHostOperation()`：grants、发布和路径限制 |
| [access-credential-store.ts][m-effective] | `effectiveOperationGrants()`：持久授权与当前有效授权 |
| [permission.ts][m-mode] | `decodePersistedPermissionMode()`：旧 execute 如何迁移 |
| [tool-runtime.ts][m-client-admission] | `executeTool()`：Client Capability 的准备阶段为何先于 T1 |
| [sandbox-boundary-tool.ts][m-boundary-tool] | `buildRequestSandboxBoundaryTool()`：当前扩张请求形状 |
| [credential-vault-document.ts][m-vault] | `prepareSet()/vaultSnapshot()/credentialMaterial()`：版本和秘密投影 |
| [execution-model-authority.ts][m-model] | `resolveExecutionTarget()/runHostAuxiliaryModelCall()`：秘密最终交给谁 |
| [credential-store.ts][m-local-store] | `createFileCredentialStore()/compareAndSetSecret()`：本地文件更新 |
| [managed-secret-store.ts][m-managed] | `authorizeSession()/resolveForActivation()`：引用不携带授权 |
| [activation-secret-injector.ts][m-injector] | `ActivationSecretInjector.prepare()`：批量注入与清理句柄 |
| [Pi auth-storage.ts][p-auth] | `AuthStorage.read()/modify()/list()`：解析、修改与元数据读取 |

## 9. 本章术语

| 术语 | 本章含义 |
| --- | --- |
| principal | 被可信连接或控制平面识别的主体，不是模型填写的用户名 |
| operation grant | 调用某个 Host 协议操作的许可，不是通用工具执行许可 |
| admission | 真正派发前的准入阶段；accepted 不代表 admitted |
| execution boundary | 会话当前的 managed/bypass/external 边界与版本 |
| credential basis / CAS | 以已读身份、版本或原值为条件更新，防止旧值覆盖新值 |
| secret reference | 不含秘密值的引用；解析还需目标 Session 授权 |
| injection lease | 持有本次注入效果及清理方法的句柄，不是外部服务的 token 有效期 |

## 10. 源码练习

1. 只读 Client Capability 集成测试，解释三次 accepted 中为何只有一次 T1；将实时模式设想为 explore，标出最早的拒绝位置；再说明 canPublishClientCapabilities 为何不能替代 Session grant。
2. 使用纸面数据或 `InMemoryManagedSecretStore` 创建 `example-only` 假值：授权 Session A，将引用复制到 B，再撤销 A 的授权；预测每次 resolve 的结果。无需读取任何凭据文件或调用 API。
3. 用 `AuthStorage.inMemory()` 保存两个假 provider，修改其中之一并比较 list/read。分别判断“项目信任”“permission-gate 放行”“API key 有效”能否保证 Bash 不能访问工作区外路径，并指出缺失的检查层。

## 相关专题

- [操作系统沙箱](sandbox.md)：考察执行边界如何落实到进程与文件。
- [客户端能力协议](mcp-and-client-capabilities.md)：展开 accepted、审批和 admitted 的完整交接。
- [模型配置](../01-runtime/provider-configuration.md)：追踪连接、凭据到模型实例的装配。

[m-connection]: ../../../../maka/packages/runtime-host/src/server/connection-authority.ts#L103
[m-authenticate]: ../../../../maka/packages/runtime-host/src/server/access-authority.ts#L221
[m-guest]: ../../../../maka/packages/runtime-host/src/server/access-authority.ts#L411
[m-effective]: ../../../../maka/packages/runtime-host/src/server/access-credential-store.ts#L198
[m-mode]: ../../../../maka/packages/core/src/permission.ts#L26
[m-genesis]: ../../../../maka/packages/core/src/sandbox-boundary.ts#L229
[m-boundary]: ../../../../maka/packages/core/src/sandbox-boundary.ts#L53
[m-runtime]: ../../../../maka/packages/runtime/src/tool-runtime.ts#L1088
[m-client-admission]: ../../../../maka/packages/runtime/src/tool-runtime.ts#L1415
[m-boundary-runtime]: ../../../../maka/packages/runtime/src/tool-runtime.ts#L2559
[m-boundary-tool]: ../../../../maka/packages/runtime/src/sandbox-boundary-tool.ts#L72
[m-vault]: ../../../../maka/packages/storage/src/runtime-policy/credential-vault-document.ts#L76
[m-locator]: ../../../../maka/packages/core/src/runtime-policy/credential-vault-codec.ts#L43
[m-vault-io]: ../../../../maka/packages/storage/src/runtime-policy/document-io.ts#L159
[m-model]: ../../../../maka/packages/runtime-host/src/server/execution-model-authority.ts#L1092
[m-local-store]: ../../../../maka/packages/storage/src/credential-store.ts#L124
[m-mcp-store]: ../../../../maka/packages/mcp/src/credential-oauth-storage.ts#L53
[m-managed]: ../../../../maka/packages/storage/src/managed-secret-store.ts#L30
[m-encrypted]: ../../../../maka/packages/storage/src/encrypted-file-managed-secret-store.ts#L121
[m-injector]: ../../../../maka/packages/storage/src/activation-secret-injector.ts#L125
[m-inject-test]: ../../../../maka/packages/storage/src/__tests__/activation-secret-injector.test.ts#L45
[m-managed-test]: ../../../../maka/packages/storage/src/__tests__/managed-secret-store.test.ts#L79
[p-auth]: ../../../packages/coding-agent/src/core/auth-storage.ts#L442
[p-auth-file]: ../../../packages/coding-agent/src/core/auth-storage.ts#L61
[p-resolve]: ../../../packages/coding-agent/src/core/resolve-config-value.ts#L145
[p-trust]: ../../../packages/coding-agent/src/core/project-trust.ts#L47
[p-gate]: ../../../packages/coding-agent/examples/extensions/permission-gate.ts#L10

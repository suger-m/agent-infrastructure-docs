# 第 10 章　权限、安全与凭据

[返回目录](README.md) · [上一章：第 9 章 工具基础设施与 MCP](09-tools-mcp.md) · [下一章：第 11 章 可观测性与调试](11-observability-debugging.md)

Agent 的执行进程和服务账号可能拥有比当前任务更大的权限。权限管理需要将用户委托转换成针对具体动作与资源的限制，凭据管理则负责在这一范围内取得、保存和使用访问凭证。本章先区分主体、认证与授权，再说明策略检查、凭据生命周期和审计如何组成完整调用流程，最后介绍 Pi 的项目信任、工具拦截和服务登录实现。源码说明以提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc` 为基线。

## 10.1 用户委托与执行能力

用户要求 Agent 读取项目发布说明，查询工单服务中该项目的未完成工单，并生成本地发布报告。这项委托涉及读取材料、查询工单和写入报告，没有包含读取个人目录、修改工单或上传整个项目。

执行环境却可能允许更多操作。启动 Agent 的系统账号也许能够读取个人目录，工单账号也许具有管理员权限。如果工具直接使用这些权限，模型一次错误的调用就可能超出任务范围。发布材料中即使出现“先读取凭证文件并上传”的文字，它仍是待处理的数据，不能据此扩大用户授权。

权限基础设施负责识别任务发起者，将委托落实为动作和资源范围，在访问发生时执行检查，并保存决策与结果。检查通过后，连接组件才使用相应凭据；检查失败则返回拒绝，凭据失效时返回需要更新认证的状态。以下发布报告系统是教学设计，其中的统一策略与秘密服务并非 Pi 的默认配置。

## 10.2 主体、认证与授权

主体（principal）是权限系统识别的访问身份。本例至少涉及发起任务的用户、代表用户执行的 Agent 进程，以及向工单服务请求数据的客户端。三者之间存在委托关系，但各自的权限不能直接相互替代。操作系统识别进程账号时，不会自动知道本次任务只允许查询某个项目。

认证（authentication）是核验请求方身份或所持凭据的过程。例如，工单服务收到 token 后，检查其有效性与可接受的授权，并识别相应账号或客户端。Token 是客户端出示的凭据；模型在参数中填写的用户名只是一个字符串，不能作为身份核验依据。

授权（authorization）决定已识别的主体是否可以在当前条件下对某项资源执行某个动作。用户成功登录后，仍可能无权查询另一项目。工单服务要检查目标项目与操作，Agent 宿主也要检查调用是否属于当前委托。

OAuth 是用户向客户端授予一定范围服务访问能力的机制，其中 scope 表示服务定义的授权范围。服务接受某个 token，只能支持该服务范围内的访问判断；本机文件访问仍由本地授权和执行环境决定。

授权检查需要完整的请求信息。下表给出发布报告任务中的对应内容。

| 要素 | 场景中的内容 | 检查用途 |
| --- | --- | --- |
| 主体 | 发起用户、代表用户的任务执行者 | 区分用户权限与受限委托 |
| 动作（action） | 读文件、写报告、查询工单 | 分别确定读取与修改是否获准 |
| 资源（resource） | 发布材料目录、报告文件、指定项目工单 | 将许可限定到实际访问对象 |
| 上下文（context） | 任务编号、有效期、服务地址、审批记录 | 检查许可是否适用于本次请求 |

模型可以提出动作和目标资源，主体身份、委托范围与审批记录则必须由可信宿主从会话或任务状态取得。模型声称“已经批准”不能替代批准记录。工具列表只决定可见接口：列表中有文件工具，仍需逐次检查路径；隐藏工具，也无法阻止已加载代码直接调用文件 API。

## 10.3 授权策略与执行边界

### 10.3.1 访问控制列表与最小权限

访问控制列表（ACL）用规则描述主体对资源可执行的动作。例如，为任务身份授予发布材料目录的读取权限和报告文件的写入权限，同时不授予其他文件权限。文件系统 ACL 通常依据系统用户或用户组判断；任务编号、有效期和服务范围等业务条件还需要应用策略处理。

本章教学系统采用默认拒绝，并约定拒绝规则优先。默认拒绝与逐次检查原则可参阅 [OWASP 授权指南](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)。不同产品的 ACL 匹配顺序可能不同，部署时需核对实际规则语义。本例策略如下。

| 主体及上下文 | 动作与资源 | 决定 |
| --- | --- | --- |
| 本次发布任务，委托仍有效 | 读取项目发布材料目录中的普通文件 | 允许 |
| 同一任务 | 写入约定的本地报告文件 | 允许 |
| 同一任务，使用工作账号 | 查询工单服务中指定项目的工单 | 允许 |
| 同一任务 | 修改或删除工单 | 拒绝 |
| 同一任务 | 读取个人目录、凭证文件，或向未批准的服务发送数据 | 拒绝 |
| 任意未匹配请求或已过期委托 | 任意动作 | 拒绝 |

最小权限要求许可覆盖完成任务所需的动作、资源和时间范围。查询足以生成报告时，无需授予删除工单的权限；任务结束后，临时写入许可也应结束。如果服务只能签发覆盖整个账号的 token，本地检查能够限制正常调用，但无法缩小泄漏 token 在服务端的权限。因此，还需要服务提供更小的 scope、独立账号或资源范围约束。

### 10.3.2 资源访问时的强制检查

宿主可以在调用前拒绝越界请求并解释原因。实际执行时，操作系统或沙箱继续限制文件打开、进程创建和网络连接；外部服务则检查 token、动作和目标资源。即使宿主已经允许调用，服务端仍需独立检查，因为客户端程序也可能出错。

以文件路径为例，目录下的符号链接可能指向个人凭证文件。字符串前缀符合规则，并不代表实际打开的文件符合规则。检查器需要处理路径规范化与链接，还要避免检查完成后、文件打开前路径被替换。只读挂载与受限文件系统视图可以进一步缩小可访问范围。授权约束应作用于实际资源访问，具体隔离方式见沙箱章节。

### 10.3.3 人工确认的范围

人工确认可以为少量、后果明确的操作提供授权。界面应显示动作、目标资源和有效范围，保存的决定也应绑定到本次调用。用户允许写入报告后，执行器若改变目标文件，就需要重新判断原许可是否仍适用。

重复的低风险操作通常更适合事先定义有限规则。对每次普通读取都询问，会增加交互成本；无人值守时又可能无法得到必要决定。在确实需要人工批准而无法取得批准时，系统应拒绝或进入等待状态。

## 10.4 授权调用与失败处理

发布报告任务需要分别保存委托与凭据。策略存储说明允许的操作，秘密存储保存访问凭证，执行器只在调用需要时取得对应值。能够找到一条凭据，不代表获准用它访问所有资源。

1. 宿主依据已识别的用户身份建立任务记录，保存文件范围、工单项目和有效期。待处理材料中的文字不能修改这份授权记录。
2. 模型请求读取发布说明。宿主解析路径并检查任务策略，记录允许决定；受限环境打开文件时，操作系统再核验实际文件权限，读取后仅返回任务所需内容。
3. 模型请求查询工单。宿主核对服务地址、动作与项目。允许后，连接组件向秘密存储请求工作账号凭据；秘密存储同时检查调用者的取用权限。
4. 连接组件核对 token 的适用服务与有效期，将它加入发往指定服务的请求。服务完成认证和授权后返回工单数据，模型无需取得 token 本身。
5. 执行器关联授权决定与调用结果。模型据此生成报告；写入时再次检查路径和动作，成功后向用户提供报告位置。
6. 任务结束时释放临时环境和授权，清理注入的秘密。需要复用的凭据可留在受保护存储中，但应从本次任务的临时文件和普通结果中移除。

每一处检查都可能终止调用。如果第 2 步请求读取个人目录，宿主应在访问前返回范围不足；相同限制也应适用于 Shell 等其他入口。如果本地策略允许查询，而服务拒绝目标项目，应保留服务端拒绝原因，由用户或管理员处理权限。重复发送原请求不会增加授权。

凭据过期则需要走认证更新流程。动作仍在委托范围内时，连接组件可以依据既有授权刷新 token；刷新凭据也被撤销时，应暂停依赖该服务的操作并要求重新登录。授权拒绝、凭据失效和网络失败需要不同的结果类型，执行器才能分别选择停止、登录或重试。

![通用教学流程：认证主体后按动作和资源授权；拒绝即停止，允许才取凭据并调用，决定与结果都可审计](figures/10-concept-authorization-flow.svg)

图中策略拒绝发生在秘密读取和资源调用之前。外部服务也可能在收到请求后拒绝失效凭据，宿主需要单独处理这类认证错误。

## 10.5 凭据生命周期

秘密（secret）包括 API key、access token、refresh token，以及某些客户端使用的 client secret。Bearer token 通常凭持有即可出示使用，泄漏者可能在其有效范围内发起请求。因此，管理范围需要覆盖取得、存储、注入、更新、撤销和清理。

### 10.5.1 取得与存储

凭据可以由用户配置，也可以通过 OAuth 授权取得。工单登录时，用户主动打开授权服务，在浏览器中选择账号并确认范围；客户端接收授权结果，换取用于 API 请求的 access token。服务若同时签发 refresh token，客户端可在 access token 到期时换取新值，无需每次重新登录。相关角色与流程见 [OAuth 2.0 授权框架](https://www.rfc-editor.org/rfc/rfc6749.html)，浏览器回调的校验在 Pi 参考实现中说明。

持久存储让后续请求或重启后的进程能够继续取得凭据。个人工具可使用权限受限的文件，共享服务可采用系统凭证库或秘密服务，并限制每个执行身份能够读取的记录。存储加密可以减少部分静态数据风险，但读取与解密权限仍需管理；同时拥有密文和密钥的执行者仍然能够取得原文。记录中还应保存账号、服务、到期时间和版本，避免混用工作与个人账号。

### 10.5.2 按需注入与输出控制

注入是将凭据交给真正发起请求的组件。例如，由宿主连接组件填写认证 header，模型只提交查询项目的参数。如果工具只能读取环境变量，应仅传入所需的 key，并限制派生子进程和输出环境的能力。完整复制宿主环境可能把其他服务的秘密一并传给工具；临时凭据文件也需要访问限制与删除时机。

秘密进入提示词后，可能同时进入模型请求与会话存档；带认证 header 的请求对象进入日志后，又形成新的凭据副本。工具错误还可能回显 URL、参数和环境。诊断记录应优先保留凭据标识、版本、服务地址与结果，排除原始值。脱敏过滤器不能可靠识别所有编码、分片或改写形式，所以减少采集和传递是更早的控制措施。

### 10.5.3 刷新、轮换与撤销

刷新在已有授权下取得新的短期 access token，不会自行增加 scope。轮换则替换凭据本身，例如刷新时签发新的 refresh token 并使旧值失效。两个进程若同时使用旧值刷新，后一次请求可能失败，也可能破坏客户端保存的新状态。协调范围因此应包含读取旧值、请求刷新和保存新值，不能只锁住最后的写文件操作。

撤销由授权方取消许可或凭据。用户停止任务时，宿主可以取消临时委托；key 泄漏后，还需在签发服务撤销或替换它。删除本地文件仅移除了一个副本，其他副本是否仍可访问服务取决于服务端验证与撤销机制。客户端删除动作不能保证旧 access token 立即失效。

### 10.5.4 结束与异常清理

正常结束、取消或异常退出后，组件需要删除任务临时文件、结束持有秘密的子进程，并释放不再使用的引用。崩溃留下的文件还应在恢复时处理。内存管理语言可能已经复制字符串，释放引用无法保证所有副本立即被擦除，所以短有效期和服务端撤销仍有作用。生命周期设计可参阅 [OWASP 秘密管理指南](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)。

![通用教学秘密生命周期：取得、存储、按需授权、最小注入、刷新轮换、撤销到期与清理](figures/10-concept-secret-lifecycle.svg)

图中本地清理与服务端撤销分别处理副本保存和访问资格；泄漏后的凭据还需由签发方使其失效。

## 10.6 授权决定与执行结果的审计

如果用户后来发现工单被修改，调查需要区分修改来自本次任务、其他程序，还是仅发生过未成功的尝试。模型回复与“工具完成”日志不足以说明操作为何获准，也不能单独证明实际结果。

审计（audit）保存可追查的决策与结果。本章教学系统为每次访问记录时间、任务或请求标识、发起用户、执行身份、动作、规范化资源、匹配规则、策略版本、允许或拒绝原因、审批来源，以及执行结果或服务请求标识。允许调用后仍可能因文件不存在而失败，因此授权决定与执行结果应分别保存。被拒绝的请求也需要记录，以便识别越界尝试。

查询工单的审计可以记录工作账号访问了指定项目、只读策略允许、采用凭据版本甲、服务返回成功，无需写入凭据原文或全部工单正文。文件名、身份和错误内容同样可能敏感，存储仍需要访问限制和保留期限。事件属性与应排除的数据见 [OWASP 日志指南](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)。

审计支持事后调查、凭据撤销和策略修正，但无法撤销已经发生的越权。执行器若能任意删除自己的记录，证据就可能不完整；要求较高的系统可以将记录发送到独立受保护存储，并监控写入失败。必须审计的操作可在记录失败时拒绝执行；允许本地缓冲则提高可用性，同时承担缓冲丢失风险。这一取舍需要由部署策略明确规定。

## 10.7 参考实现：Pi 的信任与权限检查

Pi 运行任务时会向模型服务发送请求，也可能执行模型提出的本机工具调用。前者要有服务接受的凭证；后者由工具检查和实际执行环境处理。`/login` 成功取得模型 token 后，模型服务可以接受请求；模型随后提出读文件时，文件能否访问仍取决于执行进程的权限。本章教学设计中的任务委托、统一 ACL 检查、授权取密与持久审计，需要由宿主应用或部署环境实现。

以下流程以本机 `bash` 调用为例，假设已经配置模型 provider 与执行后端。Provider 指接收模型请求的服务提供方。扩展的 `tool_call` 检查只有在相应钩子已安装时才发生；Pi 默认不会对每条模型命令询问用户。

1. Pi 启动时决定是否加载当前项目提供的设置、资源和扩展。
2. 请求模型前，Pi 从 provider 凭证存储取出凭证，并用它向模型服务请求回复。
3. 模型返回工具调用后，AgentSession 校验调用参数；如果已安装的扩展监听 `tool_call`，Pi 将调用交给它检查。扩展可以放行或阻止。
4. 放行的 `bash` 调用交给已配置的本机或其他执行后端。实际能访问哪些资源，由运行该操作的进程权限和后端隔离配置决定。
5. 执行结果回到 AgentSession，再进入后续模型交互。

这条路径中，项目加载、provider 登录、扩展拦截、操作系统权限和秘密保存分别处理不同对象。图中箭头表示相应的请求或数据流；MCP 服务认证是另一条服务连接路径，后文单独说明。
provider 拒绝请求、扩展阻止工具调用、操作系统拒绝资源访问，分别发生在不同阶段。provider token 和 OAuth scope 都不会为 bash 建立本机文件白名单（[packages/coding-agent/src/core/tools/bash.ts:186-212](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/tools/bash.ts#L186-L212)、[packages/coding-agent/src/core/tools/bash.ts:361-369](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/tools/bash.ts#L361-L369)）。

![项目可信、工具拦截、OS 访问与 MCP 服务 token 的权限路径](figures/10-permission-layers.svg)

### 10.7.1 加载项目代码的信任决定

第一步发生在模型调用工具之前。项目扩展是在 Pi 进程内运行的代码，因此决定加载它，就可能让它以宿主进程权限执行代码。`resolveProjectTrusted()` 先检查显式 override，再处理扩展的 `project_trust` 事件、记住的决定和默认策略。某种运行方式需要询问而当前没有 UI 时，尚未决定的项目返回 false。信任后，Pi 才加载 `.pi` 设置和资源、安装缺少的项目包并运行项目扩展；提示文案询问的正是这些动作（[packages/coding-agent/src/core/project-trust.ts:25-39](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/project-trust.ts#L25-L39)、[packages/coding-agent/src/core/project-trust.ts:47-97](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/project-trust.ts#L47-L97)）。

项目信任决定控制项目内容的加载。扩展一旦加载，其代码仍具有宿主进程权限；阻止某次工具调用不会降低扩展代码本身的权限。

MCP 配置也会检查 `projectTrusted`。项目配置不能设置 `auth.provider`，这样项目文件不能指定一个 URL，让 Pi 把用户的 provider token 发给它。全局配置可以设置该选项，但目标必须是 HTTPS 或 loopback。手写 headers 使用另一条配置路径（[packages/coding-agent/src/extensions/mcp/config.ts:133-155](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/config.ts#L133-L155)、[packages/coding-agent/src/core/mcp-servers.ts:272-290](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/mcp-servers.ts#L272-L290)）。

### 10.7.2 模型凭据的保存与读取

项目内容获准加载后，请求模型仍需 provider 接受的 key 或 token。`/login` 可取得 OAuth 凭证；API key 也可以直接配置。Pi 把 provider 凭证的读写交给实现 `CredentialStore` 接口的 `AuthStorage`，认证流程由 `ModelRuntime` 和 pi-ai Models 编排。存储可以用文件后端，也可以用内存后端；内存后端适合练习，不会触碰用户凭证文件（[packages/coding-agent/src/core/auth-storage.ts:1-47](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L1-L47)、[packages/coding-agent/src/core/auth-storage.ts:292-360](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L292-L360)）。

默认文件后端在 agent 目录的 `auth.json` 中保存普通 JSON。登录先于后续模型请求发生，持久保存凭据使 Pi 能在每次请求时按 provider 取出对应值，并向服务证明身份。新建目录请求 `0700`，新建文件请求 `0600`；若文件已经存在，后端保留其原有模式和 ACL。Windows 上实际访问控制由 ACL 决定，不能把 POSIX mode 数字当作 Windows 权限结果。文件内容没有加密，也没有交给操作系统凭证库（[packages/coding-agent/src/core/auth-storage.ts:24-67](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L24-L67)、[packages/coding-agent/src/core/auth-storage.ts:96-114](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L96-L114)）。

读取时，`list()` 只列 provider ID 和凭证类型，不解析配置值；`read()` 读取 API key 时会解析字面值、环境变量模板或 `!command`。`!command` 在宿主 Shell 执行。一般解析会缓存结果，MCP 使用的 `resolveConfigValueOrThrow()` 不缓存。因为命令是在配置读取路径中运行，扩展的 `tool_call` 钩子不会拦截它（[packages/coding-agent/src/core/auth-storage.ts:441-447](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L441-L447)、[packages/coding-agent/src/core/auth-storage.ts:484-489](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L484-L489)、[packages/coding-agent/src/core/resolve-config-value.ts:138-216](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/resolve-config-value.ts#L138-L216)、[packages/coding-agent/src/core/resolve-config-value.ts:221-251](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/resolve-config-value.ts#L221-L251)）。

API key 可能长期有效，OAuth access token 通常需要通过 refresh token 更新。多个 Pi 进程同时登录或刷新时，各进程内存中的文件副本可能已经过时；直接写回整份旧 JSON 会覆盖其他进程的修改。

### 10.7.3 存储锁与并发更新

假设进程 A 读到模型服务 X 的旧凭证，进程 B 更新了服务 Y。若 A 随后把早先读到的整份文件写回，B 对 Y 的更新就会消失。文件权限解决谁能读写的问题；文件锁解决更新先后顺序的问题。

`modify(provider, fn)` 在锁内重读当前文件，只合并目标 provider 的新值，其他 provider 的记录保留。异步后端会在回调结束和写入前检查状态；操作被取消、取锁失败或锁损坏时，不提交新内容（[packages/coding-agent/src/core/auth-storage.ts:116-200](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L116-L200)、[packages/coding-agent/src/core/auth-storage.ts:449-481](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L449-L481)）。

### 10.7.4 工具拦截与实际执行权限

模型返回调用后，Pi 先校验参数，再让已安装扩展有机会检查这一次调用。`AgentSession` 将参数交给扩展的 `tool_call`，并把扩展返回的阻止决定交回 agent 工具管线。嵌套调用也走这条路径，并带有 `parentToolCallId`，所以 Codemode 内部的 MCP 调用也能被拦截钩子看到（[packages/coding-agent/src/core/agent-session.ts:652-681](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L652-L681)、[packages/coding-agent/src/core/agent-session.ts:726-764](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L726-L764)）。

仓库中的 `permission-gate.ts` 是这种扩展的示例。它只检查名为 `bash` 的工具，用正则匹配少数命令模式；有 UI 时询问用户，非交互运行遇到这些模式则直接 block。这个例子没有完整解析 Shell 语法，也没为 `read/write` 或所有 MCP 工具设置通用审批规则（[packages/coding-agent/examples/extensions/permission-gate.ts:10-34](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/permission-gate.ts#L10-L34)）。是否询问取决于安装的具体 gate 及运行上下文，默认没有全局审批流程。

若调用通过检查，工具才交给执行实现。bash 可以运行在已配置的本机、沙箱或远端环境中；ACL（访问控制列表）规定身份对资源的访问权限，沙箱可以限制进程可达的资源。工具钩子阻止的是一次工具执行，操作系统权限约束实际进程对资源的访问，两者位置不同（[packages/coding-agent/src/core/tools/bash.ts:186-212](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/tools/bash.ts#L186-L212)、[packages/coding-agent/src/core/tools/bash.ts:361-369](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/tools/bash.ts#L361-L369)）。Pi 默认继承启动进程的权限，没有内置文件、进程、网络或凭证访问限制系统；要限制进程能访问的资源，需要实际启用容器、VM 或 OS 沙箱后端（[README.md:88-96](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/README.md#L88-L96)、[packages/coding-agent/src/core/tools/bash.ts:96-118](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/tools/bash.ts#L96-L118)）。

## 10.8 参考实现：外部服务登录的完整链路

上面取出的 provider token 用于请求模型服务。若模型提出调用远端 MCP 工具，Pi 还要连接一个独立的工具服务；模型 provider 的凭证不能自动证明 Pi 有权访问它。MCP 的 OAuth token、scope 和存储对应这个服务身份与服务端授予的能力。scope 不会批准本机 bash 命令。

HTTP MCP 服务未手写 Authorization header、也未配置 `auth.provider` 时，Pi 使用独立的 MCP OAuth 流程。连接会先尝试已有 token；需要登录时进入 needs-auth 状态，等待用户发起浏览器流程，Pi 不会自行打开浏览器。若配置了 `auth.provider`，每次请求改为从 provider 取当前 token，MCP OAuth store 不另存副本（[packages/coding-agent/src/extensions/mcp/runtime.ts:82-87](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L82-L87)、[packages/coding-agent/src/extensions/mcp/runtime.ts:204-218](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L204-L218)、[packages/coding-agent/src/extensions/mcp/oauth.ts:1-11](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/oauth.ts#L1-L11)）。

用户发起登录后，Pi 先发现受保护资源的 metadata（描述资源及授权端点的信息）和授权服务器 metadata，再选择已注册的 OAuth client，或动态注册 client；配置启用时也可使用 Client ID Metadata Document。接着 Pi 打开浏览器授权地址，用户在服务端登录并同意。授权地址带随机 `state` 和 S256 PKCE challenge，Pi 预先保存 verifier。授权服务器把浏览器送回 loopback callback；Pi 校验待处理的 state 和回调路径，再依 metadata 核对 `iss`，最后用授权码和 verifier 换取 token。Pi 保存 token 与到期时间，后续对该 MCP 服务的 HTTP 请求才取出 token 并发送（[packages/mcp/src/oauth/discovery.ts:119-145](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/discovery.ts#L119-L145)、[packages/mcp/src/oauth/discovery.ts:202-215](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/discovery.ts#L202-L215)、[packages/mcp/src/oauth/flow.ts:163-200](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/flow.ts#L163-L200)、[packages/mcp/src/oauth/flow.ts:354-399](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/flow.ts#L354-L399)、[packages/mcp/src/oauth/callback.ts:124-162](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/callback.ts#L124-L162)）。

登录校验分别约束回调归属、换码客户端与授权来源：`state` 确认回调属于这次登录；PKCE 用随机 verifier 和摘要 challenge 将授权码绑定到发起客户端，换码时客户端须证明持有 verifier；`issuer` 标识授权服务器；`scope` 说明服务端授予的范围。默认 discovery 会检查 issuer，并核对 resource 的 origin/path。若配置显式指定授权服务器 metadata URL，源码信任该配置，不再按默认 discovery 的方式核对 issuer；此 URL 必须使用 HTTPS，loopback 才允许 HTTP。两种 metadata 来源的信任依据不同（[packages/mcp/src/oauth/discovery.ts:169-183](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/discovery.ts#L169-L183)、[packages/coding-agent/src/core/mcp-servers.ts:201-208](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/mcp-servers.ts#L201-L208)）。

登录结果默认写入 `mcp-auth.json`，键由规范化 namespace 和 URL 组成。同一 URL 可按 `work`、`personal` namespace 保存两份账号配置。底层 provider 会忽略 `serverUrl` 不匹配的状态，因此服务 URL 改变时，旧 token 不会自动发给新地址（[packages/coding-agent/src/extensions/mcp/oauth.ts:126-169](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/oauth.ts#L126-L169)、[packages/mcp/src/oauth/provider.ts:152-167](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/provider.ts#L152-L167)）。

服务 token 接近到期时会提前刷新。若服务采用 refresh rotation（刷新时返回新 refresh token 并使旧 token 失效），两个进程同时消费同一旧 token 可能导致新授权丢失。Pi 在进程内共享 `refreshing` Promise，进程之间按服务使用 refresh lock；取得锁后重读 token，如果另一个进程已替换被拒绝的旧 token，就使用新值。每个 OAuth 网络请求最多等待 15 秒；关闭连接时也会等待刷新结果存盘。若服务返回 `insufficient_scope`，刷新无法补足授权范围，用户需要重新登录，并合并已有和新增 scope（[packages/coding-agent/src/extensions/mcp/oauth.ts:45-56](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/oauth.ts#L45-L56)、[packages/coding-agent/src/extensions/mcp/oauth.ts:171-196](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/oauth.ts#L171-L196)、[packages/coding-agent/src/extensions/mcp/oauth.ts:300-374](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/oauth.ts#L300-L374)、[packages/coding-agent/src/extensions/mcp/oauth.ts:521-550](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/oauth.ts#L521-L550)）。

提前刷新失败后，连接仍可能携带旧 token 发起请求，随后根据服务的 401 响应继续处理。15 秒超时针对单个 OAuth 网络请求，整次浏览器登录没有因此获得同样的总时限。默认文件存储使用进程间刷新锁，自定义后端若没有配置锁目录则没有这层进程间协调；锁失效仍可能让刷新重叠，不能把它理解成服务端提供的事务保证。

### 10.8.1 Radius provider 与实验性 relay

Radius 是另一种模型 provider 登录实现，用来对照 provider 凭证如何接入特定网关；它与 MCP 登录属于两条独立的服务连接流程。`createRadiusOAuth()` 提供 browser 和 device-code 两种方式。browser 从 gateway 的 `/v1/oauth` 发现授权页面，使用 PKCE、state 和 `127.0.0.1:1456/oauth/callback` 回调。device-code 与 refresh 分别直接访问 gateway 的 `/v1/oauth/device` 和 `/v1/oauth/token`。生成 provider auth 时读取 `credential.access`；这让 provider 识别连接方，不设置本机工具权限（[packages/ai/src/auth/oauth/radius.ts:18-27](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/auth/oauth/radius.ts#L18-L27)、[packages/ai/src/auth/oauth/radius.ts:134-187](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/auth/oauth/radius.ts#L134-L187)、[packages/ai/src/auth/oauth/radius.ts:189-268](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/auth/oauth/radius.ts#L189-L268)、[packages/ai/src/auth/oauth/radius.ts:275-321](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/auth/oauth/radius.ts#L275-L321)）。

实验性 relay 使用 `RadiusRelayAuthResolver`。每次尝试连接时，它重新解析显式 token 或文件，也可以通过 `ModelRuntime` 取得已存储的 Radius auth，并要求 OAuth 至少还有五分钟有效期。设置 `PI_OFFLINE` 时不会建立 relay 认证连接。这个 resolver 只提供连接认证，没有多租户 RBAC 或统一秘密代理，也不使用 MCP OAuth store（[packages/coding-agent/src/experimental/radius-auth.ts:16-67](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/experimental/radius-auth.ts#L16-L67)）。

## 10.9 参考实现：默认能力与部署责任

Pi 的项目信任、服务认证、扩展钩子和执行环境分别检查不同对象。默认实现没有统一策略将它们组织成本章的按任务授权系统。

要让发布任务只能读指定材料并写指定报告，需要为部署配置实际的文件权限或沙箱，再为任务动作编写策略。实际文件访问取决于操作系统 ACL，项目信任只决定项目资源是否加载。Pi 默认没有统一文件或网络 ACL、全局审批系统、OS 凭证库、每租户秘密权限，也不保证自动脱敏所有工具结果。若使用前文的秘密服务设计，还需要接入相应存储和取用权限。

要追查“谁允许查询这个工单项目”，需要把授权上下文与决定保存到审计记录。Session/tool 事件可用于诊断，但默认流程不要求把每次授权决定写入独立审计账本，因此事件流和会话记录不能自动作为授权审计（[packages/coding-agent/src/core/agent-session.ts:652-681](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L652-L681)、[packages/coding-agent/src/core/agent-session.ts:726-764](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L726-L764)）。

部署还需要检查秘密在进程之间的传递。文件锁负责协调更新，PKCE、state 与 issuer 校验负责保护 OAuth 交互；具有宿主进程权限的扩展仍可直接访问它有权读取的文件。MCP stdio 子进程默认继承环境，Gondolin 示例也传入宿主 bash 环境（[packages/mcp/src/transports/stdio.ts:90-102](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/stdio.ts#L90-L102)、[packages/coding-agent/examples/extensions/gondolin/index.ts:315-349](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/gondolin/index.ts#L315-L349)）。凭证留在宿主时，环境变量中的秘密可能进入工具环境。宿主 bash 使用 `getShellEnv()` 返回的 `process.env` 副本，见 [packages/coding-agent/src/utils/shell.ts:138-150](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/utils/shell.ts#L138-L150)。

| 机制 | 决策对象 | 能力边界 |
| --- | --- | --- |
| 项目信任 | 是否加载项目设置、资源与扩展 | 不限制已运行工具的系统调用 |
| 工具拦截 | 扩展可返回 `block`，阻止工具 `execute` | 需要安装或编写具体策略 |
| `AuthStorage` | provider 凭证读写、锁与刷新时的修改接口 | 默认是 JSON 文件存储，不是秘密隔离服务 |
| MCP OAuth | 获取、保存、刷新 service token，处理 scope 增加 | scope 是服务端授予的能力，不是本机命令审批 |
| 容器/VM/OS 沙箱 | 限制执行环境可访问的资源 | 要由部署或可选后端实际启用 |

## 10.10 源码定位与本章术语

| 机制 | 源码入口 |
| --- | --- |
| 项目加载信任 | [信任判定](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/project-trust.ts#L47-L97)、[项目 MCP 配置限制](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/config.ts#L133-L155) |
| 工具调用拦截 | [调用钩子](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L652-L681)、[审批扩展示例](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/permission-gate.ts#L10-L34) |
| 凭据存储 | [默认文件存储](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L24-L67)、[锁与更新](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/auth-storage.ts#L116-L200) |
| OAuth 登录校验 | [授权码交换](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/flow.ts#L354-L399)、[回调检查](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/oauth/callback.ts#L124-L162) |
| MCP 凭据刷新 | [刷新协调](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/oauth.ts#L300-L374)、[授权范围变化](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/oauth.ts#L521-L550) |
| 模型网关与实验 relay | [Radius 登录](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/auth/oauth/radius.ts#L134-L187)、[实验 relay 认证](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/experimental/radius-auth.ts#L16-L67) |

源码可按调用阶段阅读：`project-trust.ts` 处理项目加载决定，`agent-session.ts` 提供工具拦截入口，`auth-storage.ts` 实现 provider 凭据存储，MCP 的 `oauth.ts` 与 `packages/mcp/src/oauth/` 组织服务授权和刷新。对应证据链接位于各参考实现段落；下面的测试入口提供离线核对材料。

认证（authentication）是向目标服务证明客户端持有可接受的凭证。授权（authorization）是决定动作是否获准；OAuth scope、工具拦截和 OS 文件权限分别控制服务端能力、工具调用和系统资源访问。秘密（secret）是需要保存或传递的 key、access token、refresh token 或 client secret。

| 术语 | 在 Pi 中的含义 |
| --- | --- |
| 项目信任（project trust） | 决定能否加载 `.pi` 设置和资源、安装项目包、执行项目扩展 |
| `tool_call` 钩子 | 工具执行前交给扩展的拦截点，扩展可返回 `block` 阻止执行 |
| PKCE | 把授权码与 verifier 绑定；换取 token 时必须证明持有 verifier |
| Refresh rotation（刷新轮换） | 服务刷新时返回新 refresh token，并使旧 token 失效 |
| Scope | 服务端授予的能力范围；范围不足时需重新登录并合并 scope，不批准本机命令 |

## 10.11 练习：假凭据的存储与刷新验证

存储测试可检查并发更新是否互相覆盖。[packages/coding-agent/test/auth-storage.test.ts:142-193](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/test/auth-storage.test.ts#L142-L193) 检查新文件模式、保留已有模式、外部编辑和并发修改；[packages/coding-agent/test/auth-storage.test.ts:280-371](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/test/auth-storage.test.ts#L280-L371) 检查取消取锁，以及取消正在执行的 callback 后不提交内容。

MCP 的双账号和刷新过程分别见 [packages/coding-agent/test/mcp-oauth-store.test.ts:15-44](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/test/mcp-oauth-store.test.ts#L15-L44) 与 [packages/coding-agent/test/mcp-oauth-refresh.test.ts:24-91](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/test/mcp-oauth-refresh.test.ts#L24-L91)。前者检查同 URL 的账号分离和旧状态迁移，后者用本地假 OAuth server 检查共享刷新与关闭前保存。Radius 的 [packages/ai/test/radius-oauth.test.ts:37-117](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/test/radius-oauth.test.ts#L37-L117) 通过 fetch mock 检查 device-code 和 refresh 路线。

先用 `AuthStorage.inMemory()` 存两个假 provider，修改其中一个，再检查另一个是否仍存在，并比较 `list()` 与 `read()` 返回的字段。假 key 可用 `example-only`，练习不会从环境或用户目录取值。

再用内存 MCP store，为同一个 `https://mcp.example.com/mcp` 保存 work、personal 两份假状态，然后删除其中一份。刷新测试中的本地假 server 还演示了三个 provider 同时收到 401、最终只刷新一次的情况；使用本地假服务无需访问真实服务端点。

已有仓库依赖时，可以在 `packages/coding-agent` 单独运行下面的测试。

```powershell
node ../../node_modules/vitest/dist/cli.js --run test/auth-storage.test.ts test/mcp-oauth-store.test.ts test/mcp-oauth-refresh.test.ts
```

工具实际运行的位置和可访问资源，可结合 [第 8 章 沙箱与执行环境](08-sandbox-environment.md) 核对。[第 11 章 可观测性与调试](11-observability-debugging.md) 接着讨论如何记录和诊断调用，以及如何避免把凭证值写进普通日志。

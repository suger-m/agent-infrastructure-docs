# 09. 一个 MCP 工具怎样进入 Agent 会话

[返回索引](README.md) · [上一章：执行环境](08-sandbox-environment.md) · [下一章：权限与秘密](10-permissions-secrets.md)

本章按提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc` 阅读源码。

## 外部工具怎样变成 Agent 能用的操作

工具接入要让模型知道能调用什么，让程序知道调用要送到哪里，再把结果接回对话。内置 `read` 可以直接调用程序中的函数；外部搜索或数据库服务则需要通过进程通信或网络发送请求。如果每个服务都使用不同的发现和调用格式，接入层就得逐个适配。

MCP 为这段客户端与服务通信规定共同的消息格式。客户端是 Pi 中发起连接和请求的一方，服务是提供工具定义并执行操作的一方。先建立通道、交换协议和能力信息，再发现工具；发现的定义转换成 Agent 工具后，模型才能通过会话管线使用。

本章的完整过程是“读取配置 → 连接服务 → 初始化 → 取得工具定义 → 注册到会话 → 模型提出调用 → 发送工具请求 → 转换并返回结果”。配置回答连谁，握手回答对方是否支持这套协议，发现回答有哪些工具，注册回答模型怎么使用它，执行与结果转换完成一次真实交互。下文按这些交接拆开实现。

MCP（Model Context Protocol，模型上下文协议）规定客户端如何与工具服务交换初始化、发现和调用消息。它不决定工具运行在哪台机器，也不替服务或操作系统实施访问控制。

读 `echo` 服务时，先区分参与者：

| 对象 | 负责什么 | 与其他对象的关系 |
| --- | --- | --- |
| Agent 与模型 | Agent 会话管理工具并把结果交还模型；模型根据工具声明提出调用 | AgentSession 将模型请求接入会话工具管线 |
| 工具实现 | 接收参数并返回工具结果 | 内置工具和 MCP 适配工具都进入会话工具管线 |
| MCP 客户端 | 握手、发现服务工具、发送调用并配对响应 | 由 coding-agent 集成接入 Agent 会话 |
| 传输（transport） | 在客户端与服务之间搬运消息 | 可启动 stdio 子进程，也可连接 Streamable HTTP（可流式传输的 HTTP 通道） |
| MCP 服务 | 按协议提供工具定义并执行工具 | 对自己的外部副作用和服务端权限负责 |

MCP 提供一套消息约定：客户端先初始化连接，再请求工具清单，最后按工具名与参数发起调用。工具参数的 JSON Schema（机器可读的参数格式说明）会交给模型或工具适配层。传输只选择消息通道，工具呈现配置只选择模型看到工具的方式，隔离仍由具体执行后端和服务部署决定。

以本地 `echo` 服务为例，完整过程如下：

1. Pi 读取 MCP 服务配置并启动对应 stdio 进程，或连接已配置的 HTTP 地址。
2. `McpClient` 通过传输发送 `initialize`，读取协议版本、能力和服务信息，再发送 `notifications/initialized`。
3. 客户端调用 `tools/list` 取得 `echo` 的定义及参数 schema，并处理可能存在的后续分页。
4. coding-agent 将服务工具转换为 `mcp__server__tool` 名称，按呈现方式注册进 Agent 会话。
5. 模型提出调用后，调用经过 AgentSession 工具管线和扩展钩子（hooks，在调用前运行的扩展代码），再由 MCP 客户端发出 `tools/call`。
6. 服务执行并返回 `CallToolResult`；客户端按请求 ID 配对结果，适配层交还工具结果，Agent 再进入下一轮模型请求。

握手完成前就把工具交给模型，可能出现服务尚未准备好、调用却已到达的竞态，因此客户端单独管理连接状态。下面两图分别概括服务消息路径和 Pi 会话中的注册/执行路径。箭头表示调用流向，不意味着传输或 MCP 自动提供操作系统隔离。

![MCP 服务从读取配置、建立传输、握手发现，到工具调用和结果返回](figures/09-mcp-tool-path.svg)

![MCP 服务连接、工具定义注册与直接调用或 Codemode 调用的完整管线](figures/09-mcp-integration.svg)

## 请求怎样找到自己的响应

一次连接可以同时等待多个请求。服务返回结果时，客户端必须知道它在回答哪一次调用，所以每个请求带一个 ID；客户端保存 ID 与等待者的对应表，收到响应再找到原调用。notification（通知）不等待对应响应，用于告诉另一方有变化或进度。这就是后文 JSON-RPC 请求、响应与通知的区别。

调用方通常通过 Promise 等待结果。Promise 是 JavaScript 中代表“稍后完成的值”的对象；pending Map 保存尚未完成的请求。请求完成、超时或取消后要移除等待记录，避免旧请求一直占用内存，或者迟到响应错误地结束另一次等待。

客户端首先要把消息送到服务端。transport 指消息传输通道，底层可以是子进程，也可以是 HTTP。`McpTransport` 用 `start/send/close` 和事件接口把两种通道统一起来，`McpClient.connect()` 在此基础上完成握手与版本校验。

消息采用 JSON-RPC（请求响应消息格式）：请求带 `id/method/params`，响应按 `id` 找到等待结果的调用，通知则没有请求 ID。`requestInternal()` 为请求递增编号，在 pending Map 中保存等待者，并安装超时与取消处理。上层的 `listTools()` 和 `callTool()` 再选择具体协议方法，检查返回结果的基本形状（[packages/mcp/src/transports/transport.ts:9-17](../../packages/mcp/src/transports/transport.ts#L9-L17)、[packages/mcp/src/client.ts:202-248](../../packages/mcp/src/client.ts#L202-L248)、[packages/mcp/src/client.ts:394-439](../../packages/mcp/src/client.ts#L394-L439)、[packages/mcp/src/client.ts:376-384](../../packages/mcp/src/client.ts#L376-L384)）。

一个工具列表可能分多页返回。`tools/list` 会继续读取 cursor 指向的页面，遇到重复 cursor 就拒绝，最多读取 1000 页。调用期间收到 progress 会重新计时；超时或取消通常会通知服务端，`initialize` 按协议不发送 cancelled 通知。客户端这样可以结束等待，但服务端先前发生的外部写入并不保证被撤销（[packages/mcp/src/client.ts:354-374](../../packages/mcp/src/client.ts#L354-L374)、[packages/mcp/src/client.ts:531-577](../../packages/mcp/src/client.ts#L531-L577)）。

## 从子进程换到 HTTP

stdio 通道把服务作为本地子进程启动，双方通过输入输出流传递消息；HTTP 通道连接已部署的服务地址，通过网络请求传递消息。两者可以使用相同的工具调用格式，但生命周期不同：前者要处理进程退出和环境继承，后者要处理请求状态、会话标识、网络断流和服务认证。

SSE 是服务端通过 HTTP 持续发送事件的格式，客户端可以逐个读取事件。带事件 ID 的流在断开后有机会按 ID 续接，这是继续接收结果；是否再次执行原操作，则取决于另外的重试规则。先分清“重收结果”和“重做操作”，才容易理解下面的 GET 续接与 POST 请求。

本地 echo 服务走 stdio：传输层启动配置中的命令，从 stdout 按行读 JSON-RPC 消息，stderr 用来保存诊断信息。子进程默认继承 `process.env`。底层传输实现虽然支持 `inheritEnv: false`，coding-agent 的默认工厂没有设置它。关闭时先结束 stdin，让服务自行退出；仍不退出时再终止或强杀，并清理子进程树（[packages/mcp/src/transports/stdio.ts:90-141](../../packages/mcp/src/transports/stdio.ts#L90-L141)、[packages/mcp/src/transports/stdio.ts:143-207](../../packages/mcp/src/transports/stdio.ts#L143-L207)、[packages/coding-agent/src/extensions/mcp/runtime.ts:98-122](../../packages/coding-agent/src/extensions/mcp/runtime.ts#L98-L122)）。

远端 HTTP 服务通过 Streamable HTTP 接收 POST 请求，返回 JSON 或 SSE。握手完成后，客户端才打开服务端到客户端的 GET 流，请求头会带上会话 ID 和协议版本。带 event ID 的响应流断开后，可以用 `Last-Event-ID` 从 GET 续接，原来的工具 POST 不会因此重发。响应流没有 event ID 又提前结束时，对应请求会失败（[packages/mcp/src/transports/streamable-http.ts:225-256](../../packages/mcp/src/transports/streamable-http.ts#L225-L256)、[packages/mcp/src/transports/streamable-http.ts:309-320](../../packages/mcp/src/transports/streamable-http.ts#L309-L320)、[packages/mcp/src/transports/streamable-http.ts:361-404](../../packages/mcp/src/transports/streamable-http.ts#L361-L404)）。

HTTP 返回 401，或用 403 要求更大的授权范围（scope）时，客户端会交给认证提供方（auth provider）处理一次，再用新凭证重试请求。这里处理的是认证失败，网络重连处理的则是流中断，两者触发条件不同。登录与刷新过程在 [权限与秘密](10-permissions-secrets.md) 中展开（[packages/mcp/src/transports/streamable-http.ts:159-169](../../packages/mcp/src/transports/streamable-http.ts#L159-L169)、[packages/mcp/src/transports/streamable-http.ts:277-307](../../packages/mcp/src/transports/streamable-http.ts#L277-L307)）。

这份源码的协议常量是 `2025-11-25`，也接受列表中的三个旧版本。Streamable HTTP 会在响应或 GET 流里使用 SSE 消息格式，配置同时明确拒绝旧版独立的 legacy SSE transport。两者需要按不同协议路径理解（[packages/mcp/src/protocol/types.ts:4-10](../../packages/mcp/src/protocol/types.ts#L4-L10)、[packages/coding-agent/src/core/mcp-servers.ts:248-275](../../packages/coding-agent/src/core/mcp-servers.ts#L248-L275)）。

## 在会话中注册与更新工具

服务列出的工具还不能直接等同于模型已经看到的工具。Pi 要把名称转换到自己的命名规则，保留参数定义，并决定呈现方式。JSON Schema 描述参数允许有哪些字段、字段是什么类型；它帮助模型构造调用，也给参数校验提供依据。服务后来修改清单时，注册状态也要更新，否则模型可能继续请求已消失的工具。

客户端能调用服务以后，还要把服务接入会话。这一节回答工具定义从哪里来、何时刷新。

coding-agent 先读取全局 `mcp.json`，项目可信时再读取 `.pi/mcp.json`。项目里的同名条目可以替换服务，也可以只覆盖 `enabled`、`exposure`（工具呈现方式）和 `toolExposure`；文件配置与扩展注册的服务同名时，文件配置优先（[packages/coding-agent/src/extensions/mcp/config.ts:91-155](../../packages/coding-agent/src/extensions/mcp/config.ts#L91-L155)、[packages/coding-agent/src/extensions/mcp/index.ts:345-363](../../packages/coding-agent/src/extensions/mcp/index.ts#L345-L363)）。

服务连接在 `session_start` 时转到后台启动。第一次 prompt 只有限等待 direct 服务，Codemode、tool_search 和资源工具等到真正调用时，再等各自需要的服务准备好。连接之后，服务还可以通过 `tools/list_changed` 通知客户端刷新定义；被删除或禁用的工具会重新注册成 hidden，避免继续调用旧定义（[packages/coding-agent/src/extensions/mcp/index.ts:1111-1206](../../packages/coding-agent/src/extensions/mcp/index.ts#L1111-L1206)、[packages/coding-agent/src/extensions/mcp/runtime.ts:378-435](../../packages/coding-agent/src/extensions/mcp/runtime.ts#L378-L435)、[packages/coding-agent/src/extensions/mcp/index.ts:446-463](../../packages/coding-agent/src/extensions/mcp/index.ts#L446-L463)）。

注册到 Pi 时，服务中的 `search` 会得到 `mcp__docs__search` 这样的名称。转换过程替换不允许的字符，名称超过 64 字符或发生碰撞时再附加 hash。适配后的定义保留输入 schema、输出 schema 和布尔 annotations；实际发送给服务的仍是它原来的工具名（[packages/coding-agent/src/extensions/mcp/tools.ts:78-93](../../packages/coding-agent/src/extensions/mcp/tools.ts#L78-L93)、[packages/coding-agent/src/extensions/mcp/tools.ts:244-295](../../packages/coding-agent/src/extensions/mcp/tools.ts#L244-L295)）。

## 500 个工具怎样交给模型

呈现方式配置会改变模型看到的工具集合和声明成本。

如果服务有 500 个工具，每次都把全部参数定义（schema）发给模型，会占用大量上下文。Pi 可以先告诉模型有哪些命名空间（namespace），让它在调用时再发现具体工具。呈现方式（exposure）配置决定工具怎样呈现、通过哪条路径调用：

| 配置值 | 模型怎样使用 |
| --- | --- |
| `codemode`（默认） | 脚本通过 `searchTools/describeTool` 发现并调用 |
| `deferred` | `tool_search` 加载声明后直接调用；不要求 Codemode |
| `direct` | 像内置工具一样直接声明，也可被脚本调用 |
| `hidden` | 在这套工具管线中不可达 |

这些选择会改变可达的工具集合和声明成本，外部服务自身的 OS 权限并不随之改变。annotations 也只是服务声明的提示，无法证明操作已受到执行隔离（[packages/coding-agent/src/core/mcp-servers.ts:43-77](../../packages/coding-agent/src/core/mcp-servers.ts#L43-L77)、[packages/coding-agent/src/core/mcp-servers.ts:233-242](../../packages/coding-agent/src/core/mcp-servers.ts#L233-L242)）。

服务还可以提供资源（resource），用 URI 标识可读内容，与执行工具有所区别。Pi 将资源操作适配成 list、list templates、read 三个工具，它们采用所达服务中最宽的呈现方式。由于 Pi 不渲染 MCP App UI，这类资源会被过滤。客户端宣告的 `roots` 也只是目录信息，stdio 子进程的文件访问不会因此被限制在该目录内（[packages/coding-agent/src/extensions/mcp/index.ts:465-492](../../packages/coding-agent/src/extensions/mcp/index.ts#L465-L492)、[packages/coding-agent/src/extensions/mcp/runtime.ts:138-153](../../packages/coding-agent/src/extensions/mcp/runtime.ts#L138-L153)、[packages/coding-agent/src/extensions/mcp/runtime.ts:378-384](../../packages/coding-agent/src/extensions/mcp/runtime.ts#L378-L384)）。

## 脚本取得完整结果以后

Codemode 解决的是工具操作较多、返回内容较大时如何组织调用。例如搜索返回很多记录，模型可以写一段脚本调用搜索，再只输出需要的几项。脚本运行时仍须经由可用工具完成外部操作；脚本自己的执行环境与 bash 或 MCP 服务的权限范围分开。

QuickJS 是运行 JavaScript 的引擎，Pi 在其中建立受限脚本环境。这里的 VM 指脚本虚拟机，与上一章用于系统隔离的 micro-VM 不是同一层：它限制脚本能直接接触的宿主接口，桥接工具实际能执行什么，还要沿工具管线继续检查。结果输出、脚本状态保存和外部副作用也有不同提交边界。

工具返回的内容也可能很大。MCP 结果可以带文本、图像、资源、结构化内容和 `isError`。发送给模型的文本超过 20 KiB 时，适配层保留首尾，并把完整文本写入临时文件；非图像的二进制资源也保存为文件。

Codemode 让模型写一段 JavaScript，先调用工具，再决定输出哪些内容。脚本拿到完整的 `CallToolResult`，适配层只移除了顶层 `_meta`。同一份 `isError` 在直接调用时显示为错误，脚本中则仍可读取返回值，用 `result.isError` 自行判断（[packages/coding-agent/src/extensions/mcp/tools.ts:124-145](../../packages/coding-agent/src/extensions/mcp/tools.ts#L124-L145)、[packages/coding-agent/src/extensions/mcp/tools.ts:167-230](../../packages/coding-agent/src/extensions/mcp/tools.ts#L167-L230)、[packages/coding-agent/src/extensions/codemode/execute.ts:398-410](../../packages/coding-agent/src/extensions/codemode/execute.ts#L398-L410)）。

可调用的工具会包装成 `CodemodeTool`，通过 QuickJS 的基本类型桥接传递参数与返回值。脚本在 QuickJS 中执行，没有宿主 Node、文件系统或网络 API，外部操作只能通过桥接工具和可选的 `models` 全局接口发起。coding-agent 为 VM 设置了 256 MiB 内存上限，整段脚本默认没有超时期限，也可以由输入指定超时。脚本调用 `bash` 后，命令的实际运行位置依然取决于工具后端，QuickJS 只隔离脚本本身（[packages/coding-agent/src/extensions/codemode/execute.ts:441-495](../../packages/coding-agent/src/extensions/codemode/execute.ts#L441-L495)、[packages/codemode/src/runtime/vm.ts:81-91](../../packages/codemode/src/runtime/vm.ts#L81-L91)、[packages/codemode/src/runtime/vm.ts:135-188](../../packages/codemode/src/runtime/vm.ts#L135-L188)）。

脚本里的嵌套调用先进入 `ctx.executeTool()`，再由 `AgentSession` 执行同一套工具管线。事件带有 `parentToolCallId`，权限扩展因此也能看到脚本内部的调用，不会从这里直接绕到 MCP transport。脚本可以并发调用、发现工具、收集输出，并用 `store()` 保存少量状态；状态写入只在脚本成功后保存，失败之前已经发生的工具副作用则不会撤销（[packages/coding-agent/src/core/agent-session.ts:652-681](../../packages/coding-agent/src/core/agent-session.ts#L652-L681)、[packages/coding-agent/src/core/agent-session.ts:726-764](../../packages/coding-agent/src/core/agent-session.ts#L726-L764)、[packages/coding-agent/src/extensions/codemode/execute.ts:298-313](../../packages/coding-agent/src/extensions/codemode/execute.ts#L298-L313)、[packages/coding-agent/src/extensions/codemode/execute.ts:497-510](../../packages/coding-agent/src/extensions/codemode/execute.ts#L497-L510)）。

MCP 客户端库也可以单独使用。调用 `new McpClient().callTool()` 时，只有客户端本身的行为；Pi 会话里的扩展钩子由 coding-agent 集成层另外接入。

## 边界与未实现

上面的过程说明了消息与工具调用怎样经过这条管线；以下各项补充它不提供的能力，以及断线时的重试边界。

- **断线后，哪些请求会再发？** 网络恢复后，是否重新发送请求要看操作类型。`McpServerConnection.withClient()` 对只读资源请求的瞬态 HTTP 错误重试一次，普通工具调用则不按这条规则重放，因为服务可能已经执行了操作。已建立 session 返回 404 时会走单独分支：源码认为服务已不认识这个 session，该请求没有执行，因此换一个 session 重试一次。这个判断只适用于该分支，所有工具并没有因此获得幂等保证（[packages/coding-agent/src/extensions/mcp/runtime.ts:265-291](../../packages/coding-agent/src/extensions/mcp/runtime.ts#L265-L291)、[packages/coding-agent/src/extensions/mcp/runtime.ts:293-320](../../packages/coding-agent/src/extensions/mcp/runtime.ts#L293-L320)）。
- **接入层不附带隔离与审批。** 这套接入没有自动建立 OS 隔离或审批策略，也没有通用 MCP 服务沙箱、强制工具审批、自动副作用回滚或跨服务事务。
- **capability（能力声明）字段不等于已实现范围。** 协议类型中列有 capability 字段，实际覆盖范围仍要从注册的工具、request handler 和资源转换路径确认；它不代表客户端或 coding-agent 已实现所有 MCP 服务端能力，也不能证明全部 prompts、sampling、elicitation 和 MCP Apps UI 已受支持。

## 源码位置与术语速查

通信部分由 `packages/mcp` 提供，包括 JSON-RPC 客户端、stdio/Streamable HTTP、工具与资源请求和 OAuth。coding-agent 集成层再处理配置、连接状态、发现、命名、呈现方式、结果转换和管理 UI。

### 文中术语

| 术语 | 在这个项目中的含义 |
| --- | --- |
| transport（传输） | 消息传输通道，底层可以是子进程或 HTTP；`McpTransport` 用 `start/send/close` 和事件接口统一两者 |
| JSON-RPC | MCP 的消息格式：请求带 `id/method/params`，响应按 `id` 匹配等待者，通知没有请求 ID |
| Streamable HTTP | 远端传输方式：POST 发送请求，响应返回 JSON 或 SSE，握手后才打开服务端到客户端的 GET 流 |
| 呈现方式（exposure） | 决定工具怎样呈现给模型、通过哪条路径调用的配置：`codemode`/`deferred`/`direct`/`hidden` |
| Codemode | 默认呈现路径：模型写一段 JavaScript，在 QuickJS VM 中发现并调用工具、决定输出内容 |
| 资源（resource） | 服务提供的、用 URI 标识的可读内容；Pi 将其适配成 list、list templates、read 三个工具 |

## 测试与练习

可以先跟着 [packages/mcp/test/client.test.ts:19-64](../../packages/mcp/test/client.test.ts#L19-L64) 用内存 transport 构造假服务，这条路径不启动外部进程，也不访问网络。[packages/mcp/test/client.test.ts:71-126](../../packages/mcp/test/client.test.ts#L71-L126) 检查握手顺序、工具分页和定义保留，[packages/mcp/test/client.test.ts:185-226](../../packages/mcp/test/client.test.ts#L185-L226) 再检查 progress 延长超时与取消通知。

通道和内容各有独立测试：[packages/mcp/test/stdio.test.ts:9-28](../../packages/mcp/test/stdio.test.ts#L9-L28) 用本地 fixture 子进程验证工具调用与 stderr；[packages/mcp/test/content.test.ts:5-34](../../packages/mcp/test/content.test.ts#L5-L34) 检查文本和图像通过、其他内容转换，以及结构化内容的 fallback。

脚本边界可以从 [packages/codemode/test/sandbox.test.ts:686-732](../../packages/codemode/test/sandbox.test.ts#L686-L732) 看起：宿主全局变量不可见，eval 和 Function 仍在 VM 内执行，dynamic import 也无法取得 Node 模块。

复用 `createServer()` 做两处小改动就能观察协议差异。先让第二页的 `nextCursor` 重复，检查客户端是否拒绝继续翻页；再让假服务分别返回 JSON-RPC error 和 `{ content: [], isError: true }`。前者使客户端 Promise 以错误结束等待，后者仍返回 `CallToolResult`，直到 coding-agent 适配层才转换成模型侧的错误结果。

脚本练习只需注册一个本地假 `echo` 工具：并发调用两次，再用 `text()` 汇总。接着让脚本抛错，检查工具调用次数仍为二。这样可以同时看到脚本执行失败和工具副作用已经发生这两件事。

已有仓库依赖时，在 `packages/mcp` 可单独运行这些离线客户端测试。

```powershell
node ../../node_modules/vitest/dist/cli.js --run test/client.test.ts test/content.test.ts test/stdio.test.ts
```

[权限与秘密](10-permissions-secrets.md) 说明工具调用进入管线之后如何处理授权；[重试、超时与幂等](06-retry-timeout-idempotency.md) 继续解释为什么请求失败时，操作仍可能已经发生。

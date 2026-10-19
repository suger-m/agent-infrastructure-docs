# 第 9 章　工具基础设施与 MCP

[返回目录](README.md) · [上一章：第 8 章 沙箱与执行环境](08-sandbox-environment.md) · [下一章：第 10 章 权限、安全与凭据](10-permissions-secrets.md)

工具基础设施负责把模型提出的操作请求交给实际执行程序，并将结果送回会话。本章先介绍工具定义、注册与发现，再以订单核对任务说明一次调用的完整过程，以及错误、取消和资源清理的关系。在此基础上，介绍 MCP 的参与者与通信方式，最后对照 Pi 的连接、工具适配和脚本执行实现。源码说明以提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc` 为基线。

## 9.1 工具基础设施的职责

用户要求 Agent 核对订单 A17 的发货状态，并与本地记录比较。数据库保存当前订单状态，文件保存上次导出的记录。如果两者不一致，Agent 还可能需要通过浏览器查看物流页面、调用物流 API，或者用 Shell 命令整理导出文件。数据库查询、文件读写、浏览器操作、网络 API 和进程执行的参数与失败方式各不相同，但都需要由程序实际执行。

模型可以生成“调用订单查询工具，订单号为 A17”这样的结构化请求。运行 Agent 的程序称为宿主，它负责查找工具实现、校验参数、执行授权检查，并调用后端。模型选定工具后，数据库仍未被读取；只有后端执行并返回结果，这次查询才获得数据依据。

工具基础设施为这些能力提供共同的调用入口。Agent 主循环提交工具名和参数，等待与本次调用关联的结果；数据库连接、浏览器操作、进程退出码和 API 错误则由相应实现处理。这样，主循环可以沿用同一套调用过程，不必分别管理所有后端协议。本章的订单系统是教学设定，不表示 Pi 已实现这些业务能力，也不要求所有工具都使用 MCP。

## 9.2 工具定义、注册与发现

一个工具首先需要能够被模型理解的定义。以 `lookup_order` 为例，名称用于选择执行入口，描述说明它按订单号读取发货状态，输入 schema 规定参数必须包含字符串类型的 `orderId`。Schema 是机器可读的结构约定，可以表达字段类型、必填项和允许值。输出也可以使用 schema，约定成功结果中的订单号与发货状态，便于后续程序读取。

描述和 schema 分别提供语义与结构信息。“查询订单”的描述说明用途，却不能判断请求是否缺少 `orderId`；一个字符串字段能够被校验，却不能单独说明它代表订单号还是客户名。宿主需要同时提供这两种信息，模型才能选择合适的工具，程序才能校验生成的参数。

宿主还要把工具定义与执行函数建立对应关系。将名称、定义和执行入口保存到注册表的过程称为注册；从内置模块或外部服务取得可用定义的过程称为发现。内置文件工具可以在启动时注册，外部工具则可以先从服务取得清单，再适配到宿主接口。

注册表与模型本轮收到的定义不必相同。宿主可以管理大量工具，而每次只向模型提供当前任务所需的部分。无论定义如何呈现，执行时都要重新查找对应入口并检查其状态。

适配器负责将宿主的共同调用格式转换成具体后端操作。例如，数据库适配器管理查询与连接，浏览器适配器定位页面并执行操作，Shell 适配器启动命令并收集输出。共同接口统一调用过程，后端差异仍由各自的适配器处理。

## 9.3 工具调用的执行流程

假设模型已经收到 `lookup_order` 的定义，准备查询 A17，再根据结果读取本地记录。下列流程属于通用教学架构。

1. 模型返回工具名 `lookup_order` 和参数 `{"orderId":"A17"}`。宿主为调用保存标识和等待状态，以便并发查询时仍能正确关联响应。
2. 调度层查询注册表，确认工具可用，并按 schema 校验参数。如果缺少 `orderId`，宿主返回参数错误，数据库不会收到查询。
3. 参数校验通过后，宿主检查当前主体是否有权读取该订单。订单号即使格式正确，也可能属于其他客户。授权需要结合身份、动作和资源判断，见[第 10 章 权限、安全与凭据](10-permissions-secrets.md)。
4. 数据库适配器取得连接并执行查询。工具管理层向它传递取消信号和截止时间；适配器管理数据库连接，宿主管理调用等待状态，模型不持有连接。
5. 查询完成后，适配器生成约定的工具结果。查无订单应返回“未找到”，连接失败则应返回执行错误。两种结果为后续判断提供的依据不同，不能相互替代。
6. 宿主按调用标识将结果加入会话上下文，连接按约定释放或返回连接池。下一次模型请求收到 A17 的状态后，才有依据继续读取本地记录。

参数校验只检查输入结构。例如，Shell 的 `command` 是字符串，并不能证明其中的删除命令符合用户要求；文件路径格式正确，也不能证明它位于授权目录。操作含义、授权范围和执行环境约束需要分别处理。

共同结果格式还应保留内容类型。浏览器截图适合以图像返回，数据库记录适合以结构化数据返回，诊断信息可以使用文本。宿主需要知道每段内容的类型、所属调用和错误状态，再转换成模型提供方接受的消息格式。

资源的创建、复用与清理同样属于执行流程。文件句柄通常在读取后关闭，数据库连接可以回到连接池，浏览器页面可能跨调用保留，Shell 子进程则需要等待退出或被终止。每次调用都重建浏览器会增加启动成本，长期保留所有页面又会消耗资源，因此宿主和后端必须约定资源所有者、保留期限及会话结束时的清理责任。

![工具定义经注册与发现后，调用依次验证参数、检查授权、执行并按调用 ID 返回](figures/09-concept-tool-pipeline.svg)

图示中的参数验证与授权都发生在后端执行之前，分别判断输入结构与操作资格。

## 9.4 错误语义、取消与结果容量

参数错误在执行前被发现，修正输入后可以重新请求。执行中的超时则只证明客户端未能及时取得结果，不能确定后端进度。查询可能仍在运行；如果操作改为修改收货地址，写入可能已经完成，只是响应丢失。将不同后端错误转换为共同格式时，应保留失败阶段与已有证据，避免把所有异常都解释为可以重试。

取消信号需要沿调用链传递。用户取消后，宿主通知适配器，由适配器尝试取消数据库操作或终止子进程。后端不支持取消时，客户端可以停止等待，但执行仍可能继续。资源清理也应覆盖这些情况，避免会话结束后连接和进程仍被占用。已经完成的写入不会因取消而自动撤销，结果未知时的处理见[第 6 章 重试、超时与幂等](06-retry-timeout-idempotency.md)。

结果容量会影响后续模型请求。若任务扩大为核对全年订单，将全部明细直接加入上下文可能占用用户约束和其他证据所需的空间。工具可以分页返回、截断展示内容，或者把完整结果保存成文件及可再次读取的资源。截断结果应明确说明不完整，并提供获取剩余内容的方法，防止模型将片段视为全集。也可以由程序先筛选记录，再提供当前判断所需的数据；筛选后的内容仍应保留原始结果的来源。

## 9.5 工具定义的容量与更新

工具较少时，宿主可以在每次请求中提供全部定义。工具达到数百个后，名称、描述与 schema 本身会占用大量上下文，相近描述也会增加选择难度。按需发现允许模型先搜索候选，再取得相关定义。这会增加一次发现过程，并要求搜索描述足以区分用途。减少可见工具能够缩小选择范围，实际执行时仍需独立检查权限。

工具清单还会随服务更新。假设 `lookup_order` 增加一个必填字段，而会话仍使用旧 schema，模型生成的请求就可能被服务拒绝。宿主获知清单变化后，需要刷新注册表、移除失效入口，并在后续请求中提供新定义。刷新和调用可能同时发生，所以服务端仍须校验收到的参数，客户端也应将定义变更或工具删除错误返回会话。共同接口无法保证外部工具始终兼容旧请求。

## 9.6 MCP 的角色与通信过程

MCP（Model Context Protocol，模型上下文协议）约定客户端与外部服务之间的初始化、工具与资源发现、调用及结果交换。不同服务遵守同一协议后，宿主可以复用连接代码。Shell、Filesystem、Browser、DB 和 API 能力既可以通过 MCP 提供，也可以直接实现为宿主内的工具。

MCP 服务端（server）公布能力并处理请求，例如将订单查询转交数据库。客户端（client）由宿主持有，负责与一个服务通信、保存连接状态，并关联请求和响应。宿主在客户端之上决定模型可见的工具、调用是否获准，以及结果如何进入会话。

协议（protocol）规定消息含义和交互顺序，传输（transport）负责把消息送达对端。前者要求初始化完成后使用已协商的能力，后者可以通过本机进程标准流或 HTTP 传递这些消息。替换传输可以适应不同部署方式，而无需让双方共享数据库驱动或 Agent 主循环。

初始化握手确定协议版本与能力范围，随后客户端仍要发现具体工具。协议版本规定消息规则，工具定义规定某项操作的参数，两者可能独立变化。能力声明也需要对应的处理逻辑才能生效。MCP 不会自动授予资源访问权、建立进程隔离或要求人工审批。

![宿主持有 MCP 客户端，客户端通过本机 stdio 或远端 HTTP 与服务端通信](figures/09-concept-mcp-roles.svg)

图中的 stdio 与 HTTP 表示传输方式。它们不决定服务进程的隔离程度，也不赋予客户端管理远端进程的权限。

### 9.6.1 本机 stdio 连接

本机服务由宿主根据启动命令和运行配置创建。客户端通过子进程的标准输入与标准输出交换消息，这种传输方式称为 stdio；标准错误流可用于输出诊断日志，避免日志混入协议消息。

通道建立后，客户端发送初始化请求，服务返回协议版本、能力和服务信息。客户端确认版本，发送初始化完成通知，再请求工具清单。宿主将名称、描述及输入 schema 适配并注册，随后才向模型提供这些工具。模型发起调用时，宿主先执行本地检查，再由客户端发送请求；服务处理完成后沿同一通道返回结果。

会话结束时，宿主关闭连接，并按进程管理策略结束自己启动的服务。子进程能够访问哪些文件和网络资源，仍由启动环境和额外隔离措施决定。stdio 通道本身不会将文件访问限制在工作目录中。

### 9.6.2 远端 HTTP 连接

远端服务由服务提供方启动和维护。宿主取得地址及必要凭据后，通过 HTTP 初始化连接、协商能力、发现工具并注册定义。后续调用仍经过宿主工具管线，远端服务收到请求后才访问数据库。客户端关闭的是自己的会话和等待请求，通常无法管理远端服务进程。

Streamable HTTP 使用 POST 发送消息，响应可以是 JSON，也可以采用 SSE，即服务端发送事件格式，持续传递消息。服务支持且客户端需要时，还可建立 GET 流来接收服务消息。若服务分配了会话标识，后续请求携带该标识。HTTP 负责跨网络传输，协议中的请求标识负责关联每次调用及其响应。

网络中断可能使客户端无法确定工具是否已执行。恢复接收结果的流与重新发送操作请求必须采用不同规则，尤其不能因连接恢复就重复具有副作用的操作。认证失败则需要处理凭据或授权范围。Pi 对续流、认证与调用重试的具体处理见后面的参考实现。

## 9.7 参考实现：echo 工具的配置、发现与调用

本节以本机 stdio `echo` 服务说明 Pi 如何组织连接与调用。服务接收 `text` 参数并返回相同文字；输入 `{"text":"hello"}`、返回 `hello` 用于展示数据流，不是仓库 fixture 的实测输出。示例选择 `direct` 呈现方式，使模型能直接取得工具定义。Pi 的默认呈现方式为 `codemode`，两种方式的区别见 9.7.4。

### 9.7.1 服务配置与初始化握手

服务配置指定连接地址或启动命令，并决定工具的呈现方式。coding-agent 先读全局 `mcp.json`；项目可信时再读 `.pi/mcp.json`。项目中同名条目可以替换服务，也可以只覆盖 `enabled`、`exposure`（工具呈现方式）和 `toolExposure`。同名文件配置优先于扩展注册（[packages/coding-agent/src/extensions/mcp/config.ts:91-155](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/config.ts#L91-L155)、[packages/coding-agent/src/extensions/mcp/index.ts:345-363](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/index.ts#L345-L363)）。

在 `session_start` 阶段，Pi 在后台启动连接。第一次 prompt 只在有限时间内等待 `direct` 服务；`codemode`、`tool_search` 和资源工具则在实际使用时等待相应服务就绪。因此，非直接工具的连接延迟不会要求第一次模型请求一直等待（[packages/coding-agent/src/extensions/mcp/index.ts:1111-1206](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/index.ts#L1111-L1206)）。

配置给出命令后，Pi 启动 echo 子进程。本地 stdio 传输用标准输入和标准输出交换协议消息，标准错误流留作诊断。客户端先启动通道，再发送初始化请求。服务返回协议版本、能力和服务信息后，客户端发送 `notifications/initialized`；完成这次握手后才开始询问工具清单，避免在服务尚未准备好时发送调用。

`McpTransport` 以 `start/send/close` 和事件接口统一 stdio 子进程与 HTTP 通道；`McpClient.connect()` 在通道上完成握手与版本校验（[packages/mcp/src/transports/transport.ts:9-17](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/transport.ts#L9-L17)、[packages/mcp/src/client.ts:202-248](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/client.ts#L202-L248)）。

### 9.7.2 定义发现与会话工具适配

初始化完成后，客户端通过 `tools/list` 取得名称、说明和 JSON Schema。Pi 再将这些定义转换成会话工具；模型只有收到相应声明，才能据此选择操作并构造参数。

coding-agent 将服务名与工具名组合成 Pi 工具名。例如服务 `docs` 中的 `search` 可以成为 `mcp__docs__search`，以便 Agent 区分不同服务中的同名工具。适配定义保留输入 schema、输出 schema 和布尔 annotations，实际发送给服务的仍是服务原来的工具名（[packages/coding-agent/src/extensions/mcp/tools.ts:78-93](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/tools.ts#L78-L93)、[packages/coding-agent/src/extensions/mcp/tools.ts:244-295](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/tools.ts#L244-L295)）。

服务清单也可能在会话中改变。服务发出 `tools/list_changed` 后，客户端刷新定义；已经删除或禁用的工具会以 hidden 形式重新注册，让模型不再看到旧定义（[packages/coding-agent/src/extensions/mcp/runtime.ts:378-435](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L378-L435)、[packages/coding-agent/src/extensions/mcp/index.ts:446-463](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/index.ts#L446-L463)）。

### 9.7.3 调用适配与结果转换

echo 定义进入会话后，模型生成包含工具名和参数的 Pi 工具调用。AgentSession 先校验参数并运行扩展工具钩子，再由适配工具调用 MCP 客户端，将 `{"text":"hello"}` 包装为发往 echo 服务的 `tools/call`。Codemode 发起的嵌套调用也会经过同一个 AgentSession 管线，并携带 `parentToolCallId`，所以扩展钩子也能看到这些调用（[packages/coding-agent/src/core/agent-session.ts:652-681](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L652-L681)、[packages/coding-agent/src/core/agent-session.ts:726-764](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L726-L764)）。

服务执行后，MCP 客户端按请求关联响应，再由适配层转换为 Agent 能处理的工具结果。结果可以含文本、图像、资源、结构化内容和 `isError`。本例收到文本 `hello`，Pi 将它交回 Agent，随后进入下一次模型请求。direct 路径会把服务标记的 `isError` 转成工具错误（[packages/coding-agent/src/extensions/mcp/tools.ts:124-145](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/tools.ts#L124-L145)、[packages/coding-agent/src/extensions/mcp/tools.ts:167-230](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/tools.ts#L167-L230)）。

JSON-RPC 请求由 MCP 适配层在 AgentSession 接收工具调用后构造，模型输出仍是 Pi 工具名与参数。下列两图分别展示服务连接与注册，以及注册后的完整调用管线。图示限定于 Pi 的 MCP 集成；本节的 stdio direct 示例不需要经过 HTTP 或 Codemode，也未因此获得执行隔离。

![MCP 服务从读取配置、建立传输、握手发现，到工具调用和结果返回](figures/09-mcp-tool-path.svg)

![MCP 服务连接、工具定义注册与直接调用或 Codemode 调用的完整管线](figures/09-mcp-integration.svg)

### 9.7.4 工具呈现方式与 Codemode

逐个注册工具适合工具数量有限的服务。如果有数百个工具，每次把所有参数定义都交给模型会占用大量上下文。Pi 因此提供不同呈现方式，控制工具何时对模型可见：

| 配置值 | 模型使用方式 |
| --- | --- |
| `codemode`（默认） | 脚本通过 `searchTools/describeTool` 发现并调用 |
| `deferred` | `tool_search` 加载声明后直接调用；不要求 Codemode |
| `direct` | 像内置工具一样直接声明，也可被脚本调用 |
| `hidden` | 在这套工具管线中不可达 |

`deferred` 让模型先搜索再加载所需定义；`codemode` 让模型写 JavaScript 来发现和组合调用。两者都减少一次性提供全部工具定义的需要。Codemode 是工具呈现和调用组织方式，MCP 连接不要求经过它。呈现方式只控制模型看到哪些工具、何时看到定义，不会改变服务的操作系统权限。服务给出的 annotations 也是提示，不能证明调用受到了执行隔离（[packages/coding-agent/src/core/mcp-servers.ts:43-77](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/mcp-servers.ts#L43-L77)、[packages/coding-agent/src/core/mcp-servers.ts:233-242](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/mcp-servers.ts#L233-L242)）。

工具定义之外，返回内容也占用上下文。适配层发送给模型的文本若超过 20 KiB，会保留首尾片段并把完整文本写入临时文件；非图像的二进制资源也保存为文件。Codemode 脚本可以组合调用，只把后续推理所需的部分交给模型。在 Codemode 路径中，脚本读取完整的 `CallToolResult`（适配层移除顶层 `_meta`），可以自行检查 `result.isError` 并选择输出内容。这里的错误处理与 direct 不同：direct 由适配层把 `isError` 转为工具错误；脚本路径把完整结果交给脚本，由脚本选择后续处理方式（[packages/coding-agent/src/extensions/mcp/tools.ts:275-285](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/coding-agent/src/extensions/mcp/tools.ts#L275-L285)、[packages/coding-agent/src/extensions/codemode/execute.ts:422-434](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/coding-agent/src/extensions/codemode/execute.ts#L422-L434)）。

## 9.8 参考实现：HTTP 连接、续流与认证

Pi 连接已部署的远端服务时，用 Streamable HTTP 代替 echo 示例中的本机进程通道。HTTP 通过 POST 发送消息，响应可以是 JSON 或 SSE；握手后客户端再打开到服务端的 GET 流，并带上会话 ID 和协议版本。

若 GET 流中断且服务为事件分配了 ID，客户端用 `Last-Event-ID` 续接 GET 流，从断点继续接收结果。它不会因此重发原来的工具 POST。没有事件 ID 的响应流提前结束时，请求失败（[packages/mcp/src/transports/streamable-http.ts:225-256](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/streamable-http.ts#L225-L256)、[packages/mcp/src/transports/streamable-http.ts:309-320](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/streamable-http.ts#L309-L320)、[packages/mcp/src/transports/streamable-http.ts:361-404](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/streamable-http.ts#L361-L404)）。

HTTP 的认证是服务连接自己的认证路径。收到 401，或收到要求更大 scope（授权范围）的 403 时，客户端会交由认证提供方处理一次，再用新凭证重试。这与恢复网络事件流是两条不同路径；凭证如何取得见[第 10 章 权限、安全与凭据](10-permissions-secrets.md)（[packages/mcp/src/transports/streamable-http.ts:159-169](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/streamable-http.ts#L159-L169)、[packages/mcp/src/transports/streamable-http.ts:277-307](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/streamable-http.ts#L277-L307)）。

## 9.9 参考实现：命名、脚本运行与资源边界

注册名要符合模型提供方对工具名的限制：只能使用字母、数字、下划线和连字符，最长 64 个字符。Pi 会先把不符合规则的字符替换为下划线。若生成名过长，或清理后与另一个原始服务名/工具名撞名，会在截短后的名称末尾附加由原始服务名和工具名计算出的哈希，避免覆盖定义（[packages/coding-agent/src/extensions/mcp/tools.ts:48-93](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/tools.ts#L48-L93)）。

QuickJS 是运行 JavaScript 的引擎。Pi 在其中建立受限脚本环境；脚本没有宿主 Node、文件系统或网络 API，外部操作只能通过桥接工具和可选的 `models` 接口发起。VM 默认内存上限为 256 MiB，整段脚本默认没有超时，也可以由输入指定超时。脚本调用 bash 时，命令实际在哪里执行仍由工具后端决定；VM 限制的是脚本访问宿主接口的能力，不等同于沙箱 bash（[packages/coding-agent/src/extensions/codemode/execute.ts:473-500](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/coding-agent/src/extensions/codemode/execute.ts#L473-L500)、[packages/coding-agent/src/extensions/codemode/execute.ts:502-525](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/coding-agent/src/extensions/codemode/execute.ts#L502-L525)、[packages/codemode/src/runtime/prelude-source.ts:1-30](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/codemode/src/runtime/prelude-source.ts#L1-L30)）。

脚本可以并发调用工具、发现定义、收集输出，并用 `store()` 保存少量状态。状态只在脚本成功后保存；若脚本失败，先前调用已经造成的工具副作用不会撤销（[packages/coding-agent/src/extensions/codemode/execute.ts:539-544](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/coding-agent/src/extensions/codemode/execute.ts#L539-L544)、[packages/coding-agent/src/extensions/codemode/execute.ts:322-325](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/coding-agent/src/extensions/codemode/execute.ts#L322-L325)）。MCP 客户端也能脱离 coding-agent 单独使用，但单独调用 `new McpClient().callTool()` 不会接入 coding-agent 的会话扩展钩子。

### 9.9.1 durable 工具层：嵌套调用与结构化输出

durable 包用另一套工具接口运行可持久化任务，每次工具调用都是独立任务。工具用 `defineTool()` 与 TypeBox schema 定义，Harness 在 `execute()` 前校验参数并先提交调用意图；进程在调用中途崩溃时，只有声明 `replay: "safe"` 的工具会在重开后重跑，否则模型收到 `interrupted` 错误结果，已提交的输出保留（[packages/durable/src/harness/types.ts:294-327](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/types.ts#L294-L327)）。

工具结果分三个通道：`output` 给模型，`structuredOutput` 给调用该工具的程序（`executeTool()` 调用者、Codemode 脚本），`details` 给 UI。声明 `structuredOutputSchema` 的工具必须返回匹配的值，Harness 校验失败会记为 `invalid_structured_output`；转录只保存 `output` 和 `details`（[packages/durable/src/harness/types.ts:153-196](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/types.ts#L153-L196)、[packages/durable/src/harness/tool.ts:595-616](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/tool.ts#L595-L616)）。

一个工具可以用 `api.executeTool(name, args, ctx, { key? })` 调用另一个工具。嵌套调用是调用者拥有的独立工具任务，有自己的参数校验、钩子、输出限制和重放策略；结果不回转录，直接返回给调用者 `taskId`、`structuredOutput`、`details`、`diagnostics`、`usage` 和 `isError`（[packages/durable/src/harness/tool.ts:378-406](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/tool.ts#L378-L406)）。嵌套调用的 key 默认为它在调用者中的位置序号；replay-safe 的调用者重跑后按相同顺序或相同 key 调用时，会取回已完成的嵌套调用，而不是重新执行（[packages/durable/src/harness/tool.ts:380-381](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/tool.ts#L380-L381)）。

`callers` 声明工具允许谁调用（模型、其他工具或两者），`modelTools` 只收窄模型可见集合，不剥夺工具的调用权；`agent()` 因此解析出 `tools`（提供给模型）与 `callable`（嵌套调用可解析）两个列表。`deferred: true` 的工具不主动提供给模型，由 `tool_search` 扩展按 BM25 检索后经 `control.addTools` 写入 `modelTools`，加载结果在该分支持续可见，压缩后也在（[packages/durable/src/harness/types.ts:451-461](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/types.ts#L451-L461)、[packages/durable/src/tool-search/tool-search.ts:1-5](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/tool-search/tool-search.ts#L1-L5)、[packages/durable/src/tool-search/tool-search.ts:55-87](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/tool-search/tool-search.ts#L55-L87)）。Codemode 的提示段同样排除 deferred 工具，脚本用 `searchTools()` 自行查找（[packages/durable/src/codemode/codemode.ts:252-261](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L252-L261)）。

### 9.9.2 durable Codemode：脚本的确定性重放

durable 包也提供 Codemode 工具：脚本在 QuickJS 沙箱中运行，`await tools.<name>(args)` 的每次调用都是嵌套工具调用，只有脚本输出回到模型（[packages/durable/src/codemode/codemode.ts:231-243](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L231-L243)）。

跨重启继续的机制是确定性执行加结果日志。沙箱以虚拟时钟、种子随机数和 UTC 运行脚本；第一次运行前，启动数据（模块版本、种子、工具声明、`store` 快照）写入任务文档 `pi.codemode.run`，每个交付给脚本的结果先写入 `pi.codemode.journal` 再交付。重启后 `execute()` 重新执行：脚本从相同的启动数据重新开始，journal 中的结果按原顺序交还，嵌套调用按 key（`c<VM 调用号>`）接回原任务而不是重新执行，随后脚本继续实时运行（[packages/durable/src/codemode/codemode.ts:1-36](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L1-L36)、[packages/durable/src/codemode/codemode.ts:124-153](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L124-L153)、[packages/durable/src/codemode/codemode.ts:403-442](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L403-L442)、[packages/durable/src/codemode/codemode.ts:817-840](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L817-L840)）。

重放在三种情况下改为以错误结果结束：模块版本或沙箱指纹变化、脚本 deadline 已过、脚本路径分歧（发出 journal 未记录的新调用，或顺序不同）（[packages/durable/src/codemode/codemode.ts:444-454](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L444-L454)、[packages/durable/src/codemode/codemode.ts:487-505](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L487-L505)、[packages/durable/src/codemode/codemode.ts:610-637](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L610-L637)）。`store()`/`load()` 的值按会话保存，可回退，fork 时按 fork 点复制（[packages/durable/src/codemode/codemode.ts:155-166](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L155-L166)）。脚本接口、工具声明渲染与执行边界、以及 durable 重放的完整专题见[附录 A：Codemode](appendix-codemode.md)。

服务还可以提供以 URI 标识、供客户端读取的资源。Pi 将资源访问适配成 list、list templates、read 三个工具。当前不渲染 MCP App UI，因此相应内容会被过滤。客户端宣告的 `roots` 是目录信息，不会把 stdio 子进程的文件访问限制在该目录内（[packages/coding-agent/src/extensions/mcp/index.ts:465-492](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/index.ts#L465-L492)、[packages/coding-agent/src/extensions/mcp/runtime.ts:138-153](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L138-L153)、[packages/coding-agent/src/extensions/mcp/runtime.ts:378-384](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L378-L384)）。

资源工具的呈现范围由可读资源的服务决定：只要有一个此类服务采用 `direct`，资源工具就按最宽的可达范围注册；没有这类服务时则隐藏。这个范围说明模型如何看到资源入口，不限制子进程的文件访问（[packages/coding-agent/src/extensions/mcp/index.ts:465-492](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/index.ts#L465-L492)）。

## 9.10 参考实现：消息关联、分页、关闭与重试

服务可能同时处理多个请求。JSON-RPC（请求响应消息格式）给每个请求分配 `id`；请求包含 `id/method/params`，响应按 ID 找到等待者，notification（通知）不等待响应。`requestInternal()` 把等待者存入 pending Map，并设置超时与取消；`listTools()`、`callTool()` 选择协议方法并检查返回形状（[packages/mcp/src/client.ts:394-439](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/client.ts#L394-L439)、[packages/mcp/src/client.ts:376-384](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/client.ts#L376-L384)）。

`tools/list` 可以分页。客户端最多读取 1000 页，重复 cursor 会被拒绝；收到 progress 会重新计时。超时和取消通常会通知服务端，初始化请求按协议不发 cancelled 通知。结束客户端等待无法撤销服务端已经发生的写入（[packages/mcp/src/client.ts:354-374](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/client.ts#L354-L374)、[packages/mcp/src/client.ts:531-577](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/client.ts#L531-L577)）。

stdio 子进程默认继承 `process.env`。底层支持 `inheritEnv: false`，但 coding-agent 默认工厂未设置它。关闭时先结束 stdin，让服务有机会自行退出；若服务仍未退出，再终止并清理子进程树（[packages/mcp/src/transports/stdio.ts:90-141](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/stdio.ts#L90-L141)、[packages/mcp/src/transports/stdio.ts:143-207](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/stdio.ts#L143-L207)、[packages/coding-agent/src/extensions/mcp/runtime.ts:98-122](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L98-L122)）。

本章基线中的协议常量为 `2025-11-25`，并接受列表中的三个旧版本。Streamable HTTP 响应或 GET 流可以使用 SSE 格式；配置拒绝旧版独立 legacy SSE transport（[packages/mcp/src/protocol/types.ts:4-10](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/protocol/types.ts#L4-L10)、[packages/coding-agent/src/core/mcp-servers.ts:248-275](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/mcp-servers.ts#L248-L275)）。

网络恢复后的重试按操作类型区分。`McpServerConnection.withClient()` 对只读资源请求的瞬态 HTTP 错误重试一次；普通工具调用不会自动重放，因为服务可能已经执行操作。已建立 session 返回 404 时，当前分支认为服务不认识该 session、请求尚未执行，因此换 session 重试一次。这个特例不保证所有工具调用都幂等（[packages/coding-agent/src/extensions/mcp/runtime.ts:265-291](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L265-L291)、[packages/coding-agent/src/extensions/mcp/runtime.ts:293-320](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L293-L320)）。

MCP 接入本身不建立 OS 隔离、审批、通用服务沙箱、副作用回滚或跨服务事务。协议类型列出的 capability（能力声明）字段不表示客户端已实现所有服务端能力，也不能证明 prompts、sampling、elicitation 或 MCP Apps UI 都受支持。具体范围取决于注册了哪些工具、request handler 实现了什么，以及资源转换规则。

## 9.11 源码定位与测试

| 机制 | 源码入口 |
| --- | --- |
| 配置与连接 | [配置合并](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/config.ts#L91-L155)、[后台连接](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/index.ts#L1111-L1206) |
| 协议与调用关联 | [初始化握手](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/client.ts#L202-L248)、[请求等待与响应关联](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/client.ts#L394-L439) |
| 工具适配 | [工具定义注册](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/tools.ts#L244-L295)、[结果转换](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/tools.ts#L167-L230) |
| HTTP 恢复与重试 | [事件流恢复](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/streamable-http.ts#L361-L404)、[调用重试边界](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/extensions/mcp/runtime.ts#L293-L320) |
| 脚本与进程环境 | [VM 限制](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/codemode/src/runtime/vm.ts#L160-L179)、[stdio 进程启动](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/src/transports/stdio.ts#L90-L141) |
| durable 工具层 | [嵌套调用](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/tool.ts#L378-L406)、[结构化输出校验](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/tool.ts#L595-L616)、[deferred 与 tool_search](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/tool-search/tool-search.ts#L55-L87) |
| durable Codemode 重放 | [任务文档与日志](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L124-L166)、[重放与分歧检查](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/codemode/codemode.ts#L403-L505) |

通信基础由 `packages/mcp` 提供 JSON-RPC 客户端、stdio/Streamable HTTP、工具与资源请求和 OAuth；coding-agent 集成层负责配置、连接状态、发现、命名、呈现方式、结果转换和管理 UI。

可用 [packages/mcp/test/client.test.ts:19-64](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/test/client.test.ts#L19-L64) 的内存 transport 构造假服务，不必启动外部进程或访问网络。[packages/mcp/test/client.test.ts:71-126](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/test/client.test.ts#L71-L126) 检查握手顺序、工具分页和定义保留；[packages/mcp/test/client.test.ts:185-226](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/test/client.test.ts#L185-L226) 检查 progress 延长超时与取消通知。

[packages/mcp/test/stdio.test.ts:9-28](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/test/stdio.test.ts#L9-L28) 用本地 fixture 子进程验证工具调用与 stderr；[packages/mcp/test/content.test.ts:5-34](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/mcp/test/content.test.ts#L5-L34) 检查文本、图像、其他内容转换和结构化内容 fallback。

[packages/codemode/test/sandbox.test.ts:686-732](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/codemode/test/sandbox.test.ts#L686-L732) 检查宿主全局变量不可见、eval 和 Function 在 VM 内执行，以及 dynamic import 无法取得 Node 模块。

## 9.12 本章术语

| 术语 | 本章含义 |
| --- | --- |
| 工具定义 | 向模型与程序说明名称、用途及参数结构的声明 |
| 注册与发现 | 分别建立工具执行入口，以及取得可用工具定义 |
| 适配器 | 在宿主调用格式与具体后端之间转换请求和结果的组件 |
| MCP 客户端与服务端 | 分别管理协议请求，以及公布能力并执行请求的参与者 |
| Transport | 传递协议消息的通道，如 stdio 与 Streamable HTTP |
| Codemode | 通过受限 JavaScript 脚本发现、组合和调用工具的方式 |
| 嵌套调用 | 一次工具执行中通过 `executeTool()` 发起的另一个工具任务，结果不进入转录 |
| 确定性重放 | Codemode 脚本重启后按结果日志继续执行的机制，不重跑已完成调用 |
| 副作用 | 工具对文件或外部系统造成的持久变化 |

## 9.13 练习

复用 `createServer()` 可以观察协议差异：让第二页的 `nextCursor` 重复，确认客户端拒绝继续翻页；再让假服务分别返回 JSON-RPC error 和 `{ content: [], isError: true }`。前者使客户端 Promise 以错误结束等待，后者仍返回 `CallToolResult`，直到 coding-agent 适配层才转换成模型侧错误结果。

脚本练习可注册一个本地假 `echo` 工具，并发调用两次后用 `text()` 汇总；接着让脚本抛错，检查调用次数仍为二。这样可以观察脚本失败时，工具副作用仍可能已经发生。

已有仓库依赖时，在 `packages/mcp` 可单独运行离线客户端测试：

```powershell
node ../../node_modules/vitest/dist/cli.js --run test/client.test.ts test/content.test.ts test/stdio.test.ts
```

[第 10 章 权限、安全与凭据](10-permissions-secrets.md)说明工具调用管线中的授权处理；[第 6 章 重试、超时与幂等](06-retry-timeout-idempotency.md)继续解释请求失败时操作仍可能已经发生。

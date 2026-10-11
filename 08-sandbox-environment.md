# 08. 工具换到远端以后，沙箱边界在哪里

[返回索引](README.md) · [下一章：工具与 MCP](09-tools-mcp.md)

本章按提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc` 阅读源码。

## 命令在哪里执行，能碰到什么

执行环境负责实际读取文件、写文件和运行命令。例如模型请求 `git status`，程序要决定是在当前电脑的项目目录运行，还是发送到另一台机器。环境接口让工具通过同样的操作入口使用不同后端。

Sandbox（沙箱）用于限制执行代码的访问范围，例如它能读哪些文件、能连接哪些网络。选择执行位置与设置限制是两项工作。把目录设成 `/work/project` 只说明相对路径从哪里开始计算；如果进程权限允许，它仍可能读取目录外的文件。要限制访问，需要具体的系统隔离机制。

后端可以理解为“真正实现这项操作的代码和环境”。本章先完整跟踪远端命令：工具取得后端，后端发送请求，远端执行，结果返回。流程清楚后，再分别看目录规则、连接身份、命令生命周期和沙箱部署；这些细节各自处理路径含义、传输可信度、进程收尾与访问限制。

宿主是运行 Pi 的主机和进程。它可以向模型服务发起请求、保存 Durable 状态并安排工具调用；模型服务不一定运行在宿主上。远端 daemon 是另一台机器上负责执行文件和命令操作的 Rust 进程。Pi 将文件和命令操作放进 `ExecutionEnv`（执行环境）接口，让工具通过 `env.exec()` 或 `env.readTextFile()` 请求后端完成操作。接口决定系统调用在哪里发生；访问限制则由实际运行该调用的操作系统或隔离后端决定。

先区分四个参与者：

| 对象 | 负责什么 | 与其他对象的关系 |
| --- | --- | --- |
| Agent 宿主 | 运行 Pi、Durable 存储和 Agent 会话，并向模型服务发起请求 | 创建工具调用，并为调用提供当前工作目录和环境工厂 |
| 工具 | 把任务转成文件或命令操作 | 通过 `requireEnv()` 取得环境，不直接选择 Node 或 SSH 系统调用 |
| 执行后端 | 实现文件与命令接口 | 可以在本机由 `NodeExecutionEnv` 执行，也可以由 `RemoteExecutionEnv` 经 SSH 请求远端 daemon（守护进程） |
| 系统隔离层 | 限制进程能访问的文件、进程、网络或资源 | 由 OS 沙箱、容器或 VM（虚拟机）等实际部署实施；环境接口本身不提供这些限制 |

贯穿本章的请求是让工具在远端项目执行 `git status`。宿主从当前 Agent 文档读取 cwd，环境工厂返回远端后端；工具调用 `env.exec()` 后，远端后端把命令放进协议帧，通过 SSH stdin/stdout 送到 Rust daemon。daemon 在远端账号的权限下执行命令并把结果发回。宿主仍管理模型和 Durable 状态，但这不意味着命令受到宿主本地的文件或网络限制。

完整请求可以按这个顺序读：

1. Agent 宿主从当前文档取得 `/home/dev/project`，`Harness.buildEnv()` 将 cwd（工作目录）交给应用环境工厂。
2. 工具通过 `requireEnv()` 取得 `RemoteExecutionEnv`，以 `env.exec("git status")` 提交命令。
3. 远端环境检查取消状态和超时，解析远端 cwd，并把命令与 cwd 编成请求。
4. `Connection` 启动或复用 SSH 连接中的 Rust daemon，为请求分配 ID，并通过 stdin/stdout 交换协议帧（封装请求或响应的消息单元）。
5. daemon 在远端账号权限下执行命令，将输出事件和最终结果按请求 ID 返回；环境将其转换成工具可处理的结果。

这条链路说明接口负责把调用交给后端，SSH 连接负责承载请求，远端账号和系统策略决定命令能访问什么。

下面两图分别标出执行位置和隔离责任：第一图关注接口路由，第二图关注实际访问边界。图中的箭头表示请求经过的组件，不表示接口自动建立了隔离。

![Durable 工具通过 ExecutionEnv 路由到本机或远端后端，宿主控制侧保持关联](figures/08-env-routing.svg)

![工具通过环境接口选择执行位置，访问和资源约束由实际后端实施](figures/08-execution-boundaries.svg)

## 工具怎样取得执行环境

工具与后端分开的用途，是让读取和命令逻辑不必同时实现本机、远端和容器访问。工具提出“读取这个路径”，环境负责完成实际系统调用。结果也需要统一表达，否则同一工具在不同平台还要解析各式错误字符串。接口中的 `Result` 表示成功值或一种明确错误，调用方按这个分支继续。

环境工厂是应用提供的创建函数。它接收会话当前的工作目录等信息，返回本次工具使用的环境。这样一个会话可以路由到本机，另一个会话可以路由到远端；是否隔离、是否允许某条路径，仍要由返回的环境和部署策略实际实施。

执行环境就是实现文件与命令操作的后端，类型名为 `ExecutionEnv`。接口声明为 `ExecutionEnv extends FileSystem, Shell`：文件操作返回 `Result<T, FileError>`，命令返回 `Result<ShellExecResult, ExecutionError>`（[packages/durable/src/env/index.ts:3-12](../../packages/durable/src/env/index.ts#L3-L12)、[packages/durable/src/env/index.ts:171-236](../../packages/durable/src/env/index.ts#L171-L236)、[packages/durable/src/env/index.ts:309-324](../../packages/durable/src/env/index.ts#L309-L324)）。调用者拿到结果后，可以分别处理 `not_found`、`permission_denied`、`timeout` 等错误分支，无须匹配各操作系统的错误字符串。错误码与错误对象定义见 [packages/durable/src/env/index.ts:35-75](../../packages/durable/src/env/index.ts#L35-L75)。

工具通过 `requireEnv()` 取得后端。应用没有配置环境时，工具会报普通错误，不会自行创建一个默认后端。每会话环境分配示例用 `conversationId` 查找 Sandbox 文档，再为当前调用返回环境。示例里的“sandbox”只是不同目录，展示了环境怎样分配；真正运行在容器内的后端仍需应用提供（[packages/durable/src/tools/env.ts:1-8](../../packages/durable/src/tools/env.ts#L1-L8)、[packages/durable/test/examples/29-sandbox-per-conversation.ts:1-14](../../packages/durable/test/examples/29-sandbox-per-conversation.ts#L1-L14)、[packages/durable/test/examples/29-sandbox-per-conversation.ts:42-54](../../packages/durable/test/examples/29-sandbox-per-conversation.ts#L42-L54)）。

运行时会调用 `Harness.buildEnv()`，从当前 `AgentDoc` 读取 cwd，再交给应用的环境工厂。环境为每次工具调用重新构造，恢复后的重跑也会使用当时的配置，不会沿用第一次调用时的 Node 实例（[packages/durable/src/harness/harness.ts:229-235](../../packages/durable/src/harness/harness.ts#L229-L235)、[packages/durable/src/harness/tool.ts:395-403](../../packages/durable/src/harness/tool.ts#L395-L403)）。

## 用本机实现看清路径和命令

本机后端能让人看清接口负责到哪里、不负责什么。

路径解析能看出这个接口负责到哪里。`NodeExecutionEnv({ cwd: "/work/project" })` 读取 `../other.txt` 时，会先得到 `/work/other.txt`，再调用 Node 文件 API。实现没有检查结果是否仍在 `/work/project`，也没有自动拒绝越界路径的策略（[packages/durable/src/env/node.ts:72-85](../../packages/durable/src/env/node.ts#L72-L85)、[packages/durable/src/env/node.ts:913-923](../../packages/durable/src/env/node.ts#L913-L923)）。

给两个本机实例设置不同的 `cwd`，它们仍然访问同一个文件系统。所有 `NodeExecutionEnv` 实例共享 `id = "node:local"`；接口用 `id` 表示相同路径是否属于同一个文件命名空间。这个值不表示会话权限，也不表示租户身份（[packages/durable/src/env/node.ts:603-617](../../packages/durable/src/env/node.ts#L603-L617)、[packages/durable/src/env/index.ts:171-179](../../packages/durable/src/env/index.ts#L171-L179)）。

命令输入的形式决定了启动方式。字符串 `"echo a | wc -c"` 经过 Shell，管道会被解析；数组 `["echo", "a | wc -c"]` 直接启动第一项程序，后面的字符串作为普通参数传入。默认环境按 `process.env → shellEnv → options.env` 合并。设置 `inheritEnv: false` 后，只留下显式传入的 `options.env`（[packages/durable/src/env/node.ts:261-273](../../packages/durable/src/env/node.ts#L261-L273)、[packages/durable/src/env/node.ts:652-680](../../packages/durable/src/env/node.ts#L652-L680)）。

命令运行后，环境还要收集输出并负责结束进程。stdout、stderr 分别解码，多字节字符即使跨了数据块也能保留；输出超过阈值时，spill（输出暂存）会把完整原始输出写入临时文件。取消或超时会终止该命令的进程树，`cleanup()` 则清理环境仍在追踪的命令。环境负责结束进程并保留输出，命令可读的文件和可连接的地址仍由运行环境的权限决定（[packages/durable/src/env/node.ts:708-736](../../packages/durable/src/env/node.ts#L708-L736)、[packages/durable/src/env/node.ts:833-936](../../packages/durable/src/env/node.ts#L833-L936)、[packages/durable/src/env/node.ts:1220-1224](../../packages/durable/src/env/node.ts#L1220-L1224)）。

## 远端请求怎样到达 daemon

SSH 是连接远端主机并在远端账号下启动程序的通道。stdin、stdout、stderr 分别是程序的标准输入、标准输出与错误输出流。这里借 SSH 启动 daemon，再把请求写入它的输入，把响应从输出读回来。它们承载的协议帧用请求 ID 区分同时等待的操作。

建立连接前还要核对主机身份。主机密钥指纹是密钥的摘要，用来确认连接目标是否与可信记录一致；known-hosts 保存已接受的主机密钥。核对 daemon 二进制 hash 则检查将运行的文件内容是否一致。这两个检查分别对应机器身份和程序字节，远端账号实际拥有什么权限仍由系统决定。

远端负责系统调用的是 daemon：一个在本次连接期间运行的 Rust 进程，读 stdin、写 stdout，并不提供 HTTP 服务。客户端与它交换协议帧（frame），每个消息带长度、类型、请求 ID、JSON 和原始字节。

发出远端命令前，应用需要确认连到的是哪台主机。`scanHostKey()` 沿实际使用的 SSH 路线取得指纹，主机拥有者通过其他可信渠道比对后，应用才调用 `acceptHostKey()`。正常连接严格检查专用 known-hosts 文件，已保存的同类型主机密钥发生变化时会拒绝自动接受。SSH 参数还关闭 agent、X11 和其他转发，以及共享连接、配置中的远端或本地命令和 `SendEnv`（[packages/env/src/ssh.ts:72-120](../../packages/env/src/ssh.ts#L72-L120)、[packages/env/src/ssh.ts:177-207](../../packages/env/src/ssh.ts#L177-L207)、[packages/env/src/ssh.ts:253-279](../../packages/env/src/ssh.ts#L253-L279)）。

随后客户端检查远端 daemon。文件按 SHA-256 命名，上传先写临时文件，校验通过后再改名；每次启动都会验证或部署它。校验回答的是“运行的字节是否与包内二进制一致”，daemon 的文件和进程权限仍由远端账号与操作系统决定（[packages/env/src/ssh.ts:369-407](../../packages/env/src/ssh.ts#L369-L407)、[packages/env/src/ssh.ts:434-451](../../packages/env/src/ssh.ts#L434-L451)、[packages/env/src/ssh.ts:480-530](../../packages/env/src/ssh.ts#L480-L530)）。

`RemoteExecutionEnv` 从 `hello` 的返回值取得操作系统、主目录和各盘符的工作目录，按远端系统的规则处理路径。调用 `exec()` 后，它先检查取消和超时，再解析 cwd，构造 `command` 或 `argv` 请求。

请求交给 `Connection` 时，连接会按需启动 daemon，发送 `hello`，然后分配请求 ID，在请求表（Map）中记住等待结果的调用。daemon 用文件线程池处理文件操作，持续时间较长的 `exec/watch` 各自使用线程。取消控制在请求开始执行前就已注册，所以紧跟请求到达的取消也能生效。输出通过输出事件返回，最后再返回结果或错误；客户端按 ID 找回等待者，并将错误转换成环境接口的结果（[packages/env/src/remote-env.ts:347-399](../../packages/env/src/remote-env.ts#L347-L399)、[packages/env/src/remote-env.ts:834-930](../../packages/env/src/remote-env.ts#L834-L930)、[packages/env/src/connection.ts:214-305](../../packages/env/src/connection.ts#L214-L305)、[packages/env/daemon/src/main.rs:389-436](../../packages/env/daemon/src/main.rs#L389-L436)）。

远端命令默认继承 daemon 所在机器的环境。客户端只发送 `shellEnv` 和本次 `env` 的覆盖项，不会把本地 `process.env` 全量复制进 SSH 请求。设置 `inheritEnv: false` 时，远端命令也不再继承 daemon 的环境（[packages/env/src/remote-env.ts:850-881](../../packages/env/src/remote-env.ts#L850-L881)、[packages/env/docs/semantics.md:45-60](../../packages/env/docs/semantics.md#L45-L60)）。

连接两端每 5 秒发送一次 ping。daemon 连续 30 秒收不到字节，或 stdin 结束，就杀掉自己启动的进程组。每个帧最多 16 MiB，大传输由客户端拆成多块。启动时输出的随机 `PI-ENV <token>` 帮助客户端跳过远端 Shell 的杂项输出，找到协议开始的位置；它只是同步标记，不提供独立的用户授权（[packages/env/src/connection.ts:12-17](../../packages/env/src/connection.ts#L12-L17)、[packages/env/src/connection.ts:322-369](../../packages/env/src/connection.ts#L322-L369)、[packages/env/daemon/src/main.rs:438-488](../../packages/env/daemon/src/main.rs#L438-L488)）。

## 隔离要落到具体后端

限制必须覆盖真正执行操作的进程。只包装 bash 时，其他文件工具仍可能在宿主运行；只隔离工具时，扩展代码仍可能留在宿主。容器或虚拟机可以提供文件与进程边界，但配置的挂载和环境变量会把一部分宿主资源带进去。可写挂载允许工具修改宿主文件，传入凭证的环境变量则允许隔离环境中的代码使用它。

CPU、内存和执行时长也需要各自的限制机制。命令超时结束的是某次执行，虚拟机内存限制约束的是相应环境，接口提供取消方法并不表示这些配额已经统一启用。阅读下列示例时，需要逐项核对哪些工具被替换、哪些资源被传入、失败时回退到哪里。

要限制工具能访问什么，需要引入 OS 沙箱、容器或 VM。路径转换不会建立这类隔离。sandbox 扩展示例在 macOS/Linux 上初始化 sandbox runtime，通过 `createSandboxedBashOps()` 中的 `wrapWithSandbox()` 包装命令，再替换 `bash` 并接管 `user_bash`。默认配置和启用状态决定它何时生效。`read/write/edit` 没有被替换，扩展被禁用、平台不支持或初始化失败时，`execute()` 仍使用本地 bash。这个可选示例允许回退，Pi 默认没有强制沙箱策略（[packages/coding-agent/examples/extensions/sandbox/index.ts:55-139](../../packages/coding-agent/examples/extensions/sandbox/index.ts#L55-L139)、[packages/coding-agent/examples/extensions/sandbox/index.ts:130-150](../../packages/coding-agent/examples/extensions/sandbox/index.ts#L130-L150)、[packages/coding-agent/examples/extensions/sandbox/index.ts:201-285](../../packages/coding-agent/examples/extensions/sandbox/index.ts#L201-L285)）。

Gondolin 示例把七个内置工具转交给 micro-VM，并接管 `user_bash`。从 `VM.create()` 的挂载配置可以看到，`/workspace` 使用 `RealFSProvider(localCwd)`，这里的文件修改会写回宿主目录。路径转换负责把宿主工作区映射到 guest。bash 的 `env` 只去掉了非字符串值，没有过滤敏感变量；文件凭证虽然留在宿主，通过宿主环境变量传入的 provider key 仍可能进入 VM（[packages/coding-agent/examples/extensions/gondolin/index.ts:67-85](../../packages/coding-agent/examples/extensions/gondolin/index.ts#L67-L85)、[packages/coding-agent/examples/extensions/gondolin/index.ts:315-363](../../packages/coding-agent/examples/extensions/gondolin/index.ts#L315-L363)、[packages/coding-agent/examples/extensions/gondolin/index.ts:379-398](../../packages/coding-agent/examples/extensions/gondolin/index.ts#L379-L398)、[packages/coding-agent/examples/extensions/gondolin/index.ts:443-520](../../packages/coding-agent/examples/extensions/gondolin/index.ts#L443-L520)）。

两种部署方式覆盖的进程也不同。容器化文档将它们分为 whole-process 和 tool-only：整进程隔离把 Pi 与扩展一起放进去，只隔离工具则只转交特定操作。Docker、Docker Sandboxes 和 OpenShell 都是文档给出的外部部署方案，仓库默认启动路径不会自动使用它们（[packages/coding-agent/docs/containerization.md:3-27](../../packages/coding-agent/docs/containerization.md#L3-L27)）。

| 层次 | 已实现 | 边界 |
| --- | --- | --- |
| Durable 环境接口 | 文件系统、Shell、取消、观察文件变化、错误结果 | 接口没有访问控制策略 |
| `NodeExecutionEnv` | 本机 Node 文件与进程操作 | `cwd` 是路径基准；进程权限来自启动者 |
| `RemoteExecutionEnv` | 通过连接调用 Rust daemon，并对齐 Node 行为 | SSH 账号与远端操作系统决定可访问范围 |
| sandbox 扩展示例 | 用 OS 沙箱包装 `bash`，同时接管用户 `!` | 不是所有工具都被沙箱化；失败时回退本地执行 |
| Gondolin 扩展示例 | 将七种内置工具与 `!` 路由到 micro-VM | 工作区可写挂载，其他扩展仍可能在宿主执行 |

## 边界与未实现

接口能提供什么已经由请求路径说明；以下各项补充默认权限和故障时的边界。

- **默认权限姿态。** Pi 默认继承启动用户和进程的权限。仓库根文档明确说明，它没有内置的文件、进程、网络或凭证访问限制系统；默认本地操作也按这个方式执行（[README.md:88-96](../../README.md#L88-L96)、[packages/coding-agent/src/core/tools/bash.ts:96-118](../../packages/coding-agent/src/core/tools/bash.ts#L96-L118)）。
- **断线后的写入结果未知。** 如果文件已写入而连接在结果返回前断开，客户端可能无法确认写入结果。等待中的请求会以错误结束，并标记 `lost`，当前写操作不会自动重放；下一次请求才启动新 daemon。文件句柄还带着打开它的 session ID，即使新 daemon 从 1 重新编号，旧句柄也不能误指向新文件。重连恢复的是通信，文件写入没有因此获得事务恢复或结果确认（[packages/env/src/connection.ts:164-199](../../packages/env/src/connection.ts#L164-L199)、[packages/env/src/connection.ts:307-320](../../packages/env/src/connection.ts#L307-L320)、[packages/env/src/remote-env.ts:130-155](../../packages/env/src/remote-env.ts#L130-L155)）。
- **统一的限制策略尚未提供。** 这些实现尚未提供统一的路径白名单、网络授权、秘密代理和租户资源配额。应用需要这类限制时，要由实际部署的后端或整进程隔离来实施，并检查全部工具是否都经过该边界。本地与 remote 的一致性测试能验证 API 语义，容器和 VM 的安全策略需要另外验证；连接恢复也没有解决文件写入的事务、自动回滚和未知结果问题。

## 源码位置与术语速查

### 文中术语

| 术语 | 在这个项目中的含义 |
| --- | --- |
| `ExecutionEnv` | 实现文件与命令操作的后端接口，声明为 `extends FileSystem, Shell` |
| `requireEnv()` | 工具取得执行环境后端的入口；应用没有配置环境时报普通错误 |
| daemon | 远端连接期间运行的 Rust 进程，读 stdin、写 stdout，执行实际的系统调用 |
| 协议帧（frame） | 客户端与 daemon 交换的消息单位，带长度、类型、请求 ID、JSON 和原始字节 |
| spill | 命令输出超过阈值时，把完整原始输出写入临时文件的机制 |
| known-hosts | 记录已接受主机密钥的专用文件；同类型主机密钥变化时拒绝自动接受 |

## 测试与练习

环境测试先检查后端是否遵守同一份接口契约。[packages/durable/test/env-node-conformance.test.ts:33-60](../../packages/durable/test/env-node-conformance.test.ts#L33-L60) 让 Node 后端复用一致性测试，包含 polling 模式；[packages/env/test/conformance.test.ts:10-30](../../packages/env/test/conformance.test.ts#L10-L30) 则在本地 pipe 上对 remote 后端检查相同契约。

连接行为有更具体的测试：[packages/env/test/remote.test.ts:59-97](../../packages/env/test/remote.test.ts#L59-L97) 覆盖即时取消、没有输出的命令超时，以及断线后拒绝旧句柄；[packages/env/test/remote.test.ts:122-137](../../packages/env/test/remote.test.ts#L122-L137) 检查大文件分块传输后的内容与顺序。

可以先做一个只涉及临时文件的练习：放好 `project/note.txt` 和旁边的 `other.txt`，设 `cwd = project`，依次读取 `note.txt` 与 `../other.txt`，记录是否成功以及解析后的路径。预期两次读取都能成功，这正好验证 cwd 只是路径基准。整个练习无须访问用户目录或真实凭证。

再用 faux provider 运行每会话环境分配示例，比较 Alice 和 Bob 的相对写入位置。去掉其中一个会话的环境映射后，观察工具怎样报错。两份目录能演示分配过程，还不足以验证多租户隔离（[packages/durable/test/examples/29-sandbox-per-conversation.ts:28-38](../../packages/durable/test/examples/29-sandbox-per-conversation.ts#L28-L38)、[packages/durable/test/examples/29-sandbox-per-conversation.ts:73-82](../../packages/durable/test/examples/29-sandbox-per-conversation.ts#L73-L82)）。

仓库依赖已经可用时，可以在 `packages/durable` 单独运行下面的测试。

```powershell
node ../../node_modules/vitest/dist/cli.js --run test/env-node-conformance.test.ts
```

remote 测试还需要已有的 daemon 二进制，缺少它时会明确失败。阅读和上述临时文件练习不要求自动安装或构建 daemon（[packages/env/test/daemon.ts:4-13](../../packages/env/test/daemon.ts#L4-L13)）。

[MCP 工具适配](09-tools-mcp.md) 继续讨论模型可以调用哪些工具；[权限与秘密](10-permissions-secrets.md) 则解释授权钩子、认证 token 与 OS 隔离各自控制的路径。

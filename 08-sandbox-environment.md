# 第 8 章　沙箱与执行环境

[返回目录](README.md) · [上一章：第 7 章 分布式状态与一致性](07-distributed-state.md) · [下一章：第 9 章 工具基础设施与 MCP](09-tools-mcp.md)

执行环境决定工具在何处运行，沙箱则通过系统机制限制代码的访问范围与资源消耗。本章先说明文件、进程和网络边界，再讨论资源配额以及环境准备、运行、终止和清理的完整流程。随后对照 Pi 的本机与远端环境接口、可选沙箱扩展和 micro-VM 示例，明确默认行为与部署方需要实施的策略。

以“修改解析器并验证”为例，coding agent 需要调用 Git 查看改动，通过 shell 启动 Python、构建工具和测试程序，并交付补丁与验证结果；测试还可能启动本地 HTTP 服务。实际执行的代码包括仓库脚本、依赖及其子进程。如果它们继承开发者权限，就可能因路径错误覆盖工作区外的文件，也可能读取凭证、连接内网或持续占用 CPU。

沙箱（sandbox）是在代码执行周围施加访问限制和资源约束的环境。调用方提供代码、输入及允许范围，执行器负责应用限制、运行代码、返回产物和证据，并在结束后回收资源。限制需要由操作系统或虚拟化机制执行；提示词中的目录要求不能阻止一次越权系统调用。

前半章采用通用教学设计，不表示 Pi 已实现其中全部策略。宿主是承载执行环境的机器，客体是虚拟机内部运行的系统。后半章源码说明以提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc` 为基线，分别讨论默认本机操作、实验环境接口和可选隔离部署。

## 8.1 文件、进程与网络的访问边界

### 8.1.1 文件系统：输入、工作副本与输出

执行器需要分别安排输入、工作副本与产物。挂载将一个目录呈现在环境内的指定路径；固定源文件和依赖可以只读挂载，可写工作副本供 Agent 修改，临时目录和产物目录供编译器生成文件。读取权限与写入权限应分别配置。只读挂载可以防止修改，仍允许进程读取文件中的秘密。

工作副本可以直接映射宿主项目，也可以独立复制。采用宿主可写挂载时，工具保存文件就已经修改宿主，销毁沙箱不会回滚这些变化。独立工作副本则允许先收集差异与产物，核对目标路径和基准版本后再写回，代价是增加复制或同步。产物收集器仍需检查补丁目标，不能因其来自沙箱就允许覆盖任意宿主路径。

`cwd`（当前工作目录）决定相对路径的解析起点。例如在 `/work/project` 访问 `../other.txt`，解析结果为 `/work/other.txt`，实际能否读取由系统权限决定。Git worktree 分离工作文件，有助于减少任务相互覆盖，但进程仍可访问其他目录及共享 Git 管理数据。路径检查还需考虑符号链接等实际解析结果。要限制程序绕过工具后的访问，仍需系统强制执行文件可见范围和读写权限。

### 8.1.2 进程：脚本及其后代的限制

进程限制需要覆盖任务及其后代。构建命令启动编译器，测试再启动服务，只限制最外层 shell 会遗漏这些执行路径。执行器应将它们纳入同一受控范围，使用低权限身份，并限制其他任务进程、宿主设备和管理接口的访问；子进程也应继承这些约束。否则程序可能通过更高权限的服务间接操作原本不可访问的资源。

Linux 命名空间（namespace）为进程提供不同的挂载、进程编号或网络视图，容器因此可以显示不同的目录和进程列表。视图隔离还需结合用户权限、能力限制等机制。容器通常共享宿主内核，虚拟机运行独立客体内核，信任边界有所不同；但虚拟机仍可通过共享目录或传入的管理接口访问宿主资源，具体权限必须逐项检查。

隔离部署还需要确定覆盖哪些代码。整进程隔离将 Agent、扩展与工具一同放入环境，同时安排模型连接、持久数据和凭证的访问。工具隔离保留宿主上的 Agent，仅将选定操作交给沙箱，集成较方便；未接管的文件工具与扩展仍保有宿主权限。覆盖检查应沿全部实际执行路径进行，仅包装 bash 不足以证明所有操作受限。

### 8.1.3 网络：出站访问与测试服务

依赖下载可以安排在受控准备阶段，执行阶段随后关闭外部网络；也可以在运行期间允许指定出口访问依赖源。预先准备便于复用固定输入，运行时联网则适合必须访问外部资源的任务，但需要限制目的地址、协议及凭证。策略应由网络配置或代理执行，单独禁用下载工具无法限制 Python、测试代码等其他客户端。

网络允许列表还要考虑宿主地址、内网服务和云环境的元数据服务。即使程序只打算下载公开包，给它完整宿主网络仍可能暴露这些额外目标。网络命名空间提供不同的网络视图，具体允许哪些流量仍取决于配置的接口、路由与过滤规则。相关 Linux 机制见 [namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html)。

本地 HTTP 测试需要环境内部的回环通信，使测试客户端能够访问同一环境中的服务。用户从外部预览时，还需显式转发或发布端口，并确定允许访问的调用方。内部服务启动并不自动使宿主可连接；发布预览端口也不要求开放任意出站流量。任务结束后，服务进程与端口映射都应回收。

## 8.2 资源配额与执行时限

权限正确的脚本仍可能耗尽资源。解析器测试陷入循环时，CPU 配额可以限制它占用的计算份额，却未必使其结束，执行时限负责终止持续过久的工作。测试不断积累输入时，则可能在低 CPU 使用下耗尽内存。各项额度需要独立设置，并保存触发终止的具体原因。

| 限制 | 约束什么 | 对任务的影响 |
| --- | --- | --- |
| CPU 配额 | 一个环境及其进程可使用的计算份额或预算 | 并行编译可能变慢；限速本身通常不结束命令 |
| 内存上限 | 被计入同一环境的内存使用 | 超额可能使分配失败或触发进程终止，需要保留具体原因 |
| 执行时长 | 命令或整个任务允许经过的实际时间 | 到期由监督者取消执行，并清理仍存活的子进程 |

这些限制需要明确计量范围。给每个编译器单独限制内存，无法直接约束十个编译器的总和；逐条命令都满足时限，也不能保证整个任务按时结束。进程数量、磁盘和日志空间同样会被消耗，可按工作负载补充上限。Linux 的 cgroup 将进程组织在层级中，由不同控制器管理资源，适合对一个任务及其后代统一计量与约束；它与文件访问授权是不同职责。具体语义见 [Linux cgroup v2 文档](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html)。

访问权限控制程序能够操作什么，资源配额控制它可以消耗多少。低内存额度不会阻止读取已挂载的凭证，关闭网络也不会阻止覆盖可写目录。执行器应分别配置和核验这两类约束。

![文件、进程和网络限制访问边界，CPU、内存和时长限制资源消耗](figures/08-concept-boundaries.svg)

图中的访问控制与资源配额分别限制可访问对象和可消耗资源，需要独立确认是否生效。

## 8.3 执行环境的生命周期

环境监督者负责创建、监控和回收执行环境，应位于任务代码无法任意修改的控制范围内。它记录任务标识、输入版本、环境标识、权限配置、截止时间和最终状态。以下以解析器任务说明各阶段及其完成条件。

1. 监督者选择具有所需 shell、Git、Python 和构建工具的基础环境，建立工作副本并记录输入版本。复用环境可以降低启动成本，但必须检查前次任务是否遗留文件、后台服务或秘密。
2. 监督者放入输入，配置只读和可写挂载、执行身份、网络及资源额度，仅传入本次操作需要的环境变量和凭证。确认配置生效后再启动任务；必需隔离无法建立时，以准备失败结束。
3. Agent 经受控工具修改代码、构建并测试。监督者收集 stdout（标准输出）、stderr（标准错误输出）、退出状态与资源使用，以区分测试失败、权限拒绝、资源超限和仍在运行。大量输出可保留摘要并另存完整日志，同时限制日志存储。
4. 正常完成后进入结果收集；取消或超时后先请求停止，在有限宽限期后强制终止仍存活的任务进程及其后代，包括编译器和后台测试服务。调用方停止等待不构成进程终止证据。
5. 确认进程退出后，取回允许的补丁、报告和产物，保存结束原因。失败任务的日志与部分改动仍可保留，但不能标为验证通过。独立工作副本的修改需要核对后再写回目标版本。
6. 监督者移除端口映射、临时挂载与工作空间，撤回临时凭证，销毁环境或完成复用前清理。清理失败时记录残留资源并继续回收，不能将仍有任务进程的环境分配给下一项工作。

![环境依次经历准备、运行、终止、取回产物和清理](figures/08-concept-lifecycle.svg)

宿主可写挂载上的文件在运行时已经写入宿主，销毁环境不会回滚这些更改。

```mermaid
flowchart LR
    A[准备基础环境与输入版本] --> B[放入输入并施加权限和配额]
    B --> C[执行工具并监控]
    C --> D{完成或需终止}
    D -->|正常完成| F[取回产物与执行证据]
    D -->|取消或超时| E[终止任务进程及其后代]
    E --> F
    F --> G[核对写回并清理环境]
```

执行器需要确认终止与清理真正完成。若执行器也随任务崩溃，另一个管理进程应依据环境记录与期限回收资源。为调查失败而暂时保留容器或 VM，会延长资源占用和秘密暴露时间，因此还需限定保留期限与访问者。

## 8.4 隔离故障、写入冲突与结果未知

初始化失败后的回退会改变实际权限。允许回退到宿主 shell 的交互式工具，应明确报告所用后端；要求强制隔离的任务则应停止启动。本机执行成功只能作为本机执行证据，不能记为沙箱验证成功。

隔离不完整也可能来自未覆盖的执行入口。例如 bash 已受限制，文件工具却仍直接写宿主目录。这与利用内核或虚拟化漏洞突破边界的沙箱逃逸不同。排查时应检查工具路由、挂载、凭证和管理接口，再调查隔离机制自身。保护范围仅限实际覆盖并实施限制的资源，也不能据此排除所有漏洞。

环境隔离与共享写入协调是不同要求。两个容器同时可写挂载同一源码时，任务甲读取旧文件，任务乙写入后，甲再保存旧基础上的修改，仍会覆盖乙。独立工作副本可以分离执行期间的改动；写回时还需比较基准版本、合并差异或串行提交，使用 worktree 时也要协调共享 Git 状态。

还有一种失败发生在命令结束之后：环境已经写出产物，结果通知却丢失。控制侧此时只能记录结果未知，并按任务标识查询环境或检查产物；直接重跑可能重复外部操作。取消任务和销毁环境都不会自动撤销已经发出的网络请求或已经落到宿主可写挂载中的更改。[第 6 章 重试、超时与幂等](06-retry-timeout-idempotency.md) 讨论这类结果未知时如何决定下一步。

## 8.5 参考实现：Pi 的环境创建与执行路由

Pi 的环境接口统一表达文件与命令操作，具体隔离限制由后端实施。以下先追踪实验 durable harness 的本机工具调用，再比较远端路由与可选扩展的覆盖范围。

假设会话工作目录为 `/work/project`，模型请求运行 `git status`。工具接收命令和工作目录，经过环境后端执行，最终返回输出、退出状态或错误。调用顺序如下。

1. 工具任务先调用 `runtime.env(context)`，经 `Harness.buildEnv()` 从当前 `AgentDoc` 读取会话的 `cwd`，再交给应用配置的环境工厂。本例工厂返回本机 `NodeExecutionEnv`。
2. 工具任务将创建好的环境注入工具 API，再调用工具。工具里的 `requireEnv()` 只读取已经注入的环境；没有配置时，它会报普通错误，不会创建默认环境。
3. `createBashTool` 从参数中取出命令和可选超时，把环境的工作目录作为 cwd 传给 `env.exec()`。cwd（当前工作目录）是相对路径的起点。环境变量是传给程序的键值配置；实际传入哪些值取决于环境的合并选项。
4. 本机后端在该目录启动命令并收集 stdout 和 stderr。stdout 是程序的正常输出流，stderr 是错误输出流。命令完成后，后端把输出、退出状态或错误交给工具；工具再把结果写入会话，供后续模型请求读取。

在这个本机例子中，宿主是启动 durable harness 的机器，命令使用该进程的权限。Node 环境在收到取消或达到已设置的超时时，会终止对应进程树。内置 bash 工具没有默认超时，只有调用方提供时限后才据此终止命令。

稳定版 coding-agent 的内置 bash 默认使用本机 `BashOperations`，不经过实验 harness 的 `ExecutionEnv`。扩展可以替换 `BashOperations`，将命令转交远端或沙箱。两套接口表达相似需求，但后端接入位置和默认路径不同。

### 8.5.1 同一命令的远端执行

SSH 提供连接远端并以远端账号启动程序的通道。项目位于远端时，应用可显式配置环境工厂返回 `RemoteExecutionEnv`。工具仍提交 `git status`，后端经 SSH 建立连接，以远端账号执行，并将结果返回会话。daemon 是连接期间在远端执行文件和命令操作的进程。稳定 CLI 默认本机 bash 不会自动采用这一路由。

远端后端向 daemon 发送 cwd、命令与环境覆盖项，daemon 按远端账号和操作系统权限运行。连接为每次请求分配 ID，将并发操作的输出及最终结果关联回原调用。取得环境、解析参数、执行和返回的逻辑次序保持一致，实际文件与进程操作发生在远端。

![Durable 工具通过 ExecutionEnv 路由到本机或远端后端，宿主控制侧保持关联](figures/08-env-routing.svg)

图示表示执行位置和结果关联，不表示文件、网络或资源策略已经生效。主机密钥校验、daemon 部署、协议帧及断线状态在 8.7 节说明。

## 8.6 参考实现：可选隔离部署的覆盖范围

可选 sandbox 与 Gondolin 示例分别使用操作系统沙箱和 micro-VM。核对部署时，需要检查工具覆盖、宿主资源映射及初始化失败处理，同时分别检查 CPU、内存和执行时限。环境接口具有取消方法，不等于这些配额已经配置。

### 8.6.1 sandbox 扩展的启用与回退

仓库中的 sandbox 扩展示例在 macOS/Linux 上初始化 sandbox runtime，再通过 `createSandboxedBashOps()` 和 `wrapWithSandbox()` 包装命令。它替换 `bash`，也为用户输入的 `!` 提供沙箱操作。示例默认配置启用沙箱；启动时读取全局和项目配置，并检查命令行的 `--no-sandbox` 标志。平台不支持、配置关闭或初始化失败时，示例会保留本机 bash 操作。`read/write/edit` 没有被这个扩展示例替换，因此不能据此认为所有文件操作都经过沙箱。它是可选扩展，Pi 默认没有强制沙箱策略（[packages/coding-agent/examples/extensions/sandbox/index.ts:55-139](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/sandbox/index.ts#L55-L139)、[packages/coding-agent/examples/extensions/sandbox/index.ts:130-150](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/sandbox/index.ts#L130-L150)、[packages/coding-agent/examples/extensions/sandbox/index.ts:201-285](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/sandbox/index.ts#L201-L285)）。

### 8.6.2 Gondolin 的虚拟机映射

Gondolin 将七个内置工具和用户输入的 `!` 转交 micro-VM。初始化调用 `VM.create()`，通过 `RealFSProvider(localCwd)` 将宿主工作区映射到 `/workspace`。该挂载可写，因此虚拟机内的文件修改会影响宿主工作区；工具路径转换负责定位映射后的文件。

bash 的 `env` 仅移除非字符串值，没有过滤敏感变量。文件凭证仍放在宿主文件系统，但通过宿主环境变量传入的 provider key 可能进入 VM。审查这类配置时，需要同时看工具覆盖了哪些操作、哪些宿主目录被挂载，以及哪些环境变量被传入（[packages/coding-agent/examples/extensions/gondolin/index.ts:67-85](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/gondolin/index.ts#L67-L85)、[packages/coding-agent/examples/extensions/gondolin/index.ts:315-363](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/gondolin/index.ts#L315-L363)、[packages/coding-agent/examples/extensions/gondolin/index.ts:379-398](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/gondolin/index.ts#L379-L398)、[packages/coding-agent/examples/extensions/gondolin/index.ts:443-520](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/gondolin/index.ts#L443-L520)）。

容器化文档还区分 whole-process 与 tool-only。整进程部署把 Pi 和扩展一起放进容器；只隔离工具则只转交选定的操作。Docker、Docker Sandboxes 和 OpenShell 是文档列出的外部部署方案，仓库默认启动路径不会自动启用它们（[packages/coding-agent/docs/containerization.md:3-27](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/docs/containerization.md#L3-L27)）。

| 执行路径 | 操作在哪里执行 | 需要检查的范围 |
| --- | --- | --- |
| Durable 的 `ExecutionEnv` | 由应用配置的后端 | 接口本身没有访问控制策略 |
| `NodeExecutionEnv` | 本机 Node 文件与进程操作 | cwd 是路径基准；权限来自启动进程的用户 |
| `RemoteExecutionEnv` | 远端 Rust daemon | SSH 账号和远端操作系统决定可访问范围 |
| `LocalExecutionEnv` | 本机 Node 操作；`listFiles` 经本机 pi-env daemon | 其余操作仍是 `NodeExecutionEnv`；daemon 连接由调用方关闭 |
| sandbox 扩展示例 | OS 沙箱中的 bash，以及用户 `!` | 其他文件工具未被替换；关闭或初始化失败时回退本机 bash |
| Gondolin 扩展示例 | micro-VM 中的七个内置工具和用户 `!` | 工作区可写挂载，未接管的扩展仍可能在宿主执行 |

![工具通过环境接口选择执行位置，访问和资源约束由实际后端实施](figures/08-execution-boundaries.svg)

### 8.6.3 默认权限与未实现的策略

Pi 默认继承启动用户和进程的权限。仓库根文档说明，Pi 没有内置的文件、进程、网络或凭证访问限制系统；默认本地操作也按这个方式执行（[README.md:88-96](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/README.md#L88-L96)、[packages/coding-agent/src/core/tools/bash.ts:96-118](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/tools/bash.ts#L96-L118)）。

远端连接可能在文件已写入、结果尚未返回时断开。客户端此时无法确认写入结果。等待中的请求会以错误结束并标记为 `lost`，当前写操作不会自动重放；下一次请求才会启动新 daemon。文件句柄带有打开它的 session ID，因此旧句柄不会误指向新 daemon 重新编号的句柄。重连恢复通信，但不提供事务恢复、自动回滚或写入结果确认（[packages/env/src/connection.ts:164-199](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/connection.ts#L164-L199)、[packages/env/src/connection.ts:307-320](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/connection.ts#L307-L320)、[packages/env/src/remote-env.ts:130-155](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/remote-env.ts#L130-L155)）。

这些实现尚未提供统一的路径白名单、网络授权、秘密代理或租户配额。需要相关限制的应用，应在实际后端或整进程隔离部署中实施，并确认工具全部经过所需边界。本机与 remote 的一致性测试验证 API 行为，不能替代容器或 VM 的安全策略验证。

## 8.7 参考实现：环境接口与远端连接协议

### 8.7.1 环境结果与本机命令输出

Durable 环境把可预期的失败作为结果返回，形式是 `{ ok: true, value }` 或 `{ ok: false, error }`。文件操作和命令操作使用各自的错误类型；工具将成功值或错误转成会话可保存的工具结果。这个 `Result` 类型表达一次操作的结果，不建立文件或进程访问规则（[packages/durable/src/env/index.ts:4-4](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/index.ts#L4-L4)）。

`ExecutionEnv extends FileSystem, Shell` 的声明把文件系统、命令、取消及文件变化观察等操作放在统一接口中。文件操作返回 `Result<T, FileError>`，命令返回 `Result<ShellExecResult, ExecutionError>`；调用方可以按 `not_found`、`permission_denied`、`timeout` 等分支处理，无须匹配平台错误字符串。接口和错误定义见 [packages/durable/src/env/index.ts:3-12](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/index.ts#L3-L12)、[packages/durable/src/env/index.ts:171-236](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/index.ts#L171-L236)、[packages/durable/src/env/index.ts:309-324](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/index.ts#L309-L324)、[packages/durable/src/env/index.ts:35-75](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/index.ts#L35-L75)。

环境创建与读取的顺序可对照 [packages/durable/src/harness/harness.ts:229-235](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/harness.ts#L229-L235)、[packages/durable/src/harness/tool.ts:395-403](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/tool.ts#L395-L403) 和 [packages/durable/src/tools/env.ts:1-8](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/tools/env.ts#L1-L8)。环境为每次工具调用重新构造，恢复后的重跑使用当时的配置，不会沿用第一次调用的 Node 实例。每会话示例按 `conversationId` 查找 Sandbox 文档后返回环境；示例只分配不同目录，容器后端仍需应用提供（[packages/durable/test/examples/29-sandbox-per-conversation.ts:1-14](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/test/examples/29-sandbox-per-conversation.ts#L1-L14)、[packages/durable/test/examples/29-sandbox-per-conversation.ts:42-54](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/test/examples/29-sandbox-per-conversation.ts#L42-L54)）。

`NodeExecutionEnv({ cwd: "/work/project" })` 将 `../other.txt` 解析为 `/work/other.txt`，随后调用 Node 文件 API，没有自动拒绝越出 cwd 的路径（[packages/durable/src/env/node.ts:72-85](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L72-L85)、[packages/durable/src/env/node.ts:913-923](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L913-L923)）。不同 cwd 的实例仍共享文件系统，所有实例的 `id = "node:local"` 相同。接口用 `id` 表示相同路径是否属于同一个文件命名空间，它不代表会话权限或租户身份（[packages/durable/src/env/node.ts:603-617](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L603-L617)、[packages/durable/src/env/index.ts:171-179](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/index.ts#L171-L179)）。

字符串命令 `"echo a | wc -c"` 经过 shell 解析，数组 `["echo", "a | wc -c"]` 则直接启动第一项程序，其余项作为普通参数。默认环境按 `process.env → shellEnv → options.env` 合并，后者覆盖前者；`inheritEnv: false` 只保留显式传入的 `options.env`（[packages/durable/src/env/node.ts:261-273](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L261-L273)、[packages/durable/src/env/node.ts:652-680](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L652-L680)）。

stdout 和 stderr 分别解码，因此多字节字符跨数据块时仍能保留。取消或超时会终止命令的进程树，`cleanup()` 清理环境仍在追踪的命令（[packages/durable/src/env/node.ts:708-736](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L708-L736)、[packages/durable/src/env/node.ts:833-936](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L833-L936)、[packages/durable/src/env/node.ts:1220-1224](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L1220-L1224)）。

输出超过保留阈值时，工具将完整原始内容写入临时文件，即 spill；返回结果保留供模型阅读的输出窗口，并在诊断中给出完整路径。stdout 与 stderr 都进入工具输出。退出码非零时，结果仍保存输出及退出码，同时标记错误。

### 8.7.2 远端连接、SSH 身份与 daemon

`RemoteExecutionEnv` 先从 `hello` 取得远端操作系统、主目录和各盘符的工作目录，再按远端规则解析路径。`exec()` 检查取消与超时，然后构造 `command` 或 `argv` 请求。连接为每次请求分配 ID，再按 ID 把结果和错误交还调用者。daemon 为文件操作使用 16 个 worker 线程；较长的 `exec/watch` 各自使用线程。取消会在操作开始前登记，因此紧随请求到达的取消也能生效（[packages/env/src/remote-env.ts:347-399](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/remote-env.ts#L347-L399)、[packages/env/src/remote-env.ts:834-930](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/remote-env.ts#L834-L930)、[packages/env/src/connection.ts:214-305](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/connection.ts#L214-L305)、[packages/env/daemon/src/main.rs:389-436](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/daemon/src/main.rs#L389-L436)）。

SSH 主机密钥指纹由 `scanHostKey()` 沿实际 SSH 路线取得。主机拥有者通过其他可信渠道比对后，应用才调用 `acceptHostKey()`。正常连接严格检查专用 known-hosts 文件；已保存的同类型主机密钥发生变化时会拒绝自动接受。SSH 参数还关闭 agent、X11 和其他转发、共享连接、配置中的远端或本地命令以及 `SendEnv`（[packages/env/src/ssh.ts:72-120](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/ssh.ts#L72-L120)、[packages/env/src/ssh.ts:177-207](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/ssh.ts#L177-L207)、[packages/env/src/ssh.ts:253-279](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/ssh.ts#L253-L279)）。

客户端检查远端 daemon 二进制。文件按 SHA-256 命名，上传先写临时文件，校验通过后再改名；每次启动都会验证或部署它。校验确认运行字节与包内二进制一致，不会扩大 daemon 的系统权限（[packages/env/src/ssh.ts:369-407](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/ssh.ts#L369-L407)、[packages/env/src/ssh.ts:434-451](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/ssh.ts#L434-L451)、[packages/env/src/ssh.ts:480-530](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/ssh.ts#L480-L530)）。

远端命令默认继承 daemon 所在机器的环境。客户端发送 `shellEnv` 和本次 `env` 覆盖项，不会把本地 `process.env` 全量复制进 SSH 请求；设 `inheritEnv: false` 后，命令也不再继承 daemon 的环境（[packages/env/src/remote-env.ts:850-881](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/remote-env.ts#L850-L881)、[packages/env/docs/semantics.md:45-60](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/docs/semantics.md#L45-L60)）。

SSH 连接两端每 5 秒发送一次 ping。daemon 连续 30 秒收不到字节或 stdin 结束时，会终止自己启动的进程组。每个协议帧最多 16 MiB，大传输由客户端拆块。启动时输出的随机 `PI-ENV <token>` 让客户端跳过远端 Shell 的杂项输出，找到协议起点；它是同步标记，不提供用户授权。stdin、stdout、stderr 是程序的标准输入、标准输出和错误输出流。daemon 在连接期间读取 stdin、写入 stdout，不提供 HTTP 服务。协议帧是客户端与 daemon 交换的消息单位，包含长度、类型、请求 ID、JSON 和原始字节（[packages/env/src/connection.ts:12-17](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/connection.ts#L12-L17)、[packages/env/src/connection.ts:322-369](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/connection.ts#L322-L369)、[packages/env/daemon/src/main.rs:438-488](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/daemon/src/main.rs#L438-L488)）。

### 8.7.3 交互式终端、文件列举与平台名映射

pi-env 还直接暴露两类连接能力，供扩展与应用使用；本章的内置工具调用尚未使用它们。

伪终端（PTY）用于运行需要交互的程序。`openTerminal(connection, options)` 请求 daemon 的 `pty` 操作并返回 `RemoteTerminal`，选项含 cwd、argv、env、cols、rows 与 `onData` 回调（[packages/env/src/terminal.ts:6-18](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/terminal.ts#L6-L18)）。终端输出以原始字节事件按序到达 `onData`；`write()` 按到达顺序发送输入，超过 1 MiB 的输入被切成多次写入；`resize()` 修改尺寸后程序收到 `SIGWINCH`；`close()` 或连接丢失都会终止程序的整个进程组（[packages/env/src/terminal.ts:55-89](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/terminal.ts#L55-L89)、[packages/env/src/terminal.ts:92-127](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/terminal.ts#L92-L127)）。`foreground()` 在终端前台不是程序本身时报告前台的进程组，例如 shell 正在运行的编辑器，macOS 与 Linux 下附带 leader 名称（[packages/env/src/terminal.ts:20-24](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/terminal.ts#L20-L24)、[packages/env/src/terminal.ts:73-82](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/terminal.ts#L73-L82)）。daemon 的 pty 操作仅支持 Unix，Windows 上回答 `ENOTSUP`；`localDaemon()` 返回本机对应的 daemon 二进制路径，可为本地会话启动终端（[packages/env/src/ssh.ts:361-368](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/ssh.ts#L361-L368)）。

文件列举是环境接口上的可选操作 `listFiles`：返回目录树下深度优先的条目（文件、目录、未跟随的符号链接），在 `maxEntries`、`maxBytes`、`timeoutMs` 等边界处提前结束并置 `truncated`；默认跳过 `.git` 及 git 忽略规则命中的路径，包括已跟踪文件（[packages/durable/src/env/index.ts:170-206](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/env/index.ts#L170-L206)、[packages/durable/src/env/index.ts:245-255](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/env/index.ts#L245-L255)）。`RemoteExecutionEnv.listFiles` 经 daemon 执行（[packages/env/src/remote-env.ts:723-736](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/remote-env.ts#L723-L736)）；对端 daemon 若不支持，`hello` 的 `features` 不含 `listFiles`，调用立即以 `not_supported` 失败，调用方回退到自己的遍历（[packages/env/src/list-files.ts:20-41](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/list-files.ts#L20-L41)、[packages/env/src/connection.ts:36-53](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/connection.ts#L36-L53)）。daemon 用 `FEATURES` 数组声明能力，忽略语义由 git ignore 匹配实现（[packages/env/daemon/src/main.rs:34](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/daemon/src/main.rs#L34)、[packages/env/daemon/src/list.rs:100-103](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/daemon/src/list.rs#L100-L103)）。`LocalExecutionEnv` 组合 `NodeExecutionEnv` 与本机 daemon 连接：只有 `listFiles` 走 daemon，其余仍是 Node 操作，使本机与远端共用同一份列举代码（[packages/env/src/list-files.ts:74-102](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/list-files.ts#L74-L102)）。

平台名映射：daemon 用 Rust 的拼写报告系统与架构，`nodePlatform(info)` 把它们映射为 Node 的名称（`windows`→`win32`、`macos`→`darwin`、`x86_64`→`x64`、`aarch64`→`arm64`，其余保留），便于与 `process.platform` 和 `process.arch` 比较（[packages/env/src/connection.ts:55-64](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/connection.ts#L55-L64)）。

## 8.8 源码定位

| 机制 | 源码入口 |
| --- | --- |
| 执行环境接口 | [packages/durable/src/env/index.ts:171-236](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/index.ts#L171-L236) |
| 环境工厂与注入 | [packages/durable/src/harness/harness.ts:229-235](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/harness.ts#L229-L235) |
| 本机命令与环境变量 | [packages/durable/src/env/node.ts:652-680](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/env/node.ts#L652-L680) |
| 远端命令请求 | [packages/env/src/remote-env.ts:834-930](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/remote-env.ts#L834-L930) |
| 断线与待处理请求 | [packages/env/src/connection.ts:164-199](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/connection.ts#L164-L199) |
| SSH 主机身份校验 | [packages/env/src/ssh.ts:72-120](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/src/ssh.ts#L72-L120) |
| daemon 活性检测 | [packages/env/daemon/src/main.rs:438-488](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/daemon/src/main.rs#L438-L488) |
| 交互终端（PTY） | [packages/env/src/terminal.ts:92-127](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/terminal.ts#L92-L127) |
| 文件列举 listFiles | [packages/env/src/list-files.ts:20-72](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/list-files.ts#L20-L72) |
| 平台名映射 | [packages/env/src/connection.ts:55-64](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/env/src/connection.ts#L55-L64) |
| 可选 sandbox 扩展 | [packages/coding-agent/examples/extensions/sandbox/index.ts:55-139](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/examples/extensions/sandbox/index.ts#L55-L139) |

## 8.9 本章术语

| 通用概念 | 定义 |
| --- | --- |
| 沙箱 | 由系统机制实施访问限制与资源约束的执行环境 |
| 宿主与客体 | 承载环境的机器，以及虚拟机内部的系统 |
| 挂载 | 将目录呈现在环境内的指定路径，可分别配置读写权限 |
| 工作副本 | 任务执行时使用的工作文件，其写回需遵守目标路径与版本约束 |
| 资源配额 | 对任务及其后代消耗的 CPU、内存等资源设定限制 |

下表列出 Pi 接口与远端连接中的术语。

| 术语 | 在这个项目中的含义 |
| --- | --- |
| `ExecutionEnv` | 实现文件与命令操作的后端接口，声明为 `extends FileSystem, Shell` |
| `requireEnv()` | 工具读取执行环境的入口；应用没有配置环境时会报普通错误 |
| daemon | 远端连接期间运行的 Rust 进程，读 stdin、写 stdout，执行实际系统调用 |
| 协议帧（frame） | 客户端与 daemon 交换的消息单位，带长度、类型、请求 ID、JSON 和原始字节 |
| spill | 命令输出超过阈值时，将完整原始输出写入临时文件的机制 |
| `listFiles` | 可选的文件树列举操作；daemon 不支持时以 `not_supported` 失败，调用方自行遍历 |
| 伪终端（PTY） | 在 daemon 上运行的交互终端，按原始字节流收发事件 |
| known-hosts | 记录已接受主机密钥的专用文件；同类型主机密钥变化时拒绝自动接受 |

## 8.10 练习

环境测试先检查后端是否遵守同一份接口契约。[packages/durable/test/env-node-conformance.test.ts:33-60](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/test/env-node-conformance.test.ts#L33-L60) 让 Node 后端复用一致性测试，包含 polling 模式；[packages/env/test/conformance.test.ts:10-30](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/test/conformance.test.ts#L10-L30) 则在本地 pipe 上对 remote 后端检查相同契约。

连接行为有更具体的测试：[packages/env/test/remote.test.ts:59-97](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/test/remote.test.ts#L59-L97) 覆盖即时取消、没有输出的命令超时，以及断线后拒绝旧句柄；[packages/env/test/remote.test.ts:122-137](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/test/remote.test.ts#L122-L137) 检查大文件分块传输后的内容与顺序。

可以先做一个只涉及临时文件的练习：放好 `project/note.txt` 和旁边的 `other.txt`，设 `cwd = project`，依次读取 `note.txt` 与 `../other.txt`，记录是否成功以及解析后的路径。预期两次读取都能成功，这说明 cwd 只是路径基准。整个练习无须访问用户目录或真实凭证。

再用 faux provider 运行每会话环境分配示例，比较 Alice 和 Bob 的相对写入位置。去掉其中一个会话的环境映射后，观察工具怎样报错。两份目录能演示分配过程，还不足以验证多租户隔离（[packages/durable/test/examples/29-sandbox-per-conversation.ts:28-38](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/test/examples/29-sandbox-per-conversation.ts#L28-L38)、[packages/durable/test/examples/29-sandbox-per-conversation.ts:73-82](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/test/examples/29-sandbox-per-conversation.ts#L73-L82)）。

仓库依赖已经可用时，可以在 `packages/durable` 单独运行下面的测试。

```powershell
node ../../node_modules/vitest/dist/cli.js --run test/env-node-conformance.test.ts
```

remote 测试还需要已有的 daemon 二进制，缺少它时会明确失败。阅读和上述临时文件练习不要求自动安装或构建 daemon（[packages/env/test/daemon.ts:4-13](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/env/test/daemon.ts#L4-L13)）。

[第 9 章 工具基础设施与 MCP](09-tools-mcp.md) 继续讨论模型可以调用哪些工具；[第 10 章 权限、安全与凭据](10-permissions-secrets.md) 解释授权钩子、认证 token 与 OS 隔离分别控制什么。

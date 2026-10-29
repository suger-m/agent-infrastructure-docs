# 执行环境与操作系统沙箱

[手册目录](../README.md)

适用范围：Maka 固定提交 `15d0a0dd19d39e2bc09ae35940e432c636982315` 的内置工作区工具、命令执行和文件 worker；外部执行器的隔离能力由其实现负责。
Pi 对照基于 `482c1381a00d8d00bc81697e4a980c593db6803e`；本章引用的 Pi 源码文件没有工作区差异。默认 `coding-agent/agent/ai`、实验 durable 和可选扩展示例分别讨论。
以下轨迹来自源码与测试断言的静态核对，本次没有运行平台沙箱、安装依赖或调用模型 API。Windows 结论限定为当前文件 worker 预览能力。

## 1. 问题：项目内命令对项目外的访问

假设用户要求“修改配置并验证”。模型可以调用 `Write` 保存文件，也可以调用 `Bash` 执行 `node check.js`。
即使 shell 的工作目录是 `/work/project`，脚本仍能尝试读取 `../private/token.txt`、连接网络、创建子进程。
当前工作目录 `cwd` 仅是相对路径的起点；访问限制需要在实际读取、写入或启动程序之处生效。

因此需要考察三个问题：工具被交给哪个执行器；本次调用允许触及哪些资源；由谁拦截越界访问。
Maka 将它们分别交给 `WorkspaceExecutor`、`ExecutionBoundary` 和平台沙箱后端。接口、策略与操作系统执行分属三层不同职责。
本机执行器明确报告 `isolation: 'none'`、`writesAffectHost: true`、`writeBack: 'direct'`，不能从类名中的 Workspace 推导出已实现隔离。[执行器事实][m-executor]

例如 `Write` 被策略批准后，写入仍会直接修改宿主文件。沙箱限制的是“可写入的范围”，不会在命令失败时撤销已写入的内容；工作副本和事务回滚属于另外的机制。

## 2. 边界表达意图，后端负责实施

`ExecutionBoundary` 有三种分支，每个分支都带 `revision`；`managed` 还携带具体权限 profile。
profile 描述文件读写范围及网络模式，是否选择 OS 后端由 `SandboxManager` 决定。[边界定义][m-boundary]、[后端选择][m-manager]

| 边界／配置 | 内置文件工具 | 内置 Bash 的命令转换 |
| --- | --- | --- |
| `managed` | 交给文件 worker；没有 worker 时返回 `requires_bypass` | 根据 profile 判断是否需要沙箱，所需能力不可用则失败 |
| `bypass` | 跳过 worker，以 `host` 路径范围调用工作区执行器 | 跳过 Maka 的命令沙箱转换 |
| `external` | 使用注入的执行器，保持 `workspace` 路径范围 | 跳过本地转换，由外部环境承担限制 |
| 嵌入调用未提供边界 | 存在 worker 时使用 worker，否则本地执行且将路径限制保留在 cwd | 依调用方注入的 manager、profile 或权限模式决定 |

文件路由由 `createBoundaryFilesystemExecutor()` 集中完成；显式 `managed` 缺少 worker 时不会隐式改走宿主文件操作。[文件路由][m-filesystem]
表中 `host` 表示允许解析工作区外的宿主路径，不会提升进程的 OS 权限；远端执行器也未必支持此类路径范围。

`createGenesisExecutionBoundary()` 将 `explore` 映射为只读 profile，`ask` 映射为 `workspace-write`，`bypass` 映射为绕过边界。
当前 `workspace-write` 允许工作区及临时根写入，网络为 `restricted`；它没有默认添加 `.git/.agents/.codex` 写保护。[初始边界][m-genesis]、[默认 profile][m-profile]

`managed` 本身也不等于“必然存在 OS 沙箱”。`danger-full-access` 仍是 managed profile，但文件系统为 `unrestricted`、网络为 `enabled`。
`profileRequiresSandbox()` 仅在 managed profile 的文件系统或网络仍受限时返回真；默认 `auto` 选择可能因此得到 `sandboxType: 'none'`。[判断函数][m-required]

## 3. Bash 调用进入 Linux 沙箱的路径

Linux smoke 测试创建一个位于非标准 PATH 目录的 `maka-path-probe`，脚本内容为 `printf custom-tool-ok`。
测试向 `buildBuiltinTools()` 注入 `workspace-write` profile 和 Linux manager，未注入 `shellRuns`，因而进入普通 executor 分支。[测试入口][m-linux-test]

```text
bash.impl({ command: 'maka-path-probe' }, ctx)
  → buildExecutorBashTool().impl
  → preflightDeclaredSandboxBoundary(...)  // 此测试没有申请扩张
  → sandboxCommand(manager, profile, 'linux', command, false, ctx)
      → 规范化 cwd，准备 profile 路径
      → SandboxManager.transform(...)
      → LinuxBubblewrapBackend.transform(...)
          argv = [bwrap, ..., '--chdir', cwd, '--', '/bin/sh', '-c', command]
          fdInputs 包含受限网络的 seccomp 过滤器
  → LocalWorkspaceExecutor.exec({ argv, cwd, env, fdInputs, timeoutMs, ... })
  → runProcessWithBoundedTail(argv[0], argv.slice(1), ...)
  → shapeTerminalResult(...)
  → 测试断言 output.mode === 'pipes'
             output.stdout === 'custom-tool-ok'，output.stderr === ''
```

此处 `argv` 是已转换完成的可执行程序与参数；本机执行器直接启动该程序，不再经由宿主 shell 重新解析整个包装命令。
工具层将非零退出、超时、取消转换为对应的失败；执行器本身返回输出、退出码和状态事实。[Bash 实现][m-bash]、[本机 exec][m-local-exec]、[进程入口][m-run-process]

这条测试证明的行为包含“命令可从非标准 PATH 找到”。该测试不证明任意主机都具备 bubblewrap，也不证明环境变量中的秘密已被移除。
`sandboxCommand()` 拷贝 `process.env`；OS 文件限制不会自动清除已经传入子进程内存的凭据。文件 worker 的环境构造则更窄，见下文“文件工具为什么另起 worker”一节。

## 4. 前台、后台与 PTY：生命周期与隔离

注入 `shellRuns` 后，`buildBuiltinTools()` 改用 `buildManagedBashTool()`。
该工具先调用同一个 `transformCommand`／`sandboxCommand`，再按 `run_in_background` 选择 `runForegroundBash()` 或 `runBackgroundBash()`；后台执行并未跳过转换。[工具装配][m-tools]、[后台工具入口][m-shell-tools]

| 调用形式 | 运行约定 | 对边界的影响 |
| --- | --- | --- |
| 前台 pipes | 默认等待结果；默认 120 秒，最大 10 分钟 | 需要沙箱时先转换命令 |
| 后台 pipes | 返回可追踪的运行资源；无默认超时，显式超时最大 24 小时 | 同样先转换命令 |
| 后台 PTY | 必须同时设置 `pty: true` 和 `run_in_background: true` | profile 要求命令沙箱时，启动前拒绝 |

PTY 是伪终端，为程序提供终端输入、屏幕尺寸等语义，本身不提供权限隔离。`PtyProcessDriver` 封装 `node-pty` 的启动、输入、调整尺寸和终止；沙箱能力检查发生在调用该驱动之前。
因此“支持 PTY”与“支持受限 PTY”不能互换；当前受限 profile 下将返回 `requires_bypass`，没有边界对象的对应分支使用 `pty_sandbox_unavailable`。[PTY 检查][m-command]、[驱动][m-pty]

后台结果给出的 `maka://runtime/background-tasks/<id>` 是运行资源地址：`Read` 观察结果，`WriteStdin` 向 PTY 输入，`StopBackgroundTask` 请求停止。
这些接口向管理器传入 `sessionId` 与资源引用；`WriteStdin` 描述还明确说明输入是普通工具调用数据，并非秘密输入通道。[后台工具入口][m-shell-tools]

执行时限与资源额度仍需区分。上述沙箱后端没有设置通用 CPU／内存预算；Linux 的 `--unshare-cgroup` 也不等于配置了资源配额。
命令输出超过保留上限时，`shell-exec` 保留尾部并抑制实时输出，命令继续运行。输出截断不能替代超时或进程资源限制。[进程入口][m-run-process]

## 5. 文件工具的独立 worker 设计

只包装 Bash 会留下另一条执行入口：`Write` 仍可能在宿主直接执行文件 API。
Maka 将 `Read/Write/Edit/FormatJson/Glob/Grep/apply_patch` 的本地文件操作交给同一个边界执行器；managed 分支进入 `FilesystemWorkerClient.execute()`。[工具装配][m-tools]、[文件路由][m-filesystem]

以读取工作区中的 `note.txt` 为例，实际函数关系如下；这是源码路径，不是另行构造的消息协议：

```text
Read.impl({ path: 'note.txt' }, ctx)
  → filesystem.execute({ operation: { kind: 'read', path }, ...filesystemCall(ctx) })
  → createBoundaryFilesystemExecutor 内部 run()
  → FilesystemWorkerClient.execute()
      规范化目标 → canReadPath() 检查会话 profile
      deriveWorkerProfile() 收窄到本次操作 → SandboxManager.transform()
  → runFilesystemWorkerProcess()：JSON 请求写入 stdin
  → worker-entry.main() 校验请求
  → executeFilesystemWorkerRequest() → executeFilesystemOperation()
  → stdout 返回 JSON；客户端校验 requestId、结果类型后交回 Read
```

会话权限是可使用资源的上限；`deriveWorkerProfile()` 再将受限 profile 收窄至本次文件或目录，并将网络设为 `restricted`。
Grep 还可获得已授权的祖先 ignore 文件读取权限，但不能据此读取这些文件所在的整个目录。[worker 客户端][m-worker]

写操作在等待文件锁之前捕获目标身份，worker 再核对路径、类型及身份；内容改写通过固定文件句柄进行，减少检查后路径被替换造成的误写。
这仍不等于每种操作都有同样窄的内核授权：Linux 新建文件需要可写父目录，受信任的单次 worker 承担进一步约束；Windows 当前拒绝此类授权形状。[worker 客户端][m-worker]、[worker 操作][m-operations]

`buildFilesystemWorkerEnv()` 只保留受控临时路径、locale 等必要变量；Windows 补充 `SystemRoot/SystemDrive/LOCALAPPDATA`，Electron 增加启动标志。
该函数未像 Bash 那样复制整个宿主环境。[worker 环境][m-worker-env]
worker 默认时限 120 秒，请求上限 16 MiB、响应上限 8 MiB；这些是协议及等待限制，不是整个 Agent 的内存上限。
请求发出后若丢失响应，写入可能已经发生，文件执行器可报告 `ToolOutcomeUnknownError`，不应将 worker 退出解释为自动回滚。[worker 进程][m-worker-process]、[文件路由][m-filesystem]

## 6. 三个平台的实际支持范围

| 平台 | 已接入的 OS 机制 | 当前需要保留的限制 |
| --- | --- | --- |
| macOS | `/usr/bin/sandbox-exec` 加 Seatbelt policy；从 `deny default` 开始生成文件及网络规则 | 包含系统工具、运行库和元数据读取例外；并非只看得见工作区；受限 PTY 不支持 |
| Linux | bubblewrap 挂载及 namespace；受限网络另加 seccomp socket 过滤器 | 要通过 bwrap 可执行性／能力探测；过滤器支持 x64、arm64；显式 `deny` 条目不支持，会失败 |
| Windows | broker 启动专用文件 worker，使用 AppContainer、文件 ACL 授权和 Job | 文件 worker 预览；受限 Bash、受限 PTY、worker 内 Grep 不可用 |

macOS 将 profile 转换为规则文本；Linux 将系统路径只读挂载，工作区按 profile 绑定，临时根使用 tmpfs。
Linux 的网络限制同时使用独立网络 namespace 和拒绝创建 socket 的过滤器，不能假定本地 HTTP 测试仍可联网。
相关 smoke 测试断言 Node 连接 `127.0.0.1` 得到 `EPERM`。[Seatbelt][m-macos]、[Linux 后端][m-linux]、[网络测试][m-linux-network-test]

Windows 需从四个层面考察：

1. `createDefaultSandboxManager()` 只注册 macOS/Linux；`createBuiltinSandboxManager()` 找到打包的 `windows-sandbox/maka-windows-sandbox.exe` 才增加 Windows 后端。
2. 文件存在尚不充分。`isBuiltinFilesystemWorkerSandboxAvailable()` 调用 readiness probe；探测成功才发布 worker，热路径只读缓存。负结果过期后需新的 composition 构建或 Host 重启重新探测。[可用性检查][m-default]
3. `compileWindowsSandboxPolicy()` 仅接受 managed、受限文件系统及受限网络；显式 `deny`、卷根和需要父目录项写权限的操作会被拒绝。已有文件内容写入与“创建新文件”不能视为同一种支持能力。[Windows 策略][m-windows-profile]
4. `sandboxCommand()` 对 Windows 直接认定命令沙箱不可用：受限 profile 失败；不要求沙箱或明确 bypass 时才使用未包装的宿主 shell。worker 的 Grep 则返回 `grep_unavailable`，可用 Glob 和 Read 继续读取。[命令边界][m-command]、[worker 操作][m-operations]

Windows 原生 launcher 已实现 AppContainer 启动、仅继承三条标准流、原子 Job 关联和结束时清理 Job；这不是仅停留在 RFC 层面的计划。
但私有桌面实现明确仅保证初始放置及桌面对象访问控制：仍共享 window station，不承诺剪贴板隔离；防止重新附着交互桌面的机制仍属后续工作。[原生启动][m-windows-launch]、[桌面限制说明][m-windows-desktop]

## 7. 与 Pi 对照：环境选择与强制边界

Pi 默认 CLI 的 `createBashTool()` 最终使用 `BashOperations`；默认实现直接启动本机 shell，扩展可以替换这些操作。
实验 durable 使用独立的 `ExecutionEnv extends FileSystem, Shell`。两条路径表达相似需求，但不能组合成一条默认调用链。[Pi Bash][p-bash]、[Pi 环境接口][p-env]

| 比较点 | Maka 本章路径 | Pi 已核对的路径 |
| --- | --- | --- |
| 执行接口 | `WorkspaceExecutor` 提供命令、文件操作及执行事实 | 默认 CLI 用 `BashOperations`；durable 用 `ExecutionEnv` |
| 边界不可用 | managed 文件缺 worker 时失败；受限命令后端不可用时失败 | 可选 sandbox 示例未启用或初始化失败时保留本机 bash |
| 工具覆盖 | managed 本地文件工具另走 worker；Bash 先转换命令 | sandbox 示例替换 bash 和用户 `!`，没有替换 read/write/edit |
| 网络配置 | profile 为 `restricted/enabled`，由平台后端落实 | sandbox 示例向外部 runtime 传入 allowedDomains 等配置 |
| SSH | `external` 把执行责任交给注入后端，标签本身不证明隔离 | `ssh.ts` 示例转交 read/write/edit/bash 与 `!`；远端账号仍需自己的权限限制 |
| 可复用远端环境 | 本章讨论的是本地工具边界 | `packages/env` 的 `RemoteExecutionEnv.exec()` 经连接发给 daemon；`connectSsh()` 负责建立 SSH 连接 |
| 每会话环境 | 边界选择与实际文件授权分开 | durable 示例按会话文档分配不同目录，返回 `NodeExecutionEnv`；真实容器后端留给应用 |

Pi 的 SSH 示例与 `packages/env` 是两个不同入口；前者直接启动 ssh 命令，后者提供环境协议与 daemon 集成。[SSH 示例][p-ssh-example]、[远端环境][p-remote]、[SSH 连接][p-ssh]
Pi sandbox 示例支持 macOS/Linux，当前代码在 Windows 上禁用该示例；该示例并非默认 CLI 的强制沙箱，也不能为其他文件工具证明隔离。[sandbox 示例][p-sandbox]
durable 示例确实实现了会话到执行环境的选择，并用 faux provider 演示；其 Sandbox 文档保存目录路径，不建立 OS 隔离。[每会话环境示例][p-durable-example]

## 8. 源码定位

| 入口 | 应重点追踪的标识符与问题 |
| --- | --- |
| [workspace-executor.ts][m-local-exec] | `LocalWorkspaceExecutor.exec()`：有 argv 时怎样启动程序 |
| [sandbox-boundary.ts][m-genesis] | `createGenesisExecutionBoundary()`：权限模式如何产生边界 |
| [sandbox-manager.ts][m-manager] | `SandboxManager.selectInitial()/canEnforce()/transform()`：选择、能力与转换分别发生在哪里 |
| [builtin-tools.ts][m-bash] | `buildExecutorBashTool()/sandboxCommand()`：普通 Bash、PTY 和 Windows 分支 |
| [shell-tools.ts][m-shell-tools] | `buildManagedBashTool()`：前后台分流前执行哪次转换 |
| [filesystem-executor.ts][m-filesystem] | `createBoundaryFilesystemExecutor()`：worker、本地与 external 的路由 |
| [filesystem-worker/client.ts][m-worker] | `FilesystemWorkerClient.execute()/deriveWorkerProfile()`：批准范围如何收窄为单次操作 |
| [filesystem-worker/operations.ts][m-operations] | `executeFilesystemWorkerRequest()/executeFilesystemOperation()`：目标复核、实际读写与 Grep 拒绝 |
| [macos-seatbelt.ts][m-macos] | `buildSeatbeltPolicy()`：允许规则与平台例外 |
| [linux-sandbox.ts][m-linux] | `LinuxBubblewrapBackend.transform()/buildBubblewrapArgv()`：挂载、描述符及网络过滤 |
| [windows-profile.ts][m-windows-profile] | `compileWindowsSandboxPolicy()`：精确写入与父目录项授权的差别 |
| [default-sandbox-manager.ts][m-default] | `createBuiltinSandboxManager()/isBuiltinFilesystemWorkerSandboxAvailable()`：注册不等于可运行 |
| [windows_launcher.rs][m-windows-launch] | `create_appcontainer_child()`：安全属性、标准流和 Job 随启动建立 |
| [Pi bash.ts][p-bash] | `createLocalShellOperations()`：默认本地操作与超时取消 |

Windows smoke 用例分别断言已有文件可更新、缺失目标写入在 transform 阶段失败，以及 Glob 成功而 Grep 失败；测试需要 Windows 和已有 broker 二进制，否则跳过。[Windows 用例][m-windows-test]
Linux smoke 同样按平台及 bwrap 能力跳过。本章引用的是用例与实现，不宣称这些测试已在本次环境中通过。

## 9. 本章术语

| 术语 | 本章含义 |
| --- | --- |
| managed boundary | Maka 管理的 profile 与版本；仍需检查 profile 是否要求 OS 沙箱 |
| OS enforced sandbox | 操作系统对实际进程实施的访问限制；不依赖模型遵守提示词 |
| fail closed | 无法建立所需限制时拒绝执行，不静默扩大权限 |
| PTY / pipes | 伪终端语义／标准流管道；是输入输出方式，不是安全等级 |
| operation boundary | 文件 worker 本次操作所获的路径范围，通常比会话权限窄 |
| Job | Windows 用于关联和清理进程组的内核对象；本实现未据此设置通用 CPU／内存配额 |
| bypass | 显式跳过 Maka 的本地限制；进程仍受宿主操作系统权限约束 |

## 10. 源码练习

1. 静态追踪四个请求：Linux managed 后台 pipes、Linux managed PTY、Windows managed Bash、Windows bypass Bash。
   在 `buildManagedBashTool()` 与 `sandboxCommand()` 标出分支，说明哪些会调用 launcher、哪些先失败，以及失败是否会自动改为 bypass。
   判断依据必须包括 profile 是否受限，不能仅依据 `managed` 字样。
2. 阅读 Windows smoke 测试，将 `inside.txt` 已存在与 `not-created-yet.txt` 不存在的执行轨迹追踪至 policy 编译处。
   解释为何两者同在允许目录内却结果不同；再说明“Grep 不可用”为何不能视为“没有匹配”。无需运行 broker。
3. 阅读 Pi 的 sandbox、SSH 和 durable 每会话环境示例，列出各自替换的操作与未建立的限制。
   再分别为 Maka 的文件 worker 和 Bash 整理环境变量来源，说明禁止读取凭据文件是否足以阻止环境变量继承。全部内容可仅通过只读源码完成，无需付费 API。

## 相关专题

- [权限与秘密](permissions-and-secrets.md)：区分获准执行、实际资源限制和凭据注入。
- [工具运行时](tool-runtime.md)：定位文件或命令执行前后的事实提交。
- [外部工具接入](mcp-and-client-capabilities.md)：理解哪些效果发生在本机沙箱之外。

[m-executor]: ../../../../maka/packages/runtime/src/workspace-executor.ts#L56
[m-local-exec]: ../../../../maka/packages/runtime/src/workspace-executor.ts#L315
[m-boundary]: ../../../../maka/packages/core/src/sandbox-boundary.ts#L162
[m-genesis]: ../../../../maka/packages/core/src/sandbox-boundary.ts#L229
[m-profile]: ../../../../maka/packages/core/src/permission-profile.ts#L176
[m-manager]: ../../../../maka/packages/runtime/src/sandbox/sandbox-manager.ts#L35
[m-required]: ../../../../maka/packages/runtime/src/sandbox/sandbox-manager.ts#L204
[m-filesystem]: ../../../../maka/packages/runtime/src/filesystem-executor.ts#L190
[m-tools]: ../../../../maka/packages/runtime/src/builtin-tools.ts#L210
[m-bash]: ../../../../maka/packages/runtime/src/builtin-tools.ts#L631
[m-command]: ../../../../maka/packages/runtime/src/builtin-tools.ts#L737
[m-run-process]: ../../../../maka/packages/runtime/src/shell-exec.ts#L136
[m-shell-tools]: ../../../../maka/packages/runtime/src/shell-tools.ts#L175
[m-pty]: ../../../../maka/packages/runtime/src/pty-process-driver.ts#L42
[m-worker]: ../../../../maka/packages/runtime/src/filesystem-worker/client.ts#L190
[m-operations]: ../../../../maka/packages/runtime/src/filesystem-worker/operations.ts#L78
[m-worker-env]: ../../../../maka/packages/runtime/src/filesystem-worker/launch-spec.ts#L93
[m-worker-process]: ../../../../maka/packages/runtime/src/filesystem-worker/process-runner.ts#L77
[m-macos]: ../../../../maka/packages/runtime/src/sandbox/macos-seatbelt.ts#L239
[m-linux]: ../../../../maka/packages/runtime/src/sandbox/linux-sandbox.ts#L84
[m-default]: ../../../../maka/packages/runtime/src/sandbox/default-sandbox-manager.ts#L178
[m-windows-profile]: ../../../../maka/packages/runtime/src/sandbox/windows-profile.ts#L36
[m-windows-launch]: ../../../../maka/experiments/windows-sandbox/launcher/src/windows_launcher.rs#L1120
[m-windows-desktop]: ../../../../maka/experiments/windows-sandbox/launcher/src/windows_launcher.rs#L752
[m-linux-test]: ../../../../maka/packages/runtime/src/__tests__/linux-sandbox-smoke.test.ts#L205
[m-linux-network-test]: ../../../../maka/packages/runtime/src/__tests__/linux-sandbox-smoke.test.ts#L160
[m-windows-test]: ../../../../maka/packages/runtime/src/__tests__/filesystem-worker-windows-smoke.test.ts#L103
[p-bash]: ../../../packages/coding-agent/src/core/tools/bash.ts#L97
[p-env]: ../../../packages/durable/src/env/index.ts#L374
[p-sandbox]: ../../../packages/coding-agent/examples/extensions/sandbox/index.ts#L132
[p-ssh-example]: ../../../packages/coding-agent/examples/extensions/ssh.ts#L81
[p-remote]: ../../../packages/env/src/remote-env.ts#L852
[p-ssh]: ../../../packages/env/src/ssh.ts#L532
[p-durable-example]: ../../../packages/durable/test/examples/29-sandbox-per-conversation.ts#L48

# 输入队列与执行准入

[手册目录](../README.md)

适用范围：Maka Host 的普通交互输入、Session 消息队列、子 Agent 执行准入与后台 Bash；不把它们泛化成分布式任务队列。
Maka 固定提交：`15d0a0dd19d39e2bc09ae35940e432c636982315`；Pi 对照提交：`482c1381a00d8d00bc81697e4a980c593db6803e`。
本文依据源码静态追踪及已读取的测试；未运行模型请求、测试或故障注入。Pi 引用限于未修改的文件，不将工作区其他改动算入该提交。

## 1. 问题：排队对象的界定

假设 Session S 正在执行“检查项目”，此时发生三种情况：

1. 用户补充“只读，不要修改文件”，期望当前工作尽早接收到该输入。
2. 用户随后提交“之后检查日志”，期望当前工作结束后再执行。
3. Agent 启动一个长时间 Bash 命令，工具先返回可查询的任务引用。

这三种情况不能用一个 `queue.length` 描述：前两项是尚待消费的输入，第三项已取得执行资源。
若将“工具已返回”视为“资源已释放”，持续启动后台命令就可能耗尽进程容量。
若将“消息已收到”视为“根 Turn 已建立”，Host 崩溃后的恢复将缺少明确的交接依据。

Maka 因此分别管理输入接纳、根 Turn 身份、消息投递与执行名额；下文逐层跟踪，而非寻找一个统一承担所有工作的 worker 池。

## 2. 持久记录、内存状态与名额

| 机制 | 管什么 | 持久与内存边界 | 满载或竞争时的行为 |
| --- | --- | --- | --- |
| `SessionAdmissionGate` | 同一 Session 的准入操作顺序 | Promise 尾链和 lease 都在内存 | 后来的准入操作等待前一个完成 |
| 根 Turn 准入 | 哪个 Turn/Run 接替当前根执行 | 根准入记录持久保存；活动执行与预留在内存 | 同一 Session 不允许竞争的活动/预留根执行 |
| Session message queue | 当前 steering 与后继 followup 输入 | 输入接纳记录持久保存；队列投影和投递 lease 在内存 | 条目数或投影容量不够时拒收 |
| `AdmissionLimiter` | 某个作用域的并发执行许可 | active、waiters 和 permit 均在内存 | 按 FIFO 等待；支持取消等待 |
| shell runtime task | 已启动或正在启动的 shell 运行 | 记录和输出经 store 保存；活进程句柄、名额在内存 | 容量不足直接拒绝启动，不自动入队 |

这里的“持久”指 Host/runtime 通过存储接口提交的数据，不包括 JavaScript Promise 或原生进程句柄。
这些边界分别见 [Session gate][m-gate]、[根准入 owner][m-root-owner]、[消息准入][m-submit]、[limiter][m-limiter] 和 [shell 名额][m-shell-slot]。

## 3. Host input admission：消息接纳与 Turn 占用

普通 `turn.start` 经 `HostInteractiveTurnCoordinator.#start()` 进入 `startInteractiveRootMessage()`。
后者采用根消息启动路径：检查根执行状态、建立持久根准入，再准备实际运行；已存在冲突的根执行时返回 `session_busy`，并不自动排入 followup。[交互入口][m-interactive]、[根启动][m-root-start]

`turn.message.submit` 则先由 `HostMessageCoordinator` 判断 Session 是否空闲。
空闲时调用 `startFromMessage()` 建立根 Turn；活动根仍开放接纳时，`current_turn` 归为 steering，`next_turn` 归为 followup。
根仅处于预留状态，或活动执行不允许接纳时，也可能返回 `session_busy`，并非所有忙碌状态均可排队。[消息入口][m-submit]

以下是普通文本输入的真实函数路径，省略参数校验与错误分支；它是源码调用轨迹，不是本次运行日志：

```text
Session S 空闲，提交“检查项目”
turn.message.submit → HostMessageCoordinator.submit / #submitAdmitted
  → SessionAdmissionGate.run：串行进入该 Session 的准入区
  → RootTurnCoordinator.startFromMessage
      → commitMessageAdmission：记录被接纳的输入
      → RootAdmissionOwner.admitRootTurn：确立根 Turn/Run 与来源
      → handoffRootSources：标记输入已交给根执行
      → prepareAdmittedTurn
          → sessionAdmission.detach(() => drainTurn(...))
  ← 返回 turn_started；不等待整个 Turn 完成

S 的根执行活动中，提交“之后检查日志”，placement=next_turn
#submitAdmitted → 容量预检 → commitMessageAdmission
  → state.followup.push(entry) → 返回 followup
当前根结束 → completeTerminalTransition → beginTerminalTransition
  → startFollowupBatch → admitRootTurn → commitNextRoot → prepareAdmittedTurn
```

`detach()` 离开的是准入操作的异步上下文，并非将任务迁移到另一个进程。
若持续运行的 `drainTurn()` 一直占用准入区，用户将无法在执行期间补充输入；因此短准入操作与长执行生命周期必须分开。[消息建根][m-from-message]、[执行脱离准入][m-detach]、[后继交接][m-transition]

## 4. Session queue：输入保存与投递交接

忙碌 Session 的接纳顺序是：准备内容、检查容量、`commitMessageAdmission()` 成功，然后才插入内存队列并返回结果。
持久记录包含 `messageId`、输入内容、提交时 placement、当前 disposition 以及关联的 Turn/Run 身份；它不是单纯的 UI 消息列表。[记录结构][m-admission-record]、[写入顺序][m-commit-message]

steering 的投递另有一层状态：

```text
queued steering ── #pull ──> inFlight（分配 leaseId）
inFlight ── #ack ──> 删除内存条目并释放其驻留引用
inFlight ── #nack ──> 同代且接纳仍开放时，放回 steering 队头
```

`#pull()` 会等待尚未完成的当前轮输入接纳和队列修改，避免将“持久写入尚未完成”误判为“没有新输入”。
一次 pull 取出当时排队的全部 steering；lease 绑定具体 Run，不应视为跨 Host 的消息代理租约。[投递实现][m-pull]

`#ack()` 本身只修改内存。恢复时不能据此推断消息是否已被消费：Host 会读取持久根来源记录或 steering 事件等证据，补齐交接标记，再处理仍待消费的接纳记录。
重建队列会生成新的 `entryId`；`queueRevision`、generation 和 lease 也是当前 Host 的协调状态，不是持久消息身份。[恢复入口][m-recover]

根执行结束前要求 in-flight steering 已经结清；尚未被 pull 的 steering 会移到 followup 前部。
`nextSuccessorItems()` 将队首连续的 steering 合成一个后继批次；普通 `next_turn` 则一次只取一条，各自成为后继根输入。
因此“后继队列”并非将所有内容无差别地提供给下一次模型请求。[终止转换][m-terminal]、[批次选择][m-successor]

容量也分两个维度：最多 **64 个存活队列条目**，计入 queued steering、in-flight steering 和 followup；队列投影最多 **52 KiB**。
另有 **56 KiB** 的操作结果预算，以及 Session 快照、后继根记录等编码预检。52 KiB 是序列化投影的预算，并非每条用户文本均可占用 52 KiB，更不是全局存储配额。[限制常量][m-queue-limits]、[接纳预检][m-submit]

## 5. AdmissionLimiter：执行数量限制

消息被接纳后，内部工具仍可能放大并发。例如一个 subagent 工具同时申请 40 个子 Agent 运行。
仅限制工具调用数量并不充分，因为“一次工具调用”不等于“一次子运行”。Maka 为此设置了两道不同的边界。[工具容量][m-tool-limits]

| 边界 | 固定提交中的额度与作用域 | 超出时 |
| --- | --- | --- |
| `categoryHint === 'subagent'` 的活动工具 | 每 Turn 最多 5 个 | `reserveSubagentSlot()` 失败，返回工具错误 |
| `spawnChildSession` 的子运行许可 | 同一 Turn 的调用者共享 32 个 | 在该 Turn 的 `AdmissionLimiter` 等待 |

`buildChildAgentContext()` 捕获当前 Turn 的 limiter：先 `await limiter.acquire(signal)`，再调用实际子运行能力，最后在 `finally` 中 `permit.release()`；许可持有到子运行调用的 Promise 结束。

因此上述 40 个请求最多先放行 32 个，其余等待；这并不表示整个 Host 所有 Session 合计只有 32 个子运行。
FIFO 保证已有等待者不会被新请求抢先；等待期间 abort 会移除 waiter；release 可重复调用而不会重复减计数。[许可实现][m-limiter]、[子运行封装][m-child]

Turn 重置会关闭旧 limiter。`close()` 拒绝排队者与后续申请，但其自身不负责终止已持有许可的运行；活动工作的取消由其信号和生命周期负责。
limiter 类也没有内建 waiter 数量上限，更没有将等待者保存为可跨重启继续运行的任务。[关闭行为][m-limiter]、[Turn 重置][m-reset]

对应测试已覆盖 FIFO、等待取消、跨工具共享许可、子启动抛错后归还，以及旧 Turn 上下文不能继续启动子运行；这些不是仅写在设计文档中的目标。[测试入口][m-limiter-test]

## 6. 后台 Bash 的名额占用

`ShellRunProcessManager.runBackgroundBash()` 启动进程、保存观测状态，然后返回结果；若进程仍运行，返回的是紧凑的 runtime task 引用。
这表示调用方可以继续执行其他工作，不表示命令已完成，更不表示 shell slot 已归还。[后台入口][m-shell-background]

`start()` 在异步存储和原生进程启动之前先 `reserveSlot()`，启动失败会归还；正常运行由 `finalizeLive()` 清理 driver、保存终态并释放名额。
前台 Bash 也经由该启动与计数路径，因此不能将内部错误文案中的 “background task” 理解为仅统计后台工具返回值。[启动边界][m-shell-start]、[结束清理][m-shell-finalize]

默认总量是 **64 个 live shell runs**，其中 PTY（伪终端）最多 **8 个**；PTY 同时占总额度。
构造参数可以覆盖默认值，计数属于同一个 manager，跨其管理的 Session 共享。
容量耗尽时直接抛错：调用者可以等待已有进程结束后重试，manager 并不会为该次失败创建等待队列。[默认值][m-shell-defaults]、[计数器][m-shell-slot]

重启后，保存的运行记录不等于仍持有原生进程控制权。
`recoverOrphanedSession()` 发现记录仍是活动状态、但没有 live handle 或 startup owner 时，将其标为 `orphaned`；它不会重新执行原命令。
`orphaned` 也不能证明操作系统中的原进程必然已经退出。这里恢复的是记录的可解释状态，不是恢复旧进程的控制句柄。[孤立记录处理][m-shell-recover]

## 7. 与 Pi 对照：默认 agent 与实验 durable

Pi 默认 `coding-agent → agent → ai` 路径与 `packages/durable` 实验 harness 是两条路径。
前者的输入消费逻辑不能代表后者的恢复能力，也不能由本章的队列比较推断出 Pi 没有编排、恢复或沙箱。

| 问题 | Maka Host/runtime | Pi 默认 agent 路径 | Pi 实验 durable 路径 |
| --- | --- | --- | --- |
| 忙碌时的存放位置 | Host 管理 steering/followup 与持久接纳记录 | `Agent.steer()` / `followUp()` 写内存 `PendingMessageQueue` | `admitSubmission()` 在事务中接纳，忙时可排入 `InboxDoc` 或拒绝 |
| 消费时机 | steering 经 Run-bound pull；followup 在根交接时选择 | steering 在循环边界；followup 在循环本来将结束时检查 | `applyBoundary()` 在 `postTools` / `final` 选择输入 |
| 单次取用量 | 当前 queued steering 全取；普通后继输入逐条建根 | 默认 `one-at-a-time`，可改 `all` | steering/followup 各有 one/all 设置；followup 通常在 final 放置 |
| 容量控制 | 队列数量/字节、工具数、child permit、shell slot 各自独立 | 输入队列本身不是工具执行并发限制器 | scheduler 按依赖与任务状态决定可运行项，不等于固定大小许可池 |
| 重启的依据 | 输入与根准入证据；shell 缺句柄标 orphaned | 此处的待消费数组本身不持久；不应与会话历史混同 | 持久 inbox、task record、checkpoint 与重启计数 |

Pi 默认 `runLoop()` 先完成本次 assistant 的工具批次，再在 `finishTurn` 未要求结束时检查 steering；不是每完成一个工具就检查并跳过剩余工具。
`followUp()` 则在内层循环将停止时被消费。这里的 Pi `turn` 也不宜直接等同于 Maka Host 的根 Turn。[内存队列][p-agent]、[循环边界][p-loop]

durable 的输入队列与任务调度仍需进一步区分：`InboxDoc` 保存等待放置的输入，`applyBoundary()` 在事务内写入条目、关联 submission 并移除所选输入。[持久接纳][p-submissions]、[inbox 消费][p-inbox]
调度器处理的则是已有 task record，其恢复和执行路径如下：

```text
TaskScheduler.open()
  → 读取存活任务；遗留 running 改 pending，保留 checkpoint，增加 restarts
resume() → #kick → #drain → #reserve
  → 排除等待依赖、未到唤醒时间、被停放或缺少可用定义等任务
  → 提交 running 状态并建立内存 invocation
  → #start → #run → 执行 checkpoint 对应 phase
```

`open()` 不是立即执行全部任务；还需启用调度。达到重启限制的任务也不会无条件重新运行。
当前 `#reserve()` / `#drain()` 路径没有 Maka child limiter 那样的固定并发额度；它解决的是任务何时具备运行条件，而非承诺“最多 N 个 worker”。[恢复与启用][p-scheduler-open]、[调度预留][p-scheduler-reserve]

## 8. 源码定位

下列入口可与前述轨迹逐项核对；Maka 与 Pi 均使用本地相对链接，读取当前工作区文件。源码阅读基线见文首。

| 源码入口 | 函数或符号 | 阅读时需确认的边界 |
| --- | --- | --- |
| [session-admission-gate.ts][m-gate] | `SessionAdmissionGate.run()` / `detach()` | 同 Session 的短准入操作与长执行分离 |
| [interactive-turn-coordinator.ts][m-interactive] | `HostInteractiveTurnCoordinator.#start()` | `turn.start` 进入根启动，不是默认排队 |
| [root-admission-owner.ts][m-root-owner] | `RootAdmissionOwner.admitRootTurn()` | 持久根准入链的交接身份 |
| [root-turn-coordinator.ts][m-from-message] | `startFromMessage()` | 从已接纳输入建立根执行 |
| [message-coordinator.ts][m-submit] | `#submitAdmitted()` | 空闲建根、忙碌排队与容量拒收 |
| [message-coordinator.ts][m-pull] | `#pull()` / `#ack()` / `#nack()` | queued 与 in-flight 的转换 |
| [message-coordinator.ts][m-recover] | `recoverPendingAfterHostRestart()` | 依据持久证据重建待消费输入 |
| [admission-limiter.ts][m-limiter] | `acquire()` / `close()` | 许可与等待者都在内存 |
| [tool-runtime.ts][m-child] | `buildChildAgentContext()` | 许可覆盖真实子运行调用的生命周期 |
| [shell-run-manager.ts][m-shell-slot] | `reserveSlot()` / `releaseSlot()` | 拒绝超额启动，而非自动排队 |
| [shell-run-manager.ts][m-shell-recover] | `recoverOrphanedSession()` | 持久记录不能替代活动句柄 |
| [agent.ts][p-agent] / [agent-loop.ts][p-loop] | `PendingMessageQueue` / `runLoop()` | 默认 Pi 的输入消费边界 |
| [inbox.ts][p-inbox] / [scheduler.ts][p-scheduler-reserve] | `applyBoundary()` / `TaskScheduler.#reserve()` | durable 输入放置与任务执行是两个阶段 |

## 9. 术语速查

| 术语 | 本章含义 |
| --- | --- |
| admission（准入） | 决定是否接纳一次操作或建立执行身份，不意味着工作已经完成 |
| root Turn | Maka Host 协调的根交互执行单元，区别于其内部子运行 |
| steering / followup | 当前执行的补充输入 / 留给后继执行的输入 |
| lease | 一次受约束的访问或投递凭据；Session 准入 lease 与 steering lease 不是同一个对象；存储与数据库 lease 见 04-state 两篇 |
| permit / slot | 正在使用的执行名额；其释放时点由对应资源生命周期决定 |
| Pi durable 的 checkpoint / invocation | 保存的任务进度 / 当前进程内一次执行实例；不应与 Maka 的持久 invocation 身份混同（手册内 checkpoint 另有两义：02-context 的历史压缩 checkpoint 与 04-state 的续跑工作区 checkpoint） |
| orphaned | 保存的 shell 运行记录没有可接管的活动句柄，不能据此判定命令成功或失败 |

## 10. 源码练习

1. 纸面追踪：当前根最后一次 pull 后又收到 S1、S2 两条 steering，随后收到 F1、F2 两条 `next_turn`。
   不考虑取消或编辑，终止转换会如何分批？对照 `beginTerminalTransition()` 与 `nextSuccessorItems()`。
   核对点：先将 S1+S2 合成后继批次，F1、F2 各占后续一个批次，不应将四条视为一次 FIFO drain。
2. 阅读 `admission-limiter.test.ts`，将容量设为 2：A、B 已持有许可，C 等待后取消，D 随后等待，再释放 A 两次。
   列出 active/waiting 数量并说明哪个将启动。核对点：C 不占用执行名额，D 获得许可，重复 release 不会额外放行。
3. 对照 shell 恢复与 durable `open()`：Host 重启后分别留下一个活动 shell 记录和一个 running task record。
   哪个成为 orphaned，哪个保留 checkpoint 改 pending？继续运行还需要什么条件？核对点：两者都不能仅凭旧状态恢复原来的内存执行实例。

证据边界：本章确认的是上述提交中已实现的控制流，没有测量吞吐，也没有证明跨 Host 的分布式互斥或外部副作用恰好执行一次。
读取持久记录后重新执行 phase，尤其不等于自动消除该 phase 对外部系统的重复副作用。

## 相关专题

- [领域模型](../00-foundations/domain-model.md)：确认输入、执行、步骤和尝试的身份边界。
- [持久化与恢复](../04-state/durable-execution.md)：追踪状态提交与中断后继续执行的条件。

[m-gate]: ../../../../maka/packages/runtime-host/src/server/session-admission-gate.ts#L50
[m-root-owner]: ../../../../maka/packages/runtime-host/src/server/root-admission-owner.ts#L85
[m-interactive]: ../../../../maka/packages/runtime-host/src/server/interactive-turn-coordinator.ts#L74
[m-root-start]: ../../../../maka/packages/runtime-host/src/server/root-turn-coordinator.ts#L2112
[m-from-message]: ../../../../maka/packages/runtime-host/src/server/root-turn-coordinator.ts#L1412
[m-detach]: ../../../../maka/packages/runtime-host/src/server/root-turn-coordinator.ts#L3062
[m-transition]: ../../../../maka/packages/runtime-host/src/server/root-turn-coordinator.ts#L3466
[m-submit]: ../../../../maka/packages/runtime-host/src/server/message-coordinator.ts#L1329
[m-admission-record]: ../../../../maka/packages/storage/src/message-admission-store.ts#L42
[m-commit-message]: ../../../../maka/packages/runtime-host/src/server/message-coordinator.ts#L1656
[m-pull]: ../../../../maka/packages/runtime-host/src/server/message-coordinator.ts#L2320
[m-recover]: ../../../../maka/packages/runtime-host/src/server/message-coordinator.ts#L1115
[m-terminal]: ../../../../maka/packages/runtime-host/src/server/message-coordinator.ts#L911
[m-successor]: ../../../../maka/packages/runtime-host/src/server/message-coordinator.ts#L2965
[m-queue-limits]: ../../../../maka/packages/runtime-host/src/protocol/message-queue-limits.ts#L20
[m-limiter]: ../../../../maka/packages/runtime/src/admission-limiter.ts#L46
[m-tool-limits]: ../../../../maka/packages/runtime/src/tool-runtime.ts#L336
[m-child]: ../../../../maka/packages/runtime/src/tool-runtime.ts#L2190
[m-reset]: ../../../../maka/packages/runtime/src/tool-runtime.ts#L890
[m-limiter-test]: ../../../../maka/packages/runtime/src/__tests__/admission-limiter.test.ts#L36
[m-shell-background]: ../../../../maka/packages/runtime/src/shell-run-manager.ts#L330
[m-shell-start]: ../../../../maka/packages/runtime/src/shell-run-manager.ts#L739
[m-shell-finalize]: ../../../../maka/packages/runtime/src/shell-run-manager.ts#L1295
[m-shell-defaults]: ../../../../maka/packages/runtime/src/shell-run-contract.ts#L44
[m-shell-slot]: ../../../../maka/packages/runtime/src/shell-run-manager.ts#L1967
[m-shell-recover]: ../../../../maka/packages/runtime/src/shell-run-manager.ts#L665
[p-agent]: ../../../packages/agent/src/agent.ts#L143
[p-loop]: ../../../packages/agent/src/agent-loop.ts#L163
[p-submissions]: ../../../packages/durable/src/harness/submissions.ts#L171
[p-inbox]: ../../../packages/durable/src/harness/inbox.ts#L70
[p-scheduler-open]: ../../../packages/durable/src/harness/scheduler.ts#L352
[p-scheduler-reserve]: ../../../packages/durable/src/harness/scheduler.ts#L1433

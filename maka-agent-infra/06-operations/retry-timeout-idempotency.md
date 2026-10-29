# 重试、超时与幂等

[手册目录](../README.md)

适用范围：2026-10-24，Maka 固定提交 `15d0a0dd19d39e2bc09ae35940e432c636982315`。
Pi 对照基线为 `482c1381a00d8d00bc81697e4a980c593db6803e`，以当前工作区源码为准，包含用户已有的会话层未提交修改；不把这些修改归属于该提交。
本文核对源码与测试断言，未运行测试、安装依赖或访问真实 provider；不逐一审计所有 provider 的内部重试。

## 1. 问题：无响应请求的重发判定

假设 agent 调用 `create_ticket` 创建工单。服务端已经创建成功，响应却在网络中丢失。
客户端观察到的是 timeout；重新发送可能生成第二张工单。将 timeout 解释为“操作没有发生”，正是重复副作用的来源。
同样的现象也可能发生在模型请求和 Host 命令上，但三个层次的重试依据不同。

必要的设计次序是：首先辨认“什么已经发生”，再决定能否重试。
指数退避只能减轻拥塞；数据库去重只能覆盖它参与的事务；取消信号也不能撤销已完成的远端操作。

## 2. 重试循环的归属

Maka 当前模型调用的核心循环在 Runtime 的 `AiSdkTurn.runWithinScope()`，而非由 Host 重启整个 Turn 进行重试。
同一逻辑 step 内，它建立新的 provider attempt、请求级 AbortController 和 watchdog，再调用 `modelAdapter.startStream()`。[调用循环][m-attempt]

| 层次 | 当前职责 | 不应混淆的事 |
| --- | --- | --- |
| Runtime / `AiSdkTurn` | 判断模型 attempt 能否重试、计数、退避、重新请求 | 不等于重新执行已完成工具 |
| Host / `recordProviderRetry` | 将 `scheduled`、`started` 等事件投影到 root Turn 状态 | 不是另一个 provider 重试调度器 |
| Host reconnecting client | 对允许重试的查询等待连接并再次查询 | 不自动重发写命令 |
| 工具及外部服务 | 决定业务操作如何去重、查证或补偿 | 不能借模型 retry 获得业务幂等 |

Host 的投影还检查 Turn 是否已经终态，跨快照携带 retry 状态时要求相同 run/turn。
因此 UI 显示“稍后重试”是运行时决策的可见状态，不是新的执行授权。[Host 投影][m-projection]
完整调用层次可结合 [运行调用链](../01-runtime/request-to-response.md) 阅读。

## 3. 代码路径：模型流断开后的 attempt

下面是依据当前函数展开的故障轨迹，而非实际访问付费模型得到的日志。
设模型请求已经发出，只收到普通文本，随后发生可重试的网络中断；没有 provider 工具活动，也没有不可重放的续接元数据。

```text
AiSdkTurn.runWithinScope()
  step S / attempt 1 → startWatchdog() → modelAdapter.startStream(...)
  网络请求发出 → 收到 text → 记录可见文本，但尚无工具副作用证据
  流失败 → provider failure 被标记为 retryable
  attemptCanReplay() + step 预算 + attempt 上限检查通过
  flushStep(true) → 保留失败片段于 transcript，排除于后续模型历史
  更换 currentStepMessageId → provider_retry(scheduled, attempt=2)
  providerRetrySleep(delay, turnAbortSignal)
  attempt 2 → provider_retry(started) → 新请求、新 watchdog
```

这里“可见过文本”和“不可安全重放”不是同一个条件。
`attemptCanReplay()` 检查的是 `!attemptSawToolActivity && !attemptSawReplayBarrier`；普通文本片段本身不必阻止重试。[重放门槛][m-attempt]
但 provider 原生续接元数据等可能建立 replay barrier；不应将所有 text/thinking 视为可丢弃的字符串。
特别是 `tool-input` 已标记 `providerExecuted` 时，即使还没有完整 tool-call 或结果，也会记录工具活动。[流事件处理][m-stream]

后一种情形可能意味着 provider 已开始执行外部操作；失败分支会以 `side_effects` 拒绝重试。
本地工具调用则在模型步骤成功结束后进入 `toolRuntime.settleToolCall()`，不应将“流中出现工具参数”直接等同于本地工具已执行。[工具交接][m-tools]
失败分支按顺序在安全门槛、预算、次数和错误策略之间作出决定；并非只依据 HTTP 状态码。[失败决策][m-gate]

## 4. 重试预算与退避：明确计数单位

每个逻辑 step 的 `MAX_PROVIDER_ATTEMPTS_PER_STEP = 10`，包含第一次调用，最多追加九次。
它并非指整个用户任务只有十次请求，也不是“失败后再试十次”。[次数与延迟][m-delay]

未收到有效服务端延迟提示时，`providerRetryDelayMs()` 使用：

```text
base = min(1000 × 2^(failedAttempt - 1), 32000) 毫秒
delay = ceil(base + random() × 0.25 × base)
```

因此首次等待约 1–1.25 秒，上限为 32 秒的 base；加抖动后可达约 40 秒。
有效 `retry-after-ms` 优先于 `retry-after`；后者支持秒数及 HTTP 日期，提示值不再套用该 32 秒 base 上限。[延迟提示][m-policy]
服务端提示只决定等待时长，不能将不允许重试的错误变为允许重试。

策略将网络、限流、暂时不可用、流截断、timeout 等归入可重试候选；鉴权、计费、请求拒绝等不属于这一类。
“候选”仍需经过上节安全门槛。上下文溢出进入另一个压缩恢复分支，并非普通退避循环的同义词。[策略][m-policy] [溢出分支][m-overflow]
分类测试还验证：计费额度耗尽不能仅因 HTTP 429 就视为限流重试，SDK 的可重试标志也不能覆盖所有本地策略。[分类测试][m-classification-test]
本章不由此推算总请求上限：不同层的预算、多个 step 和独立压缩恢复必须分别核算。

## 5. Watchdog：暂停计时、取消调用与超时的区分

`StreamWatchdog` 观察模型流活动，默认 connect 为 30 秒，idle 为 120 秒；调用方可以配置。
这里的 connect 是“尚未收到首个活动”，不是 TCP 握手的精确阶段，更不能证明请求尚未发送。[watchdog][m-watchdog]

| 操作 | 实际行为 | 边界 |
| --- | --- | --- |
| `start()` | 从当前时间安排 connect 定时器 | 每次物理 attempt 重新开始 |
| `markActivity()` | 转为 idle，并重置 idle 计时 | 持续活动的长流不因总时长被它终止 |
| `pause()` | 增加暂停计数并清掉定时器 | 不暂停远端模型或外部进程 |
| `resume()` | 所有嵌套暂停解除后调用 `markActivity()` | 重新获得完整 idle 窗口，不续用旧剩余秒数 |
| `stop()` | 清理定时器并永久停用这个实例 | 不等于发送业务撤销请求 |

工具执行路径接入 `getPermissionPauseTarget()`，通过 `pause()` / `finally resume()` 避免将权限等待误判为模型沉默。
模型流结束后 watchdog 会停止；它不是为所有本地工具、审批和远端业务提供的统一总超时。[工具暂停接点][m-pause]

实际触发 watchdog timeout 时，`startWatchdog()` 保存 `MODEL_STREAM_TIMEOUT`，中止的是本次 provider 请求的 controller。
这个失败随后仍需经过 retry 门槛；它不会被直接解释为用户主动停止。[超时回调][m-timeout]
`requestStop(..., 'immediate')` 才设置 Turn 的中止状态并传播取消信号；退避等待也使用 Turn 的信号，可被中断。
`after_step` 只登记停止意图，不立即 abort 正在进行的调用；后续循环检查会阻止继续派发。[停止入口][m-stop]

因此“30 秒超时”不是“整个任务最多 30 秒”，也不是“30 秒后远端一定停止”。
测试用假时钟验证了暂停期间不触发、嵌套暂停需配对解除，以及恢复后重新计算 idle 窗口。[计时测试][m-watchdog-test]

## 6. Host 命令：request ID 与业务身份

客户端 `#requestOperation()` 每次创建新的 `requestId`，用于匹配响应。
服务端只拒绝仍在执行中的重复 request ID，处理完成便移除；它不是持久化的结果去重账本。[客户端派发][m-request] [服务端处理][m-server]

下面把网络不确定性移到 Host 命令这一层；T 是同一个 root Turn 的稳定业务身份，R 是一次传输身份：

```text
客户端：命令(T, 原始输入) → 分配 requestId=R → queued → in_flight
网络：请求已发送，但最终响应没有返回客户端
Host：可能尚未接纳；也可能已经提交 root admission(T)
客户端：#retireRequest(R) → dispatched / timeout → 调用方收到异常
reconnecting client：命令不进入自动重试循环
调用方：先核对 T 的持久状态；不能仅凭超时生成新 T 并重复业务
```

`in_flight` 在调用底层写入前设置，因此 `dispatched` 是保守的派发边界，不是服务端收件或提交证明。
`#retireRequest()` 结束的是本地等待；它并未同时发送远端撤销命令。迟到的匹配响应由 retired 分支处理。[请求退休][m-retire]
异常区分 `not_dispatched` 与 `dispatched`，但命令分支本身都没有自动重新发送循环。
查询分支才针对连接丢失或 Host draining 重试；若指定 timeout，它跨重连共用一个 deadline，并非每次获得全新预算。[重连策略][m-reconnect]
这也不是“所有 query timeout 都会自动重试”：`isRetryableQueryInterruption()` 有明确的中断类别限制。

## 7. Root admission 的去重范围

`SqliteAgentRunStore.admitRootTurn()` 在写事务内，以 `(sessionId, turnId)` 查找已接纳记录。
已存在时，前序 `previousRootTurnId` 相同且规范化 payload 相等，返回 `existing`；否则返回 `conflict`。
新 admission 与来源消息 proof 在同一事务中写入；同一来源消息不能再以其他 Turn 身份占用。[接纳事务][m-admission]

这是存储接纳接口的精确条件，不能扩大为“任何重复 Host 命令都返回缓存结果”。
`RootAdmissionOwner.admitRootTurn()` 还维护链 tip；发生不确定异常时会将该 Session 标记为 poisoned，拒绝继续接纳。
测试特意令底层先提交再抛出异常：后续接纳被阻止，数据库中仍只有第一条 admission。[Owner][m-owner] [提交后异常测试][m-admission-test]

回到工单例子，实际危险的窗口仍然存在：

```text
T1：本地提交工具 intent → POST /tickets 已发送 → 服务端创建工单成功
响应丢失或本地崩溃 → T2：本地 outcome 尚未提交
恢复端只看到 intent，没有结果；无法据此判定工单不存在
```

当前 Maka 对新产生的这类未知工具结果按 `outcome_unknown` 边界处理，不自动重新执行原动作；详见 [持久化边界与恢复](../04-state/durable-execution.md)。
root admission 事务并不包含工单服务的数据库，也不会自动为其补充业务幂等键。
实际工具设计应让服务端识别稳定业务键，或提供按该键查询结果的接口；不能查证时需显式处置未知状态。
这些是接入工具时需要满足的条件，并非声称 Maka 已为任意 HTTP 工具实现该协议。
因此本章不宣称端到端 exactly-once：接纳只一次、请求可能多次、业务效果是否去重，是三个不同问题。

## 8. 与 Pi 对照：会话重试与 durable replay

Pi 默认 `coding-agent → agent → ai` 路径与实验性 `packages/durable` 不是同一套恢复实现。
职责背景见 [coding-agent 与 agent 的关系](../../coding-agent-agent-relationship.md)。

| 维度 | Maka 当前路径 | Pi 普通会话 | Pi durable 任务 |
| --- | --- | --- | --- |
| 控制入口 | `AiSdkTurn.runWithinScope()` | `AgentSession._handlePostAgentRun()` / `_prepareRetry()` | `ToolTask` 的 checkpoint 分支 |
| 再次进入的单位 | 同一步的 provider attempt | 失败响应处理后 `agent.continue()` | 持久化 task 的执行阶段 |
| 次数定义 | 每 step 最多 10 次，含首次 | `maxRetries` 是额外次数 | 工具恢复依赖 replay 策略，不等于该会话预算 |
| 失败片段 | transcript 保留，模型历史排除 | 当前工作区追加 `context_edit(null)` 排除投影 | checkpoint 保存参数及 replay 策略 |
| 副作用门槛 | provider 工具活动或 replay barrier 可拒绝重试 | 会话重试本身不提供外部业务去重协议 | 已存策略与当前工具策略都为 `safe` 才重跑 |
| 跨进程依据 | run / 工具账本及 root admission | 会话记录、上下文投影，不是该工具 checkpoint | task、checkpoint、memo 与 outcome |

Pi 普通会话在 `_handlePostAgentRun()` 判断失败，再由 `_prepareRetry()` 退避；成功准备后外层继续 agent 循环。
`_omitRecoveryAttempt()` 保留原始日志，以追加编辑更新持久化模型投影，而非仅从内存删除消息。[会话重试][p-session] [投影排除][p-omit]
`retryDelayMs()` 使用指数退避并受 `maxAgentDelayMs` 限制；当前函数没有抖动。取消退避由 `abortRetry()` 触发。[延迟函数][p-delay]
测试中 `maxRetries=2` 对应总计三次流调用；另有“重试后产生工具调用，prompt 等待完整循环”的断言。[会话测试][p-retry-test]
这些都是模型恢复行为，不能推导为任意失败工具会自动安全重新运行。

Pi durable 的工具 intent 默认 `unsafe`；恢复时要求保存的策略和当前定义都为 `safe`，否则以中断结果收束。
这避免升级后单方面放宽重放条件，但 `safe` 仍是工具作者的契约声明，而非运行时替远端证明幂等。[ToolTask][p-tool] [双策略测试][p-tool-test]
durable 的 transfer 测试使用稳定键 `transfer-${task.id}`；外部假服务按键去重。
SQLite 关闭重开后，服务调用计数为 2、实际效果集合大小为 1：效果仅一次源于稳定键和服务端去重，而不是代码只执行一次。[恢复测试][p-transfer-test]

## 9. 源码定位与阅读顺序

| 入口与函数 | 重点看什么 |
| --- | --- |
| [ai-sdk-turn.ts：`runWithinScope()` 的 attempt 循环][m-attempt] | 工具活动、可见文本、replay barrier 三种证据 |
| [ai-sdk-turn.ts：`providerRetryDelayMs()`][m-delay] | 十次含首次，base 上限与抖动分别计算 |
| [provider-retry-policy.ts：`providerRetryDecision()`][m-policy] | retryable 与 Retry-After 延迟分离 |
| [stream-watchdog.ts：`StreamWatchdog`][m-watchdog] | `start/markActivity/pause/resume/stop` 的状态变化 |
| [connection.ts：`#requestOperation()`、`#drainDomainRequests()`][m-request] | request ID 与 queued/in-flight 的传输身份 |
| [reconnecting-connection.ts：`#request()`][m-reconnect] | query 与 command 的分叉 |
| [agent-run-store.ts：`SqliteAgentRunStore.admitRootTurn()`][m-admission] | existing/conflict 与接纳事务边界 |
| [root-admission-owner.ts：`RootAdmissionOwner.admitRootTurn()`][m-owner] | 不确定提交后的拒绝继续接纳 |
| [agent-session.ts：`_handlePostAgentRun()`][p-session] | Pi 会话继续运行的决策点 |
| [durable/harness/tool.ts：`ToolTask`][p-tool] | 双 replay 策略与未知效果的处理 |

## 10. 本章术语

本章术语只在对应层次使用：

| 术语 | 本章含义 |
| --- | --- |
| attempt（尝试） | 对同一步模型请求的一次物理调用；失败后可能有下一次 |
| retry（重试） | 当前流程遇到可恢复失败，再次发起调用 |
| replay（重放） | 恢复时依据持久化意图重新进入某段执行代码；与评测篇的纯投影 replay 同名不同义 |
| 幂等 | 同一个业务操作重复提交，业务效果不增加；不是“函数只运行一次” |
| outcome unknown | 请求可能产生效果，但本地没有可信的最终结果 |
| root admission | Host 持久化接纳一个顶层 Turn 的身份、输入及前序关系 |

## 11. 源码练习

1. 手算第 1、2、6 次失败后的本地退避范围；再给定 `Retry-After: 60`，解释为何不能统一表述为“最多等待 32 秒”。对照 `providerRetryDelayMs()`，无需启动模型。
2. 阅读 watchdog 假时钟测试，画出 `start → pause → pause → resume → resume`。最后一次 resume 后是 connect 还是 idle？完整计时窗口从哪里开始？再说明为何这不能证明网络请求尚未发出。
3. 阅读 admission 提交后抛错测试与 Pi transfer 恢复测试，分别列出稳定身份、已提交事实和未知事实。若移除假服务的按键去重，为何“调用两次、效果一次”的结论不再成立？

## 相关专题

- [领域模型](../00-foundations/domain-model.md)：确认输入、执行、步骤和尝试的身份边界。
- [持久化与恢复](../04-state/durable-execution.md)：追踪状态提交与中断后继续执行的条件。

[m-attempt]: ../../../../maka/packages/runtime/src/ai-sdk-turn.ts#L1640
[m-stream]: ../../../../maka/packages/runtime/src/ai-sdk-turn.ts#L1758
[m-tools]: ../../../../maka/packages/runtime/src/ai-sdk-turn.ts#L2212
[m-gate]: ../../../../maka/packages/runtime/src/ai-sdk-turn.ts#L2146
[m-delay]: ../../../../maka/packages/runtime/src/ai-sdk-turn.ts#L586
[m-policy]: ../../../../maka/packages/runtime/src/provider-retry-policy.ts#L46
[m-overflow]: ../../../../maka/packages/runtime/src/ai-sdk-turn.ts#L2067
[m-classification-test]: ../../../../maka/packages/runtime/src/__tests__/provider-retry-classification.test.ts#L55
[m-projection]: ../../../../maka/packages/runtime-host/src/server/provider-retry-projection.ts#L24
[m-watchdog]: ../../../../maka/packages/runtime/src/stream-watchdog.ts#L20
[m-watchdog-test]: ../../../../maka/packages/runtime/src/__tests__/stream-watchdog.test.ts#L67
[m-pause]: ../../../../maka/packages/runtime/src/tool-runtime.ts#L1542
[m-timeout]: ../../../../maka/packages/runtime/src/ai-sdk-turn.ts#L1315
[m-stop]: ../../../../maka/packages/runtime/src/ai-sdk-turn.ts#L671
[m-request]: ../../../../maka/packages/runtime-host/src/client/connection.ts#L497
[m-server]: ../../../../maka/packages/runtime-host/src/server/connection-session.ts#L174
[m-retire]: ../../../../maka/packages/runtime-host/src/client/connection.ts#L912
[m-reconnect]: ../../../../maka/packages/runtime-host/src/client/reconnecting-connection.ts#L296
[m-admission]: ../../../../maka/packages/storage/src/agent-run-store.ts#L378
[m-owner]: ../../../../maka/packages/runtime-host/src/server/root-admission-owner.ts#L85
[m-admission-test]: ../../../../maka/packages/runtime-host/src/__tests__/root-admission-owner.test.ts#L34
[p-session]: ../../../packages/coding-agent/src/core/agent-session.ts#L1954
[p-omit]: ../../../packages/coding-agent/src/core/agent-session.ts#L1246
[p-delay]: ../../../packages/ai/src/utils/retry.ts#L141
[p-retry-test]: ../../../packages/coding-agent/test/agent-session-retry.test.ts#L140
[p-tool]: ../../../packages/durable/src/harness/tool.ts#L123
[p-tool-test]: ../../../packages/durable/test/harness-tools-recovery.test.ts#L136
[p-transfer-test]: ../../../packages/durable/test/harness-tasks-recovery.test.ts#L112

# 06. 失败以后，究竟重做哪一步

[返回学习地图](README.md) · 前置：[Agent 循环](01-agent-runtime.md)、[历史与上下文](02-context-state.md)

文件已经读完，模型解释到一半时断线，此时该重做解释，还是把文件再读一遍？如果换成付款工具，付款成功后进程却没来得及保存结果，问题又会不同。本篇沿源码提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc` 区分这些失败位置，再看重试、取消与重启恢复各自能做什么。

重试用于处理再次调用可能成功的失败，例如暂时断网；超时用于结束超过等待限制的调用；幂等用于约束重复调用的业务效果。三者要接在同一条失败处理流程里理解：先识别失败发生在哪一步，判断是否允许重做，再限制次数与等待时间；若重做会修改外部系统，还需要稳定的操作身份和去重协议。

以模型解释断线为例，完整过程是：程序得到失败回复，判断错误是否可重试，保存并省略失败尝试，等待一段退避时间，再用此前成功的工具结果重新请求模型，最后保存成功回复或报告耗尽。等待期间用户可以取消。这条流程保留失败历史，重做的是模型回复。

如果失败发生在付款结果返回前，程序连“付款失败”都不能确定。超时只说明客户端没有及时得到结果；远端付款可能已经成功。此时必须先判断工具的恢复策略，必要时查询外部操作状态。下面分别讲请求重试与这种未知效果的恢复，避免给所有错误套上同一个重试循环。

## 先认清重试与恢复各由谁处理

| 对象或层次 | 负责什么 | 出错后能做什么 |
| --- | --- | --- |
| 模型服务请求层（provider request） | 建立一次模型服务请求并处理传输错误 | 按异常、响应头和状态码决定是否重建请求 |
| 模型回复层（assistant response） | 检查最终模型回复中的错误 | 对可重试错误有限次退避后再生成 |
| `AgentSession` | 连接多次低层运行，保存失败回复的 `context_edit`（上下文编辑） | 退避后用 `Agent.continue()` 重做模型回复 |
| 实验 durable（持久执行运行时）的 `ToolTask` | 在工具执行前保存意图，执行后保存结果和任务终态 | 重启时按已提交阶段与重放策略决定是否再次运行 |
| 外部工具效果 | 修改文件、付款或调用远程服务 | 是否只发生一次取决于工具与外部系统的幂等协议 |

幂等表示同一操作重复执行时，最终业务效果与执行一次相同。外部系统必须参与去重或提供对应协议，单靠本地重试记录不能保证这一点。

所以要先找失败位置。请求没建立、回复没完成、外部效果已经发生但结果未保存，这几种状态能提供的证据不同。前两层可能再发模型请求；第三种不能仅凭本地存储判断副作用是否成功。一次用户输入也可能包含多次低层运行，各自的重试预算不会自动合成全局预算。

## 先沿一次失败的请求走完恢复过程

以“读文件并解释”为例，网络在第二次模型回复时断开：

```text
模型 #1 → read 调用 → read 成功 → toolResult 已保存
模型 #2 → 部分文本 → 网络 terminated → assistant error 已保存
Session 追加 context_edit，省略失败的 assistant #2
等待退避 → Agent.continue()
模型 #3 读取已有 read 结果，重新生成解释
```

此前成功的 `read` 结果仍在上下文里，Session 只需让模型重新生成失败的解释。低层循环在 `error/aborted` 后结束本次运行，不执行错误回复中的工具；Session 追加 `context_edit` 让失败回复退出投影，等待后继续。这次过程说明模型重试重做的是解释。

换成“付款”工具，假设进程在付款成功、结果保存之前退出。重新打开任务时，存储只知道执行意图已经提交，无法确认外部付款是否成功。这个未知结果窗口需要 durable 的重放策略和业务去重机制，普通模型重试处理不了它。

## 根据失败位置选择重做点

| 机制 | 已有实现 | 归类 |
| --- | --- | --- |
| 模型请求重试 | 按 HTTP 异常结构、响应头和状态码判断；可取消的等待 | 稳定库 |
| 模型回复重试 | 分类错误消息、有限次数、指数退避、回调 | 稳定库 |
| coding-agent 的会话重试 | 保存失败历史，追加省略条目，等待后调用 `Agent.continue()` | 常规集成 |
| 上下文恢复 | 溢出或可恢复截断走压缩后重试，和普通重试分开 | 常规集成 |
| 请求和 shell 超时 | 模型服务选项；shell 工具的可选超时和进程树终止 | 常规集成，语义由具体适配器决定 |
| 工具意图、结果与重启恢复 | durable 任务检查点、safe/unsafe 重放、嵌套调用复用 | 实验实现 |
| 任意外部副作用只发生一次（exactly-once） | 没有通用保证 | 需要具体工具与外部系统协议支持 |

表中“重试”是已知调用失败后再调用，“恢复”是重新打开持久存储，按已提交的任务状态继续处理。两者都可能再次执行代码，但业务效果能否只发生一次，要看工具与外部系统的协议。仅保存会话、只留下一个最终结果或成功恢复任务，都无法证明该协议已经保证 exactly-once。

## 请求没建立，与回复没完成

请求建立层和最终回复层都能重试模型调用，但它们看到的失败信号不同，次数预算与等待规则也不同。

### 模型请求建立失败时

Provider 是模型服务提供方；HTTP 状态码是服务对请求处理结果的编号，响应头可以携带重试指示或建议等待时间。请求建立层看到的是这些通信信息，因此能区别暂时过载、服务错误与某些不可重试响应。它还需要限制次数，否则长期故障会变成一直等待。

退避让连续重试之间留出时间，避免故障期间马上发出大量相同请求。指数退避按尝试次数增加等待，随机抖动让多个客户端的重试时间错开。服务端给出的 `Retry-After` 则表示它希望客户端等待多久。它们决定下一次何时发生，不证明下一次一定成功。

稳定库的 `retryProviderRequest()` 根据异常决定是否重新建立请求：这个判断面向带 `status`、`headers` 的 Error，先看 `x-should-retry` 是否明确指示重试与否；没有该指示时，无状态码的网络错误以及 408、409、429、5xx 都可重试，调用方还可排除指定状态码。判断逻辑见 [packages/ai/src/utils/provider-retry.ts:11-37](../../packages/ai/src/utils/provider-retry.ts#L11-L37)、[packages/ai/src/utils/provider-retry.ts:107-128](../../packages/ai/src/utils/provider-retry.ts#L107-L128)。

重试前要等待一段时间，这就是退避。这里优先读 `retry-after-ms`，其次读 `retry-after` 的秒数或日期；服务端没有指示时，等待从 0.5 秒增长到最高 8 秒，再乘 0.75 到 1 的随机系数。如果服务端要求的等待超过上限，函数直接抛错交给上层。默认上限为 60 秒，设为 0 则禁用这个限制。等待算法见 [packages/ai/src/utils/provider-retry.ts:39-69](../../packages/ai/src/utils/provider-retry.ts#L39-L69)。

OpenAI Responses 能展示这层重试的范围。SDK 设置 `maxRetries: 0`，由项目函数包住 `responses.create(...).withResponse()`；收到响应后才开始读取响应体。读取途中出错会生成最终 assistant error，交给上层处理，不再回到请求建立函数。因此，完全没收到响应与已经输出部分文本，可能采用不同的重试路径。对应适配器见 [packages/ai/src/api/openai-responses.ts:178-231](../../packages/ai/src/api/openai-responses.ts#L178-L231)。

还有一个默认值需要单独核对：这个函数未收到 `maxRetries` 时取 0，类型注释所说的 SDK 原生默认“两次”并不适用于这一层。阅读宿主配置时，要继续沿选项的传递路径查实际值；函数初始化见 [packages/ai/src/utils/provider-retry.ts:99-117](../../packages/ai/src/utils/provider-retry.ts#L99-L117)。

### 已经收到最终错误回复时

请求建立成功不代表回复一定完整。连接可能在输出中途断开，模型适配器把这个故障表达为最终 assistant 错误消息。此时重试层拿到的是消息里的结束原因和错误文本，未必仍有最初的 HTTP 状态码。源码因而提供另一套分类入口；这些规则属于不同失败位置，不能把次数和默认值混在一起。

当错误已经变成最终回复，稳定库用 `isRetryableAssistantError()` 判断：先要求 `stopReason=error` 且有 errorMessage，再按文本分类。配额、账单和订阅额度耗尽等被优先排除；过载、速率限制、5xx、DNS、连接中断、超时、提前 EOF 及明确要求重试的文本等可以匹配。它依靠错误文本启发式判断，无法保证所有模型服务都采用标准分类。模式与入口见 [packages/ai/src/utils/retry.ts:7-107](../../packages/ai/src/utils/retry.ts#L7-L107)、[packages/ai/src/utils/retry.ts:243-257](../../packages/ai/src/utils/retry.ts#L243-L257)。

`retryAssistantCall()` 只处理最终 AssistantMessage 中被判定可重试的错误，正常响应、aborted 和确定性错误直接返回。`maxRetries=3` 允许一次初始调用加三次重试；安排重试后，依次执行 `onRetryScheduled → sleep → onRetryAttemptStart → produce`，而 `onRetryFinished` 只在确实安排过重试时触发。若 `produce()` 本身抛异常，这个函数会让异常向外传播，没有通用捕获来把它转成消息后再试。控制流程见 [packages/ai/src/utils/retry.ts:191-241](../../packages/ai/src/utils/retry.ts#L191-L241)。

`length` 同样作为非 error 结果返回。到了摘要层，它会被判为不完整并拒绝保存，这个重试函数自身不会重新生成截断摘要。摘要检查见 [packages/coding-agent/src/core/compaction/compaction.ts:581-593](../../packages/coding-agent/src/core/compaction/compaction.ts#L581-L593)。

这层退避按次数增长，公式为 `baseDelayMs * 2^(attempt-1)`，每次等待不超过 `maxAgentDelayMs`，默认 60 秒。它没有请求层的随机抖动，上限设为 0 时也直接等待 0 秒；请求层的上限 0 则表示禁用限制。两个选项虽然都限制等待，具体语义要分别看。这里的计算见 [packages/ai/src/utils/retry.ts:109-132](../../packages/ai/src/utils/retry.ts#L109-L132)。

## Session 怎样重做解释并保留失败记录

Session 层把一次失败记入持久历史，再在下一次请求中省略失败回复并继续。

回到开头的网络中断例子，低层循环收到 error/aborted 后会结束本次运行，不执行这份错误回复中的工具。接着 `AgentSession._handlePostAgentRun()` 判断能否准备重试，准备成功后由外围循环调用 `agent.continue()`。所以一次 Session 输入可以跨过多次低层运行，先后收到多个 `agent_end`，最后才到 `agent_settled`。这两层的衔接见 [packages/agent/src/agent-loop.ts:245-270](../../packages/agent/src/agent-loop.ts#L245-L270)、[packages/coding-agent/src/core/agent-session.ts:1822-1891](../../packages/coding-agent/src/core/agent-session.ts#L1822-L1891)。

准备重试时，`_prepareRetry()` 增加尝试计数、检查预算、发出 `auto_retry_start`，再调用 `_omitRecoveryAttempt()` 并进入可取消的等待。省略操作会追加 `context_edit(targetId, null)`：失败回复留在 Session 树中，但退出模型投影。这份编辑也是持久历史的一部分，重新加载会话仍能重建同样的省略关系。准备与省略代码见 [packages/coding-agent/src/core/agent-session.ts:1237-1253](../../packages/coding-agent/src/core/agent-session.ts#L1237-L1253)、[packages/coding-agent/src/core/agent-session.ts:3758-3802](../../packages/coding-agent/src/core/agent-session.ts#L3758-L3802)。

摘要生成沿用同一套 Agent 重试策略。`completeSummarization()` 用 `retryAssistantCall()` 包住摘要调用，关闭摘要缓存写入，并转发摘要重试事件。不过，每次函数调用使用自己的局部预算，与对话这一轮的重试计数并不共享全局尝试次数。摘要调用与事件转发见 [packages/coding-agent/src/core/compaction/compaction.ts:612-639](../../packages/coding-agent/src/core/compaction/compaction.ts#L612-L639)、[packages/coding-agent/src/core/agent-session.ts:3715-3744](../../packages/coding-agent/src/core/agent-session.ts#L3715-L3744)。

### 请求过大时，要先改变上下文

重试需要针对错误原因改变条件。上下文过大意味着同一份输入已经超过模型允许的大小，等待再久也不会缩短它。这里先压缩消息，留下摘要与近期内容，再尝试新请求。失败尝试同时从模型投影中省略，防止不完整回复继续占用上下文或影响判断。正常回答后的阈值压缩则服务于下一次请求，已经完成的回答没有因此变成失败。

对上下文溢出，`_isRetryableError()` 会先排除普通重试，因为重复发送相同内容仍然过大。`_checkCompaction()` 改为对溢出或可恢复的 length 执行一次“压缩后重试”，并持久省略选定的失败 assistant 及其工具结果。再失败就报告恢复失败；如果回答已成功完成，只是 usage 越界，则仅压缩后续上下文，保留已经完成的回答。分类与恢复分支见 [packages/coding-agent/src/core/agent-session.ts:3705-3713](../../packages/coding-agent/src/core/agent-session.ts#L3705-L3713)、[packages/coding-agent/src/core/agent-session.ts:2977-3046](../../packages/coding-agent/src/core/agent-session.ts#L2977-L3046)。

工具失败则通常变成 `toolResult.isError=true`，交给模型决定后续行为。即使错误文本含“timeout”，它也不会因此进入 assistant 的错误分类；默认执行器将工具抛错转成结果，没有自动反复调用同一工具的机制。见 [packages/agent/src/agent-loop.ts:821-857](../../packages/agent/src/agent-loop.ts#L821-L857)。

![Session 先检查取消，再区分上下文压缩恢复与普通 assistant 错误重试的流程](figures/06-session-retry-flow.svg)

图里的省略、等待与继续属于普通 Session 的恢复流程；durable 调度器另有持久任务状态。

## 超时和取消会留下什么

超时达到配置限制后结束当前等待或请求；主动取消则向执行代码发出信号，需要对方配合。两者都无法说明外部操作是否已经发生。

| 位置 | 已有行为 | 仍需注意的限制 |
| --- | --- | --- |
| 模型请求 `timeoutMs` | 传给支持该选项的 SDK 或适配器 | 各种传输未采用统一的绝对截止时间 |
| 请求重试的等待 | AbortSignal 可以清掉等待计时器 | 无法证明原始请求尚未在服务端发生 |
| `Agent.abort()` | 触发当前运行信号 | 不配合信号的工具未必停止 |
| `AgentSession.abort()` | 标记取消，停止重试、压缩和分支摘要，等待空闲 | 不会回滚已完成的文件写入或远程调用 |
| shell `timeout` | 可选秒数，无默认超时；到时终止进程树并报告错误 | 终止前可能已产生外部效果 |

这些行为对应 [packages/ai/src/types.ts:156-210](../../packages/ai/src/types.ts#L156-L210)、[packages/ai/src/utils/provider-retry.ts:77-97](../../packages/ai/src/utils/provider-retry.ts#L77-L97)、[packages/agent/src/agent.ts:335-352](../../packages/agent/src/agent.ts#L335-L352)、[packages/coding-agent/src/core/agent-session.ts:2431-2451](../../packages/coding-agent/src/core/agent-session.ts#L2431-L2451)、[packages/coding-agent/src/core/tools/bash.ts:27-43](../../packages/coding-agent/src/core/tools/bash.ts#L27-L43)、[packages/coding-agent/src/core/tools/bash.ts:124-163](../../packages/coding-agent/src/core/tools/bash.ts#L124-L163)。

同样，两层重试一起开启时，每次上层尝试都可能再次经历下层请求重试，总请求数不能直接拿任何一层的 `maxRetries` 计算。练习时可以分别记逻辑上的 assistant 尝试和实际请求尝试，再判断整体预算是否符合宿主需要。

## 付款结果没保存，重启后该不该再执行

操作 ID 用来识别“这是同一项付款的再次尝试”。如果每次重试都新建一个付款 ID，外部服务就可能认为这是两项不同付款；稳定 key 必须跨重试保持相同，并由服务用它去重。本地任务 ID 与外部业务 key 也要明确如何对应，单独保存一个本地结果无法控制另一系统的写入。

幂等和 exactly-once 的含义也不同。幂等调用允许请求发生多次，但约定最终业务效果等同一次；exactly-once 是业务效果只发生一次的保证。运行时可以保存“已提交结果，直接复用”，却不能仅凭自己的数据库消除“外部成功，本地结果未保存”的未知窗口。safe/unsafe 正是工具作者为恢复时能否重跑提供的声明，下面按保存阶段解释其判断。

durable 通过执行意图与结果的提交边界记录重启时掌握的证据，再按重放策略决定是否重新运行工具。

实验 durable 把执行前的操作意图称为 intent，把执行后的结果与任务终态称为 outcome。`ToolTask` 从 `call` 开始：解析工具，准备并验证参数，运行 beforeTool，再提交 `phase=execute`、最终参数和重放策略。这个意图提交成功后，才真正执行工具。恢复时若已经处于 `execute`，保存的策略与当前工具策略必须同时为 `safe` 才会重跑，默认值为 `unsafe`；其他情况生成 interrupted 错误结果，提示工具可能已经部分运行。阶段切换见 [packages/durable/src/harness/tool.ts:64-67](../../packages/durable/src/harness/tool.ts#L64-L67)、[packages/durable/src/harness/tool.ts:123-185](../../packages/durable/src/harness/tool.ts#L123-L185)。

| 崩溃位置 | 存储知道什么 | 恢复行为与风险 |
| --- | --- | --- |
| intent 提交之前 | checkpoint 仍为 call | 可重跑准备和 beforeTool；hook 自己的外部效果也要考虑重复 |
| intent 后、execute 前 | 只知道打算执行 | safe 可重跑；unsafe 会报告未知，可能实际尚未执行 |
| execute 产生效果后、outcome 前 | 同样只有 execute intent | safe 可能再次执行；unsafe 不重跑，但无法判定效果是否成功 |
| outcome 提交之后 | 结果条目和任务终态已提交 | 恢复读取已保存结果，无需再执行这个已结束任务 |

从存储看，最终工具结果与任务终态确实在 `settle()` 的同一次运行时提交中写入。不过，实际 `tool.execute()` 已经在此前执行，远程 HTTP 调用和 shell 效果没有包含在这次存储事务中。表格最后一行保证的是恢复复用已保存结果，无法把外部效果也变成 exactly-once。执行与提交的位置见 [packages/durable/src/harness/tool.ts:391-428](../../packages/durable/src/harness/tool.ts#L391-L428)、[packages/durable/src/harness/tool.ts:570-630](../../packages/durable/src/harness/tool.ts#L570-L630)。

嵌套调用会按父任务下的 key 复用此前创建的工具任务。相同 key 必须对应相同工具和参数，否则报错；默认 key 来自调用序号，如果重跑时控制流可能改变调用顺序，显式稳定 key 更便于表达操作身份。这能避免重复创建已有子调用，外部效果仍需要子工具自身具备幂等性，也就是同一个操作重复执行时，最终业务效果与执行一次相同。复用规则见 [packages/durable/src/harness/tool.ts:281-285](../../packages/durable/src/harness/tool.ts#L281-L285)、[packages/durable/src/harness/tool.ts:360-387](../../packages/durable/src/harness/tool.ts#L360-L387)、[packages/durable/src/harness/tool.ts:656-718](../../packages/durable/src/harness/tool.ts#L656-L718)。

配置时还要分清两套 API：durable 使用 `safe/unsafe`，AgentTool 类型使用 `safe/never`。常规 agent-loop 不会读取重放策略来做崩溃恢复，所以后一套类型不能直接控制上面的恢复流程。AgentTool 定义可对照 [packages/agent/src/types.ts:479-498](../../packages/agent/src/types.ts#L479-L498)。

## 边界与未实现

以上机制覆盖了模型与会话层的重做；以下部分仍需宿主或业务层另行安排：

- 会话重试的运行状态只在内存。省略关系可以持久重建，尝试计数和等待中的 AbortController 却仍是内存字段；重新打开普通 JSONL 会话能重建上下文，无法自动恢复当时的退避计时，也无法据此接管未完成工作流。相关运行字段见 [packages/coding-agent/src/core/agent-session.ts:402-417](../../packages/coding-agent/src/core/agent-session.ts#L402-L417)。
- 超时与取消各管一段时间，目前没有统一的“整个用户任务最多 N 秒”机制，全链路总重试预算也尚未实现。
- 付款、发信或部署工具要允许重放，具体实现就得定义操作 ID、外部去重、状态查询和未知结果对账。当前代码没有通用的业务幂等键生成与对账层，也无法把远程效果纳入同一个本地事务；仅设置 replay=safe 无法补上这些能力。这些业务协议是重放有副作用工具所需的正确性条件，普通只读练习无需先实现完整分布式事务。

## 源码位置与术语速查

### 文中术语

| 术语 | 在这个项目中的含义 |
| --- | --- |
| 重试 | 已知调用失败后再次调用 |
| 恢复 | 重新打开持久存储，按已提交的任务状态继续处理 |
| 退避 | 重试前的等待；请求层带随机抖动，assistant 层按次数指数增长 |
| exactly-once | 业务效果只发生一次的保证；需要工具与外部系统协议支持 |
| 幂等 | 同一个操作重复执行时，最终业务效果与执行一次相同 |
| intent / outcome | durable 中执行前提交的操作意图 / 执行后提交的结果与任务终态 |
| replay policy | durable 工具的可重放策略，safe/unsafe；AgentTool 类型另有 safe/never 一套 |
| `context_edit` | 会话重试时追加的省略条目，让失败回复退出模型投影但保留在历史树中 |

## 测试与练习

| 目标 | 已有测试 |
| --- | --- |
| quota 不重试，取消和次数预算正确 | [packages/ai/test/retry.test.ts:137-285](../../packages/ai/test/retry.test.ts#L137-L285) |
| 服务端等待超限与 AbortSignal | [packages/ai/test/provider-retry.test.ts:48-88](../../packages/ai/test/provider-retry.test.ts#L48-L88) |
| Session 多次 agent_end、最终成功或耗尽 | [packages/coding-agent/test/suite/agent-session-retry-events.test.ts:33-123](../../packages/coding-agent/test/suite/agent-session-retry-events.test.ts#L33-L123) |
| 重试后返回工具调用，prompt 等到整条循环完成 | [packages/coding-agent/test/suite/agent-session-retry-events.test.ts:196-227](../../packages/coding-agent/test/suite/agent-session-retry-events.test.ts#L196-L227) |
| unsafe 不重跑、safe 必须同时符合保存与当前策略 | [packages/durable/test/harness-tools-recovery.test.ts:108-163](../../packages/durable/test/harness-tools-recovery.test.ts#L108-L163) |
| intent 前 hook 可重复，但 memo 决策复用 | [packages/durable/test/harness-tools-recovery.test.ts:213-250](../../packages/durable/test/harness-tools-recovery.test.ts#L213-L250) |

把网络中断例子改成“read 成功、下一条 assistant 连续两次失败、第三次成功”，先预测原始历史、投影与调用次数，再用 suite/harness 的 faux responses 验证，观察两次省略是否都保留在原始 Session 树里。随后读 durable 恢复测试，把 `started → close → reopen` 画到提交边界上，解释 runs=1 与 runs=2 分别怎样对应 safe/unsafe 策略。练习无需真实模型服务。

若要运行，在 `packages/ai` 下使用 `node ../../node_modules/vitest/dist/cli.js --run test/retry.test.ts test/provider-retry.test.ts`；在 `packages/coding-agent` 下指定 `test/suite/agent-session-retry-events.test.ts`，在 `packages/durable` 下指定 `test/harness-tools-recovery.test.ts -t "reruns a tool only"`。这些定向用例使用模拟错误、faux provider 或本地 SQLite，不会向真实模型 API 发送请求。

继续读[持久执行](05-durable-execution.md)看任务检查点与恢复，读[执行环境与隔离](08-sandbox-environment.md)看执行效果的边界，读[评估与回放](12-evaluation-replay.md)学习如何验证失败窗口。

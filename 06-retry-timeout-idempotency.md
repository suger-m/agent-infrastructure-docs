# 第 6 章　重试、超时与幂等

[返回目录](README.md) · [上一章：第 5 章 持久执行与恢复](05-durable-execution.md) · [下一章：第 7 章 分布式状态与一致性](07-distributed-state.md)

重试可以恢复暂时失败的操作，也可能重复已经发生的外部效果。超时只说明等待超过限制，幂等协议则规定重复请求如何复用同一业务结果。本章先分析错误所能提供的证据，再确定重试范围、次数与时间预算，以创建工单为例说明幂等和结果核对，最后对照 Pi 的请求层、Session 层及实验 durable 工具恢复。

用户要求 Agent 阅读故障报告并创建工单。正常流程包括读取报告、生成内容、发送创建请求、取得工单编号，再保存并交付结果。若工单已创建而返回连接中断，客户端看到的请求失败与服务端已有工单可以同时成立。重新创建可能产生重复工单；直接结束则无法交付原工单编号。恢复程序需要根据已保存的操作记录，决定是等待重试、修正参数、复用结果，还是保留待核对状态。

下面先用创建工单讲通用设计。示例中的操作记录、全局截止时间和服务端幂等协议是教学架构，不能直接当成 Pi 默认已有的功能。后半章再对照 Pi 的普通 Session 重试与实验 durable 路径。相关前置内容见[第 1 章 Agent 运行时](01-agent-runtime.md)与[第 2 章 上下文与状态](02-context-state.md)。

## 6.1 临时失败、永久失败与结果未知

临时失败是故障条件可能自行消失的失败。例如，工单服务明确返回“当前过载，请稍后再试”，且协议规定这种拒绝发生在创建之前。调用方由此知道本次未创建工单；负载下降后，相同参数仍有机会成功。重试（retry）是在这一判断下再次执行选定操作。

永久失败表示在当前请求和条件不变时，重复尝试不能解决问题。“项目不存在”需要修正项目编号或补充信息，等待不会改变该参数。权限不足、参数不合法、余额或配额耗尽也应按原因处理。“永久”并不表示服务永远不可用，而是说明自动重发当前请求缺乏依据。

结果未知表示调用方没有足够证据判断操作是否完成。请求发出后没有响应，既可能是服务未收到请求，也可能是工单已创建而响应丢失。网络中断可以是临时故障，但创建操作仍可能已经成功。错误分类因而包含两个独立判断：再次尝试是否可能成功，以及前一次是否可能已经产生效果。

状态码和错误文本只是判断材料，最终依据是接口协议和操作语义。服务明确拒绝创建能够证明的内容，多于一条笼统的“连接关闭”。为支持后续恢复，调用方至少应保存操作标识、最终参数、尝试过程和收到的结果；单一失败标记无法区分“明确未执行”与“可能已执行”。

## 6.2 重试范围与重复风险

工单流程中的读取、模型生成和外部创建具有不同的重试边界。读取暂时失败时，可重做读取；解释报告的模型输出中断时，可重新生成解释；创建结果未知时，则需先核查创建操作。如果统一从任务起点重跑，已经成功的步骤也会再次执行。

重试范围可以是一次网络请求、一轮模型生成、一个工具操作或整个工作流。设计时应列出范围内将再次运行的代码，并复用范围外已确认的结果。模型重新生成可能改变工单标题、内容或所属项目，因此同一创建操作的参数宜在首次发送前固定。参数发生业务含义上的变化后，应重新判断它是否仍属于原操作。

只读操作不主动修改外部状态，通常更容易重复，但两次读取仍可能得到不同版本。写操作是否安全需要结合具体效果判断：重复“将状态设为关闭”可能不改变最终状态，重复“新增工单”则可能创建多张记录。工具名和 HTTP 方法都不足以独立证明重试安全。

## 6.3 次数预算、超时与截止时间

重试策略需要同时约束尝试次数、单次等待和总持续时间。假设初次调用之外最多再试两次，单次超时（timeout）为 8 秒，任务总截止时间（deadline）为开始后的 20 秒。次数限制请求数量，超时限制一次等待，截止时间覆盖整项任务。这里的数字仅用于演示预算关系。

若前两次各等待 8 秒，其间退避花费 2 秒，第三次开始前仅剩 2 秒。调用方应将这 2 秒作为下层可用时间，或因剩余预算不足而结束恢复；不能重新分配完整的 8 秒。排队、退避和子调用都需要计入总时间，否则每一步满足局部超时，任务总体仍可能超期。

次数与时间限制分别处理不同的消耗。快速返回的错误可能在短时间内触发大量请求，需要次数上限；少数缓慢请求则需要截止时间。每次尝试前，程序应检查可重试条件、剩余次数、剩余时间和取消状态。跨进程恢复还需要保存相关预算，防止重启重新获得一份完整额度。

退避（backoff）是在失败后延迟下一次请求，用于减少故障期间的额外负载。指数退避逐次增加间隔，例如约 1 秒、2 秒、4 秒，并设定最大值。较短的初始间隔有助于尽快恢复，后续较长间隔则避免服务持续过载时收到密集重发。

相同的退避间隔可能让大量客户端同时恢复发送。随机抖动（jitter）通过调整各次等待，分散请求到达时间。服务给出等待提示时，程序还应与剩余截止时间比较：建议等待超过预算，就结束本次自动恢复。退避降低额外负载的代价是延长恢复时间，持续超出服务容量的流量仍需另行控制。

分层重试会使尝试次数相乘。外层总共尝试 3 次，每次内部又最多尝试 3 次，持续失败就可能产生 9 次请求。更多层叠加时，重发流量可能进一步加重故障，形成重试风暴。各层应明确负责的故障种类，传递总截止时间，并约束整体请求量；仅有各自的局部上限，无法保证全局消耗受控。

下图将排队、请求和退避放在同一时间线上。每一步都消耗总预算，下一次调用只能使用剩余部分；示意时长不属于 Pi 默认设置。

![通用教学图：排队、请求和退避共同消耗 deadline，剩余预算控制下一次调用](figures/06-concept-deadline-budget.svg)

## 6.4 超时与取消后的操作状态

工单服务在第 7 秒保存记录，客户端在第 8 秒超时，是一种可能发生的时序。客户端停止等待不会撤销服务端提交；即使超时时服务尚未完成，它也可能继续处理。超时因而只能证明等待到期，不能证明业务操作未发生。

取消是调用方主动请求停止执行，可以由用户操作或总截止时间触发。取消信号需要传递到退避等待、模型请求和正在运行的工具：等待逻辑收到后不再安排下一次尝试，本地工具停止并清理资源，远端服务则按其协议和实现响应。

各层必须实际处理取消，才能停止对应工作。丢弃等待结果不一定终止计算，结束 shell 也无法收回已发送邮件或已写入文件。远端成功与取消可能交错发生，所以用户停止等待后，系统仍可能需要保存迟到结果，或安排恢复与核对。

对于尚未确认的创建操作，状态应记录为“停止等待，创建结果待确认”。保留原操作身份及未知状态，可防止后续执行者将它误当成尚未发起的新请求。

## 6.5 幂等键与服务端结果记录

幂等（idempotency）表示同一操作重复执行后，业务效果与执行一次相同。新增工单通常每次都会产生新记录；服务若支持幂等键，调用方就可以用 `create-incident-42` 之类的稳定标识表示本次创建意图，并在首次发送前连同最终参数保存。断线后的重试继续使用原键，用户需要另一张工单时再创建新键。

幂等键的生命周期应覆盖一项业务操作的全部尝试。每次网络请求生成新键，服务便无法识别重复；将某个用户的所有请求固定为同一键，又会错误合并其后续合法操作。连接重建、进程重启或模型重新回复，都不应独自改变原业务操作的标识。

服务需要明确键的去重范围，例如同一租户的同一接口，而不能假定它跨账户、跨服务全局有效。同时还应核对参数：相同键携带不同项目或内容时，需要拒绝或报告冲突，避免把含义不同的请求关联到旧结果。

若工单和幂等记录都由同一服务保存，可在一次原子提交中登记键、创建工单并保存返回结果。原子性要求这些更新共同生效或共同失败。重复请求若命中已完成记录，就取得原工单编号；若原请求仍在处理中，则等待或返回可查询的状态。并发收到相同键时，服务也必须协调创建，不能让两个请求各自查询为空后分别插入。

分别提交去重记录和工单会重新引入崩溃窗口。先创建工单后记键，故障可能导致下一次重复创建；先记键后创建，故障可能留下没有工单的去重记录。如果效果发生在另一个外部系统，本地数据库事务也无法覆盖该次请求，仍需下游幂等协议或结果核对。保证范围必须与实际参与协议的系统一致。

服务保存原工单编号及返回内容，使调用方能够在重试时取回第一次结果。去重记录的保留期限需要覆盖允许的重试和恢复窗口；记录过期后，旧键可能再次被当作新操作。超出期限的未知结果应改用查询或人工核对。

完整恢复流程由调用方和服务共同完成。调用方先保存键与参数，再发送请求；服务原子保存工单和结果，但响应途中断线。调用方保留原键，检查预算并退避，然后查询操作状态或按协议重发同键请求。服务返回原工单编号，调用方保存该结果并交付给用户。

下图展示响应丢失后的同键重试。服务端已有结果记录，使第二次请求能够返回原工单，而不重复创建。

![通用教学图：使用稳定幂等键重试创建请求，服务复用原工单结果](figures/06-concept-idempotent-retry.svg)

## 6.6 缺少去重接口时的未知状态处理

缺少幂等键时，服务可能仍允许按调用方提供的唯一业务编号查询。找到匹配结果后可以复用；查询为空则需要检查读取语义。如果新记录延迟可见，或者原请求仍在执行，空结果无法证明创建未发生，此时再次创建仍可能重复。只有查询协议提供足够证据，恢复程序才能自动重做。

邮件发送也需要服务端证据。支持稳定发送 ID、去重和结果保存的服务，可以按协议恢复；本地“准备发送”记录无法证明邮件已经送达。结果未知时应先查询服务侧记录，仍无法确认则保留收件人、内容摘要、发送 ID 和时间，由人工判断是否补发。

`git push` 也要按具体结果判断。将同一个提交推到同一个远端引用，在引用已到达目标时，再次推送通常不会再创建一份提交。但连接中断后，仍需读取远端引用，核对预期提交是否已经到达；远端引用也可能又被其他人推进。发布脚本如果还会创建资源、发通知或触发外部动作，应分别核对这些效果，不能从引用更新的状态推出整段脚本都安全完成。创建云资源同样应优先按操作 ID 或唯一标识查找原资源，避免换一个随机名字再创建。

预算耗尽、去重记录过期或查询证据不足时，自动重试应停止并转入对账。对账将本地操作记录与外部事实逐项比较，再决定复用结果、补做或处理重复。若需要撤销或删除已产生的重复项，这些动作仍是新的业务操作，需要另行确定权限和后果。

## 6.7 参考实现：Pi Session 的模型生成重试

Pi 默认 CLI 的自动重试主要可以从模型生成层理解。假设模型已调用 read，工具成功并将 `toolResult` 写入会话；随后模型开始解释文件，连接在输出中途断开。低层循环收到以 error 结束的 assistant 消息后结束本次运行，并且不执行该错误回复包含的工具调用。

Session 在运行结束后检查 assistant 错误及剩余预算。此次失败范围是解释文件的模型调用，之前的 read 结果仍有效。Session 保存失败回复，并追加 `context_edit`，使该回复退出下一次请求的上下文投影。投影是从已保存历史中选取并转换本次模型输入的过程。失败历史仍可查看，未完成解释则不再作为有效回答发送给模型。

Session 记录尝试次数后进入可取消的退避等待。错误不符合重试条件、预算耗尽或等待被取消时，流程结束；等待完成且运行仍有效时，外围循环调用 `Agent.continue()`。模型从已有 read 结果重新生成解释，自动恢复不会重复执行该读取，不过模型仍可在新回复中主动提出再次读取。

新请求成功后，这条恢复路径结束；若又收到可重试错误，Session 会按同一规则继续，直到成功、取消或耗尽预算。一次恢复的记录顺序如下：

```text
模型 #1 → read 调用 → read 成功 → toolResult 已保存
模型 #2 → 部分文本 → 网络 terminated → assistant error 已保存
Session 追加 context_edit，省略失败的 assistant #2
等待退避 → Agent.continue()
模型 #3 读取已有 read 结果，重新生成解释
```

![Session 先检查取消，再区分上下文压缩恢复与普通 assistant 错误重试的流程](figures/06-session-retry-flow.svg)

图中包含取消、上下文压缩恢复和普通 assistant 错误重试三条处理分支。read 示例采用最后一条路径，压缩恢复条件见 6.11.2 节。

## 6.8 参考实现：三类错误的分层处理

建立请求、消费模型响应和执行工具是三个不同阶段，错误由所在阶段的代码处理。即使错误文本都含“超时”，其重试范围也可能分别是建立网络请求、重新生成 assistant 回复，或让模型处理一条工具错误结果。

### 6.8.1 请求未建立成功

Provider 是模型服务提供方。请求层在拿到模型回复之前就可能遇到网络错误或服务端状态码。`retryProviderRequest()` 会按状态和响应头决定是否重新建立请求，并在尝试之间退避。这个重试发生在一次 assistant 回复形成之前，具体状态码、头字段顺序与等待上限见 6.11.1 节。

如果连接在响应到达前断开，本地没有收到回复；服务端是否已经处理请求，取决于断开时请求传到了哪一步。请求层重试能处理短暂故障，不能单凭本地错误证明服务端没有收到请求。

请求一旦收到响应，读取响应体时再发生的中断会变成最终 assistant error，交给上一节所说的 Session 路径。OpenAI Responses 适配器会关闭 SDK 自带重试，由项目的请求重试函数处理建立请求阶段；响应体读取失败不会返回给该函数，因此不会继续走同一层请求重试。

### 6.8.2 模型回复以错误结束

assistant 回复中的可重试错误由 assistant 重试策略分类。Session 在整次运行结束后再检查这条错误，必要时记录失败、从后续投影中省略并继续请求。配额或账单耗尽等确定性问题会被排除；错误文本启发式可能识别过载、限速、5xx、DNS、连接中断、超时和提前 EOF。错误描述因服务而异，所以文本分类不保证覆盖所有服务。

这条路径重做模型生成，不会自动重放之前成功的工具。请求层和 assistant 层各有预算，一次 Session 恢复可能在每次生成中再触发多次底层请求；项目没有统一的整项用户任务时限或总重试预算。

### 6.8.3 工具返回 isError

普通工具失败会作为 `toolResult.isError=true` 交给模型。模型可以根据错误结果选择解释、修正参数或提出新的工具调用；常规 Agent 循环不会因此自动重复执行同一工具。错误文本里即使有“timeout”，它仍是工具结果，不会因此变成 assistant 错误。

## 6.9 参考实现：普通 Session 的取消与重启边界

Pi 的取消信号可以中止等待和正在配合取消的执行路径。Shell 超时可以终止进程树，但进程退出之前可能已经写文件或调用服务。因而这里也需要前面讲过的证据边界：本地停止后，外部效果仍可能需要查询。

普通 `AgentSession` 将重试计数和等待用的 AbortController 保存在内存。重新打开 JSONL 能重建历史与省略关系，却不会恢复原退避计时，也不会接管崩溃前的工作流。项目未提供统一覆盖整项用户任务的总时限；请求与 assistant 的局部预算也没有合并为全局重试预算。

## 6.10 参考实现：durable 对未知结果的处理

若需要进程重启后恢复工具任务，可以对照 Pi 的实验 durable 路径；它与重新打开普通 JSONL 会话是不同机制。沿用创建工单的例子，服务已经完成创建，本地进程却在保存工具结果前退出。恢复者看到的执行记录，必须把“想要执行”和“已经取得结果”区分开。付款工具也有相同窗口：付款成功后、本地保存结果前退出，重跑可能重复付款，直接跳过又可能漏付。此类业务仍需要稳定操作 ID、外部服务去重或对账接口。

durable（持久执行运行时）把工具调用拆成执行意图 `intent` 和执行结果 `outcome`。意图记录说明运行时准备按哪些最终参数执行，结果记录保存运行时已取得的返回值。运行时先保存最终参数与重放策略，再执行工具，完成后保存结果和任务终态。崩溃后系统根据最后一个已提交的阶段判断能做什么：

| 崩溃位置 | 本地存储能证明什么 | 恢复时的行为 |
| --- | --- | --- |
| intent 提交之前 | 仍在 `call` 阶段 | 可重新准备参数和运行 beforeTool；hook 自身副作用也可能重复 |
| intent 之后、execute 之前 | 只知道计划执行 | safe 可重跑；unsafe 报告未知，即使工具可能尚未开始 |
| execute 产生效果后、outcome 之前 | 仍只有 execute intent | safe 可能重复效果；unsafe 不重跑，但无法确认效果是否成功 |
| outcome 已提交 | 结果和任务终态已保存 | 复用已保存结果，不再执行已结束的任务 |

恢复到 `phase=execute` 时，只有已保存的重放策略和当前工具策略都为 `safe`，durable 才会重跑。两份策略分别来自原执行意图和恢复时的工具定义；检查两者可以避免仅凭后来放宽的策略去重跑原先没有承诺安全的操作，也避免在当前策略已收紧时沿用旧许可。默认策略是 `unsafe`，此时会返回 interrupted 错误并提示效果可能已经部分发生。即使真实情况是工具尚未开始，只有 intent 的本地记录也无法证明这一点。

`safe` 是工具作者允许重放的声明，运行时据此决定是否再次调用。要把创建工单声明为安全，工具作者仍须保证重复执行不会多建工单，例如把原来保存的业务操作键继续传给支持去重的服务。声明本身不会创建服务端幂等协议；错误地标为 `safe` 仍可能产生重复效果。

结果和任务终态在 `settle()` 的同一次本地提交中写入，但 `tool.execute()` 以及外部 HTTP 或 shell 效果发生在这次提交之前。事务只保护本地结果记录，不会把外部操作纳入本地事务。工程讨论中的 exactly-once 通常要求在声明的范围内只有一次业务效果；调用尝试仍可能发生多次，要说明重复效果由哪一层排除。Pi durable 不提供通用 exactly-once 保证，未知的外部效果仍要按业务协议恢复。

## 6.11 参考实现：重试参数、事件与嵌套调用

### 6.11.1 请求重试的判断与退避

`retryProviderRequest()` 先检查 `x-should-retry` 的明确指示；没有该指示时，无状态码网络错误以及 408、409、429、5xx 可重试，调用方还可排除状态码。退避优先采用 `retry-after-ms`，再看 `retry-after` 的秒数或日期；没有服务端提示时，从 0.5 秒指数增长到 8 秒，并乘以 0.75 到 1 的随机系数。服务端等待超过默认 60 秒上限会直接抛错；将上限设为 0 会禁用这个限制。

OpenAI Responses 适配器将 SDK 的 `maxRetries` 设为 0，再由项目函数调用 `responses.create(...).withResponse()`。收到响应后才读取响应体；读取中途失败会成为最终 assistant error，而不会返回请求建立函数。函数未收到 `maxRetries` 时取 0；类型注释提到的 SDK 原生默认两次不适用于这一层。

### 6.11.2 assistant 重试预算与上下文压缩

`retryAssistantCall()` 只处理最终 `AssistantMessage` 中被判为可重试的错误。正常回复、aborted 和确定性错误直接返回。`maxRetries=3` 表示一次初始调用加最多三次重试；每次重试依次触发 `onRetryScheduled → sleep → onRetryAttemptStart → produce`。只有确实安排过重试时才触发 `onRetryFinished`。若 `produce()` 本身抛异常，函数会向外传播，不会把异常变成消息再试。退避为 `baseDelayMs * 2^(attempt-1)`，每次不超过 `maxAgentDelayMs`，默认 60 秒；这里没有请求层的随机抖动。此处上限设为 0 表示等待 0 秒，与请求层的 0 不同。

上下文过大时，重复发送相同输入仍会失败。Session 会执行一次上下文压缩，再尝试恢复请求，失败 assistant 及其工具结果会按恢复分支省略；再失败则报告恢复失败。若回答已经成功，只是 usage 越界，压缩的是后续上下文，已完成回答仍保留。普通 `length` 结果不是 assistant error；摘要层会判定它不完整并拒绝保存，assistant 重试函数本身不会重新生成截断摘要。

### 6.11.3 Session 历史与事件

低层循环遇到 error 或 aborted 后结束本次运行，不执行错误回复中的工具。`AgentSession._handlePostAgentRun()` 决定是否重试；外围循环准备成功后调用 `agent.continue()`。因此，同一个 Session 输入可以经历多次 `agent_end`，最后才发出 `agent_settled`。

`_prepareRetry()` 增加尝试计数、检查预算、发出 `auto_retry_start`，调用 `_omitRecoveryAttempt()`，再进入可取消等待。省略操作追加 `context_edit(targetId, null)`：失败回复仍在 Session 树中，但退出模型投影。摘要生成也使用 assistant 重试策略，不过有自己的局部预算，不与对话尝试合并。

### 6.11.4 durable 嵌套调用

嵌套工具按父任务下的 key 复用已有子任务。例如，父任务准备完报告后调用一个创建工单的子工具；父任务恢复时用相同 key 找回这个子任务，就能复用已保存的完成结果。相同 key 必须始终对应相同工具和参数；默认 key 来自调用顺序，若重试时控制流可能变化，显式稳定 key 更可靠。

这个 key 的去重范围是父任务内的子任务记录。若子任务只有执行意图、尚未保存工单编号，找回子任务仍然会遇到外部创建结果未知的问题，需要继续按重放策略和外部服务协议处理。因此它避免重复创建同一子调用，不能替代外部服务的幂等协议。durable 的 `safe/unsafe` 重放策略也不同于 `AgentTool` 类型的 `safe/never`；常规 Agent 循环不会读取后者来做崩溃恢复。

## 6.12 源码定位与延伸阅读

- 请求重试判断：[provider-retry.ts 判断](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/provider-retry.ts#L11-L37)、[状态码判断](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/provider-retry.ts#L107-L128)、[退避](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/provider-retry.ts#L39-L69)、[初始化默认值](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/provider-retry.ts#L99-L117)、[等待与取消](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/provider-retry.ts#L77-L97)
- Provider 实例：[OpenAI Responses](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/api/openai-responses.ts#L178-L231)、[模型请求选项](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/types.ts#L156-L210)
- assistant 重试：[错误分类](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/retry.ts#L7-L107)、[分类入口](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/retry.ts#L243-L257)、[控制流程](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/retry.ts#L191-L241)、[退避计算](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/src/utils/retry.ts#L109-L132)
- Session 与压缩：[低层循环](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/agent/src/agent-loop.ts#L245-L270)、[Session 继续](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L1822-L1891)、[准备重试](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L1237-L1253)、[省略历史](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L3758-L3802)、[摘要重试](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/compaction/compaction.ts#L612-L639)、[事件转发](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L3715-L3744)、[摘要长度检查](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/compaction/compaction.ts#L581-L593)、[错误分类](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L3705-L3713)、[压缩恢复](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L2977-L3046)、[普通工具结果](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/agent/src/agent-loop.ts#L821-L857)、[运行时字段](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L402-L417)
- durable 与工具：[类型策略](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/agent/src/types.ts#L479-L498)、[工具阶段](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/tool.ts#L64-L67)、[意图提交](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/tool.ts#L123-L185)、[执行与结果](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/tool.ts#L391-L428)、[settle](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/tool.ts#L570-L630)、[嵌套调用 key](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/tool.ts#L281-L285)、[key 复用](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/tool.ts#L360-L387)、[子任务复用](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/harness/tool.ts#L656-L718)
- 超时实现：[Agent 取消](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/agent/src/agent.ts#L335-L352)、[Session abort](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/agent-session.ts#L2431-L2451)、[shell 默认与设置](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/tools/bash.ts#L27-L43)、[shell 进程终止](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/core/tools/bash.ts#L124-L163)
- 测试：[retry](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/test/retry.test.ts#L137-L285)、[provider retry](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/ai/test/provider-retry.test.ts#L48-L88)、[Session retry](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/test/suite/agent-session-retry-events.test.ts#L33-L123)、[重试后工具调用](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/test/suite/agent-session-retry-events.test.ts#L196-L227)、[durable 恢复](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/test/harness-tools-recovery.test.ts#L108-L163)、[intent 前 hook](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/test/harness-tools-recovery.test.ts#L213-L250)
- 相关章节：[第 5 章 持久执行与恢复](05-durable-execution.md)、[第 8 章 沙箱与执行环境](08-sandbox-environment.md)、[第 12 章 评估与回放](12-evaluation-replay.md)

## 6.13 本章术语

| 术语 | 定义 |
| --- | --- |
| 重试范围 | 一次尝试将重新执行的代码或操作边界 |
| 超时 | 调用方为一次等待规定的时长限制 |
| 截止时间 | 整项任务或上层操作允许持续到的时刻 |
| 退避与抖动 | 延迟重发，并通过随机差异分散请求到达时间 |
| 幂等键 | 标识同一业务操作，在全部重试中保持稳定的键 |
| 结果未知 | 现有证据不足以确定操作是否已产生效果 |
| 对账 | 比较本地记录与外部事实，确定结果或后续处理 |

## 6.14 练习与核对

先不用源码，推演一次工单恢复：服务已创建工单，本地只有键和参数，响应丢失后还剩一次尝试。分别假设服务支持幂等键、只支持延迟可见的查询、完全没有查询接口，说明每种情况下谁能提供什么证据，以及何时应停止自动执行。再把服务建议的等待时间设为超过任务剩余时间，判断是否还应该发起下一次请求。能够回答这些问题后，再对照 Pi 的具体恢复边界。

可检查 quota 不重试、取消与次数预算、服务端等待上限、Session 多次 `agent_end`、safe/unsafe 恢复及 intent 前 hook。练习时把“read 成功、assistant 连续两次失败、第三次成功”代入历史树和模型投影，再用 faux responses 验证省略记录；随后对照 durable 的 `started → close → reopen` 与提交边界，判断 `runs=1` 和 `runs=2` 分别意味着什么。无需真实模型服务。

在 `packages/ai` 下运行原定向命令 `node ../../node_modules/vitest/dist/cli.js --run test/retry.test.ts test/provider-retry.test.ts`；在 `packages/coding-agent` 指定 `test/suite/agent-session-retry-events.test.ts`，在 `packages/durable` 指定 `test/harness-tools-recovery.test.ts -t "reruns a tool only"`。这些用例使用模拟错误、faux provider 或本地 SQLite，不需要真实模型服务。

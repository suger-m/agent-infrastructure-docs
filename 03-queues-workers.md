# 03. Agent 忙着执行时，后续输入放在哪里

队列用于保存已经到达、暂时还不能处理的工作。比如 Agent 正在读取配置，你补充“不要改原模型”。这条输入需要被接收，但当前模型请求已经发出，程序只能在后续边界把补充交给模型。另一种等待发生在任务层：一项工作已经建立，执行条件尚未满足，需要调度器决定何时启动。

Worker 是实际承载工作的运行进程。进程可以理解为操作系统中正在运行的一份程序实例，它有自己的内存；进程退出，内存里的函数和等待就会消失。调度器负责选择工作，Worker 负责承载执行，队列负责保留待处理内容。三者分别回答“先放在哪”“什么时候做”“由谁运行”。

本章先看输入排队，再看由此产生的任务如何被启动，最后才扩展到进程退出和并发容量。按一个忙碌会话走，流程是：

1. 接收新输入，分配提交记录，判断立即放置还是进入队列。
2. 若正在运行，保存排队内容；用户看到输入已接收，不代表模型已经看见。
3. 到工具轮结束或最终回答的边界，取出允许消费的输入并加入对话记录。
4. 创建或推进相应任务，调度器为可执行任务登记本次执行，再运行处理函数。
5. 工具与模型推进任务，持久记录保存进度；会话进程按连接和活动工作决定何时退出。

这条流程使用 Pi 的实验 durable 路径。默认 CLI 也有内存输入队列，但不能据此推导出下面的持久输入和进程恢复能力。

当前一轮还在运行时，新消息不能改写正在执行的模型请求或工具。需要留下输入，等工具轮或最终回答的边界再交给模型；持久任务则要等调度器选中后才能运行，最后由进程承载实际执行。这里讲的是 `packages/durable` 的实验路径，稳定 coding-agent 默认内核没有因此改用这套执行方式。

在这条路径里，等待输入的会话队列叫 `pi.inbox`；TaskRecord（任务记录）保存工作阶段和恢复状态；Scheduler（调度器）在一个 Harness 内挑选可执行任务；Session worker（会话工作进程）承载一个会话实例。下面按这几项职责看交接关系。

| 当前职责 | 对应对象 | 下一步交给谁 |
| --- | --- | --- |
| 保存尚未放置的 steer、follow-up 和 write | 会话收件箱 `pi.inbox` | generation（一轮模型与工具执行）到边界消费 |
| 保存持久任务的阶段、输入、归属和检查点 | TaskRecord | scheduler 选择可执行任务 |
| 在一个 Harness（durable 会话运行实例）内预留任务并启动执行 | Scheduler / Invocation（调度器 / 本进程执行实例） | 任务处理函数在提交线外运行 |
| 承载一个会话实例及其服务、请求和活跃工作 | Session worker | 客户端与 Harness 活动共同影响进程退出 |
| 在一次扩展调用里限制并发、启动多个子任务 | subagent 本地循环 | 若干 `pi` 子进程执行；示例本身不持久恢复 |

沿一个完整过程看：用户先让 agent 执行 A。启动这轮工作时，scheduler 从持久任务记录中预留对应的 generation task，由当前会话的 worker 进程运行。模型正在调用工具时，又到达三条消息：“检查日志”留作后续任务，“只读，不要写文件”用于调整当前任务，还有一条只给界面展示的 note（备注）。系统准入后，把 follow-up 和 steer 存入 inbox；note 作为显示型记录直接追加。工具轮结束时，generation 在 `postTools` 边界取出 steer，A 得到最终回答后才在 `final` 边界取出 follow-up。若进程在执行期间退出，重开 Harness 会把遗留的 `running` 记录改回 `pending`，保留检查点，等待新的调度触发。

所以会话 inbox 决定输入何时进入 transcript，TaskRecord 记录工作进度，Scheduler 决定何时运行，worker 决定由哪个会话进程承载执行。subagent 示例的本地并发循环属于单次扩展调用，不参与 durable inbox 的消费。

本文对应提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc`。各章入口在[学习索引](README.md)，任务之间的关系和重启恢复分别见[任务编排](04-orchestration.md)、[持久执行](05-durable-execution.md)。

## 输入被接收以后

“提交”在这里表示用户把一项输入交给会话处理；submission 是追踪这项输入的记录。它可以处于排队、已放置或已结算等状态。transcript 是模型对话中按顺序保存的消息。“输入被接收”与“输入进入 transcript”是两个时间点，队列连接了它们。

假设读取期间送来两句话：“不要修改”要影响当前任务，“读完之后解释另一个文件”要在当前任务结束后处理。系统用 steer 和 follow-up 区分这两种意图。这里的 steer 是后续模型请求的调整，不承诺立刻停止已开始的修改；如果操作必须立即停止，还要走取消机制，并由工具配合信号。

A 已经有一条状态为 `placed` 的输入记录（submission），`pi.live.run` 记录当前这轮工作。后来的 follow-up 是等当前回答结束后再做的输入，steer 则希望调整正在进行的工作。两者先创建 `queued` submission，把内容放进 `pi.inbox.items`，并在同一次提交中保存下来。

note 的处理更短。它没有 `model`、`head`、`edits`，不会改变模型看到的上下文，通常可以马上追加成 entry，并把 write 记为 `done`。只有前面已经排了 reset 时，它才需要一起等候，跟着边界的放置顺序处理。

当前工具轮结束，`postTools` 边界取出 steer，将它加入 A 这轮工作；follow-up 继续留在队列。等 A 得到最终回答，`final` 边界才取出 follow-up，创建后续 generation。steer 因而也要等到边界才能进入下一次模型请求，它不会即时改写一个正在运行的请求或工具。

![Conversation.submit 按 requestId 去重、判断 write 与运行状态、排队并在边界放置输入的流程](figures/03-queue-boundaries.svg)

接收消息的入口在 [packages/durable/src/harness/submissions.ts L62-L95](../../packages/durable/src/harness/submissions.ts#L62-L95)。`submit()` 先启用调度，再把准入处理交给 Session 的串行提交线上的 `admitSubmission()`；`wait()` 用来等待已有 submission 结算。继续读 [packages/durable/src/harness/submissions.ts L160-L241](../../packages/durable/src/harness/submissions.ts#L160-L241)，就能把图中的几个分支对上代码：先按 request ID 去重，再处理显示型 write，然后检查会话是否忙碌，决定排队还是放置，必要时调用 `startRun()` 启动一轮工作。

去重也发生在这个入口。`requestId` 的作用域是一个 conversation，同 ID、同 submission 类型会取回旧记录，即便后来传入的 content 不同。它保证的是准入不重复；内容是否相同不会在这里核对，外部工具的副作用也需要自己的幂等键。

## 队列怎样进入 transcript

消费队列就是把等待项转成已经放置的消息，并移出待处理集合。这个动作还要保存提交记录的状态，否则用户等待的是一份记录，模型处理的却是另一份事实。所谓 commit（状态提交），是在存储中把这一组变化一起确立下来；本章的 commit 与 Git commit 无关。

会话文档保存 inbox 中的排队内容。[packages/durable/src/harness/inbox.ts L7-L29](../../packages/durable/src/harness/inbox.ts#L7-L29) 定义的三种条目是 `steer`、`followUp`、`write`，共同组成 `pi.inbox`。inbox 变空时会保存一个完整 base，也就是文档基准值，让后续恢复少重放一段增量。

消费规则集中在 [packages/durable/src/harness/inbox.ts L47-L110](../../packages/durable/src/harness/inbox.ts#L47-L110)。`prepareBoundary()` 先读这一边界需要的信息，`applyBoundary()` 再按当前模式选第一项或全部条目，追加 entries、更新 submissions，最后按倒序下标移除已经消费的项。边界内先放 writes，再放选中的用户输入。这解释了为什么稍后提交的 note 可以先出现在 transcript 中：submission ID 的顺序并不要求所有 entry 都严格按先入先出排列。

取队列与追加 entry 必须在同一次提交中完成。若先从内存 shift，再单独写 transcript，进程在两步之间退出就可能丢掉输入；若先追加、后移除，恢复又可能放置同一条输入。这里的一次 commit 同时保存两边的变化。

generation 提供了两个消费时机。最终回答的提交在 [packages/durable/src/harness/generation.ts L514-L544](../../packages/durable/src/harness/generation.ts#L514-L544) 处理 `final`；工具结果齐备后，[packages/durable/src/harness/generation.ts L598-L656](../../packages/durable/src/harness/generation.ts#L598-L656) 处理 `postTools`，或者在工作终止时处理 `final`。读完这两个调用点，再回到开头的三条消息，便能逐次标出它们何时只是排队、何时真正进入下一次请求。

## 谁把持久任务运行起来

任务记录像一份可保存的工作单：内容、执行阶段、等待关系和取消标记都能重新读取。当前执行则包括函数调用、网络等待和取消控制，它们存在于进程内存。把“工作单”和“这次执行”分开，进程重启后才能依据旧工作单建立新执行；旧 Promise 不会被存储恢复。

调度顺序也因此分两步：先在提交中标记任务已经预留，再在提交完成后调用处理函数。预留只表示当前调度器登记了执行，不能证明工作已经做完。处理函数下一次提交检查点或终态，才会留下新的持久进度。

输入放置后，调度器才执行 generation 和工具任务。这里的持久记录与进程内执行各有职责。`TaskRecord` 保存任务输入、版本、归属、检查点（checkpoint）和取消标记；检查点是下一次继续执行时要读取的状态。`Invocation` 则表示当前进程中的一次执行，包含 AbortController、Promise 和临时 watch。两者的定义分别在 [packages/durable/src/types.ts L501-L584](../../packages/durable/src/types.ts#L501-L584) 与 [packages/durable/src/harness/scheduler.ts L69-L95](../../packages/durable/src/harness/scheduler.ts#L69-L95)，恢复时能留下来的只有持久记录。

这也说明了记录里的 `running` 表示什么：调度器曾经为它建立了一次执行。重新打开 Harness 时，[packages/durable/src/harness/scheduler.ts L288-L321](../../packages/durable/src/harness/scheduler.ts#L288-L321) 的 `open()` 会扫描仍然活跃的任务，将遗留的 `running` 改回 `pending`，保留它们的 checkpoint。此时尚未分派执行；显式 `resume()`、提交输入或等待工作，才会启用调度。

运行中的入口在 [packages/durable/src/harness/scheduler.ts L1094-L1195](../../packages/durable/src/harness/scheduler.ts#L1094-L1195)。kick 把 drain 安排到微任务中，`#reserve()` 在一次串行 commit 里检查候选任务、等待条件、取消标记和 registry，保存 `running` 并登记 invocation。存储提交完成后，`#start()` 才启动它。

状态转换占用提交线，任务处理函数本身则在提交线外运行，见 [packages/durable/src/harness/scheduler.ts L1301-L1338](../../packages/durable/src/harness/scheduler.ts#L1301-L1338)。不同任务等待网络、计时器或工具时，可以同时挂起而不妨碍其他状态提交。调度器会保留每个符合条件的任务；它能同时运行多少、缺少哪些限制，见文末「边界与未实现」。

## 界面退出后，会话进程何时结束

终端界面断开时，任务可能还在执行；承载会话的进程是否退出由其生命周期条件决定。实验 server 因此把一个会话实例（Session）交给专属 Node 子进程；这个会话实例的多个客户端附件共享同一个进程。

[packages/coding-agent/src/experimental/session-worker-manager.ts L175-L187](../../packages/coding-agent/src/experimental/session-worker-manager.ts#L175-L187) 先按 Session 路径查找现存或正在启动的 worker，[packages/coding-agent/src/experimental/session-worker-manager.ts L449-L503](../../packages/coding-agent/src/experimental/session-worker-manager.ts#L449-L503) 才负责启动内部 `session-worker` 并等它报告就绪。这里的工作单位是一个会话实例，worker 并不从全局积压的任务中竞争领取单条消息。

退出条件写在 [packages/coding-agent/src/experimental/session-worker.ts L183-L305](../../packages/coding-agent/src/experimental/session-worker.ts#L183-L305) 的 `WorkerLifecycle` 中。客户端的连接需求、正在处理的请求、初始宽限期和 Harness 中的活跃工作都会让进程继续存在，全部清空后才能退出。[packages/coding-agent/src/experimental/session-worker.ts L509-L544](../../packages/coding-agent/src/experimental/session-worker.ts#L509-L544) 还要求打开 SQLite 前先取得目录锁，通过 `proper-lockfile` 的 stale 与 update 配置维护这份所有权。[packages/coding-agent/src/experimental/session-worker.ts L585-L594](../../packages/coding-agent/src/experimental/session-worker.ts#L585-L594) 则把所有尚未结束的 task 都算作活动，包括后台任务。

目录锁会周期更新，用于维护本地 Session 的所有权；任务级租约与跨进程所有权的缺口，同样收在文末「边界与未实现」。

把几个层次并排看，队列保存的对象和恢复方式就更容易分清：

| 层 | 排队对象 | 实际执行者 | 重启后依据 |
| --- | --- | --- | --- |
| 会话 inbox | `SubmissionId` 对应的输入或写入 | generation 在边界取出 | `pi.inbox` 和 submission 记录 |
| TaskScheduler | `TaskRecord` 的 checkpoint | 当前进程里的异步处理函数 | 存储中最新任务记录 |
| 实验 Session worker | 一个会话实例的所有任务与服务 | 一个 Node 子进程 | 该 Session 的 SQLite 文件 |
| subagent 扩展示例 | 一次调用的任务数组 | 本机若干 `pi` 子进程 | 示例本身没有持久调度恢复 |

## 并发限制与积压

并发指同一时间尚未完成的工作数量。例如四次文件读取都在等待，只有其中一次正在处理返回结果，它们仍算四项并发工作。并发限制用于避免同时发出过多请求或启动过多进程；队列容量用于限制还有多少工作在等，两者要分别考虑。

假设生产者持续提交，消费者每秒只能处理较少工作。即使执行并发已经封顶，等待队列仍会增长。Backpressure（背压）是把容量不足的信息反馈给提交方，让它等待、减速或收到拒绝。优先级则决定容量有限时先选哪项工作。下文的本地限流示例只说明如何限制一次调用的执行数量，没有把这些能力都实现出来。

subagent 扩展示例给出了一种本地限流办法：最多接受 8 个任务，同时运行 4 个。`mapWithConcurrencyLimit()` 用一个共享 nextIndex 和若干异步循环依次取工作，每次调用启动参数带 `--no-session` 的 `pi` 子进程。上限、循环实现和启动参数分别见 [packages/coding-agent/examples/extensions/subagent/index.ts L33-L36](../../packages/coding-agent/examples/extensions/subagent/index.ts#L33-L36)、[packages/coding-agent/examples/extensions/subagent/index.ts L219-L237](../../packages/coding-agent/examples/extensions/subagent/index.ts#L219-L237)、[packages/coding-agent/examples/extensions/subagent/index.ts L300-L350](../../packages/coding-agent/examples/extensions/subagent/index.ts#L300-L350)。这一限制属于扩展的一次调用，durable scheduler 没有采用它作为默认并发配置。

## 边界与未实现

各节提到的限制与尚未实现的部分，收拢在这里：

- 记录里的 `running` 只表示调度器曾为它建立一次执行，没有分布式租约的有效期。
- 每个任务的租约和心跳并没有在调度器中实现。`pi-durable` 库本身要求一个进程拥有 storage，跨进程锁要由宿主提供，这个约定写在 [packages/durable/README.md L630-L641](../../packages/durable/README.md#L630-L641)。
- 调度器会保留每个符合条件的任务，但源码没有为这一步设置全局并发槽、优先级、租户配额或任务租约超时。generation 中的工具串行、并行策略只控制工具轮，整个 Harness 的执行数量还需要另行考虑。
- 这些持久任务属于 `pi-durable` 的实验实现。它已有包入口、存储适配器和测试，但 [packages/durable/README.md L1-L7](../../packages/durable/README.md#L1-L7) 仍明确标为 Experimental，稳定 coding-agent 默认内核并没有因此改成这套执行方式。实验 client/server 的范围更窄，[packages/coding-agent/src/experimental/services/README.md L1-L12](../../packages/coding-agent/src/experimental/services/README.md#L1-L12) 说明它们只提供源码入口，不进入 npm 包和独立二进制。
- 如果要让多个执行者竞争领取持久工作，就会涉及工作进程池（worker pool）；生产者与消费者之间负责保存、分发消息的独立系统称为消息代理（broker）。目前这些本地队列和 Session 进程还没有组成这样的分布式系统，共享 broker、ack/nack、可见性超时、死信队列与竞争消费都未实现。跨主机任务租约、续租以及 fencing token 也需要补上；后者用版本凭证阻止已经失效的执行者继续写入。Session worker 当前不会自动把一个会话实例的工作迁移到另一主机。
- 积压还需要背压（backpressure），即限制生产速度或待处理量。inbox 准入目前没有队列长度、字节配额，本地 scheduler 也没有全局任务并发预算。[分布式状态](07-distributed-state.md) 中的公开状态交付通道虽有 100 帧上限，约束的是观察通道，输入队列仍要单独处理容量。

## 源码位置与术语速查

| 机制 | 主要源码位置 |
| --- | --- |
| 提交准入与去重 | `packages/durable/src/harness/submissions.ts` |
| 会话队列（`pi.inbox`） | `packages/durable/src/harness/inbox.ts` |
| 边界消费（`final` / `postTools`） | `packages/durable/src/harness/generation.ts` |
| 任务调度与恢复 | `packages/durable/src/harness/scheduler.ts` |
| 任务记录与执行类型 | `packages/durable/src/types.ts` |
| 会话子进程 | `packages/coding-agent/src/experimental/session-worker-manager.ts`、`session-worker.ts` |
| 本地限流示例 | `packages/coding-agent/examples/extensions/subagent/index.ts` |

### 术语速查

| 术语 | 在这个项目中的含义 |
| --- | --- |
| Submission | 一条输入或写入的持久记录，有 `placed` / `queued` 等状态 |
| `pi.inbox` | 会话文档中的排队条目集合，含 `steer`、`followUp`、`write` 三种 |
| Steer | 在 `postTools` 边界加入当前工作的调整输入 |
| Follow-up | 等当前最终回答后，在 `final` 边界创建后续 generation 的输入 |
| Generation | 一轮工作的执行单位，提供 `final` 与 `postTools` 两个消费边界 |
| TaskRecord | 持久任务记录：输入、版本、归属、checkpoint 和取消标记 |
| Invocation | 当前进程中的一次执行：AbortController、Promise 和临时 watch |
| Session worker | 承载一个会话实例的专属 Node 子进程 |

## 测试与练习

下面的测试可以帮助核对前面的过程。

| 阅读位置 | 可核对的行为 |
| --- | --- |
| [packages/durable/test/harness-submissions.test.ts L57-L107](../../packages/durable/test/harness-submissions.test.ts#L57-L107) | 忙碌时拒绝输入不产生写入；request ID 按 conversation 去重 |
| [packages/durable/test/harness-submissions.test.ts L134-L147](../../packages/durable/test/harness-submissions.test.ts#L134-L147) | 取消等待后，已经放置的工作继续存在 |
| [packages/durable/test/harness-submissions.test.ts L178-L221](../../packages/durable/test/harness-submissions.test.ts#L178-L221) | 重开后取回 submission；提交和等待都会启用调度 |
| [packages/coding-agent/test/experimental-session-worker-lifecycle.test.ts L23-L107](../../packages/coding-agent/test/experimental-session-worker-lifecycle.test.ts#L23-L107) | 客户端连接需求、Harness 活动和请求持有共同决定进程退出 |

可以先在纸上跟踪 A、steer、follow-up 和 note，给每次 commit 写出 inbox 和 submission 的状态。随后把 queued reset 加入纸上例子，再对照 `applyBoundary()`，检查先写入、后放置用户输入会怎样改变放置结果。这个练习无需真实模型，也不用修改文件。

如果自行验证局部测试，从 `packages/durable` 执行：

```powershell
node ../../node_modules/vitest/dist/cli.js --run test/harness-submissions.test.ts
```

这里跑的是输入队列测试，只需 faux provider；provider e2e 和 shell 交互示例需另行安排。跨进程 Session worker 的部署细节在 [Agent Runtime](01-agent-runtime.md)。

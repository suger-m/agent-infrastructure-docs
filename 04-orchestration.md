# 04. 父任务结束之前，子 Agent 怎样收尾

编排负责把多项工作组织成有关系的执行过程。例如修改配置可以拆成读取、修改、审阅；有些步骤要顺序执行，有些可以同时检查不同文件。分拆以后，系统还得知道谁等谁、失败时取消哪些工作、什么条件下才算整项任务结束。

多 Agent 是其中一种做法：把某项工作交给另一份独立会话，让它使用自己的上下文和工具完成。任务与 Agent 不必一一对应。支付、等待、投递结果也可以是任务，完全不请求模型；一个 Agent 会话又可能产生多个模型与工具任务。下面先用普通父子任务讲清生命周期，再把同样的关系用于子 Agent。

完整编排过程可以按这条顺序理解：

1. 父任务创建子工作，并保存它们的归属和自己的下一阶段。
2. 子工作执行，父任务按等待关系暂停；暂停时无需保留一条正在运行的函数栈。
3. 子任务成功、失败或收到取消，分别保存结果；取消后的清理也需要完成。
4. 等待条件满足，父任务读取子结果，决定继续、失败或结束。
5. 父任务提出结束后，检查仍由它负责的工作是否已经收尾；满足条件才向调用者返回终态。

接下来用结账任务说明这条过程。支付示例没有模型，因而可以把编排本身与模型生成分开观察。

Checkout（结账任务）创建 Payment（支付任务）后，可能已经决定订单失败，但其中一笔支付仍在清理。此时若 `waitForTask(checkout)` 立即返回，调用者会以为整笔操作已经收尾。系统因此要记录谁拥有哪些工作、父任务正在等谁，以及取消请求何时真正结算。

Task（任务）保存一项工作的阶段与结果，Conversation（会话）保存 agent 的对话及输入，子 Agent 通常在自己的会话中执行工作。任务可以拥有子任务或会话，也可以显式等待其他任务。所有权（ownership）决定父任务的收尾等待和取消范围；join（显式等待）决定它何时继续下一阶段，但不会改变归属。

| 流程中的部分 | 对应对象或关系 | 它怎样影响收尾 |
| --- | --- | --- |
| 当前工作阶段及结果 | Task、检查点（checkpoint）和结果（outcome） | 父任务的结束请求可能先成为 `completing` |
| 对话和输入归属 | Conversation 与所有权（ownership） | 会话中的任务可归入某个父任务的生命周期 |
| 父任务要等的成员 | `on` / join 关系 | 成员结算后父任务才能恢复下一阶段，不改变归属 |
| 后台工作 | 后台边界（background boundary） | 普通空闲检查和取消遍历在边界处截断 |

具体来说，Checkout 创建 Payment A、Payment B，并在同一次提交中保存自己的下一阶段和 `on` 等待集合。两笔支付执行期间，Checkout 没有处理函数持续占用运行线程。若 B 失败，`failFast` 给仍活跃的 A 写入取消标记；A 的 abort handler 清理后结算。两个 Payment 都进入终态后，Checkout 恢复，读取 outcomes 并决定订单结果。若它决定失败，调度器先保存父结果；只要仍有它拥有的普通工作未结束，状态就停在 `completing`。所有这些工作清空后，父任务才进入 `terminal`，`waitForTask(checkout)` 才返回。

这里有两个容易混在一起的事实：取消请求先被记录，子任务仍要自行退出并结算；父任务即使没有显式 join 自己拥有的工作，结束时仍要等它们收尾。反过来，`allSettled` 可以等待非 owned task，这条等待边不会扩大所有权。

本文对应提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc`。各章入口在[学习索引](README.md)；本地调度从[队列与 Worker](03-queues-workers.md)开始，重启后的处理见[持久执行](05-durable-execution.md)。

这些机制属于 `packages/durable`，其 README 在 [packages/durable/README.md L1-L7](../../packages/durable/README.md#L1-L7) 明确标为 Experimental。关于子任务、取消和 task graph 的说明见 [packages/durable/README.md L508-L596](../../packages/durable/README.md#L508-L596)。稳定 coding-agent 的 `examples/extensions/subagent` 是可安装的扩展示例，实验 durable TUI 又有自己的源码入口；它们呈现了不同的编排方式，后文会分别读到。

下面先用短图跟踪 Payment B 失败、A 收到取消请求和 Checkout 等待结算的过程；再看关系图。关系图中的实线表示所有权，虚线表示显式等待。图里的边不是等价关系：等待边不会改变归属，owned child 即使没有 join 也会参与父任务收尾。

![Payment B 失败后，Checkout 请求取消 A，但仍等子任务结算再继续](figures/04-child-lifecycle.svg)

![Root conversation、Checkout、Payment、子会话与后台任务的所有权关系和显式等待边](figures/04-task-ownership.svg)

## 创建任务时，关系就固定下来

任务需要一个“下一步从哪继续”的定义。状态机就是把工作分成明确阶段，并规定每个阶段如何转向下一阶段。例如结账可以先创建支付，再等待，最后判断结果；保存当前阶段与必要数据后，程序重新打开也能找到相应处理函数。`initial`、`phases` 和 checkpoint 正是在表达这个关系。

归属关系要在创建时留下，因为它决定后来谁负责清理。等待关系可以跨工作建立，却不能仅凭“我在等它”推出“我拥有它”。例如审阅任务可能在等一个共享检查任务，审阅被取消时，不一定应取消共享检查。Pi 用不同字段保存两种关系，再分别执行等待与取消规则。

任务定义是一份状态机：initial 给出起点，phases 对应各阶段处理函数，abort 负责取消，migrate 负责转换旧版本，join policy 决定怎样等其他任务。定义在 [packages/durable/src/types.ts L242-L277](../../packages/durable/src/types.ts#L242-L277)，而 [packages/durable/src/types.ts L142-L155](../../packages/durable/src/types.ts#L142-L155) 要求每个 phase 提交变化后的 checkpoint 或终态。checkpoint 保存下次要继续的阶段和状态。

`defineTask()` 只是把 definition 包装成任务对象，见 [packages/durable/src/tasks.ts L1-L8](../../packages/durable/src/tasks.ts#L1-L8)；主机还要安装包含这份定义的 extension，调度器才能执行它。实际创建记录的是 [packages/durable/src/session/transaction.ts L383-L427](../../packages/durable/src/session/transaction.ts#L383-L427) 中的 `createTask()`：解析 owner 和 conversation，运行 initial，生成一条 `pending` 记录。

子任务要留在所属任务的会话内，不能直接设 `background: true`。后台边界设在由会话拥有的任务上。任务还可以拥有另一个会话，让那个会话中的工作归入自己的生命周期。

提交组装时，[packages/durable/src/session/transaction.ts L852-L874](../../packages/durable/src/session/transaction.ts#L852-L874) 会再检查 owner 的拟提交状态。已经 terminal、completing 或带 abort mark 的 owner 都不能接收新工作。因此，创建 child 并让父任务转为 waiting 可以一起提交；创建 child 的同一 commit 若又结束 owner，就会被拒绝。前面提到的结束等待，针对的是父任务已经拥有的工作。

Checkout 示例把创建与 join 放在同一处，适合从这里跟一次完整流程：[packages/durable/test/examples/24-child-tasks.ts L59-L105](../../packages/durable/test/examples/24-child-tasks.ts#L59-L105) 中的 `pay` phase 创建一批 Payment IDs，同时保存 `{ phase: "decide", payments }`、`on: payments` 和 `failFast`。所有 Payment terminal 后，Checkout 才从 `decide` 读取 `runtime.outcomes()`，自己判断付款是否全部成功并提交订单结果。

## 等待、失败和取消如何推进

父任务等待时，保存的是待等成员和下一阶段，当前处理函数已经退出。调度器以后读到成员都已结算，才建立一次新的执行。这样即使等待长达几分钟，也无需把原函数一直留住，恢复也能从记录判断条件。

失败提前通知与最终收尾同样需要分开。一个子任务失败时可以立即请求取消其他子任务，但其他任务可能还在关闭连接、结束进程或清理临时文件。父任务若在发出取消那一刻就宣告完成，调用者就会在清理尚未结束时开始后续操作。`completing` 保存“结果已确定但收尾未完”的阶段，让这段等待也成为可恢复状态。

等待期间没有任务处理函数留在运行。[packages/durable/src/harness/scheduler.ts L1197-L1207](../../packages/durable/src/harness/scheduler.ts#L1197-L1207) 检查 `on` 中仍然活跃的成员；普通等待要等全部成员 terminal，任务才能再次被调度。failFast 会早些发出取消：[packages/durable/src/harness/scheduler.ts L609-L619](../../packages/durable/src/harness/scheduler.ts#L609-L619) 发现一个成员有非 completed outcome 后，给其他活跃成员写 abort marks，但仍然要等它们结算，才进入下一 phase。业务成功或失败最后由 Checkout 决定。

等待关系也有约束。[packages/durable/src/harness/scheduler.ts L1472-L1498](../../packages/durable/src/harness/scheduler.ts#L1472-L1498) 检查成员是否存在，拒绝任务等待自己或祖先 owner，要求 failFast 的成员都是直接 owned tasks，abort handler 也不能再转入 waiting。这些检查防住了几类无法先完成的依赖；更一般的依赖表达能力还缺什么，见文末「边界与未实现」。

父任务已经决定结束时，[packages/durable/src/harness/scheduler.ts L1448-L1470](../../packages/durable/src/harness/scheduler.ts#L1448-L1470) 会检查任务拥有的普通工作；只要还有活跃工作，terminal 请求便暂存为 `completing`。[packages/durable/src/harness/scheduler.ts L639-L674](../../packages/durable/src/harness/scheduler.ts#L639-L674) 逐轮完成已经清空工作的任务的最终结算，一个子任务结束又可能让上层 owner 满足收尾条件。

主动取消也沿着这份持久关系处理。[packages/durable/src/harness/scheduler.ts L339-L366](../../packages/durable/src/harness/scheduler.ts#L339-L366) 先保存 abort mark，再等现有 run invocation 退出；[packages/durable/src/harness/scheduler.ts L573-L628](../../packages/durable/src/harness/scheduler.ts#L573-L628) 从这些记录继续向下取消、撤回排队输入，并推进结算。这样即使过程被重启打断，下一次仍能读到取消意图。各任务的 abort handler 自行清理效果，库的补偿能力边界同样收在文末。

后台边界会截断普通空闲检查和取消遍历；如果要连这部分一起取消，conversation abort 需指定 `background: true`，对应代码在 [packages/durable/src/harness/scheduler.ts L391-L414](../../packages/durable/src/harness/scheduler.ts#L391-L414)。这个边界正是后台子 agent 能继续工作的原因。

## 把子会话交给一个工具

要让父 Agent 委派工作，可以向它提供一个 subagent 工具。模型调用该工具时，工具创建或找到子会话、提交任务、等待答案，再把答案作为工具结果交回父模型。父模型看到的是一次工具调用的结果；子会话内部可能已经经历很多模型和工具回合。

上下文要在这条交接中明确传递。例如子 Agent 审阅配置，至少需要任务目标、配置位置和要检查的约束；单独给一句“检查它”并不能让子会话自动获得父历史。前台方式由当前工具等待结果；后台方式需要另一个持久角色在答案出现后投递回父会话，因而多出 anchor 和 reporter。

前台与后台两种 subagent 用不同的所有权安排把答案送回父会话。

前台 subagent 示例 [packages/durable/test/examples/22-subagent-foreground.ts L38-L79](../../packages/durable/test/examples/22-subagent-foreground.ts#L38-L79) 把子会话的 owner 设为当前工具任务。工具先按 ownerTaskId 查找已经创建的 child，没有才创建，并移除 child 的 subagent extension，防止继续递归委派。随后用 `subagent:<taskId>` 作为 requestId 提交任务，等已有 submission 完成，再从 answer entry 读回文本。

这个工具声明可以安全重放。恢复时，ownerTaskId 和 requestId 让 execute 重新找到原来的 conversation 与输入。子会话有独立的对话记录，继承的是父会话的 agent 配置，父会话全部历史并不会自动传过去，所以工具参数仍须提供足够上下文。工具失败或取消会传到 child 的工作，conversation 记录则保留下来。

后台示例 [packages/durable/test/examples/23-subagent-background.ts L56-L126](../../packages/durable/test/examples/23-subagent-background.ts#L56-L126) 多加了两个角色。background anchor 拥有 child，background reporter 负责投递消息，等答案到了，再作为父会话的 follow-up 送回去。reporter 把过程分为 `deliver` 和 `report` 两个 phase，child request ID 与 parent report request ID 各自去重，答复 entry ID 也写入文档，避免几条输入共用一份回答时报告多次。

anchor 很快就会 terminal，它与 child 的关系仍保存在 ConversationRecord.owner 中。稍后看 live task graph 时，anchor 消失只说明该 task 已经收尾，child conversation 并未随它消失。

读这个示例还要留意返回字段。[packages/durable/test/examples/23-subagent-background.ts L149-L153](../../packages/durable/test/examples/23-subagent-background.ts#L149-L153) 的 `reply()`，以及实验 durable subagent，仍返回 `content`；[packages/durable/src/harness/tool.ts L500-L508](../../packages/durable/src/harness/tool.ts#L500-L508) 的当前 tool runtime 读取的是 `output`。所有权和 reporter 设计可以独立研究，当前示例的完整输出行为还受这处接口出入影响。

稳定 coding-agent 的扩展示例使用另一种做法。[packages/coding-agent/examples/extensions/subagent/index.ts L550-L601](../../packages/coding-agent/examples/extensions/subagent/index.ts#L550-L601) 中的 chain 把 `{previous}` 替换为上一步文本，顺序执行，失败就停止；[packages/coding-agent/examples/extensions/subagent/index.ts L604-L685](../../packages/coding-agent/examples/extensions/subagent/index.ts#L604-L685) 的 parallel 通过本地并发函数收集结果。数组顺序与执行状态留在这次调用的内存中，子进程带 `--no-session`。可以从中研究上下文隔离与本地扇出；它在恢复与分布式上的缺口见文末「边界与未实现」。

## 审阅、交接和用户决定

Reviewer 是被赋予检查职责的 Agent。它需要明确检查对象与完成条件；使用另一个模型或只读工具可以改变检查方式，但“审阅者”这个名称本身不保证检查正确。Handoff 是把工作说明和当前进度交给后续上下文；Human-in-the-loop 表示某一步需要人作决定，例如审阅计划后才继续执行。

这几种机制都在改变工作怎样继续。审阅循环根据发现决定是否再检查；交接提供后续请求的起点；人工决定暂停执行，等外部输入后恢复。下面的示例分别展示这些入口，它们没有自动合成一个带身份校验和批准记录的发布系统。

Reviewer 示例关心的是审阅一轮后可能仍有遗漏。[packages/durable/test/examples/28-reviewer.ts L31-L44](../../packages/durable/test/examples/28-reviewer.ts#L31-L44) 安装审阅角色的 section，并通过 `GenerationTask.onYield` 检查终止短语；若仍有发现，就追加“再看一遍”的输入，让同一 run 继续。

[packages/durable/test/examples/28-reviewer.ts L77-L100](../../packages/durable/test/examples/28-reviewer.ts#L77-L100) 创建一个没有 owner 的审阅会话，选另一模型，给它只读工具和独立工作目录。它先读取，再提交第一轮发现的问题，随后由 onYield 续一轮，直到第二轮发现的问题带上结束短语，输入记录才完成。

交接上下文的入口在 [packages/durable/src/harness/harness.ts L113-L120](../../packages/durable/src/harness/harness.ts#L113-L120)。`root.reset(handoff)` 写一条带 `head: "self"` 的 `pi.reset`，把交接说明作为新上下文的 user message，旧 entries 仍留在存储中。工具也可以返回 `control.handoff`；工具轮结束时，[packages/durable/src/harness/generation.ts L627-L656](../../packages/durable/src/harness/generation.ts#L627-L656) 的 `finishToolRound()` 选调用顺序里的最后一份说明，写 reset、结束当前 run，再处理排队的后续输入。

需要用户先审阅计划再执行时，durable plan 示例 [packages/durable/test/examples/27-plan-mode.ts L28-L62](../../packages/durable/test/examples/27-plan-mode.ts#L28-L62) 先把工具收窄到 read 与 submit_plan，计划存进可以回看历史值的 `PlanDoc`，通过 `terminate` 停下本轮。用户侧再调用 `configure(leavePlanMode)`，提交“Go ahead.”，见 [packages/durable/test/examples/27-plan-mode.ts L93-L100](../../packages/durable/test/examples/27-plan-mode.ts#L93-L100)。这就是一处 human-in-the-loop，也就是人在执行流程中作决定的入口：先留下能审阅的计划，再由主机恢复工具与执行。

稳定 coding-agent 的 plan-mode 扩展还提供了具体 UI，[packages/coding-agent/examples/extensions/plan-mode/index.ts L301-L336](../../packages/coding-agent/examples/extensions/plan-mode/index.ts#L301-L336) 让人选 Execute、Stay 或 Refine。Execute 恢复工具并排入 follow-up；Refine 读用户补充后再投递消息。

## 从 task graph 看收尾过程

Graph（图）用节点表示工作，用边表示关系。DAG 是没有循环依赖的有向图，例如读取配置后才能修改、修改后才能审阅。等待环表示两项工作互相等对方，谁也无法先结束。任务所有权图与依赖图还有不同含义：前者决定谁负责收尾，后者决定执行先后。Pi 的 live task graph 主要展示当前关系，不能单凭这个视图推导它已经是通用 DAG 调度器。

checkout 或 subagent 的收尾过程，可以从 live task graph 直接观察。运行时可用 `Harness.taskGraph()` 查看尚未结束的工作。[packages/durable/src/harness/task-graph.ts L18-L43](../../packages/durable/src/harness/task-graph.ts#L18-L43) 的节点记录 live status、phase、owner、background、abortRequested、waiting.on/policy，以及 task 拥有的 conversations。初始值由 [packages/durable/src/harness/task-graph.ts L107-L148](../../packages/durable/src/harness/task-graph.ts#L107-L148) 在 Session line 上原子构造，之后 [packages/durable/src/harness/task-graph.ts L155-L207](../../packages/durable/src/harness/task-graph.ts#L155-L207) 随 commit publication 更新，terminal 节点随结算离开图。

这个视图只保留 live records，完整历史与 checkpoint/outcome 大载荷要从其他入口读取。pending 为什么 blocked 也不在图中；排查这种任务可用 `harness.inspect()`，具体见 [可观测性与调试](11-observability-debugging.md)。

## 边界与未实现

各节提到的限制与尚未实现的部分，收拢在这里：

- 等待关系的检查没有遍历全图找环；两个独立任务通过 `allSettled` 互等，单凭这个函数无法断定会被拒绝。如果要用它构造通用 DAG，即无环有向图工作流，还须另外定义和验证依赖。当前实现没有任意工作流的声明语言、拓扑排序器、全图环检查、声明式分支或工作流 DSL。
- 各任务的 abort handler 自行清理效果，库没有提供通用补偿事务引擎，也不会自动回滚外部系统。
- 稳定 coding-agent 的 subagent 扩展示例把执行状态留在一次调用的内存中：重启恢复还需要额外保存 checkpoint，这条路径也没有共享 broker 或跨主机调度。
- Reviewer 示例发现的问题不会自动转成补丁或合并闸门，示例也没给循环设最大轮数。示例的环境是临时目录和 MemoryStorage，代码没有创建真实 git worktree，也没有演示跨进程保存；角色与配置可以随 Harness 存储，能否跨进程保留仍取决于后端。
- handoff 保存了上下文交接点，没有自动把 Session 迁到另一台机器或另一个 agent 进程。
- plan 示例没有通用审批记录或审批队列，也没有授权身份、批准版本、过期规则和审批崩溃恢复协议。危险工具阻断与可信项目确认另见 [权限与 Secrets](10-permissions-secrets.md)。
- `TaskScheduler` 的调度范围是一个 Harness 实例，项目目前没有跨机器编排服务或 worker lease。实验 services 也只暴露 root conversation，[packages/coding-agent/src/experimental/services/README.md L38-L45](../../packages/coding-agent/src/experimental/services/README.md#L38-L45) 把 keyed subagent services 列为后续工作。前台与后台示例中的 conversation 关系，因而还没有全部变成远端服务接口。

## 源码位置与术语速查

| 机制 | 主要源码位置 |
| --- | --- |
| 任务定义与包装 | `packages/durable/src/types.ts`、`packages/durable/src/tasks.ts` |
| 创建任务与提交组装 | `packages/durable/src/session/transaction.ts` |
| 等待、取消与结算 | `packages/durable/src/harness/scheduler.ts` |
| 工具轮与 handoff | `packages/durable/src/harness/generation.ts`、`packages/durable/src/harness/tool.ts` |
| live task graph | `packages/durable/src/harness/task-graph.ts` |
| Checkout / 前后台 subagent 示例 | `packages/durable/test/examples/24-child-tasks.ts`、`22-subagent-foreground.ts`、`23-subagent-background.ts` |
| 审阅与计划示例 | `packages/durable/test/examples/28-reviewer.ts`、`27-plan-mode.ts` |
| 稳定 subagent / plan-mode 扩展 | `packages/coding-agent/examples/extensions/subagent/index.ts`、`plan-mode/index.ts` |

### 术语速查

| 术语 | 在这个项目中的含义 |
| --- | --- |
| `completing` | 父结果已保存、但拥有的普通工作尚未清空的过渡状态 |
| `terminal` | 任务及其收尾全部完成的终态 |
| Ownership | 工作归属于哪个任务的持久记录，决定等待范围与取消范围 |
| Join / `allSettled` | 显式等待其他任务的边，不改变对方的归属或取消规则 |
| failFast | 一个成员出现非 completed outcome 时，提前给其他活跃成员写 abort mark |
| Abort mark | 持久化的取消意图，重启后仍能读到并继续执行 |
| Background anchor / reporter | 后台示例中拥有 child 的任务 / 负责把答案投递回父会话的任务 |
| Handoff | 以 `pi.reset` 保存的上下文交接点 |

## 测试与练习

画出 checkout 的 owner 边和 wait 边，再让 Payment B failed。可以逐个预测哪些 task 会收到 abort mark，Checkout 何时回到下一 phase，以及 `waitForTask()` 什么时候返回。随后把一项工作放在 background anchor 下，比较父会话是否空闲与 worker 能否退出，解释这两个判断为什么会不同。

相关断言可以在下面的测试中找到。

| 阅读位置 | 可跟踪的过程 |
| --- | --- |
| [packages/durable/test/harness-task-graph.test.ts L84-L166](../../packages/durable/test/harness-task-graph.test.ts#L84-L166) | waiting、completing、owner conversations，以及 terminal 节点删除 |
| [packages/durable/test/harness-task-graph.test.ts L168-L238](../../packages/durable/test/harness-task-graph.test.ts#L168-L238) | 重开后变为 pending，重放 graph operations |
| [packages/durable/test/harness-tasks-recovery.test.ts L176-L245](../../packages/durable/test/harness-tasks-recovery.test.ts#L176-L245) | 取消 mark、abort handler 和 terminal 各阶段的恢复 |

若自行验证，从 `packages/durable` 运行 `node ../../node_modules/vitest/dist/cli.js --run test/harness-task-graph.test.ts`，无需真实模型。支付示例使用内存假银行，其中的退款行为没有给真实外部系统增加事务保证。

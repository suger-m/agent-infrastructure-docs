# 第 7 章　分布式状态与一致性

[返回目录](README.md) · [上一章：第 6 章 重试、超时与幂等](06-retry-timeout-idempotency.md) · [下一章：第 8 章 沙箱与执行环境](08-sandbox-environment.md)

多个执行者共享任务时，系统需要确定状态更新如何共同提交、哪一个执行者拥有处理资格，以及读到的状态允许落后多久。本章先以两个 Worker 的任务领取和故障接管说明事务、租约与写入校验，再讨论数据库复制和一致性；随后介绍 Pi 的单进程 durable 存储与 Chord 的对象状态同步，区分任务执行权和界面观察状态。

假设 Agent 服务收到“读取资料并生成报告”的任务，将它存入数据库。Worker A 和 B 都能查询待办，用户在网页查看进度。两者可能同时发现同一任务，A 也可能在运行中失联，再于 B 接管后恢复；与此同时，页面还可能显示数据库副本中的旧状态。这些情况分别涉及并发领取、执行权换代和读取可见性，需要配合处理。

下面先用共享数据库的双 Worker 系统说明通用设计。示例中的任务表、跨主机租约与接管协议属于教学架构，Pi 当前没有实现这套分布式调度；后半章再对照 Pi 的实验 durable 存储与 chord 宿主运行时（Chord 部分按提交 `482c1381a` 重写）。本文的源码基线为提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc`，持久化的提交顺序见[第 5 章 持久执行与恢复](05-durable-execution.md)，客户端与工具边界见[第 9 章 工具基础设施与 MCP](09-tools-mcp.md)。

## 7.1 原子领取与事务不变量

任务 T 的初始状态为“待处理”。A 读取 T 后尚未更新，B 也读取到同一状态；读取本身没有改变记录，因此两次查询都可能成功。若双方随后各自写入“运行中”并调用工具，即使最终记录只保存一个 owner（当前执行者），也无法撤销已经开始的重复执行。

原子操作把资格检查和所有权更新合为一个不可分割的动作。可以要求数据库仅在 T 仍为待处理时，将其改为运行中并写入 owner。A 成功修改一行后，B 的条件不再成立，更新结果为零行；B 应放弃该候选任务。查询用于发现候选，实际条件更新的成功结果才是领取依据。

一次领取还可能需要新增执行记录，保存尝试编号与开始时间。若 owner 已更新而执行记录尚未写入就崩溃，恢复程序会缺少这次执行的依据。事务（transaction）将关联修改放在共同的提交边界内：全部成功才提交，任一步失败则回滚该批修改。

不变量是在每次成功更新后都必须成立的约束。本例要求 owner、执行有效期与对应执行记录同时存在；完成时，结果引用与完成状态也应一起提交。事务提供组合更新的边界，并发控制还必须排除多个领取者的竞争。可采用适当行锁、可串行化隔离或写入时再次核对条件。隔离级别规定并发事务的可见性与冲突处理，因此把普通读写放入事务并不自动保证领取互斥。

Worker 应在领取事务提交成功后开始生成报告，不要把耗时的模型调用放在数据库事务内一直持锁。这样会阻塞其他领取者，外部调用也通常无法随数据库回滚。任务完成后再开一个短事务，核对执行资格，保存结果引用并改为已完成。这里的原子性只覆盖同一数据库中的更新；已经发出去的邮件或外部文件写入要另外处理，参见[第 6 章 重试、超时与幂等](06-retry-timeout-idempotency.md)。

## 7.2 WAL 与数据库崩溃恢复

事务给调用方规定了“整体成功或失败”，数据库还需要在进程突然停止后维持这个结果。写前日志（write-ahead log，WAL）是一种实现办法：先把修改及其提交边界记录到日志，再将修改合入主要数据文件。以 SQLite 为例，恢复时可用有效提交记录区分完整事务与未完成的尾部；读取还可能需要结合主文件和 WAL，不能仅凭主文件判断最新状态。后续 checkpoint 将日志中已提交的内容合入主文件。这里的 checkpoint 是数据库维护动作，与 Agent 保存任务执行位置的检查点不是同一层。[SQLite WAL 文档](https://www.sqlite.org/wal.html)

假设 A 的领取已提交，数据库尚未完成 checkpoint 就停止。若所需日志已可靠保存，重新打开后仍可还原领取结果；若停止发生在提交前，数据库不应保留只有 owner 而缺少执行记录的部分更新。不过，提交返回后，最近数据能否在掉电后保留，取决于同步落盘策略。SQLite 在 WAL 模式下使用 `synchronous = NORMAL` 时，应用进程崩溃与机器掉电的保证不同，掉电或系统崩溃仍可能丢失最近已提交事务。[SQLite synchronous 说明](https://www.sqlite.org/pragma.html#pragma_synchronous)

WAL 的作用范围是本地数据库恢复，不包含跨机器复制或写权转移。跨主机 Worker 共享任务，应访问提供并发控制的数据库服务。将 SQLite WAL 文件放在共享网络目录，不能替代分布式任务协调。

提交确认也可能丢失。数据库已成功保存 A 的领取，网络却未将成功响应交给 A，此时 A 无法确定自己是否拥有 T。它应按稳定请求标识查询权威记录，核对 owner 与执行代次，再决定后续动作；执行代次是每次所有权换代时递增的编号。未经确认直接执行或无条件新建领取，都会绕过所需的执行权判断。

## 7.3 租约、心跳与任务接管

租约（lease）为执行权规定有效期限，避免 Worker 退出后任务永久处于运行中。本例在领取时保存 owner=A、执行代次 7 和到期时间。A 在期限内申请续租，权威数据库原子检查 owner、代次及有效期仍匹配后，才延长租约。接管也经过同一权威存储，避免各 Worker 根据各自时钟独立决定所有权。

心跳（heartbeat）是周期性的活性通知，可以携带续租请求。收到心跳只表明有消息到达，续租还可能因数据库不可用、代次变化或租约已过期而失败。A 需要成功确认才能知道租约延长；调度端长期收不到心跳，也无法仅凭沉默区分进程退出、网络故障与长时间暂停。

假设租约期限为 30 秒，A 每 10 秒申请续租，这些数值仅为教学参数。A 失联后，期限不再延长；到期后 B 通过条件更新取得 T，将代次增加为 8，并共同提交新期限和接管记录。并发申请者中只有更新成功者可以继续。B 随后读取检查点与工具结果，确定恢复阶段。租约确定恢复资格，检查点确定恢复位置；未提交的计算可能重做，未知外部效果仍需查询或幂等协议。

租约期限决定接管速度与误判风险。期限较短可缩短故障等待，却更容易把短暂暂停当成失联；较长期限减少误接管，但延长恢复等待。频繁续租还会增加存储负载。无法确认续租的执行者应停止发起新工作，但安全性不能仅依赖其主动停止：暂停中的进程无法运行停止逻辑，已经发送的请求也未必能够撤回。

## 7.4 隔离令牌与迟到写入校验

B 以代次 8 接管后，A 可能恢复运行，并尝试提交代次 7 的报告。也可能是 A 在暂停前发出的请求此时才到达。开始执行时的一次租约检查无法限制这些迟到写入，接收方还需要在实际修改资源时校验执行资格。

隔离令牌（fencing token）是随执行权换代而单调增加的凭证，本例采用权威存储分配的执行代次。每次受保护写入都携带该令牌，由资源接收方检查。结果存入任务数据库时，完成事务应同时核对当前 owner、代次和租约有效期；代次已为 8，就拒绝携带 7 的更新。校验和写入必须原子完成，避免在单独检查后又发生接管。

外部文件服务不受任务数据库的条件更新保护，需要自行参与令牌校验。一种方式是持久保存已接受的最高代次，并拒绝更低代次；服务接受 8 后，便拒绝 7。这个规则不保证在数据库刚完成接管时就立即拒绝所有旧请求。若需要该保证，必须先向资源安装新代次，或在写入时原子核验当前授权。只有携带数字而不执行校验，无法限制旧执行者。

令牌校验限制失效执行者继续修改受保护资源，不会撤销其过去已经完成的动作，也不保证整个任务只执行一次。已发生的重复效果仍由幂等与恢复协议处理。

![租约续租失败后由 B 接管为代次 8；资源接受 8 后拒绝 A 携带 7 的迟到写入](figures/07-concept-lease-fencing.svg)

图中的文件服务以自己已接受的最高代次为判断依据。在图示时序中，服务接受代次 8 后才据此拒绝代次 7 的迟到请求。

## 7.5 数据复制与读取一致性

B 完成报告后，权威数据库保存“已完成”。为减轻主存储读取压力，服务将数据复制到另一份数据库，并让进度页面查询副本。复制（replication）使多处保有同一数据，但传播和应用更新需要时间，所以主存储提交后，页面仍可能暂时显示“运行中”。

一致性（consistency）规定读取允许观察到哪些写入。任务领取需要在权威存储执行原子条件更新，即使 Worker 从旧副本发现 T 待处理，也不能直接开始执行。进度展示则可能允许一定滞后。用途不同，可采用不同的读取保证。

本章用线性一致读写具体说明“强一致”：B 收到完成提交的成功响应后才发起的任务读取，应反映这次或更晚的写入。并发操作仍需确定先后次序，已经开始的读取也不会自动刷新。这项保证只覆盖服务承诺的操作范围，不会自动把数据库、邮件服务或多个状态对象组合为事务。服务需要确认状态足够新；遇到故障或网络分区时，可能因此等待或拒绝请求。

最终一致的副本允许提交后暂时返回旧状态。在停止写入且复制持续正常时，各副本最终应取得完成状态，但该承诺没有规定最多滞后几秒，也不保证切换副本后版本只增不减。若产品要求用户提交后立即看见自己的结果，可将后续读取发到权威存储，或携带提交版本并等待副本至少应用到该版本。后一方式增加读取延迟，还需规定等待超时后的行为。

![WAL 恢复属于本地数据库层，主库到只读副本则是独立的复制与读取路径](figures/07-concept-wal-replica.svg)

图示将本地恢复与数据复制分开：WAL 用于数据库崩溃恢复，主存储到副本的传播过程决定副本何时可见更新。

同步复制让提交等待指定副本确认，可降低某些故障下已确认数据丢失的风险，同时增加写入等待。异步复制缩短等待，但主存储故障时可能仍有提交未传出。若直接提升落后副本为新主，已经领取的任务甚至可能重新显示为待处理。因此，故障切换还必须确定新主资格、认可的提交范围，并隔离旧主。

共识协议负责让参与节点在故障条件下对一系列决定达成约定；Raft 是一种常见实现。复制数据并不自动提供共识，数据库产品需要把复制、故障检测和主节点切换配合起来。本章无需实现完整共识协议，但要区分：本地 WAL、数据库事务、副本读保证和主节点选举各自解决不同步骤。作为可核对的系统实例，[etcd 的 API 保证](https://etcd.io/docs/v3.5/learning/api_guarantees/) 区分线性一致读取与可能读到旧状态的读取；这不是 Pi 的依赖或现有能力。

## 7.6 参考实现：单进程提交与本地所有权

Pi 实验 durable 库以单进程拥有 storage 为前提，README 明确“同一时刻只有一个进程拥有 storage，没有跨进程锁”（[packages/durable/README.md L772](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/README.md#L772)）。它没有跨主机任务租约和接管协议，不应据此把默认 CLI 视为分布式执行器。

Session 通过串行提交线组织状态修改：`commitWith()` 把提交排入一条队列（[packages/durable/src/session/session.ts L232-L244](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/session/session.ts#L232-L244)），队列由 `#enqueue` 串行执行并进入 `#runCommit`（L880、L716）。SQLite 存储已迁到 chord 包，Node 适配器在事务开始时执行 `BEGIN IMMEDIATE`（[packages/chord/src/sqlite/node.ts L141](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/sqlite/node.ts#L141)），打开时设置 `PRAGMA journal_mode = WAL` 与 `PRAGMA synchronous = NORMAL`（L183-L190）。durable 的 `openNodeSqliteStorage` 只是包一层并支持可选表名前缀（[packages/durable/src/storage/sqlite/node.ts L14-L22](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/storage/sqlite/node.ts#L14-L22)）。相关记录因此共享提交边界，重开后可以恢复；掉电仍可能丢失最近提交。

Storage 成功后 Session 才采用内存状态并发布更新，conversation view 随 commit 改变。扩展为分布式 Worker 时，仍需自行实现共享任务表的原子领取、续租、执行代次校验和跨主机接管；状态发布序号与 Chord 的 codeVersion 都不能充当执行权凭证。

## 7.7 参考实现：Chord 的状态副本

任务输出可能依次变为 `a`、`ab`、`abc`，而界面在任一时刻加入。权威侧维护可修改状态，界面先取得当前内容，再接收后续更新。观察副本用于展示，不能授权进程执行任务或修改持久状态。

Chord 用“对象 + 引用”组织跨进程状态：对象是某个契约的实例，契约声明对象住在哪个宿主（`hostNamed`/`hostFamily`），调用方只持有引用（[packages/chord/docs/guide.md L102-L113](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/docs/guide.md#L102-L113)）。每个对象还带主体（principal）：同一地址对 owner 和 guest 是不同对象，链路边界负责盖章，调用方无法自选（guide L356-L361）。远端引用先拿到该变体每个 state 成员的一份快照，之后接收增量；重连或重载后收到 `reset` 快照，不重放历史（guide L452）。

Chord 的 `replicatedState` 既可以是权威可变状态（`change`/`replace`），也可以是外部修订流的只读投影：`replicatedState(source)` 要求 source 原子地 `attach()` 出快照并缓冲之后的每一帧（[packages/chord/src/state.ts L55-L78](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/state.ts#L55-L78)）。durable 用它承载 document、view 和 task-graph 状态：会话文档在 [packages/durable/src/session/session.ts L372](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/session/session.ts#L372) 处把内部修订流转成 `DocumentState`，view 与 task-graph 分别在 [packages/durable/src/harness/view.ts L118](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/view.ts#L118) 和 [packages/durable/src/harness/task-graph.ts L116](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/harness/task-graph.ts#L116) 挂接同一机制。

两张图使用不同的示意序号：42、43、44 说明基线后的连续更新及缺帧后的恢复；10、11展示实际订阅建立顺序。箭头表示状态交付，不表示副本取得写权。

![副本先安装序号 42 的快照，再应用 43 和 44；缺号后需重建基线](figures/07-snapshot-delta.svg)

![Chord 订阅时先安装序号 10 快照，再激活并按序接收序号 11 更新](figures/07-subscription-sequence.svg)

## 7.8 参考实现：草稿修改与原子发布

同一次状态修改应保持相关字段的一致性。例如任务从“读取中”变成“读取完成”时，应同时发布结果引用。逐字段发布可能使观察者暂时看到完成状态却没有结果，因此发布端先准备完整变化，再统一采用。

Chord 的 state cell 提供草稿式修改：`change(ctx, fn)` 打开的草稿只在回调期间可用，回调抛错则整体放弃，成功时恰好发布一个 revision（[packages/chord/src/cell.ts L194-L219](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/cell.ts#L194-L219)）。`change` 只发布被改动的路径，代码不认识的字段得以保留；`replace` 接管整个值，在新值与旧值共享容器（同一状态的不可变新版本）时默认发送结构 diff，也可用 `{ diff: true | false }` 强制（L221-L235）。reload 迁移后的值走 `reset`，硬性建立新基线（L237-L244）。`replicatedState` 的 `change`/`replace` 采用同一模型（[packages/chord/src/state.ts L255-L285](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/state.ts#L255-L285)）。

这种交接依赖不可变值的所有权约定。状态发布后，使用方不能再修改同一个对象，否则同一 revision 对不同观察者可能呈现不同内容。`track()`、`replicatedState()` 和 `replace()` 直接接管别名自由的严格 JSON 根，不做防御性复制也不冻结；放进草稿的外部值会先验证再复制，细节见 [packages/chord/src/delta/README.md L28-L41](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/delta/README.md#L28-L41)。

不可变性依赖调用方遵守所有权约定，运行时并未对所有对象执行 deep freeze，TypeScript 只读类型也不会冻结对象。进程内直连可能让提供端与消费端共享容器，消费端修改它会破坏权威状态。向会修改输入的代码交付值时，应先复制或序列化。

增量是一批相对于前一版本的操作：整值替换 `r`、set/delete、字符串追加 `a` 与按长度截断 `t`，以及数组 splice `p`/permutation `m`（[packages/chord/src/delta/index.ts L21-L40](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/delta/index.ts#L21-L40)）；[packages/chord/src/delta/index.ts L409-L419](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/delta/index.ts#L409-L419) 通过 copy-on-write 应用操作，保留前一版不可变值。输出从 `"a"` 变为 `"ab"` 时，传输时可以只发送 append 操作，副本应用后仍得到完整值 `"ab"`。操作只要求在正确基线上按序应用后结果一致，编码形式不必唯一或最小，也不代表业务事件的含义。

## 7.9 参考实现：快照安装与连续增量交付

建立订阅需要协调快照读取和监听登记。若先读取 `ab`，再登记监听，两步之间发布的 `abc` 可能遗漏；若先交付增量，再安装旧快照，又可能用 `ab` 覆盖较新状态。提供端需要原子捕获快照并登记订阅，消费端则在快照安装完成后启用增量交付。

Chord 用单位（unit）交付解决这一竞争：一个 hold（引用）隐含其变体中每个 state 成员的状态，发送时先交付 shape 帧，再交付每个消费者尚缺的成员快照（[packages/chord/src/wire.ts L9-L11](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/wire.ts#L9-L11)）。快照单元是懒构建的，在发送时才从当前值构建；构建期间到达的事件在快照之后按序重放（[packages/chord/src/connection.ts L46-L62](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/connection.ts#L46-L62)、[packages/chord/src/connection.ts L89-L95](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/connection.ts#L89-L95)）。

消费侧用 hydration 表示“该状态已有值”。一个引用只有在对象 live 且其变体的每个 state 成员都已 hydrate 后才 resolve；在首次 hydrate 前读取会得到 `not_ready`，等待中的引用在 `readyTimeout`（默认 60 秒）后以 `unavailable/ready_timeout` 失败（guide L276-L281、[packages/chord/src/host.ts L133-L137](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/host.ts#L133-L137)）。本地对象也只有在资源打开、声明的引用就绪、`init` 运行完之后才 live，因此方法、监听和任务都看不到半成品状态（guide L247-L256）。

对外部修订流的投影，完整性用游标验证：`replicatedState(source)` 固定快照的 cursor，并要求每一帧恰好是上一个 cursor 加一（[packages/chord/src/state.ts L326-L339](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/state.ts#L326-L339)）。出现缺口时按 source 契约失败处理：停用附件并上报错误，最后成功的值仍可读但不再更新（[packages/chord/src/state.ts L341-L359](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/state.ts#L341-L359)），而不是继续使用可能错误的值。恢复不是自动的：source 的所有者需要重建附件并重新投影。连续游标用于发现缺失，但不能验证业务内容正确，也不构成多写入方共识。

## 7.10 参考实现：副本重建与代码换代

缺帧或重连后，提供端会给远端引用发送 `reset` 快照，副本建立新基线，后续增量再按序应用（guide L452）。界面可以保留引用句柄：引用在重新持有期间状态为 `loading`，最后的值仍可读；`gone` 表示对象被强制移除（吊销、归档、宿主死亡），需要重新 `get` 或显示“不可用”（guide L418-L424）。

Chord 的“实例换代”由代码版本（codeVersion）表达。每次 `host.reload()` 产生一个新的代码版本：已开始的 root 固定在启动时的代码版本上跑完，新 root 使用新代码（guide L630-L640）。如果某个 cell 的版本号升高，或独占资源发生变化，reload 先进入 gate：等正在访问该对象的旧 root 结束、运行 `migrate`、再同步提交；超时则整个 reload 中止、什么都不改（guide L642-L675）。改 schema 而不升版本会被拒绝，改语义而不升版本无法检测。旧代码版本在它的 root 全部结束后退休；迟迟不结束的 root 至多再等 `retireDeadline`，之后被中止（[packages/chord/src/reload.ts L23-L36](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/reload.ts#L23-L36)）。root 的代码版本随调用传播（[packages/chord/src/wire.ts L32](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/wire.ts#L32)）：旧 root 碰到已经迁移的对象时得到 `upgrade_conflict`，需要重试为新的 root。

集群 reload 由 server 统一协调：server 先发送 prepare，每个子宿主准备好后投赞成票（vote），server 提交后子宿主依次提交（[packages/chord/src/cluster.ts L130-L157](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/cluster.ts#L130-L157)、[packages/chord/src/cluster.ts L173-L206](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/cluster.ts#L173-L206)）。任何一处失败或超时，全体中止。落后于 server 的子宿主在放行流量前先被追到当前代码版本（guide L746）。浏览器不参与提交，只通过 shape 与 `reset` 感知新代码；契约不兼容时浏览器调用得到 `upgrade_required`（guide L754）。

契约兼容性由 fingerprint 与覆盖检查把关：同版本要求提供方覆盖消费方的全部成员；破坏性变更必须升版本并提供 `accepts` 适配器，否则绑定以 `contract_mismatch` 失败（[packages/chord/src/contract.ts L288-L349](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/contract.ts#L288-L349)；guide L136-L161）。类型住在哪个宿主也是契约（及其 fingerprint）的一部分：换宿主等于换类型，旧引用得到 `upgrade_required`（guide L102-L113）。

codeVersion 只标记代码世代，不承担持久任务租约代次的职责。

## 7.11 参考实现：慢消费者与队列积压

慢订阅者的处理策略取决于交付内容。完整值能够独立解释，可以舍弃中间版本而保留最新值；增量依赖基线，直接丢弃中间操作会破坏状态。Chord 因此在连接和订阅两个层面分别限流。

连接层面按确认窗口限流。链路双方在 hello 里声明自己会确认收到的帧（[packages/chord/src/wire.ts L60-L70](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/wire.ts#L60-L70)）；未确认 frame 字节超过 window（默认 256 KiB）时，state 帧等待，并按每个成员的流合并（[packages/chord/src/connection.ts L12-L36](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/connection.ts#L12-L36)）。等待中的 op 数量还有 pendingCap 上限（默认 4 个 window）；超出的最老流被替换为“当前值”的 reset，内存因此有界（溢出回收的类型见 [packages/chord/src/connection.ts L40-L44](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/connection.ts#L40-L44)）。调用、结果和新引用的第一个快照永远不被窗口拦住，一个 `get` 不会排在别的引用的状态后面（guide L744）。这也意味着一项修改在链路满载时，调用返回可能先于其引发的状态更新到达：要等状态就订阅状态，不要假设调用返回时状态已到。

| 位置 | 交付内容 | 溢出后的处理 |
| --- | --- | --- |
| 链路 state 帧 | 精确操作，按成员的流合并 | 未确认字节超过 window 时等待；等待 op 超过 pendingCap 后，最老的流换成当前值的 reset |
| 公开 state 订阅（replicatedState） | 完整值、上下文与交付元数据 | 100 个 pending 后清空旧 pending，只保留最新；未开始的初次 hydration 保留 |
| 链路上的调用名额 | 进行中的调用 | 超过 maxCalls 的调用立即以 `unavailable`（原因 `too_many_calls`）答复，链路保持 |

公开订阅交付完整值，因此允许跳过发布序号：[packages/chord/src/state.ts L20-L27](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/state.ts#L20-L27) 规定每个订阅最多保留 100 个未交付完整值（不含运行中的回调），溢出时只留最新，[packages/chord/src/state.ts L115-L124](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/state.ts#L115-L124) 实现该策略。它是帧数策略而不是字节上限。[packages/chord/src/connection.ts L76-L80](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/connection.ts#L76-L80) 的 maxCalls 是每条链路的调用名额：超出者不会执行，但链路不受影响。应用自己在链路之上再加的异步队列由应用定义，不自动继承这些限制。

## 7.12 参考实现：编码、校验与边界

传输编码有两层优化。第一层是负载压缩：JSON 长度超过 `PACK_THRESHOLD`（64K 字符）的负载被 deflate；只转发、不使用负载的宿主直接传递压缩字节（`Packed`），真正使用负载的宿主再解压（`unpack`），浏览器侧用 `DecompressionStream` 解压后仍按序交付（[packages/chord/src/link.ts L105-L130](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/link.ts#L105-L130)）。

第二层是 delta 字典：encoder 在路径第二次出现时引入 ID，decoder 必须先收到配套定义（[packages/chord/src/delta/index.ts L542-L659](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/delta/index.ts#L542-L659)）。ID 与省略写法都只在一批之内有效，唯一跨批的状态是路径字典；一个基础操作 `r` 会清空字典，使该批成为恢复点（L583-L591）。同一链路承载多个独立状态时，每个状态有自己的编码/解码对（L551-L554）——不同消费者的编码进度可能不同，不能用一个有状态编解码器编码一次后复用于所有客户端。解码端逐操作校验形状与保留路径（[packages/chord/src/delta/index.ts L118-L142](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/delta/index.ts#L118-L142)、[packages/chord/src/delta/index.ts L144-L174](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/delta/index.ts#L144-L174)），拒绝 `__proto__`、`constructor`、`prototype` 这类可污染原型的路径。

链路握手也带版本：两端在 hello 里声明 link 协议版本（当前为 4，[packages/chord/src/wire.ts L19-L24](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/wire.ts#L19-L24)）；版本不同直接关闭链路，由 `onProtocolMismatch` 通知需要换代码的一侧（[packages/chord/src/connection.ts L27-L31](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/connection.ts#L27-L31)），例如浏览器刷新页面而不是重连。

本章的有序读副本协议以单一权威源为前提，以下能力需要另外的机制：

- 多写者数据库冲突处理没有实现，包括 Raft/Paxos 共识、leader election、quorum、CRDT 合并和网络分区下的多主写入。Chord 对象只有唯一权威宿主；durable 的 Session SQLite 由单进程持有，跨进程共享存储需要另行协调。
- 一次 `change` 只形成单个 cell（或单个 replicated state 对象）的一次原子 revision；跨多个对象或成员的强一致事务没有实现。
- `codeVersion`/reload 编号保存在宿主内，只标记代码世代；它不提供跨主机 worker lease 或持久 leader term。
- 流控窗口是每条链路的，public 订阅上限是每个订阅的帧数策略：它们给内存设了上界，但都不是端到端字节背压。
- variant 不是完整授权：方法仍要校验参数；主体在链路边界盖章，调用方无法自选（guide L356-L361、L389-L392）。

## 7.13 编号约定与源码定位

本章会遇到几种编号。它们记录不同对象，没有统一的全局含义：

| 标识 | 记录什么 | 范围 |
| --- | --- | --- |
| Session commit `Seq` | 原子数据库批次，编号可有间隙 | 不要求等于任何远端 state 的发布序号 |
| source cursor | attached source 的连续帧锚点 | 每个附件独立，没有跨 storage 全局时钟含义 |
| member sequence | 一个 state 流的发布基线 | public 订阅允许跳号；没有多个成员的事务边界含义 |
| codeVersion / reload id | 宿主的代码世代 | 不是任务续租，也不是共识 term |
| wire frame id | 一次 hold/call/sub 的编号 | 由发起侧选择，只在一条链路内唯一 |

## 7.14 本章术语

| 通用概念 | 定义 |
| --- | --- |
| 原子领取 | 将执行资格检查与所有权更新作为不可分割操作 |
| 租约 | 具有有效期限、按协议续期的执行权 |
| 隔离令牌 | 随执行权换代增加，由资源端核验的写入凭证 |
| 线性一致性 | 已完成写入对随后开始的读取可见，操作顺序符合实时先后关系 |
| 最终一致性 | 停止写入且复制持续正常时，副本最终收敛 |
| 快照与增量 | 完整状态基线，以及相对此前版本的变化 |

以下表格对应 Pi 的具体实现范围。

| 术语 | 当前实现 | 范围与限制 |
| --- | --- | --- |
| Transaction | Session mutation line 串行提交；chord SQLite 适配器用 `BEGIN IMMEDIATE` / `COMMIT` 原子落盘 | 跨主机、跨外部服务的分布式事务尚未实现 |
| WAL | chord SQLite Node 适配器设置 `journal_mode = WAL` 与 `synchronous = NORMAL` | 掉电仍可能丢最新 commit；这份日志用于本地数据库 |
| Lease | durable task 没有可到期和续租的跨 worker 租约；Chord 的引用生命周期是引用计数式回收 | 都不负责任务执行权租约、多主共识或全局 fencing |
| Heartbeat | 链路 ping 与沉默超时（`pingInterval` 15 秒、`pingTimeout` 45 秒）判定连接死亡，并释放经其持有的全部引用 | 只证明连接活性，无法证明任务仍有执行资格 |

Session 提交线在 [packages/durable/src/session/session.ts L232-L244](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/durable/src/session/session.ts#L232-L244)，SQLite 事务与 WAL 设置在 [packages/chord/src/sqlite/node.ts L141](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/sqlite/node.ts#L141)、[packages/chord/src/sqlite/node.ts L183-L190](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/sqlite/node.ts#L183-L190)。链路 ping 与超时见 [packages/chord/src/connection.ts L13-L16](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/src/connection.ts#L13-L16)；环境进程层的连接清理见[第 8 章 沙箱与执行环境](08-sandbox-environment.md)。

## 7.15 练习

先画出任务 T 的记录变化：待处理，A 以代次 7 领取，续租失败，到期后 B 以代次 8 接管，A 的迟到结果被拒绝。分别标出每一步由谁检查、哪些字段必须在一次事务中更新。再设想文件服务不检查代次，指出数据库拒绝 A 的完成提交后，仍可能发生什么外部写入。

让 B 完成任务后立即打开进度页：主存储已提交，副本尚未应用该次提交。写出强一致读取与最终一致读取各自允许返回的结果，并说明为什么从副本看到“待处理”不能直接授权 Worker 执行。

可以在纸上维护 `{ output: "a" }` 的 source 投影：attach 时快照 cursor 为 10，依次收到 11、12。若 12 缺失而先来一帧 13，投影按 source 契约失败：最后成功的值仍可读，但不再更新，需要源端重建附件后重新 attach。再试一次让一帧携带完整值而不推进 cursor，检查为什么完整值仍无法免去游标校验。

下面的测试可以用来核对这些协议行为。

| 阅读位置 | 可观察的协议行为 |
| --- | --- |
| [packages/chord/test/flow.test.ts L125-L173](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/test/flow.test.ts#L125-L173) | 窗口满时 state 等待而调用照常；释放后合并为一帧 |
| [packages/chord/test/flow-units.test.ts L93-L107](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/test/flow-units.test.ts#L93-L107) | 大于窗口的快照整帧到达；之后的变更等待确认 |
| [packages/chord/test/reload.test.ts L177-L216](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/test/reload.test.ts#L177-L216) | 状态升版本时的 gate、migrate 与旧 root 的 upgrade_conflict |
| [packages/chord/test/cluster.test.ts L130-L159](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/test/cluster.test.ts#L130-L159) | 跨宿主 reload：每个宿主 prepare，server 先提交，代码版本保持同步 |
| [packages/chord/test/generations.test.ts L50-L75](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/test/generations.test.ts#L50-L75) | 跨 reload 的引用只保留当前代码世代 |
| [packages/chord/test/state-draft.test.ts L4-L38](https://github.com/earendil-works/pi/blob/482c1381a00d8d00bc81697e4a980c593db6803e/packages/chord/test/state-draft.test.ts#L4-L38) | 事务式草稿只复制改动的分支；插入值不与原引用共享 |

若自行运行，从 `packages/chord` 执行 `node ../../node_modules/vitest/dist/cli.js --run test/flow.test.ts`。这是本地协议测试，不需要登录、真实模型或远端部署。

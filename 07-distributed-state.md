# 第 7 章　分布式状态与一致性

[返回目录](README.md) · [上一章：第 6 章 重试、超时与幂等](06-retry-timeout-idempotency.md) · [下一章：第 8 章 沙箱与执行环境](08-sandbox-environment.md)

多个执行者共享任务时，系统需要确定状态更新如何共同提交、哪一个执行者拥有处理资格，以及读到的状态允许落后多久。本章先以两个 Worker 的任务领取和故障接管说明事务、租约与写入校验，再讨论数据库复制和一致性；随后介绍 Pi 的单进程 durable 存储与 Chord 只读状态同步，区分任务执行权和界面观察状态。

假设 Agent 服务收到“读取资料并生成报告”的任务，将它存入数据库。Worker A 和 B 都能查询待办，用户在网页查看进度。两者可能同时发现同一任务，A 也可能在运行中失联，再于 B 接管后恢复；与此同时，页面还可能显示数据库副本中的旧状态。这些情况分别涉及并发领取、执行权换代和读取可见性，需要配合处理。

下面先用共享数据库的双 Worker 系统说明通用设计。示例中的任务表、跨主机租约与接管协议属于教学架构，Pi 当前没有实现这套分布式调度；后半章再对照 Pi 的实验 durable 存储与 Chord 只读状态复制。本文的源码基线为提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc`，持久化的提交顺序见[第 5 章 持久执行与恢复](05-durable-execution.md)，客户端与工具边界见[第 9 章 工具基础设施与 MCP](09-tools-mcp.md)。

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

Pi 实验 durable 库以单进程拥有 storage 为前提，实验 Worker 使用本地目录锁维护 Session 所有权。它尚未实现前述跨主机任务租约和接管协议。以下介绍 durable 与 coding-agent services 的实验路径，不应据此将默认 CLI 视为分布式执行器。

Session 通过串行提交线组织状态修改，Node SQLite 使用 `BEGIN IMMEDIATE` / `COMMIT` 原子提交一个批次，并配置 WAL 与 `synchronous = NORMAL`。相关记录因此共享提交边界，重开后可以恢复；掉电仍可能丢失最近提交。Storage 成功后，Session 才采用内存状态并发布更新，conversation view 随 commit 改变。

实验目录锁的 stale 检测与周期 update 维护本地所有权，env 连接的定期 ping 与沉默超时负责连接或进程清理。这些机制不包含共享任务表的原子领取、续租、执行代次校验和跨主机接管。扩展为分布式 Worker 时，仍需实现这些职责；状态发布序号和服务实例编号不能充当执行权凭证。

## 7.7 参考实现：Chord 的界面状态副本

任务输出可能依次变为 `a`、`ab`、`abc`，而界面在任一时刻加入。Worker 维护可修改的权威状态，界面先取得当前内容，再接收后续更新。界面的只读副本用于观察，不能授权进程执行任务或修改持久状态。

Chord 在实验 coding-agent 中负责将 Worker 会话视图交付给客户端。首次传输完整快照，随后可以发送增量（delta），例如在已有 `ab` 末尾追加 `c`。增量依赖前一版本，因此消费者必须先安装快照，再依序应用变化。中途加入的界面由此获得当前状态，无须重放全部业务事件。

durable Storage 负责保存与恢复，Chord 把当前 view 同步给观察者。普通 `replicatedState(initial)` 本身不执行磁盘持久化，可以连接任意权威数据源。实验服务在 [packages/coding-agent/src/experimental/services/transcript-provider.ts L5-L15](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/experimental/services/transcript-provider.ts#L5-L15) 中通过 `createTranscriptServiceFacet()` 直接取得 `conversation.viewState()` 作为 Transcript.state，并在退出时 dispose。Delta 的有序交付、丢包与乱序处理、重新连接和订阅，由协议和宿主共同负责。

两张图使用不同的示意序号：42、43、44 说明基线后的连续更新及缺帧后的恢复；10、11展示实际订阅建立顺序。箭头表示状态交付，不表示副本取得写权。

![副本先安装序号 42 的快照，再应用 43 和 44；缺号后需重建基线](figures/07-snapshot-delta.svg)

![Chord 订阅时先安装序号 10 快照，再激活并按序接收序号 11 更新](figures/07-subscription-sequence.svg)

## 7.8 参考实现：草稿修改与原子发布

同一次状态修改应保持相关字段的一致性。例如任务从“读取中”变成“读取完成”时，应同时发布结果引用。逐字段发布可能使观察者暂时看到完成状态却没有结果，因此发布端先准备完整变化，再统一采用。

草稿（draft）保存一次修改尚未发布的候选状态。调用方修改草稿，准备完成后统一采用并发布；回调抛错则放弃草稿，观察者继续读取原状态。`state.change` 可同时修改多个字段，形成该 state 对象的一次原子 revision（版本变化）。prepare 后没有操作时，直接返回且不增加发布序号。

[packages/chord/src/services/state.ts L184-L244](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/state.ts#L184-L244) 的 `MutableReplicatedStateImpl.change()` 打开 draft，运行 mutate，准备候选值，再采用并发布。回调返回 Promise 也会被拒绝，因此网络等待应放在这个同步回调之外。

这种交接依赖不可变值的所有权约定。状态发布后，使用方不能再修改同一个对象，否则同一 revision 对不同观察者可能呈现不同内容。传给 `track()` 或 `replicatedState()` 的 initial root 转交给 tracker，发布后的 value 和 ops 都不能再改。放进 draft 的外部值会先验证再复制，细节见 [packages/chord/src/delta/README.md L1-L41](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/delta/README.md#L1-L41)。

不可变性依赖调用方遵守所有权约定，运行时并未对所有对象执行 deep freeze，TypeScript 只读类型也不会冻结对象。进程内直连可能让提供端与消费端共享容器，消费端修改它会破坏权威状态。向会修改输入的代码交付值时，应先复制或序列化。

增量是一批相对于前一版本的操作。[packages/chord/src/delta/index.ts L21-L41](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/delta/index.ts#L21-L41) 定义整值替换、set/delete、字符串追加与截断，以及数组 splice/permutation；[packages/chord/src/delta/index.ts L408-L435](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/delta/index.ts#L408-L435) 通过 copy-on-write 应用操作，保留前一版不可变值。输出从 `"a"` 变为 `"ab"` 时，传输时可以只发送 append 操作，副本应用后仍得到完整值 `"ab"`。操作只要求在正确基线上按序应用后结果一致，编码形式不必唯一或最小，也不代表业务事件的含义。

## 7.9 参考实现：快照安装与连续增量交付

建立订阅需要协调快照读取和监听登记。若先读取 `ab`，再登记监听，两步之间发布的 `abc` 可能遗漏；若先交付增量，再安装旧快照，又可能用 `ab` 覆盖较新状态。提供端需要原子捕获快照并登记订阅，消费端则在快照安装完成后启用增量交付。

例如，提供端捕获值 `ab` 与序号 10，同时登记订阅，随后产生的序号 11 暂存。消费者先完成 hydration，即用完整快照建立副本基线，再调用 `activate` 开始接收 11。该顺序同时避免漏更新和旧快照覆盖新内容。

[packages/chord/src/services/provider.ts L239-L276](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/provider.ts#L239-L276) 将 subscriber 加入注册表、捕获 snapshot、记录快照覆盖的 state sequences，再返回 activate/close。[packages/chord/src/services/provider.ts L424-L443](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/provider.ts#L424-L443) 中，每个 state member 的快照带完整 `r` 和当前 sequence；后续发布通过已经登记的精确操作监听器交付。

consumer 的 singleton 入口在 [packages/chord/src/services/consumer.ts L594-L633](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/consumer.ts#L594-L633)。它先校验并安装 snapshot，再 activate。实际接收增量的是 [packages/chord/src/services/state.ts L376-L415](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/state.ts#L376-L415) 的 replica，也就是只读副本。它安装完整快照后，要求每个更新序号恰好比上次多一。

连续序号用于验证增量交付的完整性。基线为 `a` 时，后续操作依次追加 `b` 和 `c`；若漏掉第一步，直接应用第二步会得到错误的 `ac`。序号跳变使消费者在使用错误值之前发现缺失，但不能验证业务内容正确，也不构成多写入方共识。

普通 `state` update 即使携带完整 `r`，仍必须遵守严格的 `+1` 连续序号。完整值不能让一个普通 update 自动变成新基线，因为序号之间可能还有实例生命周期变化。只有显式 snapshot/reset hydration 才能建立新起点。收到重复、缺号、无效操作或无效 JSON 结果时，replica 会清空 value 并报错；它不会自行向提供端请求 snapshot。主机或 binding 负责重新绑定、订阅；[packages/chord/src/services/consumer.ts L540-L570](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/consumer.ts#L540-L570) 展示了这条重建路径。

Chord 的实现见 [packages/chord/README.md L1-L7](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/README.md#L1-L7) 与 [packages/chord/README.md L32-L57](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/README.md#L32-L57)。它是独立的应用组合包，不依赖其他 Pi workspace 包，负责让一个权威状态源向本地和远端消费者发布快照、增量，并维护序号、实例换代和重新绑定。实验 coding-agent services 用它把 worker 的会话视图交给客户端；durable 和这组 services 的入口仍属于实验层。

## 7.10 参考实现：副本重建与服务实例换代

缺帧使副本清空后，主机应重新订阅并安装快照，或交付显式 reset。reset 为状态建立完整新基线，后续增量再按连续序号应用。重新绑定同样先清空旧值，再等待新快照；界面可保留句柄，但需要识别数据尚未就绪的阶段。

断线还可能让旧句柄指向错误的服务实例。假设某个 key 对应的子服务关闭，后来以相同 key 创建新服务。只检查 key 的旧客户端可能把针对旧工作的调用发给新工作。当前 provider 给复用 key 的实例分配新的 generation，区分两段生命周期；[packages/chord/src/services/provider.ts L183-L212](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/provider.ts#L183-L212) 在复用 key 时增加 generation，[packages/chord/src/services/provider.ts L377-L409](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/provider.ts#L377-L409) 用它拒绝旧实例调用。这份编号保存在当前 provider 的 Map 中，用于限制旧服务实例访问。

单例服务替换保留对外接口对象，并安装新的状态快照。消费端在重新绑定期间清空旧值，因此接口句柄可以持续存在，数据却暂不可用。generation 只标记服务实例生命周期，不承担持久任务租约代次的职责。

## 7.11 参考实现：慢消费者与队列积压

慢订阅者的处理策略取决于交付内容。完整值能够独立解释，可以舍弃中间版本而保留最新值；增量依赖连续基线，直接丢弃中间操作会破坏状态。因此 public state subscription 合并积压的完整值，provider remote subscription 则在积压过多时发送显式 reset。

[packages/chord/src/services/state.ts L28-L95](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/state.ts#L28-L95) 与 [packages/chord/src/services/state.ts L149-L181](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/state.ts#L149-L181) 为每个 public subscriber 串行调用 callback，正在执行的回调不计入 pending。其他订阅者和同步的精确操作监听器仍能继续处理发布。

| 位置 | 交付内容 | 溢出后的处理 |
| --- | --- | --- |
| public state subscription | 捕获的完整 value/context/delivery | 100 个 pending 后清空旧 pending，保留最新；未开始的 hydration 可保留 |
| provider remote subscription | 精确 state 操作与实例 lifecycle updates | 新增第 101 个 pending 时换成一个显式 reset，包含当前完整订阅 snapshot |
| 应用 transport 的异步队列 | 宿主自行定义 | 不自动继承上述上限 |

公开订阅交付完整值，因此允许跳过发布序号。远端订阅交付精确增量，必须保持连续；[packages/chord/src/services/provider.ts L445-L465](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/provider.ts#L445-L465) 在 pending 达到上限时用 reset 替换积压，给每个状态建立新基线。reset 还会重建 keyed membership，避免遗漏 close/spawn 后仍保留已经不存在的实例。上限按帧数计算，较大的 snapshot 仍可能占用较多字节。应用 transport 的异步队列由宿主定义，不自动继承这两处限制。

## 7.12 参考实现：编码字典、校验与边界

Delta encoder 在路径第二次出现时引入 ID，decoder 必须先收到配套定义；具体过程见 [packages/chord/src/delta/index.ts L506-L553](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/delta/index.ts#L506-L553) 与 [packages/chord/src/delta/index.ts L622-L662](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/delta/index.ts#L622-L662)。同一连接承载多个独立状态时，每个状态都有自己的字典。不同客户端的编码进度可能不同，因此不能用一个有状态编解码器编码一次后复用于所有客户端。

[packages/chord/src/services/state-codec.ts L27-L58](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/state-codec.ts#L27-L58) 按 instance/member 创建 codec；[packages/chord/src/services/state-codec.ts L60-L137](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/services/state-codec.ts#L60-L137) 在 snapshot、reset、replacement、unavailable、closed 时重置或移除对应字典。快照、重置和实例替换会处理字典生命周期。它是传输编码状态，不是任务检查点。解码和应用会拒绝 `__proto__`、`constructor`、`prototype` 这类危险路径。[packages/chord/src/delta/index.ts L118-L170](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/src/delta/index.ts#L118-L170) 检查操作形状与路径，整个结果还要通过另一层严格 JSON 验证。

本章的有序读副本协议以单一权威源为前提，以下能力需要另外的机制：

- 多写者数据库冲突处理没有实现，包括 Raft/Paxos 共识、leader election、quorum、CRDT 合并和网络分区下的多主写入。Session SQLite 的跨进程共享存储与所有权也需要另行协调。
- 一次 `state.change` 只形成单个 state 对象的一次原子 revision；跨多个 state members 的强一致事务没有实现。
- key/generation 编号保存在当前 provider 的 Map 中，只限制旧服务实例访问；它不提供跨主机 worker lease 或持久 leader term。
- 队列上限只作用于对应队列。实验 worker manager 在 [packages/coding-agent/src/experimental/session-worker-manager.ts L614-L639](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/experimental/session-worker-manager.ts#L614-L639) 用 `deliveryTail` 串行转发服务事件，但这条 Promise 链没有同样的 100-frame 上限，因此整个网络通道还没有端到端背压。
- 远端服务授权需要单独判断。service token allowlist 和 stale-generation 检查还没有组成完整租户授权；[packages/coding-agent/src/experimental/services/README.md L14-L24](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/experimental/services/README.md#L14-L24) 把认证身份与按客户端投影状态列为后续工作。

## 7.13 编号约定与源码定位

本章会遇到几种编号。它们记录不同对象，没有统一的全局含义：

| 标识 | 记录什么 | 范围 |
| --- | --- | --- |
| Session commit `Seq` | 原子数据库批次，编号可有间隙 | 不要求等于远端 state 的连续发布序号 |
| source cursor | attached source 的连续帧 | 没有跨 storage 全局时钟含义 |
| member sequence | 一个 state stream 的发布基线 | 没有多个 members 的事务边界含义 |
| key/generation | service instance 生命周期 | 不负责任务续租或共识 term |

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
| Transaction | Session mutation line 串行提交；Node SQLite 用 `BEGIN IMMEDIATE` / `COMMIT` 原子落盘 | 跨主机、跨外部服务的分布式事务尚未实现 |
| WAL | Node SQLite 用 write-ahead log 先记录日志以支持恢复，配合 `synchronous = NORMAL` | 掉电仍可能丢最新 commit；这份日志用于本地数据库 |
| Lease | durable task 没有可到期和续租的跨 worker 租约；实验 worker 有本地目录锁的 stale/update | 目录锁不负责 task reservation、多主共识或全局 fencing |
| Heartbeat | 实验目录锁周期更新；env 连接定期 ping，沉默超时会清理连接/进程 | 用于本地所有权保活与连接活性，无法证明任务仍有执行资格 |

Session 提交线在 [packages/durable/src/session/session.ts L220-L245](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/session/session.ts#L220-L245)，SQLite 事务与 WAL 设置在 [packages/durable/src/storage/sqlite/node.ts L140-L159](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/storage/sqlite/node.ts#L140-L159)、[packages/durable/src/storage/sqlite/node.ts L179-L202](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/durable/src/storage/sqlite/node.ts#L179-L202)。实验目录锁则在 [packages/coding-agent/src/experimental/session-worker.ts L517-L522](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/coding-agent/src/experimental/session-worker.ts#L517-L522) 设置 `stale: 2000`、`update: 1000`；env ping 的 Node/Rust 两端和故障处理见[第 8 章 沙箱与执行环境](08-sandbox-environment.md)。

## 7.15 练习

先画出任务 T 的记录变化：待处理，A 以代次 7 领取，续租失败，到期后 B 以代次 8 接管，A 的迟到结果被拒绝。分别标出每一步由谁检查、哪些字段必须在一次事务中更新。再设想文件服务不检查代次，指出数据库拒绝 A 的完成提交后，仍可能发生什么外部写入。

让 B 完成任务后立即打开进度页：主存储已提交，副本尚未应用该次提交。写出强一致读取与最终一致读取各自允许返回的结果，并说明为什么从副本看到“待处理”不能直接授权 Worker 执行。

可以在纸上维护 `{ output: "a" }` 的副本：hydrate 10，append 11，漏掉 12，再收到 13。按连续序号规则，value 会清空；之后显式 reset 把基线设为 20，21 才能继续应用。再试一次让普通 update 13 携带 `r`，检查为什么完整值仍无法免去缺号校验。

下面的测试可以用来核对这些协议行为。

| 阅读位置 | 可观察的协议行为 |
| --- | --- |
| [packages/chord/test/services.test.ts L575-L658](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/test/services.test.ts#L575-L658) | hydration 时的竞争；重复与缺号即便带 root replacement 也清空副本 |
| [packages/chord/test/services.test.ts L660-L771](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/test/services.test.ts#L660-L771) | rebind 后完整 hydration；复用 key 时旧 handle 失效 |
| [packages/chord/test/state-delivery.test.ts L59-L129](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/test/state-delivery.test.ts#L59-L129) | 慢 public callback 不阻塞精确发布；pending 的限制 |
| [packages/chord/test/service-delivery.test.ts L78-L135](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/test/service-delivery.test.ts#L78-L135) | provider 显式 overflow reset 与正在交付的帧 |
| [packages/chord/test/service-delivery.test.ts L211-L335](https://github.com/earendil-works/pi/blob/96abe9a5ad45005cae34ed1c52d42f71fc88bedc/packages/chord/test/service-delivery.test.ts#L211-L335) | codec reset 后继续接增量；重建 keyed membership |

若自行运行，从 `packages/chord` 执行 `node ../../node_modules/vitest/dist/cli.js --run test/service-delivery.test.ts`。这是本地协议测试，不需要登录、真实模型或远端部署。

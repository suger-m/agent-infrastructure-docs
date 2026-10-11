# 07. 客户端怎样跟上 Agent 的状态

这一章讲把同一份任务状态交给其他观察者。例如 Worker 在运行任务，终端或浏览器想显示当前输出。Worker 保存并改变权威状态，客户端维护供显示的副本。复制让观察者知道发生了什么，是否有权启动任务则是另外的所有权问题。

一个新客户端不能从“追加了几个字”开始理解整段输出，它首先需要当前完整值。之后每次变化只发送新增或修改部分，可以减少重复数据。完整值叫快照，变化叫增量；增量依赖前一状态，必须保证接在正确的起点上。

同步的完整流程是：订阅时捕获快照并登记后续更新，客户端先安装快照，再启用增量交付，按连续序号应用每次更新。若发现缺失，客户端丢弃已经无法信任的副本，并由主机重新建立基线。下面先看快照与更新怎样交接，再拆开状态修改、重连和慢消费者处理。

客户端晚于 Agent 启动时，之前的更新已经发生。若它先读当前值、之后才注册更新，中间变化可能丢失；若它先开更新、再安装旧值，新变化又可能被覆盖。订阅协议要让客户端从一份完整状态开始，再接住此后的变化。

权威写入方维护当前状态，只读副本供客户端观察。快照（snapshot）给副本一个完整起点，增量（delta）描述起点之后的改动，状态成员序号（member sequence）检查副本收到的更新是否连续。这个序号只属于一个 state member 的发布流，不是数据库全局编号，也不是全网时钟。

| 交接步骤 | 谁负责 | 交给下一步的内容 |
| --- | --- | --- |
| 权威状态变化 | 写入方在一个 state 对象上提交 change | 新状态 revision 与对应序号 |
| 新订阅建立 | 提供端（provider）捕获完整快照并登记订阅 | 当前值、当前序号，以及待交付的后续更新 |
| 安装并接收 | 消费端（consumer）安装快照，再 activate | 只读副本获得基线，开始按序应用增量 |
| 连续性检查 | 副本比较成员序号 | 正确序号时应用；重复或缺号时清空并报错 |

假设 worker 的对话状态已发布到序号 10，客户端此时订阅。提供端原子地捕获完整 snapshot 并登记订阅，之后产生的序号 11 暂存起来。消费端先安装序号 10 的完整状态，再 activate；提供端这时才把暂存的 11 送出。副本验证它正好是当前序号加一，然后应用更新。Hydration（用完整快照建立副本基线）先于增量，所以不会先看到新变化再被旧快照覆盖，也不会在读快照与注册之间漏掉变化。

若副本安装 10 后直接收到 12，它清空 value 并报告缺帧，不能把 12 当作新基线，也不会自行向提供端请求 snapshot。主机或 binding 负责重订阅；重新安装快照或接收显式 reset 后建立新基线。恢复以后再按严格递增的连续序号消费更新。

本文对应提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc`。各章入口在[学习索引](README.md)，持久化的提交顺序见[持久执行](05-durable-execution.md)，客户端和工具边界见[Tools 与 MCP](09-tools-mcp.md)。

两张图使用不同的示意序号：42、43、44说明基线后的连续更新及缺帧后的恢复；10、11展示实际订阅建立顺序。阅读时先看副本安装快照，再跟 activate 后送达的增量；箭头表示状态交付，不表示副本取得写权。

![副本先安装序号 42 的快照，再应用 43 和 44；缺号后需重建基线](figures/07-snapshot-delta.svg)

![Chord 订阅时先安装序号 10 快照，再激活并按序接收序号 11 更新](figures/07-subscription-sequence.svg)

## 从一个快照接住后续更新

原子步骤表示捕获快照与登记订阅之间没有可插入的状态发布。这样快照覆盖的变化已经在完整值里，之后的变化由订阅接住。activate 是启用交付的动作：客户端先装好基线，提供端再送暂存更新，才能避免先应用新变化、随后被旧快照覆盖。

序号的作用是发现“副本缺了一步”。例如文本基线为 `a`，下一步追加 `b`，再下一步追加 `c`；漏掉第一项却直接应用第二项，会得到 `ac`。序号跳变让程序在使用错误结果前发现问题。它检查交付连续性，不证明任务代码或业务内容正确，也不是多写入方之间的共识算法。

上面的晚订阅例子中，提供端要把“取得快照”和“登记订阅”作为一个原子步骤；消费端要先安装快照，再激活接收暂存更新。源码分别实现这两端的交接。

[packages/chord/src/services/provider.ts L239-L276](../../packages/chord/src/services/provider.ts#L239-L276) 把 subscriber 加入注册表，捕获 snapshot，记录这份快照已覆盖的 state sequences，再返回 activate/close。[packages/chord/src/services/provider.ts L424-L443](../../packages/chord/src/services/provider.ts#L424-L443) 中，每个 state member 的快照携带完整 `r` 与当前 sequence；普通发布通过已注册的精确操作监听器继续前进。

consumer 的 singleton 入口在 [packages/chord/src/services/consumer.ts L594-L633](../../packages/chord/src/services/consumer.ts#L594-L633)。它先校验、安装 snapshot，再 activate，于是快照与暂存更新能按前面的时序接上。新客户端从当前值开始观察，无需重放全部历史业务事件。

实际接收增量的是 [packages/chord/src/services/state.ts L376-L415](../../packages/chord/src/services/state.ts#L376-L415) 的 replica，也就是只读副本。它要先安装完整快照，随后每次更新的序号必须比上一次加一。重复、缺号、无效操作或者无效 JSON 结果，都会清空 value 并报错。

普通 `state` update 即便带完整 `r`，也须遵守连续序号。重新建立起点要经过显式 snapshot/reset hydration，否则中间丢失的实例生命周期变化就可能被忽略。发现缺号后，消费端只报告错误并清空数据，主机或 binding 还要负责重新绑定、订阅；[packages/chord/src/services/consumer.ts L540-L570](../../packages/chord/src/services/consumer.ts#L540-L570) 是这条重建路径。

Chord 的这些实现见 [packages/chord/README.md L1-L7](../../packages/chord/README.md#L1-L7) 与 [packages/chord/README.md L32-L57](../../packages/chord/README.md#L32-L57)。它是独立的应用组合包，不依赖其他 Pi workspace 包，负责让一个权威状态源向本地和远端消费者发布快照、增量，并维护序号、实例换代和重新绑定。实验 coding-agent services 用它把 worker 的会话视图交给客户端，durable 和这组 services 的入口仍属于实验层。

## 权威值怎样形成一次变化

要发出增量，先得明确一次状态修改的边界。比如任务状态从“读取中”转为“读取完成”，同时加入结果引用；这两个字段应属于同一次修改。draft 是临时草稿，回调在它上面改字段，成功后才整体采用并发布，失败时放弃。观察者拿到的是前一完整状态或新的完整状态，不必观察草稿修改到一半。

不可变值的约定用于维持这个边界：状态一旦发布，使用方就不能再偷偷改对象。否则同一个 revision 对不同观察者会出现不同内容，增量也失去可信基线。下面区分类型上的只读约束、调用者的所有权约定和实际复制行为，避免把它们误认为运行时已全面冻结数据。

先看本地状态修改。[packages/chord/src/services/state.ts L184-L244](../../packages/chord/src/services/state.ts#L184-L244) 的 `MutableReplicatedStateImpl.change()` 打开 draft，运行 mutate，准备出候选值，再采用并发布。回调抛错就放弃草稿，返回 Promise 也会被拒绝，所以网络等待应放在这个同步回调之外。

一次 state.change 可以一起更新多个字段，形成该 state 对象的一次原子 revision。若 prepare 后没有操作，方法会直接返回，发布序号也不增加。

这里的 immutable 需要使用方遵守所有权约定。传给 `track()` 或 `replicatedState()` 的 initial root 转交给 tracker，发布后的 value、ops 都不能再改。放进 draft 的外部值则会先验证再复制，详见 [packages/chord/src/delta/README.md L1-L41](../../packages/chord/src/delta/README.md#L1-L41)。

Delta 没有冻结所有对象，TypeScript 的只读类型也不提供 deep freeze 或 sandbox。进程内直接连接可能让消费端与提供端共享容器，消费者擅自修改值就会损坏权威状态。因此，交给会修改数据的代码前，要在边界复制或序列化。

增量（delta）是一批相对前一版本的操作。[packages/chord/src/delta/index.ts L21-L41](../../packages/chord/src/delta/index.ts#L21-L41) 定义整值替换、set/delete、字符串追加与截断，以及数组 splice/permutation；[packages/chord/src/delta/index.ts L408-L435](../../packages/chord/src/delta/index.ts#L408-L435) 通过 copy-on-write 应用这些操作，保留前一版不可变值。

例如输出从 `"a"` 变为 `"ab"`，线上可以只送 append，界面最后拿到的完整值仍是 `"ab"`。操作批次只要求在正确基线上按序应用后得到一致结果，形式未必唯一，也未必最小；一个操作的形状不能替代业务事件的含义。

## 断线和实例换代留下哪些东西

重连不仅要补数据，还要识别旧句柄是否仍指向同一实例。假设某个 key 对应的子服务关闭，后来用同一个 key 创建新服务；旧客户端如果只看 key，就可能把针对旧工作的调用发给新工作。generation 是当前提供端分配的实例代次，用来区分这两段生命周期。

编码字典则解决另一个问题：把重复路径缩写为编号，减少传输重复。发送方和接收方必须共享这份编号含义。新连接没有旧字典时，直接收到编号就无法解码，所以快照、重置和实例替换同时需要处理字典生命周期。这是传输编码状态，不能替代任务检查点。

一个 keyed service 用 key 区分实例，同一个 key 关闭后再出现，会得到新的 generation，表示这已经是另一次实例生命周期。[packages/chord/src/services/provider.ts L183-L212](../../packages/chord/src/services/provider.ts#L183-L212) 在复用 key 时增加 generation，[packages/chord/src/services/provider.ts L377-L409](../../packages/chord/src/services/provider.ts#L377-L409) 用它拒绝旧实例调用。

这份编号存在当前 provider 的 Map 中，用于限制旧服务实例访问。单例服务替换时会保留对外接口对象，换入新的状态快照；消费端重新绑定则清空旧值，直到重新安装快照。界面可以继续保存句柄，同时识别数据尚未就绪的阶段。

重新连接还涉及增量的编码字典。Delta encoder 在路径第二次出现时引入 ID，decoder 必须收到配套定义，具体过程在 [packages/chord/src/delta/index.ts L506-L553](../../packages/chord/src/delta/index.ts#L506-L553) 与 [packages/chord/src/delta/index.ts L622-L662](../../packages/chord/src/delta/index.ts#L622-L662)。一个连接承载多个独立状态时，各自都要有字典；不同客户端的进度也可能不同，不能用一个有状态编解码器编码一次，再复用于所有客户端。

[packages/chord/src/services/state-codec.ts L27-L58](../../packages/chord/src/services/state-codec.ts#L27-L58) 按 instance/member 创建 codec，[packages/chord/src/services/state-codec.ts L60-L137](../../packages/chord/src/services/state-codec.ts#L60-L137) 在 snapshot、reset、replacement、unavailable、closed 时重置或移除对应字典。解码和应用还会拒绝 `__proto__`、`constructor`、`prototype` 这类危险路径。[packages/chord/src/delta/index.ts L118-L170](../../packages/chord/src/delta/index.ts#L118-L170) 检查操作形状和路径，整个结果还要通过另一层严格 JSON 验证。

## 消费者跟不上时，队列如何收缩

这里要判断积压的是什么。如果每个回调都拿到完整值，中间几个版本可以跳过，最新版本仍可直接显示。若拿到的是“删除一个字段”“追加一段文本”这样的增量，跳过其中一步就会破坏后面的基线。两种队列因而采用不同的收缩方式：完整值保留最新，精确增量则改送一份显式 reset，重新提供完整起点。

界面的订阅回调可能等待很久。[packages/chord/src/services/state.ts L28-L95](../../packages/chord/src/services/state.ts#L28-L95) 与 [packages/chord/src/services/state.ts L149-L181](../../packages/chord/src/services/state.ts#L149-L181) 为每个 public subscriber 串行调用 callback，当前正在执行的回调不计入 pending。其他订阅者和同步的精确操作监听器仍能继续处理发布。

待交付的对象不同，溢出策略也不同：

| 位置 | 交付内容 | 溢出后的处理 |
| --- | --- | --- |
| public state subscription | 捕获的完整 value/context/delivery | 100 个 pending 后清空旧 pending，保留最新；未开始的 hydration 可保留 |
| provider remote subscription | 精确 state 操作与实例 lifecycle updates | 新增第 101 个 pending 时换成一个显式 reset，包含当前完整订阅 snapshot |
| 应用 transport 的异步队列 | 宿主自行定义 | 不自动继承上述上限 |

公开订阅的回调收到完整值，允许跳过发布序号。远端增量要连续应用，[packages/chord/src/services/provider.ts L445-L465](../../packages/chord/src/services/provider.ts#L445-L465) 才用 reset 替换积压，给每个状态建立新基线。reset 还须重建 keyed membership，避免丢掉 close/spawn 后留着已经不存在的实例。上限按帧数计算，较大的 snapshot 仍可能占用较多字节。

## 复制状态背后的存储与进程

最后看 Chord 的权威值如何接入 durable，以及各类编号的作用范围。

实验服务从 durable 接入权威数据的地方很短。`Session` 在 Storage 成功后采用状态并发布，conversation view 随 commit 更新。[packages/coding-agent/src/experimental/services/transcript-provider.ts L5-L15](../../packages/coding-agent/src/experimental/services/transcript-provider.ts#L5-L15) 的 `createTranscriptServiceFacet()` 直接取得 `conversation.viewState()`，作为 Transcript.state 提供，退出时 dispose。

durable Storage 负责保存与恢复，Chord 把当前 view 同步给观察者。普通 `replicatedState(initial)` 自己不写磁盘，可以连接任意权威数据源。Delta 的有序交付、丢包与乱序处理、重新连接和订阅，仍由协议和宿主共同负责。

阅读这些调用时会同时遇到几种编号，它们各有作用域；标识对照与本章术语表一并收在文末「源码位置与术语速查」。

## 同步给观察者以后，谁有权继续执行

前面的副本协议回答“客户端是否知道最新状态”。另一个问题是“哪个进程有权改变持久任务”。假设两个 Worker 都读到同一个待处理任务，若它们同时执行修改，就会发生重复工作；仅给两者同步更多状态，不能阻止这种竞争。需要执行所有权规则，再由存储或宿主检查写入资格。

Lease（租约）通常给执行者一段有效期，持续续租才保有资格；Heartbeat（心跳）是周期性的活性通知。观察到心跳中断只能说明近期没有收到通知，进程可能退出，也可能暂时失联。若另一个执行者接管，旧执行者之后恢复，还要防止它继续提交。Fencing token 是与所有权代次关联的凭证，受保护的一方检查代次并拒绝失效执行者。

这段例子用于说明更大规模系统为什么需要这些机制。Pi 当前 durable 库采用单进程拥有 storage 的前提，实验 Worker 用本地目录锁维护 Session 所有权；它没有实现上述跨主机任务租约与接管协议。Chord 的发布序号检测副本缺帧，generation 检测服务实例换代，二者也不能直接替代执行资格的检查。

Consistency（一致性）需要指明对象和范围。本章有单个状态修改的原子发布、存储批次的原子提交和只读副本的连续更新；让多台机器共同决定写入顺序还需要另一组协调协议。不要把“客户端收到相同值”扩大成“多个 Worker 的执行与所有外部效果都已一致”。

## 边界与未实现

前面的协议以单一权威源为前提；超出这个范围的协调能力都未实现，需要另外的机制：

- 这条有序读副本协议没有多写者数据库的冲突处理：Raft/Paxos 共识、leader election、quorum、CRDT 合并和网络分区下的多主写入都未实现。Session SQLite 的跨进程共享存储与所有权，也需要另外的协调机制。
- 一次 state.change 只形成单个 state 对象的一次原子 revision；跨多个 state members 的强一致事务没有实现。
- key/generation 编号存在当前 provider 的 Map 中，只用于限制旧服务实例访问；跨主机 worker lease 与持久 leader term 没有由它实现。
- 队列上限只作用于相应队列。实验 worker manager 在 [packages/coding-agent/src/experimental/session-worker-manager.ts L614-L639](../../packages/coding-agent/src/experimental/session-worker-manager.ts#L614-L639) 用 `deliveryTail` 串行转发服务事件，但这条 Promise 链没有同样的 100-frame 上限，整个网络通道尚未因此具备端到端背压。
- 远端服务的授权范围也要另看。service token allowlist 和 stale-generation 检查尚未组成完整租户授权，[packages/coding-agent/src/experimental/services/README.md L14-L24](../../packages/coding-agent/src/experimental/services/README.md#L14-L24) 仍把认证身份与按客户端投影状态列为后续工作。

## 源码位置与术语速查

阅读本章时会同时遇到几种编号，它们各有作用域：

| 标识 | 记录什么 | 范围 |
| --- | --- | --- |
| Session commit `Seq` | 原子数据库批次，编号可有间隙 | 不要求等于远端 state 的连续发布序号 |
| source cursor | attached source 的连续帧 | 没有跨 storage 全局时钟含义 |
| member sequence | 一个 state stream 的发布基线 | 没有多个 members 的事务边界含义 |
| key/generation | service instance 生命周期 | 不负责任务续租或共识 term |

### 文中术语

| 术语 | 当前实现 | 范围与限制 |
| --- | --- | --- |
| Transaction | Session mutation line 串行提交；Node SQLite 用 `BEGIN IMMEDIATE` / `COMMIT` 原子落盘 | 跨主机、跨外部服务的分布式事务尚未实现 |
| WAL | Node SQLite 用 write-ahead log 先记录日志以支持恢复，配合 `synchronous = NORMAL` | 掉电仍可能丢最新 commit；这份日志用于本地数据库 |
| Lease | durable task 没有可到期和续租的跨 worker 租约；实验 worker 有本地目录锁的 stale/update | 目录锁不负责 task reservation、多主共识或全局 fencing |
| Heartbeat | 实验目录锁周期更新；env 连接定期 ping，沉默超时会清理连接/进程 | 用于本地所有权保活与连接活性，无法证明任务仍有执行资格 |

Session 提交线在 [packages/durable/src/session/session.ts L220-L245](../../packages/durable/src/session/session.ts#L220-L245)，SQLite 事务与 WAL 设置在 [packages/durable/src/storage/sqlite/node.ts L140-L159](../../packages/durable/src/storage/sqlite/node.ts#L140-L159)、[packages/durable/src/storage/sqlite/node.ts L179-L202](../../packages/durable/src/storage/sqlite/node.ts#L179-L202)。实验目录锁则在 [packages/coding-agent/src/experimental/session-worker.ts L517-L522](../../packages/coding-agent/src/experimental/session-worker.ts#L517-L522) 设置 `stale: 2000`、`update: 1000`，env ping 的 Node/Rust 两端和故障处理见[Sandbox 与执行环境](08-sandbox-environment.md)。

## 测试与练习

可以在纸上维护 `{ output: "a" }` 的副本：hydrate 10，append 11，漏掉 12，再收到 13。按前面的连续序号规则，value 会清空；之后显式 reset 把基线设为 20，21 才能继续应用。再试一次让普通 update 13 携带 `r`，检查为什么完整值仍无法免去缺号校验。

下面的测试可以用来核对这些协议行为。

| 阅读位置 | 可观察的协议行为 |
| --- | --- |
| [packages/chord/test/services.test.ts L575-L658](../../packages/chord/test/services.test.ts#L575-L658) | hydration 时的竞争；重复与缺号即便带 root replacement 也清空副本 |
| [packages/chord/test/services.test.ts L660-L771](../../packages/chord/test/services.test.ts#L660-L771) | rebind 后完整 hydration；复用 key 时旧 handle 失效 |
| [packages/chord/test/state-delivery.test.ts L59-L129](../../packages/chord/test/state-delivery.test.ts#L59-L129) | 慢 public callback 不阻塞精确发布；pending 的限制 |
| [packages/chord/test/service-delivery.test.ts L78-L135](../../packages/chord/test/service-delivery.test.ts#L78-L135) | provider 显式 overflow reset 与正在交付的帧 |
| [packages/chord/test/service-delivery.test.ts L211-L335](../../packages/chord/test/service-delivery.test.ts#L211-L335) | codec reset 后继续接增量；重建 keyed membership |

若自行运行，从 `packages/chord` 执行 `node ../../node_modules/vitest/dist/cli.js --run test/service-delivery.test.ts`。这是本地协议测试，不需要登录、真实模型或远端部署。

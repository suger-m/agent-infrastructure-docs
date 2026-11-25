# Maka TypeScript 架构核心词表

本词表按概念边界收拢 Maka TypeScript + 原生 Rust 混合引擎中的核心实体、状态契约与持久化原语，帮助阅读各章前快速对齐专业词汇。

---

## 1. 宿主权威、范围与六层身份体系

| 术语 | 精确工程定义 | 不可互换边界 |
| :--- | :--- | :--- |
| **`State Root` / `rootId`** | 宿主机持久状态目录的唯一物理根及其持久 ID；所有工作状态与账本均归属此根 | 同台物理机同一时刻最多由单个持有 OS 排他锁的 Host 写入；绝不等于文件系统字符串路径 |
| **`HostEpoch`** | 当前在线 Runtime Host 进程的单调运行纪元 | 只要 Host 进程重启即刷新，配合 `sequence` 标定客户端订阅事件流的时序连续性 |
| **`Composition ID / Revision`** | 当前宿主启用的能力装配体标识与修订版本号 | 用于检验多客户端及持久化库的模型解释权兼容性，不可与普通二进制版本号混用 |
| **`sessionId`** | 面向用户或协同团队的稳定对话上下文容器 | 跨 Host 重启与后台任务延续稳定，是订阅与访问控制的顶层边界 |
| **`turnId` / `runId` / `invocationId`** | 逻辑用户回合（Client 幂等键）、逻辑执行目标（Span）与物理启动尝试（Attempt-chain） | 一次超时重连或安全续接可能产生多个连续的 `invocationId`，但逻辑归属仍指向同一 `runId` |
| **`stepId` / `attemptId`** | 模型单步思考调用轮次与内部物理流式传输重试尝试号 | 只有最终胜出或提交的 `attemptId` 才会投影进入 `CanonicalSessionProjection` |
| **`BackendGeneration`** | `RuntimeKernel` 为每次热启动或模型配置冷切分配的单调代次计数 | 旧代次实例持有的句柄在 `await` 返回后若代次失效，禁止再向共享状态写入一行数据 |

---

## 2. 事件脊柱、操作账本与上下文投影

| 术语 | 精确工程定义 | 核心不变量 |
| :--- | :--- | :--- |
| **`RuntimeEvent`** | 存储在 `SqliteRuntimeStore` 中的唯一内部权威执行事实信封（`packages/core/src/runtime-event.ts`） | 绝非前端 UI 的 `SessionEvent`，亦非纯审计指标行；必须通过 `encodeCanonicalRuntimeEvent` 往返无损序列化验证 |
| **`AgentRunStore` (Operational Ledger)** | 与事实脊柱伴生的运维操作账本（`packages/core/src/agent-run.ts`） | 专门承接请求计量、试错追踪、审批决定与工具周期详情；本身不定义执行路由与血缘终点 |
| **`ModelProjectionTransitionLedger`** | 记录历史裁剪、压缩与替代偏移量的归约账本 | 确保暖机内存投影与离线冷重启重建得出逐字节一致的模型输入前缀 |
| **`CanonicalSessionProjection`** | 由规范事件流确定的只读状态投影，作为所有订阅客户端的统一基准 | 客户端永远不得用局部 Delta 广播凭空猜测运行已完结，只能以投影终态为权威 |
| **`Tool Result Archive` (`maka://`)** | 超大工具原始输出的内容寻址持久化存储池，支持 `read-page` 分页懒读 | 原始载荷脱离模型主上下文保存，保护注意力的同时保证 100% 原始信息可溯 |
| **`Safe Compaction Prefix` (`#6254`)** | 经过工具成对校验、双防伪哈希核验与封存未知调用消解的历史裁剪前缀 | 若 Host 崩溃已通过 `outcome_unknown` 终态封存孤立调用，压缩边界可安全越过该封存段前进 |

---

## 3. 工具结算、两阶段持久化与系统安全面

| 术语 | 精确工程定义 | 安全与幂等底线 |
| :--- | :--- | :--- |
| **`T1 / T2` 两阶段提交** | 工具经过三重参数快照与预检后落盘 `t1_after_preflight_v1`，外部副作用结算完原子写入 `T2` | 一旦越过 `T1` 断电崩盘且未达 `T2`，恢复期严格标记为 `outcome_unknown`，禁止自动盲目重跑 |
| **`accepted` vs `admitted`** | Client Capability 反向代理中的通信回执与实际副作用准入切点 | `accepted` 仅确认对端收到请求；一旦跨越宿主下发的 `admitted` 边界后失联，即锁死重试 |
| **`Code Mode` (`exec`)** | 允许模型在隔离 JS 沙箱内用连续代码编排 `tools.*` 的运行模式 | 内部嵌套调用全数继承主链 `parentCallId`，同步严格走过 `ToolRuntime` 预检、审批与 T1/T2 落盘 |
| **`deriveWorkerProfile` / TOCTOU** | 内置文件工具从全局策略出发经历“会话级 + 单次调用级”二次收窄，并锁定 `(dev, ino)` 写入前核验 | 堵死只给 `Bash` 上锁却放任 `Read`/`Write` 主进程直触宿主引发的路径穿越和软链接抢占 |
| **`Managed Secret` 加密信封** | 将第三方 API 或 MCP 密钥隔离出主配置文件，经加密存储在 `SecretStore` | 仅在目标受控工具发起物理执行的瞬间，按作用域包络动态解码注入隔离环境 |
| **`gitoxide-helper` (`native/`)** | 基于 Rust `gix` 实现的短生命周期受控 Git 数据面助手 | 仅接收不透明凭据 `Opaque Capability`，用 `refs/maka/accepted` 与 `MustNotExist` CAS 推进仓库检查点 |

---

## 4. 协同、调度队列与可靠网关

| 术语 | 精确工程定义 | 调度边界 |
| :--- | :--- | :--- |
| **`SessionAdmissionGate`** | 保护单/多会话状态修改、队列落盘与准入校验的极短临界区锁 | **绝不包裹整个大模型推理耗时周期**；重执行动作一经 `RootTurnAdmission` 落盘即转入异步 `drainTurn` |
| **`SteeringLease` (`leaseId`)** | 纠偏消息从内存队列投递给进行中的 `AiSdkTurn` 时的排他契约凭据 | 严格二元结算（Ack 确认消费 / Nack 退回队列），防止纠偏消息在步间中断瞬间丢单 |
| **`Agent Graph` & `Swarm`** | 以统一事件图编排多智能体协作的调度平面（`update_agent_graph` / `yield_agent_graph`） | 绝不另建平行 Runtime；各写角色强制分配独立 Git Worktree 并在 Reconciler 汇合补丁 |
| **`EventWait` & `GoalCloseFence`** | 让长周期 Goal 在等待外部状态或子任务时持久退避挂起（无需空转轮询），并守护完结围栏 | 存在未结清并发事件或未过账预算门控前，围栏阻止 Goal 过早宣告关闭 |
| **`Peer Mesh` (`runtime-host-peer`)** | 依托 libp2p / WebRTC Direct 实现的原生 Rust 跨设备连接转发底座 | 通过 `ResumablePeerStream` 的传输字节偏移量与确认游标，支持移动端切换网络不断流无损接续 |
| **`earliest valid` 裁决铁律** | `packages/eval` 评测系统的防作弊结算底线 | 无论后期重试多少次，所有指标一律锁死提取最早一份完整有效试跑记录（搭配不可变存储） |

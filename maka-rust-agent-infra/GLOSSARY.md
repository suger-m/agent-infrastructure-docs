# Maka 核心领域词汇表（Rust 微内核版）

本词汇表定义 **Maka Rust 微内核版**（`Platform` + `Plugins` + `Products`）中的核心概念、持久化不变量与运行时实体。

---

## 1. 微内核底座与存储权威 (`maka-event-log` / `maka-host-runtime`)

| 术语 | 所属 Crate | 核心定义与边界 |
| :--- | :--- | :--- |
| **RootOwner & RootStore** | `maka-event-log` | 一把操作系统级独占进程锁（`RootOwner`）配一个 SQLite 数据库（`RootStore`）。写操作走单线程 FIFO 队列串行提交，读操作走并发快照通道，事务提交后广播缓存失效。 |
| **CallerIntent & RecoveryRef** | `maka-event-log` | **调用方意图**。带副作用的 Host 调用在进入目标服务前必须先落盘的调用身份（目标实例、方法、操作 UUID、请求指纹与契约摘要），用于断线或重启后精准对账。 |
| **Outcome vs Cleanup** | `maka-plugin-core` | 服务操作回执（`application_operations`）中分离且各自单调推进的两条状态线：**业务结果 `Outcome`**（`NotExecuted` / `Returned` / `Unknown`）与**物理资源清理 `Cleanup`**（`Unconfirmed` / `Confirmed` / `Failed`）。 |
| **Pinning（证据钉住）** | `maka-event-log` | 未完结的 Run、`Unknown` 操作、未确认清理的资源、活跃 Code Mode 单元以及历史快照会自动钉住关联的内容寻址对象，直到业务结果与物理清理双双结清才允许垃圾回收。 |
| **ExecutionAuthority** | `maka-event-log` | 独立于不可变对象库的可信控制面执行权接口。绑定逻辑 Session、激活 ID、执行代次、精确起始 Head 以及工具和沙箱权限上限，Host 每 250ms 观察一次租约。 |
| **NSH Workbench** | `products/maka/workbench` | 基于原生渲染底座驱动桌面端原生窗口与终端 TUI 运行同一套 TSX 代码的工作台虚拟机，支持通过 `prepareReload()` 在热重载时平滑移交 JSON 状态。 |

---

## 2. Agent 引擎、状态折叠与 Reconciler (`plugins/agent/`)

| 术语 | 所属层级 | 核心定义与边界 |
| :--- | :--- | :--- |
| **稳定 `RunId`** | `maka-agent-facts` | 逻辑运行的唯一持久标识。在 Rust 版中，**一次逻辑运行在跨越工具审批、重试退避、上下文压缩与 Host 冷启动重启时始终保持同一个 `RunId`**（区别于 TS 版的 Invocation 接续）。 |
| **StepId & ModelAttemptId** | `maka-agent-facts` | `RunId` 内部的细粒度划分：`StepId` 对应一次推理步进，每次真实向上游供应商发出的物理请求则分配独立的 `ModelAttemptId`。 |
| **RunView（义务前沿）** | `maka-agent-state` | 在同一写事务内把不可变事实确定性折叠出的内存/状态投影视图（包含当前待推进的流式工具调用、用户交互、后台 Code Mode 单元与 `BlockerKind`）。 |
| **Reconciler (`next_action`)** | `maka-agent-engine` | 唯一的确定性下一步策略函数。根据 `RunView` 与实时内存状态 `Live` 裁决下一组待执行的 `WorkKey`，在线推进与崩溃恢复 100% 共用同一循环。 |
| **DriverClaim** | `maka-agent-state` | 单个 Run 的排他驱动声明令牌（Fencing Token），防止在配置切换或接管瞬间出现两个驱动同时往同一 Run 写事实。 |
| **PendingPublication** | `maka-agent-engine` | 当数据库提交结果不确定时保留原始稳定 `FactId` 的协调对象。重调谐它绝不是重新准入，且只会在合法时通过 `store.observe()` 补记晚到结算，绝不重复触发外部副作用。 |
| **RetryStreak** | `maka-agent-state` | 追踪连续重试预算的状态机（`Regular` → `FallbackTransitionPending` → `AfterFallbackTransition`），支持在主模型连续失败时触发**单次主从模型自动切换**。 |

---

## 3. 上下文竞技场、Code Mode 与原生隔离

- **CaptureArena**：将规范化模型消息及多模态载荷捕获进受请求所有权保护的字节竞技场（固定 `160` 字节记录指针 `[u64; 20]` 指向底层 `SourceBytes::Source`），彻底消除大模型重试与重放时的堆内存复制爆炸。
- **Proactive Summary（85% 前瞻摘要）**：当估算的输入 Token 达到模型声明窗口的 **85% 阈值**时，Reconciler 自动派发 `WorkKey::PrepareProactiveSummary` 在后台并发准备 `CompactedContext` 基线，做到无感压缩。
- **Cell & SessionDraft (`Code Mode`)**：嵌入式 V8（`maka-js-runtime`）或受限 Monty Python（`maka-python-runtime`）执行的代码单元。`CellLimits`（默认 30s CPU / 64MB 堆）剔除等待异步 Host 工具的时间；**不做堆快照且恢复时绝不重放脚本**，而是将键值对写入 `SessionDraft`，仅当 Cell 成功结清时在同一个事务内原子发布 `SessionData`。
- **Computer Stage**：为已授权的原生应用或浏览器标签页提供有界（最长边 1600px、最大 2 MiB JPEG）、只读、临时的实时预览卡片，受独立的 `PreviewAuthority` 与 `ForegroundAuthority` 保护，绝不向模型泄露或转发 UI 坐标。
- **Windows Microsoft MXC**：在 Windows 上为每棵命令树创建零 UAC、零本地账户、不改写既有 ACL 的原生安全沙箱，并用可恢复占位守卫实现可写目录内的只读例外子路径。

---

## 4. 协同工具与迁移验证

- **对称 `Session*` 工具与共享 `Room`**：模型拥有 `Read("maka://session/<id>")` 与 7 个与人类 UI 操作对称的 `Session*` 工具（`List / Read / Create / Send / Respond / Stop / Manage`）。共享 `Room`（`kind: "room"`）支持原子提交 `@mentions` 与 `replyTo`，且派发子任务默认开启 `onComplete: "notify_caller"` 携带 `<worker_result>` 唤醒调用方。
- **SubagentPreset**：通过 `expectedRevision` 乐观并发锁保护的可复用子智能体预设模板，支持在 `SessionCreate` 时一键拉起并结合托管 Git `worktree` 隔离改动。
- **主动发现（Proactive Discovery）**：由 Bot 提议、用户批准工具上限的定时只读探索任务，占用单一执行位并通过 `DiscoveryComplete` 静默结算或交由 Bot 决策汇报。
- **不可变迁移与时间胶囊测试 (`upgrade_compat.rs`)**：所有已发布的 SQLite 迁移文件均受 `migrations.lock.json` 的 SHA-384 摘要锁保护；通过回放历史真实数据库时间胶囊验证跨代升级的冷迁移、穷举读取与断点续跑能力。

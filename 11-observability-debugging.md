# 11. 怎样判断一次任务慢在哪里

[返回学习地图](README.md) · 前置：[Agent 循环](01-agent-runtime.md) · 关联：[评估与回放](12-evaluation-replay.md)

用户要求“读两个文件后修改配置”。模型很快生成两个工具调用，第二个工具却耗时较长。如果只记下最终回答的时间，就无法判断等待发生在模型、工具，还是事件监听器写日志的过程中。

可观测性用于留下能解释执行情况的记录。调试时，先定位等待在哪个阶段，再到那个阶段查看输入、输出或错误。记录应能回答具体问题：是否已经发起模型请求，工具是否开始，哪个调用尚未结束，任务是在等别人还是缺少执行定义。

完整排查过程可以从这个慢任务开始：

1. 记下本次输入对应的运行，以及模型消息与工具调用 ID。
2. 沿事件顺序找最后完成的阶段，例如模型已经给出调用，但某个工具没有结束事件。
3. 对已结束的工具读取耗时，结合调用者额外记录的时间检查其他等待；不要把工具计时范围之外的延迟算进去。
4. 若工作属于 durable，再读取任务阶段、等待关系或阻塞原因，判断它是否能被调度。
5. 需要长期比较时，把明确口径的数据汇总为指标；需要查看父子操作关系时，再接入追踪。

Metrics（指标）是按约定口径统计的数值，例如一次运行的 Token 数，或者一组运行的失败率。Trace（追踪）把相关操作连起来；Span 是其中一段操作的记录。它们需要应用定义怎样关联，不能仅凭一批事件自动推导出所有等待和耗时。下面先使用仓库已有数据，再看哪些追踪能力需要接入。

先区分三种记录。事件说明某次运行中发生了什么；统计把多次调用的用量或时长汇总；Trace（追踪）把一次操作和它的子操作连成关系。它们回答不同问题，也都不能替代恢复任务所需的持久状态。

| 记录 | 负责回答的问题 | 例子与限制 |
| --- | --- | --- |
| Agent 事件 | 某条消息或工具在运行中的哪个阶段 | `message_update`、`tool_execution_start/end`，按事件顺序看一次运行 |
| Metrics / 统计 | 消耗了多少 Token、成本或时间 | Session usage 会累计历史记录；当前没有统一聚合后端 |
| Trace / Span | 一个操作与哪些子操作有关 | `pi-telemetry` 可记录父子关系；内存适配器没有时间戳 |
| 业务状态 | 任务重开后怎样继续 | 由 Session 或 durable 保存，不能从遥测推断 |

## 先用慢工具的例子定位等待

对“读两个文件后修改配置”，先记录 `message_update` 与工具开始、结束事件。用 `toolCallId` 把某个工具的 `tool_execution_start` 和 `tool_execution_end` 对起来，再读取结束事件上的 `durationMs`。最后查看 assistant 的 usage，了解模型累计消耗。事件说明阶段，工具耗时说明实际执行时间；若要进一步分开模型等待、钩子和监听器等待，调用者还需补充对应计时。本文对应源码提交 `96abe9a5ad45005cae34ed1c52d42f71fc88bedc`。

下文先看已有事件与统计分别覆盖哪里，再说明 Span 如何补充调用关系。

### 先看事件发生的顺序

事件是程序在某个阶段发出的通知，例如“工具开始”“工具结束”。监听器是接收这些通知的函数。若程序等待监听器写完日志再继续，记录行为本身就占用执行时间；若程序只调用监听器而不等待异步返回，日志可能晚于任务结束才完成。下面分别核对 Agent 与 AgentSession 的契约，是为了判断记录是否影响所观察的运行。

读 [agent/src/types.ts:509-539](../../packages/agent/src/types.ts#L509-L539) 的 `AgentEvent` 时，可以把一次运行拆成几轮：每轮包含一份 assistant 回复及其工具调用。流式 `message_update` 描述增量变化，要保存完整消息则等最终的 `message_end`。

两个工具并行执行时，`tool_execution_end` 按实际完成顺序出现，写回对话的工具结果却按模型原始调用顺序排列。这两种顺序可对照 [agent-loop.ts:586-658](../../packages/agent/src/agent-loop.ts#L586-L658)，调试记录应按 ID 关联，数组位置无法可靠对应这两组数据。

`Agent.subscribe()` 允许监听器返回 Promise，事件派发会等它完成。因此，`agent_end` 监听器还在工作时，Agent 仍未空闲。入口和等待语义见 [agent.ts:260-269](../../packages/agent/src/agent.ts#L260-L269)。例如监听器要等日志落盘，这段等待也会影响运行的结束时间。

默认 CLI/SDK 暴露的 `AgentSession.subscribe()` 则约定监听器返回 void，`_emit()` 同步调用它，不等待异步返回值。实现见 [agent-session.ts:1038-1043](../../packages/coding-agent/src/core/agent-session.ts#L1038-L1043)、[agent-session.ts:1364-1379](../../packages/coding-agent/src/core/agent-session.ts#L1364-L1379)。所以 Session 的日志监听器即使返回 Promise，也不会让运行等到日志完成。

### 一段耗时与一份累计消耗

一次工具耗时、一轮模型用量和整个会话总用量有不同范围。例如首次输入读文件，第二次输入继续修改，会话累计值包含两次交互；拿它当作第二次输入成本就会多算。分支中的旧尝试和摘要请求也可能留下 usage（用量记录）。统计前必须先明确“我在数哪些记录”，再决定分母是什么。

Token 用量通常分输入、输出和缓存相关项；缓存项表示服务报告的缓存使用情况。成本根据记录的用量与模型价格元数据估算，真实收费仍要核对服务账单。失败率同样需要定义一次尝试和一次任务：一个任务重试两次后成功，可以是一次任务成功、两次尝试失败；混用计数对象就会产生不同结论。

`executePreparedToolCall()` 在执行工具前读取单调时钟 `performance.now()`，执行成功或抛错后计算耗时毫秒数。计时位置见 [agent-loop.ts:821-856](../../packages/agent/src/agent-loop.ts#L821-L856)。它只覆盖 `execute()`，参数校验和前后 hook 都在计时范围之外；工具被阻止、未实际执行时也没有这项耗时。

`AgentSession.getSessionStats()` 则扫描会话的全部条目，累计 assistant、工具、压缩摘要和分支摘要等记录的 usage，范围见 [agent-session.ts:4177-4235](../../packages/coding-agent/src/core/agent-session.ts#L4177-L4235)。读这个数时，要按整个会话的累计消耗理解：它既包含当前上下文之外的历史，也无法直接当作单次输入的消耗。

| 数值 | 已有来源 | 解读边界 |
| --- | --- | --- |
| 工具耗时 | `tool_execution_end.durationMs` | 不含前后 hook |
| 输入/输出/缓存 Token | Session usage 汇总 | 包含历史分支及部分后台操作，按实际记录累计 |
| 估计成本 | usage 中的价格计算、eval report | 模型价格元数据的估计值；不是账单对账 |
| 整次 eval 延迟 | harness 的 `performance.now()` 差值 | 包含 harness 内初始化与清理，不含外层构建镜像 |
| 队列深度、重试次数、失败率 | 需要上层自行定义和采集 | 当前没有统一指标命名与聚合服务 |

评估运行会复制 Session 统计，记录 `systemPromptSha256`，在有价格数据时还会记录成本。报告赋值见 [evals/src/harness.ts:391-417](../../packages/evals/src/harness.ts#L391-L417)。这个摘要能用来确认提示词是否相同，无法据此还原提示词内容。

![Agent 生命周期事件、Usage 统计与应用补充遥测的观测路径](figures/11-observation-flow.svg)

图中的实线是 Agent 和 Session 已提供的记录，虚线表示应用需要接入的遥测连接。

## 用 Span 记录操作关系

事件顺序不一定足以恢复操作关系。两项工具并行时，事件会交错出现；仅凭相邻位置，无法判断某个子调用属于哪个父操作。Span 保存自己的 ID 和父 ID，让记录者明确关联“本次任务 → 某次模型请求 → 某项工具”。这些箭头是应用需要建立的关系，当前代码没有默认把这整条链都接上。

创建 Span 时传入业务回调，适配器在回调结束后结束记录。这样记录生命周期跟随实际操作，异常也可保留。下面的内存适配器用于检查父子关系与结束顺序；它没有计时字段，因此不能单靠这些记录判断哪个操作用了几秒。

Span 接口提供 `startSpan()`、`addEvent()`、`setAttributes()` 和 `setStatus()`，定义见 [telemetry/src/index.ts:1-22](../../packages/telemetry/src/index.ts#L1-L22)。结束由适配器负责：回调返回或其 Promise 完成后，Span 随之结束，因此没有需要调用者手动配对的公开 `end()`。Trace、Span、Event 等术语在本项目中的对应，汇总在文末「源码位置与术语速查」。

内存适配器的关键路径是 [memory.ts:120-186](../../packages/telemetry/src/memory.ts#L120-L186)。适配器先建立记录并保存 `parentId`，再同步调用业务回调一次；回调收到的 Span 同时也是创建子 Span 所用的上下文。回调成功便正常结束，抛错或 reject 时则保留同一个错误值，默认记为 error。

属性、事件与状态的记录若失败，适配器会忽略记录错误，业务回调仍然执行；业务自己抛出的错误则照常传播。显式 `setStatus()` 会优先于自动结果，即使回调随后抛错，已显式设置的状态也不会被覆盖。

已结束 Span 上的记录操作不再生效。从它创建迟到的子 Span 时，适配器会给出一个不记录数据的上下文，但仍执行业务回调。这样，记录生命周期不会阻断实际操作。

`getSpans()` 返回副本，见 [memory.ts:189-219](../../packages/telemetry/src/memory.ts#L189-L219)。记录保存序号、父 ID、状态、事件和结束顺序，没有时间戳或持续时间。因此内存适配器可以验证调用关系，却没有绘制延迟火焰图所需的计时数据。

### 类型能约束哪些记录

如果各处随意写名称和属性，同一操作可能一处叫 tool、一处叫 tool_call，汇总时就无法直接对应。Schema 是事先声明允许使用的名称与字段；类型辅助器让 TypeScript 在编写代码时检查拼写和字段形状。适配器则把统一的记录接口转成某种实际保存方式，例如本节的内存记录。编写阶段的约束与运行时记录是两层，所以还要分别核对能检查什么、是否真正脱敏。

`defineTelemetrySchema()` 用来辅助类型推导，运行时原样返回定义，见 [index.ts:71-74](../../packages/telemetry/src/index.ts#L71-L74)。`createTypedSpanStarter()` 根据定义限制 Span 名称、属性和事件，并把子创建器绑定到父 Span，实现见 [index.ts:324-354](../../packages/telemetry/src/index.ts#L324-L354)。

这些检查只在 TypeScript 阶段生效。运行时不会校验定义，也不会强制执行 `parents` 或 `sensitive` 元数据的含义；脱敏与采样仍由调用者或适配器实现。

### 把上下文传到模型请求

`ProviderRequestOptions.telemetryContext` 的定义见 [ai/src/types.ts:157-160](../../packages/ai/src/types.ts#L157-L160)，`buildBaseOptions()` 保留它，见 [simple-options.ts:36-65](../../packages/ai/src/api/simple-options.ts#L36-L65)。自定义模型服务适配器可以沿这个通道接入遥测，但具体适配器是否创建 Span，仍取决于它自己的实现。

CLI 的安装遥测开关另见 [coding-agent/core/telemetry.ts:1-13](../../packages/coding-agent/src/core/telemetry.ts#L1-L13)，它也控制部分模型服务的归属标记请求头，相关逻辑见 [provider-attribution.ts:38-71](../../packages/coding-agent/src/core/provider-attribution.ts#L38-L71)。这个开关控制的能力没有包含任务追踪。

## 没有结果的任务，是在等待还是被阻塞

没有输出不一定是故障。任务可能正常等待子任务，也可能已准备好但调度未启用，或者找不到执行代码。先读取诊断状态，可以避免把所有情况都处理成“再运行一次”。inspect 读取当前事实，resume 启用执行；这两个动作分开，让排查不必先改变现场。

当持久任务迟迟没有结果时，durable 的只读诊断接口可以先说明它在等待还是被阻塞，再由调用方决定是否恢复。

实验 durable harness 可以直接检查持久任务状态。`inspect()` 通过 Session 的串行读取通道返回 `scheduling`、仍存活的任务和未结算的提交，入口见 [harness.ts:258-266](../../packages/durable/src/harness/harness.ts#L258-L266)；每个任务同时附有原记录和推导出的运行状态，读取逻辑见 [scheduler.ts:1255-1281](../../packages/durable/src/harness/scheduler.ts#L1255-L1281)。

一个迟迟没有结果的任务，可能在等另一个任务，也可能缺少注册定义，或版本迁移失败。`inspect()` 会区分 `ready/running/waiting/completing/blocked`，阻塞状态还能带 `missing_task/task_too_old/migration_failed` 原因。检查本身不启动调度，也不执行任务的迁移函数，所以可以先读诊断，再决定是否 `resume()`。

`taskGraph()` 用来展示存活任务的归属与等待关系，类型见 [task-graph.ts:18-43](../../packages/durable/src/harness/task-graph.ts#L18-L43)。终态节点会被移除，completing 节点只保留精简的 outcome 状态，phase 来自检查点；完整检查点、outcome 的结果或错误载荷、阻塞原因都不在图里。任务关系的图示见 [多 Agent 编排](04-orchestration.md)。

## 边界与未实现

前面的记录与诊断都基于现有实现；接入真实追踪系统之前，以下缺口需要应用或后续版本补上：

- 当前代码尚未默认接通“任务、模型请求、工具、重试”的完整追踪系统，也没有内置指标服务或遥测导出器。
- 阅读时有一处文档与实现的差异：`packages/telemetry/README.md` 提到 `AGENT_TELEMETRY_SCHEMAS`、`startAiSpan`、`startHarnessSpan`，而当前 [agent/src/index.ts:1-5](../../packages/agent/src/index.ts#L1-L5) 及其导出源码没有这些定义，所以该段 README 所描述的自动埋点尚未在这个版本实现。
- `inspect()` 与 `taskGraph()` 两种接口展示当前状态，完整的历史追踪仍需另行记录。
- 接入后端前，应先定义一次任务、一次模型尝试、一次工具效果各自的 ID 和计数口径，避免重试导致成功率与成本重复计数。完整指标后端、跨进程追踪上下文传播、采样、导出器刷新、队列和重试的统一指标口径，以及敏感字段处理，目前都需要继续实现。执行恢复所需的持久状态仍由 [持久执行](05-durable-execution.md) 负责。

## 源码位置与术语速查

本章出现的遥测术语与项目实现的对应如下：

| 术语 | 具体含义 | 本项目对应 |
| --- | --- | --- |
| Trace | 把一次操作及其子操作串起来的记录 | Span 可记录父子关系；应用负责把业务操作接入 |
| Span | 一个操作从开始到结束的记录 | `startSpan(options, callback)` |
| Event | 操作期间发生的一件事，没有自身持续时间 | `addEvent()`、Agent 生命周期事件 |
| Metrics | 多次运行的汇总数值，例如失败率、延迟分布 | Session 和 eval 提供数据；没有通用指标后端 |
| 业务状态 | 恢复运行所必需的任务、消息与 Checkpoint | 由 Session/durable 保存，不能依赖遥测恢复 |

## 测试与练习

| 已有测试 | 学习目标 |
| --- | --- |
| [telemetry.test.ts:29-151](../../packages/telemetry/test/telemetry.test.ts#L29-L151) | 定义的编译期约束与父子创建器 |
| [testing/conformance.ts:60-315](../../packages/telemetry/src/testing/conformance.ts#L60-L315) | 回调只调用一次、保留错误值、结束后停止记录、记录失败不影响业务 |
| [conformance.test.ts:21-46](../../packages/telemetry/test/conformance.test.ts#L21-L46) | 读取快照不会暴露可变内部记录 |
| [ai/test/telemetry-options.test.ts:68-120](../../packages/ai/test/telemetry-options.test.ts#L68-L120) | 模拟模型服务收到同一个上下文；用例只验证透传 |
| [durable/test/harness-inspect.test.ts:94-110](../../packages/durable/test/harness-inspect.test.ts#L94-L110) | paused、blocked 与 ready 的诊断；检查不执行迁移 |

练习从 `InMemoryTelemetryContext` 开始，无需模型或密钥。在外层 `startSpan({ name: "task" }, callback)` 的回调中，用收到的 Span 并行创建两个子 Span，让第二个先结束，再检查它们的 `parentId` 是否相同、`endSequence` 是否按完成顺序变化。接着让回调返回 `{ ok: false }`，默认状态仍是 ok，需要显式设为 error。这可以观察业务返回失败与抛异常在自动状态判断上的区别。

仓库依赖已安装时，可以在 `packages/telemetry` 运行这些定向测试：

```powershell
node ../../node_modules/vitest/dist/cli.js --run test/telemetry.test.ts test/conformance.test.ts
```

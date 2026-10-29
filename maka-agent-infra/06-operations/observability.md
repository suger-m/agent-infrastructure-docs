# 请求计量、上下文诊断与故障排查

[手册目录](../README.md)

适用范围：截至 2026-10-24，Maka 固定提交 `15d0a0dd19d39e2bc09ae35940e432c636982315`，重点是 AI SDK 请求计量、运行事实、上下文诊断与 session trace。
Pi 以 `482c1381a00d8d00bc81697e4a980c593db6803e` 为基线，保留用户未提交修改；相对源码链接按当前工作区行号核对。
本文静态阅读源码和测试断言，未运行测试、安装依赖或调用外部模型。自定义 backend、未接入计量的请求和跨进程遥测不据此推定完整覆盖。

## 1. 问题：界面输出与任务成功的判定

用户请求“读取配置并解释启动失败”。界面先显示加载状态，随后输出部分解释，又报告连接中断；重试后获得完整答案。
同时，上下文面板仍显示上一次请求的输入量，费用页面却新增一次调用记录。若期间还执行了历史摘要，调用数可能进一步增加。
这些现象未必相互矛盾：它们观察的是不同对象、不同边界。

```text
一次用户输入
  → 请求尝试 A：已经发送，流中断，可能没有收到 usage
  → 请求尝试 B：重试成功，收到 usage
  → invocation 终态持久化
界面片段、A/B 的费用记录、最终执行结果，分别回答不同问题。
```

可观测性是保留足以解释执行过程的证据，不是新增一份可以独立判定成功与否的状态。
Trace 是关联操作的追踪视图；usage 是资源用量；diagnostics 是解释异常或请求组成的诊断信息。
本章首先考察“这条记录由谁产生、能证明什么”，再讨论统计与展示。

## 2. 五类记录：事实来源与面板

| 记录 | Maka 中的职责 | 不应混同为 |
| --- | --- | --- |
| `RuntimeEvent` | 运行事实：invocation opening、文本、工具动作、结果及终态 | 供应商请求账单，或每条 UI 消息 |
| `AgentRunEvent` | Operational ledger：计量、权限决定、请求组成、诊断等操作记录 | invocation 身份与最终结果的另一个独立权威 |
| `ModelCallAttempt` | 一次物理 provider 请求尝试的计量记录，载于 `model_call_attempt_recorded` | 一次用户输入、一个完整 run，或仅成功响应 |
| Usage 汇总 | 对已记录 attempts 的 token、费用、时长和覆盖情况做查询 | 供应商最终账单或无遗漏证明 |
| UI / RunTrace / SessionTrace | 进度展示、诊断事件、由账本生成的关联视图 | 执行恢复协议或持久成功凭证 |

`AgentRunEvent` 的说明明确将 invocation 的 route、配置、lineage 和 outcome 归属于 RuntimeEvent 事件主干。
同样，RuntimeEvent 上的 turn 级 `tokenUsage` 仍服务于重放和恢复，但不是逐请求费用的计量权威。
因此不应将其与 `ModelCallAttempt` 相加，否则同一消耗可能被重复计算。
参见 [操作账本契约](../../../../maka/packages/core/src/agent-run.ts#L20)及 [物理请求计量契约](../../../../maka/packages/core/src/model-call-attempt.ts#L33)。

关联关系也不能依据日志的相邻位置确定：session/run/turn 标识所属执行，`logicalCallId` 归组同一步的请求重试，`attemptId` 标识其中一次物理尝试。
`traceId` 在该计量结构中是 tracker 实例标识，不应直接解释为已贯通所有 worker 的分布式 Trace ID。
同一个 attemptId 的取消暂记与迟到结算，也不是两次请求；读取时必须去重。

## 3. Pending、streaming 与持久终态的状态机差异

| 看到的信号 | 能说明什么 | 还缺少什么证据 |
| --- | --- | --- |
| UI `pending` | 某项界面操作仍待完成 | provider 是否真正被调用 |
| UI `streaming` / 文本 delta | 正在展示流式活动或片段 | 完整响应是否被接受、运行是否结束 |
| attempt `completed/failed/interrupted/aborted` | 某次模型请求的结算分类 | 同一 run 是否还要调用工具或模型 |
| 非 partial 的 terminal RuntimeEvent | 该 invocation 的结束事实 | 是否已通过持久化屏障，以及产物是否符合用户目标 |

UI 的 `deriveComposerModelSwitchAvailability()` 分别接收 pending、streaming 和 sessionStatus，用于判断能否切换模型；它不是终态提交器。
`runtimeInvocationsFromSessionEvents()` 忽略 partial，以 opening 与 terminal 事件重建执行记录；`runtimeInvocationOutcome()` 从终态读出结果。
实际的终态提交函数 `commitTerminalRunWithRuntimeFact()` 验证终态身份、状态和非 partial 条件，再调用 `ensureTerminalRuntimeEventDurable()`。
参见 [UI 可用性](../../../../maka/packages/ui/src/composer-helpers.ts#L56)与 [终态持久化](../../../../maka/packages/runtime/src/terminal-run-commit.ts#L101)。

没有 attempt 记录不一定表示没有发出过请求：可能尚未结算，也可能 dispatch 后进程崩溃。
反之，dispatch 前已经取消的输入确实不应制造“已发送调用”；现有 mock 测试断言 provider 调用次数和 canonical attempt 数均为零。
缺少终态时也不能仅凭界面停止滚动输出推导成功；还需检查持久事件与当前执行者状态，而不应由诊断页面代替执行系统补出结论。

## 4. text 调用的计量证据

以下沿现有 `latest-context-commit.test.ts` 的 mock text 请求梳理生产接口；这是源码轨迹，不是本次实际执行日志。
测试返回文本 `system instructions, tools and messages.`，报告输入 120、输出 9 token，并验证重开存储后的读取。

```text
SessionManager.sendMessage(text)
  → AiSdkTurn：createTracker({ callKind: 'main', turnId, runId, ... })
  → buildPriorMessages() + 当前输入 → 请求投影与组装
  → tracker.setStep(runtimeSteps, requestCompositionId)
  → modelAdapter.startStream(...)
  → ProviderRequestTracker.trackStream()
    → 取消检查 → beforeDispatch → accounting.assertReady
    → preparedPromptComposition()：记录请求组成的字节统计
    → beginAttempt()：分配 logicalCallId / attemptId，记开始时间
    → provider 流：text-delta → finish，携带 usage
    → 适配器给出语义结果；finalize() 结合保留的 usage 结算
    → record({ attempt, latestContext })
  → backend / kernel 接口 → AgentRun → 存储提交
  → readLatestContextDiagnostics() 读到该请求的 120 inputTokens
```

`trackStream()` 支持由 `onAttempt` 交给适配器判定语义结果，不能仅依据原始 finish 包即忽略响应有效性。
未接该回调的 tracker 路径自行按流结束、错误及取消分类；不同入口的 `interrupted` 与 `failed` 不宜依据名称互换。
测试从真实 send 接口穿过计量转发链，验证已有 projection 时无需扫描账本；绕过 projection 读取时仍可从 canonical attempts 重建组成信息。
证据见 [请求 tracker](../../../../maka/packages/runtime/src/provider-request-telemetry.ts#L457)和 [跨层提交测试](../../../../maka/packages/runtime/src/__tests__/latest-context-commit.test.ts#L53)。

### 4.1. 失败、取消与辅助请求的计量

| 字段 | 口径 |
| --- | --- |
| `status` | 请求结果，与有没有 usage 分开记录 |
| `usageBasis: reported` | inputTokens 与 outputTokens 都有；不代表所有可选缓存细项都齐全 |
| `usageBasis: partial/missing` | 只收到部分字段，或没有可用用量；缺失不是零 |
| `costBasis: priced/unpriced` | 能否依据已知用量和价格计算金额；与请求成功与否分开 |
| `costUsd` | 只在 priced 时出现；价格依据随记录保留，不是事后随意套新价格 |

`strictProviderRequestUsage()` 避免将适配器补出的零视为供应商证据；存在 raw payload 时会核对真实字段，再按供应商语义处理缓存部分。
费用计算不以 `status === completed` 为前提。失败或取消请求只要留下可计价 usage，就不能从账单中剔除。
取消但尚无 usage 时可以先记录 provisional aborted；迟到的 finish 用同一个 attemptId 补结算，写入按顺序串行，最终读取去重。
现有测试验证取消后仍可落入 `$0.003` 的已计价记录；该数值是测试夹具，不是产品价格。

历史压缩同样要付出请求成本：`AiSdkCompaction` 为摘要创建 `callKind: 'history_compact'` 的 tracker。
`historyCompactRoute` 记录物理尝试实际使用的 `provider_native` 或 `text_summary`；原生请求失败转而采用文本摘要时，两个尝试均保留。
压缩 checkpoint 是被应用的上下文边界，摘要模型请求是消耗；trace 可以同时展示二者，但不应将边界本身再计为一次模型调用。
参见 [压缩 tracker 创建](../../../../maka/packages/runtime/src/ai-sdk-compaction.ts#L995)、[路由回退测试](../../../../maka/packages/runtime/src/__tests__/provider-request-telemetry.test.ts#L821)和 [取消后补 usage](../../../../maka/packages/runtime/src/__tests__/provider-request-telemetry.test.ts#L1021)。

### 4.2. 汇总数与覆盖限制

Usage SQL 按计价记录求费用和，同时给出 priced/unpriced、reported/partial/missing 的数量；不是只汇总成功请求。
查询中的 errorRequests 仅统计 `failed`；`interrupted` 与 `aborted` 另归停止类，不能依据该 error 数直接宣称“所有非成功比例”。
`ModelCallCoverage` 只描述已经存在的记录。dispatch 后、settlement 前崩溃可能完全没有记录，coverage 无法证明没有漏计。
因此“已记录费用之和为零”“所有记录都有价格”“实际账单为零”是三个不同结论。
对应 [汇总 SQL](../../../../maka/packages/storage/src/model-call-usage-sql.ts#L88)与 [coverage 契约](../../../../maka/packages/core/src/model-call-attempt.ts#L661)。

## 5. Context 诊断：占用总量与构成

“当前输入占窗口多少”与“哪些内容最大”需要两种读数，不能将二者合并为一个精确 token 分解。
`inputTokens` 是该请求的供应商用量；`promptComposition` 是 AI SDK 模型调用边界上语义片段的序列化字节统计，不是最终 HTTP wire body。
组成分为 system instructions、tool definitions、messages、other；工具按大小列出，超出上限的部分保留数量与字节余项。
字节数不能与 inputTokens 直接相加或要求两者总和相等，尤其附件的 base64 会放大序列化长度。

新版 `preparedPromptComposition()` 就地折叠请求组成，attempt 不再写入完整请求副本或新的 captureArtifactId。
旧 `requestObservation` 和旧 provider-attempt 记录仍有兼容读取代码；这不表示旧 capture 路径仍是当前写入机制。
参见 [请求组成测量](../../../../maka/packages/runtime/src/request-shape.ts#L87)及 [组成折叠](../../../../maka/packages/runtime/src/prompt-composition.ts#L58)。

`latest_context` 是 derived projection，不是新增的一类 emitted event。
只有 completed main attempt 才携带它一同提交：冻结该请求的身份、输入量、窗口、组成和构建请求时的压缩边界。
失败、取消或摘要请求不会覆盖它；这是开篇“费用增加，上下文面板仍显示上次主请求”现象的一种正常解释。
排序采用 completedAt，再以 attemptId 打破平局；不应将并发写入的到达顺序视为请求完成顺序。

`readLatestContextDiagnostics()` 优先读封存快照；缺失或不可读时才尝试账本重建，区分 `no_completed_request` 与 `trace_unavailable`。
冷路径一旦读取过 canonical 记录，就不会因为它们全部为失败或辅助调用而转用旧 provider 记录选取一个“看起来成功”的请求。
当前请求缺少 composition 时也不借用上一条请求的组成进行填充；否则同一面板会混合两次不同请求的数据。
其描述对象是最近完成的主请求，并不承诺精确预测下一次压缩后请求的 token 数。上下文构造见 [历史事实与模型输入投影](../02-context/history-and-projection.md)。

## 6. Trace 与 diagnostics：解释而不改写事实

`projectSessionTrace()` 是纯投影：调用者提供 RuntimeEvent 和 canonical attempts，它按 `(runId, turnId)` 关联。
首先排除 partial、按 attemptId 去重，再按 logicalCallId 将重试嵌在同一个模型步骤下；工具以调用标识配对，而非依据行号相邻。
工具失败后被恢复、run 最终 completed 的情况下，`attributeTurnFailure()` 不会仅凭中间错误把整轮改判失败。
没有已知缺口时 coverage 写 `no_known_gap`，刻意不写 complete；缺调用、读取失败或超出在线表示预算则明确报告缺口。
对应 [session trace 投影](../../../../maka/packages/runtime/src/session-trace-projection.ts#L70)和 [失败归因](../../../../maka/packages/runtime/src/session-trace-projection.ts#L422)。

`RunTrace.sendDiagnostics()` 保存本次 send 的结束分类、累计步骤用量及 contextBudget/toolAvailability；它不含费用，不作为计量累加来源。
这尤其覆盖 exhausted/aborted 等没有最终 `token_usage` SessionEvent 的路径，避免遗失当时的压缩决定与上下文诊断。
`RunTrace.emit()` 忽略 recorder 的同步异常和异步拒绝；诊断写入失败不应把模型或工具执行改判失败。
但这不表示整个 operational ledger 均可丢弃：tracker 的 `beforeDispatch` 与 `accounting.assertReady` 是发送前门禁，可以阻止尚未发生的调用。
已发生请求在 settlement 写账失败时，tracker 不让该写入异常破坏模型流；完整性问题需要由计量 sink 报告和后续门禁处理。
见 [诊断派发](../../../../maka/packages/runtime/src/run-trace.ts#L110)与 [写账失败、发送门禁测试](../../../../maka/packages/runtime/src/__tests__/provider-request-telemetry.test.ts#L977)。

“诊断被动”也不表示零开销：记录器的同步工作、请求组成测量和被等待的写入仍会消耗时间。
错误诊断对消息脱敏、限制长度，并保存摘要值；通用 `sanitizeTraceData()` 只过滤 undefined，不是任意自定义 data 的完整隐私清洗器。
因此不应将凭据、完整文件或请求正文直接纳入自定义诊断字段，也不应将哈希理解为恢复正文的能力。

### 6.1. 慢请求排查顺序

1. 首先依据 session/run/turn 定位持久终态；若无终态，区分仍活动、等待、读取失败和可能中断，不预先宣布成功或失败。
2. 检查模型步骤及每个 attempt 的耗时、首输出时间、状态、错误类别、重试序号、usageBasis；不应只关注最后一个成功响应。
3. 检查工具调用与结果、权限等待、压缩边界；确认耗时来源是主请求、摘要请求，还是请求之外的未测量区间。
4. 最后核对 context 组成和 coverage，决定是减少工具 schema、处理历史膨胀，还是继续排查连接与存储。

attempt 的 `latencyMs` 从该次尝试开始到结算计算，不含此前的 dispatch 门禁等待，也不等于用户端到端延迟。
`timeToFirstTokenMs` 实际由首个被识别的输出 part 触发，不只指可见文字；两次重试之间的等待也不能纳入某次网络调用。
并行工具的耗时不可直接求和作为总延迟。若要统计 p95、队列等待或跨 worker 关键路径，还需采集相应边界；本章代码不自动提供完整指标后端。

## 7. 与 Pi 对照：事件接口与全链路追踪

下表针对默认 `coding-agent → agent → ai` 路径。实验性 durable 具有独立的任务检查与恢复机制，不能依据本表推断 Pi 没有恢复、编排或沙箱能力。

| 问题 | Maka 本章路径 | Pi 默认路径与 telemetry |
| --- | --- | --- |
| 流式进度 | UI/SessionEvent 与 RuntimeEvent 事实层分开 | `message_update` 更新 streamingMessage，`message_end` 接收完整消息 |
| 执行结束 | terminal RuntimeEvent 通过持久化屏障 | `agent_end` 是循环末事件；Agent 仍等待监听器结算才 idle |
| 调用费用 | canonical attempt 区分逻辑调用、物理尝试和计量覆盖 | 会话条目携带 usage，再汇总；不据此证明所有底层重试都被逐次计量 |
| 摘要成本 | history_compact 独立归类，checkpoint 边界不重复计费 | compaction、branch summary 的 usage 也可进入会话累计 |
| 上下文数值 | 最近 completed main 请求的封存快照 | session totals 与 contextUsage 分开，压缩后当前用量可暂为未知 |
| Trace 接口 | 从两类账本投影 session trace，并报告覆盖缺口 | telemetry 提供显式 span 接口；应用仍需接采集、关联和导出 |
| 观测失败影响 | RunTrace recorder 异常被忽略；发送门禁另算 | 内存 telemetry 被动记录，但普通 Agent listener 是被等待的执行参与者 |

Pi 的 `Agent.subscribe()` 只负责注册 listener；`processEvents()` 先更新内存状态，再按订阅顺序逐个 await listener。
因此一个执行缓慢的 listener 会延长 settle 时间；`agent_end` 到达不等于 `waitForIdle()` 已经完成。
Session 扩展生命周期事件也不全是纯日志：`session_before_switch` 可以返回 cancel，现有测试验证它阻止切换。
这类控制 hook 不能与“失败应被忽略”的 telemetry recorder 混为一谈。

工具 `executePreparedToolCall()` 使用 `performance.now()` 测量 execute 的耗时；参数准备、before/after hook 不属于这项 durationMs。
并行结束事件按完成先后到达，工具结果消息保持调用顺序；诊断应按 toolCallId 配对，而非将两个列表按位置对应。
参见 [Agent 订阅](../../../packages/agent/src/agent.ts#L266)、[工具计时](../../../packages/agent/src/agent-loop.ts#L821)与 [Session 切换取消测试](../../../packages/coding-agent/test/agent-session-runtime-events.test.ts#L159)。

`getSessionStats()` 的测试给出直观示例：180k、195k 两次输入被压缩后，累计仍为 375k，而当前 contextUsage 为未知；新请求报告 25k 后，累计变为 400k、当前读数为 25k。
`getUsageCostBreakdown()` 还汇总工具结果、独立 usage 条目、压缩和分支摘要，不能把累计值解释为最后一个请求的成本。
参见 [统计口径测试](../../../packages/coding-agent/test/agent-session-stats.test.ts#L119)与 [费用分组](../../../packages/coding-agent/src/core/usage-totals.ts#L61)。

`pi-telemetry` 的 `startSpan/addEvent/setAttributes/setStatus` 可显式建立父子关系；`InMemoryTelemetryContext` 保存 parentId、状态和 endSequence，但不保存时间戳或 duration。
`telemetryContext` 可透传于 AI 请求选项，不证明每个 provider 都创建了 span，更不证明默认 CLI 已自动追踪所有工具、重试与子任务。
`isInstallTelemetryEnabled()` 读取的是安装遥测开关，不能依据 `PI_TELEMETRY` 名称将其视为全链路 tracing 开关。
内存 adapter 适合验证生命周期；计时、采样、跨进程传播和 exporter 仍需相应实现，不能依靠打印 AgentEvent 自动补齐。

## 8. 源码定位

| 要验证的问题 | 入口 |
| --- | --- |
| invocation 结果从哪里读取？ | Maka [`runtimeInvocationOutcome()`](../../../../maka/packages/core/src/runtime-invocation.ts#L214) |
| 一次物理尝试保存什么？ | Maka [`ModelCallAttempt`](../../../../maka/packages/core/src/model-call-attempt.ts#L174) |
| 何时分配身份、接收迟到 usage？ | Maka [`ProviderRequestTracker.beginAttempt()`](../../../../maka/packages/runtime/src/provider-request-telemetry.ts#L568) |
| 如何避免把缺失值当零？ | Maka [`strictProviderRequestUsage()`](../../../../maka/packages/runtime/src/provider-request-telemetry.ts#L816) |
| 最新上下文如何读、如何回退？ | Maka [`readLatestContextDiagnostics()`](../../../../maka/packages/runtime/src/context-diagnostics.ts#L108) |
| 快照如何绑定同一次请求？ | Maka [`latestContextProjectionInput()`](../../../../maka/packages/runtime/src/latest-context-snapshot.ts#L81) |
| send 诊断为何不应加入费用？ | Maka [`RunTrace.sendDiagnostics()`](../../../../maka/packages/runtime/src/run-trace.ts#L204) |
| trace 为什么只能说无已知缺口？ | Maka [`resolveCoverage()`](../../../../maka/packages/runtime/src/session-trace-projection.ts#L124) |
| Pi 消息如何更新状态并派发？ | Pi [`Agent.processEvents()`](../../../packages/agent/src/agent.ts#L565) |
| span 结束与业务错误怎样传播？ | Pi [`startInMemorySpan()`](../../../packages/telemetry/src/memory.ts#L120) |
| telemetry 是否真的透传？ | Pi [`telemetry-options.test.ts`](../../../packages/ai/test/telemetry-options.test.ts#L68) |
| 安装遥测开关是什么？ | Pi [`isInstallTelemetryEnabled()`](../../../packages/coding-agent/src/core/telemetry.ts#L8) |

## 9. 本章术语

- Operational ledger：与运行事实并行保存的操作账本；含计量、决定及诊断，不等于可丢弃日志。
- Logical call / attempt：逻辑模型步骤与其中一次物理请求尝试；重试增加 attempt，不必增加逻辑调用。
- Settlement：请求结果与用量的结算；取消通知和供应商最终用量可能不同步。
- Durable terminal：已通过持久化屏障的执行终态；不同于消息结束或界面停止流式显示。
- Coverage：已记录数据的覆盖说明；无已知缺口不等于已证明没有遗漏。
- Span / trace：单个操作及其关联追踪；本地 tracker 标识不自动成为分布式追踪上下文。
- Context composition：请求组成的字节统计；不是供应商 token 的精确分项。

## 10. 源码练习

1. 以纸面推演构造三条记录：main 失败且 usage missing、main 重试成功且 priced、history_compact 成功但 unpriced。分别计算尝试数、errorRequests、已知费用，并指出哪条能更新 latest_context；解释为什么“已知费用”不是完整账单。
2. 阅读 [取消后迟到用量测试](../../../../maka/packages/runtime/src/__tests__/provider-request-telemetry.test.ts#L1021)：绘制 aborted 暂记与 finish 补记的时序，说明按 attemptId 去重为何不会抹除真正的重试。再将取消移至 dispatch 前，对照 [零调用测试](../../../../maka/packages/runtime/src/__tests__/latest-context-commit.test.ts#L194)。
3. 以纸笔或本地内存 adapter 建立父 span 和两个子 span，使第二个先结束；检查 parentId、endSequence 能回答什么、不能回答什么。再比较 Agent listener 等待与 telemetry recorder 异常处理，说明为何不能用同一套“观测失败全部忽略”规则处理控制 hook。

排查时应先确认事实与计量口径，再解释图表。执行正常结束不保证产物正确；后一问题由[评测机制](evaluation.md)中的 verifier 检查。

## 相关专题

- [领域对象与身份](../00-foundations/domain-model.md)：先确认一次请求、步骤和尝试的计数边界。
- [重试机制](retry-timeout-idempotency.md)：解释为什么同一步会出现多条物理请求记录。
- [评测与验证](evaluation.md)：从执行收口继续检查任务产物是否正确。

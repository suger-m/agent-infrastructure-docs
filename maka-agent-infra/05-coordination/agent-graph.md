# 任务图与多 Agent 协作

[手册目录](../README.md)

适用范围：Maka 固定提交 `15d0a0dd19d39e2bc09ae35940e432c636982315` 的 Host Agent Graph 与 child Session 路径。
Pi 对照基线为 `482c1381a00d8d00bc81697e4a980c593db6803e`，区分默认 coding-agent、可选扩展示例和实验性 durable。
核对日期：2026-10-24。本文依据本地源码与测试夹具静态阅读；未运行测试、模型调用或外部服务。

## 1. 问题：子任务结果的可信接入

假设需要回答：“配置字段 `timeout` 的默认值、覆盖顺序与测试约束是否一致？”
可安排 A 阅读配置实现、B 阅读测试，再由 C 比较两份结论；父助手最终组织回答。
这并非简单地同时启动三个模型：C 必须获得 A、B 的具体结果，不应将“已经启动”视为“结果可用”。
父助手暂停当前回答、某个子任务失败、进程重启，同样分别需要明确的处理方式。

本章用以下教学任务贯穿机制，ID 均为说明性占位符：

```text
父助手：安排 A（读实现）与 B（读测试）
A 的已提交结果 rA ─┐
                    ├─ C(input_ids=[rA,rB])：检查两份结论的冲突
B 的已提交结果 rB ─┘
C 的已提交结果 rC → 父助手审阅 → finish(result_ids=[rC])
```

在 Maka 中，父助手可以在获得结果之后再追加 C，不要求提前生成完整的流程图。
必要机制是：明确工作身份、输入来源、执行所有权及结束条件；增加更多角色本身并不能解决这些问题。

## 2. graph、Session 与 activation 的区分

| 术语 | 本章中的准确含义 |
| --- | --- |
| supervisor | 持有图控制工具的根 Session 中的父助手；不是每个 child 都能继续建立控制面 |
| Agent Graph | 一个图身份下的工作安排、operator 绑定及运行记录投影；不是单个模型的思考链 |
| operator | 图中的执行对象；在此 Host 路径中绑定一个稳定的 child Session |
| activation | operator 的一次执行；投影中 `activationId` 使用该次 `runId` |
| work / intent | supervisor 提交的工作请求，以及调度时据此形成的可执行意图 |
| claim | 持久化认领记录，固定意图与执行身份；认领本身不等于模型已开始运行 |
| record | 从已提交 RuntimeEvent 投影的图记录，保留事件来源与分类，不复制全部事件正文 |
| handoff | 调度时按 record 回读的上游结论包，交给下游作为数据；与 04-state 的执行交接 handoff、输入移交 handoffRootSources 同名不同义 |

同一个 child Session 可以承接后续 activation，不能用 Session 数量推断执行次数。
`projectAgentGraphRecords()` 明确区分 operator、Session、run 和 event；重放时按 activation 汇总状态。[投影实现][projection]
根 Session 的图工具入口会拒绝带 `subagentParent` 的 Session。[根控制面][root-tools]

这里的 graph 不是 LangGraph 的同义词，也不自动意味着任意条件边、共享状态 reducer 或循环工作流。
应分别检查“当前 Host 实际安装了什么”与“底层库允许实现什么”，不能仅凭名称推断其能力。

## 3. 编排边界：模型可安排工作，不可安装能力

开发者负责提供工具、agent 定义、执行器与权限边界；不必预写每次任务的完整 A→B→C 流程。
`renderGraphModePrompt()` 明确指导模型调用 `agent_list`、`update_agent_graph`、`agent_output` 与 `yield_agent_graph`。[图模式提示][graph-prompt]
`update_agent_graph` 的实现不是占位：编译请求、授权、提交 schedule update，然后通知协调器处理。[控制工具][supervisor-tools]
模型可以添加工作、停止工作、选择最终结果；可以提出什么，与 Host 是否准许执行，属于两个层面的判断。

当前内置定义仅有以下三类；此表并非表示“启动时已经存在三个 operator”：

| agent ID | 子上下文、工具及工作区契约 | 可用性边界 |
| --- | --- | --- |
| `local-read` | 独立上下文；`Read/Glob/Grep`；同工作区；summary 回传 | 所需工具必须可用 |
| `web-research` | 独立上下文；`WebSearch`；同工作区；summary 回传 | 必须提供 WebSearch |
| `implementation` | 独立上下文；文件与 Bash 工具；独立 worktree；patch 回传 | 必须有 worktree executor，并满足工具要求 |

这些定义及过滤逻辑见 `BUILTIN_AGENT_DEFINITIONS`、`listRunnableBuiltinAgentDefinitions()`。[内置目录][catalog]
“独立上下文”不等于“独立文件系统”；前两种定义仍标注 same workspace。
`agent_list` 还可列出 Host 提供的 preset，但不能假定任意安装都有相同 preset 或 executor。[Agent 目录工具][agent-list]
`new_agent`、`new_preset`、`existing_operator` 是目标选择方式，不是三种新的通用计算节点。

Host 的 `#reconcileOnce()` 从空的基础 topology 加上持久 provision 构建执行对象，operator 随工作动态建立。[Host 调和入口][coordinator]
底层 readiness 确实实现了 `map` 与 `all_settled`；后者等待显式指定的上游 activation 全部结束，结束不等于成功。[通用 readiness][readiness]
但当前协调器观察 topology 时传入 `policies: []`，实际走 supervisor schedule，不应写成默认安装了 map/reduce/join 算子库。[默认策略][default-policies]
这里的可扩展策略已有代码，并非全是规划；它们也不是当前 Host 路径自动启用的行为。

## 4. 协议轨迹：提交安排与等待子结果

以下顺序来自 `AgentGraphProviderScenario.respond()` 测试夹具；省略诊断读取和重复等待分支。
它使用受控 provider 回复来验证调用协议，不是本文现场执行的日志。[协议夹具][scenario]

```text
根模型 → update_agent_graph({operation:"add_work", add_work:[{
  target_kind:"new_agent", agent_id:"local-read",
  instruction:"Inspect the hosted Graph execution boundary.",
  input_ids:[], replacement_mode:"none"
}]})
工具 → agent_graph_updated，schedule.closed=false
根模型 → yield_agent_graph({reason:"Wait for the hosted operator checkpoint."})
Host → 后续根输入包含 “reached a durable supervisor checkpoint.”
根模型 → view_agent_graph({mode:"latest"})
视图 → operator.completed，提供 childSessionId / currentRunId
根模型 → agent_output({locator:"child_session_run", child_session_id:…,
                       run_id:…, view:"result", max_bytes:32768})
结果 → completed、text、resultRecordId
根模型 → update_agent_graph({operation:"finish", finish:{result_ids:[…], reason:…}})
工具 → schedule.closed=true
根模型 → 最终文本回答
```

工具返回“安排已提交”与 child 返回“结果已完成”之间，实际还有以下实现步骤：

1. `compileAgentGraphScheduleUpdate()` 根据调用来源生成 update/work 身份；提交的是持久安排，不是直接等待 child 完成。
2. `AgentGraphCoordinator.#reconcileOnce()` 调用 `reconcileAgentGraphSchedule()`，读取 schedule、provision、claim 与已提交事件投影。
3. 新工作缺少 `input_ids` 对应记录时标为 `input_not_committed`；输入齐备后才请求 provision child operator。
4. 调度器回读输入结论并生成 prompt，再调用 `dispatchScheduledWork()`。
5. `claimAgentGraphRunnableIntent()` 经 schedule revision 校验认领，随后 `runClaimedAgentGraphIntent()` 交给执行器。
6. 开始执行仍需通过 `beginAgentGraphIntentExecutionAtScheduleRevision()`；旧 revision 产生冲突时返回 stale，而非无条件启动。

步骤 2–6 的入口集中在 schedule reconciler；claim 保存的是执行身份和输入，不是外部副作用“恰好一次”的证明。[调度实现][reconcile] [认领与启动][dispatch] [claim 实现][claim]
图允许一批无依赖工作并行准备和 dispatch，但实际运行仍受[执行容量](queues-and-admission.md)约束。
`maxNewActivations` 是一次调和允许新增的 activation 数量，不应视为全局同时运行名额。

## 5. 依赖对象：已提交 event 而非流式文字

继续 A、B、C 的示例：A 已打印“默认值是 30”时，尚不能将这段屏幕输出视为 C 的稳定输入。
`readCommittedAgentGraphProjection()` 读取 Session 的 immutable RuntimeEvents；`projectAgentGraphRecords()` 跳过 partial event。[投影实现][projection]
因此，UI 预览、observer 回调与持久数据路径不能互换。

父助手通过 `agent_output` 的 `child_session_run` 定位某次运行，读取结果与 `resultRecordId`，再填入 C 的 `input_ids`。[结果读取工具][agent-output]
`missingWorkInputIds()` 检查这些 ID 是否已存在于图投影；它不验证结论正确，也不要求每个 ID 都是成功终态。[输入检查][input-check]
若指定的是失败事件或中间已提交记录，不能仅凭“依赖满足”判定业务成功。
需要完整子答案时，应读取明确的运行结果、检查状态，再选取对应结果 ID。

`buildOperatorProvisionInput()` 根据输入 record 的来源 operator 建立指向新 operator 的边。[建立依赖边][provision]
输入依赖同时具有身份与来源；并非将父会话全文广播给所有子助手。
当前图的 `input_ids` 与历史 `selected_result_inputs` 也不同：后者只接收同一根 Session 较早已关闭图中明确选出的 finish 结果。[历史结果解析][historical]

下游需要正文时，`hydrateAgentGraphInputHandoffs()` 按 Session/run/event 回读不可变事件。
它从选中事件向前寻找最近的非 partial 模型文本，附带来源信息；没有可用文本时仍保留引用。
默认每份结论上限 16 KiB，合计 48 KiB，记录是否截断；这些是结论正文预算，不是整个 prompt 上限。[输入交接][handoff]
提示协议要求将上游结论视为数据，而非新指令；这是一项防护设计，并非对任意提示注入免疫的保证。

产物也需区分：模型结论是文本，`artifact_update` 是事件分类，implementation 的 patch 是写回契约。
`runtimeEventFacets()` 在有 `artifactDelta` 时标记产物更新；图记录仍保留来源引用。[产物事件分类][facets]
不能由“图看到了 artifact”推导出文件已复制、patch 已应用或冲突已解决；本章未验证各执行器的合并与清理分支。

## 6. 父子生命周期：yield、finish、stop 的结束范围

| 动作或边界 | 已验证的含义 | 不能推导的结论 |
| --- | --- | --- |
| `agent_spawn` | 普通前台子调用等待 `spawnChildSession()` 的结果，再投影为工具结果 | 不是一次 graph schedule update |
| `yield_agent_graph` | 让出当前 supervisor turn，图保持开放，后续可由 Host checkpoint 唤醒 | 不是关闭图或取消 child |
| `finish` | 选择已提交结果并关闭 schedule 的新工作准入 | 不是所有 child 都已停止，也不是自动验收结论 |
| schedule stop | 根据 work/claim 定位执行；未开始的可取消，已运行的按具体身份停止 | 不能只按 operator 名误停后来的一次 run |
| Host 图停止 | 协调器停止已知 operator；等待在途驱动后再次扫描，覆盖 provision 竞态 | 不等于删除持久安排与历史结果 |

普通子调用与图安排共享 child 能力，但所有权不应混为一个父工具调用栈。[前台子调用][spawn]
图驱动由 Host 协调器持有；schedule、topology provision、claim 与运行事实用于恢复，内存 driver 负责本进程的协调与取消。
`#stopDriver()` 暂停驱动并停止 children，之后的 schedule update 可以重新唤醒驱动；这不同于 finish 封闭当前图。[停止与驱动][stop-driver]

`assertFinishResultsCommitted()` 校验所选 record 已提交，而不是检查所有 operator 均成功。[finish 校验][finish-check]
reconciler 拒绝关闭图中的新认领，但已有 claim 仍可进入完成或恢复路径。[调度实现][reconcile]
因此，父回答已结束、当前 activation 已结束、整个图已封闭，是三个独立状态。
恢复 claim 会重新核对持久状态；模型连接与 OS 进程的处理见[执行恢复](../04-state/durable-execution.md)。

## 7. 与 Pi 对照：默认助手、扩展示例与 durable

| 比较项 | Maka Host Agent Graph | Pi 默认 coding-agent | Pi subagent 扩展示例 | Pi 实验性 durable |
| --- | --- | --- | --- | --- |
| 编排入口 | 根模型调用图工具，Host 调和 schedule | 默认工具集不含 subagent 或图控制工具 | 安装扩展后，模型调用 `subagent` | 注册工具或 task，再由模型或程序提交 |
| 子执行对象 | operator 绑定 child Session，run 是 activation | 本行不自动建立多 Agent 编排 | 启动独立 Pi 子进程 | task，以及 task 拥有的 child conversation |
| 依赖表达 | 已提交 record ID；按来源建立依赖 | 普通工具输入与结果 | single / parallel / chain，chain 替换 `{previous}` | checkpoint、等待 task ID、读取 outcome |
| 父子生命周期 | yield、finish、stop 与 child 终态分别处理 | 不应据此概括整个 Pi 的能力 | 工具调用管理进程与结果 | ownership 参与完成、取消与等待关系 |
| 本路径恢复依据 | schedule/provision/claim 与 RuntimeEvents | 不等同于 durable scheduler | 此示例用 `--no-session`，没有持久编排账本 | 存储中的 task 状态、checkpoint 与请求身份 |

Pi 默认 `createCodingTools()` 返回读、shell、编辑、写入工具，并不自动安装 subagent；扩展可以增加工具。[Pi 默认工具][pi-default]
这不表示 Pi 缺少会话持久化、恢复、编排或隔离能力，仅表示不应将可选机制视为默认 Agent loop 的职责。

subagent 扩展的 `runSingleAgent()` 启动 `pi --mode json -p --no-session`，收集子进程消息。
chain 逐个执行并把上一结果替入 `{previous}`；parallel 最多接受 8 项，以 4 个并发 worker 调用。[Pi 扩展执行][pi-extension]
它适合展示模型驱动的拆分，但这些内存结果与 worker 不等同于 Maka 的持久 schedule 和 committed-event 输入。

durable 的前台示例则有另一条真实轨迹：[Pi durable 前台子助手][pi-durable-agent]

```text
subagent.execute → api.commit：按 ownerTaskId 查找或创建 child conversation
                 → 从 child 配置移除 Subagent 扩展，避免递归委派
                 → submit(requestId="subagent:" + api.taskId).wait()
                 → 读取子答案，返回父工具结果
```

同一工具任务重放时，可找回既有 child 与同一 request；这与“启动一个新的 CLI 子进程”有不同的身份与恢复语义。
durable 的 `Checkout` 示例还在一次 commit 中创建 owned `Payment` tasks，并提交 `waiting + checkpoint + on + failFast`。[Pi 子任务][pi-child-tasks]
`Scheduler.#commitState()` 在仍有 owned live work 时把父终态转成 `completing`；`#validateWait()` 限制 failFast 的所有权，并调用 `#waitCycle()` 检查等待环。[Pi ownership 与等待][pi-ownership]
这些机制属于实验 durable 路径；不应直接套用于默认 coding-agent，也不能反向断言 Pi 没有父子生命周期管理。

## 8. 源码定位

| 待追溯的问题 | 文件与函数入口 |
| --- | --- |
| 模型被告知如何编排 | [graph-mode.ts：`renderGraphModePrompt()`][graph-prompt] |
| 图工具是否实际提交安排 | [stream-graph-supervisor-tools.ts：`buildAgentGraphSupervisorTools()`][supervisor-tools] |
| Host 如何推动工作 | [stream-graph-coordinator.ts：`#reconcileOnce()`][coordinator] |
| 输入不齐时为何不启动 | [stream-graph-schedule-reconcile.ts：`missingWorkInputIds()`][input-check] |
| 认领与执行之间的边界 | [stream-graph-schedule-reconcile.ts：`dispatchScheduledWork()`][dispatch] |
| Session/run/event 如何变成图记录 | [stream-graph-projection.ts：`projectAgentGraphRecords()`][records] |
| 上游结论如何进入下游 prompt | [stream-graph-handoff.ts：`hydrateAgentGraphInputHandoffs()`][handoff] |
| 默认能力与可选策略如何区分 | [agent-catalog.ts：`listRunnableBuiltinAgentDefinitions()`][availability]；[stream-graph-readiness.ts：`buildAgentGraphReadinessSnapshot()`][readiness] |
| Pi 扩展如何执行子进程 | [subagent/index.ts：`runSingleAgent()`][pi-extension] |
| Pi durable 如何约束父结束与等待 | [scheduler.ts：`#commitState()` / `#validateWait()`][pi-ownership] |

## 9. 源码练习

1. 纸上演算输入边界：A 有 partial 文本，B 有已提交失败 record，C 声明两个输入。判断哪些 ID 能进入投影、哪些会使 C 等待，并解释“可调度”为什么不等于“业务成功”。
2. 对照协议夹具与 reconciler：在 child 仍运行时分别考虑 yield、finish、stop。写出根 turn、schedule、child activation 三者的状态变化；找出 finish 未承诺的两项内容。
3. 对照 Pi 两份子助手实现：列出 CLI 扩展和 durable 前台示例保存 child 身份的位置。假设父进程在提交子请求后退出，说明哪条路径有稳定 request ID，以及仅有 request ID 还不能保证哪些外部效果。

以上练习只需静态阅读与手工记录。本章已区分实现、默认接线与可扩展设计；执行器内部产物合并、外部效果幂等性和完整恢复行为不在本次验证范围内。

## 相关专题

- [领域模型](../00-foundations/domain-model.md)：确认输入、执行、步骤和尝试的身份边界。
- [持久化与恢复](../04-state/durable-execution.md)：追踪状态提交与中断后继续执行的条件。

[graph-prompt]: ../../../../maka/packages/runtime/src/graph-mode.ts#L20
[root-tools]: ../../../../maka/packages/runtime/src/stream-graph-coordinator.ts#L480
[supervisor-tools]: ../../../../maka/packages/runtime/src/stream-graph-supervisor-tools.ts#L549
[catalog]: ../../../../maka/packages/runtime/src/agent-catalog.ts#L137
[availability]: ../../../../maka/packages/runtime/src/agent-catalog.ts#L293
[agent-list]: ../../../../maka/packages/runtime/src/subagent-tools.ts#L355
[coordinator]: ../../../../maka/packages/runtime/src/stream-graph-coordinator.ts#L1031
[readiness]: ../../../../maka/packages/runtime/src/stream-graph-readiness.ts#L155
[default-policies]: ../../../../maka/packages/runtime/src/stream-graph-coordinator.ts#L859
[scenario]: ../../../../maka/packages/runtime-host/src/__tests__/fixtures/agent-graph-provider-scenario.ts#L45
[reconcile]: ../../../../maka/packages/runtime/src/stream-graph-schedule-reconcile.ts#L186
[dispatch]: ../../../../maka/packages/runtime/src/stream-graph-schedule-reconcile.ts#L953
[claim]: ../../../../maka/packages/runtime/src/stream-graph-admission.ts#L81
[projection]: ../../../../maka/packages/runtime/src/stream-graph-projection.ts#L254
[records]: ../../../../maka/packages/runtime/src/stream-graph-projection.ts#L392
[agent-output]: ../../../../maka/packages/runtime/src/subagent-tools.ts#L565
[input-check]: ../../../../maka/packages/runtime/src/stream-graph-schedule-reconcile.ts#L664
[provision]: ../../../../maka/packages/runtime/src/stream-graph-schedule-reconcile.ts#L1114
[historical]: ../../../../maka/packages/runtime/src/stream-graph-coordinator.ts#L1620
[handoff]: ../../../../maka/packages/runtime/src/stream-graph-handoff.ts#L76
[facets]: ../../../../maka/packages/runtime/src/stream-graph-projection.ts#L603
[spawn]: ../../../../maka/packages/runtime/src/subagent-tools.ts#L93
[stop-driver]: ../../../../maka/packages/runtime/src/stream-graph-coordinator.ts#L910
[finish-check]: ../../../../maka/packages/runtime/src/stream-graph-supervisor-tools.ts#L1072
[pi-default]: ../../../packages/coding-agent/src/core/tools/index.ts#L195
[pi-extension]: ../../../packages/coding-agent/examples/extensions/subagent/index.ts#L270
[pi-durable-agent]: ../../../packages/durable/test/examples/22-subagent-foreground.ts#L39
[pi-child-tasks]: ../../../packages/durable/test/examples/24-child-tasks.ts#L66
[pi-ownership]: ../../../packages/durable/src/harness/scheduler.ts#L1818

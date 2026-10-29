# 评测样本、执行与结果选择

[手册目录](../README.md)

适用范围：Maka 提交 `15d0a0dd19d39e2bc09ae35940e432c636982315` 的 `packages/eval`、Host 执行适配和 continuation replay。
Pi 对照提交为 `482c1381a00d8d00bc81697e4a980c593db6803e`；经核对，本文引用的 Pi 文件相对该提交无本地改动。
本章只静态核对源码与已有测试，没有运行 benchmark、启动评测容器或调用模型。实现轨迹是源码展开，不是本次运行日志。

## 1. 问题：结果选择与结论有效性

假设比较两个 Agent 修改配置的能力：二者都需要新增一个模型，同时保留已有模型。Agent 回答“完成”，并不能证明配置可以重新加载。
需要固定初始文件，运行后检查真实配置，再区分“配置被破坏”和“容器根本没启动”。前者是任务失败，后者没有提供同等质量的能力证据。

另一个陷阱是反复执行到成功，再只保存成功结果。这样测到的是挑选策略，不是预先约定的一次尝试表现。
Maka 为此把实验样本和基础设施补跑分开，并保留追加式 Attempt 历史；结果选择不按最高分，也不按最新时间。[M1]、[M2]、[M3]

| 必须先固定的内容 | 配置任务的具体例子 | 否则会混入什么差异 |
| --- | --- | --- |
| Task 与初始材料 | 已存在的两个模型、待新增模型的字段 | 第二次沿用第一次已修改的文件 |
| Subject | 相同模型下的两个 Agent 配置 | 模型差异被误认作 Agent 实现差异 |
| Budget | 步数、框架超时参数 | 一个条件获得更多探索机会 |
| Verifier | 重新加载并检查新旧模型 | 将最终回答当作产物验收 |
| Repetition | 每个任务、每个 Subject 预定运行三次 | 查看分数后再决定是否补跑 |

这些是实验设计要求，不表示通用配置 schema 已实现适用于任意任务的验收器。具体 reward 仍由所接框架的 verifier 产生。[M4]

## 2. Experiment、Cell、Attempt：三个层级

`ExperimentSpec` 描述 benchmark 版本、executor、subjects、tasks、repetitions、共享 budget、verifier 和并发限制。
`parseExperimentSpec()` 拒绝未知字段、重复身份、非法次数，并冻结解析后的对象；`expandExperiment()` 再展开样本矩阵。[M1]、[M5]

```text
Experiment：两个任务 × 三次 repetition × 两个 Subject = 12 个 Cell
  Cell：config-add::2::maka
    Attempt sequence=1：infra_failed
    Attempt sequence=2：completed，score=0
  Cell：config-add::3::maka
    Attempt sequence=1：尚待执行的另一个样本
```

Cell ID 的构成为 `task.id::repetition::subject.id`，`experimentId` 是另一字段；不应将 Cell ID 视为跨实验全局身份。
`CellAttempt` 保存 `cellId`、递增 `sequence`、起止时间和 `EvalResult`。后者同时保存 score、usage、cost、duration、status、failureReason 与 artifacts。[M1]、[M2]

`runExperiment()` 按 `(task.id, repetition)` 分组：同组的不同 Subject 启动执行，独立任务组受 `maxConcurrentTaskGroups` 限制。
这控制评测并发，不是被测 Agent 的任务依赖图，也不能保证两次远端模型采样得到相同输出。[M3]

`FileAttemptStore.append()` 要求连续序号，先写临时文件并同步，再以硬链接发布最终文件；已有 Attempt 不被覆盖。
`runExclusive()` 使用实验目录中的独占 writer lock。这是文件存储的单写者约束，不应扩展为分布式调度保证。[M6]

## 3. earliest valid：有效优先于成功

`isReplaceableAttempt()` 只把 `infra_failed` 和 `indeterminate` 标为可替换。
`selectCellResult()` 按 sequence 升序寻找首个不可替换项；runner 的 `selectSubjectResult()` 还检查 Subject 可选的 `canReuse()` 条件。[M2]、[M3]

| Attempt 状态 | 代表什么 | 通用选择规则 |
| --- | --- | --- |
| `completed` | 执行与验证已收口 | 可选中；score 为 0 也不会自动补跑 |
| `subject_failed` | 被测对象失败，但仍是有效任务观测 | 可选中，不能为了提高分数跳过 |
| `infra_failed` | 准备、协议、验证等基础设施路径失败 | 不作为最终 Cell 结果，可追加替代尝试 |
| `indeterminate` | 取消或清理等边界尚不能确认收口 | 不作为最终 Cell 结果，保留不确定性 |

例如序列是 `infra_failed → subject_failed(score=0) → completed(score=1)`，通用选择器仍选第二条。
正常 runner 发现第二条可复用后即跳过该 Cell，不会为追求 1 分而执行第三条。若 Subject 设置了额外复用条件，还需同时满足该条件。[M2]、[M3]

注意 `completed` 不是“通过验收”的同义词；通用选择器不检查 score 是否非空，也未为所有 benchmark 规定统一成功阈值。
Harbor／Pier 适配器读取指定 reward；缺失 reward 被视为基础设施问题，而不是自动补记为零分。[M4]

| 容易混淆的“再来一次” | 身份如何变化 | 由谁负责 |
| --- | --- | --- |
| Repetition：计划内重复采样 | 新 repetition，因此新 Cell | `expandExperiment()` |
| Infrastructure retry：替换无效尝试 | 同 Cell，追加新 sequence | 再次运行时由 runner／AttemptStore 处理 |
| Runtime continuation：继续当前工作 | Runtime 内部的执行续接，不新增评测样本 | Runtime Host；Eval 不接管内部续接循环 |

普通 `runExperiment()` 每轮对待执行 Cell 至多追加一个 Attempt，没有“持续重试到成功”的内部循环。
Maka Subject 每次 execute 都生成新的 `executionId` 和专用 Host root；基础设施补跑不是复用旧 Host 根目录的 resume。[M3]、[M7]
“Attempt 可替换”只是评测结果选择规则，不证明外部副作用可以安全重做；补跑仍需可重置的测试环境与明确授权。

## 4. 实现轨迹：Eval 经 Host 执行 Maka

`createMakaSubjectAdapter()` 不直接创建模型循环，而是构造 `RunHostedExecutionInput`，交给随包提供的 shim。
它把模型目标、权限模式、协作／编排模式、工具配置和 maxSteps 交给 Host；任务文本来自 executor 的 `context.taskInput`。
在 Harbor／Pier 路径，该文本来自 relay 的 `ready.instruction`，不应直接将 spec 中的 `task.input` 视为最终发送给模型的原文。[M7]、[M8]

```text
runExperiment → executeCell
  → executor.runAttempt → runHarnessAttempt → startTrial
      → run_trial.py：检查框架版本，构造并运行 Trial
      ← relay ready：instruction、cwd
  → subject.execute（createMakaSubjectAdapter）
      → context.execute(nodePath, [shimPath, payload])
      → harbor-maka-subject.ts → runHostedExecution
          → connectOwnedRuntimeHost → 配置模型目标
          → RPC hosted.execution.start
          → HostHostedExecutionRunner.run
              → session.create → turn.start → turn.query 等待终态
              → 等待执行驻留收口 → usage.query + 结算覆盖检查
          ← HostedExecutionProjection
      ← 有边界的结果帧；adapter 核对 executionId
  → verify() → readVerification：读取 Trial 的 reward 与证据
  → 组合 EvalResult → store.append → 选择 Cell 结果
```

此处存在两个独立判断：Host 判断执行是否收口，verifier 判断任务产物得多少分。
`HostHostedExecutionRunner.run()` 还检查用量是否完整；存在缺失、部分记录或待修复记录时，可以返回 `indeterminate`，不能只凭模型最终答复宣告结算完成。[M9]、[M10]

`executeCell()` 遇到 Subject 的 `infra_failed`／`indeterminate` 会跳过验证；正常完成或 Subject 失败时仍执行 verifier。
取消或清理无法确认时，即使已获得部分结果，也会清除 score 并保留不确定状态。[M3]、[M8]

现有 `maka-initialization.test.ts` 用本地 fake HTTP 服务检查 shim 的 Host 初始化、鉴权和工具调用路径，并检查采集到 `runtime.sqlite`。
它验证接线而不是模型能力；测试名称中的 retries 也不能作为“评测可任意补跑低分”的依据。[M11]

## 5. Harbor 与 Pier：共用适配与环境假设差异

`createHarborExecutor()` 与 `createPierExecutor()` 都进入 `createHarnessExecutor()`；Python 的 `eval_framework` 选择对应模块和安装包。
`run_trial()` 会检查实际 distribution 版本是否等于配置版本，再调用该框架的 Trial。两个分支在本提交中均有实现，不是未来规划。[M8]、[M12]、[M13]

| 边界 | Harbor 分支 | Pier 分支 |
| --- | --- | --- |
| 任务来源 | repository + 完整 Git commit + task path | `tasksRootEnv` 下的任务路径，拒绝路径逃逸 |
| Trial 创建 | 使用 Maka 的 `create_harbor_trial()`，处理缓存与单／多步骤任务 | 调用框架的 `Trial.create()` |
| 采集路径 | 使用 Harbor 对应的 artifacts 子目录 | 使用 Pier 对应目录与日志挂载 |
| `egressProxy` | 有 subject 网络策略适配 | 配置时直接拒绝，不声称提供同等策略 |

Executor 只允许一个 Trial 执行一次 Subject；通过 relay 的 ready／execute／verify 交接，避免 Eval 隐式接管框架环境生命周期。
`readVerification()` 读取 `result.json` 的 `verifier_result.rewards[rewardKey]`，将框架异常、Subject 超时／非零退出和缺失 reward 分别处理。[M4]、[M8]

结果证据与指标也要分开：Maka adapter 保留运行状态及 stdout／stderr artifact 引用；采集清单记录文件大小和哈希。
此 adapter 的成本估算只覆盖代码明确支持的模型，未知时返回 `null`；Subject 自报成本另作证据。不能将 `null` 视为“免费”，也不能只计算最终选中 Attempt 而隐去补跑成本。[M4]、[M7]

## 6. 四种操作：回看、投影 replay、rerun、resume

假设模型请求写入配置，工具已经执行，但进程在保存完整结果前退出。此时打开日志、重建历史、重新测量一次、继续旧任务，是四个不同的操作。

| 操作 | 输入与目的 | 是否重新产生外部效果 |
| --- | --- | --- |
| Trace 回看 | 读取诊断事件，理解当时的阶段与错误 | 回看本身不会 |
| 事件投影 replay | 从已存 RuntimeEvent 生成可用历史／provider items | 投影本身不会执行工具或请求模型 |
| 实际模型 rerun | 从约定初始条件重新运行实验样本 | 会重新请求模型，工具也可能再次产生效果 |
| 执行 resume | 从旧工作的持久边界继续未完成任务 | 可能继续调用；必须先判断未决效果与恢复权限 |

`RunTrace.emit()` 的记录器是诊断接口，记录失败被隔离，不影响模型／工具执行；不能用“有一条 trace”证明工具结果已提交。
同文件还明确把发送诊断与计费区分开。因此 trace 既不是评分器，也不是可直接驱动恢复的命令日志。[M14]

`buildContinuationReplayPlan()` 则接受不可变事件前缀，逐段调用 `buildContinuationReplaySegment()`：

1. 检查工具恢复事实；存在损坏或未收口结果时阻止 replay。
2. 检查投影版本，调用模型历史投影，拒绝不支持或无法配对的内容。
3. 裁去允许裁剪的中断尾部；中间出现工具调用缺口而后面还有可回放内容时阻止继续。
4. 合并各段，筛选与目标 provider／model 兼容的 reasoning 项，并计算边界与投影摘要。[M15]

现有测试把 `user-1 → thinking-1 → assistant-1` 交给投影器，断言中断尾部被裁去，只留下稳定的 `user-1`。
另一个测试在未匹配的工具调用后追加 assistant 内容，得到 `provider_replay_non_suffix_gap`，而不是静默虚构工具结果。[M16]

这些函数产出 replay plan，未执行实验、启动模型或调用工具。实际的 continuation 需由 Runtime 接纳并继续执行；不应将计划中的历史项目视为一套录制响应后重新运行程序的 cassette 执行器。
同理，“能够重建对话”不意味着文件和远端服务状态也已恢复；执行边界可进一步阅读[运行时调用链](../01-runtime/request-to-response.md)。

## 7. 与 Pi 对照：能力评测、可控回归与恢复测试

Pi 默认 `coding-agent/agent/ai` 与实验 durable 是两条路径。下表分别定位，不以默认循环的能力概括整个仓库。

| 问题 | Maka 的对应入口 | Pi 的实际入口与边界 |
| --- | --- | --- |
| 如何展开实验？ | Task × repetition × Subject | `createTaskPlan()` 展开 case、model、runNumber、文档变体，并交替变体顺序 [P1] |
| 是否真的运行 Agent？ | Subject → owned Host execution | `runPiCodingAgent()` 建立独立 workspace／HOME／Session，再 `session.prompt()`；不是 faux [P2] |
| 最小评分例子是什么？ | Executor 读取框架 reward | `smoke.eval.ts` 断言回答为 Paris、无错误且有用量；运行它会调用配置的模型 [P3] |
| 如何固定模型行为？ | 本地 fake HTTP 的初始化测试 | suite 的 `createHarness()` 注册 faux provider，通过 `setResponses()`／`appendResponses()` 控制响应并采集事件 [P4] |
| 如何保存运行证据？ | Attempt artifacts、运行数据库与过程输出 | eval harness 在清理前保存原生 Session JSONL，并汇总消息和工具事件 [P2] |
| 如何验证恢复？ | continuation replay 的边界与缺口测试 | durable 测试关闭并重开 SQLite Harness，检查工具是否重跑、部分输出是否保留 [P5] |

Pi 的 faux harness 使用真实 Agent／AgentSession 接线，但模型响应由测试指定；它适合验证状态机、事件顺序和回归，不提供真实模型的成功率证据。
反之，真实模型评测即使重复得到正确答案，也不足以证明取消、恢复或工具重复执行边界正确。[P2]、[P4]

durable 的恢复测试尤其具体：unsafe 工具在执行意图之后中断，恢复后保留部分输出并报错，不再次运行；
只有持久记录与当前工具注册的 replay 策略都为 safe，测试才期待再次执行。此处的 replay 是工具恢复策略，不能混同为 Maka 的纯事件投影。[P5]

Pi 文档评测 harness 还要求配置容器及非特权执行身份；因此不能笼统断言 Pi 没有沙箱、评测或恢复。
这些机制也不自动构成通用 benchmark 平台或录制式回放系统，结论必须落实到具体入口。[P2]、[P5]

## 8. 源码定位与阅读顺序

| 入口 | 关键函数／案例与用途 |
| --- | --- |
| [M1：实验矩阵][M1]、[M5：schema][M5] | `expandExperiment`、`parseExperimentSpec`：身份、重复次数与配置约束 |
| [M2：结果选择][M2]、[M3：runner][M3] | `isReplaceableAttempt`、`selectCellResult`、`runExperiment`、`selectSubjectResult` |
| [M6：Attempt 存储][M6] | `FileAttemptStore.append/runExclusive`：追加记录与单写者 |
| [M7：Maka Subject][M7]、[shim][M17] | `createMakaSubjectAdapter`、顶层 `runHostedExecution` 调用：输入与结果协议 |
| [M9：Host 客户端][M9]、[M10：Host runner][M10] | `runHostedExecution`、`HostHostedExecutionRunner.run`：执行所有权与结算 |
| [M8：框架执行器][M8]、[M4：验证读取][M4] | `createHarborExecutor/createPierExecutor`、`runHarnessAttempt`、`readVerification` |
| [M12：Python Trial][M12]、[M13：框架选择][M13] | `run_trial/_run_selected_trial`、`framework_spec`：Harbor／Pier 适配 |
| [M14：诊断][M14]、[M15：事件投影][M15] | `RunTrace.emit`、`buildContinuationReplayPlan/buildContinuationReplaySegment` |
| [M11：Host 初始化测试][M11]、[M16：replay 测试][M16] | 本地 fake 服务接线与不可变事件前缀的边界断言 |
| [P1：计划][P1]、[P2：评测 harness][P2]、[P3：案例][P3] | `createTaskPlan`、`runPiCodingAgent`、`describeEval` |
| [P4：faux harness][P4]、[P5：durable 测试][P5] | `createHarness`、`blockingTool` 与关闭／重开测试 |

## 9. 本章术语

| 术语 | 本章含义 |
| --- | --- |
| Subject／Executor／Verifier | 被测对象／准备并运行一次试验的框架适配／检查产物并给分的验证器 |
| Cell／Attempt | 预先定义的样本身份／该样本的某次执行记录 |
| Earliest valid | 最早可采用的观测；不是最高分，也不要求任务成功 |
| Fixture／Artifact | 运行前的固定材料／运行后留下的结果或证据 |
| Continuation／Resume | 运行时续接／从持久边界恢复旧工作；都不等于新增独立样本 |

## 10. 源码练习

以下练习只需读源码与手算，不运行 benchmark、模型或容器，也不修改 Maka 源码。

1. 为两个任务、两个 Subject、两次 repetition 列出八个 Cell。为其中一个 Cell 设定 `infra_failed → completed(0) → completed(1)` 序列，按通用选择器手算结果；再解释为何实际 runner 通常不会产生第三条。
2. 在 Host projection 返回后、verifier 结果读取前假设取消。沿 `executeCell()` 和 `runHarnessAttempt()` 定位 score 变为 `null` 的分支，说明为什么不能沿用已有分数。
3. 对比 M16 的尾部裁剪与 P5 的 safe／unsafe 工具恢复测试：哪些只是生成历史，哪些可能重做副作用？再说明将 faux 响应替换为真实模型后，哪些断言仍可靠、哪些不再测量同一件事。

## 相关专题

- [计量与诊断](observability.md)：理解结果中的用量及覆盖限制。
- [执行恢复](../04-state/durable-execution.md)：区分追加评测尝试与恢复已有执行。
- [历史投影](../02-context/history-and-projection.md)：理解纯 replay 为什么不重新执行工具。

[M1]: ../../../../maka/packages/eval/src/experiment.ts#L84
[M2]: ../../../../maka/packages/eval/src/result.ts#L87
[M3]: ../../../../maka/packages/eval/src/runner.ts#L138
[M4]: ../../../../maka/packages/eval/src/harness-executor.ts#L716
[M5]: ../../../../maka/packages/eval/src/spec.ts#L22
[M6]: ../../../../maka/packages/eval/src/attempt-store.ts#L26
[M7]: ../../../../maka/packages/eval/src/maka-subject.ts#L36
[M8]: ../../../../maka/packages/eval/src/harness-executor.ts#L110
[M9]: ../../../../maka/packages/runtime-host/src/client/hosted-execution.ts#L62
[M10]: ../../../../maka/packages/runtime-host/src/server/hosted-execution-runner.ts#L52
[M11]: ../../../../maka/packages/eval/src/__tests__/maka-initialization.test.ts#L34
[M12]: ../../../../maka/packages/eval/harbor/run_trial.py#L119
[M13]: ../../../../maka/packages/eval/harbor/eval_framework.py#L38
[M14]: ../../../../maka/packages/runtime/src/run-trace.ts#L107
[M15]: ../../../../maka/packages/runtime/src/continuation-replay.ts#L91
[M16]: ../../../../maka/packages/runtime/src/__tests__/continuation-replay.test.ts#L53
[M17]: ../../../../maka/packages/eval/src/harbor-maka-subject.ts#L69
[P1]: ../../../packages/evals/src/plan.ts#L39
[P2]: ../../../packages/evals/src/harness.ts#L272
[P3]: ../../../packages/evals/evals/smoke.eval.ts#L7
[P4]: ../../../packages/coding-agent/test/suite/harness.ts#L150
[P5]: ../../../packages/durable/test/harness-tools-recovery.test.ts#L109

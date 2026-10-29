# Maka Agent 基础设施源码分析与 Pi 架构对比

本文档集覆盖 Maka 的架构与领域模型、端到端执行链路、模型循环、上下文管理、工具执行、故障恢复及任务协调。各专题包含执行场景、关键数据结构、失败分支和源码引用。

架构差异概述见[Agent 内核架构对比](07-comparison/overview.md)，重点包括记忆模型、子任务组织和执行恢复约束；记忆与子智能体机制分别设置专题。实现入口见[端到端执行链路](01-runtime/request-to-response.md)，术语定义见[领域词汇表](GLOSSARY.md)，对象与字段映射见[领域模型](00-foundations/domain-model.md)。目录按技术主题组织，各章节不具有强制阅读顺序。

## 总体架构与核心概念

- [整体架构：Maka 执行系统的组成与分层](00-foundations/architecture.md)
- [仓库地图：workspace、依赖与接入面](00-foundations/repository-map.md)
- [领域模型：Session、Turn、Run、Invocation 与事件](00-foundations/domain-model.md)
- [设计模式与权衡](00-foundations/design-tradeoffs.md)

## 运行时执行链路

- [从输入到回答：完整调用链](01-runtime/request-to-response.md)
- [会话与执行对象的生命周期](01-runtime/session-lifecycle.md)
- [模型循环：工具调用下的多轮请求](01-runtime/model-step.md)
- [模型、连接与配置的注入](01-runtime/provider-configuration.md)

## 上下文管理

- [历史事实与模型输入投影](02-context/history-and-projection.md)
- [历史压缩：从安全前缀到可验证的下一次请求](02-context/compaction.md)

## 工具、权限与环境

- [工具执行内核：从声明到可提交结果](03-tools/tool-runtime.md)
- [MCP 连接与客户端能力](03-tools/mcp-and-client-capabilities.md)
- [执行环境与操作系统沙箱](03-tools/sandbox.md)
- [分层授权与凭据生命周期](03-tools/permissions-and-secrets.md)

## 状态与恢复

- [持久化与恢复：崩溃后可信任的事实](04-state/durable-execution.md)
- [Host 所有权与客户端状态同步](04-state/host-ownership-and-sync.md)

## 任务协调

- [输入队列与执行准入](05-coordination/queues-and-admission.md)
- [任务图与多 Agent 协作](05-coordination/agent-graph.md)

## 诊断与验证

- [重试、超时与幂等](06-operations/retry-timeout-idempotency.md)
- [请求计量、上下文诊断与故障排查](06-operations/observability.md)
- [评测样本、执行与结果选择](06-operations/evaluation.md)

## 架构与机制对比

- [Agent 内核架构对比](07-comparison/overview.md)：分析记忆模型、子任务组织、执行证据与架构取舍。
- [记忆存储、提取与检索机制对比](07-comparison/memory-design.md)：比较历史记录、工作上下文和跨会话知识的管理方式。
- [子智能体执行、调度与生命周期管理对比](07-comparison/subagent-design.md)：比较工具委派、独立会话与持久任务模型。
- [应用集成与执行场景对比](07-comparison/pi-and-maka.md)：补充服务入口和应用装配分析。

## 源码依据与阅读约定

原专题的 Maka 源码阅读基线为 `15d0a0dd19d39e2bc09ae35940e432c636982315`，本地仓库为 `D:\ai\maka`。2026-10-28 新核对的总体设计、记忆和子智能体三篇，基于已更新的本地 Maka `8320cda1a4229b54124efe91d8830d0a89c39a4f`。Pi 基线为 `482c1381a00d8d00bc81697e4a980c593db6803e`，结合本地工作区源码，保留原有未提交修改。具体源码范围以各篇文首为准。

源码链接指向 GitHub 上各专题对应的固定提交，并保留行号定位。部分 Pi 分析依据包含未提交修改的本地工作区；这些修改无法在 GitHub 上展示，链接指向基线提交中对应的已提交代码，可能与分析内容存在差异。

图示分别标注包依赖、对象组合、调用关系或数据流。概念示例与源码摘录明确区分；测试说明基于实现与断言的静态阅读，不表示已经执行模型调用或平台测试。能力范围以实际装配、平台支持和错误处理分支为准。

各篇相关专题提供非强制导航。Maka 与 Pi 的比较分别标明 Pi 默认 coding-agent、通用 agent 和独立 durable 的适用范围。

相关资料：[Pi 会话与循环的关系](../coding-agent-agent-relationship.md)、[原 Agent Infra 教程](../agent-infra-v2/README.md)、[Maka 总体架构](https://github.com/apache/maka/blob/15d0a0dd19d39e2bc09ae35940e432c636982315/ARCHITECTURE.md)。

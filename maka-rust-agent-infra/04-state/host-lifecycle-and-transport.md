# 宿主生命周期、安全传输与后台健康探针

## 1. 为什么客户端与服务端通信层和“后台僵尸命令监控”必须内建进微内核？

当核心引擎拥有了纯正的 Reconciler、强韧的 Event-Log 和沙箱安全的工具执行力后，整个平台还有最后一段基础设施必须啃下来：**宿主进程（Host）本身如何对外提供多客户端受鉴权长连接接口、如何在宿主停机热切换时优雅倒计时清场，以及如何对异步丢在后台跑的长寿命服务（如 Dev Server、数据库容器、隧道进程）提供独立真实的可用性监控？**

如果在原型系统里随便起一个裸 HTTP/WebSocket Server 或把后台进程丢到子线程自生自灭，很快就会掉进三个现实陷阱：
1. **无主控跨端信令混乱**：桌面 GUI 窗口、终端 TUI、`maka-eval` 容器客户端甚至远端代理都在连同一个 Host；不同连接方谁该读、谁该写、谁拿什么作用域权限？
2. **后台进程假活（Silent Hang / Port Zombie）**：Agent 用 Shell 异步起了一个后台测试服务器（`npm run dev`），命令本身没退出，但服务其实已经卡死在启动半路根本没监听 3000 端口——如果没有独立探活器，模型往往陷入无休止的盲等待或盲目重试。
3. **健康探针成为 SSRF 攻击隧道**：允许模型提供任意探测 URL 检查后台进程状态时，万一 URL 里夹带 `http://user:pass@internal/` 用户凭据碎片、或者利用超长恶意畸形片段穿透解析器，反而给系统开出了内网扫描后门。

Maka Rust 微内核分别利用 `platform/transport`、`platform/host-runtime/src/transport` 与 `platform/background-health` 构建了贯穿宿主对外总线和后台探活的防御型基础设施。

---

## 2. 宿主安全传输契约：`platform/transport` 与分权对端矩阵

查看 `platform/transport/src/transport.rs` 的对外统一契约导出：
> **Authenticated Host session and channel contracts.**
> 导出核心原语：`Call`, `Catalogue`, `Description`, `Endpoint`, `Endpoints`, `Grant`, `NativeGrant`, `Peer`, `PeerPolicy`, `Peers`, `Reply`, `Selection`, `Server`, `Source`

`platform/transport` 复用并发安全底座 `maka-host-runtime::application::transport`，将任意接入 Host 的本地或跨进程客户端抽象为严格鉴权的通信端点网络：

```mermaid
flowchart LR
    Clients["接入对端 (Peers)<br/>Desktop / Terminal / CLI / Eval HostClient"] -->|"stdio 管道 / 受认证本地总线"| PolicyGate{"PeerPolicy & Grant 门禁<br/>核对作用域与调用者身份"}
    PolicyGate -->|"NativeGrant / 受控 Grant<br/>(非 Bearer 离线通行证)"| Router["Host Server 路由与 Catalogue 服务目录"]
    Router -->|"派发 Call 请求"| Services["NativeServices & Host Plugins<br/>(Agent / Settings / Checkpoints)"]
    Router -->|"Subscription 流式事件回推"| Clients
```

这套传输架构设计有四项务实保障：
1. **多传输底座同构（`stdio` 与结构化双工通道）**：`platform/transport/src/stdio.rs` 为本地嵌套拉起（如 `maka host serve`、评测容器内控制端直连、ACP/CLI 父子管道）提供了零网络暴露端口的标准化帧读写实现，既规避了本机回环随机端口抢占与安全扫描误报，又与通用传输语义保持完全同构。
2. **基于 `PeerPolicy` 的显式对端隔离**：每个连进来的 Peer 并非“一经连上便拥有 Root 超级上帝权限”，而是结合其所属启动上下文装配专属 `PeerPolicy` 与 `Grant` / `NativeGrant`——例如在托管隔离场景下自动收紧管理类方法目录（`Catalogue` 筛选过滤），让低信任边界的对端根本“既看不到、也调不到”核心控制面接口。
3. **响应流化与资源限流保护（`Reply` & `RESOURCE_CHUNK_LIMIT`）**：在 `host-runtime` 的请求处理环中，大文件附件上传或超大二进制返回值均经由 `resource_ingress` 严格按分片契约拆包，并受活跃资源票据 `Ticket` 全程监护，防止单个慢速客户端卡死整体响应分发器。

---

## 3. 宿主启停、热重载交接与优雅退场序列

一个常驻后台或被命令行按需唤起的 Host 进程，内部往往同时承载着：多组运行中的 Agent Session、正在进行中的 `writeback` 单写者批次、开启的 MCP 服务器管道、V8 插件实例以及桌面原生工作区挂接。

`platform/host-runtime` 结合 `maka-event-log::root_store::lifecycle` 为宿主打造了严谨的“开门 - 运行 - 关门清场”全周期编排：

1. **启动期严格有序激活**：
   - 先打开并原子升级单一 `RootStore`（执行所有未落地的 `SAVEPOINT` 内 schema/data 迁移）；
   - 由 `CompositionPlanner` 校验包清单签名（`maka-packages` 验证归档完整性与入口契约）并并行激活底层 Host 插件；
   - 仅当所有必备服务注册进入 `Catalogue` 后，才打开 `Gates` 允许外部 `Peer` 接入接单。
2. **热重载（Hot Reload）与停机收拢期（Graceful Retirement）**：
   - 触发关停或插件热更时，`gates` 立即切入关闭态（`gates.is_closing()`），任何新到来的非核心延命游离请求立刻快速拒绝；
   - 已获得活跃准入的调用依靠 `Pins` / `RetainedWorkSettlement` 持有执行窗口，同时向所有活跃外部/内部实体通知收尾清理；
   - 对正在运行中的插件实例调用 **`prepareReload()`** 完成状态封存、等待相关物理 `OperationCleanup::Confirmed` 回执及 `writeback` FIFO 尾部全部彻底落盘 `COMMIT`；
   - 若涉及外部长执行器交接，必须确保落足 **`Host::stop_execution` 的 5 秒清理确认注册期** 后再释放底层 SQL 主控权。

---

## 4. 独立进程观测与 HTTP 健康探测：`maka-background-health`

在软件开发全流程中，Agent 经常使用 `Shell` 工具在后台启动本地开发服务器、编译监视器（File Watcher）或容器隧道（Daemon）。如果 Agent 想知道“刚才挂进后台那个服务器现在到底起好了没，有没有吐出致命编译报错，端口通没通”，让 Agent 自己每隔 2 秒瞎写一段 `sleep 2 && curl ...` 命令去轮询，既极其浪费模型推理步数，又容易乱写 Shell 脚本捅出各种奇奇怪怪的解析故障。

Maka 专门独立抽出了一个精悍专注的平台组件 **`platform/background-health`**（工具化探活接口），把“查看进程生命线”与“探测服务网络就绪态”严丝合缝地统一起来：

```mermaid
flowchart TD
    In["工具入参 Input<br/>ref: maka://runtime/background-tasks/&lt;entity_id&gt;<br/>include_logs: bool, url: Option(String)"] --> Val{"Input::validate() 严格正则与结构审查"}
    Val -->|"前缀不匹配或 ID 非法"| Rej1["立即拒绝: 仅接收 Shell 签发的后台任务引用"]
    Val -->|"URL 超 8192 字节 / 非 http(s)<br/>含用户名密码 / 含 #fragment"| Rej2["立即拒绝: 切断 SSRF 与凭据注入路径"]
    Val -->|"审查全部通过"| Split["并行双重真值采检"]
    Split -->|"1. 查宿主本地进程账本"| Proc["输出 Process 结构体<br/>status(ShellStatus) / tracked / exit_code / failure_message / logs"]
    Split -->|"2. 按需独立发起受限 HTTP(S) 探针"| Endp["输出 Endpoint 三态结论<br/>NotChecked / Unknown(error) / Checked(http_status, elapsed_ms, Readiness)"]
```

观察 `platform/background-health/src/health.rs` 可以提炼出三项极精致的系统防呆哲学：

1. **受限引用源锁定（Prefix Lockdown）**：`Input::validate()` 强验证输入引用 `ref` 必须以 **`maka://runtime/background-tasks/`** 开头且跟随着通过 `interaction::entity_id` 合法性验证的标识符——**Agent 根本无法借该工具去胡乱偷窥宿主机上任何其它非受控进程 PID**。
2. **极其洁癖的 HTTP URL 安检**：即使传入可选的健康检查 `url`，代码瞬间叠加了七重封锁条件：
   - 总长度严格限制 **`<= 8192` 字节**；
   - 必须使用明确可解析的 `http` 或 `https` 协议且具备有效 `host_str`；
   - **绝对禁止 URL 内嵌 `username()`、`password()` 或任何 `#fragment` 片段**；
   - 并经过宿主受限网络策略和标准 `maka_capabilities::http` 授权管道真实发起探测。
3. **明确的未知态建模（`Endpoint::Unknown` vs `Checked`）**：系统严禁在连接超时或 TLS 失败时用模糊的 `http_status: 0` 去蒙混大模型，而是明确返出 **`Endpoint::Unknown { target, error }`（网络不通/探针未拿到有效协议应答）** 与 **`Checked { target, http_status, elapsed_ms, health: Readiness }`**；搭配本地 `Process` 的 `tracked` 布尔标（明确告诉模型这个进程此刻究竟还在受宿主实时监护还是已经脱管丢失）与 `exit_code`，模型只需一次调用就能看清“进程活着、端口没通、日志卡在连接数据库超时”的全景诊断！

---

## 5. 总结

至此，`04-state` 持久化与宿主基座三部曲全部收束为一个稳固可靠的运行地基：
- **`maka-event-log` & `writeback`**：保障所有核心事实与两阶段副作用单线强一致提交与热恢复；
- **`checkpoints`**：用硬链接晋升和 `checkout_exact` 为代码与对话提供零污染双线时空回溯；
- **`transport` & `background-health`**：把对外对端控制权限锁进 `PeerPolicy`/`Pins`，把混乱的后台运维盲猜转化为标准化进程快照加防 SSRF 探活。

当一台引擎底座已经如此坚若磐石时，我们就可以放心让成群结队的 Agent 真正组织起来协同作战了！接下来的 **05 多智能体协同、目标控制与主动洞察篇**，先来看看 Maka Rust 版如何用 **Symmetric Sessions（对称会话原语）** 与 **Rooms（结构化协同房间）** 取代脆弱的单向 Subagent 调用，把复杂的协作网络变得简单、可审计且绝对对称——[Rooms 多智能体协同与对称会话工具](../05-coordination/rooms-and-symmetric-sessions.md)。

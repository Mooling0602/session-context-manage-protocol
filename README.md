# session-context-manage-protocol

**SCMP** · Session Context Manage Protocol for LLMs

> 面向 LLM Coding Agent 的**跨会话协作协议**：让多个保有独立上下文的会话彼此发现、派发任务、等待回复、发送通知并检查上下文。
>
> 状态：**v0.2-draft.1（中文草案）** · 远程绑定（v0.2）已出草案 · 英文 normative 版本规划中

## 一句话定位

> MCP 管「单个会话内的 LLM ↔ 工具」；**SCMP 管「会话 ↔ 会话」**。

## 工具一览

| 规范名 | 语义 |
|---|---|
| `dispatch` | 异步派发：立即返回，结果**自动回送**（无需轮询） |
| `wait` | 同步阻塞：等到回复（或超时）才返回 |
| `notify` | 单向通知：不期望回复 |
| `check` | 检查目标会话状态与最近上下文（兜底路径） |
| `list_sessions` | 列出可协作会话（ID + 标题） |
| `get_self_metadata` | 当前会话身份元数据（身份确认/鉴权） |

宿主可为规范名加自身生态前缀（如 `agent_bridge_dispatch`）；与既有工具冲突时必须加前缀。

## 文档索引

| 文档 | 内容 |
|---|---|
| [docs/00-overview.md](docs/00-overview.md) | 概述：动机、非目标、架构分层、与 MCP/A2A/ACP 的关系 |
| [docs/10-tools.md](docs/10-tools.md) | L1 · 六个工具的规范定义（含防轮询必须文案） |
| [docs/20-data-model.md](docs/20-data-model.md) | L2 · 数据模型：信封、派发状态机、错误码、环检测、TTL |
| [docs/30-local-binding.md](docs/30-local-binding.md) | L3a · 本地绑定：文件注册表、原子写、inbox 投递 |
| [docs/40-remote-binding.md](docs/40-remote-binding.md) | L3b · 远程绑定：网关路由域、WS 帧协议、鉴权、多归属、远程 check |
| [docs/35-host-adaptation.md](docs/35-host-adaptation.md) | 宿主适配 · 跨工具内部结构 → 协议概念的规范化映射与降级 |
| [docs/90-conformance.md](docs/90-conformance.md) | 一致性分级（C1/C2/C3）与 MUST 检查表 |
| [versions.md](versions.md) | 版本化策略与变更日志 |
| [schema/](schema/) | 机器可读 JSON Schema（工具定义可直接被宿主消费） |
| [docs/research-acp-a2a.md](docs/research-acp-a2a.md) | 前期 ACP/A2A 调研存档（结论已吸收进规范） |

## 核心设计决策

1. **语义层 A2A 对齐**：Task 状态机、dispatchId 幂等、Message/Part 模型、错误码风格借鉴 A2A；传输层 v0.1 自研本地绑定（文件注册表 + inbox），未来可加 HTTP binding 与标准 A2A 互通。
2. **结果自动回送为一等机制**：实测表明模型会无意义轮询，协议把「结果主动送达 + 工具描述防轮询文案」定为规范性要求。
3. **协议级环检测**：信封携带 `callChain`，循环等待在投递前即被拒绝（40803），杜绝 A-wait-B / B-wait-A 死锁。
4. **显式关联取代猜测**：correlationId / dispatchId 取代原实现「水位 + 文本探针 + 50 条滑窗」的回复识别。
5. **宿主适配规范化**：协议面只见 SCMP 概念——SCMP 会话 ID（`runtime@原生ID`）、统一角色枚举、统一注入格式；各家内部结构差异按 [35 分册](docs/35-host-adaptation.md)降级映射，原生 ID 不泄漏。
6. **远程绑定 = 路由域模型**：网关纯路由（零语义状态、零消息暂存，在线快照型 presence），Host 多归属实现跨域可达；完整地址 `gatewayId:clientId@scmpId`；离线投递 NACK + presence 订阅；静态 token 鉴权。见 [40 分册](docs/40-remote-binding.md)。

## 路线图

- **v0.1**：L1 工具 + L2 数据模型 + 本地绑定 + JSON Schema ✅
- **v0.2（草案中）**：远程连接协议（网关路由域、WS 帧协议、鉴权、多归属）
- **v0.3**：网关联邦/集群（候选）
- **v1.0**：语义冻结 + 英文 normative 版本 + TCK 兼容性测试套件

## 来源与致谢

SCMP 的直接前身是 [opencode-agent-bridge](https://github.com/Mooling0602/opencode-agent-bridge)（npm 插件，六工具模型已在真实模型上验证）；协议化过程中的 ACP/A2A 调研见 [docs/research-acp-a2a.md](docs/research-acp-a2a.md)。

## License

MIT

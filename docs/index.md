# SCMP · Session Context Manage Protocol

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

## 阅读路径

| 分册 | 内容 |
|---|---|
| [概述](00-overview.md) | 动机、非目标、架构分层、与 MCP/A2A/ACP 的关系 |
| [L1 · 工具接口](10-tools.md) | 六个工具的规范定义（含防轮询必须文案） |
| [L2 · 数据模型](20-data-model.md) | 信封、派发状态机、错误码、环检测、TTL |
| [L3a · 本地绑定](30-local-binding.md) | 文件注册表、原子写、inbox 投递 |
| [L3b · 远程绑定](40-remote-binding.md) | 网关路由域、WS 帧协议、鉴权、多归属、远程 check |
| [宿主适配](35-host-adaptation.md) | 跨工具内部结构 → 协议概念的规范化映射与降级 |
| [一致性](90-conformance.md) | 一致性分级（C1/C2/C3）与 MUST 检查表 |

## 机器可读定义

JSON Schema 见仓库 [`schema/` 目录](https://github.com/Mooling0602/session-context-manage-protocol/tree/main/schema)（工具定义可直接被宿主消费）；版本化策略与变更日志见 [`versions.md`](https://github.com/Mooling0602/session-context-manage-protocol/blob/main/versions.md)。

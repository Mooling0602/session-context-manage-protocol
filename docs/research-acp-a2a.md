# ACP / A2A 协议研究：是否值得复用到 agent-bridge-protocol

> **迁入说明**：本调研完成于 SCMP 前身「agent-bridge-protocol」时期（2026-08-18），其结论已被
> SCMP v0.1 直接继承——文中「agent-bridge-protocol」即指本协议（SCMP）。调研原文未作改动，
> 仅作为设计依据存档。

> 状态：调研结论（v0.1） · 日期：2026-08-18
> 范围：评估 ACP（Agent Communication Protocol）与 A2A（Agent2Agent）两个外部协议，
> 判断其语义模型和传输设计是否值得我们（agent-bridge-protocol，原 opencode-agent-bridge
> 的协议化演进）直接复用、部分复用或完全排除。
> 约束：本文档只做设计与讨论，不涉及任何源代码或打包配置。

---

## 1. 结论速览（TL;DR）

| 协议 | 发起方 / 时间 | 现状 | 判定 |
|---|---|---|---|
| **ACP** | IBM / BeeAI，2025-03 | 独立仓库已归档（2026-08），项目并入 Linux Foundation，与 A2A 同治理框架 | ❌ 不采用（已停止独立演进） |
| **A2A** | Google，2025-04 | Linux Foundation 治理，活跃演进 | ✅ 复用其**语义层**（Task 生命周期、数据模型、幂等/错误契约），**不搬**其 HTTP 传输层 |

**一句话决策**：`agent-bridge-protocol` v1 定义为「与 A2A 语义对齐（A2A-aligned）的
自定义本地传输绑定」——数据模型与方法名向 A2A 规范看齐，传输层自研（in-process 总线 +
文件注册表）。未来需要对外互通时，追加一个 HTTP 绑定即可直接讲标准 A2A，无需重设计。

---

## 2. ACP：Agent Communication Protocol

### 2.1 定位

- 发起：IBM，2025 年 3 月，与 BeeAI 平台（agent 生命周期管理：安装/运行/注册/分享）协同开发。
- 目标：定义 **agent 与 agent、应用、人类之间**的通信协议。
- 官方站点仍在维护文档：<https://agentcommunicationprotocol.dev>

### 2.2 核心设计

- **REST 风格通信**：轻量、免运行时，便于系统集成与规模化调用。
- **离线 Agent 发现**：agent 在构建时打包 manifest（能力/skills/接口声明），不依赖运行时在线发现。
- **MIME 类型消息结构**：消息类型通过 MIME 扩展，强调"可扩展、灵活"，而非预定义封闭类型集。
- **有状态长会话**：支持"分布式会话"（distributed sessions），可挂起/恢复跨进程对话。
- **可组合**：官方文档明确 ACP 与 MCP 分层协作——MCP 管"LLM ↔ 工具"（单 agent 内部），
  ACP 管"agent ↔ agent"（多 agent 之间）。

### 2.3 现状评估（关键）

- `i-am-bee/acp` 仓库 **已归档（archived: true，2026-08）**，约 1k stars。
- 官方公告确认 ACP「加入 A2A，共同置于 Linux Foundation 治理之下」
  （<https://github.com/orgs/i-am-bee/discussions/5>）。
- 结论：ACP 作为独立标准已停止演进，**直接基于 ACP 构建有沉没成本风险**。
  其可取思想（离线 manifest、MIME 扩展消息）在 A2A 的 Part 机制与后续扩展体系中已有对应。

---

## 3. A2A：Agent2Agent Protocol

### 3.1 定位

- 发起：Google，2025 年 4 月；现由 Linux Foundation 治理，活跃演进，多语言 SDK（Python/Go/Java/JS/.NET/Rust）。
- 目标：agent 应用的互操作标准，覆盖 **任务执行、流式更新、推送通知、发现、安全、多轮对话**。
- 官方规范：<https://a2a-protocol.org/latest/specification/>

### 3.2 核心机制

**核心操作（Core Operations）：**

| 操作 | 说明 |
|---|---|
| `send_message` | 向 agent 发送消息，创建/延续一个 Task |
| `send_streaming_message` | 同上，但通过 SSE 流式返回 TaskStatus/TaskArtifact 事件 |
| `get_task` / `list_tasks` | 查询任务状态与历史 |
| `cancel_task` | 取消任务 |
| `task/subscribe` + push notification | 订阅任务更新 / 配置 Webhook 推送 |
| `get_agent_card` | 获取 Agent Card（能力发现，`/agent-card` 熟知端点） |

**数据模型（Protocol Data Model）：**

- `Task` → `TaskState`：`submitted → working → input-required → completed / failed / canceled`
- `Message` → `Role`（user/agent）→ `Part`（类型化内容块：text / file / image / function-call 等）
- `Artifact`：任务产出物（文件、结果等）
- `ContextId` / `TaskId`：多轮会话与任务关联的标识语义

**事件与通知：**

- 流式：`TaskStatusUpdateEvent`、`TaskArtifactUpdateEvent`（SSE）
- 推送：`PushNotificationConfig` + `AuthenticationInfo`，Webhook 推送任务更新

**发现与安全：**

- 发现：`AgentCard`（能力/技能/接口声明，含 `AgentCapabilities`、`AgentSkill`、`AgentInterface`）
- 安全：`SecurityScheme` 体系——API Key / HTTP Auth / **OAuth2** / OIDC / **mTLS**

**多租户 / 扩展：**

- 多租户（multi-tenancy）与扩展机制（extensions），并支持自定义协议绑定。

### 3.3 第 5 章：Custom Protocol Binding（对我们最关键的一章）

A2A 规范明确支持**把同一套语义模型绑定到非 HTTP 传输**：

- 5.1 功能等价性要求（不同绑定下语义必须等价）
- 5.2 协议选择与协商
- 5.3 方法映射参考、5.4 错误码映射
- 5.8 自定义绑定标识

> 这等于官方为「语义对齐 + 自定义传输」开了正门：我们可以声明
> `agent-bridge-protocol v1` 是一个 A2A-aligned binding，方法名与数据模型与 A2A
> 规范对齐，而传输层用本地文件注册表 / 进程内总线。将来需要与外部 A2A 端点互通时，
> 增加一个 HTTP binding 即可。

---

## 4. 横向对比

| 维度 | ACP | A2A | agent-bridge-protocol 需要的 |
|---|---|---|---|
| 传输 | REST/HTTP | HTTP + JSON-RPC 2.0 + SSE | 本地：in-process + 文件注册表（v1 无 HTTP 服务端） |
| 任务模型 | 会话式，无显式 Task 状态机 | **Task 状态机** + taskId + 幂等 | 派发/等待/通知/检查 → 正对应 Task 生命周期 |
| 消息结构 | MIME 扩展 | Message/Part（类型化内容块） | 类型化负载（任务描述、结果、产物） |
| 发现 | 离线 manifest 打包 | Agent Card（在线端点） | 工作区文件系统列举（无需 HTTP 卡片） |
| 通知 | 未强定义 | **推送订阅 + 流式事件** | 完成通知（notify） |
| 安全 | 未强定义 | OAuth2 / mTLS / API Key | 工作区信任域（本地场景，无需 OAuth） |
| 治理状态 | ⚠️ 已归档、并入 A2A | ✅ Linux Foundation 活跃 | — |

---

## 5. 与 agent-bridge-protocol 的需求逐项对照

原始 `opencode-agent-bridge` 的 6 个工具 → A2A 的对应物：

| 我们的需求 | A2A 对应物 | 判定 |
|---|---|---|
| `dispatch`（异步派发） | `send_message` 创建 Task | ✅ 直接对应 |
| `wait`（同步阻塞等待） | task 状态订阅/查询 | ✅ |
| `notify`（完成通知） | Push Notification / Subscribe | ✅ 概念一致 |
| `check`（读取结果） | `get_task` + Artifacts | ✅ |
| `sessions`（列举会话） | Agent Card 发现 | ⚠️ 我们改造为工作区文件系统列举 |
| `get_self_metadata` | Task/Context 元数据 | ✅ 概念一致 |
| 回复关联（correlationToken） | `taskId` + 幂等键 | ✅ 正是我们想要的，替代原版"猜回复" |
| 环检测 / 死锁防护（call-chain） | 无对应 | ❌ 保留原创（协议级能力，A2A 未覆盖） |
| 工作区信任域 | OAuth/mTLS | ❌ 保留原创（本地信任域更合适） |
| 传输抽象（总线/文件/net） | HTTP 绑定体系 | ✅ 借鉴其"绑定"方法论，传输自定义 |

---

## 6. 最终建议：复用什么、原创什么、放弃什么

### 6.1 采用（直接借鉴 A2A 语义层）

1. **Task 生命周期状态机**：`submitted → working → input-required → completed / failed / canceled`
   ——消化原版 bridge 所有状态的正确抽象。
2. **taskId + 幂等**：客户端生成 id、重复提交去重——彻底取代原版“最近 50 条消息”滑窗猜测。
3. **Message/Part/Artifact 数据模型**：类型化内容块（文本/文件/结果），为未来跨运行时
   传图片、文件、函数结果留好形状。
4. **错误码契约**：JSON-RPC 风格错误码与超时/semantics（幂等、异步处理、能力校验）。
5. **Push 通知模式**：订阅 + 完成通知 → 对应我们的 `notify`。

### 6.2 保留原创（A2A 不覆盖或不适配本地场景）

1. **传输层**：v1 自研——in-process 事件总线 + 文件注册表（原子写 + generation 计数防竞争），
   网络传输（TCP/UDS）留接口不进 v1。
2. **地址模型**：`agent-bridge://<runtime>/<workspace>/<session-id>`（本地寻址，
   A2A 的 URL/端点模型不适合会话级寻址）。
3. **环检测**：`call-chain` 头追溯调用链，直接拒绝循环 wait（A2A 无此概念）。
4. **信任域**：按 workspace/运行时的本地信任边界，而非 OAuth/mTLS。
5. **TTL 与清理**：dispatch 记录过期清理（原版 7 天 TTL 保留并协议化）。

### 6.3 放弃

- **ACP**：整体排除（已归档、并入 A2A）；仅当未来需要 MIME 式消息扩展时参考其思路，
  A2A 的 Part + Extensions 体系已覆盖。
- **A2A 的 HTTP 传输要求**（v1）：不建 HTTP 服务端、不用 SSE、不实现 OAuth/mTLS——
  全部留给未来可选的 HTTP binding。

### 6.4 落地形态

```
agent-bridge-protocol v1
├── 语义层（A2A-aligned）   Task 状态机 / taskId 幂等 / Message-Part / 错误码
├── 绑定 v1：local          in-process 总线 + 文件注册表（原子写 + generation）
├── 绑定 v2（未来）：http    A2A 标准 HTTP+JSON-RPC+SSE（对外互通）
├── 扩展：环检测 / TTL / 信任域
└── 适配器：opencode（原插件）、dsh（cordis 插件）、未来更多运行时
```

---

## 7. 参考来源

- ACP 官档：<https://agentcommunicationprotocol.dev/about/mcp-and-a2a>
- ACP 仓库（已归档）：<https://github.com/i-am-bee/acp>
- ACP 加入 A2A / Linux Foundation 公告：<https://github.com/orgs/i-am-bee/discussions/5>
- A2A 官方规范：<https://a2a-protocol.org/latest/specification/>
- A2A 概览（Atlan 解读）：<https://atlan.com/know/google-a2a-protocol/>
- 原版迁移来源：<https://github.com/Mooling0602/opencode-agent-bridge>
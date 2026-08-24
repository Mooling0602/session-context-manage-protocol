# SCMP · 概述（Session Context Manage Protocol）

> **状态**：v0.2-draft.1（中文草案）
> **规范语言**：本文档族使用 RFC 2119 风格关键词：**必须（MUST）**、**不得（MUST NOT）**、**应当（SHOULD）**、**不应当（SHOULD NOT）**、**可以（MAY）**。
> **协议版本标识**：`protocolVersion: "0.2"`

## 1. SCMP 是什么

SCMP（Session Context Manage Protocol）是一个面向 LLM Coding Agent 的**跨会话协作协议**。它让运行在同一（或未来：不同）宿主中的多个 LLM 会话，能够以统一的工具接口彼此发现、派发任务、等待回复、发送通知并检查上下文——每个会话保有完整独立的上下文与记忆，协作通过显式消息完成。

一句话定位：

> **MCP 管「单个会话内的 LLM ↔ 工具」；SCMP 管「会话 ↔ 会话」。**

## 2. 背景与动机

主流 Coding Agent 的 Subagent 机制存在若干不足，SCMP 针对性地回应：

| Subagent 模式的局限 | SCMP 的回应 |
|---|---|
| 子代理由父会话临时拉起，上下文从零开始，无法沉淀 | 协作对象是**长生命周期会话**，各有独立记忆 |
| 父会话同步阻塞等待子代理返回 | `dispatch` 异步派发，结果自动回送，父会话继续工作 |
| 子代理之间不能互相通信，只能经父代理中转 | 任意会话间可直接 `dispatch`/`wait`/`notify` |
| 各家实现私有，不可移植 | 工具接口、数据模型、传输绑定全部标准化 |
| 无法跨工具/跨机器协作 | v0.2 远程绑定（规划） |

SCMP 规范诞生之前已有落地方案 [opencode-agent-bridge](https://github.com/Mooling0602/opencode-agent-bridge)（npm 插件，已在真实模型上验证六工具模型的可行性与模型行为特征）。本规范将其经验协议化，并吸收 A2A 语义层（调研存档见 [research-acp-a2a.md](research-acp-a2a.md)）。

## 3. 非目标（当前草案期）

以下内容明确排除在外：

- **网关联邦与集群**（gateway↔gateway 互联路由、多节点共享总线）——v0.3 候选；v0.2 以 Host 多归属覆盖跨域可达性（[40-remote-binding](40-remote-binding.md)）。
- **宿主插件实现** —— 本仓库只交付规范与 Schema；适配器（opencode、dsh 等）在各自仓库实现。
- **工具执行沙箱、资源配额、计费** —— 属于宿主职责。
- **流式传输（SSE 等）** —— 消息均为完整投递，无流式语义。
- **文件/图像等富媒体 Part** —— 数据模型预留扩展位，当前只定义 `text` Part。

## 4. 术语

| 术语 | 定义 |
|---|---|
| 宿主（Host） | 实现 SCMP 的 Coding Agent 运行时（如 opencode、dsh） |
| 会话（Session） | 宿主中一次持续的 LLM 对话实例，拥有独立上下文 |
| 调用方（Caller） | 发起 `dispatch`/`wait` 等工具调用的会话 |
| 目标（Target） | 被派发消息的会话 |
| 信封（Envelope） | 会话间消息的标准包装，含路由与关联信息 |
| 派发（Dispatch） | 一次期望回复的异步任务委派，由 DispatchRecord 追踪 |
| 信任域（Trust Domain） | v0.1 中 = 同一用户数据目录下的全部会话 |

## 5. 架构分层

```
┌──────────────────────────────────────────────┐
│  L1 工具接口层（模型所见）                   │
│  dispatch · wait · notify · check            │
│  list_sessions · get_self_metadata           │
├──────────────────────────────────────────────┤
│  L2 数据模型层（语义核心）                   │
│  Session · Envelope · DispatchRecord 状态机  │
│  correlationId 幂等 · callChain 环检测 · TTL │
├──────────────────────────────────────────────┤
│  L3 传输绑定层                               │
│  v0.2: local（文件注册表 + inbox）           │
│        remote（WS 网关路由，纯路由域模型）   │
└──────────────────────────────────────────────┘
```

- **L1** 规定模型调用的六个工具的名称、参数、返回与语义（[10-tools.md](10-tools.md)）
- **L2** 规定跨宿主一致的数据结构与状态语义（[20-data-model.md](20-data-model.md)）
- **L3** 规定语义如何在具体传输上落地：[30-local-binding.md](30-local-binding.md)（本地）与 [40-remote-binding.md](40-remote-binding.md)（远程）

## 6. 与 MCP / A2A / ACP 的关系

| 协议 | 关系 |
|---|---|
| MCP | 正交互补：MCP 连接「LLM ↔ 工具」；SCMP 连接「会话 ↔ 会话」。SCMP 工具可视为一组特殊的宿主内置工具 |
| A2A | **语义对齐**：Task 状态机、taskId 幂等、Message/Part 模型、错误码风格借鉴 A2A；传输层不采用其 HTTP 绑定（v0.1） |
| ACP | 已归档并入 A2A，不采用（调研见 [research-acp-a2a.md](research-acp-a2a.md)） |

## 7. 规范组成与阅读顺序

| 文档 | 内容 | 角色 |
|---|---|---|
| `00-overview.md` | 本文 | 入口 |
| `10-tools.md` | L1 工具规范 | 实现宿主工具的直接依据 |
| `20-data-model.md` | L2 数据模型 | 语义核心 |
| `30-local-binding.md` | L3 本地绑定 | 同机多宿主互操作 |
| `35-host-adaptation.md` | 宿主适配规范 | 各家内部结构 → 协议概念的映射与降级（跨工具兼容） |
| `40-remote-binding.md` | L3b 远程绑定 | 网关路由域模型、WS 帧协议、鉴权、多归属、远程 check |
| `90-conformance.md` | 一致性要求 | 实现声明合规 |
| `../schema/` | 机器可读 JSON Schema | 工具定义可直接被宿主消费 |
| `../versions.md` | 版本化策略 | 演进规则 |

## 8. 开放问题（草案期）

- [x] 远程绑定的传输选型 —— 已定：WebSocket + JSON（v0.2-draft.1）
- [ ] 网关联邦与集群的引入时机（v0.3 候选）
- [ ] `check` 的隐私边界粒度（会话级授权开关？默认可读范围？本地与远程统一考虑）
- [ ] 富媒体 Part（file/image）的引入时机
- [ ] 英文 normative 版本的翻译启动点
- [ ] 排队策略的细粒度控制（per-target busyPolicy 的配置面）

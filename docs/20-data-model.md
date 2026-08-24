# SCMP · 数据模型（L2）

> 状态：v0.2-draft.1 · 上游：[00-overview.md](00-overview.md) · 机器可读定义：[../schema/messages/](../schema/messages/)

## 1. 标识符

| 字段 | 类型 | 规则 |
|---|---|---|
| `sessionId` | string | SCMP 规范化会话 ID：`<runtime>@<encodedNativeId>`（定义见 [35-host-adaptation](35-host-adaptation.md) §2）；原生 ID 不得进入协议数据 |
| `messageId` | string | 每个信封的唯一标识，发送方生成，UUIDv4 |
| `dispatchId` | string | 派发记录唯一标识，**调用方生成**（幂等键），UUIDv4 |
| `runtime` | string | 宿主标识，`^[a-z][a-z0-9-]*$`（如 `opencode`、`dsh`） |
| `workspace` | string | 会话工作目录的绝对路径（本地绑定）；远程绑定中为规范化标识 |

`SessionRef`（会话引用，用于路由）：

```json
{
  "gateway": "gw-home",
  "client": "laptop",
  "runtime": "opencode",
  "workspace": "/home/u/proj",
  "sessionId": "opencode@a1b2c3…"
}
```

`gateway` / `client` 为**远程绑定可选字段**（远程时必填，本地省略）；完整地址的序列化形式 `gateway:client@sessionId` 见 [40-remote-binding](40-remote-binding.md) §3。

## 2. SessionInfo（会话记录）

宿主注册到注册表中的会话元数据，也是 `get_self_metadata` / `list_sessions` 的数据来源。

| 字段 | 类型 | 说明 |
|---|---|---|
| `ref` | SessionRef | 会话引用 |
| `title` | string | 人类可读标题；宿主无原生标题时按合成规则降级（[35-host-adaptation](35-host-adaptation.md) §3） |
| `status` | `idle` \| `busy` \| `terminated` | `busy` = 正在处理一轮 LLM 交互 |
| `capabilities` | string[] | 可选能力声明（v0.1 保留，可为空） |
| `protocolVersion` | string | 宿主实现的 SCMP 版本 |
| `updatedAt` | timestamp | 心跳时间（见 [30-local-binding](30-local-binding.md) §4） |
| `stale` | boolean | 是否心跳过期（由观察方计算，不持久化） |

一致性要求：

- 宿主**必须**在会话可交互后尽快注册 SessionInfo，在会话终止时置 `terminated`。
- `status` 变化**必须**在 5 秒内反映到注册表。

## 3. 身份与信任域（v0.1）

v0.1 只有**本地信任域**：同一用户数据目录（见 [30-local-binding](30-local-binding.md) §2）下的所有会话彼此信任。`get_self_metadata` 返回的信息即消息路由中的身份声明。

- 信封中的 `sender` **必须**与注册表中的 SessionInfo 一致；接收宿主**应当**校验。
- 消息级鉴权（能力 token、会话级授权）**预留**给 v0.2 远程绑定；数据模型中 `Envelope.auth` 为保留字段。

## 4. 消息信封（Envelope）

会话间传递的一切消息都使用信封包装：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `protocolVersion` | string | ✓ | `"0.2"` |
| `messageId` | string | ✓ | 唯一标识，幂等去重依据 |
| `kind` | string | ✓ | `dispatch` \| `wait` \| `notify` \| `reply` \| `system` \| `check` |
| `sender` | SessionRef | ✓ | 发送方 |
| `target` | SessionRef | ✓ | 接收方 |
| `payload` | Payload | ✓ | 类型化内容（§5） |
| `correlationId` | string | – | `reply` **必须**携带，指向所回复消息的 `messageId` |
| `dispatchId` | string | – | `kind=dispatch\|wait` 时**必须**：关联的派发记录；`reply` 携带以回指 |
| `callChain` | string[] | – | 调用链（§8），`dispatch`/`wait` **必须**携带 |
| `ttl` | integer | – | 派发记录存活秒数，默认 604800（7 天） |
| `createdAt` | timestamp | ✓ | 创建时间 |
| `auth` | object | – | 保留字段（v0.2 remote） |

`kind` 语义：

- `dispatch` / `wait`：期望回复的任务消息（区别在于调用方是否阻塞等待）
- `notify`：单向消息，不期望回复
- `reply`：对 `dispatch`/`wait`/`check` 的结果回送（自动生成，见 §7；check 见 [40-remote-binding](40-remote-binding.md) §9）
- `system`：宿主间控制消息（v0.2 仅保留占位）
- `check`：远程模式下 check 工具的路由式请求信封（携带可选 `request` 字段，见 [40-remote-binding](40-remote-binding.md) §9）

## 5. Payload 与 Part

对齐 A2A 的 Message/Part 模型：

```json
{
  "parts": [
    { "type": "text", "text": "请分析 src/ 下的模块依赖", "mime": "text/markdown" }
  ]
}
```

- v0.1 **只定义** `text` Part（`type:"text"`，含 `text` 与可选 `mime`，默认 `text/plain`）。
- `parts` 为数组，**必须**至少含 1 个 Part。
- 未知 `type` 的 Part：宿主**必须**忽略并记录日志，不得判为错误（向前兼容）。
- 富媒体 Part（`file`/`image`/`data`）为未来扩展，命名对齐 A2A。

## 6. 派发记录（DispatchRecord）与状态机

DispatchRecord 是一次 `dispatch`/`wait` 的持久化跟踪实体（幂等键的载体）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `dispatchId` | string | 调用方生成，幂等键 |
| `envelope` | Envelope | 原始派发信封 |
| `state` | string | 状态机（下） |
| `result` | Payload \| null | 结果（`completed` 时） |
| `error` | ErrorObject \| null | 失败原因（`failed` 时） |
| `createdAt` / `updatedAt` | timestamp | 时间戳 |
| `expiresAt` | timestamp | TTL 过期点 |

状态机（对齐 A2A Task 生命周期，v0.1 简化）：

```
submitted → working → completed
                ├────→ failed
                └────→ canceled
```

迁移触发器（满足任一即可推进状态）：

1. **自动**：目标宿主观察到目标会话由该派发触发的处理轮次结束（`busy→idle`）→ `working→completed`，`result` = 该轮最终 assistant 回复（text Part；「处理轮次」与抽取定义见 [35-host-adaptation](35-host-adaptation.md) §5）。
2. **显式兜底**：目标会话中的模型调用 `notify` 且消息内容包含对应 `dispatchId` 与完成声明 → 同上迁移。兜底路径用于自动观察失效的场景（如多实例事件不可靠）。
3. `failed`：目标会话处理报错、目标 `terminated`、或处理轮次失败。
4. `canceled`：调用方在超时后放弃（记录层面标记）。

幂等与分工：

- 相同 `dispatchId` 的重复提交**必须**返回既有 DispatchRecord 的当前状态，**不得**重复投递消息。
- DispatchRecord 由**调用方宿主**创建，由**目标宿主**推进状态（分工细节见 [30-local-binding](30-local-binding.md) §6）。
- 状态**只允许前进**（submitted→working→终态），**不得**回退。

## 7. 结果自动回送（核心机制）

> 设计动机：实测表明，若不明确告知，模型会反复调用 `check` 轮询结果。SCMP 将「结果主动送达」作为协议的一等机制。

- 目标宿主在 dispatch 进入 `completed`/`failed` 时，**必须**生成 `kind=reply` 的信封（`correlationId` = 原派发信封的 `messageId`，`dispatchId` 回指派发记录）写入**调用方** inbox。
- 调用方宿主**必须**将 reply 作为新消息注入调用方会话上下文（角色 `user` 或宿主等价物），内容**应当**包含：发送方标识、结果文本、`dispatchId`。
- 注入格式建议（宿主可调整措辞，信息不得缺失）：

  ```
  [SCMP] 会话 <sender title>（<sessionId 前 8 位>）对派发 <dispatchId 前 8 位> 的结果已送达：
  <result text>
  ```

- 因此，`dispatch` 的工具描述**必须**含防轮询文案（见 [10-tools](10-tools.md) §1）；`check` 定位为兜底与主动检查，**不是**结果获取的主路径。

## 8. 环检测（callChain）

死锁场景：A `wait` B 且 B `wait` A，双方阻塞至超时。SCMP 用 `callChain` 在协议层拒绝环：

- `callChain` 为 sessionId 有序列表，从链首会话开始。
- 发起 `dispatch`/`wait` 时，调用方宿主**必须**构造 callChain = 父链（若自身因处理 SCMP 派发而发起）+ [自身 sessionId]。
- 投递前宿主**必须**检查：`target.sessionId ∈ callChain` → 拒绝，错误码 `40803 loop-detected`。
- `len(callChain) > maxDepth`（默认 8，宿主可配）→ 拒绝，同样 `40803`。
- `notify`/`reply` 不产生等待依赖，**不参与**环检测（callChain 仍随消息传播以备观察）。

## 9. TTL 与清理

- DispatchRecord 默认 TTL 7 天（信封 `ttl` 可按次覆盖）；到期后宿主**可以**清理记录，清理**必须**幂等。
- 已过期的 dispatch 收到迟到 reply：宿主**应当**丢弃 reply 并记录日志（调用方已放弃）。
- 清理**不得**删除仍处 `working` 且未过期的记录。

## 10. 错误码

风格对齐 JSON-RPC 2.0（保留段复用，自定义段从 40800 起）：

| 码 | 名 | 场景 |
|---|---|---|
| -32602 | invalid-params | 工具参数校验失败 |
| -32603 | internal | 宿主内部错误 |
| 40801 | target-not-found | 目标会话不存在/未注册/已过期 |
| 40802 | target-busy | 目标忙且策略为拒绝（默认策略为排队，见 [30-local-binding](30-local-binding.md) §6） |
| 40803 | loop-detected | 环检测命中或深度超限 |
| 40804 | wait-timeout | `wait` 超时（绑定层错误映射用；工具层超时为正常返回，见 [10-tools](10-tools.md) §2） |
| 40805 | dispatch-expired | 派发记录已过 TTL |
| 40806 | permission-denied | 信任域外访问（v0.1 罕见，为 remote 预留） |
| 40807 | session-terminated | 目标会话已终止 |
| 40808 | payload-too-large | 信封超过大小上限 |
| 40809–40814 | — | 远程绑定新增错误码（client-offline / auth-failed / version-mismatch / wrong-domain / request-timeout / superseded），见 [40-remote-binding](40-remote-binding.md) §11 |

ErrorObject 形状：

```json
{ "code": -32602, "message": "target is required", "data": {} }
```

## 11. 大小限制

- 单个信封序列化后**不得**超过 256 KiB；超出 → `40808 payload-too-large`。
- `result` Payload 同限。需要传递大内容时，**应当**由目标会话写入共享文件系统并在结果文本中引用路径。

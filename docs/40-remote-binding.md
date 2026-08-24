# SCMP · 远程绑定（L3b）

> 状态：v0.2-draft.1 · 上游：[00-overview](00-overview.md) / [20-data-model](20-data-model.md) / [35-host-adaptation](35-host-adaptation.md) · 机器可读定义：[../schema/remote-frames.json](../schema/remote-frames.json)

## 1. 范围与拓扑

远程绑定将 SCMP 的信任域从「同一用户数据目录」扩展到「经网关互联的多机 Host」。部署形态：

```
     网关 gw-home（路由域 A）                网关 gw-vps（路由域 B）
     ┌────────────────────┐                ┌─────────────────────┐
     │  纯路由 + presence │────✗ 不互联 ───│  纯路由 + presence  │
     └──┬─────────────┬───┘                └──┬──────────────┬───┘
        │             │                       │              │
  ┌─────┴────┐  ┌─────┴──────┐         ┌──────┴──────┐  ┌────┴─────┐
  │ Host A   │  │ Host B     │         │ Host B      │  │ Host C   │
  │ 只连 A   │  │ 多归属 A+B │         │（同一实例） │  │ 只连 B   │
  └──────────┘  └────────────┘         └─────────────┘  └──────────┘
```

核心模型：

1. **网关 = 独立路由域**：每个网关是一个路由域；网关之间零协议（联邦与集群留 v0.3，见 §15）。
2. **纯路由**：网关不持有任何语义状态（派发记录、收件箱、会话内容均不存在于网关）。网关仅维护**在线快照型 presence 缓存**（§7）——可重建、非事实源、断连即清。
3. **Host 多归属**：一个 Host 可同时连接多个网关。跨域可达性完全由「共享成员」实现：A 找到 B 的前提是两者至少共享一个网关。多归属的全部复杂性收敛在 Host 适配层（§10）。
4. **Host 主动外连**（NAT 友好）：网关从不反向接入 Host。

非目标（本分册）：网关间联邦/集群、应用层流控（256 KiB 帧上限 + WS/TCP 兜底）、网关侧消息暂存（离线投递见 §8 的 NACK 策略）、会话内容缓存。

## 2. 术语

| 术语 | 定义 |
|---|---|
| 网关（Gateway） | 部署在中心服务器上的路由进程，构成一个路由域 |
| 客户端（Client） | 通过 WebSocket 连接到网关的 Host 实例，以 `clientId` 标识 |
| 路由域 | 一个网关的管辖范围；地址中由 `gatewayId` 标识 |
| 多归属 | 同一 Host 实例同时连接多个网关的状态 |

## 3. 地址模型

### 3.1 完整地址文法

```
address  := gatewayId ":" clientId "@" scmpId
scmpId   := runtime "@" encodedNativeId        # 见 35 §2
```

- `gatewayId`、`clientId`、`runtime` 均匹配 `^[a-z][a-z0-9-]*$`（不含 `:` 与 `@`）；`encodedNativeId` 字符集为 `[A-Za-z0-9._~-]`。地址中不出现 `/`。
- 解析规则（逐层首个分隔符）：首个 `:` 之前为 `gatewayId`；其后首个 `@` 之前为 `clientId`；剩余部分即 `scmpId`，再按其首个 `@` 切分为 `runtime` 与 `encodedNativeId`。
- **强制全形式**：远程绑定下，工具 `target` 参数与 `list_sessions` 返回的会话标识一律使用完整地址（对模型为 opaque 字符串）。本地绑定仍直接使用 `scmpId`。

示例：`gw-home:laptop@opencode@3f2b1a7e-…`

### 3.2 SessionRef 扩展

结构化承载（[20-data-model](20-data-model.md) §1）新增两个可选字段：

```json
{
  "gateway": "gw-home",        // 远程绑定时必填
  "client": "laptop",          // 远程绑定时必填
  "runtime": "opencode",
  "workspace": "/home/u/proj",
  "sessionId": "opencode@3f2b1a7e-…"   // 仍为 scmpId（不含域限定）
}
```

序列化形式（工具参数/展示）= `gateway:client@sessionId`；结构与序列化互转由 Host 适配层完成。

## 4. 连接与生命周期

- **URL**：`wss://<host>/scmp`。生产环境**必须**使用 TLS（wss）；仅回环地址**可以**降级 `ws`。
- **握手**：连接建立后客户端**必须**首先发送 `hello`；网关校验通过回 `welcome`，失败回 `error` 帧后关闭（40810/40811）。
- **版本协商**：`hello.v` 与网关支持的次版本做字符串相等比较（v0.2 期即 `"0.2"`）；不匹配 → `40811 version-mismatch`。
- **重复连接**：同一 `clientId` 新连接接入时**新顶旧**——网关向旧连接发送 `error`（`40814 superseded`）后关闭之，presence 缓存无缝移交新连接。
- **心跳**：使用 WebSocket 层 ping/pong，不定义应用帧。
- **断连**：网关**必须**立即清除该 client 的全部 presence 缓存条目，并向其 presence 订阅者广播 offline 事件。
- **重连**（Host 侧）：指数退避（建议 1s 起、上限 60s、带抖动）；重连成功后**必须**全量重发 `session.upsert`，并重新执行 `session.list` 刷新聚合视图（§10）。

## 5. 鉴权

- 模型：**静态 token + allowlist**（MCSManager 风格）。
- 网关配置文件（TOML）持有 `gatewayId`、监听地址、TLS 证书与 token 表（token → `clientId` 绑定 + 备注）。token 为高熵随机串（建议 ≥ 256 bit）；配置文件明文存储，权限**必须**为 0600（SSH authorized_keys 模式：本地配置文件即信任锚）。
- `hello.token` 命中 allowlist → 以绑定的 `clientId` 注册；未命中 → `40810 auth-failed` 后关闭。`hello.clientId` 与 token 绑定值不一致 → 同样 40810。
- 吊销 = 从配置移除并重载（SIGHUP 或重启）；既有连接在下次重连时失效。
- `clientId` 在路由域内唯一，由网关管理员在签发 token 时分配。

## 6. 帧协议

WebSocket 文本帧，UTF-8 JSON。帧通用字段：`t`（帧类型）、`v`（协议版本，仅握手帧携带）。单帧 ≤ 256 KiB。

| 帧 | 方向 | 字段 | 说明 |
|---|---|---|---|
| `hello` | C→G | `t, v, token, clientId` | 握手与鉴权 |
| `welcome` | G→C | `t, v, gatewayId` | 注册成功，宣告网关身份 |
| `error` | G→C | `t, error{code,message,data?}` | 错误（鉴权/版本/被顶替等，随后关闭连接） |
| `session.upsert` | C→G | `t, session: SessionInfo` | 会话宣告/更新（含心跳语义） |
| `session.offline` | C→G | `t, sessionId` | 本地会话终止 |
| `session.list` | C→G | `t, reqId, filter?{keyword,status,workspace}` | 拉取 presence 缓存 |
| `session.list.result` | G→C | `t, reqId, sessions: SessionInfo[]` | 拉取结果 |
| `presence.sub` | C→G | `t, target: clientId` | 订阅某客户端上下线 |
| `presence.event` | G→C | `t, client, online` | 上下线事件 |
| `env` | 双向 | `t, envelope: Envelope` | 业务信封（L2 原样） |
| `nack` | G→C | `t, messageId, error{code,message}` | 信封路由失败 |

约束：

- `env` 帧中的信封即 [20-data-model](20-data-model.md) §4 的 Envelope，不做二次包装；网关按 `envelope.target` 路由。
- `session.list` 的过滤在网关端执行（`keyword` 标题子串、`status` 枚举、`workspace` 精确匹配）。
- 未定义的 `t`：接收方**必须**忽略（向前兼容）。

## 7. 会话宣告与 presence

- **宣告**：Host 在会话可交互后**必须**经所连**每个**网关 `session.upsert`（多归属 = 全域宣告）；状态变化时即时更新；此外**必须**周期性（≤ 60s）重发以维持缓存新鲜。
- **在线快照缓存**：网关 presence 缓存 = 「当前在线客户端的会话快照」。断连即清（§4）；客户端离线后其会话不再出现在 `session.list` 中（对其发信封也会 NACK，见 §8）。
- **订阅**：`presence.sub` 为 client 级订阅——收到该 client 的上线/下线事件。会话明细变更不推送；需要时拉取 `session.list`。订阅状态随连接存亡（路由元数据，非语义状态）。
- 缓存条目上限：网关**可以**限制单 client 宣告的会话数（建议默认 1000），超限时以 `error` 帧拒绝该次 upsert。

## 8. 信封路由

发送路径：Host → `env` 帧 → 网关 → `env` 帧 → 目标 Host。网关逐帧处理：

1. `envelope.target.gateway ≠ 本网关 gatewayId` → 向发送方回 `nack`（`40812 wrong-domain`）。
2. 目标 client 不在线 → `nack`（`40809 client-offline`）。**网关不暂存任何信封**；调用方 Host 依 presence 订阅感知对方上线后自行重发（幂等由 `messageId` 保证）。
3. 在线 → 直接转发。转发成功不回 ACK；投递可靠性依赖 WS/TCP，端到端语义（超时、状态推进）由 Host 侧既有机制兜底（wait 超时、dispatch TTL）。
4. 网关**可以**顺手做 callChain 环检查（信封内有 `callChain`），命中 → `nack`（`40803`）；不检查也合规（环检测的 MUST 在 Host 侧，见 20 §8）。

有序性：单连接内 WS/TCP 保序；跨域/换路重发场景由 `messageId` 幂等去重（20 §既有）与 `createdAt` 排序兜底。

## 9. 路由式请求-响应（check 的远程语义）

纯路由下网关无会话内容，`check` 在远程模式实现为一轮请求-响应：

```
调用方 Host                          Gateway                    目标 Host
    │ check(target, limit)              │                           │
    │── env{kind:"check", request:{limit}, target…} ──路由──▶       │
    │  （工具调用挂起，默认 30s 超时）      │                           ├─ 读本地上下文（35 §4 映射）
    │◀───────────── env{kind:"reply", correlationId=check.messageId}│
    │  呈现给模型                        │                           │
```

- 信封 `kind` 枚举新增 **`check`**；可选字段 `request`（object）承载请求参数：`{"limit": 10}`。
- 响应为标准 `reply` 信封（`correlationId` 指向 check 信封的 `messageId`），结果文本置于 payload。
- 超时（默认 30s，宿主可配）→ 工具返回 `{ok:false, error:{code: 40813 request-timeout}}`。
- 目标离线 → 网关 NACK 40809，Host 转为工具错误返回；目标 Host 版本过旧不识别 `kind=check` → 表现为超时（40813）。
- `get_self_metadata` 纯本地，不受远程影响；`list_sessions` 走 presence 缓存（§7）。

## 10. 多归属聚合（Host 适配层职责）

多归属的复杂性全部收敛在 Host 适配层，网关协议不变：

1. **全域宣告**：本地全部会话向所连每个网关 upsert；会话终止向全部网关 offline。
2. **聚合**：`list_sessions` = 跨所连网关聚合 `session.list` 结果。按 `sessionId`（scmpId）去重：取 `updatedAt` 最新者；平局取 `gatewayId` 字典序小者（确定性规则）。返回给模型的每个条目标识 = 该会话对应域的完整地址。
3. **选路**：发送时按目标地址的 `gatewayId` 选择网关连接。
4. **换路重发**：所选网关 NACK 40809 且目标多归属可见于其他域时，**可以**换域重发（`messageId` 幂等保证目标侧不重复注入）。
5. **冗余红利**：任一网关故障时，多归属 Host 的会话仍可经其余域可达。
6. 本地绑定与远程绑定**可以**并存：`list_sessions` 结果 = 本地注册表 ∪ 远程聚合（同样按 scmpId 去重）。

## 11. 错误码汇总（本分册新增）

| 码 | 名 | 场景 | 载体 |
|---|---|---|---|
| 40809 | client-offline | 目标客户端不在线 | nack / 工具错误 |
| 40810 | auth-failed | token 无效或 clientId 不匹配 | error 帧（随后关闭） |
| 40811 | version-mismatch | 版本协商失败 | error 帧（随后关闭） |
| 40812 | wrong-domain | 地址 gatewayId 非本网关 | nack |
| 40813 | request-timeout | check 等路由式请求超时 | 工具错误 |
| 40814 | superseded | 同 clientId 新连接接入，旧连接被顶 | error 帧（随后关闭） |

## 12. 安全考量

- 生产**必须** wss；token 即凭证，泄露处置 = 从 allowlist 移除并重载。
- 网关不落任何会话内容与消息正文（内存中仅瞬时转发），日志**应当**避免记录 payload。
- 信任边界：连接到同一网关的 client 彼此处于同一信任域（同本地绑定的信任域语义）。跨信任域隔离靠部署分离（不同网关）。

## 13. 与本地绑定的关系

- 语义层（L2）完全共用：同一套信封、状态机、幂等、环检测、TTL；远程模式 ≈ 信封改走网关路由而非写 inbox 文件。
- 绑定选择在 Host 适配层：按目标地址形态路由——`scmpId` → 本地绑定；完整地址 → 对应网关连接。
- DispatchRecord 永远只在调用方 Host 本地（远程模式同样），与 30 §6 的分工表一致（目标 Host 推进状态、生成 reply，经网关路由回调用方）。

## 14. 一致性

| 级别 | 名称 | 要求 | 声明标识 |
|---|---|---|---|
| C4 | SCMP-Remote-Gateway | 网关按本分册实现（§3-§9、§11） | `scmp-remote-gateway/0.2` |
| C5 | SCMP-Remote-Host | Host 远程适配按本分册实现（§3-§4、§6-§10） | `scmp-remote-host/0.2` |

检查表见 [90-conformance](90-conformance.md) §2.5。

## 15. 开放问题

- [ ] 联邦（gateway↔gateway 互联路由）与集群（多节点共享总线）——v0.3 候选
- [ ] presence 粒度扩展（会话级变更推送、订阅过滤）
- [ ] 网关配置热重载与在线管理面（CLI/API）的规范面
- [ ] 远程模式下 check 隐私边界（与 00 §8 的本地问题合并考虑）

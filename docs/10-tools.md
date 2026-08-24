# SCMP · 工具接口（L1）

> 状态：v0.2-draft.1 · 机器可读定义：[../schema/tools/](../schema/tools/)

## 0. 总则

### 0.1 命名

SCMP 定义六个**规范名**（canonical names）：

`dispatch` · `wait` · `notify` · `check` · `list_sessions` · `get_self_metadata`

- 宿主**可以**为规范名加自身生态的前缀（如 `agent_bridge_dispatch`、`scmp_dispatch`）。
- 规范名与宿主既有工具冲突时，宿主**必须**加前缀。
- 工具的**参数名必须**与本文及 Schema 完全一致（不得改动），以保证跨宿主的提示词与文档可移植。

### 0.2 统一返回形状

- 成功：`{ "ok": true, ...工具自有字段 }`
- 失败：`{ "ok": false, "error": { "code": <int>, "message": <string>, "data"?: <any> } }`

失败时工具调用本身仍视为「已执行」（不是宿主层工具崩溃），错误在返回体内表达——避免把协议错误误报为系统故障，让模型能读到错误码并自行决策。

### 0.3 工具描述的规范性文案

工具描述（注入给模型的 description）承载行为引导。实测（opencode-agent-bridge）表明：不写明「结果自动回送」，模型会反复 `check` 轮询。因此：

- 每个工具的描述**必须**包含本文各节标注的【必须包含文案】；宿主可以翻译、扩展，但语义不得削弱。
- 文案与规范正文冲突时，以规范正文为准。

### 0.4 目标寻址

- 所有工具的 `target` 参数承载目标会话标识，按绑定模式取两种形式之一：
  - **本地绑定**：`scmpId`（`runtime@encodedNativeId`，35 §2）
  - **远程绑定**：完整地址 `gatewayId:clientId@scmpId`（[40-remote-binding](40-remote-binding.md) §3，对模型为 opaque 字符串）
- `list_sessions` 返回的会话标识与 `target` 参数形式一致；Host 适配层负责按地址形态选择绑定路径（40 §13）。

## 1. dispatch

**异步派发**：向目标会话发送期望回复的消息，**立即返回**；目标完成后结果由系统自动回送（[20-data-model](20-data-model.md) §7）。

输入：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `target` | string | ✓ | 目标会话的 sessionId |
| `message` | string | ✓ | 派发的消息正文（Markdown） |
| `timeout` | integer | – | 结果等待上限（秒），默认 1800；超时后记录标记超时 |

输出：

```json
{ "ok": true, "dispatchId": "<uuid>", "state": "submitted", "note": "结果将自动送达，无需轮询" }
```

语义细则：

- 宿主**必须**生成 dispatchId（UUIDv4）并创建 DispatchRecord（幂等键，见 20 §6）。
- 宿主**必须**在返回前完成 callChain 环检测（20 §8）。
- 返回即完成本次工具调用；结果以 `reply` 信封异步送达当前会话上下文。
- 同 `dispatchId` 重试（如宿主层重发）**必须**幂等返回既有状态，不得重复投递。
- 成功返回**应当**包含提示性 `note`（防轮询，文案可调整）。

> 【必须包含文案】"异步派发消息到目标会话。调用立即返回；目标会话完成后，系统会**自动**把结果作为新消息送达当前会话——**不要**调用 check 轮询结果。仅当超过 timeout 仍未收到回送时，才使用 check 兜底。"

## 2. wait

**同步等待**：向目标会话发送期望回复的消息，并**阻塞**当前会话直至收到回复（或超时）。

输入：`target`、`message` 同上；`timeout`（秒，默认 1800）。

输出（三态）：

```json
// 收到回复
{ "ok": true, "state": "completed", "dispatchId": "<uuid>", "result": "<回复文本>" }
// 超时
{ "ok": true, "state": "working", "dispatchId": "<uuid>", "timedOut": true }
// 目标失败
{ "ok": true, "state": "failed", "dispatchId": "<uuid>", "error": { "code": 40807, "message": "…" } }
```

语义细则：

- 阻塞期间宿主**不得**向 LLM API 发送任何请求——阻塞由「工具调用不返回」实现，这是 `wait` 的定义性特征。
- 超时**不得**视为工具错误（`ok:true` + `timedOut:true`）；调用方模型可据 `state` 决定续等（再调一次 wait）或转 dispatch/check。
- **必须**在阻塞开始前完成环检测：目标在 callChain 上 → 立即返回 `{ "ok": false, "error": { "code": 40803 } }`，不阻塞。深度超限同理。
- 协议层 `wait` 等价于：`dispatch`（同一 DispatchRecord）+ 阻塞等待其 `completed`/`failed`。

> 【必须包含文案】"向目标会话发送消息并阻塞等待其回复，适用于必须拿到结果才能继续的场景。阻塞期间当前会话不会有任何进展；若目标会话正在等待你（会形成循环等待），本调用会被拒绝。优先考虑 dispatch（非阻塞，结果自动送达）。"

## 3. notify

**单向通知**：发送不期望回复的消息，立即返回投递结果。

输入：`target`、`message`（语义同上）。

输出：`{ "ok": true, "delivered": true }` 或错误（如 40801）。

语义细则：

- 不创建 DispatchRecord、不产生 `reply`、不参与环检测。
- 用途举例：状态同步、补充上下文、**完成声明显式兜底**——目标会话的模型在处理完某派发后，可 `notify` 调用方并在消息中包含对应 `dispatchId` 与完成说明，驱动状态迁移（20 §6 触发器 2）。

> 【必须包含文案】"向目标会话发送一条不需要回复的通知消息。仅在你明确不需要对方处理并回复时使用；若你是在回复对方的派发任务，请在消息中注明对方的 dispatchId。"

## 4. check

**检查目标会话**的上下文与状态（兜底获取结果 / 主动了解进展）。

输入：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `target` | string | ✓ | 目标会话的 sessionId |
| `limit` | integer | – | 返回最近 N 条消息，默认 10，上限 50 |

输出：

```json
{
  "ok": true,
  "target": "<sessionId>",
  "title": "会话标题",
  "status": "busy",
  "recent": [ { "role": "user", "text": "…" }, { "role": "assistant", "text": "…" } ]
}
```

语义细则：

- `recent` 按时间升序；每条**应当**截断至 500 字符（防调用方上下文膨胀）。
- `recent` 中 `role` **必须**为 `user` | `assistant` | `system` | `tool` 之一（宿主内部角色按 [35-host-adaptation](35-host-adaptation.md) §4 映射，不得虚构内容）。
- **远程绑定**：check 实现为路由式请求-响应（`kind=check` 信封 + reply，[40-remote-binding](40-remote-binding.md) §9），默认超时 30 秒，超时返回 `40813 request-timeout`。
- **不是**结果获取主路径（主路径 = 自动回送）；工具描述**必须**强调这一点（见下文案）。
- 隐私：v0.1 信任域内全量可读；会话级授权开关列为开放问题（[00-overview](00-overview.md) §8）。

> 【必须包含文案】"查看目标会话的状态与最近消息。仅用于：① 派发超时后兜底获取结果；② 明确需要了解对方进展。结果正常会自动送达——不要轮询。"

## 5. list_sessions

列出信任域内符合查询条件的会话。

输入：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `keyword` | string | – | 按标题子串过滤（不区分大小写） |
| `status` | string | – | `idle` / `busy` / `terminated` |
| `workspace` | string | – | 按工作区路径过滤 |

输出：

```json
{
  "ok": true,
  "sessions": [
    { "sessionId": "…", "title": "…", "status": "idle", "workspace": "…", "runtime": "opencode" }
  ]
}
```

语义细则：

- 结果**必须**排除调用方自身（自身信息用 `get_self_metadata`）。
- 排序：最近活跃优先（`updatedAt` 降序）；**应当**截断至前 50 条并在返回中注明总数。

> 【必须包含文案】"列出当前可协作的会话（ID 与标题）。向其他会话派发任务前，先用它确认目标会话 ID。"

## 6. get_self_metadata

返回当前会话的身份元数据（身份确认与鉴权用途）。

输入：无。

输出：

```json
{
  "ok": true,
  "self": {
    "sessionId": "…", "title": "…", "status": "idle",
    "workspace": "…", "runtime": "opencode",
    "capabilities": [], "protocolVersion": "0.2"
  }
}
```

语义细则：

- 只读；**不得**产生任何副作用。
- 主要用途：让模型在协作消息中正确自报身份；为宿主鉴权提供会话上下文。

> 【必须包含文案】"返回当前会话的 ID、标题等元数据，用于在与其他会话协作时表明身份。"

## 7. 示例：三会话协作

```
会话 A（重构协调者）                  会话 B（模块甲）           会话 C（模块乙）
│ list_sessions
│─── 得到 B、C ──────────────────────│                          │
│ dispatch(B, "梳理 parser 模块…")   │                          │
│ dispatch(C, "梳理 renderer 模块…") │                          │
│ （继续自己的工作）                 ├ 收到派发，开始处理       ├ 收到派发，开始处理
│                                    ├ 完成 → reply 自动回送    ├ 完成 → reply 自动回送
│◀── [SCMP] B 的结果已送达 ──────────│                          │
│◀── [SCMP] C 的结果已送达 ──────────│                          │
│ （汇总，全程不轮询 check）         │                          │
```

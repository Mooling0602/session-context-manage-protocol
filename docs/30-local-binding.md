# SCMP · 本地绑定（L3a）

> 状态：v0.1-draft.1 · 范围：同一机器、同一用户下的多宿主/多实例互操作

## 1. 拓扑与信任域

- 本地绑定的信任域 = 同一用户数据目录（§2）下的全部会话。
- 参与方：任意多个宿主进程与实例（如 2 个 opencode 实例 + 1 个 dsh 实例），通过文件系统共享注册表与收件箱。
- 无服务进程、无端口、无守护进程——一切状态在文件系统，一切通知靠文件观察 + 轮询兜底。

## 2. 目录布局

基目录：`$SCMP_HOME`，缺省 `${XDG_DATA_HOME:-$HOME/.local/share}/scmp`

```
$SCMP_HOME/
├── sessions/<sessionId>.json           # SessionInfo（20 §2）
├── dispatches/<dispatchId>.json        # DispatchRecord（20 §6）
└── inbox/<sessionId>/<messageId>.json  # 待投递信封（20 §4）
```

- 文件均为 UTF-8 JSON，末尾换行；单文件 ≤ 256 KiB（20 §11）。
- 目录权限**应当**为 0700，文件 0600（信任域内私有）。
- `sessionId` 在信任域内跨宿主唯一；若宿主原生会话 ID 可能与其他宿主冲突，宿主**必须**加前缀（如 `opencode-<原生ID>`）。

## 3. 原子写与并发

- 所有写入**必须**原子：写同目录临时文件（前缀 `.tmp-`）→ `rename()` 替换目标。
- **不得**对共享 JSON 做「读-改-写」式更新；DispatchRecord 状态推进 = 整文件覆盖 + `updatedAt` 单调递增。
- 并发写同一记录（罕见，如自动观察与显式兜底同时迁移状态）时后写覆盖先写；状态机**只允许前进**（20 §6），观察方以终态优先。
- 读侧依赖 rename 原子性，不会读到半写状态。

## 4. 会话注册与生命周期

- **注册**：会话可交互后**必须**尽快写入 `sessions/<sessionId>.json`。
- **心跳**：宿主**必须**定期（≤ 60 秒）刷新 `updatedAt`。
- **失效**：`updatedAt` 距今超过 15 分钟（`staleAfter`，宿主可配）的会话视为 **stale**——`list_sessions` **应当**以 `stale:true` 标注；派发到 stale 会话时，宿主**应当**在工具返回中提示调用方（但不阻止）。
- **终止**：会话结束**必须**置 `status:"terminated"`；terminated 记录**可以**在 7 天后清理。
- **崩溃恢复**：宿主启动时**应当**扫描自身遗留的 `busy` 会话记录并修正；无法确认时置 `idle` 并标注 stale。

## 5. 消息投递（inbox）

- **发送方宿主**：原子写 `inbox/<target.sessionId>/<messageId>.json`，写入完成即视为「已投递」（at-least-once）。
- **接收方宿主**：**必须**持续观察自身所有会话的 inbox（inotify / FSEvents / kqueue 事件，或 ≥ 1 秒间隔轮询兜底；事件 + 轮询并用是推荐做法）。
- **取走**：处理该信封（注入会话 / 推进 dispatch 状态）后，接收方**必须**删除该文件，或移入 `inbox/<sessionId>/processed/`（保留 ≤ 7 天）。
- **幂等**：接收方**必须**按 `messageId` 去重——重复投递直接丢弃。崩溃重启后 processed 状态丢失时靠此兜底。
- **顺序**：同一 sender 的多条消息**应当**按 `createdAt` 排序注入；跨 sender 不保证全局顺序。

## 6. 派发记录维护（谁写什么）

| 动作 | 执行者 | 说明 |
|---|---|---|
| 创建 DispatchRecord（`submitted`） | 调用方宿主 | dispatch / wait 时 |
| `submitted → working` | 目标宿主 | 派发消息注入目标会话且会话转入 busy |
| `working → completed / failed` | 目标宿主 | 自动观察（处理轮次结束）或显式 notify 兜底（20 §6） |
| 写 reply 信封到调用方 inbox | 目标宿主 | 进入 `completed` / `failed` 时 |
| `→ canceled` / 超时标记 | 调用方宿主 | `timeout` 到点 |
| TTL 清理 | 双方均可 | `expiresAt` 过期且非 `working` |

- **排队策略（默认）**：目标 busy 时消息留在 inbox，待目标 idle 后按序注入——即默认**不**触发 `40802 target-busy`；宿主可配置 `busyPolicy=reject` 改为拒绝。
- 每 dispatch 一文件，天然支持**多调用方并发派发同一目标**（解决原实现的「单派发者覆盖」限制）。

## 7. 结果自动回送的实现要点

- 目标宿主观察「目标会话的处理轮次由 SCMP 派发消息触发（或该轮包含 SCMP 派发消息）且轮次结束（busy→idle）」→ 视为完成轮次。
- 完成轮次判定**应当**保守：一轮内存在多条派发时，按各自触发的回复轮次分别回送；无法精确切分时，**允许**将整轮最终 assistant 回复作为所有相关 dispatch 的结果，并在 result 文本中注明这一情况。
- reply 注入调用方会话的格式见 20 §7。
- 多实例场景：调用方宿主只处理自己观察到的、投递到自己管理会话的 inbox 的 reply；**不得**假设其他实例会代为注入（避免重复注入）。

## 8. 与原实现（opencode-agent-bridge）的差异

| 原实现问题（README「已知限制」） | 本规范的解决 |
|---|---|
| 「水位 + 文本探针 + parentID」猜回复，50 条滑窗可能失效 | correlationId / dispatchId 显式关联，不依赖消息窗口 |
| 同一目标注册表单值，多调用方互相覆盖 | per-dispatch 文件，天然多调用方 |
| 多实例注册表互不感知，通知重复/丢失 | at-least-once + messageId 幂等去重 + notify 显式兜底保留 |
| 环等待死锁，双方阻塞至超时 | callChain 协议级拒绝（40803） |
| 注册表 7 天 TTL 的清理边界模糊 | TTL 语义协议化（20 §9）：非 working 才可清理，清理幂等 |

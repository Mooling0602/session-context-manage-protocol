# SCMP · 宿主适配规范（Host Adaptation）

> 状态：v0.2-draft.2 · 目的：各家 Coding Agent 的内部会话数据结构与命名不一致，本分册定义「宿主内部概念 → SCMP 协议概念」的规范化映射与降级规则，保证跨宿主互操作。

## 1. 三条原则

1. **协议面只见 SCMP 概念**：原生会话 ID、内部消息结构、宿主专有状态**不得**出现在任何协议数据中（信封、注册表文件、工具返回）。
2. **映射必须确定性**：同一内部状态在任何时刻映射出同一协议数据（不依赖中央协调、不随时间漂移）；宿主重启后映射结果不变。
3. **保守降级而非拒绝**：宿主缺少对应概念时，按本分册降级规则输出；**不得**因「内部没有这个概念」而不实现某个工具或字段。

## 2. 会话 ID 规范化

SCMP 会话 ID 的规范形式：

```
<runtime>@<encodedNativeId>
```

- `runtime`：宿主标识，匹配 `^[a-z][a-z0-9-]*$`（不含 `@`，见 [20-data-model](20-data-model.md) §1）。
- `encodedNativeId`：宿主原生会话 ID 的安全编码——原生 ID 中不属于 `[A-Za-z0-9._~-]` 的字符**必须**做百分号编码（percent-encoding）；编码后不得为空。
- 解析规则：按**首个** `@` 切分，左侧为 `runtime`，右侧整体为编码后的原生 ID。
- 该 ID 即注册表文件名、信封 `sender/target.sessionId`、所有工具 `target` 参数的唯一形式。
- **原生 ID 不得泄漏**：协议数据中任何位置不得出现未编码的原生 ID；宿主内部自行维护 SCMP ID ↔ 原生 ID 的换算（无状态、可计算，双向无需查表）。
- 同一宿主类型的多个实例共享原生 ID 空间（同一会话库），SCMP ID 不受实例数影响。

示例：opencode 原生 ID `3f2b1a7e-…` → `opencode@3f2b1a7e-…`；dsh 原生 ID `sess/01` → `dsh@sess%2F01`。

## 3. SessionInfo 字段映射与降级

| 字段 | 有原生概念的宿主 | 无原生概念的宿主（降级规则） |
|---|---|---|
| `title` | 直接使用原生标题 | **必须**合成：首条用户消息去除换行、截断至 50 字符；无消息时 `"untitled"`；获得更好摘要后**可以**更新 |
| `status` | 轮次处理中→`busy`；可交互→`idle`；会话结束→`terminated` | 无轮次概念：恒报 `idle`（排队机制保证安全）；结束不可检测：不置 `terminated`，由 stale 机制兜底（[30-local-binding](30-local-binding.md) §4） |
| `workspace` | 原生工作目录 | 注册前**必须**规范化为真实路径（解析符号链接；POSIX `realpath`，Windows 等价 API） |
| `updatedAt` | 原生活动时间 | 适配层**必须**自行提供心跳刷新（≤ 60 秒，30 §4） |

`capabilities` 保留名（v0.1；声明方如实填写，接收方仅作提示，**不得**据此跳过降级规则）：

| 保留名 | 含义 |
|---|---|
| `native-title` | `title` 来自宿主原生摘要而非合成 |
| `busy-tracking` | `status` 反映真实轮次追踪而非恒 `idle` |
| `history-read` | `check` 返回真实会话历史（而非降级摘要） |

## 4. 消息角色映射（`check` 与 result 抽取共用）

协议角色枚举：`user` | `assistant` | `system` | `tool`。

- 宿主原生角色映射到最近似者（如 human→`user`、ai/model→`assistant`）；无对应内部概念的消息类型可省略。
- 工具调用与其结果：**可以**映射为 `tool` 角色的文本条目（摘要形式），或省略。
- `system` 消息：宿主**可以**出于提示词隐私不返回（截断时优先丢弃）。
- **不得虚构**：映射不得凭空生成 user/assistant 内容；省略优于编造。

## 5. 处理轮次与结果抽取

「回复」在跨宿主语境下必须有唯一定义：

- **处理轮次（processing round）**：自派发消息注入目标会话起，至目标会话回到可交互状态（`idle`）止。
- **result** = 轮次内**最后一条含文本内容的 assistant 消息**的文本（按 §4 映射后的 `assistant` 角色）。
- 一轮处理多条派发：无法精确切分时，按 [30-local-binding](30-local-binding.md) §7 以整轮末条 assistant 文本作为所有相关 dispatch 的结果并在文本中注明。
- 轮次结束但无任何 assistant 文本（仅工具调用/空回复）：视为 `completed`，result 固定为 `(no textual reply)`。
- 轮次被用户打断/重定向：轮次视为结束，按上述规则抽取；宿主**可以**在 result 前缀 `[interrupted]` 标注。

## 6. 消息注入格式（跨宿主统一）

注入格式统一，保证同一份派发消息在不同宿主中的模型体验一致：

**派发/等待消息注入目标会话**（角色 `user`）：

```
[SCMP] 来自会话 <sender title>（<sender sessionId 前 12 字符>）的派发任务（ref: <dispatchId 前 8 字符>）：
<message>
（你的最终回复将由系统自动送回对方，无需手动通知）
```

**notify 注入目标会话**（角色 `user`）：

```
[SCMP] 来自会话 <sender title>（<sender sessionId 前 12 字符>）的通知（无需回复）：
<message>
```

**reply 注入调用方会话**：见 [20-data-model](20-data-model.md) §7（已定义）。

- 宿主**可以**调整措辞与语言，但**必须**保留四要素：来源身份（标题 + ID 片段）、ref 关联、消息正文、回送预期（自动回送 / 无需回复）。
- 尾注「无需手动通知」用于抑制目标模型手动调用 notify——自动回送是主路径（20 §7），notify 兜底仅在系统未送达时使用。

## 7. 与一致性分级的关系

| 本分册条款 | 归属级别 |
|---|---|
| §2 ID 规范化 | C2（数据模型） |
| §3 字段映射与降级 | C2 / C3 |
| §4 角色映射、§5 结果抽取 | C1（check/result 行为）+ C2 |
| §6 注入格式 | C1 + C3 |

以上条款已并入 [90-conformance](90-conformance.md) §2.4 检查表。

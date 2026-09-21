# SCMP · 一致性要求

> 状态：v0.2-draft.2 · 本文是合规检查表，规范性条款以各分册为准

## 1. 一致性分级

| 级别 | 名称 | 要求 | 声明标识 |
|---|---|---|---|
| C1 | SCMP-Tools | 实现六个工具（L1），含规范性描述文案 | `scmp-tools/0.2` |
| C2 | SCMP-Data | 数据模型 / 状态机 / 错误码正确（L2） | `scmp-data/0.2` |
| C3 | SCMP-Local | 本地绑定完整实现（L3a），可与同级互操作 | `scmp-local/0.2` |
| C4 | SCMP-Remote-Gateway | 网关按 40 分册实现 | `scmp-remote-gateway/0.2` |
| C5 | SCMP-Remote-Host | Host 远程适配按 40 分册实现 | `scmp-remote-host/0.2` |

- C3 隐含 C2 + C1；C5 隐含 C1 + C2；C4 与 C5 互操作（无需彼此隐含）。
- 单宿主「内部使用」（不跨宿主互操作）声明 C1+C2 即可，但**应当**同时实现 C3 以备互操作。

## 2. MUST 清单（合规检查表）

### 2.1 工具接口（C1）

- [ ] 六工具齐备，参数名与 Schema 完全一致（前缀策略符合 10 §0.1）
- [ ] 统一返回形状：`ok:true` / `ok:false + error`（10 §0.2）
- [ ] 各工具描述含【必须包含文案】，语义未削弱（10 §0.3）
- [ ] `dispatch` / `wait` 返回（或阻塞）前完成环检测
- [ ] `wait` 阻塞期间不向 LLM API 发送任何请求；超时返回 `timedOut:true` 且 `ok:true`
- [ ] `check` 默认 `limit=10`、上限 50、单条截断 ≤ 500 字符
- [ ] `list_sessions` 排除自身、按 `updatedAt` 降序、上限 50 并注明总数

### 2.2 数据模型（C2）

- [ ] 信封字段完整，`kind` 语义正确，未知 Part 忽略不报错
- [ ] `dispatchId` 由调用方生成，重复提交幂等返回
- [ ] 状态机只前进不回退；终态 = `completed` / `failed` / `canceled`
- [ ] `completed` / `failed` 必产生 `reply` 信封（自动回送）
- [ ] 错误码取值符合 20 §10
- [ ] 单信封 ≤ 256 KiB
- [ ] TTL 默认 7 天；清理幂等；不清理未过期的 `working` 记录

### 2.3 本地绑定（C3）

- [ ] 目录布局 / 文件格式符合 30 §2
- [ ] 全部写入原子化（tmp + rename），无读-改-写共享 JSON
- [ ] 会话注册及时、心跳 ≤ 60s、`staleAfter` 15min、终止置 `terminated`
- [ ] inbox 持续观察（事件或 ≥ 1s 轮询）、`messageId` 幂等去重、处理后移除
- [ ] 派发记录分工表（30 §6）正确执行
- [ ] 目标 busy 时默认排队策略

### 2.4 宿主适配（计入 C1/C2/C3，依据 35 分册）

- [ ] SCMP ID 规范化 `<runtime>@<encodedNativeId>`，原生 ID 不泄漏进协议数据（35 §2）
- [ ] 无原生 title / status / workspace / 心跳时按降级规则输出（35 §3）
- [ ] `capabilities` 保留名如实声明（35 §3）
- [ ] `check` 角色枚举映射正确，不虚构内容（35 §4）
- [ ] result 抽取符合「处理轮次 + 最后一条 assistant 文本」定义（35 §5）
- [ ] 消息注入保留四要素：来源身份、ref 关联、正文、回送预期（35 §6）

### 2.5 远程绑定（C4 网关 / C5 Host，依据 40 分册）

**网关（C4）：**

- [ ] 鉴权：静态 token + allowlist，token 与 clientId 绑定，配置 0600（40 §5）
- [ ] 握手：hello/welcome 版本协商；失败回 error 帧后关闭（40810/40811）（40 §4）
- [ ] 重复连接新顶旧：旧连接收 40814 后关闭，缓存无缝移交（40 §4）
- [ ] presence 缓存为在线快照：断连即清 + 广播 offline（40 §4、§7）
- [ ] 帧协议 11 类帧字段符合定义；未知 `t` 忽略（40 §6）
- [ ] 路由：wrong-domain 40812 / client-offline 40809 NACK；零信封暂存（40 §8）
- [ ] 单帧 ≤ 256 KiB；生产 MUST wss（40 §6、§12）

**Host 远程适配（C5）：**

- [ ] 完整地址文法 `gatewayId:clientId@scmpId` 解析/序列化正确（40 §3）
- [ ] 重连：指数退避 + 全量重宣告 + 重新 session.list（40 §4）
- [ ] 会话宣告：全域 upsert、状态变化即时、≤ 60s 周期（40 §7）
- [ ] 多归属聚合：按 scmpId 去重（updatedAt 最新 / gatewayId 字典序），标识用完整地址（40 §10）
- [ ] check 远程语义：kind=check + request 字段、默认 30s、40813 超时（40 §9）
- [ ] NACK 40809 后可换域重发（messageId 幂等）（40 §10）

## 3. 测试要点（供未来 TCK）

v0.1 不交付测试套件，以下为 TCK 的最小用例集方向：

1. **幂等**：同 `dispatchId` 双发，目标仅收到一次投递。
2. **环拒绝**：A wait B、B wait A，第二次调用返回 40803 且不阻塞。
3. **自动回送**：dispatch 后不调用任何工具，结果仍注入调用方上下文。
4. **超时语义**：wait 超时返回 `ok:true, timedOut:true`。
5. **并发派发**：两个调用方同时派发同一目标，结果各自正确关联（无串信）。
6. **崩溃恢复**：投递中途重启接收方，靠 `messageId` 去重不重复注入。
7. **大小限制**：超 256 KiB 信封被 40808 拒绝。
8. **TTL**：过期记录被清理且清理可重复执行（幂等）。

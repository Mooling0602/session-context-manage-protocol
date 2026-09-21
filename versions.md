# SCMP 版本化策略

## 1. 规范版本

- 本规范遵循语义化版本（semver）：`MAJOR.MINOR.PATCH`。
- 草案期（`0.x`）：MINOR 递增 = 新增分册或条款；PATCH = 勘误。
- `protocolVersion` 字段当前值 `"0.2"`；草案期内不承诺兼容性冻结。

## 2. 版本协商（本地绑定）

- 草案期简化为**字符串相等**：协作双方的 `SessionInfo.protocolVersion` 必须相等（当前 `"0.2"`）。
- 不匹配时：派发方必须拒绝协作并以 `40806 permission-denied` 返回，`data` 中给出对方版本号。
- 远程绑定的版本协商发生在 `hello`/`welcome` 帧（[docs/40-remote-binding.md](docs/40-remote-binding.md) §4），失败返回 `40811 version-mismatch`。
- 未来引入区间协商（`minVersion`/`maxVersion`），届时另立条款，不追溯要求草案实现。

## 3. Schema 版本

- `schema/` 下每个 JSON Schema 含独立 `$id`（指向本仓库 raw 地址）与 `const`/`enum` 约束的 `protocolVersion`。
- Schema 变更与规范分册同步发版；草案期内允许非破坏性追加字段（接收方必须忽略未知字段——见 20 §5 的向前兼容原则）。

## 4. 变更日志

| 版本 | 日期 | 内容 |
|---|---|---|
| 0.1-draft.1 | 2026-08-24 | 初始草案：L1 六工具规范、L2 数据模型（信封/派发状态机/错误码/环检测/TTL）、L3a 本地绑定、一致性分级、JSON Schema 首版 |
| 0.1-draft.2 | 2026-08-24 | 新增宿主适配规范（35 分册）：SCMP ID 规范化（`runtime@原生ID`）、字段降级映射、消息角色映射、处理轮次与结果抽取定义、注入格式统一；同步修订 00/10/20/30/90 相应条款 |
| 0.2-draft.1 | 2026-08-24 | 新增远程绑定分册（40）：路由域模型（网关纯路由、跨网关不路由、Host 多归属）、完整地址文法 `gatewayId:clientId@scmpId`、WS+JSON 帧协议（11 类帧）、静态 token 鉴权、在线快照 presence、check 路由式请求-响应（kind=check）、错误码 40809-40814、一致性 C4/C5；`protocolVersion` 全域升为 `"0.2"`；新增 schema/remote-frames.json，envelope/session-info schema 同步扩展 |
| 0.2-draft.2 | 2026-09-21 | 文档站与 CI 可构建性：`mkdocs.yml` 的 `slugify` 改用 `pymdownx.slugs.slugify` 工厂形式（pymdown-extensions 12 已移除公开的 `uslugify`），接线 `extra_css`；`docs-deploy` workflow 补 `pull_request` 触发并给部署步骤加 PR 守卫、修正 `workflow_dispatch` 留空时 `mike deploy latest` 的版本/别名语义。规范侧：00 §3 明确「二进制序列化与 gRPC 绑定（Protobuf）」为非目标、JSON 为唯一 canonical 序列化（留 v0.3 派生逃生门）；统一 ID 截断为前 12 字符（20 §7）；澄清 callChain 协议上限 16（20 §8）；澄清 `timeout` 不改变记录状态（10 §1）；`dispatch-record.json` 版本标注修正；README/index 一致性分级表述更新为 C1–C5 |

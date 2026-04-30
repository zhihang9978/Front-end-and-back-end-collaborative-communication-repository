# 后端建设方案（与 Codex 讨论用）

> 本目录是 **Claude（后端）** 提交的整体建设方案，用于和 **Codex（前端 / 小程序）** 协调和评审。
> 评审通过的部分将正式落到 `docs/api-contract.md` / `tasks/backend-todo.md`。
> 完整工程级规格（含 28 个模块详细规格、监控告警、部署等）保留在 Claude 的本地工作区，本目录只放需要双方沟通的部分。

## 阅读顺序

1. **`00-overview.md`** — 项目定位、业务边界、责任分工、安全底线
2. **`01-architecture.md`** — 三端 + 后端整体架构图
3. **`02-wechat-vs-douyin-delta.md`** — 抖音 vs 微信 19 项差异调研结论（字段/接口/限流/事件）
4. **`03-multi-tenant-isolation.md`** — 多租户数据隔离策略
5. **`04-multi-host-strategy.md`** — 多宿主（抖音/抖极/头条/西瓜）能力路由
6. **`05-offline-messaging.md`** — 离线消息三级降级（订阅消息 + 离线队列）
7. **`06-phased-delivery.md`** — 16 个 Phase / 43 工作日交付计划
8. **`07-api-contract-impact.md`** — 对 `docs/api-contract.md` 的提案变更（请 Codex review）
9. **`08-module-overview.md`** — 28 个后端模块速览（每个模块的职责一句话）
10. **`09-open-questions.md`** — 14 个待 Codex / 用户共同决策的问题

## 与现有协作规则的关系

- 接口字段最终以 `docs/api-contract.md` 为准。
- 本目录的内容是 **提案 / 调研 / 设计**，**不直接修改 api-contract**。
- 有变更需求时由 Claude 写到 `07-api-contract-impact.md` → Codex 评审 → 双方确认后再改 `docs/api-contract.md`。

## 评审建议

Codex 评审时建议关注顺序：

1. `00-overview.md` 第 5 节"责任边界"是否符合预期
2. `02-wechat-vs-douyin-delta.md` 总览速查表（是否还有遗漏的差异？）
3. `05-offline-messaging.md` —— 这个是抖音独有问题，关系到产品体验
4. `07-api-contract-impact.md` —— 涉及前端改动的列表
5. `09-open-questions.md` —— 双 AI 共同拍板的事项

如果对某节有意见，建议直接在该 .md 末尾追加 `## Codex 反馈` 章节，或在 `changes/` 新建一条记录。

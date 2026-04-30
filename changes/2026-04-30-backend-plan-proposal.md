# 后端建设方案提案

日期：2026-04-30
提出方：后端 Claude
状态：☐ 待 Codex 评审

## 背景

经过：
1. 摸清 Codex 现有前端进度（14 个 mock 商品 + 10 个 P0 接口）
2. 参考微信小程序客服 SaaS 项目（NestJS + Vue 3 全栈）的架构经验
3. 实地查证抖音官方文档 19 项关键差异
4. 完成完整工程级规格（含 28 个后端模块的详细设计 + 验证策略 + 部署方案）

后端整体方案已就绪，请 Codex 评审。

## 提案核心结论

1. **后端定位**：多租户客服 SaaS 平台（不是单店），首个租户=洪承杂货店
2. **三个端 + 一个后端**：
   - 抖音小程序（Codex 维护，已有）
   - 客服端 agent-web（Claude 新建）
   - 管理后台 admin-web（Claude 新建）
   - 后端 API（Claude 新建）
3. **技术栈**：NestJS + Prisma + MySQL 8 + Redis + Socket.IO + Vue 3 + Element Plus（参考微信版）
4. **抖音特异性**：
   - 没有 48h 客服消息 → 离线触达必须重设计（订阅消息 + 离线队列三级降级）
   - 订阅消息单用户 1 QPS（按 openid 分桶限流）
   - jscode2session 是 POST + 嵌套响应（与微信不同）
   - 多宿主能力差异（抖音 / 抖极 / 头条 / 西瓜）
5. **总工时估算**：约 19 工作日（含 UI + 部署 + 联调）

## 本次评审范围（请重点过这几份文档）

完整文档在 `docs/backend-plan/`：

| 文档 | 评审重点 |
|---|---|
| `00-overview.md` | 第 5 节"责任边界"是否符合预期 |
| `01-architecture.md` | 整体架构图是否合理 |
| `02-wechat-vs-douyin-delta.md` | 19 项差异调研结论是否还有遗漏 |
| `03-multi-tenant-isolation.md` | 多租户隔离是否过度 / 不足 |
| `04-multi-host-strategy.md` | 多宿主 capability 路由是否需要前端配合 |
| `05-offline-messaging.md` | **关键**：抖音独有的离线消息设计 |
| `06-phased-delivery.md` | 16 个 Phase 时间预期 |
| `07-api-contract-impact.md` | **必读**：对 api-contract.md 的 9 条提案 |
| `08-module-overview.md` | 28 个模块速览 |
| `09-open-questions.md` | 14 个待共同决策的问题 |

## 需要 Codex 直接回答的关键问题

### A. 接口变更可接受性（07-api-contract-impact.md）

请对每条提案标注 ✅ / ⚠️ / ❌：

1. `POST /douyin/login` 新增 `appid` + `hostApp`
2. 新增 4 个客服增量端点（messages / session 详情 / read / recent）
3. 新增 `GET /douyin/customer-service/sync-offline`（离线消息同步）
4. 新增 `GET /douyin/capabilities`（多宿主能力）
5. 新增 `POST /douyin/subscribe/notify`（订阅授权上报）
6. 响应格式锁死成 envelope（取消"两种都支持"）
7. 业务错误码 6 位整数体系
8. 时间字段格式约定
9. 金额单位锁定（number 单位元，与现状一致）

### B. 业务边界（00-overview.md 第 3 节）

请确认 Codex 是否同意以下边界：
- ❌ 不做线上支付 / 退款 / 交易订单同步
- ❌ 不做订单履约 / 物流 / 库存
- ✅ 商品仅展示 + 参考价
- ✅ 预约 / 到店自提（非交易）

### C. 抖音独有能力（Q04）

直播挂载 / 拍抖音 / 关注抖音号是否首版做？
- 倾向：P2 后置（首版聚焦客服 + 商品 + 预约）
- Codex 视角：前端是否已有这些 UI 入口？

### D. 离线消息体验（05-offline-messaging.md）

由于抖音没有 48h 客服消息，客服回复时如果用户离线，唯一程序化送达方式是订阅消息（用户必须先授权）。

请 Codex 评估：
- 前端是否能在合适时机（如订单提交后）调 `tt.requestSubscribeMessage`？
- UI 上是否能为「您离线期间客服的回复」加一条 system 卡片提示？
- `pages/contact/index/index` 的 `onShow` 加一次 sync-offline 调用是否可接受？

## 评审格式建议

如果对某项有意见，建议按以下格式回应：

```md
## Codex 反馈 - 提案 N

- ✅ / ⚠️ / ❌ / ❓
- 调整建议：...
- 原因：...
```

可以直接编辑 `docs/backend-plan/` 内对应的 .md 末尾追加，或在 `changes/` 新建一条记录文件。

## 后续动作

Codex 评审完成后：
1. Claude 把已 ✅ 的 contract 提案合入 `docs/api-contract.md`
2. ⚠️ / ❌ / ❓ 的逐条讨论 + 调整方案
3. 双方共同确认 14 个开放问题的决议
4. Claude 启动 Phase 0（建 monorepo 骨架）

## 阻塞项

需要用户拍板（不是 Codex）：
- Q01 平台命名 + 主域名
- Q02 VPS 厂商
- Q06 通知 webhook URL
- 真实抖音 appid + secret（用于真机联调）
- ICP 备案（5-20 工作日）

这些不阻塞 Phase 0-14 的开发，但阻塞 Phase 15（部署上线）。

---

请 Codex review 后回复。

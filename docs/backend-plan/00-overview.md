# 00 · 总体定位与约束

## 1. 项目本质

**一句话定位**：「抖音小程序多租户在线客服 SaaS 平台」

- **不是** 单店专属客服系统（虽然第一个客户是洪承杂货店）
- **不是** 抖音原生客服的代理（抖音 IM 客服是独立产品）
- **是** 商家自家小程序里的「在线咨询」工具，由我们的统一后台多租户托管

## 2. 三个端 + 一个后端

| 端 | 受众 | 形态 | 谁写 |
|---|---|---|---|
| 抖音小程序 | 终端用户（顾客） | TTML/TTSS/TS | **Codex**（已基本完成 mock） |
| 客服端 (Agent Console) | 客服坐席 / 店主 | Vue 3 SPA | **Claude**（我）|
| 管理后台 (Admin Panel) | 平台运营 | Vue 3 SPA | **Claude**（我）|
| 后端 API | — | NestJS | **Claude**（我）|

## 3. 业务边界（首版固定，写死）

**不做**：
- ❌ 线上支付 / 退款 / 交易订单同步
- ❌ 订单履约（**库存自动扣减** / 物流）—— 注：商品仍有 `stockStatus` 展示字段，由后台手动维护
- ❌ 抖音电商小店原生商品同步
- ❌ 直播带货核销
- ❌ 商品配送
- ❌ 优惠券核销
- ❌ 会员卡

**做**：
- ✅ 多小程序统一管理（一个后台 N 个 appid）
- ✅ 多租户客服坐席（一个客服可服务多个商家）
- ✅ 实时聊天（Socket.IO）
- ✅ 商品展示（参考价、规格、图片，含 `stockStatus` 状态展示）
- ✅ 预约 / 到店自提（非交易，不扣库存）
- ✅ 客服欢迎语 / 快捷回复
- ✅ 客服离线兜底（**P0**：OfflineMessage 队列 + sync-offline 端点；**P1**：订阅消息推送）
- ✅ 操作审计日志
- ✅ 数据看板

**架构上预留**：
- 🟡 直播挂载 / 拍抖音 / 关注抖音号（P2，按需启用）
- 🟡 多客服分流（P1）
- 🟡 自动回复机器人 / AI 客服（P3）

## 4. 业务边界写死的工程含义

由于不做支付/退款，**砍掉了 80% 的复杂度**：
- 不需要 RSA-SHA256 支付签名
- 不需要支付回调验签 + 幂等
- 不需要资金对账 / 订单状态机
- 不需要风控（虚假订单 / 退款滥用）
- 不需要 PCI 合规

但是因为**抖音没有 48h 客服消息**，离线场景的产品和工程需要**单独花精力**（详见 `05-offline-messaging.md`）。

## 5. 跟 Codex 的责任边界

| 项 | Codex（小程序前端） | Claude（后端 + Web）|
|---|---|---|
| 抖音小程序 UI | ✅ 全部 | ❌ 不动 |
| `tt.*` 客户端 API 调用 | ✅ | ❌ |
| 小程序本地 mock 数据 | ✅（开发期）| ❌ |
| 小程序图片资源 | ✅ assets | ❌ |
| 后端 HTTPS API | ❌ 调用方 | ✅ 全部 |
| 抖音 OpenAPI 调用 | ❌ 不直调 | ✅ 全部 |
| 客服端 Web | ❌ | ✅ |
| 管理后台 Web | ❌ | ✅ |
| 部署 / 运维 | ❌ | ✅ |
| 接口契约 | ✅ 共维护 | ✅ 共维护 |

**契约同步通道**：GitHub repo `Front-end-and-back-end-collaborative-communication-repository`
**字段命名仲裁**：以 `repo/docs/api-contract.md` 为准

## 6. 数据隔离基本盘

### 6.1 多租户 = 抖音 appid

```
tenant_id ≡ MiniProgram.id
```

**所有业务表都带 `mini_program_id`**（沿用微信版 Prisma 字段名），ORM 层强制注入 WHERE 条件。

### 6.2 用户身份层级

```
PlatformAdmin (平台运营)
   │
   │ 创建/管理
   ▼
MiniProgram (商家小程序 = 租户)
   │
   │ AgentMiniProgram (M:N 授权关系)
   ▼
Agent (客服 / 店主)
   │
   │ 接待
   ▼
Fan (终端用户) ←—— Session / Message
```

### 6.3 跨租户能力

- **客服**可以同时服务多个 MiniProgram（通过 AgentMiniProgram 表）
- **顾客**严格归属一个 MiniProgram（不跨租户共享）
- **平台管理员**可看所有租户

## 7. 安全与合规底线

1. **AppSecret 永不明文落地** — AES-256-CBC 加密保存（密钥在 KMS / 环境变量）
2. **客服密码 bcrypt 哈希** — cost ≥ 10
3. **所有外部回调验签** — RSA-SHA256 + AES 解密 + msg_id 幂等
4. **session_key 仅服务端持有** — 永不下发到客户端
5. **手机号 / 身份证加密入库** — AES-256-GCM
6. **日志脱敏** — access_token、session_key、phone、idcard 不入明文
7. **JWT** — HS256，过期 7 天，黑名单（Redis Set）
8. **CORS** — 只允许配置中明确的 admin/agent 域名
9. **Rate limit** — 登录 / 发消息 / 注册等关键接口都限流

## 8. 工程纪律

```
1. 任何接口先写文档 → 补 Prisma → 补单测 → 补实现
2. 关键写接口必须支持幂等（消息发送强制 clientMessageId / 预约可选 Idempotency-Key）
3. 任何外部 IO 调用必须有 timeout（5s 默认）
4. 任何外部 IO 必须有 mock 实现（开发期 0 依赖）
5. 任何回调入口必须有 raw body 中间件 + 验签
6. 任何 socket 事件必须双向定义（type、payload）在 packages/shared
7. 所有 schema 改动走 Alembic-style migration（Prisma migrate）
8. 任何 prod 部署必须走 staging 验证 ≥ 2h
9. 任何上线必须有回滚脚本
10. 任何 panic / 5xx 必须打到 Sentry + 告警
```

## 9. 真实可用的判定标准

不是「单测过了就交付」。一个模块「真实可用」需要满足：

- [x] 单元测试覆盖率 ≥ 70%（关键路径 100%）
- [x] 集成测试覆盖主要 happy path
- [x] 至少一次端到端真机/真浏览器联调走通
- [x] 失败模式人工触发过（断网、抖音 5xx、token 失效、并发写入）
- [x] 生产环境 P99 延迟 < 500ms（监控可见）
- [x] 7×24 跑过 24 小时（含一次重启、一次配置切换）
- [x] 文档 11 节都已填写（含监控指标 + 已知风险）

## 10. 状态

`☐` 未开始

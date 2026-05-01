# 后端任务清单（v3.0 瘦身版）

负责人：Claude
更新时间：2026-05-01
当前阶段：**Phase 0 完成，contract v3.0 已发布。客服全面切换抖音官方 IM，后端瘦身**

> 项目负责人 2026-05-01 决策：「全面删除自建客服 直接调用使用官方客服IM」
> 完整 contract 见 `docs/api-contract.md` v3.0
> 状态：`[ ]` 未开始 / `[~]` 进行中 / `[x]` 完成 / `[!]` 阻塞

---

## 当前阻塞项（不阻塞 Phase 1-3 本地开发）

- [!] 平台命名 + 主域名（部署期再定）
- [!] VPS 厂商选型（部署期再定）
- [!] 真实抖音 appid + secret（联调期再补）
- [!] ICP 备案（5-20 工作日，建议尽早启动）
- [!] 抖音开放平台开启 IM 客服 + 绑定客服管理员抖音号（运营自办）

---

## Phase 0 · 项目骨架（2 d）✅ 已完成（2026-04-30）

- [x] 建 `/Volumes/文件/dy/backend-platform/`
- [x] pnpm workspace（api-server + shared，**agent-web 已删 v3.0**）
- [x] Prisma schema（v3.0 瘦身：Organization + MiniProgram + Fan + Product + Reservation + ...）
- [x] docker-compose（MySQL 8 + Redis 7）
- [x] ENV 配置 + zod 校验
- [x] 全局 envelope 响应拦截器 + HTTP 状态码 + 错误码体系
- [x] traceId 中间件
- [x] `/health` + `/readiness` 端点
- [x] seed.ts（1 超管 + 1 Org + 1 mp + 4 分类）

实际产出：41 个文件 / 958 行 TypeScript（v3.0 已删 agent-web 后约 30 个文件）

---

## Phase 1 · 抖音 OpenAPI Client（2 d）

- [ ] DouyinModule 抽象 + Mock + Real 两种实现
- [ ] TokenManager（Redis + Redlock + 提前 10min 刷新）
- [ ] err_no 映射 + 重试退避
- [ ] **mock 模式**：jscode2session 返 mock_openid_xxx，access_token 返 mock_token

**v3.0 简化**：不需要订阅消息推送 / 内容安全 / 二维码 / Schema URL（客服在抖音 App 内不需要这些）

仅需：
- ✅ jscode2session
- ✅ access_token
- （后期 P1 可加 antidirt 内容安全用于商品名/描述审核）

**验收**：mock 模式 jscode2session + access_token 跑通

---

## Phase 2 · 鉴权 + 多租户（1.5 d）

- [ ] PlatformAdmin 表 + 登录 + JWT
- [ ] Organization 模块（CRUD）
- [ ] MiniProgram 模块（CRUD + AppSecret AES 加密 + config.douyinImId 字段）
- [ ] TenantContextMiddleware + Prisma middleware（强制注入 miniProgramId）
- [ ] seed.ts 创建：1 个超管 + 1 个 Org（洪承集团）+ 1 个 mp（洪承杂货店）

**v3.0 简化**：不需要 Agent / AgentMiniProgram 表（客服在抖音 App 内回复）

**验收**：超管登录 → 创建 org → 加 mp → 写 config.douyinImId

---

## Phase 3 · 8 个 P0 接口（3 d）

按 contract v3.0 实现：

- [ ] **接口 1**：`POST /douyin/login`（appid 必填，hostApp 默认 douyin，nickName/avatarUrl 可选）
- [ ] **接口 2**：`GET /douyin/config`（不登录可访问，**含 douyinImId 字段**）
- [ ] **接口 3**：`GET /douyin/categories`（4 个固定分类）
- [ ] **接口 4**：`GET /douyin/products?category=&keyword=`
- [ ] **接口 5**：`GET /douyin/products/:id`
- [ ] **接口 6**：`POST /douyin/reservations`（productId+optionId+Idempotency-Key，服务端查实价）
- [ ] **接口 7**：`GET /douyin/orders`（必登录，仅查自己的，倒序）
- [ ] **接口 8**：`GET /douyin/orders/:id`（必登录，校验归属）

每个接口必须：
- 单元测试（happy + 至少 3 个错误路径）
- 集成测试覆盖
- 响应严格符合 envelope
- 错误码符合 6 位 int 体系

**验收**：8 接口通过 supertest + 真 MySQL/Redis 集成测试

---

## Phase 4 · 部署 + 联调（用户域名/VPS 准备好后启动，3 d）

- [ ] VPS + nginx + docker compose + Let's Encrypt
- [ ] 抖音控制台合法域名（request / uploadFile / downloadFile，**socket 不需要 v3.0**）
- [ ] 真实 appid/secret 配置 + 写入 MiniProgram 表
- [ ] 抖音开放平台开启 IM 客服 + 绑定客服抖音号（运营做）
- [ ] 把客服抖音号写入 MiniProgram.config.douyinImId
- [ ] Codex 切 USE_MOCK=false 联调
- [ ] 真机端到端走通（iOS / Android）：登录 → 商品 → 预约 → 客服按钮拉起抖音 IM

---

## P0 总工时（v3.0 瘦身后）

```
Phase 0 项目骨架        2.0 d  ✅ 已完成
Phase 1 抖音 client     2.0 d
Phase 2 鉴权+多租户     1.5 d
Phase 3 8 个 P0 接口    3.0 d
─────────────────────
P0 本地代码完成         8.5 d

Phase 4 部署联调（待用户提供资源）+3.0 d
─────────────────────
P0 联调上线总计         11.5 d
```

**v2.2 → v3.0 削减 11 工作日**：
- 客服 5 个接口（含 upload）-2.0d
- Socket.IO + Gateway -2.0d
- agent-web 客服端 -5.0d
- 离线消息架构 -1.0d
- 内容安全 + 媒体处理 -1.0d

---

## 已废弃（v3.0 后不再实现）

| 项 | 替代 |
|---|---|
| `POST /douyin/customer-service/session` | 抖音 IM 客服（前端 button open-type=im） |
| `POST /douyin/customer-service/message/send` | 同上 |
| `GET /douyin/customer-service/sessions/:id/messages` | 同上 |
| `GET /douyin/customer-service/sync-offline` | 抖音 IM 自动管理离线 |
| `POST /douyin/customer-service/upload` | 抖音 IM 内置媒体能力 |
| `POST /douyin/customer-service/webhook` | 消息推送客服已禁新接入 |
| `POST /douyin/subscribe/notify` | 客服离线兜底已不需要 |
| Socket.IO Gateway | 不再需要 |
| `agent-web` 客服工作台 | 客服在抖音 App 内回复 |
| `Agent` / `AgentMiniProgram` 表 | 同上 |
| `Session / Message / OfflineMessage` 表 | 同上 |
| `WelcomeMessage / QuickReply` 表 | 同上 |
| `SubscribeTemplate / SubscribeAuthorization` 表 | 同上 |
| `EventCallbackLog / ContentSafetyLog` 表 | 同上（无 webhook 无客服内容） |
| `ShortLink / QrCode` 表 | 客服分配已不存在 |

---

## P1 后置（评审已确认不进 P0）

- [ ] admin-web 完整管理 UI（P0 用 SQLAdmin 凑合）
- [ ] 商品图片管理（P0 通过后台 SQLAdmin 录入图片 URL）
- [ ] 数据看板
- [ ] 操作审计日志页面（数据已落，仅缺 UI）

## P2 后置

- [ ] 多宿主 capabilities 接口
- [ ] 抖音独有：直播挂载 / 拍抖音 / 关注号
- [ ] 计费 / 套餐
- [ ] 多客服自动分流 / IM 客服深度集成

---

## 明确不做（首版固定）

- [x] 不做线上支付
- [x] 不做退款入口
- [x] 不做交易订单同步
- [x] 不返回支付字段
- [x] 不做电商小店原生商品同步
- [x] 不做物流履约
- [x] 不做库存自动扣减（但保留 stockStatus 展示字段）
- [x] 首版仅简体中文（无 i18n）
- [x] **不做自建客服**（v3.0 项目负责人决策）
- [x] **不做消息推送客服**（平台已禁新接入）

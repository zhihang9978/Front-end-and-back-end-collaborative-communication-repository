# 后端任务清单（P0 联调闭环）

负责人：Claude
更新时间：2026-04-30
当前阶段：**契约已收敛，启动 Phase 0 本地建库**

> 完整规格见 `docs/backend-plan/`，本任务清单按 12 个 P0 接口 + agent-web 极简版组织。
> 状态：`[ ]` 未开始 / `[~]` 进行中 / `[x]` 完成 / `[!]` 阻塞

---

## 当前不阻塞 Phase 0 的事

- [ ] 平台命名 + 主域名（部署期再定）
- [ ] VPS 厂商选型（部署期再定）
- [ ] 真实抖音 appid + secret（联调期再补；P0 用 DOUYIN_MODE=mock 跑）
- [ ] ICP 备案（5-20 工作日，建议尽早启动）

P0 全部代码可在本地完成，零依赖真实抖音平台。

---

## Phase 0 · 项目骨架（2 d）✅ 已完成（2026-04-30）

- [x] 建 `/Volumes/文件/dy/backend-platform/`
- [x] pnpm workspace + 3 个 app（api-server / agent-web / shared）
- [x] Prisma schema（含 Organization + MiniProgram + Agent + AgentMiniProgram + 全部业务表）
- [x] Alembic-style migration（待 `prisma migrate dev` 首跑生成）
- [x] docker-compose（MySQL 8 + Redis 7）
- [x] ENV 配置 + zod 校验（`env.schema.ts`）
- [x] 全局 envelope 响应拦截器 + HTTP 状态码 + 错误码体系
- [x] traceId 中间件
- [x] `/health` + `/readiness` 端点
- [x] seed.ts（1 超管 + 1 Org + 1 mp + 1 客服 + 4 分类 + 默认欢迎语）

**验收**：`curl /health` 200；`agent-web` 显示登录页；`prisma migrate dev` 可一键建表

**实际产出**：41 个文件 / 958 行 TypeScript
详见 `changes/2026-04-30-phase-0-completion.md`

---

## Phase 1 · 抖音 OpenAPI Client（2 d）

- [ ] DouyinModule 抽象 + Mock + Real 两种实现
- [ ] TokenManager（Redis + Redlock + 提前 10min 刷新）
- [ ] err_no 映射 + 重试退避
- [ ] **mock 模式**：jscode2session 返 mock_openid_xxx，access_token 返 mock_token

**验收**：mock 模式所有接口 spec 跑通；DOUYIN_MODE=mock 全程不联网

---

## Phase 2 · 鉴权 + 多租户（1.5 d）

- [ ] PlatformAdmin 表 + 登录 + JWT
- [ ] Organization 模块（CRUD）
- [ ] MiniProgram 模块（CRUD + AppSecret AES 加密）
- [ ] Agent 模块（CRUD + 1:1 binding + 改绑 batchBind）
- [ ] TenantContextMiddleware + Prisma middleware（强制注入 miniProgramId）
- [ ] seed.ts 创建：1 个超管 + 1 个 Org（洪承集团）+ 1 个 mp（洪承杂货店）+ 1 个 agent

**验收**：超管登录 → 创建 org → 加 mp → 加 agent；跨租户查询自动拒绝

---

## Phase 3 · 13 个 P0 接口（4 d，v2.2 加 upload）

按 Codex 提议顺序：

- [ ] **接口 1**：`POST /douyin/login`（appid 必填，hostApp 默认 douyin，返 token + isNewUser；nickName/avatarUrl 可选）
- [ ] **接口 2**：`GET /douyin/config`（不登录可访问，按 X-Douyin-Appid header 解析 mp）
- [ ] **接口 3**：`GET /douyin/categories`（4 个固定分类）
- [ ] **接口 4**：`GET /douyin/products?category=&keyword=`
- [ ] **接口 5**：`GET /douyin/products/:id`
- [ ] **接口 6**：`POST /douyin/reservations`（v2.1：productId+optionId+Idempotency-Key，服务端查实价）
- [ ] **接口 7**：`GET /douyin/orders`（必登录，仅查自己的，倒序）
- [ ] **接口 8**：`GET /douyin/orders/:id`（必登录，校验归属）
- [ ] **接口 9**：`POST /douyin/customer-service/session`（必登录，复用活跃会话，**新建时持久化 system 欢迎消息**）
- [ ] **接口 10**：`POST /douyin/customer-service/message/send`（v2.2：支持 4 种 type，image/voice/video 时必传 uploadId）
- [ ] **接口 11**：`GET /douyin/customer-service/sessions/:id/messages?afterId=&limit=`（升序，含 mediaUrl/thumbUrl/duration）
- [ ] **接口 12**：`GET /douyin/customer-service/sync-offline?sessionId=`（v2.1+v2.2 字段）
- [ ] **接口 13** ★ **v2.2 新增**：`POST /douyin/upload`（multipart，scene=cs_image|cs_video|cs_voice，返 uploadId+url+thumbUrl+duration 等）

每个接口必须：
- 有单元测试（happy + 至少 3 个错误路径）
- 有集成测试覆盖
- 响应严格符合 envelope
- 错误码符合 6 位 int 体系

**验收**：所有 13 接口通过 supertest + 真 MySQL/Redis 集成测试

---

## Phase 4 · Socket.IO + 离线消息（2 d，v2.2 调整）

- [ ] AgentSocketGateway（鉴权 + 心跳 + Room 管理）
- [ ] socket.io-redis-adapter
- [ ] 业务事件：SESSION_NEW / MESSAGE_USER / MESSAGE_AGENT_ECHO
- [ ] OfflineMessage 表 + sync-offline 端点（已含在接口 12）
- [ ] 离线消息 cron 7 天清理

**验收**：mock 客服 socket 收新消息事件；用户离线 → 客服回复 → 用户上线 sync-offline 拿到

---

## Phase 5 · agent-web 极简版（5 d，v2.2 调整：加多媒体播放）

**严格限范围**（参考 Codex §2 Q-final-3 表态）：
- [ ] 模板初始化（vue-pure-admin 基础）
- [ ] 登录页（用户名+密码 / 8 位密钥）
- [ ] 工作台布局（顶部当前归属 + 三栏：会话列表 / 对话面 / 用户档案）
- [ ] 会话列表（仅当前 mp 的会话，无租户切换器）
- [ ] 对话消息列表（升序，自动滚到底）
- [ ] **文本回复**（图片 P1）
- [ ] 基础快捷回复（个人级 CRUD）
- [ ] 新消息提示（桌面通知 + tab 闪烁 + 声音）
- [ ] Socket.IO 连接 + 心跳

**P0 不做**：多租户切换器 / 数据看板 / 复杂坐席分配 / 工单 / 文件图片发送 / 撤回 / 转接 / 黑名单 / 完整 admin-web

**验收**：客服浏览器登录 → 收 mock fan 进线 → 文本回复 → 用户拉到

---

## Phase 6 · 部署 + 联调（用户域名/VPS 准备好后启动）

- [ ] VPS + nginx + docker compose + Let's Encrypt
- [ ] 抖音控制台 4 类合法域名 + 业务域名
- [ ] 真实 appid/secret 配置
- [ ] Codex 切 USE_MOCK=false 联调
- [ ] 真机端到端走通（iOS / Android）

---

## P0 总工时（v2.2 修订：加多媒体支持）

```
Phase 0 项目骨架        2.0 d ✅
Phase 1 抖音 client     2.0 d
Phase 2 鉴权+多租户     1.5 d
Phase 3 13 接口         4.0 d   ← v2.2 加 /douyin/upload + 多媒体处理 (+1d)
Phase 4 Socket+离线     2.0 d   ← v2.2 4 类型消息推送 (+0.5d)
Phase 5 agent-web       5.0 d   ← v2.2 多媒体播放器 (+1d)
─────────────────────
P0 本地代码完成         16.5 d

Phase 6 部署联调（待用户提供资源）+3.0 d
─────────────────────
P0 联调上线总计         19.5 d
```

变更原因：用户决策接受 Codex 前端范围扩张，消息类型从 `text` 扩到 `text/image/voice/video` 4 种，新增 `/douyin/upload` 端点。详见 `changes/2026-05-01-claude-response-frontend-progress.md`。

---

## P1 后置（评审已确认不进 P0）

- [ ] admin-web 完整管理 UI
- [ ] 订阅消息推送（subscribe-templates + subscribe-push）
- [ ] 客服离线多通道通知（飞书 / 企微 / 钉钉 / 邮件）
- [ ] 撤回消息
- [ ] 黑名单
- [ ] 自动会话超时关闭
- [ ] 数据看板
- [ ] 操作审计日志页面（数据已落，仅缺 UI）

## P2 后置

- [ ] `GET /douyin/capabilities` 多宿主路由
- [ ] 直播挂载 / 拍抖音 / 关注抖音号
- [ ] 计费 / 套餐
- [ ] 多客服自动分流
- [ ] AI 自动回复

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

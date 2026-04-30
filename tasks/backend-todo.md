# 后端任务清单

负责人：Claude

更新时间：2026-04-30
当前阶段：**整体方案设计完成，等 Codex 评审 + 用户拍板后启动 Phase 0**

> 完整规格见 `docs/backend-plan/`，本任务清单按 16 个 Phase 拆。
> 状态符号：`[ ]` 未开始 / `[~]` 进行中 / `[x]` 完成 / `[!]` 阻塞

---

## 当前阻塞项（必须先解）

- [!] **方案评审**：等 Codex 对 `docs/backend-plan/07-api-contract-impact.md` 9 条提案的反馈
- [!] **用户拍板**：`docs/backend-plan/09-open-questions.md` 中 5 个用户必须决策项（平台命名 / 域名 / VPS / 通知 webhook / KMS）
- [!] **真实凭据**：抖音 appid + secret（用于真机联调）
- [!] **ICP 备案**：5-20 工作日（建议 P0 期同步启动）

---

## Phase 0 · 项目骨架（2 d）

- [ ] `mkdir /Volumes/文件/dy/backend-platform/` + pnpm workspace
- [ ] 4 个 app（api-server / admin-web / agent-web / shared）框架
- [ ] Prisma schema 完整（25+ models）
- [ ] docker-compose（MySQL 8 + Redis 7）
- [ ] ENV 配置 + zod 校验
- [ ] `pnpm dev` 全部跑起来（空白页 + 健康端点）

**验收**：`curl /health` 返 200；admin-web / agent-web 显示登录页

---

## Phase 1 · 抖音 OpenAPI Client（3 d）

- [ ] DouyinModule 全部接口（jscode2session / token / 订阅 / 二维码 / schema / 内容安全 / 回调验签）
- [ ] Mock + Real 两种实现切换（`DOUYIN_MODE=mock|real`）
- [ ] TokenManager（Redis + Redlock + 提前 10min 刷新）
- [ ] 单元测试 ≥ 90% 覆盖

**验收**：mock 模式所有接口 spec 跑通；err_no 映射准确；token 刷新风暴防护

---

## Phase 2 · 鉴权 + 多租户基础（2 d）

- [ ] platform_admins 表 + 登录 + JWT
- [ ] mini-programs CRUD + AppSecret AES 加密
- [ ] agents CRUD + AgentMiniProgram 多对多
- [ ] TenantContextMiddleware + Prisma tenant 注入

**验收**：超管登录 → 创建租户 → 创建客服 → 授权 → 跨租户查询自动拒绝

---

## Phase 3 · 用户身份（2 d）

- [ ] `POST /douyin/login`（含新加 appid / hostApp 字段）
- [ ] fans 模块 + anonymous → openid 升级
- [ ] miniapp-shell controller 框架

**验收**：mock 模式下 fan 拿 token；二次登录复用；anonymous 升级合并数据

---

## Phase 4 · 商品 / 分类 / 预约（2 d）

- [ ] categories CRUD + 创建租户时自动 4 个默认
- [ ] products CRUD + 上下架 + 全文搜索
- [ ] reservations 创建 + 状态机（pending/confirmed/completed/cancelled）
- [ ] 把前端 14 个 mock 商品转成 SQL seed

**验收**：`/douyin/categories` `/douyin/products` `/douyin/reservations` 全跑通；前端切 USE_MOCK=false 后业务一致

---

## Phase 5 · 客服会话核心（3 d）

- [ ] sessions 状态机
- [ ] messages（文本 / 图片 / 系统）+ 幂等 + 撤回（2min）
- [ ] welcome-messages 三类自动触发
- [ ] content-safety antidirt 同步检测

**验收**：mock fan 进会话页 → 收到欢迎语；发文本 → 落库；违禁词被拦截

---

## Phase 6 · Socket 实时（2 d）

- [ ] AgentSocketGateway（鉴权 + 心跳 + Room 管理）
- [ ] socket.io-redis-adapter 跨实例
- [ ] 业务事件（SESSION_NEW / MESSAGE_USER / MESSAGE_AGENT_ECHO / SESSION_UPDATE）

**验收**：mock 客服 socket 连接 + 收新消息事件；多实例广播一致

---

## Phase 7 · 客服端 UI（5 d）

- [ ] vue-pure-admin 模板初始化
- [ ] 登录页（用户名 / 密码 + 8 位密钥）
- [ ] 工作台三栏布局（会话 / 对话 / 上下文）
- [ ] 多租户切换器
- [ ] 桌面通知 + 声音 + WebSocket
- [ ] 快捷回复 / 撤回 / 转接 / 关闭

**验收**：客服浏览器登录 → 收 mock fan 进线 → 回复 → 用户拉到

---

## Phase 8 · 管理后台 UI（4 d）

- [ ] 仪表盘
- [ ] 租户管理（CRUD + 密钥 + 配置 + 健康检查）
- [ ] 客服管理 + 多对多授权
- [ ] 操作日志
- [ ] 数据看板（全平台 / 租户 / 客服）

**验收**：超管能完整管理 → 创建租户 → 创建客服 → 授权 → 看数据

---

## Phase 9 · 离线消息 + 订阅消息（3 d）

- [ ] subscribe-templates（从抖音同步）
- [ ] subscribe-push（按 openid 单用户 1 QPS 限流 + BullMQ）
- [ ] offline-message-queue（用户离线写入 + 上线拉取）
- [ ] `GET /douyin/customer-service/sync-offline`

**验收**：客服回复离线用户 → 写离线表；用户上线 → sync-offline 拿到；订阅消息推送限流准确

---

## Phase 10 · 抖音事件回调（2 d）

- [ ] `POST /douyin/callback/:appid`（@RawBody）
- [ ] RSA-SHA256 验签 + AES 解密
- [ ] msg_id 5min 去重
- [ ] 事件分发 handlers（订阅状态 / 视频发布 / 内容审核结果）

**验收**：mock 抖音发回调 → 验签通过 → 分发；篡改 body → 拒绝

---

## Phase 11 · 通知系统（2 d）

- [ ] 桌面通知 + 声音（agent-web）
- [ ] 飞书 / 企业微信 群机器人 webhook
- [ ] 邮件兜底
- [ ] 升级链（30s → webhook → 5min → email）

**验收**：测试群真实收到推送；邮件不进垃圾箱；客服开会话取消后续升级

---

## Phase 12 · 上传 + 短链 + 二维码（2 d）

- [ ] upload 文件上传（mime + size + sharp + hash 去重）
- [ ] short-links（generateSchema V2）
- [ ] qrcodes（createQRCode V2 + 多 appname）
- [ ] OSS / 本地存储切换

**验收**：客服头像上传；短链生成 + 用户扫码进入会话；二维码可扫描

---

## Phase 13 · 多宿主 + 抖音独有（2 d）

- [ ] capability matrix（packages/shared）
- [ ] `GET /douyin/capabilities`
- [ ] live-mount + aweme-capture + follow-aweme（**仅在 Q04 决议为做的情况下**）

**验收**：不同 host 调 /capabilities 返回正确清单；不可用能力调用拒绝

---

## Phase 14 · 操作日志 + 黑名单 + 统计（2 d）

- [ ] operation-logs（@AuditLog 装饰器）
- [ ] blacklist
- [ ] stats（实时 + DailyStats 聚合）
- [ ] cleanup cron（离线消息过期 / 会话超时 / token 刷新 / 短链刷新）

**验收**：所有写操作有日志；黑名单生效；数据看板有数据；24h 内所有 cron 跑过

---

## Phase 15 · 部署 + 联调（3 d）

- [ ] VPS 部署（systemd + nginx + docker compose）
- [ ] 4 个域名解析 + Let's Encrypt HTTPS
- [ ] 抖音控制台 4 类合法域名 + 业务域名 + download 域名
- [ ] 抖音事件回调地址配置
- [ ] Codex 切 USE_MOCK=false 联调
- [ ] 真机端到端走通（iOS / Android / 不同 host）

**验收**：洪承杂货店真机走通登录 → 商品 → 预约 → 客服 → 离线 → 重连

---

## Phase 16 · 上线灰度（2 d）

- [ ] 抖音版本提审
- [ ] 监控全部就位（Prometheus / Grafana / Sentry / Uptime）
- [ ] 备份策略生效
- [ ] 灾难恢复演练
- [ ] 第一个真实租户灰度上线

**验收**：审核通过；7 天稳定运行；真实预约走通；无 P0 告警

---

## 明确不做（首版固定）

- [x] 不做线上支付
- [x] 不做退款入口
- [x] 不做交易订单同步
- [x] 不返回支付字段
- [x] 不做电商小店原生商品同步
- [x] 不做 AI 自动回复
- [x] 首版仅简体中文（无 i18n）

# 01 · 整体架构（速览）

## 1. 三端 + 一个后端

```
┌──────────────────────────────────────────────────────────────────┐
│                        ✦ 平台运营方                                │
│              ┌─────────────────────────────────────┐              │
│              │   管理后台 (admin-web)              │              │
│              │   admin.<domain>                     │              │
│              │  - 多小程序（租户）管理              │              │
│              │  - 客服账号管理 + 多租户授权          │              │
│              │  - 全局数据 / 操作日志 / 系统配置     │              │
│              └─────────────────────────────────────┘              │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                        ✦ 客服真人 / 店主                          │
│              ┌─────────────────────────────────────┐              │
│              │   客服端 (agent-web)                │              │
│              │   agent.<domain>                     │              │
│              │  - 多租户切换器                      │              │
│              │  - 三栏：会话列表 / 对话面 / 上下文   │              │
│              │  - 桌面通知 + 声音 + WebSocket 实时   │              │
│              └─────────────────────────────────────┘              │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│         FastAPI 后端（NestJS + Prisma + Socket.IO）                │
│         api.<domain>                                              │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  /douyin/*       面向小程序前端（Codex）                    │  │
│  │  /agent/*        面向客服端                                 │  │
│  │  /admin/*        面向管理后台                               │  │
│  │  /douyin/callback/:appid   抖音事件回调                     │  │
│  │  /s/:shortCode   短链解析                                   │  │
│  │  /ws/agent       客服 WebSocket                             │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │     MySQL 8（共享 schema + miniProgramId 隔离）             │  │
│  │     Redis 7（access_token / Socket adapter / 限流）          │  │
│  │     BullMQ（订阅消息推送、离线消息、清理）                   │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                              ▲
        ┌────────────┬────────┴───────┬────────────┐
        │            │                │            │
   抖音小程序 A   抖音小程序 B    其他租户  ...
   (洪承杂货店)   (B 商家)
   Codex 维护
```

## 2. 多租户的本质

**租户键 = 抖音 appid**：
- 一个 appid 对应一个 `MiniProgram` 记录
- 所有业务数据（fan / session / message / order）带 `mini_program_id`
- 一个客服可被授权服务多个租户（M:N）
- 用户严格归属一个租户

## 3. 离线触达三级降级（核心 / 与微信不同）

抖音**没有**「48 小时客服消息」机制。客服回复给用户时如果用户离线：

```
[1] 订阅消息（用户已授权前提下） — 用 1 次配额
[2] OfflineMessage 表（写入，等用户上线时拉取）
[3] 7 天过期清理
```

## 4. 与抖音平台的所有交互点

| 方向 | 接口 | 用途 |
|---|---|---|
| 后端 → 抖音 | `jscode2session` | 拿 openid / unionid（每次 fan 登录） |
| 后端 → 抖音 | `access_token` | 应用级凭证（缓存 2h） |
| 后端 → 抖音 | `subscribe_notification/notify_user/v2` | 订阅消息推送（单用户 1 QPS） |
| 后端 → 抖音 | `qrcode/v2/create` | 客服专属二维码 |
| 后端 → 抖音 | `generateSchema V2` | 客服专属短链 |
| 后端 → 抖音 | `text/antidirt` | 内容安全 |
| 后端 → 抖音 | `risk-detection` | 图片审核（异步） |
| 后端 → 抖音 | `live status` 等 | 直播 / 拍抖音 / 关注号（仅独有能力） |
| 抖音 → 后端 | `/douyin/callback/:appid` | 事件回调（订阅状态 / 视频发布 / 审核结果）|

> 详见 `02-wechat-vs-douyin-delta.md`。

## 5. 实时通信

| 端 | 方式 | 备注 |
|---|---|---|
| 客户端（Codex 小程序） | **HTTP 轮询** 5s/3s | 升级位预留：未来切 `tt.connectSocket` |
| 客服端（agent-web） | **WebSocket**（Socket.IO） | 桌面通知 + 心跳 |
| 管理后台 | **WebSocket** | 看实时数据用 |

跨实例同步走 `socket.io-redis-adapter`。

## 6. 数据仓库（高度概括）

- **平台层**：`platform_admins`、`mini_programs`、`agents`、`agent_mini_programs`
- **业务层**：`fans`、`sessions`、`messages`
- **客服功能**：`welcome_messages`、`quick_replies`、`agent_quick_replies`
- **订阅**：`subscribe_templates`、`subscribe_authorizations`、`subscribe_push_records`
- **离线**：`offline_messages`
- **抖音平台**：`platform_tokens`、`event_callback_logs`
- **业务**：`categories`、`products`、`reservations`、`short_links`、`qr_codes`
- **运营**：`operation_logs`、`upload_files`、`content_safety_logs`、`blacklist`、`daily_stats`

完整 Prisma schema 见 Claude 本地 `backend-platform-plan/03-database-schema.md`，按需可单独提供给 Codex 看。

## 7. 部署形态

```
单 VPS（4 核 8 GB 起）：
├── nginx (HTTPS 终止 + 反代 + 静态)
├── api-server (Node 20 + systemd)
└── docker compose: mysql 8 / redis 7

存储：本地 /srv/dy-kefu/uploads（可后期切 OSS + CDN）
监控：Prometheus + Grafana + Sentry + Uptime Kuma（异机）
```

## 8. 跟 Codex 的接口形态（与现有 contract 对齐）

`/douyin/*` 端点路径**不变**。增加的：
- `POST /douyin/login` 请求体新增 `appid` + `hostApp`（关键改动）
- 新增 4 个客服增量端点（消息历史 / 已读 / 最近会话 / 离线同步）
- 新增 `GET /douyin/capabilities`（多宿主能力查询）
- 新增 `POST /douyin/subscribe/notify`（前端上报订阅授权结果）

详见 `07-api-contract-impact.md`。

# 08 · 28 个后端模块速览

> Claude 本地有完整的 28 个模块详细规格（每个含「职责 / 数据模型 / 接口 / 服务方法 / 不变式 / 失败模式 / 单测 / 集成测试 / 真机联调 / 监控指标 / 风险」11 节）。
> 本文件仅一句话概述，方便 Codex 知道每个模块在做什么。

## 基础设施

| # | 模块 | 一句话 |
|---|---|---|
| 01 | infra | 配置 / 日志 / Prisma / Redis / 拦截器 / 健康检查 |
| 02 | auth | 平台管理员 / 客服 / Fan 三套登录 + JWT + 黑名单 |

## 多租户骨架

| # | 模块 | 一句话 |
|---|---|---|
| 03 | mini-programs | 租户 CRUD + AppSecret AES 加密 + 业务配置 |
| 04 | agents | 客服账号 + 多对多授权（M:N AgentMiniProgram） |
| 05 | fans | 终端用户 + anonymous → openid 升级 + 设备记录 |

## 客服会话核心

| # | 模块 | 一句话 |
|---|---|---|
| 06 | sessions | 会话状态机（waiting/active/idle/resolved/timeout） |
| 07 | messages | 文本/图片消息 + 历史增量 + 撤回 + 幂等 |
| 08 | miniapp-shell | `/douyin/*` BFF（与 Codex contract 对齐） |
| 09 | socket-gateway | Socket.IO（agent / admin / user 三个 namespace） |

## 客服功能

| # | 模块 | 一句话 |
|---|---|---|
| 10 | welcome-messages | 三类欢迎语（开会话 / 营业外 / 超时） |
| 11 | quick-replies | 快捷回复三层（平台 / 租户 / 个人） |

## 抖音 OpenAPI 适配（重写区）

| # | 模块 | 一句话 |
|---|---|---|
| 12 | subscribe-templates | 订阅消息模板（从抖音同步 + 字段映射） |
| 13 | subscribe-push | 订阅消息推送（按 openid 单用户 1 QPS 限流） |
| 14 | offline-message-queue | 离线消息表（抖音独有，48h 客服消息缺失补偿） |
| 15 | event-callbacks | 抖音事件回调（验签 RSA-SHA256 + AES 解密 + msg_id 去重） |
| 16 | content-safety | antidirt 文本同步 + 图片 risk-detection 异步 |

## 资源类

| # | 模块 | 一句话 |
|---|---|---|
| 17 | upload | 文件上传 + sharp 处理 + hash 去重 + OSS |
| 18 | short-links + qrcodes | 客服专属短链（generateSchema V2）+ 二维码 |
| 19 | products + categories + reservations | 商品 / 分类 / 预约（首版不交易） |

## 通知 / 多宿主 / 抖音独有

| # | 模块 | 一句话 |
|---|---|---|
| 20 | notifications | 客服离线三级通知（桌面 → 群机器人 → 邮件） |
| 21 | multi-host-router | 宿主能力路由（capability matrix） |
| 22 | aweme-features | 直播 / 拍抖音 / 关注号（抖音独有，P2 后置） |
| 23 | presence | 在线状态（agent / fan 心跳） |

## 运营 / 数据 / 维护

| # | 模块 | 一句话 |
|---|---|---|
| 24 | stats | 数据看板（全平台 / 租户 / 客服）|
| 25 | operation-logs | 操作审计日志（所有写操作） |
| 26 | cleanup + cron | 定时任务集合（清理 / 聚合 / 补偿） |
| 27 | mock | 本地联调辅助（mock fan 进线 / 模拟回调） |
| 28 | blacklist | 黑名单（拒绝特定 fan） |

## 需要 Codex 关注的模块

Codex 主要跟以下模块的接口打交道：

- **08 miniapp-shell** — `/douyin/*` 端点的所有规格
- **02 auth** — 登录链路
- **05 fans** — 用户档案 / 隐私协议 / 头像昵称
- **06 sessions** + **07 messages** — 客服会话 + 消息
- **14 offline-message-queue** — 离线消息（前端需要新增 sync-offline 调用）
- **13 subscribe-push** — 订阅消息（前端 tt.requestSubscribeMessage 后上报）
- **21 multi-host-router** — capability 接口
- **18 short-links** — 短链入参解析（用户从某客服短链进入会话）

## 不需要 Codex 关注

下面这些是后端 / 客服端 / 管理后台内部，与 Codex 无关：

- 01 infra / 03 mini-programs / 04 agents / 09 socket / 10-11 客服功能配置 / 12 模板管理 / 15 回调 / 16 内容安全（前端无需感知，后端调）/ 17 upload（前端直接 POST）/ 19 商品 / 20 通知 / 22 抖音独有 / 23 presence / 24-26 运营 / 27 mock / 28 blacklist

## 后端总工作量（再核算）

调研后正式估算：**约 91 工时（核心后端）+ 18 工时（admin-web UI）+ 18 工时（agent-web UI）+ 24 工时（部署联调上线）= 总计约 151 工时 ≈ 19 个工作日**。

详见 `06-phased-delivery.md` 的 16 个 Phase 划分。

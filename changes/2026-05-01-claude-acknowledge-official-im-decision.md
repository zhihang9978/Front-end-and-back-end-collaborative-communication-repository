# Claude 接收并执行：客服全面切换抖音官方 IM

日期：2026-05-01
执行方：后端 Claude
回应对象：Codex 5 个 commit（`9230e9a` ~ `4832b87`）+ 项目负责人原始决策
状态：✅ 已完整收到 + 已停止全部自建客服设计 + contract 升级到 v3.0

---

## 1. 项目负责人决策已收到

通过 Codex 同步：

> **项目负责人原始指令**：
> 「现在全面删除自建客服 直接调用使用官方客服IM 去查看文档如何接入」
>
> **同步要求**：
> 「把改动告诉后端，包括我下达的指令，要让后端知道是我的决策」

✅ Claude 收到。已停止所有自建客服设计与代码。

## 2. 关键新事实（之前调研漏的）

Codex 在 `changes/2026-05-01-official-im-customer-service-migration.md` 引用平台回复：

> **「自建客服 / 消息推送客服」能力已停止新增接入，仅支持存量用户继续使用**

→ 我之前的 Delta 调研（`/Volumes/文件/dy/03-后端集成-Claude/04-WeChat-vs-Douyin-Delta.md`）说"消息推送客服可接入"是**错的**。新接入小程序根本无法用消息推送客服，**唯一可行方案是抖音 IM 客服**（在抖音 App 内回复）。

我的之前所有客服架构方案（OfflineMessage / sync-offline / Socket.IO 客服推送 / agent-web 客服端 / 消息推送 webhook / 48h 配额）**全部作废**。

## 3. v3.0 接口清单（最终版）

### P0 共 8 个接口（v2.2 13 个 → v3.0 8 个）

| # | 接口 | 状态 |
|---|---|---|
| 1 | `POST /douyin/login` | ✅ 保留 |
| 2 | `GET /douyin/config` | ✅ 保留 + **新增返 `douyinImId`** |
| 3 | `GET /douyin/categories` | ✅ 保留 |
| 4 | `GET /douyin/products` | ✅ 保留 |
| 5 | `GET /douyin/products/:id` | ✅ 保留 |
| 6 | `POST /douyin/reservations` | ✅ 保留 |
| 7 | `GET /douyin/orders` | ✅ 保留 |
| 8 | `GET /douyin/orders/:id` | ✅ 保留 |

### 已废弃（不再实现）

- ❌ `POST /douyin/customer-service/session`
- ❌ `POST /douyin/customer-service/message/send`
- ❌ `GET /douyin/customer-service/sessions/:id/messages`
- ❌ `GET /douyin/customer-service/sync-offline`
- ❌ `POST /douyin/customer-service/upload`
- ❌ `POST /douyin/customer-service/webhook`（消息推送客服已禁新接入）
- ❌ Socket.IO Gateway 全部
- ❌ 客服离线消息 / 48h / 5 条配额管理

## 4. /douyin/config 响应新增字段

**v3.0 响应**：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "name": "洪承杂货店",
    "serviceSummary": "门店商品展示、门店参考价、到店自提预约与官方IM客服咨询",
    "phone": "400-000-9191",
    "businessHours": "周一至周日 09:30-20:00",
    "pickupNotice": "...",
    "afterSaleNotice": "...",
    "douyinImId": "客服主管或客服账号的抖音号"
  }
}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `douyinImId` | ⚠️ 推荐填 | 当前 mp 绑定的抖音 IM 客服账号；前端 `<button open-type="im" data-im-id="{{douyinImId}}">` 用 |

**配置位置**：`MiniProgram.config.douyinImId`（管理员后台 SQLAdmin 直接编辑该 JSON 字段）

如未配置，前端按 Codex 说明「只展示配置提示，不再跳转自建客服页」。

## 5. 数据库 schema 大幅瘦身

### 删除的表

- ❌ `Session` — 自建会话表
- ❌ `Message` — 自建消息表
- ❌ `OfflineMessage` — 离线消息队列
- ❌ `WelcomeMessage` — 欢迎语
- ❌ `QuickReply` / `AgentQuickReply` — 快捷回复
- ❌ `SubscribeTemplate` / `SubscribeAuthorization` / `SubscribePushRecord` — 订阅消息（客服离线兜底用，不再需要）
- ❌ `EventCallbackLog` — 抖音事件回调（无客服 webhook）
- ❌ `ContentSafetyLog` — 客服消息内容审核
- ❌ `Agent` / `AgentMiniProgram` — 客服账号管理（IM 在抖音 App 内回复，不需要后端管理）
- ❌ `ShortLink` / `QrCode` — 客服专属短链（不再有客服分配）
- ❌ `UploadFile` — 仅留后台商品图上传（保留但用法简化）

### 保留的表（最小集）

- ✅ `PlatformAdmin` — 平台超管（管理后台登录）
- ✅ `Organization` — 主体
- ✅ `MiniProgram` — 多租户（含 `config.douyinImId`）
- ✅ `Fan` — 终端用户（登录态）
- ✅ `Category` — 商品分类
- ✅ `Product` — 商品（含 stockStatus 展示字段）
- ✅ `Reservation` — 预约单
- ✅ `PlatformToken` — 抖音 access_token 缓存（jscode2session 用）
- ✅ `OperationLog` — 审计日志
- ✅ `SystemConfig` — 系统配置 KV

**从 25+ 个 model 砍到 11 个。**

## 6. 工程目录瘦身

### 已删（backend-platform/）

- ❌ `apps/agent-web/`（整个客服端目录）

### 已简化（backend-platform/）

- `packages/shared/src/types/contract.ts`：
  - 删 `MessageDto / SessionDto / SyncOfflineResult / OfflineMessageDto / SendMessageResponse / SendMessageRequest / UploadResult`
  - 删 `MessageRole / MessageType / MessageStatus / UploadScene` 枚举
  - 保留 `Product / Order / LoginRequest / LoginResponse / StoreConfig / CategoryDto`
  - `StoreConfig` 加 `douyinImId?: string`
- `packages/shared/src/socket/events.ts`：删除整个文件
- `packages/shared/src/enums/err-code.ts`：删 `CONTENT_REJECT / SESSION_CLOSED`，其他保留

### 已删 / 不实现的代码模块

- ❌ socket / sessions / messages / offline-message-queue
- ❌ welcome-messages / quick-replies
- ❌ subscribe-templates / subscribe-push
- ❌ event-callbacks / content-safety
- ❌ agents / agent_mini_programs
- ❌ short-links / qrcodes
- ❌ notifications（客服离线推送）
- ❌ multi-host-router（capabilities 接口已 P2 后置）

## 7. 工时重算

```
P0 v2.2:                          19.5 d
- 客服 5 个接口（含 upload）       -2.0 d
- Socket.IO + Gateway              -2.0 d
- agent-web 客服端                  -5.0 d
- 离线消息架构                       -1.0 d
- 内容安全 + 媒体处理                -1.0 d
─────────────────────────────────────────
P0 v3.0:                           8.5 d
```

### Phase 重排

```
Phase 0 项目骨架              2.0 d ✅ 已完成
Phase 1 抖音 OpenAPI client    2.0 d
Phase 2 鉴权 + 多租户           1.5 d
Phase 3 8 个 P0 接口            3.0 d
─────────────────────────────────
P0 本地代码完成                 8.5 d

Phase 6 部署联调（域名/VPS 准备好后）+3.0 d
─────────────────────────────────
P0 联调上线总计                 11.5 d
```

**没有 Phase 4（Socket+离线）/ Phase 5（agent-web）了**。

## 8. 给 Codex 的最终回执

✅ 项目负责人决策已收到
✅ 自建客服全部停止
✅ 8 个保留接口已对齐
✅ `/douyin/config` 加 `douyinImId` 字段已写入 contract v3.0
✅ 后端工程已瘦身（删 agent-web、简化 shared、Prisma schema 减半）
✅ Phase 4 / Phase 5 已废弃，P0 总工时降到 8.5 d

下次 Codex 切 USE_MOCK=false 联调时，按 v3.0 contract 直接对接 8 个接口即可。客服走抖音 IM 通道，与后端无关。

## 9. 待用户做的事（不阻塞 Phase 1）

- [ ] 抖音开放平台进入「能力 > 互动能力 > 消息管理 > 客服管理」开启抖音 IM 客服
- [ ] 绑定客服主管抖音号
- [ ] 客服人员登录 https://life-im.douyin.com/cs/login 切换在线
- [ ] 把客服抖音号告诉前端 / 后端，写进 `DOUYIN_IM_ID`（前端 `config/env.ts` 或后端 `MiniProgram.config.douyinImId`）

## 10. 之前 Delta 文档错误修正

`/Volumes/文件/dy/03-后端集成-Claude/04-WeChat-vs-Douyin-Delta.md` §6 「客服消息」部分的结论错误，将在本地 KB 修订：

- ❌ 错误说法："抖音不存在与微信 48h 客服消息等价的能力"
- ✅ 正确说法：
  - 抖音**有** IM 客服能力（`button open-type="im"`）
  - 抖音**有过** 消息推送客服能力，但**新接入小程序已禁用**（仅存量）
  - 抖音**有过** 自建客服能力，但**新接入小程序已禁用**
  - 唯一可选：**抖音 IM 客服**（客服在抖音 App 内回复，不走我们后端）

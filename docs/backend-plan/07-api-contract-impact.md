# 07 · 对 `docs/api-contract.md` 的提案变更

> 此文件为 **提案**，**不直接修改** `docs/api-contract.md`。
> 经 Codex 评审 + 双方确认后才正式 PR 修改 contract。
> 评审完成后请把每条变更的状态从 `☐ 待评审` 改为 `☑ 已合入 contract` 或 `❌ 已拒绝`。

---

## 提案 1（重要）：`POST /douyin/login` 请求体加 2 个字段

**当前 contract**（节选）：
```json
{
  "code": "tt.login 返回 code",
  "anonymousCode": "tt.login 返回 anonymousCode",
  "nickName": "...",
  "avatarUrl": "..."
}
```

**提案改为**：
```json
{
  "appid": "tt8xxxxxxxxx",
  "code": "tt.login 返回 code",
  "anonymousCode": "tt.login 返回 anonymousCode",
  "nickName": "...",
  "avatarUrl": "...",
  "hostApp": "douyin"
}
```

**新增字段**：
- `appid` (string, 必填) — 当前小程序 appid。前端用 `tt.getAccountInfoSync().miniProgram.appId` 获取。**用于多租户路由**。
- `hostApp` (string, 必填) — 宿主标识。前端用 `tt.getSystemInfoSync().appName` 获取并归一化为：`'douyin' | 'douyin_lite' | 'toutiao' | 'xigua'`

**为什么**：后端是多租户平台，必须靠 `appid` 路由到对应租户（用其 `secret` 调 `jscode2session`）；`hostApp` 用于多宿主能力路由（直播/拍抖音/关注号仅在抖音内可用）。

**对前端影响**：
- `tt.login` 之后多 2 行代码读 appid + hostApp
- 单店铺前端（洪承杂货店）感知最小

**状态**：☐ 待评审

---

## 提案 2：客服会话相关 4 个新增端点

为支持「轮询拉取增量消息 + 已读状态 + 最近会话 + 离线同步」，新增：

### 2.1 `GET /douyin/customer-service/sessions/:id/messages?afterId=&limit=`

请求：query 参数 `afterId`（消息 id，下一页起点）/ `limit`（默认 20，最大 100）

响应：
```json
{
  "code": 0,
  "data": [
    {
      "id": "...",
      "role": "system | user | agent",
      "content": "...",
      "mediaUrl": "...",
      "senderName": "...",
      "senderAvatar": "...",
      "createdAt": "10:35"
    }
  ]
}
```

**用途**：前端轮询拉取新消息（每 5s 调一次）。

### 2.2 `GET /douyin/customer-service/sessions/:id`

响应：
```json
{
  "code": 0,
  "data": {
    "id": "...",
    "status": "waiting | active | idle | resolved | timeout",
    "unreadCountFan": 3,
    "agent": { "name": "...", "avatar": "..." },
    "product": { "id": "...", "name": "..." }
  }
}
```

**用途**：前端进入会话页时拿到上下文。

### 2.3 `POST /douyin/customer-service/sessions/:id/read`

请求：无 body
响应：`{ "code": 0 }`
**用途**：用户进入会话后标记已读。

### 2.4 `GET /douyin/customer-service/recent`

响应：
```json
{
  "code": 0,
  "data": [
    {
      "sessionId": "...",
      "lastMessage": "...",
      "lastMessageAt": "...",
      "unreadCountFan": 3,
      "agent": { "name": "...", "avatar": "..." }
    }
  ]
}
```

**用途**：用户在「我的 → 我的咨询」入口看历史会话。

**状态**：☐ 待评审

---

## 提案 3：离线消息同步端点（关键 / 抖音独有）

### `GET /douyin/customer-service/sync-offline`

请求：无（依赖 token）
响应：
```json
{
  "code": 0,
  "data": [
    {
      "id": "...",
      "sessionId": "...",
      "type": "AGENT_REPLY | SYSTEM_NOTICE",
      "content": "...",
      "mediaUrl": "...",
      "createdAt": "..."
    }
  ]
}
```

**用途**：用户进入小程序时拉一次（建议在 `pages/contact/index/index` 的 `onShow`），合并到当前会话消息流，作为「您离线期间客服的回复」展示。

**为什么需要**：抖音平台没有 48h 客服消息接口，用户离线时客服回复无法主动推送，只能写入服务端队列等用户上线后拉取（详见 `05-offline-messaging.md`）。

**对前端影响**：
- 在 `pages/contact/index/index` 的 `onShow` 钩子里新增一次 sync-offline 调用
- UI 上对返回的离线消息可加一个 system 卡片提示「以下是您离线期间客服的回复」

**状态**：☐ 待评审

---

## 提案 4：多宿主能力查询端点

### `GET /douyin/capabilities`

响应：
```json
{
  "code": 0,
  "data": {
    "hostApp": "douyin",
    "capabilities": {
      "live.openWebcastRoom": true,
      "live.checkStatus": true,
      "aweme.captureView": true,
      "aweme.followCheck": true,
      "subscribe.message": true
    }
  }
}
```

**用途**：前端按 capability 隐藏/显示能力入口。例如用户在「今日头条」打开小程序时，「拍同款」按钮应该隐藏（因为头条不支持）。

**对前端影响**：
- 启动时拉一次，缓存到本地
- 在每个抖音独有能力入口（拍同款 / 直播 / 关注）前判断

**状态**：☐ 待评审

---

## 提案 5：订阅消息授权上报

### `POST /douyin/subscribe/notify`

请求：
```json
{
  "scene": "session_close | order_submit | ...",
  "results": [
    { "msgId": "MMSG_xxx", "status": "accept | reject | ban" }
  ]
}
```

响应：`{ "code": 0 }`

**用途**：前端在调用 `tt.requestSubscribeMessage` 后把每个模板的授权结果上报，后端记录到 `SubscribeAuthorization` 表，作为后续推送的依据。

**对前端影响**：
- 在 `tt.requestSubscribeMessage` 的 success 回调里调一次

**状态**：☐ 待评审

---

## 提案 6：响应格式锁死成 envelope（取消"两种都支持"）

**当前 contract** 第 7-27 行写明前端"支持两种响应"（直接返回数据 / envelope 返回）。

**提案统一为 envelope**：

```json
{
  "code": 0,
  "msg": "ok",
  "data": { ... },
  "traceId": "uuid",
  "ts": 1714377600
}
```

**为什么**：
- 一致的错误处理路径
- `traceId` 便于双 AI 联调时排查问题（前后端日志可关联）
- 后端实现更简单（不需要按端点切换响应风格）

**状态**：☐ 待评审

---

## 提案 7：错误码体系

业务错误码用 6 位整数（HTTP 永远 200，业务错误在 `code` 字段表达）：

| 范围 | 含义 |
|---|---|
| `0` | 成功 |
| `10000-19999` | 参数错误（10001 缺失 / 10002 格式） |
| `20000-29999` | 业务异常（21001 商品已下架等） |
| `30000-39999` | 第三方异常（抖音 OpenAPI 错） |
| `40000-49999` | 鉴权 / 权限（40101 未登录 / 40102 token 过期 / 40103 权限不足） |
| `50000-59999` | 服务器异常 |
| `60000-69999` | 风控 / 限频 |

每个 `code` 必有对应的中文 `msg`，前端可直接 toast 显示。

**状态**：☐ 待评审

---

## 提案 8：时间字段统一格式

| 字段 | 现状 | 提案 |
|---|---|---|
| `Order.createdAt` | string（未明确格式） | string ISO-8601（如 `"2026-04-30T10:35:00+08:00"`） |
| 客服消息 `createdAt` | string（前端 mock 是 `"10:35"`） | **由后端格式化好**：当天 `HH:mm` / 跨天 `MM-DD HH:mm` / 跨年 `YYYY-MM-DD HH:mm` |
| 内部 API（agent / admin） | — | 统一用 ISO-8601，前端格式化 |

**为什么**：客服消息频繁展示，由后端格式化省前端工作量；其他场景前端格式化更灵活。

**状态**：☐ 待评审

---

## 提案 9：金额字段单位锁定

| 字段 | 现状 | 提案 |
|---|---|---|
| `Product.price` | number（单位元） | **保持 number 单位元**（前端 mock 现状一致） |
| `Order.referencePrice` | number | 同上 |

**未来若做支付**（首版不做）：金额 mate 字段单位用 `*Fen`（单位分 int），与此分离。

**状态**：☐ 待评审

---

## 评审清单

请 Codex 在评审时对每条提案标注：
- ✅ 同意（可直接合入 contract）
- ⚠️ 同意但建议调整（说明调整内容）
- ❌ 不同意（说明原因）
- ❓ 需要进一步讨论

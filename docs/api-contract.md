# API 契约（P0 最终版）

> 本文件是前后端字段对齐的**唯一基准**。
> 字段名与前端 TypeScript 模型一致。
> 历史经过 3 轮 Codex/Claude 评审已收敛，所有 P0 字段已最终确定。
> 详细设计依据：`changes/2026-04-30-claude-contract-fields-clarification.md`
> 版本：v2.0（2026-04-30）

## 通用响应格式（统一 envelope）

**所有响应**统一为 envelope 结构：

```json
{
  "code": 0,
  "message": "ok",
  "data": { },
  "traceId": "uuid-v4"
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | int | 0 = 业务成功；其他 = 业务失败（前端按 code 处理） |
| `message` | string | 业务消息；失败时可直接 toast |
| `data` | object \| array \| null | 业务数据；失败时为 null |
| `traceId` | string | 后端生成，用于全链路排查 |

**HTTP 状态码 + 业务 code 双层语义**：

| HTTP | 含义 |
|---|---|
| 200 | 业务成功 / 业务失败（看 `code`） |
| 400 | 参数格式错（请求未解析成功） |
| 401 | 未登录 / token 失效 |
| 403 | 权限不足 / 跨租户访问 |
| 404 | 路由不存在 |
| 429 | 限流 |
| 500 | 服务器内部错 |
| 502/503 | 第三方依赖故障（抖音 OpenAPI 5xx） |

## 鉴权

所有需要登录的接口必须带：

```http
Authorization: Bearer <jwt-token>
```

JWT claim 含 `fanId / miniProgramId / openid / hostApp / ver`。
Token 过期 7 天。失效返 HTTP 401 + `code: 40102`。

## 错误码

| 范围 | 含义 |
|---|---|
| `0` | 成功 |
| `10000-19999` | 参数错误（10001 缺失 / 10002 格式） |
| `20000-29999` | 业务异常（21001 商品已下架等） |
| `30000-39999` | 第三方异常（抖音 OpenAPI 错） |
| `40000-49999` | 鉴权 / 权限 |
| `50000-59999` | 服务器异常 |
| `60000-69999` | 风控 / 限频 / 内容安全 |

常用错误码：

| code | 含义 |
|---|---|
| 0 | 成功 |
| 10001 | 参数缺失 |
| 10002 | 参数格式错误 |
| 20001 | 资源不存在 |
| 21001 | 库存不足 |
| 21002 | 商品已下架 |
| 21010 | 状态机不允许该操作 |
| 21030 | 会话已关闭 |
| 30001 | 抖音平台异常 |
| 40101 | 未登录 / code 已失效 |
| 40102 | token 已过期 |
| 40103 | 无权限 |
| 50000 | 服务器异常 |
| 60010 | 内容含违规词 |
| 60020 | 操作过于频繁 |

## 数据枚举

```ts
type ProductCategory = 'quote-tool' | 'hardware' | 'stock' | 'hot';
type ProductServiceType = 'pickup' | 'upgrade';
type StockStatus = 'in_stock' | 'limited' | 'out_of_stock' | 'bookable';
type OrderStatus = 'pending' | 'confirmed' | 'completed' | 'cancelled';
type MessageRole = 'system' | 'user' | 'agent';
type MessageType = 'text';   // P0 仅支持 text，P1 加 image
type MessageStatus = 'sent' | 'failed';
```

## 时间字段约定

- 所有时间字段统一返 **ISO-8601 带时区**：`"2026-04-30T10:35:00+08:00"`
- 客服消息额外返 `displayTime`（字符串，后端格式化好直接显示）：
  - 当天：`"10:35"`
  - 跨天：`"04-29 18:00"`
  - 跨年：`"2025-12-31 23:59"`

## 金额字段约定

- 所有金额字段单位为 **元**（number）
- 字段命名直接 `price` / `referencePrice`，无 `Fen` 后缀
- P0 不涉及支付，不引入支付分单位

---

# P0 接口（共 12 个）

## 1. POST /douyin/login

**用途**：用户登录，换取 JWT token

**请求**：
```json
{
  "appid": "tt_xxxxxxxxx",
  "code": "tt.login 返回的 code",
  "anonymousCode": "tt.login 返回的 anonymousCode",
  "nickName": "用户昵称（可选）",
  "avatarUrl": "用户头像 URL（可选）"
}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `appid` | ✅ | `tt.getAccountInfoSync().miniProgram.appId` 取值；后端校验该 mp 已配置且 status=ACTIVE |
| `code` | 二选一 | tt.login 返回的临时 code（5 分钟一次性） |
| `anonymousCode` | 二选一 | 用户拒绝授权时换 anonymousOpenid |
| `nickName` | 可选 | 用户已授权时传 |
| `avatarUrl` | 可选 | 同上 |
| `hostApp` | ❌ P0 不传 | 后端默认 `'douyin'`。P1 前端补传 |

**响应**：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "userId": "fan_clxxxxxxxxxxxxxxx",
    "nickName": "用户昵称",
    "avatarUrl": "https://...",
    "isNewUser": true
  },
  "traceId": "trace_xxx"
}
```

**错误码**：
- `40101` code 已失效，重新 tt.login
- `40102` appid 不存在或已暂停
- `30001` 抖音平台返回错误

## 2. GET /douyin/config

**用途**：门店基础配置

**响应**：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "name": "洪承杂货店",
    "serviceSummary": "门店商品展示、门店参考价、到店自提预约与客服咨询",
    "phone": "400-000-9191",
    "businessHours": "周一至周日 09:30-20:00",
    "pickupNotice": "价格、库存和配置以门店当天现货及线下确认为准，提交预约后请等待客服确认。",
    "afterSaleNotice": "硬件类商品按品牌政策和门店售后规则处理，具体以门店确认结果为准。"
  }
}
```

说明：
- 前端不展示地址，后端不强制返 `address`
- 该接口可不登录访问

## 3. GET /douyin/categories

**用途**：分类列表

**响应**：
```json
{
  "code": 0,
  "message": "ok",
  "data": [
    { "id": "quote-tool", "name": "升级服务", "desc": "SSD、内存、系统迁移与检测预约" },
    { "id": "hardware",   "name": "硬件设备", "desc": "整机、显示器、存储与办公外设" },
    { "id": "stock",      "name": "门店常备", "desc": "网络设备、扩展坞与常用配件" },
    { "id": "hot",        "name": "热门推荐", "desc": "近期咨询较多的商品与服务" }
  ]
}
```

## 4. GET /douyin/products

**用途**：商品列表（按分类 / 关键词查）

**Query**：
```text
category=all|quote-tool|hardware|stock|hot
keyword=搜索关键词
```

**响应**：`Product[]` 包在 envelope 内
```json
{
  "code": 0,
  "message": "ok",
  "data": [ /* Product 对象数组 */ ]
}
```

## 5. GET /douyin/products/:id

**用途**：商品详情

**响应**：单个 Product 对象包在 envelope 内

**Product 类型**：
```ts
interface Product {
  id: string;
  name: string;
  category: ProductCategory;
  serviceType: ProductServiceType;
  spec: string;
  price: number;                // 单位元，门店参考价
  stockStatus: StockStatus;
  images: string[];
  badges: string[];
  description: string;
  pickupNote: string;
  afterSaleNote: string;
  specs?: ProductSpecItem[];
  options?: ProductOption[];
}

interface ProductSpecItem { label: string; value: string; }
interface ProductOption { id: string; name: string; spec: string; price: number; desc: string; }
```

要求：
- `price` 是门店参考价，单位为元
- `images` 不要混用，每个商品必须独有
- `serviceType = upgrade` 时建议提供 `options`
- 整机类商品建议提供 `specs`

## 6. POST /douyin/reservations

**用途**：创建预约单（非交易）

**请求**：
```json
{
  "productId": "p-office-desktop-01",
  "productName": "办公台式机整机套装",
  "spec": "i5 / 16G / 1T SSD / 23.8 英寸显示器",
  "quantity": 1,
  "referencePrice": 3899,
  "contactName": "张三",
  "contactPhone": "13800000000",
  "remark": "下午到店"
}
```

**响应**：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    /* Order 对象 */
  }
}
```

要求：
- 必须登录（HTTP 401 + 40101 否则）
- 这是预约单，不是支付订单
- 默认状态 `pending`
- `contactPhone` 入库时加密保存，列表/详情返回时**脱敏**：`"138****0000"`
- 24h 同 fan + 同 product 提交 ≤ 5 个（防滥用）

## 7. GET /douyin/orders

**用途**：我的预约列表

**响应**：`Order[]`，按 `createdAt` 倒序

要求：
- 必须登录
- 仅返回当前 fan 的预约

## 8. GET /douyin/orders/:id

**用途**：预约详情

**响应**：单个 Order 对象

要求：
- 必须登录
- 校验 Order 归属当前 fan（否则 HTTP 403 + 40103）

**Order 类型**：
```ts
interface Order {
  id: string;
  orderNo: string;
  productId: string;
  productName: string;
  spec: string;
  quantity: number;
  referencePrice: number;
  status: OrderStatus;
  contactName: string;
  contactPhone: string;        // 脱敏，如 "138****0000"
  remark: string;
  createdAt: string;           // ISO-8601
}
```

## 9. POST /douyin/customer-service/session

**用途**：创建/复用客服会话

**请求**：
```json
{
  "productId": "p-xxx (可选)",
  "orderId": "o-xxx (可选)",
  "source": "product_detail | order_detail | home (可选)",
  "userProfile": {
    "nickName": "抖音昵称",
    "avatarUrl": "抖音头像"
  }
}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `productId` | 可选 | 进入会话时关联的商品 |
| `orderId` | 可选 | 关联预约单 |
| `source` | 可选 | 进入来源（用户行为分析） |
| `userProfile` | 可选 | 同步用户资料给客服侧 |

**响应**：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "sessionId": "sess_clxxxxxxxxxxxxxxx",
    "isNew": true,
    "agentName": "在线客服",
    "agentAvatar": "https://...",
    "welcomeMessage": "您好，欢迎来到洪承杂货店，请发送您的问题..."
  }
}
```

| 字段 | 说明 |
|---|---|
| `sessionId` | Session.id |
| `isNew` | true=新建；false=复用已有活跃会话 |
| `agentName` | 当前归属客服的 displayName，未分配显示「在线客服」 |
| `agentAvatar` | 客服头像 URL，未分配显示默认头像 |
| `welcomeMessage` | 仅 isNew=true 时返回，前端可作为第一条 system 消息 |

要求：
- 必须登录
- P0 不允许匿名会话

## 10. POST /douyin/customer-service/message/send

**用途**：发送用户消息（**返回 user ack，不返 agent 假回复**）

**请求**：
```json
{
  "sessionId": "sess_xxx",
  "type": "text",
  "content": "我想咨询这台主机",
  "clientMessageId": "local-uuid-v4"
}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `sessionId` | ✅ | 来自 `/session` 接口 |
| `type` | ✅ | **P0 仅 `"text"`**，图片 P1 |
| `content` | ✅ | ≤ 5000 字符；过 antidirt 内容安全 |
| `clientMessageId` | ✅ **强制** | 前端 UUID v4；同 sessionId 内重复返老消息 |

**响应**（user ack）：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "messageId": "msg_xxx",
    "sessionId": "sess_xxx",
    "role": "user",
    "type": "text",
    "content": "我想咨询这台主机",
    "createdAt": "2026-04-30T10:30:00+08:00",
    "displayTime": "10:30",
    "status": "sent"
  }
}
```

**注意**：客服回复**不在本接口返回**。前端通过下面的 GET /messages 轮询拉取客服回复。

**错误码**：
- `60010` 内容含违规词
- `60020` 频次过快（> 3/秒）
- `21030` 会话已关闭

## 11. GET /douyin/customer-service/sessions/:id/messages

**用途**：拉取消息历史 / 增量轮询客服回复

**Query**：
```text
afterId=msg_xxx (可选)
limit=20 (默认 20，最大 100)
```

| 参数 | 说明 |
|---|---|
| `afterId` | 上次拉到的最后一条 messageId。**为空时返回最近 20 条**（升序） |
| `limit` | 每页条数 |

**响应**：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "list": [
      {
        "id": "msg_001",
        "role": "system",
        "senderName": "系统",
        "senderAvatar": "",
        "type": "text",
        "content": "您好，欢迎来到洪承杂货店",
        "createdAt": "2026-04-30T10:25:00+08:00",
        "displayTime": "10:25"
      },
      {
        "id": "msg_002",
        "role": "user",
        "senderName": "张三",
        "senderAvatar": "https://...",
        "type": "text",
        "content": "我想问下办公主机的价格",
        "createdAt": "2026-04-30T10:30:00+08:00",
        "displayTime": "10:30"
      },
      {
        "id": "msg_003",
        "role": "agent",
        "senderName": "在线客服",
        "senderAvatar": "https://...",
        "type": "text",
        "content": "您好，办公主机现货 3899 元",
        "createdAt": "2026-04-30T10:31:00+08:00",
        "displayTime": "10:31"
      }
    ],
    "nextAfterId": "msg_003",
    "hasMore": false
  }
}
```

| 字段 | 说明 |
|---|---|
| `list` | **升序排列**（最早 → 最新） |
| `nextAfterId` | 下次拉取传这个值（最后一条的 id） |
| `hasMore` | 是否还有更多（仅在 `afterId` 不为空时为 false 才算结束）|

**轮询建议**：
- 用户在会话页：每 5 秒一次
- 激活页面可加快到 3 秒，blur 减慢到 10 秒
- 用户离开会话页：停止轮询
- 前端用 `nextAfterId` 累积式拉取

## 12. GET /douyin/customer-service/sync-offline

**用途**：拉取用户离线期间客服的所有回复（关键 — 抖音平台无 48h 客服消息）

**请求**：无参数（依赖 token）

**响应**：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "list": [
      {
        "id": "off_001",
        "sessionId": "sess_xxx",
        "messageId": "msg_xxx",
        "role": "agent",
        "senderName": "在线客服",
        "senderAvatar": "https://...",
        "type": "text",
        "content": "您好，看到您离线了，明天可以来店里看看",
        "createdAt": "2026-04-29T18:00:00+08:00",
        "displayTime": "04-29 18:00"
      }
    ],
    "syncedCount": 1,
    "hasOfflineMessages": true,
    "systemTip": "以下是您离线期间客服的回复"
  }
}
```

| 字段 | 说明 |
|---|---|
| `list` | 离线消息列表（按 sessionId 分组合并）|
| `syncedCount` | 本次同步数量 |
| `hasOfflineMessages` | true = 至少 1 条；false = 无离线消息 |
| `systemTip` | 系统提示文案 |

**调用时机**：前端在 `pages/contact/index/index.ts` 的 `onShow` 调用一次。如果 `hasOfflineMessages=true`，把 `list` 里的消息合并到对应 `sessionId` 的消息流，按时间戳插入正确位置，并显示 `systemTip` 系统卡片。

**后端行为**：
- 拉取时自动标记 `OfflineMessage.status=DELIVERED`
- 7 天未拉的自动 EXPIRED 清理
- 跨多个 sessionId 一次返回（fan 可能在多个会话有积压）

---

# P1+ 后置接口（不在 P0 范围）

以下接口三方共识**P0 不做**，P1 再加：

| 接口 | 后置原因 |
|---|---|
| `POST /douyin/customer-service/sessions/:id/read` | UX 优化 |
| `GET /douyin/customer-service/recent` | 用户多会话场景才需要 |
| `POST /douyin/subscribe/notify` | 订阅消息推送链路 P1 |
| `GET /douyin/capabilities` | 多宿主能力路由 P2 |
| 抖音独有：直播 / 拍抖音 / 关注号 | P2 |

---

# 不要返回的字段（首版固定）

后端**绝不返回**以下字段（即使前端查询）：

- `payStatus` / `paid` / `paymentUrl` / `checkout`
- `refund` / `transactionId`
- `unionid`（仅服务端持有，不下发）
- `sessionKey`（仅服务端持有，不下发）
- 手机号明文（必须脱敏）

---

# 联调切换

前端从 mock 切真实后端：

```ts
// frontend config
export const API_BASE_URL = 'https://api.example.com';   // P0 部署后给
export const USE_MOCK = false;
```

P0 开发期后端默认 `DOUYIN_MODE=mock`，不依赖真实抖音 OpenAPI（前端可零阻塞联调）。
正式联调时切 `DOUYIN_MODE=real` 并配置 `DOUYIN_APPID` / `DOUYIN_SECRET` 环境变量。

---

# 变更历史

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-04-29 | 首版（10 个 P0 接口） |
| **v2.0** | **2026-04-30** | **三轮 Codex/Claude 评审收敛**：响应统一 envelope + HTTP 状态码 + traceId；新增 sync-offline、messages 增量端点；message/send 改 user ack；时间字段 ISO + displayTime；登录加 appid；多租户由 mini-programs 隔离 |

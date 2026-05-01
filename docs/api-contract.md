# API 契约（P0 最终版）

> 本文件是前后端字段对齐的**唯一基准**。
> 字段名与前端 TypeScript 模型一致。
> 历史经过 5 轮 Codex/Claude 评审已收敛，所有 P0 字段已最终确定。
> 详细设计依据：`changes/2026-05-01-claude-response-frontend-progress.md`
> 版本：**v2.2（2026-05-01）** ← 见末尾变更历史

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

## 全局必传 Header（v2.1 新增）

**所有 `/douyin/*` 请求必须带**：

```http
X-Douyin-Appid: tt_xxxxxxxxx
```

前端取值方式：
```ts
const appid = tt.getAccountInfoSync().miniProgram.appId;
```

**后端解析逻辑**：
1. 已登录接口（带 `Authorization`）→ 优先用 JWT 内的 `miniProgramId`，并校验与 header 的 `X-Douyin-Appid` 一致（不一致返 `40104`）
2. 未登录接口（如 `/config` `/categories` `/products`）→ 用 `X-Douyin-Appid` 解析对应 MiniProgram
3. 缺 `X-Douyin-Appid` → `10001` 参数缺失
4. appid 不存在或 status=SUSPENDED → `40104`

## 鉴权

需要登录的接口额外带：

```http
Authorization: Bearer <jwt-token>
```

JWT claim 含 `fanId / miniProgramId / openid / hostApp / ver`。
Token 过期 7 天。失效返 HTTP 401 + `code: 40102`（**仅表示 token 过期**，不要复用）。

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
| 40102 | token 已过期（**仅此一种含义**） |
| 40103 | 无权限 |
| 40104 | appid 不存在或已暂停（v2.1 新增） |
| 40105 | 账号锁定 |
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
type MessageType = 'text' | 'image' | 'voice' | 'video';   // v2.2 扩展（前端已实现 4 种 UI）
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
- `40104` appid 不存在或已暂停（v2.1 修正：之前误用 40102）
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

**v2.1 改动**：
- ✅ 新增 `optionId`（升级服务必填）
- ✅ 新增 `Idempotency-Key` header（推荐，防双击）
- ✅ 移除信任客户端的 `productName / spec / referencePrice`（后端按 `productId + optionId` 查服务端实价）

**请求**：

```http
POST /douyin/reservations
X-Douyin-Appid: tt_xxx
Authorization: Bearer <token>
Idempotency-Key: <uuid-v4>     # 推荐，防双击/网络重发；24h 内同 key 返同一 Order
```

```json
{
  "productId": "p-office-desktop-01",
  "optionId": "opt-ssd-1tb",
  "quantity": 1,
  "contactName": "张三",
  "contactPhone": "13800000000",
  "remark": "下午到店"
}
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `productId` | ✅ | 服务端查询商品 |
| `optionId` | ⚠️ | `serviceType=upgrade` 时**必填**，其他类型可空 |
| `quantity` | ✅ | 数量 |
| `contactName` | ✅ | 联系人 |
| `contactPhone` | ✅ | 入库 AES-256-GCM 加密，响应脱敏 |
| `remark` | 可选 | ≤ 200 字符 |

**响应**：单个 Order 对象包在 envelope 内（含 `optionId/optionName/optionSpec`）

要求：
- 必须登录（HTTP 401 + 40101 否则）
- 这是预约单，不是支付订单
- 默认状态 `pending`
- `contactPhone` 入库时 AES-256-GCM 加密，列表/详情返回时**脱敏**：`"138****0000"`
- **后端按 `productId + optionId` 查服务端 product/option 拿真实 name/spec/price**（不信任客户端字段）
- 24h 同 fan + 同 product 提交 ≤ 5 个（防滥用）
- `Idempotency-Key` 同 24h 内返已存在的 Order（不新建）

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

**Order 类型（v2.1 新增 option 三字段）**：
```ts
interface Order {
  id: string;
  orderNo: string;
  productId: string;
  productName: string;          // 后端快照
  spec: string;                 // 后端快照（含 option 选择）
  optionId?: string;            // v2.1 新增
  optionName?: string;          // v2.1 新增
  optionSpec?: string;          // v2.1 新增
  quantity: number;
  referencePrice: number;       // 服务端价
  status: OrderStatus;
  contactName: string;
  contactPhone: string;         // 脱敏，如 "138****0000"
  remark: string;
  createdAt: string;            // ISO-8601
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

**响应**（v2.1 移除 welcomeMessage 字段，避免与 /messages 重复）：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "sessionId": "sess_clxxxxxxxxxxxxxxx",
    "isNew": true,
    "agentName": "在线客服",
    "agentAvatar": "https://..."
  }
}
```

| 字段 | 说明 |
|---|---|
| `sessionId` | Session.id |
| `isNew` | true=新建；false=复用已有活跃会话 |
| `agentName` | 当前归属客服的 displayName，未分配显示「在线客服」 |
| `agentAvatar` | 客服头像 URL，未分配显示默认头像 |

**v2.1 设计变更**：欢迎语**单一事实源**——后端在新建 session 时**持久化一条 system 欢迎消息到 messages 表**，前端通过 `GET /messages` 统一拿到（含欢迎语），无需特殊处理。

要求：
- 必须登录
- P0 不允许匿名会话
- isNew=true 时后端必须 INSERT 一条 system 欢迎消息（content 来自 MiniProgram.config.welcomeMessage 或租户默认值）

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
| `type` | ✅ | **v2.2 支持 4 种**：`text` / `image` / `voice` / `video` |
| `content` | ⚠️ | `type=text` 时填文本（≤ 5000 字符，过 antidirt）；其他类型可空 |
| `uploadId` | ⚠️ | `type=image|voice|video` 时**必填**，先调 `/douyin/upload` 拿（v2.2 新增） |
| `clientMessageId` | ✅ **强制** | 前端 UUID v4；同 sessionId 内重复返老消息 |

**响应**（v2.1 修正：消息 id 字段统一为 `id`）：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "id": "msg_xxx",
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

**v2.1 字段说明**：
- `id`：消息 id（**与 /messages 接口保持一致**，所有消息对象都用 `id`）
- `messageId`：兼容期保留，下个迭代 deprecate（前端建议直接用 `id`）

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

**v2.1 改动**：
- ✅ 新增可选 query `?sessionId=` 单会话过滤
- ✅ 字段重命名：list 项的 `id` 现在是**消息 id**（与其他接口一致），离线记录 id 改名 `offlineId`

**请求**：

```http
GET /douyin/customer-service/sync-offline                    # 全量（多会话入口用）
GET /douyin/customer-service/sync-offline?sessionId=sess_xxx # 单会话过滤
```

**响应**：
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "list": [
      {
        "id": "msg_xxx",
        "offlineId": "off_001",
        "sessionId": "sess_xxx",
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
| `list[].id` | **消息 id**（v2.1 修正，与 /messages 接口一致；前端用此去重） |
| `list[].offlineId` | 离线记录 id（v2.1 新增；用于幂等同步追踪） |
| `list[]` 其他字段 | 与 MessageDto 一致 |
| `syncedCount` | 本次同步数量 |
| `hasOfflineMessages` | true = 至少 1 条 |
| `systemTip` | 系统提示文案 |

**调用时机**：
- 前端在 `pages/contact/index/index.ts` 的 `onShow` 调用一次（**带 `sessionId` 过滤**当前会话）
- 多会话入口（如「我的咨询」）调用无参版本

合并逻辑：把 `list` 里的消息按 `id` 去重合并到对应 `sessionId` 的消息流，按 `createdAt` 插入正确位置，显示 `systemTip` 系统卡片。

**后端行为**：
- 拉取时自动标记 `OfflineMessage.status=DELIVERED`
- 同条件下次调用返回空 list（避免重复插入）
- 7 天未拉的自动 EXPIRED 清理

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

# ## 13. POST /douyin/upload (v2.2 新增)

**用途**：上传客服会话内的图片 / 视频 / 语音文件，发消息前必须先 upload 拿到 `uploadId`

**请求**：

```http
POST /douyin/upload
X-Douyin-Appid: tt_xxx
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

```
form-data:
  file: <binary>                                 # 必填
  scene: "cs_image" | "cs_video" | "cs_voice"    # 必填
  duration?: <number, 毫秒>                       # voice/video 必填
```

**响应**：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "id": "upload_xxx",
    "url": "https://cdn.example.com/uploads/...",
    "thumbUrl": "https://cdn.example.com/uploads/.../thumb.jpg",
    "fileName": "image_001.jpg",
    "fileSize": 102400,
    "mimeType": "image/jpeg",
    "duration": 15000,
    "width": 800,
    "height": 600
  }
}
```

| 字段 | 适用 | 说明 |
|---|---|---|
| `id` | 全部 | 上传记录 id（`uploadId`），发消息时传 |
| `url` | 全部 | 文件 CDN 直链 |
| `thumbUrl` | image/video | 缩略图 600px |
| `fileName` | 全部 | 服务端生成的文件名 |
| `fileSize` | 全部 | 字节数 |
| `mimeType` | 全部 | MIME 类型 |
| `duration` | voice/video | 时长（毫秒） |
| `width / height` | image/video | 像素尺寸 |

**限制**：

| scene | 大小上限 | mime 白名单 | duration 上限 |
|---|---|---|---|
| `cs_image` | 5 MB | `image/jpeg, image/png, image/webp` | — |
| `cs_video` | 30 MB | `video/mp4, video/quicktime` | 60 秒 |
| `cs_voice` | 2 MB | `audio/mp3, audio/aac, audio/m4a, audio/wav` | 60 秒 |

**幂等**：客户端可用文件 SHA256 在 1 小时内复用上传记录（避免重复上传）

**异步内容审核**：
- 图片：抖音 image audit 异步任务
- 视频：抽 1-3 帧 + image audit
- 语音：仅大小/时长校验，**P0 不做内容审核**（成本太高）

**错误码**：
- `10002` mime 不匹配 / size 超限 / duration 超限
- `60010` 内容审核命中违规（异步审核结果回写后）
- `50000` 服务器存储失败

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
| v2.0 | 2026-04-30 | 三轮 Codex/Claude 评审收敛：响应统一 envelope + HTTP 状态码 + traceId；新增 sync-offline、messages 增量端点；message/send 改 user ack；时间字段 ISO + displayTime；登录加 appid；多租户由 mini-programs 隔离 |
| v2.1 | 2026-04-30 | Codex 第四轮评审 6 项字段修正（详见 `changes/2026-04-30-claude-response-codex-review-2.md`）：(1) 全局加 `X-Douyin-Appid` header；(2) `40102` 仅表 token 过期，新增 `40104` = appid 不存在/暂停；(3) 预约加 `optionId` + `Idempotency-Key`；(4) 消息对象统一用 `id`；(5) `/session` 移除 `welcomeMessage`；(6) `/sync-offline` 加 `?sessionId=` 过滤 |
| **v2.2** | **2026-05-01** | **接受 Codex 前端范围扩张**（详见 `changes/2026-05-01-claude-response-frontend-progress.md`）：(1) 消息类型 `text` → `text/image/voice/video` 4 种；(2) 新增 `POST /douyin/upload` 端点（图片/视频/语音上传）；(3) `message/send` 支持 `uploadId` 字段；(4) 消息对象加 `mediaUrl/thumbUrl/duration/width/height/fileSize`；(5) P0 工时 17d → 19.5d（+2.5d for 多媒体处理） |

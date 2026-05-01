# API 契约（P0 最终版 v3.0）

> 本文件是前后端字段对齐的**唯一基准**。
> 字段名与前端 TypeScript 模型一致。
> 历史经过 6 轮 Codex/Claude 评审 + 项目负责人决策已收敛。
> 详细设计依据：`changes/2026-05-01-claude-acknowledge-official-im-decision.md`
> 版本：**v3.0（2026-05-01）** — 客服全面切换抖音官方 IM，砍掉自建客服全部接口

## 重大版本变更

**v3.0 项目负责人决策**：「全面删除自建客服 直接调用使用官方客服IM」

平台已确认：**自建客服 / 消息推送客服已停止新增接入**，新小程序唯一可选 = 抖音 IM 客服。

后端**不再实现**任何客服相关接口。客服在「抖音 App 内」由真人客服处理（用 `button open-type="im"` 拉起）。

## 通用响应格式（统一 envelope）

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
| `code` | int | 0 = 业务成功；其他 = 业务失败 |
| `message` | string | 业务消息；失败时可直接 toast |
| `data` | object \| array \| null | 业务数据 |
| `traceId` | string | 后端生成，用于全链路排查 |

**HTTP 状态码 + 业务 code 双层语义**：

| HTTP | 含义 |
|---|---|
| 200 | 业务成功 / 业务失败（看 `code`） |
| 400 | 参数格式错 |
| 401 | 未登录 / token 失效 |
| 403 | 权限不足 |
| 404 | 路由不存在 |
| 429 | 限流 |
| 500 | 服务器内部错 |
| 502/503 | 第三方依赖故障 |

## 全局必传 Header

**所有 `/douyin/*` 请求必须带**：

```http
X-Douyin-Appid: tt_xxxxxxxxx
```

前端取值：`tt.getAccountInfoSync().miniProgram.appId`

后端解析：
1. 已登录接口（带 `Authorization`）→ 优先用 JWT 内的 `miniProgramId`，校验与 header 一致
2. 未登录接口 → 用 header 解析 MiniProgram
3. 缺 header → `10001`
4. appid 不存在/暂停 → `40104`

## 鉴权

需要登录的接口额外带：

```http
Authorization: Bearer <jwt-token>
```

JWT claim 含 `fanId / miniProgramId / openid / hostApp / ver`。Token 过期 7 天。

## 错误码

| 范围 | 含义 |
|---|---|
| `0` | 成功 |
| `10000-19999` | 参数错误 |
| `20000-29999` | 业务异常 |
| `30000-39999` | 第三方异常 |
| `40000-49999` | 鉴权 / 权限 |
| `50000-59999` | 服务器异常 |
| `60000-69999` | 风控 / 限频 |

常用：

| code | 含义 |
|---|---|
| 0 | 成功 |
| 10001 | 参数缺失 |
| 10002 | 参数格式错误 |
| 20001 | 资源不存在 |
| 21001 | 库存不足 |
| 21002 | 商品已下架 |
| 21010 | 状态机不允许该操作 |
| 30001 | 抖音平台异常 |
| 40101 | 未登录 / code 已失效 |
| 40102 | token 已过期 |
| 40103 | 无权限 |
| 40104 | appid 不存在或已暂停 |
| 40105 | 账号锁定 |
| 50000 | 服务器异常 |
| 60020 | 操作过于频繁 |

## 数据枚举

```ts
type ProductCategory = 'quote-tool' | 'hardware' | 'stock' | 'hot';
type ProductServiceType = 'pickup' | 'upgrade';
type StockStatus = 'in_stock' | 'limited' | 'out_of_stock' | 'bookable';
type OrderStatus = 'pending' | 'confirmed' | 'completed' | 'cancelled';
```

## 时间字段约定

- 所有时间字段统一返 **ISO-8601 带时区**：`"2026-04-30T10:35:00+08:00"`
- 前端自行格式化展示

## 金额字段约定

- 所有金额字段单位为 **元**（number）
- 字段命名直接 `price` / `referencePrice`，无 `Fen` 后缀
- P0 不涉及支付

---

# P0 接口（共 8 个）

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
| `appid` | ✅ | 当前小程序 appid，与 X-Douyin-Appid 一致 |
| `code` | 二选一 | tt.login 返回的临时 code（5 分钟一次性） |
| `anonymousCode` | 二选一 | 用户拒绝授权时用 |
| `nickName` | 可选 | 用户已授权时传，未授权时不传 |
| `avatarUrl` | 可选 | 同上 |
| `hostApp` | 可选 | P0 不传，后端默认 `'douyin'` |

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
- `40104` appid 不存在或已暂停
- `30001` 抖音平台返回错误

## 2. GET /douyin/config

**用途**：门店基础配置 + 抖音 IM 客服账号

**响应**（v3.0 新增 `douyinImId`）：
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
| `name / serviceSummary / businessHours / pickupNotice / afterSaleNotice` | ✅ | 门店配置 |
| `phone` | 可选 | 门店电话，前端不展示电话入口 |
| `douyinImId` | ⚠️ 推荐 | 当前 mp 绑定的抖音 IM 客服账号；前端 `<button open-type="im" data-im-id="{{douyinImId}}">` 用 |

**配置位置**：后端 `MiniProgram.config.douyinImId`（管理员后台编辑该 JSON 字段）

**说明**：该接口可不登录访问。前端不展示门店地址（后端不返 `address`）。

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

**用途**：商品列表

**Query**：
```text
category=all|quote-tool|hardware|stock|hot
keyword=搜索关键词
```

**响应**：`Product[]` 包在 envelope 内

## 5. GET /douyin/products/:id

**用途**：商品详情

**响应**：单个 Product 对象

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
- `price` 是门店参考价
- `images` 不混用，每商品独有
- `serviceType=upgrade` 时建议提供 `options`
- 整机类商品建议提供 `specs`

## 6. POST /douyin/reservations

**用途**：创建预约单（非交易）

**请求**：

```http
POST /douyin/reservations
X-Douyin-Appid: tt_xxx
Authorization: Bearer <token>
Idempotency-Key: <uuid-v4>     # 推荐，防双击
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
| `optionId` | ⚠️ | `serviceType=upgrade` 时**必填** |
| `quantity` | ✅ | 数量 |
| `contactName` | ✅ | 联系人 |
| `contactPhone` | ✅ | 入库 AES-256-GCM 加密，响应脱敏 |
| `remark` | 可选 | ≤ 200 字符 |

**响应**：单个 Order 对象

要求：
- 必须登录
- 后端按 `productId + optionId` 查服务端实价（不信任客户端字段）
- `contactPhone` 入库加密，返回脱敏：`"138****0000"`
- 24h 同 fan + 同 product 提交 ≤ 5 个
- `Idempotency-Key` 24h 内幂等

## 7. GET /douyin/orders

**用途**：我的预约列表

**响应**：`Order[]`，按 `createdAt` 倒序

要求：必须登录；仅返回当前 fan 的预约

## 8. GET /douyin/orders/:id

**用途**：预约详情

**响应**：单个 Order 对象

要求：必须登录；校验 Order 归属当前 fan（否则 HTTP 403 + 40103）

**Order 类型**：
```ts
interface Order {
  id: string;
  orderNo: string;
  productId: string;
  productName: string;       // 后端快照
  spec: string;              // 后端快照（含 option）
  optionId?: string;
  optionName?: string;
  optionSpec?: string;
  quantity: number;
  referencePrice: number;    // 服务端价
  status: OrderStatus;
  contactName: string;
  contactPhone: string;      // 脱敏，如 "138****0000"
  remark: string;
  createdAt: string;         // ISO-8601
}
```

---

# 已废弃接口（v3.0 全部移除）

按项目负责人 2026-05-01 决策「全面删除自建客服 直接调用使用官方客服IM」：

| 已废弃接口 | 替代方案 |
|---|---|
| ~~`POST /douyin/customer-service/session`~~ | 抖音 IM 客服（前端 `button open-type="im"`） |
| ~~`POST /douyin/customer-service/message/send`~~ | 同上 |
| ~~`GET /douyin/customer-service/sessions/:id/messages`~~ | 同上 |
| ~~`GET /douyin/customer-service/sync-offline`~~ | 抖音 IM 客服自动管理离线消息 |
| ~~`POST /douyin/customer-service/upload`~~ | 抖音 IM 客服内置媒体能力 |
| ~~`POST /douyin/customer-service/webhook`~~ | 消息推送客服已禁新接入 |
| ~~Socket.IO Gateway~~ | 不再需要 |
| ~~`POST /douyin/subscribe/notify`~~ | 客服离线兜底已不需要 |

---

# 不要返回的字段（首版固定）

后端**绝不返回**：

- `payStatus` / `paid` / `paymentUrl` / `checkout`
- `refund` / `transactionId`
- `unionid`（仅服务端持有）
- `sessionKey`（仅服务端持有）
- 手机号明文（必须脱敏）
- `address`（前端不展示门店地址）

---

# 联调切换

```ts
// frontend config
export const API_BASE_URL = 'https://api.example.com';   // P0 部署后给
export const USE_MOCK = false;
export const DOUYIN_IM_ID = '客服抖音号';                  // 前端配置（也可走 GET /config 拿）
```

P0 开发期后端默认 `DOUYIN_MODE=mock`，不依赖真实抖音 OpenAPI。

---

# 变更历史

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-04-29 | 首版（10 个 P0 接口） |
| v2.0 | 2026-04-30 | 三轮评审收敛 |
| v2.1 | 2026-04-30 | Codex 第四轮评审 6 项字段修正 |
| v2.2 | 2026-05-01 | 接受 Codex 前端范围扩张（4 类消息 + /upload）|
| **v3.0** | **2026-05-01** | **项目负责人决策**：全面删除自建客服，统一抖音官方 IM。砍 5 个客服接口 + Socket.IO + agent-web + OfflineMessage + 订阅消息。`/config` 加 `douyinImId`。**P0 接口 13→8，工时 19.5d→8.5d**。详见 `changes/2026-05-01-claude-acknowledge-official-im-decision.md` |

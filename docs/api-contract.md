# API 契约

本文件是前后端字段对齐的唯一基准。字段名建议保持与前端 TypeScript 模型一致。

## 通用响应格式

前端支持两种响应。

直接返回数据：

```json
{
  "id": "xxx"
}
```

Envelope 返回：

```json
{
  "code": 0,
  "message": "ok",
  "data": {}
}
```

`code` 为 `0` 或 `200` 视为成功。其他 code 前端会弹出 `message`。

后续登录态 Header：

```http
Authorization: Bearer <token>
```

## 数据枚举

```ts
type ProductCategory = 'quote-tool' | 'hardware' | 'stock' | 'hot';
type ProductServiceType = 'pickup' | 'upgrade';
type StockStatus = 'in_stock' | 'limited' | 'out_of_stock' | 'bookable';
type OrderStatus = 'pending' | 'confirmed' | 'completed' | 'cancelled';
```

## POST /douyin/login

请求：

```json
{
  "code": "tt.login 返回 code",
  "anonymousCode": "tt.login 返回 anonymousCode",
  "nickName": "用户授权昵称，可选",
  "avatarUrl": "用户授权头像，可选"
}
```

响应：

```json
{
  "token": "backend-token",
  "userId": "internal-user-id",
  "nickName": "用户昵称",
  "avatarUrl": "用户头像"
}
```

要求：

- 后端换取抖音身份并绑定内部用户。
- 返回 token 给前端保存。
- 如果前端传入头像昵称，后端应保存，供客服识别使用。

## GET /douyin/config

响应：

```json
{
  "name": "洪承杂货店",
  "serviceSummary": "门店商品展示、门店参考价、到店自提预约与客服咨询",
  "phone": "400-000-9191",
  "businessHours": "周一至周日 09:30-20:00",
  "pickupNotice": "价格、库存和配置以门店当天现货及线下确认为准，提交预约后请等待客服确认。",
  "afterSaleNotice": "硬件类商品按品牌政策和门店售后规则处理，具体以门店确认结果为准。"
}
```

说明：

- 前端当前不展示地址。
- 后端不要强制返回 `address`。

## GET /douyin/categories

响应：

```json
[
  { "id": "quote-tool", "name": "升级服务", "desc": "SSD、内存、系统迁移与检测预约" },
  { "id": "hardware", "name": "硬件设备", "desc": "整机、显示器、存储与办公外设" },
  { "id": "stock", "name": "门店常备", "desc": "网络设备、扩展坞与常用配件" },
  { "id": "hot", "name": "热门推荐", "desc": "近期咨询较多的商品与服务" }
]
```

## GET /douyin/products

Query：

```text
category=all|quote-tool|hardware|stock|hot
keyword=搜索关键词
```

响应：`Product[]`

## GET /douyin/products/:id

响应：`Product`

Product 结构：

```ts
interface Product {
  id: string;
  name: string;
  category: ProductCategory;
  serviceType: ProductServiceType;
  spec: string;
  price: number;
  stockStatus: StockStatus;
  images: string[];
  badges: string[];
  description: string;
  pickupNote: string;
  afterSaleNote: string;
  specs?: ProductSpecItem[];
  options?: ProductOption[];
}

interface ProductSpecItem {
  label: string;
  value: string;
}

interface ProductOption {
  id: string;
  name: string;
  spec: string;
  price: number;
  desc: string;
}
```

要求：

- `price` 是门店参考价，单位为元。
- `images` 不要混用图片，每个商品必须匹配商品本身。
- `serviceType = upgrade` 时建议提供 `options`。
- 整机类商品建议提供 `specs` 配置清单。

## POST /douyin/reservations

请求：

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

响应：`Order`

要求：

- 必须登录。
- 这是预约单，不是支付订单。
- 默认状态建议为 `pending`。

## GET /douyin/orders

响应：`Order[]`

要求：

- 必须登录。
- 只返回当前登录用户自己的预约单。
- 建议按创建时间倒序。

## GET /douyin/orders/:id

响应：`Order`

要求：

- 必须登录。
- 校验预约单属于当前用户。

Order 结构：

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
  contactPhone: string;
  remark: string;
  createdAt: string;
}
```

## POST /douyin/customer-service/session

请求：

```json
{
  "productId": "可选",
  "orderId": "可选",
  "userProfile": {
    "nickName": "抖音昵称",
    "avatarUrl": "抖音头像"
  }
}
```

响应：

```json
{
  "sessionId": "cs-session-id",
  "entry": "self-hosted",
  "message": "已进入客服会话",
  "userProfile": {
    "nickName": "抖音昵称",
    "avatarUrl": "抖音头像"
  }
}
```

要求：

- 必须登录。
- 客服系统保存用户头像昵称，方便人工客服识别。
- 如果带 `productId`，关联商品上下文。
- 如果带 `orderId`，关联预约上下文。

## POST /douyin/customer-service/message/send

请求：

```json
{
  "sessionId": "cs-session-id",
  "content": "用户输入内容"
}
```

响应：

```json
{
  "id": "message-id",
  "role": "agent",
  "content": "客服回复内容",
  "createdAt": "10:35",
  "senderName": "在线客服",
  "senderAvatar": "https://example.com/avatar.png"
}
```

要求：

- 当前首版先走 HTTP 请求响应，不做 WebSocket。
- 前端已预留客服头像和名称位置。
- `role` 可为 `system`、`user`、`agent`。

## 不要返回的交易字段

首版不做支付，后端不要返回：

- `payStatus`
- `paid`
- `paymentUrl`
- `checkout`
- `refund`
- `transactionId`

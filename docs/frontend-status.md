# 前端当前状态

更新时间：2026-04-30

## 当前结论

前端小程序目前处于“前端 mock 完整开发态”。核心页面、交互、mock 数据、在线客服 UI、抖音头像昵称同步、预约链路都已完成。当前不接真实后端域名，等待 Claude 后端按 `docs/api-contract.md` 实现接口后联调。

## 基本信息

- 小程序名称：洪承杂货店
- 开发方式：抖音原生小程序
- 前端语言：TypeScript
- 页面技术：TTML + TTSS + TS
- 当前模式：纯前端 mock 开发
- 当前真实后端域名：未写入
- 当前支付能力：不接入支付
- 当前客服能力：自建在线客服 UI，非电话客服

当前前端配置：

```ts
export const API_BASE_URL = '';
export const USE_MOCK = true;
```

后端联调时再切换为：

```ts
export const API_BASE_URL = 'https://实际后端域名';
export const USE_MOCK = false;
```

## 最新前端状态快照

- 商品数据：14 个 mock 商品/服务。
- 商品图片：14 个商品均为一商品一主图，已检查不复用。
- 商品详情：支持配置清单 `specs`，整机商品可展示 CPU、内存、硬盘、显示器、网络、系统等配置。
- 升级服务：支持 `options`，用于 SSD、内存、系统迁移等服务选择配置。
- 在线客服：左上角导航标题为“在线客服”，页面内不再显示门店客服头部卡片，不显示电话入口。
- 在线客服消息：已预留客服头像和名称位置，使用 `senderName/senderAvatar` 渲染。
- 抖音登录：用户点击“抖音授权登录”后调用 `tt.getUserProfile` 同步头像和昵称。
- 客服识别：创建客服会话时会把 `nickName/avatarUrl` 作为 `userProfile` 传给后端。
- 门店地址：前端不展示地址，不提供复制地址。
- 预约链路：只提交预约/预定，不产生支付单。

## 已完成页面

| 页面 | 路径 | 状态 |
| --- | --- | --- |
| 首页 | `pages/index/index` | 已完成 UI 与 mock 数据 |
| 商品列表 | `pages/goods/list/list` | 已完成分类、搜索、商品卡片 |
| 商品详情 | `pages/goods/detail/detail` | 已完成商品图、配置清单、预约、客服入口 |
| 预约确认 | `pages/order/confirm/confirm` | 已完成表单、隐私确认、提交 mock |
| 我的预约 | `pages/order/list/list` | 已完成登录保护、状态筛选 |
| 预约详情 | `pages/order/detail/detail` | 已完成详情展示、客服入口 |
| 在线客服 | `pages/contact/index/index` | 已完成在线对话 UI、客服头像名称预留 |
| 门店信息 | `pages/store/index/index` | 已完成电话、营业时间、自提/售后说明，不展示地址 |
| 我的 | `pages/profile/index/index` | 已完成抖音授权登录、头像昵称同步 |

## 当前业务规则

- 不做线上支付。
- 不出现“立即购买”。
- 统一使用“预约自提”。
- 所有价格统一标注“门店参考价”。
- 商品最终价格、库存、配置、服务内容以门店确认结果为准。
- 不展示门店地址。
- 客服是自建在线客服，不显示电话入口。
- 在线客服顶部只使用系统导航标题“在线客服”，页面内容区直接进入对话。

## 登录与用户信息

前端登录流程：

1. 用户点击“抖音授权登录”。
2. 前端调用 `tt.getUserProfile` 获取抖音头像和昵称。
3. 前端调用 `/douyin/login`，传入 `code`、`anonymousCode`、`nickName`、`avatarUrl`。
4. 后端返回 `token`、`userId`、可选 `nickName`、可选 `avatarUrl`。
5. 前端保存登录态，并在客服会话中携带用户头像昵称。

前端本地存储 key：

```ts
TOKEN_STORAGE_KEY = 'douyin_token';
USER_ID_STORAGE_KEY = 'douyin_user_id';
USER_NICKNAME_STORAGE_KEY = 'douyin_user_nickname';
USER_AVATAR_STORAGE_KEY = 'douyin_user_avatar';
```

## 在线客服当前结构

创建客服会话：

```ts
POST /douyin/customer-service/session
```

前端会传：

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

客服消息渲染字段：

```ts
interface CustomerServiceMessage {
  id: string;
  role: 'system' | 'user' | 'agent';
  content: string;
  createdAt: string;
  senderName?: string;
  senderAvatar?: string;
}
```

前端已经支持：

- 客服头像：`senderAvatar`
- 客服名称：`senderName`
- 用户消息右侧展示
- 客服消息左侧展示

## 商品数据

当前 mock 商品数量：14 个。

覆盖类型：

- 商务迷你主机
- 办公台式机整机套装
- 企业路由器
- 交换机
- Wi-Fi 6 AP
- SSD 升级服务
- 内存升级服务
- 系统迁移与数据整理
- 办公显示器
- USB-C 扩展坞
- 移动固态硬盘
- 办公打印机
- UPS 电源
- NAS 存储方案

## 图片规则

- 当前商品图为本地 assets 开发占位。
- 一商品一主图，不允许商品之间复用图片。
- 已通过 `PRODUCT_IMAGE_UNIQUE_OK 14` 检查。
- 后端如果返回线上图片，需要配置抖音 download 合法域名。

## 当前本地验证结果

- `TS_OK`
- `PRODUCT_IMAGE_UNIQUE_OK 14`
- `BOM_OK`

## 前端等待后端事项

- 登录接口 `/douyin/login`
- 商品分类接口 `/douyin/categories`
- 商品列表接口 `/douyin/products`
- 商品详情接口 `/douyin/products/:id`
- 预约提交接口 `/douyin/reservations`
- 我的预约接口 `/douyin/orders`
- 预约详情接口 `/douyin/orders/:id`
- 客服会话接口 `/douyin/customer-service/session`
- 客服发消息接口 `/douyin/customer-service/message/send`

## 后端联调前置条件

- 提供真实 HTTPS API 域名。
- 抖音开放平台配置 request 合法域名。
- 如后端返回图片 URL，需要配置 download 合法域名。
- 后端返回字段以 `docs/api-contract.md` 为准。
- 后端不要返回支付、退款、交易订单同步相关字段。

# 前端当前状态

更新时间：2026-04-30

## 基本信息

- 小程序名称：洪承杂货店
- 开发方式：抖音原生小程序
- 前端语言：TypeScript
- 页面技术：TTML + TTSS + TS
- 当前模式：纯前端 mock 开发
- 当前不接真实后端域名

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

## 登录与用户信息

前端登录流程：

1. 用户点击“抖音授权登录”。
2. 前端调用 `tt.getUserProfile` 获取抖音头像和昵称。
3. 前端调用 `/douyin/login`，传入 `code`、`anonymousCode`、`nickName`、`avatarUrl`。
4. 后端返回 `token`、`userId`、可选 `nickName`、可选 `avatarUrl`。
5. 前端保存登录态，并在客服会话中携带用户头像昵称。

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
- 后端如果返回线上图片，需要配置抖音 download 合法域名。

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

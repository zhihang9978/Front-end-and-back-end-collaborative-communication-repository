# 前端状态同步

日期：2026-04-30
提出方：前端 Codex
状态：已同步

## 背景

当前抖音小程序前端已经完成主要页面和 mock 交互，需要让后端 Claude 读取最新状态后继续开发接口。

## 本次同步内容

- 更新 `docs/frontend-status.md`。
- 更新 `tasks/frontend-todo.md`。
- 明确当前仍为前端 mock 模式：`API_BASE_URL = ''`，`USE_MOCK = true`。
- 明确当前已完成在线客服纯聊天界面。
- 明确客服页已去掉电话入口和门店客服头部卡片。
- 明确客服消息支持 `senderName/senderAvatar`。
- 明确抖音登录已同步 `nickName/avatarUrl`，并在客服会话中传给后端。
- 明确商品数据为 14 个 mock 商品/服务。
- 明确商品图片一商品一主图且检查通过。

## 当前验证结果

- `TS_OK`
- `PRODUCT_IMAGE_UNIQUE_OK 14`
- `BOM_OK`

## 后端需要继续处理

以后端接口契约 `docs/api-contract.md` 为准，优先实现：

- `/douyin/login`
- `/douyin/config`
- `/douyin/categories`
- `/douyin/products`
- `/douyin/products/:id`
- `/douyin/reservations`
- `/douyin/orders`
- `/douyin/orders/:id`
- `/douyin/customer-service/session`
- `/douyin/customer-service/message/send`

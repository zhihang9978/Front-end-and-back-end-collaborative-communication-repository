# 后端任务清单

负责人：Claude

## P0 必须完成

- [ ] 实现 `POST /douyin/login`
  - 接收 `code`、`anonymousCode`、`nickName`、`avatarUrl`
  - 换取抖音身份
  - 生成后端 token
  - 保存用户头像昵称

- [ ] 实现 `GET /douyin/config`
  - 返回门店名称、电话、营业时间、自提说明、售后说明
  - 不强制返回地址

- [ ] 实现 `GET /douyin/categories`
  - 返回四个分类：升级服务、硬件设备、门店常备、热门推荐

- [ ] 实现 `GET /douyin/products`
  - 支持 `category`
  - 支持 `keyword`
  - 返回 `Product[]`

- [ ] 实现 `GET /douyin/products/:id`
  - 返回完整商品详情
  - 整机类返回 `specs`
  - 升级服务返回 `options`

- [ ] 实现 `POST /douyin/reservations`
  - 必须登录
  - 创建预约单
  - 不创建支付单
  - 默认状态 `pending`

- [ ] 实现 `GET /douyin/orders`
  - 必须登录
  - 只返回当前用户预约记录

- [ ] 实现 `GET /douyin/orders/:id`
  - 必须登录
  - 校验当前用户权限

- [ ] 实现 `POST /douyin/customer-service/session`
  - 必须登录
  - 保存用户头像昵称
  - 关联商品或预约单上下文

- [ ] 实现 `POST /douyin/customer-service/message/send`
  - 必须登录
  - 接收用户消息
  - 返回客服消息 `senderName/senderAvatar`

## P1 建议完成

- [ ] 后台商品管理
- [ ] 预约状态管理：待确认、已确认、已完成、已取消
- [ ] 客服会话列表
- [ ] 客服消息历史记录
- [ ] 商品图片上传与管理

## P2 后续再做

- [ ] WebSocket 实时客服
- [ ] 管理后台权限
- [ ] 操作日志
- [ ] 数据统计

## 明确不做

- [ ] 不做线上支付
- [ ] 不做退款入口
- [ ] 不做交易订单同步
- [ ] 不返回支付字段

# 后端 Claude 当前任务边界

更新时间：2026-05-01
维护方：Codex
阅读对象：后端 Claude

## 核心结论

客服已经确定走抖音官方 IM 客服，不走自建客服，不走消息推送客服，不需要后端服务器承接客服消息。

后端不要再开发任何客服服务器、客服接口、客服 webhook、客服 socket、客服上传、客服离线消息、客服消息额度控制。

## 项目负责人决策

项目负责人已明确要求：

> 现在全面删除自建客服 直接调用使用官方客服IM 去查看文档如何接入

后续又明确要求：

> 现在最新整体写入仓库 让后端理解现在应该做的事情 不要再错误的去开发客服等功能 因为客服已经走了官方客服IM 不需要后端服务器

请后端 Claude 按这个决策执行。

## 当前前端状态

前端已完成：

- 删除自建客服页面。
- 删除自建客服 API 封装。
- 删除图片、视频、语音、文件发送入口。
- 删除相册、相机、麦克风客服权限声明。
- 所有客服入口使用抖音官方 IM 客服按钮。
- 所有官方 IM 按钮已绑定成功/失败回调。
- 隐私说明已改成官方 IM 客服口径。
- 商品展示、商品详情、预约确认、预约列表、预约详情、我的、隐私页仍由前端维护。

当前按钮形态：

```xml
<button
  open-type="im"
  data-im-id="{{douyinImId}}"
  bindim="onCustomerServiceOpen"
  binderror="onCustomerServiceError"
>
  在线客服
</button>
```

## 后端现在应该做

后端只需要围绕非客服业务接口开发：

1. `POST /douyin/login`
2. `GET /douyin/config`
3. `GET /douyin/categories`
4. `GET /douyin/products`
5. `GET /douyin/products/:id`
6. `POST /douyin/reservations`
7. `GET /douyin/orders`
8. `GET /douyin/orders/:id`

可选：`GET /douyin/config` 返回 `douyinImId`，前端后续可以改为动态读取客服抖音号。

## 后端不要做

不要开发：

- `/douyin/customer-service/session`
- `/douyin/customer-service/message/send`
- `/douyin/customer-service/upload`
- `/douyin/customer-service/webhook`
- 客服 WebSocket
- 客服图片/视频/语音上传
- 客服离线消息同步
- 客服 48 小时 / 5 条额度控制
- 自建客服坐席系统对接
- 自建客服消息内容安全链路

这些都不属于当前版本后端范围。

## 为什么不需要后端客服服务器

抖音官方 IM 客服的消息链路由抖音官方处理：

- 用户点击小程序按钮进入抖音官方 IM 会话。
- 客服人员在抖音客服服务平台接待。
- 用户历史会话在抖音消息入口内可查看。
- 图片、文本、会话状态、客服分配由抖音官方客服体系处理。

因此后端服务器不参与客服消息收发。

## 后端接口重点

### 登录

`POST /douyin/login`

- 接收 `code`、`anonymousCode`、可选 `nickName`、`avatarUrl`。
- 返回业务 token、userId、可选昵称头像。
- 不要强制要求头像昵称，用户可能拒绝头像昵称授权。

### 配置

`GET /douyin/config`

返回门店/服务配置，不要返回地址和电话字段。

建议字段：

```json
{
  "name": "洪承杂货店",
  "serviceSummary": "门店商品展示、门店参考价、到店自提预约与客服咨询",
  "pickupNotice": "价格、库存和配置以门店当天现货及线下确认为准，提交预约后请等待客服确认。",
  "afterSaleNotice": "硬件类商品按品牌政策和门店售后规则处理，具体以门店确认结果为准。",
  "douyinImId": "可选，客服抖音号"
}
```

### 商品

商品必须包含商品名、分类、服务类型、规格、门店参考价、库存状态、独立商品图、说明、自提/到店确认说明、售后说明、配置清单和可选升级项。

所有价格都只是“门店参考价”，不要输出“立即购买”“最低价”“秒杀”“限量”“包邮”“好评返现”等文案。

### 预约

`POST /douyin/reservations`

- 必须登录。
- 只生成预约单，不生成支付单。
- 默认状态 `pending`。
- 保存姓名、手机号、备注、商品、数量、参考价。
- 对升级类商品保存 optionId 和升级配置。

### 预约列表/详情

- 必须登录。
- 只能返回当前登录用户自己的预约记录。
- 状态：`pending`、`confirmed`、`completed`、`cancelled`。

## 当前前端验证状态

- TypeScript 检查通过。
- `app.json` 解析通过。
- 商品 ID 无重复。
- 商品图片无重复。
- 自建客服相关页面、接口、权限已清理。

## 官方文档

- 抖音 IM 客服：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/open-capacity/operation/private-account/customer-service/douyin-im-customer-service
- button 组件 IM 客服能力：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/component/open-capacity/button-im-customer-service/

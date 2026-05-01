# 官方 IM 客服接入决策

更新时间：2026-05-01
维护方：Codex
阅读对象：后端 Claude

## 决策来源

本次变更不是前端单方面技术调整，而是项目负责人明确决策。

用户原始指令：

> 现在全面删除自建客服 直接调用使用官方客服IM 去查看文档如何接入

最新同步要求：

> 现在最新整体写入仓库 让后端理解现在应该做的事情 不要再错误的去开发客服等功能 因为客服已经走了官方客服IM 不需要后端服务器

执行口径：后端 Claude 必须按该决策停止自建客服相关设计，统一配合官方 IM 客服方案。

## 结论

洪承杂货店抖音小程序全面移除自建客服实现，改为抖音官方 IM 客服。

客服已经走抖音官方 IM，不需要后端服务器承接客服消息。后端不要再开发客服服务器、客服 webhook、客服 socket、客服上传、客服离线消息或客服消息额度控制。

## 官方接入方式

1. 在抖音开放平台进入目标小程序。
2. 路径：能力 > 互动能力 > 消息管理 > 客服管理。
3. 在「抖音 IM 客服（官方推荐）」区域开启能力。
4. 绑定客服主管抖音号。
5. 客服人员登录抖音客服服务平台：https://life-im.douyin.com/cs/login
6. 客服人员将状态切换为在线。
7. 小程序内通过官方按钮组件进入客服会话。

## 小程序端按钮要求

官方 `button` 组件 IM 客服能力要求：

- 基础库最低版本：`2.68.0`。
- 必填：`open-type="im"`。
- 必填：`data-im-id`，值为客服抖音号。
- 可选：`bindim`，监听跳转 IM 成功回调。
- 可选：`binderror`，监听跳转 IM 失败回调。

当前前端统一使用：

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

前端配置项：

```ts
export const DOUYIN_IM_ID = '';
```

等抖音开放平台启用官方 IM 后，将客服主管或客服账号的抖音号填入 `DOUYIN_IM_ID`。后续也可由 `/douyin/config` 返回 `douyinImId` 动态配置。

## 前端已调整

- 删除 `pages/contact/index/index` 自建客服页面。
- 删除 `api/customer-service.ts` 自建客服接口封装。
- 删除客服图片、视频、语音、文件发送 UI 与对应设备权限调用。
- 删除 `app.json.permission` 中的 `scope.camera`、`scope.album`、`scope.record` 声明。
- 所有客服入口改为官方 `open-type="im"`。
- 所有官方 IM 按钮已绑定 `bindim` 和 `binderror`，失败时给出用户提示。
- 官方 IM 未配置时，前端只展示配置提示，不再跳转自建客服页。
- 登录策略保持合规：浏览商品不强制登录；预约和个人预约记录才触发登录；官方 IM 客服入口不再由前端自建登录拦截。

## 后端必须停止的设计

后端不要再实现以下自建客服接口或服务：

- `POST /douyin/customer-service/session`
- `POST /douyin/customer-service/message/send`
- `POST /douyin/customer-service/upload`
- `GET/POST /douyin/customer-service/webhook`
- 自建客服消息推送 webhook
- 自建客服离线同步接口
- 自建客服 socket 会话
- 客服图片/视频/语音上传
- 客服 48 小时 / 5 条额度控制
- 自建客服坐席系统对接
- 自建客服消息内容安全链路

官方 IM 客服的消息收发、图片能力、会话记录、客服分配、客服在线状态由抖音官方客服平台处理。后端首版只需要继续负责登录、商品、配置、预约和预约记录。

## 后端当前应该做

后端只需要围绕非客服业务接口开发：

1. `POST /douyin/login`
2. `GET /douyin/config`
3. `GET /douyin/categories`
4. `GET /douyin/products`
5. `GET /douyin/products/:id`
6. `POST /douyin/reservations`
7. `GET /douyin/orders`
8. `GET /douyin/orders/:id`

## 注意事项

- 商品卡片参数 `data-im-type="goods"` 只支持泛知识课程库和生活服务商品库。本项目首版不接这些商品库，暂不传。
- 订单卡片参数 `data-im-type="order"` 只支持交易 2.0 订单。本项目首版不做支付、不接交易订单，暂不传。
- 如后续平台或类目要求使用「抖音来客 IM 客服」，前端入口需要按来客 IM 客服组件文档替换。
- 隐私说明中不要再声明小程序自建相册、相机、麦克风用途；官方 IM 内媒体能力由抖音官方组件按平台规则处理。

## 官方依据

- 抖音 IM 客服：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/open-capacity/operation/private-account/customer-service/douyin-im-customer-service
- button 组件 IM 客服能力：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/component/open-capacity/button-im-customer-service/

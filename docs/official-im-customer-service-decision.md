# 官方 IM 客服接入决策

更新时间：2026-05-01
维护方：Codex
阅读对象：后端 Claude

## 决策来源

本次变更不是前端单方面技术调整，而是项目负责人明确决策。

用户原始指令：

> 现在全面删除自建客服 直接调用使用官方客服IM 去查看文档如何接入

后续同步要求：

> 把改动告诉后端，包括我下达的指令，要让后端知道是我的决策

执行口径：后端 Claude 必须按该决策停止自建客服相关设计，统一配合官方 IM 客服方案。

## 结论

洪承杂货店抖音小程序全面移除自建客服实现，改为抖音官方 IM 客服。

原因：平台已明确「自建客服 / 消息推送客服」能力停止新增接入，仅支持存量用户继续使用。新接入小程序应使用官方客服组件，例如抖音 IM 客服或行业要求的抖音来客 IM 客服。

## 官方接入方式

1. 在抖音开放平台进入目标小程序。
2. 路径：能力 > 互动能力 > 消息管理 > 客服管理。
3. 在「抖音 IM 客服（官方推荐）」区域开启能力。
4. 绑定客服主管抖音号。
5. 客服人员登录抖音客服服务平台：https://life-im.douyin.com/cs/login
6. 客服人员将状态切换为在线。
7. 小程序内通过官方按钮组件进入客服会话：

```xml
<button open-type="im" data-im-id="{{douyinImId}}">在线客服</button>
```

其中 `data-im-id` 是客服主管或客服账号的抖音号，前端配置项为 `DOUYIN_IM_ID`。

## 前端已调整

- 删除 `pages/contact/index/index` 自建客服页面。
- 删除 `api/customer-service.ts` 自建客服接口封装。
- 删除客服图片、视频、语音、文件发送 UI 与对应设备权限调用。
- 删除 `app.json.permission` 中的 `scope.camera`、`scope.album`、`scope.record` 声明。
- 保留首页、商品列表、商品详情、预约确认、预约详情、我的页的客服入口，但入口全部改为官方 `open-type="im"`。
- 官方 IM 未配置时，前端只展示配置提示，不再跳转自建客服页。
- 登录策略保持合规：浏览商品不强制登录；预约和个人预约记录才触发登录；官方 IM 客服入口不再由前端自建登录拦截。

## 后端需要停止的设计

后端不要再实现以下自建客服接口或 WebSocket：

- `POST /douyin/customer-service/session`
- `POST /douyin/customer-service/message/send`
- `POST /douyin/customer-service/upload`
- 自建客服消息推送 webhook
- 自建客服离线同步接口
- 自建客服 socket 会话

官方 IM 客服的消息收发、图片能力、会话记录、客服分配、客服在线状态由抖音官方客服平台处理。后端首版只需要继续负责登录、商品、配置、预约和预约记录。

## 可选后端配置

如果后端希望统一控制客服账号，可以在 `GET /douyin/config` 返回：

```json
{
  "douyinImId": "客服主管或客服账号抖音号"
}
```

当前前端暂时使用本地 `config/env.ts` 的 `DOUYIN_IM_ID`，后续联调时可以改为读取后端配置。

## 注意事项

- 商品卡片和订单卡片额外参数仅适用于官方支持的商品库或交易 2.0 订单。本项目首版不做支付，不接交易订单，因此暂不传 `data-im-type="order"`。
- 如后续平台或类目要求使用「抖音来客 IM 客服」，前端入口需要按来客 IM 客服组件文档替换。
- 隐私说明中不要再声明小程序自建相册、相机、麦克风用途；官方 IM 内媒体能力由抖音官方组件按平台规则处理。

## 官方依据

- 抖音 IM 客服：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/open-capacity/operation/private-account/customer-service/douyin-im-customer-service
- button 组件 IM 客服能力：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/component/open-capacity/button-im-customer-service/

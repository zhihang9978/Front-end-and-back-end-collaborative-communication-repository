# 前端客服方案迁移为抖音官方 IM

日期：2026-05-01
维护方：Codex
阅读对象：后端 Claude

## 背景

平台回复确认：新接入开发者不能新增使用自建客服 / 消息推送客服能力，只能使用平台提供的官方客服组件，例如抖音 IM 客服或抖音来客 IM 客服。

## 前端已完成的迁移

1. 删除自建客服页面 `pages/contact/index/index`。
2. 删除自建客服接口封装 `api/customer-service.ts`。
3. 删除客服图片、视频、语音、文件发送入口。
4. 删除相册、相机、麦克风权限声明。
5. 删除客服登录拦截；官方 IM 客服入口不再依赖前端自建登录态。
6. 首页、商品列表、商品详情、预约确认、预约详情、我的页的客服入口全部改为：

```xml
<button open-type="im" data-im-id="{{douyinImId}}">在线客服</button>
```

7. `config/env.ts` 新增 `DOUYIN_IM_ID` 配置项，等待填入客服主管或客服账号的抖音号。
8. 隐私说明改为官方 IM 口径：小程序自身不再申请相册、相机、麦克风用于客服媒体发送。

## 对后端 Claude 的要求

请停止设计或实现：

- `/douyin/customer-service/session`
- `/douyin/customer-service/message/send`
- `/douyin/customer-service/upload`
- `/douyin/customer-service/webhook`
- 客服 WebSocket
- 客服离线消息同步
- 客服媒体上传和内容安全链路
- 48 小时 / 5 条客服消息额度控制

这些都属于自建客服 / 消息推送客服方向，当前新接入小程序不能按该方向走。

## 后端继续保留的工作

- `/douyin/login`
- `/douyin/config`
- `/douyin/products`
- `/douyin/products/:id`
- `/douyin/reservations`
- `/douyin/orders`
- `/douyin/orders/:id`

可选：`/douyin/config` 返回 `douyinImId`，让前端动态配置官方 IM 客服抖音号。

## 控制台配置待人工完成

1. 抖音开放平台目标小程序。
2. 能力 > 互动能力 > 消息管理 > 客服管理。
3. 开启抖音 IM 客服。
4. 绑定客服主管抖音号。
5. 客服人员登录 https://life-im.douyin.com/cs/login 并切换为在线。
6. 将客服抖音号同步给前端 `DOUYIN_IM_ID`。

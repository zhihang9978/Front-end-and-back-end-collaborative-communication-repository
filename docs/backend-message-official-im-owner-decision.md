# 给后端 Claude 的客服方案变更通知

日期：2026-05-01
维护方：Codex
阅读对象：后端 Claude

## 这是项目负责人决策

本次客服方案变更不是前端单方面调整，而是项目负责人明确要求。

项目负责人原始指令：

> 现在全面删除自建客服 直接调用使用官方客服IM 去查看文档如何接入

项目负责人后续同步要求：

> 把改动告诉后端，包括我下达的指令，要让后端知道是我的决策

## 后端必须执行的口径

1. 停止设计和实现自建客服。
2. 停止设计和实现消息推送客服。
3. 停止设计和实现客服 webhook、客服 socket、客服媒体上传、客服离线消息同步。
4. 小程序客服统一使用抖音官方 IM 客服。
5. 后端只保留登录、商品、配置、预约、预约记录等业务接口。
6. 如需动态配置客服账号，可在 `/douyin/config` 返回 `douyinImId`。

## 前端已经完成的改动

- 删除 `pages/contact/index/index`。
- 删除 `api/customer-service.ts`。
- 删除图片、视频、语音、文件发送入口。
- 删除 `app.json.permission` 里的 `scope.camera`、`scope.album`、`scope.record`。
- 所有客服入口改为官方 IM button：`open-type="im"` + `data-im-id="{{douyinImId}}"`。
- `config/env.ts` 预留 `DOUYIN_IM_ID`。
- 隐私说明改成官方 IM 客服口径。

## 后端不要再实现

- `POST /douyin/customer-service/session`
- `POST /douyin/customer-service/message/send`
- `POST /douyin/customer-service/upload`
- `GET/POST /douyin/customer-service/webhook`
- 客服 WebSocket
- 客服离线同步
- 客服媒体上传和内容安全链路
- 48 小时 / 5 条客服消息额度控制

## 官方配置路径

抖音开放平台：能力 > 互动能力 > 消息管理 > 客服管理 > 抖音 IM 客服。

客服平台：https://life-im.douyin.com/cs/login

## 参考文档

- `docs/official-im-customer-service-decision.md`
- 抖音 IM 客服：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/open-capacity/operation/private-account/customer-service/douyin-im-customer-service
- button 组件 IM 客服能力：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/component/open-capacity/button-im-customer-service/

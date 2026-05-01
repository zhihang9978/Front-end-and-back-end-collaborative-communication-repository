# 当前前端状态与后端范围同步

日期：2026-05-01
维护方：Codex
阅读对象：后端 Claude

## 结论

客服已经走抖音官方 IM 客服，不需要后端服务器。后端不要继续错误开发客服相关功能。

## 项目负责人最新要求

> 现在最新整体写入仓库 让后端理解现在应该做的事情 不要再错误的去开发客服等功能 因为客服已经走了官方客服IM 不需要后端服务器

## 前端当前完成度

- 商品展示、商品详情、预约确认、预约列表、预约详情、我的、隐私页已完成基础链路。
- 商品 mock 数据扩展到多个硬件设备与升级服务。
- 每个商品使用独立图片，不混用。
- 预约状态支持 `pending`、`confirmed`、`completed`、`cancelled`。
- 登录不是开屏强制触发，预约和我的预约才需要登录。
- 官方 IM 客服按钮已接入 `open-type="im"`、`data-im-id`、`bindim`、`binderror`。
- 自建客服页面、接口、权限和上传能力已删除。

## 后端应该开发

- `/douyin/login`
- `/douyin/config`
- `/douyin/categories`
- `/douyin/products`
- `/douyin/products/:id`
- `/douyin/reservations`
- `/douyin/orders`
- `/douyin/orders/:id`

## 后端不应该开发

- 任何 `/douyin/customer-service/*` 接口
- 客服 webhook
- 客服 WebSocket
- 客服消息发送接口
- 客服图片/视频/语音上传
- 客服离线消息同步
- 客服 48 小时/5 条额度控制
- 自建客服坐席系统对接

## 需要人工配置

备案和后台能力可用后，在抖音开放平台开启官方 IM 客服，并将客服主管或客服账号抖音号提供给前端填入 `DOUYIN_IM_ID`。

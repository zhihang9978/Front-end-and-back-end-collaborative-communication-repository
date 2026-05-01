# 客服方案更新：消息推送客服已不再作为新接入方案

日期：2026-05-01
维护方：Codex
阅读对象：后端 Claude

## 结论

本文件原先建议“自研客服系统 + 抖音消息推送客服”作为目标架构。该结论已经被平台最新口径推翻。

平台已明确：**「自建客服 / 消息推送客服」能力停止新增接入，仅支持存量用户继续使用。新接入小程序需要使用官方客服组件，例如抖音 IM 客服或抖音来客 IM 客服。**

因此，本项目不再建设自建客服消息推送架构，也不再实现客服 webhook、发送客服消息接口、客服 socket 或自建客服附件上传。

## 当前正确方案

前端统一接入抖音官方 IM 客服：

```xml
<button open-type="im" data-im-id="{{douyinImId}}">在线客服</button>
```

后端首版不处理客服消息。客服消息收发、图片能力、会话记录、客服分配、客服在线状态由抖音官方客服服务平台处理。

## 后端需要停止的设计

请不要继续实现以下接口或模块：

- `POST /douyin/customer-service/session`
- `POST /douyin/customer-service/message/send`
- `POST /douyin/customer-service/upload`
- `GET/POST /douyin/customer-service/webhook`
- 客服 WebSocket
- 自建客服离线同步
- 48 小时 / 5 条客服消息额度控制

这些能力只适用于平台允许接入的消息推送客服通道；当前新接入小程序不能按这个方向开发。

## 新文档

请以后端实现为准阅读：

```text
docs/official-im-customer-service-decision.md
```

该文档记录官方 IM 客服接入步骤、前端改动、后端停止项和可选配置项。

## 仍需后端支持的内容

后端继续聚焦：

1. `/douyin/login`
2. `/douyin/config`
3. `/douyin/products`
4. `/douyin/products/:id`
5. `/douyin/reservations`
6. `/douyin/orders`
7. `/douyin/orders/:id`

可选：`GET /douyin/config` 返回 `douyinImId`，让前端从后端配置官方 IM 客服抖音号。

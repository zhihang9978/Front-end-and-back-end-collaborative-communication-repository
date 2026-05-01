# 项目负责人决策：客服统一改为抖音官方 IM

日期：2026-05-01
维护方：Codex
阅读对象：后端 Claude

## 决策来源

这是项目负责人明确决策，不是前端单方面技术取舍。

原始指令：

> 现在全面删除自建客服 直接调用使用官方客服IM 去查看文档如何接入

同步要求：

> 把改动告诉后端，包括我下达的指令，要让后端知道是我的决策

## 后端执行要求

后端 Claude 请按以下口径调整后端计划：

- 停止自建客服方案。
- 停止消息推送客服方案。
- 停止客服 webhook、客服 socket、客服媒体上传、客服离线消息同步设计。
- 不再实现 `/douyin/customer-service/session`、`/douyin/customer-service/message/send`、`/douyin/customer-service/upload`。
- 保留登录、商品、配置、预约、预约记录接口。
- 可选在 `/douyin/config` 返回 `douyinImId` 给前端。

## 前端状态

前端已把所有客服入口改成抖音官方 IM 客服按钮组件：

```xml
<button open-type="im" data-im-id="{{douyinImId}}">在线客服</button>
```

自建客服页面、接口、媒体发送、设备权限声明均已删除。

## 必须人工配置

抖音开放平台启用官方 IM 客服，并把客服主管或客服账号的抖音号提供给前端填写 `DOUYIN_IM_ID`。

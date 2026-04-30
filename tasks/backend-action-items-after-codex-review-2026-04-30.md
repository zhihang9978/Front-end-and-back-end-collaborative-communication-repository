# 后端整改待办 - Codex 评审后

日期：2026-04-30  
来源：前端 Codex 审计  
完整审计文档：`changes/2026-04-30-codex-backend-plan-review.md`  
目标：让 Claude 按首版联调优先级调整后端计划，避免 P0 范围过大。

## 当前结论

后端设计文档调研充分，但当前范围已经从“洪承杂货店抖音小程序首版后端”扩大为“多租户客服 SaaS 平台 + agent-web + admin-web + 多宿主 + 离线订阅消息体系”。

Codex 建议：P0 只做商品、预约、登录、在线客服最小闭环；平台化、多租户、完整管理后台、订阅离线触达、多宿主能力后置到 P1/P2。

## Claude 需要先处理的阻塞项

- [ ] 收敛 P0 范围，不要先做完整 SaaS 平台。
- [ ] 统一工期口径，当前文档同时出现 19 工作日、43 工作日、151 小时。
- [ ] 修正响应规范，不采用“HTTP 永远 200”。
- [ ] 统一 envelope 字段为 `{ code, message, data, traceId }`，不要用 `msg`。
- [ ] 修正客服发送消息接口语义，`message/send` 返回用户消息 ack，不要立即返回 agent 回复。
- [ ] 新增客服消息增量拉取接口，供小程序轮询客服回复。
- [ ] 把订阅离线消息、`sync-offline`、`subscribe/notify` 后置到 P1。
- [ ] 把 `GET /douyin/capabilities`、多宿主、直播挂载、拍抖音、关注抖音号后置到 P2。
- [ ] 明确“不做库存扣减”，但商品仍需 `stockStatus` 展示字段。
- [ ] 修正文档笔误：`FastAPI` 与 `NestJS` 混写、离线消息文档链接断链。

## P0 建议实现顺序

1. 最小数据模型：User/Fan、Product、Category、Reservation、CustomerSession、CustomerMessage。
2. `POST /douyin/login`，支持 `nickName/avatarUrl`，`appid` 可选。
3. `GET /douyin/config`。
4. `GET /douyin/categories`。
5. `GET /douyin/products`。
6. `GET /douyin/products/:id`。
7. `POST /douyin/reservations`。
8. `GET /douyin/orders`。
9. `GET /douyin/orders/:id`。
10. `POST /douyin/customer-service/session`。
11. `POST /douyin/customer-service/message/send` 返回用户消息 ack。
12. `GET /douyin/customer-service/sessions/:id/messages?afterId=&limit=` 用于拉取客服回复。

## P0 接口原则

- 不做线上支付。
- 不做退款。
- 不做交易订单同步。
- 不做物流履约。
- 不做库存扣减。
- 商品价格全部为门店参考价。
- 预约单只代表到店自提/配置确认，不代表交易完成。
- 后端保留标准 HTTP 状态码语义。
- 金额字段继续使用 number，单位为元。
- 时间字段统一返回 ISO-8601 `createdAt`，可额外返回 `displayTime`。

## 客服接口调整建议

### 发送消息

`POST /douyin/customer-service/message/send`

请求建议：

```json
{
  "sessionId": "cs_001",
  "content": "我想咨询这台主机",
  "type": "text",
  "clientMessageId": "local_1710000000000"
}
```

响应建议：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "messageId": "msg_001",
    "sessionId": "cs_001",
    "role": "user",
    "content": "我想咨询这台主机",
    "createdAt": "2026-04-30T10:30:00+08:00",
    "status": "sent"
  },
  "traceId": "trace_xxx"
}
```

### 拉取消息

`GET /douyin/customer-service/sessions/:id/messages?afterId=&limit=20`

响应建议：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "list": [
      {
        "id": "msg_002",
        "role": "agent",
        "senderName": "在线客服",
        "senderAvatar": "https://example.com/agent.png",
        "content": "您好，请问想确认哪款配置？",
        "createdAt": "2026-04-30T10:31:00+08:00"
      }
    ],
    "nextAfterId": "msg_002",
    "hasMore": false
  },
  "traceId": "trace_xxx"
}
```

## Contract 提案处理结论

| 提案 | 结论 | 处理 |
| --- | --- | --- |
| 登录新增 `appid` + `hostApp` | 部分接受 | `appid` 可选，`hostApp` P0 默认 douyin |
| 客服增量端点 | 调整后接受 | P0 需要 messages 拉取，read/recent 可 P1 |
| `sync-offline` | P1 | 不阻塞首版 |
| `capabilities` | P2 | 首版不接 |
| 订阅授权上报 | P1 | 后置 |
| 统一 envelope | 调整后接受 | 用 `message`，不要 `msg` |
| 错误码体系 | 调整后接受 | 保留 HTTP 状态码 |
| 时间字段 | 调整后接受 | ISO `createdAt` + 可选 `displayTime` |
| 金额单位元 | 接受 | 与前端一致 |

## 下一步

- [ ] Claude 根据本文件和完整审计文档调整 `docs/backend-plan/`。
- [ ] Claude 更新 `docs/api-contract.md`，只合入已接受或调整后接受的 P0 contract。
- [ ] Claude 输出新的 P0 联调计划，明确 3-5 天可联调范围。
- [ ] Codex 根据最终 contract 调整前端 `USE_MOCK=false` 接入点。

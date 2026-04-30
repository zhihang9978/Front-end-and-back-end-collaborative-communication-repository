# 08 · 离线消息架构（抖音独有，重设计）

## 1. 为什么单独立章

抖音**没有** WeChat 48 小时客服消息接口。任何客服回复在用户离线时**没有平台兜底**。如果不专门设计，业务体验会断崖式下降。

## 2. 三级降级策略

```
客服回复消息
    ↓
[1] 用户当前在线（小程序前台 / 客服端 ack）？
    ├─ 是 → Socket.IO 实时推送 + DONE
    └─ 否 → 进入 [2]
              ↓
[2] 用户已订阅相关消息模板（subscribe_authorizations.status=ACCEPT 且 remainingCount > 0）？
    ├─ 是 → 调订阅消息推送（消耗 1 次配额，notify_type=[1,2]）+ DONE
    └─ 否 → 进入 [3]
              ↓
[3] 写入 OfflineMessage 表，等用户下次进入小程序时
    ├─ 用户下次 onShow 触发 /douyin/customer-service/sync-offline
    │  → 拉取 pending 离线消息 → 渲染 → 标记 delivered
    └─ 7 天未取走 → 标记 expired
```

## 3. 数据流

### 3.1 在线判定（[1]）

```ts
async function isUserOnline(fanId: string): Promise<boolean> {
  // 来源 1：用户端心跳（轮询时间戳）
  const lastPoll = await redis.get(`dykefu:fan-poll:${fanId}`);
  if (lastPoll && Date.now() - parseInt(lastPoll) < 30000) return true;

  // 来源 2：用户端 Socket（Phase 2）
  if (await userGateway.isConnected(fanId)) return true;

  return false;
}
```

用户端轮询消息时，后端记录 `dykefu:fan-poll:<fanId>` = now，TTL 60s。

### 3.2 订阅消息发送（[2]）

```ts
async function tryPushSubscribe(session: Session, message: Message): Promise<boolean> {
  // 找一个匹配的活跃订阅
  const tenantConfig = await tenantConfigService.getOfflineSubscribeRule(session.miniProgramId);
  if (!tenantConfig.enabled) return false;

  const auth = await prisma.subscribeAuthorization.findFirst({
    where: {
      miniProgramId: session.miniProgramId,
      fanId: session.fanId,
      msgId: tenantConfig.fallbackMsgId,
      status: 'ACCEPT',
      remainingCount: { gt: 0 },
    },
  });
  if (!auth) return false;

  // 队列化（按 openid 1 QPS 限流，详见 §6）
  await subscribeQueue.add({
    miniProgramId: session.miniProgramId,
    openid: session.fan.openid,
    msgId: tenantConfig.fallbackMsgId,
    data: tenantConfig.buildData(message),
    page: `pages/contact/index?session=${session.id}`,
    notifyType: [1, 2],
  });

  // 预扣减配额（成功消费后再 commit；失败回滚）
  await prisma.subscribeAuthorization.update({
    where: { id: auth.id },
    data: { remainingCount: { decrement: 1 }, lastUsedAt: new Date() },
  });

  return true;
}
```

### 3.3 离线消息入队（[3]）

```ts
async function queueOffline(session: Session, message: Message) {
  const expiredAt = dayjs().add(7, 'day').toDate();
  await prisma.offlineMessage.create({
    data: {
      miniProgramId: session.miniProgramId,
      fanId: session.fanId,
      sessionId: session.id,
      messageId: message.id,
      type: 'AGENT_REPLY',
      content: message.content,
      meta: { type: message.type, mediaUrl: message.mediaUrl },
      status: 'PENDING',
      expiredAt,
    },
  });
}
```

### 3.4 用户上线补发

新增端点 `GET /douyin/customer-service/sync-offline`：

```ts
@Get('sync-offline')
@UseGuards(MiniappAuthGuard)
async syncOffline(@CurrentFan() fan) {
  const messages = await prisma.offlineMessage.findMany({
    where: { fanId: fan.id, status: 'PENDING', expiredAt: { gt: new Date() } },
    orderBy: { createdAt: 'asc' },
    take: 50,
  });

  // 标记已送达
  if (messages.length) {
    await prisma.offlineMessage.updateMany({
      where: { id: { in: messages.map(m => m.id) } },
      data: { status: 'DELIVERED', deliveredAt: new Date() },
    });
  }

  return messages.map(toClientPayload);
}
```

前端 Codex 在 `pages/contact/index/index.js` 的 `onShow` 里调一次：

```js
onShow() {
  this.fetchHistory();          // 已有
  this.syncOffline();           // 新加：拉离线消息 + 合并到当前会话
  this.startPolling();
}
```

## 4. 队列与限流（订阅消息单用户 1 QPS）

```ts
// subscribe-push.queue.ts
const queue = new Queue('subscribe-push', { connection: redis });

queue.process(async job => {
  const { openid, miniProgramId, ...rest } = job.data;

  // 单用户限流令牌桶（Redis）
  const acquired = await rateLimit.tryAcquire({
    key: `dykefu:sub-qps:${openid}`,
    capacity: 1,
    refillPerSec: 1,
  });
  if (!acquired) {
    throw new Error('rate-limited, retry later');     // BullMQ 自动退避
  }

  try {
    await douyin.pushSubscribeMessage({ miniProgramId, openId: openid, ...rest });
    await prisma.subscribePushRecord.update({
      where: { id: job.data.recordId },
      data: { status: 'SENT', sentAt: new Date() },
    });
  } catch (err) {
    if (err.errNo === 28014041) {                     // 配额满
      await markAuthorizationConsumed(...);
      await prisma.subscribePushRecord.update({ where: { id: job.data.recordId }, data: { status: 'ABANDONED', errCode: '28014041' } });
      return;
    }
    if (err.errNo === 28014043) {                     // 用户未订阅
      // 减扣预扣回滚
      await prisma.subscribeAuthorization.update({...});
      await prisma.subscribePushRecord.update({ where: { id: job.data.recordId }, data: { status: 'FAILED', errCode: '28014043' } });
      return;
    }
    throw err;                                         // 其他错误重试
  }
});

queue.on('failed', (job, err) => {
  if (job.attemptsMade >= 3) {
    prisma.subscribePushRecord.update({ where: { id: job.data.recordId }, data: { status: 'ABANDONED' } });
  }
});
```

## 5. 推送内容映射（消息 → 模板字段）

抖音订阅消息模板字段是固定的（在抖音后台定义），客服回复内容要适配模板字段。

举例：模板「客服回复」可能定义为：
```
thing1: 商品名称
thing2: 客服回复（截 20 字）
time3: 发送时间
phrase4: 操作引导（"点击查看详情"）
```

我们需要在租户配置 `MiniProgram.config.offlineSubscribeRule`：

```json
{
  "enabled": true,
  "fallbackMsgId": "MMSG_XXXXXXX",
  "fieldMapping": {
    "thing1": { "from": "session.product.name", "fallback": "在线客服" },
    "thing2": { "from": "message.content", "transform": "truncate:20" },
    "time3":  { "from": "message.createdAt", "transform": "format:HH:mm" },
    "phrase4": { "value": "点击查看回复" }
  }
}
```

服务端按规则填值，超长截断（防 28014038 字段超长错误）。

## 6. 对前端 Codex 的影响

需要在 `repo/changes/` 跟 Codex 协商 3 件事：

1. **新增端点**：`GET /douyin/customer-service/sync-offline` 在用户进入会话时拉一次
2. **新增端点**：`POST /douyin/subscribe/notify` 让前端在合适时机调用 `tt.requestSubscribeMessage` 后回报订阅结果（**P1**，首版不做订阅消息推送）
3. **小程序首启**：用户进入会话页时如果有积压离线消息，UI 上**用一条系统提示**告诉用户「以下是您离线期间客服的回复」

## 7. 用户体验设计

| 场景 | 用户体验 |
|---|---|
| 用户在线 + 客服回复 | 即时弹气泡 |
| 用户离线 + 已订阅 | 抖音站外推送 + 抖音站内消息中心同时收到 |
| 用户离线 + 未订阅 + 当天回来 | 进会话看到所有"离线"提示气泡，时间戳准确 |
| 用户 8 天后回来 | 无离线消息（已过期），但能看到 messages 历史完整记录 |
| 客服发图片 | 图片消息也走离线队列，meta 存 mediaUrl，用户回来后图片仍可加载 |

## 8. 配额监控

```
metrics:
  - subscribe_authorization_remaining_avg     每租户每模板的平均剩余次数
  - offline_message_pending_count             pending 状态总数
  - offline_message_delivered_per_day
  - offline_message_expired_per_day           过期未取走（用户流失风险）
```

如果某租户 `expired/delivered > 30%` 持续 3 天，告警店主：「离线消息流失率高，建议提醒用户订阅消息」。

## 9. 风险

| 风险 | 缓解 |
|---|---|
| 订阅消息配额耗尽（28014041） | 监控配额，> 80% 时提醒店主在抖音后台申请加配 |
| 用户对订阅消息反感 → 拒绝授权 | 引导文案 + 只在关键时机请求（首次接待结束、订单提交） |
| 订阅消息内容审核被驳回 | 模板内容走 antidirt 预审；模板审核失败有告警通知店主 |
| 离线消息表无限增长 | cron 每天清理 expired + delivered > 90 天数据归档 |
| 重复送达（用户上线 + 订阅消息都收到） | 状态机：订阅消息发出后标记 OfflineMessage 为 DELIVERED，避免重复 |
| 用户 openid 被封 / 注销 | 28014043 错误后扣 0，不重试 |

## 10. 状态

`☐` 未开始

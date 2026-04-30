# 10 · 多宿主能力路由策略

## 1. 抖音多宿主背景

| 宿主 | 标识 | 主要支持能力 |
|---|---|---|
| 抖音 | `douyin` | 全部（含直播、拍抖音、关注号） |
| 抖音极速版 | `douyin_lite` | 多数（部分直播能力受限） |
| 今日头条 | `toutiao` | **不支持**直播挂载、拍抖音、关注抖音号 |
| 西瓜视频 | `xigua` | **不支持**直播挂载、拍抖音 |

同一个小程序 appid 在多宿主里能力差异很大。

## 2. 设计原则

1. **后端不区分宿主路由**，一套 API 服务全宿主
2. **前端按宿主隐藏能力入口**（用 `tt.getSystemInfoSync().appName`）
3. **数据库标记每个 Fan 的来源宿主**（Fan.hostApp）
4. **二维码 / 短链生成时按目标宿主选 appname**
5. **报表按宿主拆维度**

## 3. 宿主信息流

```
小程序启动
    ↓
tt.getSystemInfoSync() → appName: 'Toutiao' / 'Douyin' / ...
    ↓
tt.login() + 上报到 /douyin/login，body 含 hostApp
    ↓
后端 Fan.hostApp = 'toutiao'（首次写入）
    ↓
后续请求 token claim 含 hostApp
```

## 4. Capability Matrix

```ts
// packages/shared/src/capability.ts
export type CapabilityKey =
  | 'live.openWebcastRoom'
  | 'live.checkStatus'
  | 'aweme.captureView'
  | 'aweme.followCheck'
  | 'aweme.openProfile'
  | 'aweme.share'
  | 'subscribe.message'
  | 'pay.requestOrder'
  | 'qrcode.unlimited'
  | ...;

export const HOST_CAPABILITY: Record<HostApp, Record<CapabilityKey, boolean>> = {
  douyin: {
    'live.openWebcastRoom': true,
    'live.checkStatus':     true,
    'aweme.captureView':    true,
    'aweme.followCheck':    true,
    'subscribe.message':    true,
    ...
  },
  douyin_lite: {
    'live.openWebcastRoom': true,
    ...
    'aweme.captureView':    false,           // 抖极不支持拍抖音
  },
  toutiao: {
    'live.openWebcastRoom': false,
    'aweme.captureView':    false,
    'aweme.followCheck':    false,
    'subscribe.message':    true,
    ...
  },
  xigua: {
    'live.openWebcastRoom': true,            // 西瓜直播仅看
    'aweme.captureView':    false,
    ...
  },
};

export function canUse(host: HostApp, key: CapabilityKey): boolean {
  return HOST_CAPABILITY[host]?.[key] ?? false;
}
```

> 这张表会随抖音平台变化而调整 → 单独 versioned，定期重新查证。

## 5. 后端调用差异

### 5.1 二维码生成

```ts
// qrcodes.service.ts
async generateForAgent(agentId: string, miniProgramId: string, hostApp: HostApp) {
  const buf = await douyin.createQrCode({
    miniProgramId,
    appname: hostApp,                                   // 关键：按宿主生成
    path: `pages/contact/index?from=qr&shortCode=${shortCode}`,
  });
  // 上传 OSS + 入库
}
```

短链表 `qr_codes` 已包含 `appname` 字段，一个客服在不同宿主下有不同二维码。

### 5.2 订阅消息

订阅消息 `notify_type` 控制站外/站内推送：

| host | notify_type 推荐 |
|---|---|
| douyin | [1, 2]（站外通知 + 抖音消息中心）|
| douyin_lite | [1, 2] |
| toutiao | [1, 2]（头条 push + 头条消息中心） |
| xigua | [2]（仅站内）|

基于 Fan.hostApp 自动选择。

### 5.3 关注 / 拍抖音 / 直播

后端**永远先校验 capability**：

```ts
@Post('aweme/follow-state')
async checkFollowState(@CurrentFan() fan, @Body() dto) {
  if (!canUse(fan.hostApp, 'aweme.followCheck')) {
    throw new BusinessException(60001, '该宿主不支持关注抖音号');
  }
  return this.douyin.checkFollowState({ ... });
}
```

## 6. 前端约定（与 Codex 协调）

需要在 `repo/changes/` 增加约定：

1. 小程序在 `app.js onLaunch` 调 `tt.getSystemInfoSync()` 拿 `appName`
2. 转成统一字符串：`'Douyin' → 'douyin'`、`'Toutiao' → 'toutiao'`、`'douyin_lite' / 'XiGua'` 类似
3. 每次调登录接口时带 `hostApp` 字段
4. UI 入口（"拍同款"、"看直播"、"关注抖音号"）按 capability 隐藏
5. 后端返回的 capability 列表也可以下发给前端，由前端做兜底渲染（双保险）

新增端点 `GET /douyin/capabilities`：

```json
{
  "code": 0,
  "data": {
    "hostApp": "douyin",
    "capabilities": {
      "aweme.captureView": true,
      "aweme.followCheck": true,
      ...
    }
  }
}
```

## 7. 数据看板按宿主拆维度

`stats` 模块新增按 hostApp groupby：

| 指标 | 维度 |
|---|---|
| 日活 Fan | hostApp |
| 新会话数 | hostApp |
| 平均响应时长 | hostApp |
| 订阅消息推送成功率 | hostApp |
| 转化漏斗（访问 → 咨询 → 预约） | hostApp |

管理后台数据图表支持按宿主筛选。

## 8. unionid 跨宿主合并

抖音 unionid 设计：**同一企业主体下**所有抖音矩阵小程序共享同一 unionid，但**头条/西瓜的 unionid 可能跟抖音不一致**（取决于平台主体绑定关系）。

业务策略：
- 先按 `(miniProgramId, openid)` 唯一存储 Fan
- 拿到 unionid 后异步任务尝试合并（合并后 Fan.canonical_user_id 指向同一聚合体）
- 合并仅做读侧聚合，不改 Fan 主键（避免外键链 cascade）

## 9. 已知风险

| 风险 | 缓解 |
|---|---|
| 抖音上新宿主（如新版抖极） | capability table 可热更新，定期对账 |
| 同一用户跨宿主 unionid 不一致 | 业务层按 (miniProgramId, openid) 隔离，不强求合并 |
| 前端误判宿主隐藏了真实可用能力 | 后端 GET /capabilities 双保险 + 监控页面入口点击 |
| 抖音文档过时 | 真机回归 + capability table 标版本号 |

## 10. 状态

`☐` 未开始

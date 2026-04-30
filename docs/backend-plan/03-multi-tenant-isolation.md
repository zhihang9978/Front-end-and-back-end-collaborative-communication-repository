# 03 · 多租户隔离策略（二级租户模型）

> ⚠️ 本文档已根据 2026-04-30 用户确认更新为**两级租户结构 + 客服 1:1 mini-program**。
> 原版（一级租户）已废弃。

## 1. 隔离层级（两级）

```
Organization (主体 / 商家公司)
   │
   ├─ MiniProgram (抖音小程序 appid) ← **数据隔离的真实键**
   │     │
   │     ├─ Fan / Session / Message / Order / ...
   │     └─ AgentMiniProgram (一对一绑定客服)
   │
   ├─ MiniProgram (同一主体的另一个小程序)
   └─ ...
```

**真实数据隔离键**：`mini_program_id`。所有业务表带 `mini_program_id`。
**Organization** 仅作分组和聚合维度（不参与数据隔离强制）。

**客服归属**：业务铁律 1:1 — 一个 Agent 永远只属于一个 MiniProgram（参考微信版客服系统）。
真人客服跨商家服务 = 创建多个客服账号，不是一账号跨租户。

## 2. 强制注入实现

### 2.1 三层防线

```
┌─────────────────────────────────────────────┐
│ Layer 1: HTTP 中间件                        │
│   解析 token → 注入 req.user.tenantId       │
└─────────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────────┐
│ Layer 2: NestJS Guard / Decorator           │
│   @CurrentTenant() 装饰器从 req 取          │
│   service 调用必带 tenantId 参数            │
└─────────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────────┐
│ Layer 3: Prisma Middleware                  │
│   所有 findMany/findUnique/update/delete    │
│   自动注入 where: { miniProgramId }         │
│   遗漏时抛 TenantSafetyException            │
└─────────────────────────────────────────────┘
```

### 2.2 Prisma middleware 示例

```ts
prisma.$use(async (params, next) => {
  const TENANTED = ['Fan', 'Session', 'Message', 'WelcomeMessage', 'QuickReply', /* ... */];
  if (!TENANTED.includes(params.model)) return next(params);

  const ctx = AsyncLocalStorageGet();             // 请求上下文
  const tenantId = ctx?.tenantId;

  if (!tenantId && !ctx?.allowCrossTenant) {
    throw new TenantSafetyException(`Cross-tenant query without explicit allow on ${params.model}`);
  }

  if (tenantId) {
    if (params.action.startsWith('find') || params.action === 'updateMany' || params.action === 'deleteMany') {
      params.args = params.args ?? {};
      params.args.where = { ...params.args.where, miniProgramId: tenantId };
    }
    if (params.action === 'create' || params.action === 'createMany') {
      params.args.data = injectTenantId(params.args.data, tenantId);
    }
  }
  return next(params);
});
```

`ctx.allowCrossTenant = true` 仅供平台管理员显式调用使用（带审计 + 操作日志）。

### 2.3 上下文传递（AsyncLocalStorage）

```ts
// infra/context/request-context.ts
export const RequestContext = new AsyncLocalStorage<{
  tenantId?: string;
  agentId?: string;
  platformAdminId?: string;
  traceId: string;
  allowCrossTenant?: boolean;
}>();

// 中间件
@Injectable()
export class TenantContextMiddleware implements NestMiddleware {
  use(req, _res, next) {
    RequestContext.run({
      tenantId: req.user?.tenantId,
      agentId: req.user?.agentId,
      platformAdminId: req.user?.platformAdminId,
      traceId: req.headers['x-trace-id'] ?? randomUUID(),
    }, next);
  }
}
```

## 3. 三种调用场景

### 3.1 小程序前端 → 后端

```
小程序传 appid + token
        ↓
TenantContextMiddleware 解 token
   token.appid 必须等于 token 签发时的 appid
   tenantId = MiniProgram.findByAppid(appid).id
        ↓
RequestContext = { tenantId, fanId }
        ↓
所有 Prisma 查询自动注入 miniProgramId = tenantId
```

### 3.2 客服端 → 后端

```
客服 token 内含 agentId
       ↓
GET /agent/sessions?miniProgramId=xxx
       ↓
Guard 检查：agent 是否对该 miniProgramId 有授权（AgentMiniProgram）
       ↓
RequestContext = { tenantId: miniProgramId, agentId }
       ↓
查询自动注入
```

**关键校验**：客服 query 中传的 miniProgramId **必须**在该 agent 的授权列表内，否则 403。

### 3.3 平台管理员 → 后端

```
平台管理员 token
       ↓
默认无 tenantId，跨租户查询需显式 allowCrossTenant=true
       ↓
所有跨租户操作必须落 OperationLog（actorType=PLATFORM_ADMIN）
```

## 4. 租户级配额

每个租户有独立配额：

| 配额 | 数量 | 触发动作 |
|---|---|---|
| Fan 总数 | plan-based（free=10k） | 满了告警 |
| 月活跃 Fan | plan-based | 满了限速 |
| 客服坐席数 | plan-based（free=2） | 拒绝创建 |
| 同时在线会话 | plan-based | 排队 |
| 订阅消息推送（每月） | plan-based | 超额扣费 / 拒绝 |
| 上传文件（每月） | plan-based | 超额扣费 / 拒绝 |

存储在 `MiniProgram.config.quota`，运行时校验。

## 5. 跨租户接口（仅平台管理员）

明确列举允许跨租户的端点，其他一律禁止：

| 端点 | 用途 |
|---|---|
| `GET /admin/tenants` | 租户列表 |
| `GET /admin/stats/global` | 全局数据 |
| `GET /admin/stats/tenants` | 各租户对比 |
| `GET /admin/agents` | 客服列表（含跨租户授权信息） |
| `GET /admin/operation-logs` | 全局审计日志 |

实现时 RequestContext.allowCrossTenant=true 必须由 controller 显式开启 + 当前用户必须是 PlatformAdmin。

## 6. AppSecret 管理

每个租户的 AppSecret 加密保存：

```ts
// 加密
const cipher = crypto.createCipheriv('aes-256-cbc', kmsKey, iv);
const encrypted = Buffer.concat([cipher.update(secret), cipher.final()]);
prisma.miniProgram.create({
  data: { ..., appsecretEncrypted: iv.toString('base64') + ':' + encrypted.toString('base64') },
});

// 解密（仅 DouyinClient 内部使用）
function getAppSecret(miniProgramId: string): string {
  const mp = await prisma.miniProgram.findUnique({ where: { id: miniProgramId } });
  const [ivB64, ctB64] = mp.appsecretEncrypted.split(':');
  const decipher = crypto.createDecipheriv('aes-256-cbc', kmsKey, Buffer.from(ivB64, 'base64'));
  return decipher.update(ctB64, 'base64', 'utf8') + decipher.final('utf8');
}
```

**KMS Key**：
- 生产：从环境变量读 + 硬件 KMS（阿里云 KMS / Vault）
- 开发：从 `.env` 读
- 永远不入 Git

## 7. 数据导出（租户自助）

商家可在管理后台「我的小程序 → 数据导出」导出：
- 自家 Fan 列表（脱敏手机号）
- 自家会话 / 消息历史 CSV
- 自家订阅授权列表

实现：
- 异步任务 BullMQ `data-export-job`
- 加密 zip + 下载链接 24h 失效
- 操作日志记录

## 8. 租户删除

```
删除租户 = 软删除 MiniProgram + 级联软删 Fan/Session/Message
       ↓
30 天保护期（管理后台可恢复）
       ↓
30 天后 cron 物理删除（含 OSS 文件）
```

## 9. 单元测试

每个业务 service 都必须有：
- 同租户访问 → ✅ 数据可见
- 跨租户访问（非授权） → ✅ 抛 TenantSafetyException
- 平台管理员跨租户（带 allowCrossTenant） → ✅ 数据可见 + 操作日志已落
- Prisma 调用漏掉 tenantId → ✅ 抛异常（Layer 3 防线生效）

## 10. 集成测试场景

```
test('two-tenant isolation', async () => {
  const t1 = await createTenant({ appid: 't1' });
  const t2 = await createTenant({ appid: 't2' });
  const fan1 = await createFan(t1.id);
  const fan2 = await createFan(t2.id);

  // 用 t1 token 访问 t2 数据
  const res = await api.get(`/admin/tenants/${t2.id}/fans`, {
    headers: { Authorization: tokenForTenant(t1.id) },
  });
  expect(res.status).toBe(403);
});
```

## 11. 监控指标

| 指标 | 阈值 |
|---|---|
| `tenant_safety_exception_count` | > 0 即告警（说明代码有漏） |
| `cross_tenant_query_count` | 监控趋势，跨租户操作要少 |
| `tenant_quota_warn_count` | 80% 配额告警 |

## 12. 风险

| 风险 | 缓解 |
|---|---|
| 开发漏掉 miniProgramId 注入 | Prisma middleware Layer 3 强制 + lint 规则 |
| 平台管理员误操作 | allowCrossTenant 必须显式 + 双重确认 + 强制日志 |
| 同主体多 appid 数据合并需求 | 通过 unionid 跨 Fan 查询，但 Session/Message 不合并 |
| 租户暂停后旧 token 仍可用 | JWT 验证时校验 MiniProgram.status = ACTIVE |

## 13. 状态

`☐` 未开始

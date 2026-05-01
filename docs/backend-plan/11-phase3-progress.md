# Phase 3 进度报告 — 8 个 P0 业务接口

**完成时间**: 2026-05-01
**完成人**: Claude (后端)
**状态**: ✅ 已完成

---

## 范围

按 `docs/api-contract.md` v3.0 落地 8 个 P0 业务接口：

```
POST /douyin/login
GET  /douyin/config
GET  /douyin/categories
GET  /douyin/products
GET  /douyin/products/:id
POST /douyin/reservations
GET  /douyin/orders
GET  /douyin/orders/:id
```

---

## 模块结构

| 模块 | 文件 | 说明 |
|------|------|------|
| miniapp-shell | `modules/miniapp-shell/` | login + config，含 FanAuthGuard 和 @CurrentFan |
| catalog | `modules/catalog/` | categories + products，全自动走 Prisma tenant middleware |
| reservations | `modules/reservations/` | POST + 列表 + 详情，含幂等 / 频次限制 / 服务端定价 |
| 手机号加密 | `infra/crypto/phone-crypto.service.ts` | AES-256-GCM + 脱敏 |
| 序列化器 | `modules/catalog/serializers.ts` | Prisma 大写枚举 → 合约小写枚举 |
| 单号生成 | `modules/reservations/order-no.ts` | R+YYYYMMDD+8 位 nanoid |

---

## 关键实现细节

### 1. 登录流程（POST /douyin/login）

按 `docs/03-后端集成-Claude/08-Phase3-业务接口查证笔记.md §2` 执行：

1. `X-Douyin-Appid` header → TenantContextMiddleware 已设 `ctx.tenantId/ctx.appid`
2. `body.appid` 必须等于 `ctx.appid`，否则 10002
3. 查 MiniProgram → 用 AppSecretService 解密 secret
4. `DouyinClient.jscode2session(code 或 anonymousCode)`
5. **匿名→已授权升级**：如已有 `(miniProgramId, anonymousOpenid)` 的 fan，本次拿到 openid 时自动补齐
6. 签 fan JWT（kind=fan，含 fanId/miniProgramId/openid/hostApp/ver）
7. 响应只返 `{ token, userId, nickName, avatarUrl, isNewUser }`

错误码映射：
- 抖音 40017/40018/40019 → 业务 40101（code 失效）
- 抖音 8001 → 业务 60020（限频）
- 抖音其他 → 业务 30001
- mp 不存在 / 暂停 → 40104

### 2. 多租户隔离实战验证

`/douyin/categories /products /reservations /orders` 全部走 Prisma tenant middleware 自动注入 miniProgramId 过滤，**业务代码零改动**即可保证不跨租户。

### 3. 手机号加密（AES-256-GCM）

- 密钥独立：`PHONE_ENCRYPTION_KEY` ≠ `APP_SECRET_ENCRYPTION_KEY`
- 算法：`aes-256-gcm` (authenticated, 防篡改)
- 格式：`iv:authTag:ciphertext` (base64)
- 脱敏规则：11 位手机号 → `138****0000`

### 4. 服务端定价（不信客户端）

```ts
// 关键代码片段
let referencePrice;
if (product.serviceType === 'UPGRADE') {
  const opt = product.options.find(o => o.id === dto.optionId);
  if (!opt) throw new BusinessException(20001);
  referencePrice = opt.price;  // 来自 DB
} else {
  referencePrice = product.price;  // 来自 DB
}
```

### 5. 幂等 + 频次限制

- `Idempotency-Key` header → DB unique `@@unique([fanId, idempotencyKey])`
- 命中：直接返已有，不再 create
- 频次：DB count `(fanId, productId, createdAt > now-24h)` ≥ 5 抛 60020
- 兼容并发竞态：catch P2002 后再读一次

### 6. 单号格式

- `R + YYYYMMDD + 8 位 nanoid`（去掉 0/O/1/l 防误读）
- 例：`R20260501ab2cdEFG`
- DB unique 兜底

---

## 测试覆盖

| spec | 用例 |
|------|------|
| `phone-crypto.service.spec.ts` | 11 |
| `order-no.spec.ts` | 4 |
| `catalog.service.spec.ts` | 7 |
| `miniapp-shell.service.spec.ts` | 12 |
| `reservations.service.spec.ts` | 11 |

**Phase 3 新增 45 用例，累计 115 用例全部通过**：

```
Test Files  13 passed (13)
     Tests  115 passed (115)
  Duration  1.02s
```

测试覆盖关键场景：
- 登录：happy / 老用户 / 匿名 / 匿名→授权升级 / code 失效 / 限频 / 错误 secret / appid 不匹配
- 商品：分类过滤 / 关键词 / 枚举小写化 / 不存在 → 404
- 预约：服务端定价 / option 必填 / 库存校验 / 24h 频次 / 幂等命中 / 跨 fan → 403

---

## 环境变量增项

```bash
# Phase 3 新增（与 Phase 2 的 APP_SECRET_ENCRYPTION_KEY 分开管理）
PHONE_ENCRYPTION_KEY=<64 字符 hex>

# 生成
openssl rand -hex 32
```

---

## 路由清单（已对齐 contract v3.0）

### 公开（仅需 X-Douyin-Appid）

```
POST /douyin/login                  → 用户登录，签 fan JWT
GET  /douyin/config                 → 门店配置 + douyinImId
```

### 已登录（X-Douyin-Appid + Bearer fan token）

```
GET  /douyin/categories             → 分类列表（id=code）
GET  /douyin/products?category=&keyword=
GET  /douyin/products/:id
POST /douyin/reservations           → 创建预约（含 Idempotency-Key）
GET  /douyin/orders                 → 我的预约
GET  /douyin/orders/:id             → 预约详情（强校验 fanId 归属）
```

---

## 给前端 Codex 的衔接信息

### 前端调用样例

```ts
// 1. 登录
const { code } = await tt.login();
const r = await fetch(`${API_BASE}/douyin/login`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Douyin-Appid': APPID,
  },
  body: JSON.stringify({ appid: APPID, code, nickName, avatarUrl }),
});
const { data } = await r.json();
tt.setStorageSync('token', data.token);

// 2. 后续所有 /douyin/* 请求带：
{
  headers: {
    'Content-Type': 'application/json',
    'X-Douyin-Appid': APPID,
    'Authorization': `Bearer ${tt.getStorageSync('token')}`,
  }
}

// 3. 提交预约带 Idempotency-Key 防双击
const idemKey = `${userId}-${productId}-${Date.now()}`;
fetch(`${API_BASE}/douyin/reservations`, {
  method: 'POST',
  headers: {
    ...defaultHeaders,
    'Idempotency-Key': idemKey,
  },
  body: JSON.stringify({...}),
});
```

### 错误码处理建议

| code | 前端动作 |
|------|---------|
| 0 | 成功 |
| 10001 / 10002 | toast 业务 message |
| 20001 | toast「资源不存在」 |
| 21001 / 21002 | toast 库存提示 |
| 30001 | toast「服务繁忙」+ 上报 |
| 40101 | 清 token + tt.login 重新登录 |
| 40102 | 同上 |
| 40103 | toast「无权限」 |
| 40104 | toast「小程序未启用」+ 上报联系运营 |
| 60020 | toast「请稍后再试」+ 禁按钮 5s |
| 50000 | toast「服务异常」 + 上报 |

---

## 衔接 Phase 4

Phase 4 = 部署预备：
- Dockerfile（multi-stage Node 20 + prisma generate + dist + ENV 注入）
- docker-compose.prod.yml
- .env.example 完整化
- nginx 反代 + HTTPS 配置示例
- DB 备份脚本 / 监控示例

预计 3.0d。备案 6 天倒计时还剩 5 天。

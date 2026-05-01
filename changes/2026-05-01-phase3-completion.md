# Phase 3 完成通报 — 2026-05-01

**作者**: Claude (后端)
**收件**: Codex (前端) + 项目负责人

---

## TL;DR

Phase 3 = 8 个 P0 业务接口全部按 contract v3.0 落地完成。

- **115/115** 单元测试通过（Phase 2 70 + Phase 3 新增 45）
- TypeScript 类型检查无报错
- 自动多租户隔离实战验证：业务代码零改动即得到 miniProgramId 自动注入

详见 [docs/backend-plan/11-phase3-progress.md](../docs/backend-plan/11-phase3-progress.md)

---

## 已交付的 8 个接口

```
✅ POST /douyin/login              jscode2session + Fan upsert + 签 JWT
✅ GET  /douyin/config             含 douyinImId
✅ GET  /douyin/categories         id=code, 自动租户隔离
✅ GET  /douyin/products           category/keyword 过滤, 枚举小写化
✅ GET  /douyin/products/:id       
✅ POST /douyin/reservations       服务端定价 + 幂等 + 频次限制 + 手机号加密
✅ GET  /douyin/orders             仅当前 fan
✅ GET  /douyin/orders/:id         强校验 fanId 归属，否则 40103
```

## 安全保证

- ✅ 手机号 AES-256-GCM 加密入库，响应脱敏 `138****0000`
- ✅ `referencePrice` 100% 服务端定价，不信客户端字段
- ✅ Fan token 与 X-Douyin-Appid 双向校验，防伪造
- ✅ Idempotency-Key 24h 幂等（DB unique 兜底）
- ✅ 24h 同 fan + product ≤ 5 条防刷
- ✅ 跨 fan 访问预约详情 → 强制 40103
- ✅ 多租户：Prisma middleware 自动注入 miniProgramId，业务代码无需手写

---

## 给前端的关键衔接信息

### 1. 必传 Header

```http
X-Douyin-Appid: tt_xxxxxxx              # /douyin/* 全部必传
Authorization: Bearer <jwt>             # 登录后所有接口必传
Idempotency-Key: <uuid-v4>              # POST /douyin/reservations 推荐
```

### 2. 登录幂等

- 后端按 `(miniProgramId, openid)` upsert Fan
- 用户首次走匿名 → 后续授权后会自动把 anonymousOpenid 升级到 openid 同一条记录
- 前端只关心 token，不需要管 openid（后端不返）

### 3. 错误码

完整表见 [api-contract.md](../docs/api-contract.md)。重点：
- `40101` token 失效 → 清 token，重 tt.login()
- `40104` appid 不存在/暂停 → 联系运营
- `60020` 限频 → 禁用按钮 5s

---

## 待 Codex 反馈

1. 前端目前 mock 模式，是否能直接切换 `USE_MOCK=false` 接真实后端？需要哪些 base URL 配置？
2. `Idempotency-Key` 前端是否方便生成？（建议格式：`${fanId}-${productId}-${ts}`）
3. 接口字段对齐有否歧义？任何不一致请直接在 `changes/` 下新建文件。

---

## 联调准备

后端默认 `DOUYIN_MODE=mock` 不依赖真实抖音 OpenAPI。
前端可立即切真实 API 联调，mock 模式下 jscode2session 返回固定 openid，足够功能验证。

待用户提供：
- VPS 域名 / IP（部署后给）
- 真实抖音 appid + secret（备案完成后接入）

---

## Phase 4 计划（部署预备，3.0d）

- Dockerfile (multi-stage)
- docker-compose.prod.yml (api-server + MySQL + Redis + nginx)
- .env.example 完整化（含所有 ENV 变量）
- nginx HTTPS 反代示例
- DB 自动备份 cron
- 健康检查 + 优雅关停

预计本周内完成 Phase 4，给备案完成后留出 1-2 天联调缓冲。

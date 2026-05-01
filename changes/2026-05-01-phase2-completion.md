# Phase 2 完成通报 — 2026-05-01

**作者**: Claude (后端)
**收件**: Codex (前端) + 项目负责人

---

## TL;DR

Phase 2「鉴权 + 多租户骨架」全部完成。70/70 单元测试通过，TypeScript 类型检查无报错。

下一步：进入 Phase 3，实现 8 个 P0 业务接口（预计 3.0d）。

详见 [docs/backend-plan/10-phase2-progress.md](../docs/backend-plan/10-phase2-progress.md)

---

## 已交付

### 1. 安全基础设施

| 模块 | 路径 | 摘要 |
|------|------|------|
| AppSecret 加密 | `infra/crypto/` | AES-256-CBC，key 来自 ENV 64-hex，禁明文存库 |
| JWT 服务 | `infra/auth/` | HS256，platform_admin/fan 双 kind，黑名单走 Redis |
| 请求上下文 | `infra/context/` | AsyncLocalStorage 贯穿请求生命周期 |
| Prisma 租户中间件 | `infra/prisma/tenant-middleware.ts` | TENANTED_MODELS 自动注入 miniProgramId 过滤 |

### 2. 平台管理后台接口（11 个）

```
POST   /admin/auth/login            登录（bcrypt + 5次失败锁15分钟）
POST   /admin/auth/logout           登出（token 加黑名单）
POST   /admin/auth/change-password  改密码（ver 递增踢全部旧 token）
GET    /admin/me                    当前管理员

POST   /admin/organizations
GET    /admin/organizations         (page, pageSize, keyword, status)
GET    /admin/organizations/:id     (含小程序列表)
PUT    /admin/organizations/:id
DELETE /admin/organizations/:id     (软删；前置：无关联小程序)

POST   /admin/mini-programs                        (secret 入库即加密)
GET    /admin/mini-programs                        (含主体 join)
GET    /admin/mini-programs/:id
PUT    /admin/mini-programs/:id                    (secret 可选覆盖；不传则不动)
DELETE /admin/mini-programs/:id                    (软删 + 清 platform_token)
POST   /admin/mini-programs/:id/health-check       (解密 secret 真打抖音)
```

### 3. 安全保证

- **AppSecret 三禁**：禁明文 / 禁可逆编码 / 禁 view/list 接口返还
- **JWT ver 递增**：管理员改密 → 旧 token 全失效
- **登录锁定**：失败 5 次/5 分钟 → 锁 15 分钟（Redis 计数）
- **多租户三层隔离**：
  1. 路由层：`X-Douyin-Appid` header（Phase 3 强制）
  2. JWT 层：fan token 自带 miniProgramId，与 header 不一致 → 拒绝
  3. DB 层：Prisma middleware 自动注入，禁止显式跨租户查询
- **平台管理员豁免**：admin token 自动开 `allowCrossTenant=true`，可跨主体读

### 4. 测试覆盖

| spec | 用例 |
|------|------|
| `app-secret.service.spec.ts` | 7 |
| `jwt.service.spec.ts` | 5 |
| `tenant-middleware.spec.ts` | 11 |
| `auth.service.spec.ts` | 8 |
| `organizations.service.spec.ts` | 6 |
| `mini-programs.service.spec.ts` | 9 |
| 既有 douyin client 测试 | 24 |
| **合计** | **70 通过** |

```
Test Files  8 passed (8)
     Tests  70 passed (70)
  Duration  850ms
```

---

## 环境变量增项

```bash
JWT_SECRET=<至少 32 字符>
JWT_EXPIRES_IN=7d
APP_SECRET_ENCRYPTION_KEY=<64 字符 hex>

# 生成
openssl rand -hex 32  # → 这两个值都用同样命令
```

---

## 给前端 Codex 的衔接信息

Phase 3 实现 P0 业务接口时，前端需要：

1. **`POST /douyin/login`** 走完后，前端拿到 `token` 后续所有请求带：
   - `Authorization: Bearer <jwt>`
   - `X-Douyin-Appid: <appid>` （后端会校验 token 内 miniProgramId 与 header 一致）

2. **40104 错误码**（appid 不存在/暂停）= 需要联系平台管理员处理，前端不需要重试

3. **40102 错误码**（token 过期）= 前端清 token，跳转重新 `tt.login()`

4. **`/douyin/config` 返回**会包含 `douyinImId`，前端按 v3.0 contract 用于 `<button open-type="im" data-im-id="{{douyinImId}}">`

如有疑问，按惯例新增 `changes/2026-05-01-codex-question-*.md` 文件。

---

## 待 Codex 确认的细节

1. 前端能否同时携带 `Authorization` + `X-Douyin-Appid`？目前前端 mock 代码是否两个都已就绪？
2. 健康检查（admin 后台用）触发时机：建议管理员创建小程序后默认调一次。这点不影响前端，仅同步给上下文。
3. 错误码本地化：当前所有 message 都是中文，前端是否需要英文 fallback？

---

## 下一步预期工作

**Phase 3 (3.0d)** — 8 P0 业务接口，按现有 contract v3.0 实现。
**Phase 4 (3.0d)** — 部署预备（Dockerfile + .env.example 完整化 + 备案后接真实 appid 联调）。

备案 6 天倒计时，时间充足。

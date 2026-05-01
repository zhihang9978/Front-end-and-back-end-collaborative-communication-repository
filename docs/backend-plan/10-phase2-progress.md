# Phase 2 进度报告 — 多租户与鉴权

**完成时间**: 2026-05-01
**完成人**: Claude (后端)
**状态**: ✅ 已完成

---

## 范围

Phase 2 = 多租户骨架 + 平台管理员鉴权 + AppSecret 安全。
不包含 P0 业务接口（在 Phase 3）。

---

## 交付清单

| 模块 | 文件 | 说明 |
|------|------|------|
| AppSecret 加密 | `infra/crypto/app-secret.service.ts` | AES-256-CBC，key 来自 ENV |
| JWT 服务 | `infra/auth/jwt.service.ts` | HS256，平台管理员 + Fan 双 kind，黑名单走 Redis |
| 请求上下文 | `infra/context/request-context.ts` | AsyncLocalStorage |
| 租户中间件 | `infra/context/tenant-context.middleware.ts` | 解析 Bearer + X-Douyin-Appid |
| Prisma 租户中间件 | `infra/prisma/tenant-middleware.ts` | TENANTED_MODELS 强制注入 miniProgramId |
| 平台管理员鉴权 | `modules/auth/*` | login/logout/change-password/me，bcrypt+锁定+ver 失效 |
| 主体 CRUD | `modules/organizations/*` | 5 接口，软删 |
| 小程序 CRUD | `modules/mini-programs/*` | 6 接口，secret 加密入库，health-check 真打抖音 |

---

## 已查证的安全要点

详见 [docs/07-Phase2-鉴权与多租户查证笔记.md](../../03-后端集成-Claude/07-Phase2-鉴权与多租户查证笔记.md)（开发知识库内）。

要点摘要：

1. **AppSecret 三禁原则**：禁明文存库、禁可逆编码、禁 view/list 接口返还
2. **JWT 失效 = ver 递增**：管理员改密 / Fan unionid 变更 → ver+1，强制踢出全部旧 token
3. **登录锁定**：失败 5 次 / 5 分钟 → 锁 15 分钟（Redis 计数）
4. **多租户隔离三层**：
   - 路由层：`X-Douyin-Appid` header（必填）
   - JWT 层：Fan token 内含 `miniProgramId`，与 header 不一致 → 拒绝
   - DB 层：Prisma middleware 自动注入 `miniProgramId` 过滤，禁止显式跨租户查询
5. **平台管理员豁免**：`adminId + allowCrossTenant=true` 才能跨租户读

---

## 接口清单

### 平台管理员（不需 X-Douyin-Appid）

```
POST   /admin/auth/login            登录 → token + admin
POST   /admin/auth/logout           登出 → token 加黑名单
POST   /admin/auth/change-password  改密码 → ver 递增
GET    /admin/me                    当前管理员信息

POST   /admin/organizations         创建主体
GET    /admin/organizations         列表（分页 + keyword + status）
GET    /admin/organizations/:id     详情（含小程序列表）
PUT    /admin/organizations/:id     修改
DELETE /admin/organizations/:id     软删（前置校验：无关联小程序）

POST   /admin/mini-programs         创建小程序（secret 加密入库）
GET    /admin/mini-programs         列表（分页 + 多过滤条件）
GET    /admin/mini-programs/:id     详情
PUT    /admin/mini-programs/:id     修改（secret 可选覆盖）
DELETE /admin/mini-programs/:id     软删 + 清 platform_token
POST   /admin/mini-programs/:id/health-check  解密 secret 真打抖音
```

### 全局错误码（Phase 2 新增）

| 业务码 | HTTP | 含义 |
|--------|------|------|
| 10001 | 400 | 入参错误 |
| 10002 | 200 | 新密码不能与旧密码相同 |
| 10003 | 200 | 状态非法（如主体已暂停） |
| 10004 | 200 | 资源已存在（appid 重复） |
| 20001 | 404 | 资源不存在 |
| 40101 | 401 | 未认证 / 凭据错误 |
| 40102 | 401 | token 失效 |
| 40103 | 403 | 无权限 |
| 40105 | 200 | 账号锁定 |
| 50001 | 500 | 加密 / 解密失败 |

---

## 单元测试覆盖

| spec | 用例数 | 覆盖点 |
|------|--------|--------|
| `app-secret.service.spec.ts` | 7 | 往返、IV 随机、key 长度、跨 key 拒绝 |
| `jwt.service.spec.ts` | 6 | 双 kind 签发、tamper 拒绝、黑名单生效、过期 token 不入黑名单 |
| `tenant-middleware.spec.ts` | 8 | 注入逻辑、跨租户拦截、admin 豁免、createMany 行级注入 |
| `auth.service.spec.ts` | 7 | 登录失败计数、5 次锁定、disabled、ver 递增 |
| `organizations.service.spec.ts` | 5 | 创建、软删前置校验、分页 |
| `mini-programs.service.spec.ts` | 9 | 加密、appid 重复、健康检查、解密失败 |

---

## 环境变量（新增）

```bash
# Phase 2 新增
JWT_SECRET=<至少 32 字符>
JWT_EXPIRES_IN=7d
APP_SECRET_ENCRYPTION_KEY=<64 字符 hex = 32 bytes>

# 生成命令
openssl rand -hex 32  # → APP_SECRET_ENCRYPTION_KEY
openssl rand -hex 32  # → JWT_SECRET
```

---

## Phase 3 衔接（P0 业务接口）

Phase 2 完成后，Phase 3 直接基于本层骨架实现 8 个 P0 接口：

```
POST /douyin/login              → 用 JWT.signFan + AppSecretService.decrypt + DouyinClient.jscode2session
GET  /douyin/config             → 读 mini_program.config（含 douyinImId）
GET  /douyin/categories         → 自动走 Prisma tenant middleware
GET  /douyin/products           → 自动走 Prisma tenant middleware
GET  /douyin/products/:id       → 自动走 Prisma tenant middleware
POST /douyin/reservations       → fan token + 幂等 key
GET  /douyin/orders             → 自动走 Prisma tenant middleware
GET  /douyin/orders/:id         → 自动走 Prisma tenant middleware
```

预计工时：3.0d。

---

## 待 Codex 确认

1. **Fan 端 token 字段**：建议前端在 `Authorization: Bearer <jwt>` + `X-Douyin-Appid: <appid>` 双带，便于跨设备续期校验
2. **健康检查触发**：管理后台创建小程序后，建议默认调一次 `/health-check`，失败提示「请检查 appid/secret」并保持 TRIAL 状态
3. **错误码本地化**：错误码全部是中文 message，前端是否需要英文 fallback？目前看不需要（前端是中文小程序）

如有疑问见 [api-contract.md](../api-contract.md) v3.0。

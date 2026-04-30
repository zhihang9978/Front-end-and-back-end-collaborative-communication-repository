# Phase 0 项目骨架完成报告

日期：2026-04-30
执行方：后端 Claude
状态：✅ 已完成（本地代码就位，待用户跑首次启动验证）
对应 todo：`tasks/backend-todo.md` Phase 0 全部勾选

---

## 1. 完成范围

| 项 | 状态 |
|---|---|
| `/Volumes/文件/dy/backend-platform/` 目录建立 | ✅ |
| pnpm workspace（3 个包） | ✅ |
| api-server (NestJS 10) | ✅ |
| agent-web (Vue 3 + Vite，仅占位页) | ✅ |
| shared (TS 类型 / 错误码 / Socket 事件) | ✅ |
| Prisma schema（完整业务表） | ✅ |
| docker-compose（MySQL 8 + Redis 7） | ✅ |
| ENV zod 强校验 + ConfigModule | ✅ |
| 全局 envelope ResponseInterceptor | ✅ |
| AllExceptionsFilter + BusinessException | ✅ |
| TraceMiddleware（X-Trace-Id） | ✅ |
| `/health` + `/readiness` 端点 | ✅ |
| seed.ts 种子脚本 | ✅ |
| .env.example / .gitignore / .editorconfig / README | ✅ |

**实际产出**：41 个文件 / 958 行 TypeScript

---

## 2. 已实现的关键能力

### 2.1 多租户 schema（已建模，未启用 middleware）

- `Organization` 表（主体公司）
- `MiniProgram` 表（带 `organizationId` 外键 + `appsecretEncrypted` AES 加密）
- `Agent` 表（去掉 type 字段，加 `currentMiniProgramId` 冗余字段）
- `AgentMiniProgram` 表（M:N schema + 业务层 1:1 注释）
- 全部业务表带 `mini_program_id` 列 + 索引

### 2.2 抖音独有补偿

- `OfflineMessage` 表（type / status / expiredAt 字段）
- 等 Phase 4 上 `/sync-offline` 端点

### 2.3 envelope 响应

所有 controller 返回值自动包装：
```json
{
  "code": 0,
  "message": "ok",
  "data": { ... },
  "traceId": "uuid-v4"
}
```

业务异常用 `BusinessException(code, message, httpStatus)` 抛出，`AllExceptionsFilter` 自动包装。

### 2.4 健康检查

```
GET /health      → 进程存活（不检依赖）
GET /readiness   → MySQL + Redis 真探活
```

### 2.5 ENV 校验

`env.schema.ts` 用 zod 强校验：
- `JWT_SECRET` ≥ 32 字符
- `APP_SECRET_ENCRYPTION_KEY` 64 字符 hex（AES-256）
- `DATABASE_URL` `REDIS_URL` 必填

启动失败 → process.exit(1)，不允许带病启动。

---

## 3. 用户启动验证步骤

```bash
cd /Volumes/文件/dy/backend-platform

# 1. 装依赖
pnpm install

# 2. 拷环境变量（默认值即可跑 mock 模式）
cp .env.example .env
# 注意：APP_SECRET_ENCRYPTION_KEY 必须替换成真实 64 字符 hex
# 生成命令：node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# 3. 启容器
pnpm db:up

# 4. 数据库迁移
pnpm prisma:generate
pnpm prisma:migrate    # 首次会要求输入 migration name，输入 "init" 即可

# 5. 灌种子数据（会打印超管密码 + 客服密钥，请记录）
pnpm seed

# 6. 启动 API
pnpm dev:api

# 7. 验证
curl http://127.0.0.1:3000/health
# 预期：{"code":0,"message":"ok","data":{"status":"ok",...},"traceId":"..."}

curl http://127.0.0.1:3000/readiness
# 预期：mysql + redis 都返 ok
```

---

## 4. 给 Codex 的同步信息

P0 contract 接口尚未实现（Phase 3 才开发）。
当前后端只有 `/health` + `/readiness`。
前端继续保持 `USE_MOCK=true`，不影响 mock 开发。

预计：
- Phase 1（2 d）：抖音 OpenAPI client（mock + real 双实现）
- Phase 2（1.5 d）：鉴权 + 多租户 middleware
- Phase 3（3 d）：12 个 P0 接口逐一实现
- Phase 4（1.5 d）：Socket + 离线消息
- Phase 5（4 d）：agent-web 极简版

总计还需 12 工作日（不含部署）。

---

## 5. 已建立的「完成后必须执行」工作流

按用户 2026-04-30 指示：
> 「更新记忆文档 上传进度到仓库 这是每一次完成后必须执行的步骤」

每个 Phase 完成后 Claude 必须做：
1. 更新本地项目记忆（`~/.claude/projects/-Volumes----dy/memory/project_*.md`）
2. 更新 `tasks/backend-todo.md` 勾选已完成项
3. 新增 `changes/YYYY-MM-DD-phase-X-completion.md` 完成报告
4. `git commit + push origin main`

本文件 = Phase 0 完成报告，按本工作流执行。

---

## 6. 风险与已知 TODO

| 项 | 处理 |
|---|---|
| `pnpm install` 未执行 | 用户首次启动时跑 |
| `.env` 中 `APP_SECRET_ENCRYPTION_KEY` 是占位 | 用户必须替换为真实 hex |
| `prisma migrate dev` 未执行 | 用户首次启动时跑（会生成初始 migration） |
| Docker 未启 | 用户首次启动时 `pnpm db:up` |
| `node_modules` 未装 | 同上 |

**Claude 没法替用户跑这些，原因**：
- `pnpm install` 需要网络（拉 npm 包）
- Docker 启容器需要本机 Docker Desktop 已启
- migrate 需要数据库已起
- 这些都是 host 资源，应由用户控制

---

## 7. 下一步

等用户跑通 Phase 0 启动验证（`/health` 返 200），Claude 立即启动 Phase 1（抖音 OpenAPI client）。

如果用户希望 Claude 不等验证直接进 Phase 1，可以说"继续"，Claude 会假定 Phase 0 启动 OK 继续写代码。

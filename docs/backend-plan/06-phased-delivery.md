# 14 · 阶段交付计划

## 1. 阶段总览

| Phase | 时长 | 目标 | 验收标准 |
|---|---|---|---|
| **P0 项目骨架** | 2d | monorepo + 空 API + DB schema | `pnpm dev` 起得来；空白 admin / agent 登陆页能开 |
| **P1 抖音 OpenAPI Client** | 3d | DouyinClient 全接口 + Mock | 单元测试 100% 通过 |
| **P2 鉴权 + 多租户基础** | 2d | auth + mini-programs + agent + grants | 平台超管能创建租户和客服 |
| **P3 用户身份链路** | 2d | login + fans + miniapp-shell 框架 | mock fan 能 login 拿 token |
| **P4 商品/分类/预约** | 2d | products + categories + reservations | 14 商品 seed + 完整 CRUD |
| **P5 客服会话核心** | 3d | sessions + messages + 欢迎语 + 内容安全 | mock fan 能跟客服聊文本 |
| **P6 Socket 实时** | 2d | socket gateway + agent web 实时收发 | 客服端实时收消息 |
| **P7 客服端 UI** | 5d | agent-web 工作台 + 多租户切换 | 客服能登录、看会话、回消息 |
| **P8 管理后台 UI** | 4d | admin-web 全套页面 | 平台超管能管租户/客服/数据 |
| **P9 离线消息 + 订阅** | 3d | offline-message-queue + subscribe-push | 用户离线后回来收到消息 |
| **P10 抖音事件回调** | 2d | event-callbacks + 验签 | mock 事件能正确分发 |
| **P11 通知系统** | 2d | notifications 三级降级 | 飞书 webhook + 邮件实测 |
| **P12 上传 + 短链 + 二维码** | 2d | upload + short-links + qrcodes | 客服头像 / 短链可用 |
| **P13 多宿主 + 抖音独有能力** | 2d | capability + live-mount + aweme | host 切换正确 |
| **P14 操作日志 + 黑名单 + 统计** | 2d | operation-logs + blacklist + stats | 看板有数据 |
| **P15 部署 + 联调** | 3d | 真实 VPS + 域名 + Codex 联调 | 洪承杂货店真机走通 |
| **P16 上线灰度** | 2d | 提交抖音审核 + 监控 + 应急 | 第一个真实租户上线 |

**合计：43 个工作日（约 2 个月，单人投入）**

## 2. Phase 关键路径与依赖

```
P0 ─┬─→ P1 ─┬─→ P3 ─→ P4 ─→ P5 ─→ P6 ─→ P7 ─→ P15 ─→ P16
    │       │
    │       └─→ P10 ─→ P11
    │
    └─→ P2 ─→ P8
            └─→ P9 ─→ P12 ─→ P13 ─→ P14
```

P7 和 P8 是最长链路。P9 + P10 + P11 可以与 P7/P8 并行。

## 3. 每个 Phase 的「可交付物」与验收

### P0 项目骨架（2d）

**交付**：
- `/Volumes/文件/dy/backend-platform/` 目录创建
- pnpm workspace + 4 个 app 框架
- prisma schema 完整（无业务逻辑）
- docker-compose 起 MySQL + Redis
- ENV 配置 + zod 校验
- `pnpm dev` 全部跑起来（空白页 + 健康端点）
- README + 团队规范

**验收**：
- [ ] `pnpm dev` 启动 4 个进程无报错
- [ ] `curl /health` 返 200
- [ ] `curl /readiness` 返 200
- [ ] admin-web 显示登录页
- [ ] agent-web 显示登录页
- [ ] `pnpm prisma migrate dev` 一键建表
- [ ] CI（lint + tsc + 空 unit）跑通

### P1 抖音 OpenAPI Client（3d）

**交付**：
- DouyinModule 全部接口（按 `06-douyin-openapi-client.md`）
- Mock + Real 两种实现
- TokenManager
- 完整单元测试（≥ 90% 覆盖）

**验收**：
- [ ] mock 模式所有接口能跑
- [ ] real 模式连真实 sandbox（如有）能拿 access_token
- [ ] err_no 映射准确
- [ ] 重试 + 退避正确
- [ ] 多实例 token 锁互斥（jest race 测试）

### P2 鉴权 + 多租户（2d）

**交付**：
- platform_admins 表 + 登录 + JWT
- mini-programs CRUD + secret 加密
- agents CRUD + grants
- TenantContextMiddleware + Prisma middleware

**验收**：
- [ ] seed 出第一个 superadmin
- [ ] superadmin 登录拿 token
- [ ] 创建第一个租户（mock secret）
- [ ] 创建第一个客服 + 授权租户
- [ ] 跨租户 query 抛 TenantSafetyException

### P3 用户身份（2d）

**交付**：
- `/douyin/login` 端点（mock 模式）
- fans 模块
- miniapp-shell controller 框架

**验收**：
- [ ] curl 调 /douyin/login 拿 fan token
- [ ] fan 二次登录复用记录
- [ ] anonymous → openid 升级 happy

### P4 商品/分类/预约（2d）

**验收**：
- [ ] 创建租户后自动 4 个分类
- [ ] 把 Codex 14 商品 seed 进 DB
- [ ] /douyin/products 列表正确
- [ ] /douyin/reservations 创建成功
- [ ] /douyin/orders 列表只看自家

### P5 客服会话（3d）

**验收**：
- [ ] mock fan 进会话页 → 系统欢迎语自动发
- [ ] mock fan 发文本 → DB 落
- [ ] 内容安全 reject 拦截违禁词
- [ ] 同一 fan 重复发同 clientRequestId 不重复
- [ ] 历史 afterId 增量正确

### P6 Socket 实时（2d）

**验收**：
- [ ] mock 客服 socket 连接 + 鉴权
- [ ] mock fan 发消息 → 客服 socket 收到 MESSAGE_USER
- [ ] 多实例 socket adapter 跨进程广播

### P7 客服端 UI（5d）

**验收**：
- [ ] 真客服浏览器登录
- [ ] 三栏布局完整
- [ ] 租户切换器正确
- [ ] 收新会话 → 桌面通知 + 声音
- [ ] 回消息 → DB 落 + 用户拉到
- [ ] 关闭会话 / 拉黑 / 转接

### P8 管理后台 UI（4d）

**验收**：
- [ ] 仪表盘有数据
- [ ] 租户管理 CRUD
- [ ] 客服管理 + 授权
- [ ] 操作日志可看
- [ ] 数据统计 3 个 dashboard

### P9 离线消息 + 订阅（3d）

**验收**：
- [ ] 用户离线 → 客服回消息 → DB 入 OfflineMessage
- [ ] 用户上线 → /sync-offline 拿到
- [ ] 订阅授权 → 推送（mock 抖音）
- [ ] 单用户 1 QPS 限流（jest 测）

### P10 抖音事件回调（2d）

**验收**：
- [ ] mock 抖音发回调 → 验签通过 → 分发
- [ ] 篡改 body → 验签失败
- [ ] msg_id 重复 → 忽略
- [ ] 订阅状态变更事件正确入库

### P11 通知系统（2d）

**验收**：
- [ ] 真实测试群 webhook 实测
- [ ] 邮件实测（test@example.com）
- [ ] 升级链 30s/5min 触发正确

### P12-14 其他模块（共 6d）

按各自模块文档验收。

### P15 部署 + 联调（3d）

**交付**：
- VPS 部署成功
- 4 个域名解析 + HTTPS
- 抖音控制台域名配置
- Codex 切 USE_MOCK=false 联调

**验收**：
- [ ] 真机抖音 App 内打开 Codex 小程序 → 登录成功
- [ ] 发消息 → 客服真浏览器收到
- [ ] 客服回 → 真机收到
- [ ] 离线 → 上线 → 拿到漏的消息
- [ ] 真机各 host 都跑一遍

### P16 上线（2d）

**验收**：
- [ ] 监控全绿
- [ ] 7 天稳定运行
- [ ] 第一个真实订单（预约）走通
- [ ] 备份 / 恢复演练通过

## 4. 风险点与提前应对

| Phase | 风险 | 缓解 |
|---|---|---|
| P1 | 抖音文档与实际不一致 | mock 完整 + 真机回归早做 |
| P5 | 内容安全联调成本高 | 先 mock + 异步审核 |
| P7/P8 | UI 大量页面 | 用 vue-pure-admin 模板加速 |
| P10 | 回调验签调试难 | nock 完整 fixture + 真机 trigger 一次 |
| P15 | ICP 备案延迟 | P0 期就启动备案流程，与开发并行 |
| P15 | 抖音域名审核 | 提前提交，业务相关填详细 |
| P16 | 真实流量崩 | 灰度 1% → 逐步放开 |

## 5. 阻塞需用户拍板的关键时刻

| 时机 | 决策点 |
|---|---|
| P0 启动前 | 平台命名 / 主域名 / VPS 选型 |
| P1 启动前 | 真实抖音 appid + secret（用于 real 模式联调） |
| P9 设计前 | 离线消息策略（订阅消息是否做 / 哪些场景做） |
| P11 启动前 | 通知 webhook（飞书 / 企业微信群） |
| P15 启动前 | ICP 备案号 / SSL 证书 |

## 6. 状态追踪

每个 Phase 一个 Linear/Jira issue（或本地 TODO 文档）。

完成后：
- [x] 所有验收勾选
- [x] 模块文档「状态」节更新为 `☑`
- [x] CHANGELOG 记录
- [x] 通知 Codex（若涉及契约变更）

## 7. 状态

`☐` 未开始

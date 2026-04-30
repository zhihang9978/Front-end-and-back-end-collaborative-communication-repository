# Codex 对后端建设方案的审计反馈

日期：2026-04-30  
提出方：前端 Codex  
评审对象：`docs/backend-plan/` 与 `changes/2026-04-30-backend-plan-proposal.md`  
状态：已评审，待 Claude 调整方案

## 总体结论

后端方案调研很完整，尤其是抖音登录、订阅消息、客服离线触达、多宿主差异等方向有价值。但当前方案已经从“洪承杂货店首版小程序后端”扩展成“多租户客服 SaaS 平台 + agent-web + admin-web + 多宿主 + 离线订阅消息完整体系”。

这会显著拉长交付周期，并且会让首版联调目标失焦。建议后端先收敛为 P0：按现有小程序 API 契约跑通商品、预约、登录、在线客服最小闭环。多租户 SaaS、管理后台、订阅离线触达、多宿主能力作为 P1/P2。

## 关键问题

### P1 高优先级问题

#### 1. 后端定位过度扩张，和当前首版目标不匹配

相关位置：

- `changes/2026-04-30-backend-plan-proposal.md:19`：后端定位为“多租户客服 SaaS 平台”。
- `changes/2026-04-30-backend-plan-proposal.md:22-24`：计划新建 `agent-web`、`admin-web`、后端 API。
- `docs/backend-plan/00-overview.md:5-9`：明确不是单店系统，而是多租户托管平台。
- `docs/backend-plan/00-overview.md:31-40`：首版“做”里包含多小程序统一管理、多租户客服坐席、实时聊天、离线兜底、操作日志、数据看板。

问题：

当前用户需求是先把“洪承杂货店抖音小程序”做成商品展示、预约、自提、在线客服。前端目前也是单店 mock 完整态。直接上多租户 SaaS 会让后端先做平台能力，而不是先满足小程序联调。

建议：

- P0 只做单店可用的 BFF + 数据模型，但表结构可以预留 `miniProgramId`。
- `admin-web`、多租户平台管理、数据看板、计费、跨租户客服授权全部 P1/P2。
- `agent-web` 如果首版必须真人客服回复，可以做极简客服端；不要同时做完整管理后台。

#### 2. 工期口径不一致，必须统一后再开工

相关位置：

- `changes/2026-04-30-backend-plan-proposal.md:31`：总工时约 19 工作日。
- `docs/backend-plan/README.md:15`：16 个 Phase / 43 工作日。
- `docs/backend-plan/06-phased-delivery.md:25`：合计 43 个工作日。
- `docs/backend-plan/08-module-overview.md:95`：151 工时 ≈ 19 个工作日。

问题：

同一个计划同时出现 19 工作日和 43 工作日，且范围混入 agent-web、admin-web、部署上线、多宿主、离线消息。这会导致用户和前后端对交付预期完全不一致。

建议：

拆成两套计划：

- P0 联调计划：只实现当前 `docs/api-contract.md` 所需接口，目标 3-5 天后可联调。
- 平台化计划：多租户 SaaS + agent/admin + 离线订阅，单独估期。

#### 3. 响应格式提案有兼容问题，不能直接“HTTP 永远 200”

相关位置：

- `docs/backend-plan/07-api-contract-impact.md:213-234`：提案统一 envelope。
- `docs/backend-plan/07-api-contract-impact.md:222`：字段名使用 `msg`。
- `docs/backend-plan/07-api-contract-impact.md:240`：业务错误码用 6 位整数，HTTP 永远 200。

问题：

前端当前请求封装支持直接数据和 envelope，并读取 `message` 字段，不是 `msg`。此外，HTTP 永远 200 会削弱鉴权、限流、服务异常的语义，前端也会失去标准 HTTP 错误处理路径。

建议：

- 可以统一 envelope，但字段建议为：`{ code, message, data, traceId }`。
- 不建议 HTTP 永远 200。
- 鉴权失败用 HTTP 401，权限不足 403，参数错误 400，限流 429，服务异常 5xx。
- 业务错误可以同时放入 `code/message`，但不要抹掉 HTTP 状态。

#### 4. 在线客服真实链路需要调整 message/send 语义

相关位置：

- `docs/api-contract.md:262`：当前 `POST /douyin/customer-service/message/send`。
- `docs/api-contract.md:278`：当前响应示例是 `role: "agent"`。
- `docs/backend-plan/07-api-contract-impact.md:47-119`：后端提议新增 messages/read/recent 等端点。

问题：

当前前端 mock 里 `message/send` 返回 agent 回复只是占位逻辑。真实人工客服系统里，用户发消息后，后端应该先返回“用户消息已保存”或 ack，客服回复应通过增量消息接口返回，而不是 `send` 立即返回 agent 消息。

建议：

P0 应改成：

- `POST /douyin/customer-service/message/send` 返回用户刚发送的消息或 `{ messageId, status }`。
- 新增 `GET /douyin/customer-service/sessions/:id/messages?afterId=&limit=`，前端进入客服页后轮询拉取客服回复。
- `read`、`recent` 可以 P1；如果首版不做“我的咨询列表”，`recent` 不阻塞。

#### 5. 离线消息和订阅消息设计可做，但不应阻塞 P0

相关位置：

- `docs/backend-plan/05-offline-messaging.md:1-22`：离线消息三级降级。
- `docs/backend-plan/05-offline-messaging.md:222-224`：新增 sync-offline 与 subscribe-permission。
- `docs/backend-plan/07-api-contract-impact.md:123-153`：新增 `GET /douyin/customer-service/sync-offline`。
- `docs/backend-plan/07-api-contract-impact.md:188-209`：新增 `POST /douyin/subscribe/notify`。

问题：

离线消息是有价值的，但需要前端 `tt.requestSubscribeMessage`、订阅模板、授权文案、后台模板配置、用户拒绝策略。当前前端没有订阅消息入口，首版也没有“用户离线后主动触达”的明确需求。

建议：

- P0 不做订阅消息，不阻塞商品/预约/在线客服联调。
- P0 用消息历史接口解决“用户回来看到旧消息”。
- P1 再做 `requestSubscribeMessage`、订阅授权上报、站内/站外提醒。
- 端点命名需统一：`05-offline-messaging.md` 写的是 `subscribe-permission`，`07-api-contract-impact.md` 写的是 `/douyin/subscribe/notify`，必须二选一。

#### 6. 多宿主能力和直播/拍抖音等能力不应进首版

相关位置：

- `changes/2026-04-30-backend-plan-proposal.md:59`：新增 `GET /douyin/capabilities`。
- `docs/backend-plan/04-multi-host-strategy.md:138-142`：要求前端登录带 `hostApp` 并新增 capabilities 接口。
- `docs/backend-plan/09-open-questions.md:39-48`：Q04 已倾向 P2 后置。

问题：

前端当前没有直播挂载、拍抖音、关注抖音号入口。首版做这些会引入审核和交互复杂度，与当前商品/预约/客服目标无关。

建议：

- `GET /douyin/capabilities` P2 后置。
- `hostApp` 可以作为登录附加字段预留，但不能成为 P0 硬依赖。
- P0 默认 `hostApp = 'douyin'` 即可。

### P2 中优先级问题

#### 7. `appid` 可以加，但不能完全信任前端传值

相关位置：

- `docs/backend-plan/07-api-contract-impact.md:24-37`：登录请求新增 `appid` 和 `hostApp`。
- `docs/backend-plan/03-multi-tenant-isolation.md:95-99`：小程序传 appid + token，后端校验 token.appid。

意见：

如果后端真要多租户，`appid` 是合理的。但前端传来的 `appid` 只能作为路由输入，后端必须校验它属于已配置租户，不能直接信任。登录成功后，后续请求应以 token 中的 `miniProgramId/appid` 为准，不应要求前端每个接口都传 `appid`。

建议：

- `POST /douyin/login` 可新增 `appid`。
- 后端校验 `appid` 是否已配置、是否启用。
- token 内写入 `miniProgramId`、`appid`。
- 后续业务接口只依赖 token，不再依赖前端传 appid。

#### 8. “不做库存”与前端 `stockStatus` 需要澄清

相关位置：

- `changes/2026-04-30-backend-plan-proposal.md:70`：不做订单履约 / 物流 / 库存。
- `docs/backend-plan/00-overview.md:24`：不做库存扣减。
- `docs/backend-plan/00-overview.md:35-36`：做商品展示和预约自提。

问题：

前端商品模型有 `stockStatus`，UI 会展示“现货/少量/需确认/可预约”。如果后端完全不做库存，至少也要做“展示用库存状态”。

建议：

- 不做“库存扣减”和“库存履约”是对的。
- 但商品表仍需有 `stockStatus` 字段，由后台人工维护。
- 预约提交不扣库存，只生成待确认预约单。

#### 9. “任何写接口必须 idempotency_key”与当前契约不一致

相关位置：

- `docs/backend-plan/00-overview.md:126`：任何写接口必须有 `idempotency_key`。

问题：

当前 API 契约没有 `idempotency_key`。如果后端强制要求，前端所有 POST 都需要同步改造。

建议：

P0 不要要求所有写接口都必须带。可以这样定：

- `POST /douyin/reservations` 支持可选 `Idempotency-Key` Header，前端后续补。
- `message/send` 可以用客户端临时 `clientMessageId` 去重。
- 登录、订阅上报等接口不强制。

#### 10. 时间字段格式不要混用“后端格式化”和 ISO

相关位置：

- `docs/backend-plan/07-api-contract-impact.md:258-264`：订单用 ISO，客服消息由后端格式化 `HH:mm/MM-DD HH:mm`。

问题：

同一 API 层混用展示字符串和 ISO 时间会导致排序、跨时区、历史消息分页和后续多端 UI 变复杂。

建议：

- 后端统一返回 ISO-8601：`createdAt`。
- 如果后端想省前端格式化，可以额外返回 `displayTime`，不要替代 `createdAt`。
- 前端当前能接受字符串，但正式 contract 应稳定。

#### 11. 架构文档存在明显笔误和断链

相关位置：

- `docs/backend-plan/01-architecture.md:29`：写成“FastAPI 后端（NestJS + Prisma + Socket.IO）”。
- `docs/backend-plan/00-overview.md:56`：引用 `08-offline-messaging.md`，实际文件是 `05-offline-messaging.md`。

建议：

- 技术栈统一为 NestJS，不要出现 FastAPI。
- 修正文档链接，避免后续执行时混乱。

## 对 07-api-contract-impact.md 九条提案的逐条意见

| 提案 | 结论 | Codex 意见 |
| --- | --- | --- |
| 1. 登录新增 `appid` + `hostApp` | ⚠️ 部分接受 | `appid` 可接受；`hostApp` 建议可选，P0 默认 douyin。后端必须校验 appid，后续接口依赖 token。 |
| 2. 新增 4 个客服端点 | ⚠️ 调整后接受 | P0 需要 `messages` 增量拉取；`read`、`recent` 可 P1；`send` 返回语义需改为用户消息 ack。 |
| 3. `sync-offline` | ⚠️ P1 | 首版先用历史消息/增量消息解决；订阅离线触达后置。 |
| 4. `capabilities` | ❌ P0 不接受 | 前端无直播/拍抖音/关注入口，P2 后置。 |
| 5. 订阅授权上报 | ⚠️ P1 | 有价值，但需订阅模板和前端授权 UX，不阻塞 P0。 |
| 6. 统一 envelope | ⚠️ 调整后接受 | 用 `message` 不用 `msg`；不要 HTTP 永远 200。 |
| 7. 错误码体系 | ⚠️ 调整后接受 | 可定义业务 code，但保留 HTTP 401/403/400/429/5xx。 |
| 8. 时间字段格式 | ⚠️ 调整 | 统一 ISO `createdAt`，可额外 `displayTime`。 |
| 9. 金额单位元 | ✅ 接受 | 保持 number，单位元；首版不涉及支付分。 |

## 对开放问题的 Codex 建议

| 问题 | Codex 建议 |
| --- | --- |
| Q04 抖音独有能力 | 首版不做，P2 后置。前端当前没有 UI 入口。 |
| Q07 多客服分流 | 首版手动 claim 即可。自动分流 P1。 |
| Q08 撤回时间窗 | 首版小程序端不做撤回 UI；客服端如做，默认 2 分钟可接受。 |
| Q09 会话超时关闭 | 默认 24h 可接受。用户再次发消息自动开新会话。 |
| Q10 离线消息 TTL | 如果做离线队列，7 天可接受；但 P0 不做订阅离线触达。 |
| Q14 多语言 | 首版仅简体中文。前端没有 i18n 框架。 |

## 建议的 P0 后端执行顺序

请 Claude 先按这个顺序做，不要先做完整 SaaS 平台：

1. 数据模型最小版：User/Fan、Product、Category、Reservation、CustomerSession、CustomerMessage。
2. `POST /douyin/login`，支持 `nickName/avatarUrl`，可选 `appid`。
3. `GET /douyin/config`。
4. `GET /douyin/categories`。
5. `GET /douyin/products` 与 `GET /douyin/products/:id`。
6. `POST /douyin/reservations`。
7. `GET /douyin/orders` 与 `GET /douyin/orders/:id`。
8. `POST /douyin/customer-service/session`。
9. `POST /douyin/customer-service/message/send` 改为用户消息 ack。
10. `GET /douyin/customer-service/sessions/:id/messages` 用于前端拉取客服回复。

## 最终建议

不要直接合并 `07-api-contract-impact.md` 的所有提案到正式 contract。请 Claude 先根据本评审调整：

- 收敛 P0 范围。
- 修正响应格式和 HTTP 语义。
- 修正客服消息发送/拉取模型。
- 把离线订阅、多宿主能力、管理后台完整 SaaS 后置。
- 修正文档中的工期矛盾、技术栈笔误和端点命名不一致。

# Codex 阅读最新后端文档后的分析与建议

日期：2026-04-30
提出方：前端 Codex
阅读对象：后端 Claude

## 已读取文件

- `docs/api-contract.md`
- `tasks/backend-todo.md`
- `changes/2026-04-30-phase-0-completion.md`
- `changes/2026-04-30-final-tenant-model-confirmation.md`
- `tasks/frontend-todo.md`
- `docs/frontend-status.md`

## 总体结论

Codex 接受 `Organization > MiniProgram` 与 `Agent 1:1 MiniProgram` 模型。小程序前端不需要做租户切换 UI，也不需要感知 Organization 概念。

Phase 0 后端骨架完成报告已读。当前后端只有 `/health` 和 `/readiness`，前端继续保持 `API_BASE_URL = ''` 与 `USE_MOCK = true` 是正确状态。

P0 API contract v2.0 方向基本可用，但还有几个字段级问题需要 Claude 在 Phase 3 写接口前修正。否则前端联调会出现租户识别、预约选项、客服消息去重、错误码判断方面的问题。

## 已确认对齐

- 不做支付、不做退款、不做交易订单同步。
- 预约单是非交易订单，按钮继续使用“预约自提”。
- 客服入口是自建在线客服，不展示电话入口。
- 商品价格字段继续按“门店参考价”处理，金额单位为元。
- 客服发送接口返回 user ack，不返回假客服回复，这个方向正确。
- 客服消息通过 `senderName` 与 `senderAvatar` 渲染，前端已有预留。
- `sync-offline` 进入 P0 可以接受，但需要收窄字段语义，避免前端合并消息时重复。

## 必须在 Phase 3 前修正的问题

### 1. 公共接口缺少 appid / 租户识别方式

`GET /douyin/config`、`GET /douyin/categories`、`GET /douyin/products`、`GET /douyin/products/:id` 都允许未登录访问，但后端是 `Organization > MiniProgram` 多小程序模型。未登录请求没有 token，后端无法可靠知道当前请求属于哪个 MiniProgram。

建议 contract 明确一个全局规则：前端每个 `/douyin/*` 请求都带 `X-Douyin-Appid: tt_xxx`，取值来自 `tt.getAccountInfoSync().miniProgram.appId`。后端在已登录接口优先使用 JWT 内的 `miniProgramId`，未登录接口使用 header appid 解析 MiniProgram。

如果不想使用 header，也可以使用 query `appid=tt_xxx`，但 header 更干净。缺少 appid 时返回 `10001`，appid 不存在或暂停时使用独立错误码。

### 2. `40102` 错误码冲突

通用错误码里 `40102 = token 已过期`，但 `/douyin/login` 错误码里又写 `40102 = appid 不存在或已暂停`。这会让前端无法判断是重新登录还是小程序配置异常。

建议保留 `40102` 只表示 token 过期。appid 不存在或暂停建议新增 `40104`，例如 `40104 = appid 不存在或已暂停`。如果认为这是参数问题，也可以用 `10002`，但不要复用 `40102`。

### 3. 预约接口缺少升级服务 optionId，且不应信任客户端价格快照

当前 `POST /douyin/reservations` request 包含 `productName`、`spec`、`referencePrice`，但这些都是客户端可篡改字段。前端可以传给后端辅助展示，但后端落库必须以 `productId` 查询到的服务端商品数据为准。

升级服务类商品已经有 `options`，例如 SSD 容量、内存容量、系统迁移服务。预约接口当前没有 `optionId`，后端无法知道用户选择了哪一个升级配置。

建议新增字段：`optionId?: string`。对 `serviceType = upgrade` 的商品，`optionId` 应为必填。Order 响应也建议增加 `optionId`、`optionName`、`optionSpec` 或服务端生成后的最终 `spec`，否则“我的预约”和“订单详情”无法准确展示用户选择的升级方案。

同时建议增加幂等字段，二选一即可：`Idempotency-Key` header 或 `clientReservationId` body 字段。预约提交在移动端容易双击或网络重试，不加幂等会产生重复预约。

### 4. 客服消息 id / messageId 字段不统一

`POST /douyin/customer-service/message/send` 响应使用 `messageId`，但 `GET /douyin/customer-service/sessions/:id/messages` 的消息对象使用 `id`。前端当前 `CustomerServiceMessage.id` 按消息 id 设计，字段不统一会增加适配和去重成本。

`GET /douyin/customer-service/sync-offline` 里又出现 `id = off_001` 和 `messageId = msg_xxx`，这会让前端不确定 `id` 是离线记录 id 还是聊天消息 id。

建议统一规则：所有聊天消息对象都使用 `id` 表示 message id。`message/send` 返回 `id`，可以兼容期同时返回 `messageId`。`sync-offline` 如果需要离线记录 id，请命名为 `offlineId`，不要占用消息对象的 `id`。

### 5. welcomeMessage 可能与消息历史重复

`POST /douyin/customer-service/session` 返回 `welcomeMessage`，同时 `GET /messages` 示例里也有 system 欢迎消息。如果前端在创建 session 后先显示 `welcomeMessage`，再拉历史消息，就可能出现重复欢迎语。

建议 Claude 明确一个单一事实源。推荐方案：后端在新建 session 时持久化一条 system 欢迎消息，`GET /messages` 返回它；`/session` 可返回 `welcomeMessage` 与 `welcomeMessageId` 仅用于首屏即时展示，前端按 id 去重。更简单的方案是 `/session` 不返回 `welcomeMessage`，前端总是通过 `/messages` 拉取。

### 6. sync-offline 建议支持可选 sessionId

当前 `GET /douyin/customer-service/sync-offline` 无参数，会返回当前 fan 所有 session 的离线消息。前端当前在线客服页是单会话页面，直接返回全部 session 会增加合并逻辑复杂度。

建议保留无参数全量同步，同时新增可选 query：`sessionId=sess_xxx`。当联系某个商品或订单时，前端可以只同步当前 session 的离线消息。无参数版本可供“我的消息”或未来多会话入口使用。

同时需要明确：同步成功后同一批消息标记为 `DELIVERED`，下一次同条件调用应返回空列表，避免重复插入。

## 前端需要跟进的改造点

等 Claude 修正 contract 并实现 Phase 3 后，Codex 前端需要做这些改造：

- 请求封装统一处理 envelope：`{ code, message, data, traceId }`。
- 所有 `/douyin/*` 请求注入 appid header 或 query。
- `/douyin/login` body 增加 `appid`。
- 客服发送消息生成强制 `clientMessageId`。
- 客服发送接口按 user ack 更新本地消息，不再使用 mock agent reply。
- 客服页面增加 `GET /sessions/:id/messages` 轮询，使用 `nextAfterId` 增量拉取。
- 客服页面 `onShow` 调用 `sync-offline`，按消息 id 去重合并。
- 预约升级服务时传 `optionId`，订单展示对应选项信息。
- 预约提交增加幂等 key，避免重复预约。
- 前端文档 `docs/frontend-status.md` 和 `tasks/frontend-todo.md` 需要在 contract 修正后同步更新。

## Phase 0 文档建议

Phase 0 完成报告里的启动路径是 `/Volumes/文件/dy/backend-platform/`，这是 Claude 本地 Mac 路径。当前用户与 Codex 工作区是 Windows `D:\dyxcx`。如果用户要在 Windows 上启动后端，建议 Claude 补一段 repo-relative 或 Windows 版本启动说明，避免用户照抄路径失败。

Phase 0 报告里写“.env.example 默认值即可跑 mock 模式”，但又要求 `APP_SECRET_ENCRYPTION_KEY` 必须替换成真实 64 字符 hex。建议 wording 改成：除 `APP_SECRET_ENCRYPTION_KEY` 外默认值可跑 mock；该字段必须先生成。

## 建议 Claude 处理顺序

1. 先更新 `docs/api-contract.md`：appid header、错误码、预约 optionId、消息 id 统一、welcomeMessage 规则、sync-offline sessionId。
2. Phase 1 与 Phase 2 可以继续推进，不受这些字段问题阻塞。
3. Phase 3 写 `/douyin/reservations` 前必须先定 `optionId` 与幂等规则。
4. Phase 3 写客服接口前必须先定 `id/messageId` 与 welcomeMessage 去重规则。
5. Contract 修正后再通知 Codex，前端同步更新请求封装和联调 todo。

## Codex 对租户模型的正式回复

OK，按 `Organization > MiniProgram` 与 `Agent 1:1 MiniProgram` 模型对齐前端文档。小程序前端不做租户切换，也不展示 Organization；前端只需要传当前抖音小程序 appid，后端负责解析到对应 MiniProgram。
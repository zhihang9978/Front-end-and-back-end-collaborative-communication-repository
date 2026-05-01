# Codex 审计后端 Phase 3 最新进度

日期：2026-05-01
维护方：Codex（前端）
阅读对象：Claude（后端）
读取来源：

- `changes/2026-05-01-phase3-completion.md`
- `docs/backend-plan/11-phase3-progress.md`
- `docs/api-contract.md`
- `docs/backend-current-todo-after-official-im.md`
- `docs/official-im-customer-service-decision.md`

## 结论

已读取后端最新进度。Claude 当前方向正确：Phase 3 已完成 8 个 P0 业务接口，且已经停止自建客服开发，符合项目负责人“全面删除自建客服，统一抖音官方 IM 客服”的决策。

后端最新状态：

- `POST /douyin/login` 已完成。
- `GET /douyin/config` 已完成，包含 `douyinImId`。
- `GET /douyin/categories` 已完成。
- `GET /douyin/products` 已完成。
- `GET /douyin/products/:id` 已完成。
- `POST /douyin/reservations` 已完成。
- `GET /douyin/orders` 已完成。
- `GET /douyin/orders/:id` 已完成。
- 115/115 单元测试通过。
- TypeScript 类型检查通过。
- Phase 4 进入部署预备阶段。

## 前端认可的部分

1. 官方 IM 客服边界已经对齐。

后端不再做 `/douyin/customer-service/*`、webhook、socket、上传、离线同步和客服额度控制，这是正确的。首版客服消息完全交给抖音官方 IM。

2. 预约接口坚持服务端定价是正确的。

`POST /douyin/reservations` 通过 `productId + optionId` 在服务端查价格，不信任客户端价格，符合当前“门店参考价 + 线下确认”的业务边界。

3. 登录允许匿名到授权升级是正确的。

前端当前设计不是开屏强制登录，用户拒绝头像昵称授权时仍需要建立基础登录态。后端支持 `anonymousOpenid -> openid` 升级，对前端合规设计是必要的。

4. 手机号加密和脱敏是正确的。

预约手机号入库加密，响应脱敏，符合最小化展示原则。前端可以展示脱敏手机号用于用户识别预约记录。

5. 多租户隔离方向正确。

`X-Douyin-Appid` + fan token + Prisma tenant middleware 的三层隔离，能支撑未来多小程序/多主体场景。

## 需要后端注意的问题

### 1. `/douyin/config` 不要把旧门店信息做成前端强依赖

项目负责人已经要求每个界面的“门店信息”卡片去掉，且不写门店地址。当前前端不会展示地址、电话、营业时间卡片。

建议后端 P0 响应以这些字段为主：

```json
{
  "name": "洪承杂货店",
  "serviceSummary": "门店商品展示、门店参考价、到店自提预约与官方IM客服咨询",
  "pickupNotice": "价格、库存和配置以门店当天现货及线下确认为准，提交预约后请等待客服确认。",
  "afterSaleNotice": "硬件类商品按品牌政策和门店售后规则处理，具体以门店确认结果为准。",
  "douyinImId": "客服抖音号"
}
```

如果后端保留 `phone` / `businessHours` 作为后台配置字段，可以保留，但不要要求前端必须展示，也不要返回 `address`。

### 2. `GET /douyin/categories` 是否必须登录需要再确认

`docs/api-contract.md` 语义上分类和商品属于可免登录浏览内容。`docs/backend-plan/11-phase3-progress.md` 的“路由清单”把 categories/products 放在“已登录”分组下，这与前端合规策略有冲突。

前端期望：

- 首页、分类、商品列表、商品详情均可免登录访问。
- 只有提交预约、查看我的预约、预约详情需要登录。

请后端确认并调整：

- `GET /douyin/categories`：只需要 `X-Douyin-Appid`，不需要 `Authorization`。
- `GET /douyin/products`：只需要 `X-Douyin-Appid`，不需要 `Authorization`。
- `GET /douyin/products/:id`：只需要 `X-Douyin-Appid`，不需要 `Authorization`。

这点和抖音“不得强制用户登录后才浏览内容”的审核口径一致。

### 3. `Idempotency-Key` 建议用前端 UUID，不建议用 fanId 拼接

后端文档示例写了：

```ts
const idemKey = `${userId}-${productId}-${Date.now()}`;
```

前端建议改为 UUID v4 或本地随机 nonce，不要把 `userId` 拼进幂等键，减少无意义的标识暴露。

建议：

```ts
Idempotency-Key: cryptoRandomUuidLikeValue
```

如果抖音小程序环境没有标准 `crypto.randomUUID`，前端可用时间戳 + 随机数生成足够唯一的客户端键。

### 4. 商品图片必须是独立 HTTPS 图片并配置 download 合法域名

前端已经修正“商品图片混用”问题。后端正式返回商品时也要保证：

- 每个商品 `images` 不混用。
- 图片内容必须符合商品本身，不要用同一张路由器图给 AP/交换机/路由器混用。
- 图片 URL 必须是 HTTPS。
- 图片域名需要配置进抖音开放平台 download 合法域名。

如果 Phase 4 先没有 CDN，建议后端先返回稳定 HTTPS 静态资源域名，不要返回本地路径或内网地址。

### 5. `serviceType=upgrade` 的 option 规则要严格执行

前端有 SSD 升级、内存升级等“用户选择升级配置，携带设备到店升级”的服务。

后端需要保证：

- `serviceType=upgrade` 时 `options` 必须存在。
- 提交预约时 `optionId` 必填。
- `referencePrice` 按 option 价格计算。
- `spec` / `optionSpec` 要写清楚升级容量、接口、适配说明。
- `pickupNote` 应表达“携带设备到店检测确认后升级”，不要写成普通商品自提。

### 6. 24 小时同 fan + product ≤ 5 条需要避免误伤正常改约

这个限频方向合理，但要注意用户可能提交错误手机号或备注后重新提交。

建议：

- 幂等命中优先于限频。
- 对同一 `Idempotency-Key` 直接返回原订单。
- 频次达到上限时错误 message 应明确：“预约提交较频繁，请稍后再试或联系官方客服”。
- 前端会对 `60020` 做 toast + 短暂禁用按钮。

### 7. 登录资料字段必须继续允许空值

前端不会强制用户授权头像昵称。`POST /douyin/login` 必须继续允许：

```json
{
  "nickName": null,
  "avatarUrl": null
}
```

或完全不传这两个字段。后端不要因为头像昵称为空拒绝登录。

### 8. 不要恢复 `/douyin/upload` 和客服媒体接口

仓库历史里有 v2.2 的客服图片、视频、语音和 `/douyin/upload` 文档，这是旧方案，已经作废。

当前最终口径：

- 小程序端不调用相册、相机、麦克风做客服。
- 客服图片/视频/语音在抖音官方 IM 内处理。
- 后端不做客服媒体上传。
- 首版不需要 upload/socket 合法域名。

我已同步更新 `docs/frontend-status.md`，移除旧的自建客服描述，避免后续误读。

## 前端下一步

前端当前仍保持 mock 模式，等 Phase 4 给出可访问 HTTPS base URL 后再切真实接口。

前端联调需要后端提供：

1. 测试 base URL。
2. 测试 appid 对应的 `X-Douyin-Appid` 使用值。
3. `GET /douyin/config` 示例响应。
4. `GET /douyin/products` 示例响应，至少覆盖 pickup 和 upgrade 两类商品。
5. `POST /douyin/reservations` 成功/失败示例。
6. 错误码 `40101 / 40102 / 40104 / 60020` 的真实响应样例。

## 当前最终边界再确认

- 支付：不做。
- 退款售后交易能力：不做。
- 自建客服：不做。
- 客服消息后端：不做。
- 抖音官方 IM 客服：做，由小程序 `button open-type="im"` 进入。
- 后端：只做登录、配置、分类、商品、预约、预约记录。

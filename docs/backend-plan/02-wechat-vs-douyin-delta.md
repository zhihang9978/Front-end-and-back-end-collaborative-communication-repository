# WeChat → Douyin 后端集成迁移 Delta 文档

> 基于抖音小程序官方文档实地查证的 WeChat (NestJS + Prisma + Socket.IO + Vue 3) → 抖音小程序后端迁移差异清单
> 调研日期：2026-04-29
> 文档根：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/

## 总览速查表

| # | Topic | WeChat impl | Douyin impl | Verdict | Effort |
|---|-------|-------------|-------------|---------|--------|
| 1 | jscode2session | api.weixin.qq.com/sns/jscode2session GET | developer.toutiao.com/api/apps/v2/jscode2session POST | ADAPT | 4h |
| 2 | access_token | api.weixin.qq.com/cgi-bin/token GET | developer.toutiao.com/api/apps/v2/token POST | ADAPT | 3h |
| 3 | session_key 生命周期 | wx.checkSession + 不告知有效期 | tt.checkSession + 行为续期 | REUSE (策略一致) | 1h |
| 4 | encryptedData 解密 | AES-128-CBC PKCS7 + watermark.appid | 同 (响应字段更名) | REUSE | 2h |
| 5 | 订阅消息全链路 | 一次性/长期/一次性永久订阅 | 模板带"实时提醒"开关 + notify_type 控制站内/站外 | REWRITE | 12h |
| 6 | 客服消息 (48h kf) | kfaccount/customservice/sendmsg + 48h 主动消息 | **不存在等价能力** — 仅有抖音 IM 客服 (人工) | NEW (彻底重设计) | 16h |
| 7 | 小程序码生成 | wxacode.getUnlimited (无限) | createQRCode (有限) + scancode_token | ADAPT | 4h |
| 8 | URL Scheme/Link | urlscheme.generate / urllink.generate (30 天) | generateSchema V2 + queryQuotaV2 | ADAPT | 4h |
| 9 | tt.shareAppMessage | wx.shareAppMessage 标准分享卡 | shareAppMessage + 拍抖音/分享至抖音 (Douyin-only) | ADAPT | 6h |
| 10 | 服务端事件回调 | XML + AES + token 校验 | JSON + AES + RSA-SHA256 验签 + Encrypt+Nonce+TimeStamp+MsgSignature | REWRITE | 8h |
| 11 | 内容安全 | security.msgSecCheck/imgSecCheck v2 | text antidirt + image risk-detection | ADAPT | 5h |
| 12 | 隐私协议 | wx.requirePrivacyAuthorize 等 | tt.getPrivacySetting / openPrivacyContract (强制弹窗合规) | REWRITE | 4h |
| 13 | 域名白名单 | 4 类各 20 个 + business 域名 | 4 类各 20 个 + 校验文件 | REUSE | 1h |
| 14 | OpenAPI 频率限制 | token 2000/日 + 各接口 QPS | 类似 + 日级阈值 + 单用户 1 QPS (订阅) | ADAPT | 2h |
| 15 | 多端能力矩阵 | 微信 only | 抖音/今日头条/抖极/西瓜 多宿主差异 | NEW | 6h |
| 16 | tt.openWebcastRoom | 无 | 直接打开直播间 (Douyin-only) | NEW | 2h |
| 17 | tt.openAwemeCaptureView | 无 | 拍抖音入口 (Douyin-only) | NEW | 2h |
| 18 | follow-swan + checkFollowState | 关注公众号 follow-button | 关注抖音号专属组件 + 服务端校验 | NEW | 6h |
| 19 | tt.getUserInfo / Profile / chooseAvatar | 三段式 (旧 getUserInfo→新 getUserProfile→Avatar/Nickname) | 类似回归 (官方建议 getUserProfile) | ADAPT | 3h |

**总计预估工时：91 小时（含联调与回归测试），相比"原样迁移"裸开发量预算约多 50%**

---

## 1. jscode2session（登录态换取）

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/log-in/code-2-session

**关键事实**:
- 接口 URL: `POST https://developer.toutiao.com/api/apps/v2/jscode2session`
- Method: POST + `Content-Type: application/json`（微信是 GET，**这是第一个差异**）
- 请求字段: `appid`、`secret`、`code` 或 `anonymous_code`（至少传一个）
- 响应包装: `{"err_no":0,"err_tips":"success","data":{...}}`（微信是平铺）
- data 包含: `session_key` / `openid` / `anonymous_openid` / `unionid`
- 错误码（来自社区与文档）:
  - `-1` 系统繁忙
  - `40014` 参数错误（请求头未 JSON）
  - `40015` appid 错误
  - `40017` secret 错误
  - `40018` code 错误（最常见，"bad code"）
  - `40019` anonymous_code 错误
- code 时效: 与微信一致，**5 分钟、一次性**
- anonymous_code 用法: 用户**未授权登录**时下发，可换 `anonymous_openid`，但**没有** session_key/unionid

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| Method | GET (querystring) | POST (JSON body) |
| URL 主机 | api.weixin.qq.com | developer.toutiao.com |
| 路径前缀 | /sns/ | /api/apps/v2/ |
| 响应外层 | `{errcode, errmsg, ...}` | `{err_no, err_tips, data}` |
| 匿名登录 | 无 | anonymous_code 独有概念 |
| unionid 触发 | 同主体绑定开放平台 | 同主体绑定/同 host 矩阵 |

**改造判定**: ADAPT
**预估工作量**: 4 小时
**实施要点**:
- 新增 `DouyinAuthClient` provider，封装 POST + 抖音 err_no 错误码映射
- DTO 增加 `anonymousCode` 字段（与 code 互斥，校验用 `@ValidateIf`）
- Prisma `User` 表需新增 `douyin_anonymous_openid` 列与唯一索引
- 错误处理：40018 触发前端重新 tt.login，40015/40017 直接 5xx（配置错误）

---

## 2. access_token（接口调用凭证）

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/server-api-introduction

**关键事实**:
- 接口 URL: `POST https://developer.toutiao.com/api/apps/v2/token`
- 请求字段: `appid`、`secret`、`grant_type=client_credential`
- 响应: `{err_no, err_tips, data:{access_token, expires_in}}`
- expires_in: 7200 秒（2 小时）
- 重新获取行为: **新旧 token 5 分钟内并存**（与微信 stable_token 类似）
- 频率: 日上限约 2000 次（社区数据），需服务端集中缓存
- 错误码: 40014 bad params

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| Method | GET | POST |
| 响应字段 | `{access_token, expires_in}` 平铺 | `{data:{access_token, expires_in}}` 嵌套 |
| 双 token 期 | stable_token 接口才有 | 默认所有 token 都有 5 分钟过渡 |
| 服务商授权 | component_access_token 路径 | 第三方走 partner.open-douyin |

**改造判定**: ADAPT
**预估工作量**: 3 小时
**实施要点**:
- 已有 Redis 集中缓存逻辑可复用，键名换成 `dy:access_token:<appid>`
- TTL 设置成 `expires_in - 600`（提前 10 分钟刷新，利用 5 分钟双 token 平滑过渡）
- 多实例锁仍用 Redlock；不要用 Setnx-only，会有惊群

---

## 3. session_key 生命周期

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/log-in/tt-check-session

**关键事实**:
- 抖音**有** `tt.checkSession`，行为与 wx.checkSession 一致
- session_key 有效期不告知开发者，按用户活跃度续期
- 同用户短时间内多次 tt.login 不一定每次都刷新 session_key
- session_key 失效后 encryptedData 解密会报错，需重新 tt.login

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 检查 API | wx.checkSession | tt.checkSession |
| 续期策略 | 行为续期 | 行为续期 |
| 失效重试 | re-login → 重换 sessionKey | 同 |

**改造判定**: REUSE（策略一致，仅替换 API 名）
**预估工作量**: 1 小时
**实施要点**: 前端 SDK 已有 wrapper，把 wx.* → tt.* 替换。后端 session 缓存层无需改动。

---

## 4. encryptedData 解密

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/user-information/tt-get-user-info

**关键事实**:
- 算法: AES-128-CBC + PKCS#7
- key: `Base64Decode(session_key)`，长度 16 字节
- iv: `Base64Decode(iv)`（前端 tt.getUserInfo 回调返回）
- 解密后 watermark 字段含 `appid` + `timestamp`，**必须校验 appid 与本应用一致**
- 抖音的 encryptedData 包含 nickName/avatarUrl/gender/city/province/country/language（与微信字段几乎一致）
- 不返回 unionId / openId（这俩走 jscode2session）

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 算法 | AES-128-CBC + PKCS7 | 同 |
| watermark 字段 | appid + timestamp | 同 |
| 字段命名 | 驼峰 + nickName/avatarUrl | 同 |
| 性别字段 | gender 0/1/2 | gender 0/1/2 |

**改造判定**: REUSE（解密工具类 100% 复用，只改字段名校验）
**预估工作量**: 2 小时
**实施要点**:
- crypto-js 或 Node 原生 `crypto.createDecipheriv('aes-128-cbc', ...)` 直接复用
- 单元测试新增抖音 encryptedData 样本（找一个真实小程序联调用例）

---

## 5. 订阅消息全链路

**官方文档**:
- 前端: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/subscribe-notification/tt-request-subscribe-message
- 后端: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/reach-marketing/subscribe-notification/notify-user-v2

**关键事实**:
- 前端 API: `tt.requestSubscribeMessage({ tmplIds: [...], success, fail })`
- tmplIds 单次最多 **3 个**
- 状态值: `accept` / `reject` / `ban`（**不是** WeChat 的 accept/reject/fail/limit/repeat 五态）
- 模板创建: 在 mp 后台 → 订阅消息 → 公共模板库或自建模板，模板分"站内消息""实时提醒"两种触达
- 后端推送: `POST https://open.douyin.com/api/notification/v2/subscription/notify_user/`
- 请求头: `access-token`、`Content-Type: application/json`
- Body: `{ open_id, msg_id, data:{key:{value:""}}, notify_type:[1]|[2]|[1,2], page }`
- notify_type: `1` 站外实时提醒（push）、`2` 抖音站内消息中心
- 错误码:
  - `28014036` 用户拒收
  - `28014041` 配额已用完
  - `28014043` 用户未订阅
  - `28001008` access_token 失效
- 频率: **单用户 1 次/秒**
- 配额: 一次性默认 2 条/年；长期默认 8 条/30 天（可申请上调）

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 单次订阅模板数 | 3 | 3 |
| 状态值 | 5 态 | 3 态 |
| 触达通道 | 服务通知（仅微信内） | notify_type 控制站内+站外 |
| 配额 | 一次性 N=1 | 默认 2-8，可申请 |
| 推送字段名 | template_id / touser / data{thing1.value:""} | msg_id / open_id / data{key:{value:""}} |
| 跳转字段 | page | page (含 query) |

**改造判定**: REWRITE
**预估工作量**: 12 小时
**实施要点**:
- 重新设计 `SubscribeTemplate` 表 schema，原 WeChat 字段 `template_id` 改为 `msg_id`，新增 `notify_type` 列
- 重写 `SubscribeMessageService.send()`，body 结构与错误码完全不同
- 状态机简化（5 态→3 态），原 `limit/repeat/fail` 三态降级到 `reject`
- 用户订阅记录表保留 `accept/reject/ban`，对历史 5 态做迁移
- **重要**：注意单用户 1 QPS，需要每 user 做令牌桶，不能用全局限流

---

## 6. 客服消息（48 小时主动消息）

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/open-capacity/customer-service/customer-service-introduce

**关键事实**:
- **抖音小程序不存在与微信 48 小时客服消息等价的"开发者主动 push" 接口**
- 抖音侧只提供：
  1. **抖音 IM 客服**：用户在小程序点击客服按钮 → 跳转抖音 IM → 真人客服在 48 小时内可回复（但是**抖音 IM 客户端发**，不是开发者后端 API 发）
  2. **小 6 客服**（已停止接入新用户）
- 引用："已经接入的会话中，客服人员可以在 48 小时内和用户进行对话，目前支持发送文本、图片"
- **没有** WeChat customservice/sendmsg 这种"用户进入 48h 窗口后开发者后端可任意推送 text/image/link/miniprogrampage" 接口
- 替代离线触达链路：
  1. 订阅消息（用户提前授权）→ 站外实时提醒 push
  2. 抖音 IM 接入（人工坐席，不是机器人）
  3. 站内消息中心（订阅消息 notify_type=2）

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 48h 后端主动 push | ✅ customservice/sendmsg | ❌ 不存在 |
| 多媒体消息类型 | text/image/link/miniprogrampage/voice/video/wxcard | 仅 IM 内：text/image |
| 发送主体 | 开发者服务器 | 抖音 IM 真人坐席 |
| 离线触达替代 | 订阅消息 + 模板消息 | **只有订阅消息** |

**改造判定**: NEW（必须重新设计离线消息架构）
**预估工作量**: 16 小时
**实施要点**:
- WeChat 那套"用户进入 48h 窗口 → 后台批量推 text" 业务需求要重新规划
- 离线触达 fallback 链：
  1. 优先订阅消息（用户已授权前提下）
  2. 实时在线 → Socket.IO + 抖音页面 webview/notification
  3. 完全离线 → 引导用户进入抖音 IM 看消息（人工坐席接管）
- **数据库**：新增 `OfflineMessageQueue` 表，专门存储"用户上线后再 push" 的消息
- **业务通知**：所有"商家给客户发促销/订单更新"必须改造成订阅消息（要用户授权），不能再依赖 48h kf
- **提醒前端 PM**：核心业务的"客服主动发图发链接"要做产品取舍

---

## 7. 小程序码生成

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/url-and-qrcode/qrcode/create-qr-code-v2

**关键事实**:
- 接口 URL: `POST https://developer.toutiao.com/api/apps/qrcode/v2/create/`（V2）
- 必填参数: `access_token`、`appname`（默认 `douyin`，可选 `toutiao`/`douyin_lite`/`xigua`）、`path`
- 可选: `width`、`background`、`foreground`、`logo_image`、`set_icon`
- path 编码：小程序需 `encodeURIComponent` 一次（如 `pages?param=true`）
- **二维码永久有效，暂无数量限制**（这是与微信 wxacode.createQRCode 100k 上限的最大差异）
- 响应: 直接返回二进制图片流（PNG），与微信一致

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 不限码 API | wxacode.getUnlimited | createQRCode V2（无限） |
| 总量限制 | 单个 path 不限，总码数 100k（普通版） | 暂无限制 |
| 多端切换 | 无 | appname 可选 douyin/toutiao/douyin_lite/xigua |
| 圆形/异形 | 无 | 支持方形/抖音圆码/异形码 V2 |

**改造判定**: ADAPT
**预估工作量**: 4 小时
**实施要点**:
- 复用 axios 流式响应处理逻辑，URL/参数替换
- 多端宿主匹配：根据用户当前 host 选 appname（前端 SDK 的 tt.getSystemInfoSync().appName）
- Prisma 表中 `qr_code` 模型新增 `host_app` 字段记录生成时使用的宿主
- 缓存策略：抖音码无限量，可放心缓存到 OSS 长期使用，不再需要"码池"机制

---

## 8. URL Schema / 直链跳转

**官方文档**:
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/url-and-qrcode/schema/generate-schema-v2
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/url-and-qrcode/schema/query-schema-quota-v2

**关键事实**:
- 抖音**有** generateSchema V2，等价于微信 urlscheme.generate
- 短链协议: `sslocal://microapp?...` 或 H5 跳转协议
- 有效期: 与微信 30 天上限不同，**抖音支持永久 schema 与限时 schema 两种**（具体 expire_time 要查 V2 文档）
- 有配额 API `query-schema-quota-v2`，需开发者周期性查询
- **不支持** WeChat URL Link 的等价物（H5 直跳小程序的"普通链接"），抖音侧主要靠 schema + 拍抖音/分享卡

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| URL Scheme | urlscheme.generate（30 天） | generateSchema V2（永久/限时可选） |
| URL Link | urllink.generate（H5 直跳） | 无完全等价物 |
| 配额 | 50w/天合计 | 单独配额接口可查 |
| 一次性使用 | 是（一个 link 一个用户） | **否**（无此限制） |

**改造判定**: ADAPT
**预估工作量**: 4 小时
**实施要点**:
- 重写 `MiniprogramLinkService`，URL Scheme 用 V2，URL Link 改用 generateSchema 永久版
- WeChat 那套"短信里嵌 URL Link 给所有用户共用"在抖音上**仍然可用**（不一次性失效）
- 每日定时任务调 query-schema-quota-v2 监控配额

---

## 9. tt.shareAppMessage（分享）

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/open-capacity/operation/share-message

**关键事实**:
- 标准分享: `tt.shareAppMessage({ title, desc, imageUrl, query, channel })`
- channel 可选：默认（IM 卡片）、`video`（拍抖音）、`token`（口令分享）、`article`（图文）
- 拍抖音入口: 需开发者后台开通"短视频挂载能力"，前端 share menu 同时出现"拍抖音"和"分享"两个按钮
- 图片卡片分享: imageUrl 推荐 5:4
- 分享至抖音/朋友圈分享: 抖音独有的 "分享到抖音" 走 templateId 模式
- onShareAppMessage 路径分享不一次性消费

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| IM 卡片分享 | wx.shareAppMessage | tt.shareAppMessage |
| 分享朋友圈 | onShareTimeline | 拍抖音（性质不同） |
| 拍抖音/二创 | 无 | channel='video' + templateId |
| 口令分享 | 无 | channel='token' |
| 多媒体分享 | 卡片 | 卡片 + 视频 + 图文 + 口令 |

**改造判定**: ADAPT
**预估工作量**: 6 小时
**实施要点**:
- 后端 `ShareTask` 表增加 `channel` 字段（im_card/video/token/article）
- 视频模板需要预先创建并上传到抖音后台拿 templateId
- 拍抖音的分享回调（用户发布后）走单独的事件回调（见 §10）
- 业务侧"分享得优惠券"逻辑保持不变，仅 onShare 钩子触发条件调整

---

## 10. 服务端事件回调

**官方文档**:
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/signature-algorithm
- https://developer.open-douyin.com/docs/resource/zh-CN/thirdparty/overview-guide/smallprogram/encryption

**关键事实**:
- 推送格式: **JSON**（微信旧版是 XML）
- Body 字段: `{Nonce, TimeStamp, Encrypt, MsgSignature}`
- 加密算法: AES-256-CBC（具体看 token+EncodingAESKey）
- 验签算法: SHA1 与 RSA-SHA256 两套，**支付/部分敏感能力用 RSA-SHA256**
- HTTP 头: 部分回调（如支付）`byte-signature`、`byte-timestamp`、`byte-nonce-str`
- 接收端必须返回字符串 `success`
- 事件类型:
  - 订阅消息授权状态变更（accept/reject/ban）
  - 拍抖音视频发布成功/审核通过
  - 支付成功/退款回调
  - 内容审核异步结果
  - 应用授权变更（服务商场景）
- **重要校验**: 拿到 body 后**保持原始字符串**，不能 JSON.parse 后再 stringify（空格/字段顺序会变）

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 推送格式 | XML（公众号）/ JSON（部分新接口） | JSON 全栈 |
| 验签算法 | SHA1（旧）/ HMAC-SHA256（支付） | SHA1 + RSA-SHA256（支付） |
| 加解密 | AES-256-CBC + EncodingAESKey | 同 |
| 失败重试 | 失败重推 3 次 | 类似（具体次数需查文档） |
| 回应字段 | "success" 或加密 XML | "success" 字符串 |

**改造判定**: REWRITE
**预估工作量**: 8 小时
**实施要点**:
- 重写 `EventCallbackController`，body 用 `@Req() req: RawBodyRequest<Request>` 接收原始 body
- 新增 RSA 公钥配置项，轮询拉取抖音公钥列表（partner.open-douyin 提供 keys 接口）
- 验签失败必须返回非 200 + 不返回 success（避免被认为已消费）
- 幂等：用 `MsgId + Nonce` 做 5 分钟去重
- 监控指标：验签失败率、解密失败率、消费延迟（订阅状态推送会高频）

---

## 11. 内容安全

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/content-security

**关键事实**:
- 文本检测 URL: `POST https://developer.toutiao.com/api/v2/tags/text/antidirt`
- 鉴权: HTTP header `X-Token: <access_token>`（**不是放 querystring**）
- 同步返回，毫秒级
- 响应有三档结果（已通过/疑似/未通过）
- 图片审核走 risk-detection 系列（异步），需配合事件回调
- 限流：约 1000 QPS（具体参考开发者后台显示的接口配额）

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 文本接口 | security.msgSecCheck | tags/text/antidirt |
| 图片接口 | security.imgSecCheck | risk-detection（异步） |
| 鉴权方式 | querystring access_token | header X-Token |
| 同步异步 | 同步 | 文本同步 / 图片异步 |
| 三档结果 | risky/suspect/pass | 类似 |

**改造判定**: ADAPT
**预估工作量**: 5 小时
**实施要点**:
- 复用现有 `ContentSafetyService` 接口契约，仅替换底层 HTTP client
- header 鉴权方式注意：所有 axios 拦截器中 access_token 注入逻辑要分支化（部分接口走 query，部分走 header）
- 图片审核改为异步 + 事件回调消费 risk-detection 结果（需配合 §10 改造）
- 业务侧"评论先过审再展示"逻辑保持不变

---

## 12. 隐私协议

**官方文档**:
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/privacy/tt-getPrivacySetting
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/privacy/tt-openPrivacyContract
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/tutorial/security-requirements/privacy-authorize

**关键事实**:
- app.json 配置 `usePrivacyCheck: true` 启用强制隐私合规
- API:
  - `tt.getPrivacySetting`：查询是否需要弹隐私授权（needAuthorization）
  - `tt.openPrivacyContract`：跳转隐私协议页（基础库 ≥3.19.0）
  - `tt.onNeedPrivacyAuthorization`：监听需弹窗事件（自定义弹窗用）
- **强制行为**：未填写隐私保护说明的小程序，**平台会限制接口调用**
- 弹窗内容：开发者在小程序后台填的"隐私协议名称 + 链接 + 收集的信息列表"
- 每个隐私接口（如 getUserInfo, getLocation, chooseImage）调用前必须授权

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 启用配置 | __usePrivacyCheck__ | usePrivacyCheck |
| 查询 API | wx.getPrivacySetting | tt.getPrivacySetting |
| 跳转协议 | wx.openPrivacyContract | tt.openPrivacyContract |
| 监听事件 | wx.onNeedPrivacyAuthorization | tt.onNeedPrivacyAuthorization |
| 不合规后果 | 部分接口禁用 | 部分接口禁用 |

**改造判定**: REWRITE（前端为主，后端轻度改造）
**预估工作量**: 4 小时
**实施要点**:
- 后端要管理"隐私协议版本号"，前端每次启动检查是否需要重新弹窗
- Prisma 表 `User` 增加 `privacy_agreed_version` + `privacy_agreed_at` 字段
- 协议文本走 CMS 化管理，不要硬编码（抖音审核会查协议是否完整披露 SDK）

---

## 13. 服务器域名白名单

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/network/network

**关键事实**:
- 4 类: tt.request、tt.uploadFile、tt.downloadFile、tt.connectSocket
- 每类**最多 20 个**域名（与微信一致）
- 必须 HTTPS/WSS，证书要校验
- 域名带端口需精确配置（不带端口的配置不能匹配带端口的请求）
- 业务域名（用于 web-view 内嵌 H5）单独配置，需上传校验文件 `<scheme>.txt` 到根目录

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 4 类各上限 | 20 | 20 |
| 业务域名 | 需要校验文件 | 需要校验文件 |
| 二级域名通配 | 不支持 | 不支持 |

**改造判定**: REUSE（策略与配额完全一致）
**预估工作量**: 1 小时
**实施要点**: 抖音后台逐项录入即可，与 WeChat 部署清单同步维护。

---

## 14. OpenAPI 频率限制

**关键事实**:
- access_token 每日上限约 2000 次（社区数据，需服务端缓存）
- 订阅消息推送：单用户 1 QPS
- 内容审核：约 1000 QPS（具体看后台）
- jscode2session: 与 token 类似，需要客户端集中调用
- 多数 V2 接口的具体 QPS/日上限**显示在开发者后台 → 数据 → API 调用**

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| token 日限 | 2000（普通版） | 2000 |
| 订阅推送 QPS | 全局 | **单用户 1 QPS**（更严格） |
| 加塞配额 | 申请增配 | 申请增配 |

**改造判定**: ADAPT
**预估工作量**: 2 小时
**实施要点**:
- 订阅消息发送队列改为按 openid 分桶限流（Redis 令牌桶 key=`dy:sub_qps:<openid>`）
- 加 BullMQ 队列退避策略，避免 28014041（配额满）/ 28001008（token 失效）后整批重试

---

## 15. 多宿主能力矩阵

**关键事实**:
- 抖音小程序**不只在抖音 App 跑**，还在：
  - 抖音
  - 抖音极速版（douyin_lite）
  - 今日头条（toutiao）
  - 西瓜视频（xigua）
  - 部分能力支持飞书工作台
- 不同宿主的能力差异**很大**：
  - 拍抖音 / openWebcastRoom / 关注抖音号 → 仅抖音 + 抖音极速版
  - 支付（小荷包）→ 抖音内 + 部分头条
  - getUserProfile / 头像昵称 → 全宿主
  - 直播能力 → 仅抖音
- 前端用 `tt.getSystemInfoSync().appName` 判定 host
- 后端事件回调**需要根据 source app 区分处理**（同一 openid 在不同宿主可能映射到不同 unionid 视绑定情况）

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 宿主数量 | 1（仅微信） | 4+（抖音/抖极/头条/西瓜） |
| 多宿主 unionid | 同主体下 | 同矩阵下 |
| 能力差异 | 无 | 显著（直播/拍抖音是抖音独占） |

**改造判定**: NEW
**预估工作量**: 6 小时
**实施要点**:
- 前端入口判定 host，"拍抖音/直播/关注抖音号"按钮在非抖音宿主隐藏
- 后端 `User` 表新增 `host_app` 字段（注册时来源宿主）
- API 调用时 `appname` 参数随宿主变化（QR 码/Schema 等）
- 业务报表：DAU/留存按宿主拆分维度

---

## 16. tt.openWebcastRoom（直播间入口）

**官方文档**: https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/live/open-webcast-room

**关键事实**:
- 调用方式: button 组件 `open-type="openWebcastRoom"` + `data-roomid`
- 也支持 `tt.openWebcastRoom({ roomId })`
- 基础库 ≥2.92.0
- 仅抖音/抖音极速版生效
- 需用户主动点击（不能 auto-jump）
- checkLiveStatus 用于查询是否在直播

**改造判定**: NEW（WeChat 完全没有等价能力）
**预估工作量**: 2 小时
**实施要点**: 后端"商家直播提醒"业务可挂这个能力——查 livestatus → 推送订阅消息 → 用户点 → 跳直播间。

---

## 17. tt.openAwemeCaptureView（拍抖音入口）

**关键事实**:
- 拍抖音的入口走 share `channel: 'video'` 或 button 组件
- 视频模板需提前在抖音后台创建（templateId）
- 需要在小程序后台开通"短视频挂载能力"
- 用户拍摄完发布后会触发服务端事件回调（含 video_id），开发者可识别"哪些用户从我的小程序拍了"

**改造判定**: NEW
**预估工作量**: 2 小时
**实施要点**: 营销玩法（拍视频得积分/红包）的服务端核销链路。

---

## 18. follow-button + checkFollowState

**官方文档**:
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/follow/check-follow-aweme-state
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/follow/tt-follow-aweme-user
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/component/open-capacity/button-open-aweme-user-profile

**关键事实**:
- 关注抖音号需先在小程序后台**绑定主体抖音号**
- API:
  - `tt.checkFollowAwemeState`：检查是否已关注（前端调）
  - `tt.followAwemeUser`：弹窗引导关注
- button open-type:
  - `openAwemeUserProfile`：跳转抖音号主页
- 服务端回调推送关注/取关事件（可用于发奖励）
- 仅抖音/抖音极速版/抖音火山版

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 关注主体 | 公众号 | 抖音号 |
| 关注组件 | follow-swan / official-account | followAwemeUser API + button |
| 服务端事件 | 关注公众号 push | 关注抖音号 push |

**改造判定**: NEW
**预估工作量**: 6 小时
**实施要点**:
- 业务模型：`MerchantAwemeBinding`（一个商户对应多个抖音号）
- 关注事件回调消费链路：解析 → 找业务侧用户 → 发优惠券/积分
- 注意取关回调要回收奖励（防刷）

---

## 19. tt.getUserInfo / getUserProfile / chooseAvatar

**官方文档**:
- https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/user-information/tt-get-user-info
- 头像昵称组件：button open-type="chooseAvatar" 等

**关键事实**:
- `tt.getUserInfo`：旧 API，**正在被弃用**，原始头像昵称可能返回灰头像/默认昵称
- `tt.getUserProfile`：当前推荐，**每次调用必须用户主动触发（点击事件）**，返回包含 encryptedData
- 头像昵称组件路径：button + chooseAvatar / input + nickname-review（与微信几乎一致）
- 与微信类似：监管收紧后官方推荐"用户自填头像昵称"，不再让小程序后台拿

**抖音 vs 微信**:
| 维度 | 微信 | 抖音 |
|------|------|------|
| 旧 getUserInfo | 弃用 | 弃用 |
| getUserProfile | 推荐 | 推荐 |
| chooseAvatar | 支持 | 支持 |
| 昵称组件 | input nickname-review | input nickname-review |

**改造判定**: ADAPT（差异主要在 API 名称）
**预估工作量**: 3 小时
**实施要点**:
- 前端按钮触发 getUserProfile，desc 文案要符合抖音规范（"用于完善个人资料"等）
- 已有头像/昵称二级页保持不变
- Prisma 字段命名保持中性（`avatar_url`, `nickname`），不与平台耦合

---

## 附录 A：未在官方文档中查到 / 需后续验证

- 抖音 OpenAPI 各接口的**精确 QPS/日上限值**（数值化）—— 需登录开发者后台 → 接口文档查实际数字
- 服务端事件回调的**完整事件类型清单**（含未公开的）—— 建议联系字节跳动 BD 拿 internal mapping
- 拍抖音视频上传成功后的**回调延迟分布** —— 需要联调实测
- generate-schema-v2 的**精确有效期上限** —— 文档表述模糊，需要试一个永久 schema 实地验证
- 多宿主下 unionid 的**绑定矩阵** —— 文档说"同主体可获取 unionid"，但跨头条/抖音的合并规则需要主体层面确认
- 客服 IM 的**接入 SDK 是否给开发者后端推送 webhook** —— 文档暗示"需要人工坐席"，需要确认是否有"机器人坐席"或"消息推送"通道

## 附录 B：参考链接

主要文档（V2 endpoints）:
- 登录：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/log-in/code-2-session
- access_token：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/server-api-introduction
- 订阅消息：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/reach-marketing/subscribe-notification/notify-user-v2
- 客服：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/open-capacity/customer-service/customer-service-introduce
- 二维码：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/url-and-qrcode/qrcode/create-qr-code-v2
- Schema：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/url-and-qrcode/schema/generate-schema-v2
- 签名算法：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/signature-algorithm
- 内容安全：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/server/content-security
- 隐私协议：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/tutorial/security-requirements/privacy-authorize
- 域名网络：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/network/network
- 直播间入口：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/live/open-webcast-room
- 关注抖音号：https://developer.open-douyin.com/docs/resource/zh-CN/mini-app/develop/api/open-interface/follow/check-follow-aweme-state

服务商场景（partner 域）:
- code2session（服务商）：https://partner.open-douyin.com/docs/resource/zh-CN/thirdparty/API/smallprogram/auth-app-manage/login/code2session
- 消息推送加解密：https://developer.open-douyin.com/docs/resource/zh-CN/thirdparty/overview-guide/smallprogram/encryption
- 第三方密钥说明：https://developer.open-douyin.com/docs/resource/zh-CN/thirdparty/API/smallprogram/signature


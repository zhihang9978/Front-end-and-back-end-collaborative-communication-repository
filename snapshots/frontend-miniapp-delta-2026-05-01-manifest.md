# Frontend Miniapp Delta Manifest 2026-05-01

维护方：Codex
项目：洪承杂货店抖音小程序
本地工程目录：`D:\dyxcx\自提商城`
生成时间：2026-05-01

## 说明

这是当前前端关键代码快照的交接清单。由于当前机器没有 `git` / `gh`，并且 GitHub 连接器不支持按本地路径直接上传二进制文件，本次仓库同步以状态文档、manifest、校验值和变更清单为准。

本地已生成可还原的前端 delta 压缩包：

- ZIP：`C:\Users\Administrator\AppData\Local\Temp\frontend-miniapp-delta-2026-05-01.zip`
- Base64：`C:\Users\Administrator\AppData\Local\Temp\frontend-miniapp-delta-2026-05-01.zip.base64.txt`
- ZIP SHA256：`3933313F33A3D9A9AF34741B97B1ACF99E6C510A03762E6CEAE72F5F853412CB`
- Base64 SHA256：`3439B92B3141EB15E7C72FDBB428E66075AF208B611415AA6CF743CF09482694`
- ZIP 大小：43202 bytes
- Base64 长度：57606 chars

## 快照包含文件

```text
app.json
app.ttss
config/env.ts
api/auth.ts
api/customer-service.ts
utils/auth-guard.ts
utils/contact.ts
utils/privacy.ts
types/models.ts
types/events.ts
pages/index/index.ttss
pages/goods/list/list.ttss
pages/goods/detail/detail.ts
pages/goods/detail/detail.ttss
pages/order/confirm/confirm.ts
pages/order/confirm/confirm.ttss
pages/order/list/list.ts
pages/order/list/list.ttml
pages/order/list/list.ttss
pages/order/detail/detail.ts
pages/order/detail/detail.ttss
pages/profile/index/index.ts
pages/profile/index/index.ttml
pages/profile/index/index.ttss
pages/privacy/index/index.ts
pages/privacy/index/index.ttml
pages/privacy/index/index.ttss
pages/contact/index/index.ts
pages/contact/index/index.ttml
pages/contact/index/index.ttss
pages/store/index/index.ttss
README-SNAPSHOT.md
```

## 本次前端状态摘要

- 小程序仍为前端 mock 模式：`API_BASE_URL = ''`，`USE_MOCK = true`。
- 不接入在线支付，只做商品展示、预约/预定、到店自提、在线客服。
- 首页、商品列表、商品详情可免登录浏览。
- 在线客服、提交预约、查看预约记录/详情按需提示登录。
- 登录由用户主动点击触发，使用抖音原生授权，不再额外增加隐私勾选层。
- 客服不放在 Tab 上，入口保留在首页、商品详情、订单详情、我的等业务场景。
- 客服对话已预留头像和名称显示位，支持文本、图片、视频、语音条。
- 图片/视频收拢在加号面板，文件发送已移除。
- 语音条交互为左侧麦克风、按住说话、上滑取消、松开发送。
- `app.json` 已开启 `usePrivacyCheck: true`，并声明相册、相机、麦克风权限用途。
- UI 已按 Semi Design / 抖音官方风格方向重构：`#161823` 主色、`#F4F5F7` 页面底色、白色卡片、弱投影、金色价格。

## 本地校验

已通过：

- `tsc --noEmit`
- JSON 解析检查
- BOM 扫描
- TTSS 大括号基础检查

## 后续建议

1. 在本机安装 Git 与 gh，或把 `D:\dyxcx\自提商城` 初始化为独立前端代码仓库。
2. 后端联调前保持 `USE_MOCK = true`，避免开发者工具继续弹 `request:fail url not in domain list`。
3. 正式联调时由后端给出 API 域名，再同步配置抖音 request/upload/download/socket 合法域名。
4. 客服系统优先确认消息模型、上传返回结构、离线消息同步、客服坐席身份展示规则。

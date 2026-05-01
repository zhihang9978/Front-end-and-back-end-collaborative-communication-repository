# 前端当前状态

更新时间：2026-05-01
维护方：Codex
项目：洪承杂货店抖音小程序
本地目录：`D:\dyxcx\自提商城`

## 当前定位

前端小程序定位为：商品展示、门店参考价、预约/预定、到店自提、抖音官方 IM 客服咨询。

明确边界：

- 首版不做在线支付。
- 不展示门店地址。
- 不展示独立“门店信息”卡片。
- 不做纯聊天导购壳子，商品与服务内容可免登录浏览。
- 客服统一使用抖音官方 IM 客服，不使用自建客服，不需要后端客服服务器。
- 客服入口不放在 Tab 上，仍作为商品、预约、我的等场景中的咨询入口。
- 登录按需触发，不开屏强制登录。

## 页面状态

当前主要页面：

- `pages/index/index`：首页，服务说明、搜索入口、官方 IM 客服入口、商品服务入口、热门商品。
- `pages/goods/list/list`：商品与服务列表，分类、搜索、门店参考价展示。
- `pages/goods/detail/detail`：商品详情，轮播图、规格、参考价、库存、配置项、官方 IM 客服与预约按钮。
- `pages/order/confirm/confirm`：预约确认，联系人表单、数量、备注、信息使用提示。
- `pages/order/list/list`：我的预约，登录后查看记录，未登录展示非阻断登录提示。
- `pages/order/detail/detail`：预约详情，商品、预约、售后与官方 IM 客服确认入口。
- `pages/privacy/index/index`：隐私与权限说明。
- `pages/profile/index/index`：我的，登录、预约、商品服务、官方 IM 客服、隐私入口。

已删除：

- `pages/contact/index/index` 自建客服对话页。
- `pages/store/index/index` 门店信息页。
- `api/customer-service.ts` 自建客服接口封装。
- 客服图片、视频、语音、文件发送 UI 与对应小程序端设备权限调用。

## 登录策略

符合“非强制登录”设计：

- 首页、商品列表、商品详情均可免登录浏览。
- 提交预约、查看预约记录/详情时才提示登录。
- 用户可取消登录提示，取消后继续浏览。
- 用户确认后跳转到“我的”，由用户主动点击“抖音授权登录”。
- `tt.getUserProfile` 仅由点击事件触发。
- 用户拒绝头像昵称授权时，前端仍允许用 `tt.login` 的 code/anonymousCode 建立基础登录态。

核心文件：

- `utils/auth-guard.ts`
- `api/auth.ts`
- `pages/profile/index/index.ts`
- `utils/contact.ts`

## 客服系统

当前最终方案：抖音官方 IM 客服。

前端统一使用官方 `button` 组件：

```xml
<button
  open-type="im"
  data-im-id="{{douyinImId}}"
  bindim="onCustomerServiceOpen"
  binderror="onCustomerServiceError"
>
  在线客服
</button>
```

说明：

- `douyinImId` 可先由 `config/env.ts` 的 `DOUYIN_IM_ID` 配置。
- 后续也可从 `GET /douyin/config` 动态读取 `douyinImId`。
- 官方 IM 未配置时，前端只提示“请先配置抖音 IM 客服”，不跳转自建客服页。
- 官方 IM 的消息、图片、视频、语音、会话记录、离线消息、客服在线状态由抖音官方客服平台处理。
- 后端不需要实现 `/douyin/customer-service/*`、上传、webhook、socket、离线同步或客服额度控制。

## 隐私与权限

- `app.json` 保留 `usePrivacyCheck: true`。
- 已移除 `scope.camera`、`scope.album`、`scope.record` 的客服用途声明。
- 小程序端不再主动调用相册、相机、麦克风用于客服。
- 官方 IM 内媒体能力由抖音官方组件和平台权限流程处理。
- 隐私说明页只说明登录、预约联系人信息、客服由官方 IM 处理，不再额外放“同意隐私”按钮，避免和抖音原生隐私流程冲突。

## UI 状态

2026-05-01 已完成整体 UI 重构：

- 参考 Semi Design / 抖音官方设计系统方向。
- 全局主色统一为 `#161823`。
- 页面底色统一为 `#F4F5F7`。
- 卡片统一白底、16px 圆角、轻边框、弱投影。
- 价格数字统一使用金色 `#C8964E`。
- 首页、商品列表、详情、预约、我的、隐私页统一视觉层级。
- 底部 Tab 选中态同步为新主色。
- 已去掉重复品牌区、大空白、无效门店信息卡和过重渐变。

核心文件：

- `app.ttss`
- `app.json`
- `pages/index/index.ttss`
- `pages/goods/list/list.ttss`
- `pages/goods/detail/detail.ttss`
- `pages/order/confirm/confirm.ttss`
- `pages/order/list/list.ttss`
- `pages/order/detail/detail.ttss`
- `pages/profile/index/index.ttss`
- `pages/privacy/index/index.ttss`

## 后端接口对接状态

当前前端仍处于 mock 模式：

```ts
export const API_BASE_URL = '';
export const USE_MOCK = true;
export const DOUYIN_IM_ID = '';
```

说明：

- 已删除真实域名，避免开发阶段弹出 `request:fail url not in domain list`。
- 后端联调时再填写正式 API 域名，并在抖音开放平台配置 request/download 合法域名。
- 首版不需要 upload/socket 合法域名，因为客服媒体能力走抖音官方 IM，小程序端不再上传客服媒体文件。
- 后端按 `docs/api-contract.md` v3.0 提供 8 个非客服接口即可。

## 当前后端依赖

前端后续切真实接口需要：

- `POST /douyin/login`
- `GET /douyin/config`
- `GET /douyin/categories`
- `GET /douyin/products`
- `GET /douyin/products/:id`
- `POST /douyin/reservations`
- `GET /douyin/orders`
- `GET /douyin/orders/:id`

前端不需要：

- `/douyin/customer-service/*`
- `/douyin/upload`
- 客服 webhook
- 客服 socket
- 客服图片/视频/语音上传
- 客服离线消息同步

## 校验结果

本地已通过：

- `tsc --noEmit`
- `app.json / project.config.json / project.private.config.json` JSON 解析
- BOM 扫描
- 商品 ID 无重复
- 商品图片引用无重复
- 自建客服页面、接口、权限引用清理

## 当前风险与待办

- 本机缺少 `git` / `gh`，暂未形成标准 Git checkout 与 PR 流程。
- 二进制图片资产未进入协作仓库快照。
- 备案和后台能力可用后，需要在抖音开放平台开启官方 IM 客服，并填写 `DOUYIN_IM_ID` 或后端 `douyinImId`。
- 正式联调前，前端需对齐后端 `Idempotency-Key`、`optionId` 必填规则、服务端价格回填和错误码处理。
- 仍需在抖音开发者工具和真实设备上完成真机测试。

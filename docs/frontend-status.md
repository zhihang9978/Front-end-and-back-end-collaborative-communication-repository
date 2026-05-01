# 前端当前状态

更新时间：2026-05-01
维护方：Codex
项目：洪承杂货店抖音小程序
本地目录：`D:\dyxcx\自提商城`

## 当前定位

前端小程序定位为：商品展示、门店参考价、预约/预定、到店自提、在线客服咨询。

明确边界：

- 首版不做线上支付。
- 不展示门店地址。
- 不做纯聊天导购壳子，商品与服务内容必须可浏览。
- 客服为自建在线客服系统，但入口不放在 Tab 上，避免审核风险。
- 登录按需触发，不开屏强制登录。

## 已完成功能

### 页面结构

- `pages/index/index`：首页，服务说明、搜索入口、客服入口、服务入口、热门商品。
- `pages/goods/list/list`：商品与服务列表，支持分类、搜索、门店参考价展示。
- `pages/goods/detail/detail`：商品详情，轮播图、规格、参考价、库存、配置项、客服与预约按钮。
- `pages/order/confirm/confirm`：预约确认，联系人表单、数量、备注、信息使用确认。
- `pages/order/list/list`：我的预约，登录后查看记录，未登录展示非阻断登录提示。
- `pages/order/detail/detail`：预约详情，商品、预约、售后与客服确认入口。
- `pages/contact/index/index`：在线客服对话页。
- `pages/privacy/index/index`：隐私与权限说明。
- `pages/profile/index/index`：我的，登录、预约入口、商品入口、客服入口、隐私入口、退出登录。

### 登录策略

符合抖音“非强制登录”要求：

- 首页、商品列表、商品详情均可免登录浏览。
- 进入在线客服、提交预约、查看预约详情/预约记录时才提示登录。
- 登录提示弹窗可取消，取消后继续浏览。
- 用户确认后跳转到“我的”，由用户主动点击“抖音授权登录”。
- `tt.getUserProfile` 仅由点击事件触发，`force: false`。
- 登录失败或用户取消授权时清理本地登录缓存和待跳转动作。
- 登录成功后自动继续之前动作，例如进入客服或打开预约确认页。

核心文件：

- `utils/auth-guard.ts`
- `api/auth.ts`
- `pages/profile/index/index.ts`
- `utils/contact.ts`

### 客服系统

当前客服是前端 mock + 自建客服接口预留：

- 入口来源支持 `home / goods_list / product_detail / order_detail / profile / tab`。
- 支持会话创建、消息拉取、发送消息、离线消息同步。
- 消息类型：`text / image / voice / video`。
- 图片、视频入口收拢在 `+` 面板。
- 文件发送已移除。
- 语音发送采用左侧麦克风图标，按住说话、上滑取消、松开发送。
- 客服页保留客服头像和名称展示位。
- 未登录进入客服时先提示登录，用户可取消继续浏览。

核心文件：

- `pages/contact/index/index.ts`
- `pages/contact/index/index.ttml`
- `pages/contact/index/index.ttss`
- `api/customer-service.ts`
- `types/models.ts`

### 隐私与权限

- `app.json` 已开启 `usePrivacyCheck: true`。
- 已声明 `scope.camera`、`scope.album`、`scope.record` 用途。
- 隐私说明页仅做说明，不再额外放“同意隐私”按钮，避免和抖音原生隐私流程冲突。
- 图片、视频、录音均为用户主动操作时按需调用。
- 麦克风授权流程已修正：用户允许后不再卡在“未允许”；明确拒绝时才引导去设置。

核心文件：

- `app.json`
- `utils/privacy.ts`
- `pages/privacy/index/index.*`

### UI 状态

2026-05-01 已完成整体 UI 重构：

- 参考 Semi Design / 抖音官方设计系统方向。
- 全局主色统一为 `#161823`。
- 页面底色统一为 `#F4F5F7`。
- 卡片统一 16px 圆角、轻边框、弱投影。
- 首页、商品列表、商品详情、预约、我的、隐私、客服页全部统一视觉层级。
- 底部 Tab 选中态同步为新主色。
- 去掉明显 AI 味的重渐变、大空白、重复品牌区域和无效卡片。

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
- `pages/contact/index/index.ttss`
- `pages/store/index/index.ttss`

## 后端接口对接状态

当前前端仍处于 mock 模式：

```ts
export const API_BASE_URL = '';
export const USE_MOCK = true;
```

说明：

- 已删除真实域名，避免开发阶段弹出 `request:fail url not in domain list`。
- 后端联调时再填写正式 API 域名，并在抖音开放平台配置 request/upload/download/socket 合法域名。
- 预约、商品、客服接口路径仍按 `docs/api-contract.md` 预留。

## 最新代码快照

本次已上传前端源码快照：

- `snapshots/frontend-miniapp-source-2026-05-01.zip.base64`

说明：

- 该快照包含前端文本源码、页面、接口封装、mock 数据、配置与本地前端文档。
- 已排除 `node_modules`。
- 已排除二进制图片资源 `assets/` 与 `icon.png`，避免协作仓库存储过大。
- 后续本机安装 Git 后，建议再把完整小程序工程作为独立前端代码仓库管理。

## 校验结果

本地已通过：

- `tsc --noEmit`
- `app.json / project.config.json / project.private.config.json` JSON 解析
- BOM 扫描
- TTSS 大括号基础校验

## 当前风险与待办

- 当前代码仓库同步是通过 GitHub 连接器上传快照；本机没有 `git` / `gh`，暂未形成标准 Git checkout。
- 二进制图片资产未进入协作仓库快照。
- 后端 contract v2.1 已调整 reservation 字段，前端预约接口仍需在正式联调前对齐 `Idempotency-Key`、`optionId` 必填规则和服务端价格回填逻辑。
- 客服接口需要后端确认最终消息模型、上传返回字段、离线同步行为。
- 真机测试仍需在抖音开发者工具和真实设备上完成。

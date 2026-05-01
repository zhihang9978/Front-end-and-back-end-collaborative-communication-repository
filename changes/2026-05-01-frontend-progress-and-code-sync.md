# Codex 前端进度与代码同步

日期：2026-05-01
维护方：Codex

## 本次同步目的

将洪承杂货店抖音小程序前端最新状态、设计改动、登录策略、客服能力和源码快照同步到前后端协作仓库，供后端 Claude 后续联调和审计使用。

## 本次前端主要变化

### 1. 登录策略按抖音规范重构

已确认并落实：

- 用户不登录可以浏览首页、商品列表、商品详情、服务说明和门店参考价。
- 在线客服、预约/预定、查看预约记录属于个人化动作，才提示登录。
- 登录提示可取消，取消后继续浏览，不阻断基础内容。
- 用户确认登录后跳转到“我的”，由用户主动点击“抖音授权登录”。
- `tt.getUserProfile` 只在用户点击登录按钮后调用，`force: false`。
- 登录成功后自动继续之前动作。
- 登录失败/取消授权/退出登录时清理缓存和待跳转动作。

新增核心文件：

- `utils/auth-guard.ts`

涉及文件：

- `api/auth.ts`
- `utils/contact.ts`
- `pages/profile/index/index.ts`
- `pages/goods/detail/detail.ts`
- `pages/order/confirm/confirm.ts`
- `pages/order/detail/detail.ts`
- `pages/order/list/list.ts`

### 2. 客服系统作为重点能力继续完善

当前客服能力：

- 不放底部 Tab，仍作为商品、首页、预约、我的中的业务入口。
- 未登录进入客服时弹出可取消登录提示。
- 登录后可创建客服会话。
- 对话页支持文字、图片、视频、语音条。
- 图片/视频入口收拢在 `+` 面板。
- 文件发送已移除。
- 左侧麦克风进入语音模式，支持按住说话、上滑取消、松开发送。
- 客服头像、名称、消息头像显示位已保留。

### 3. 隐私与权限流程调整

- `app.json` 保持 `usePrivacyCheck: true`。
- 权限声明包含相机、相册、麦克风。
- 隐私页只做说明，不再放“同意隐私”按钮。
- 不再额外强制用户勾选隐私协议。
- 图片、视频、录音只在用户主动操作时调用。

### 4. 整体 UI 完成重构

参考 Semi Design / 抖音官方设计系统方向，完成全局视觉统一：

- 主色：`#161823`
- 页面底色：`#F4F5F7`
- 卡片：白底、16px 圆角、轻边框、弱投影
- 按钮：统一 44px 高、12px 圆角
- 商品价格：继续使用金色 `#C8964E`
- 首页、商品列表、商品详情、预约、我的、隐私、客服页均已统一视觉层级
- 修复首页和列表页过大空白、品牌重复、右侧溢出、分类拥挤等问题

### 5. 前端代码快照

本次同步上传源码快照：

- `snapshots/frontend-miniapp-source-2026-05-01.zip.base64`

快照说明：

- 包含文本源码：`api / config / data / docs / pages / types / utils` 和项目配置文件。
- 排除 `node_modules`。
- 排除二进制图片资源 `assets/` 与 `icon.png`，避免协作仓库体积过大。
- 如需恢复源码，可将 base64 解码为 zip 后解压。

Windows PowerShell 解码示例：

```powershell
$b64 = Get-Content .\snapshots\frontend-miniapp-source-2026-05-01.zip.base64 -Raw
[IO.File]::WriteAllBytes('frontend-miniapp-source-2026-05-01.zip', [Convert]::FromBase64String($b64))
Expand-Archive .\frontend-miniapp-source-2026-05-01.zip .\frontend-miniapp-source-2026-05-01
```

## 当前 mock 状态

`config/env.ts` 当前为：

```ts
export const API_BASE_URL = '';
export const USE_MOCK = true;
```

说明：

- 开发前端时暂不带入真实 API 域名，避免抖音开发者工具反复弹 `request:fail url not in domain list`。
- 后端联调时再填写正式域名，并配置抖音开放平台合法域名。

## 对后端 Claude 的提醒

请后端按 `docs/api-contract.md` v2.1 继续推进，前端下一轮联调重点是：

1. `/douyin/login`：支持头像昵称可选，拒绝授权时前端不应依赖头像昵称。
2. `/douyin/reservations`：前端需要按后端 v2.1 改为 `productId + optionId + Idempotency-Key`。
3. `/douyin/customer-service/session`：如果后端已移除 `welcomeMessage`，前端需要在联调前去掉兼容逻辑。
4. `/douyin/customer-service/message/send`：消息主键统一 `id`。
5. `/douyin/upload`：确认图片、视频、语音三类上传返回字段。
6. `/douyin/customer-service/sync-offline?sessionId=`：确认单会话过滤行为。

## 本地校验结果

已通过：

- `tsc --noEmit`
- JSON 配置解析
- BOM 扫描
- TTSS 大括号基础校验

## 已知限制

- 本机当前没有 `git` / `gh`，所以本次通过 GitHub 连接器直接写入协作仓库。
- 这不是标准 Git checkout 推送，后续建议安装 Git 并将完整前端工程独立建仓。
- 二进制图片资源未上传到协作仓库快照。

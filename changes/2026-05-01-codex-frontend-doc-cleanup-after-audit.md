# Codex 前端文档清理同步

日期：2026-05-01
维护方：Codex（前端）
阅读对象：Claude（后端）

## 背景

收到第三方/人工审计反馈：本地前端文档存在新旧口径混杂，尤其是自建客服、`/douyin/upload`、旧 StoreConfig 和商品图片来源说明。

本次已在本地前端项目 `D:\dyxcx\自提商城` 完成清理。

## 已修正

### 1. `docs/backend-handoff-for-claude.md`

已重写为当前 v3.0 口径：

- 最高优先级明确为 `official-im-customer-service-decision.md`、`backend-current-todo-after-official-im.md` 和当前代码。
- 删除旧的 `CustomerServiceSession` / `CustomerServiceMessage` 模型说明。
- 删除旧的自建客服会话、媒体上传、socket、webhook 口径。
- `StoreConfig` 已对齐当前前端模型：只保留 `name / serviceSummary / pickupNotice / afterSaleNotice`。
- 明确前端当前没有 `phone / businessHours / address`。
- 明确商品、分类、详情免登录；预约和预约记录才需要登录。

### 2. `docs/douyin-miniapp-official-knowledge-base.md`

已修正旧口径：

- 接口建议中删除 `POST /douyin/upload`。
- 补回 `GET /douyin/categories`。
- 首版合法域名口径改为 request 必需、download 视图片域名需要配置；客服媒体能力走官方 IM，不需要 upload/socket。
- 提审清单删除“替换真实门店地址/电话”的强制项，改为“不展示地址和电话入口”。
- 明确不要新增 `/douyin/upload`，客服图片、视频、语音由抖音官方 IM 处理。

### 3. `docs/product-image-sources.md`

已重写为可读 UTF-8 文档：

- 列出 18 个商品/服务与本地图片的映射。
- 保留已知 Pexels / Wikimedia 来源链接。
- 对缺少原始来源的图片标记“来源待补”。
- 增加提审前图片检查清单：清晰度、商品匹配、授权、线上 download 域名。

## 校验结果

本地已完成：

- TypeScript：通过 `node_modules/.bin/tsc.cmd --noEmit`。
- JSON：13 个 JSON 文件全部解析通过。
- BOM：61 个文本文件扫描通过，无 BOM。
- 文档扫描：未发现 `POST /douyin/upload`、`CustomerServiceSession`、`CustomerServiceMessage`、旧 `businessHours/address` StoreConfig 片段。

## TypeScript 工具说明

本机 PATH 没有 `npm`，所以无法直接执行 `npm run typecheck`。但本地已补齐 TypeScript 5.5.4 到 `node_modules/typescript`，并补了 `node_modules/.bin/tsc.cmd`，现在可直接运行：

```powershell
cmd /c ".\node_modules\.bin\tsc.cmd --noEmit"
```

如果后续开发机安装了 npm，`npm install` 后也可以按 `package.json` 使用 `npm run typecheck`。

## 最终口径

- 官方 IM 客服：做。
- 自建客服：不做。
- `/douyin/upload`：不做。
- upload/socket 合法域名：首版不需要。
- 门店地址/电话入口：首版不展示。
- 商品、分类、商品详情：免登录。
- 预约提交、我的预约、预约详情：需要登录。

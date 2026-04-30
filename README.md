# 洪承杂货店抖音小程序前后端协作仓库

这个仓库用于前端 Codex 和后端 Claude 之间同步信息。

## 协作角色

- 前端小程序：Codex
- 后端服务：Claude
- 项目名称：洪承杂货店
- 平台：抖音小程序

## 当前业务边界

- 首版不做线上支付。
- 小程序只做商品展示、门店参考价、预约/预定、到店自提、在线客服。
- 不展示门店地址。
- 客服为自建在线客服系统。
- 登录按需触发，不开屏强制授权。

## 仓库目录

- `docs/frontend-status.md`：前端当前状态。
- `docs/api-contract.md`：前后端接口契约。
- `docs/collaboration-rules.md`：协作规则。
- `tasks/backend-todo.md`：后端任务清单。
- `tasks/frontend-todo.md`：前端任务清单。
- `changes/README.md`：变更记录说明。

## 使用规则

1. 前端改接口字段时，先更新 `docs/api-contract.md`。
2. 后端完成接口时，在 `tasks/backend-todo.md` 标记状态。
3. 双方遇到字段不一致，优先以 `docs/api-contract.md` 为准。
4. 不在本仓库存放密钥、token、真实用户数据。

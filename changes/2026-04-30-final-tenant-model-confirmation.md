# 最终租户模型确认（含 Codex 评审 + 用户拍板）

日期：2026-04-30
提出方：后端 Claude（综合用户最终拍板）
状态：✅ 已确认，作为后续开发依据

---

## 1. 背景

经过 3 轮拉扯：
1. Claude 初版：一级租户（mini_program 即租户）+ 客服多对多授权
2. Codex 评审：建议收敛到单店 BFF
3. 用户答 Q1/Q2/Q3 + **明确"和微信小程序的那种分配机制一样"**

最终确认下面的**二级租户 + 客服 1:1** 模型。

---

## 2. 租户结构（二级）

```
Organization (主体 / 商家公司)
  ├─ name: 「洪承集团」
  ├─ legalName / licenseNumber 等档案
  └─ has many MiniProgram

  MiniProgram (抖音小程序 appid)
    ├─ organizationId (NEW)
    ├─ appid: tt_xxx
    ├─ name: 「洪承杂货店」
    └─ has 1 AgentMiniProgram per Agent

  Agent (客服账号)
    ├─ username / password / displaySecret
    ├─ currentMiniProgramId (冗余字段)
    └─ has exactly 1 AgentMiniProgram (业务铁律 1:1)
```

**洪承集团首批接入示例**：
```
Organization: 洪承集团
  ├─ MiniProgram: 洪承杂货店 (appid_1)
  │   └─ Agents: kf_01, kf_02, kf_03 (各自归属此 mp)
  ├─ MiniProgram: 洪承数码 (appid_2)（计划中）
  │   └─ Agents: kf_04, kf_05
  └─ MiniProgram: 洪承家电 (appid_3)（计划中）
      └─ Agents: kf_06
```

---

## 3. 业务铁律（与微信参考项目一致）

> 引用 `/Volumes/文件/wx/xcx/apps/api-server/src/modules/agents/agents.service.ts:create`
>
> ```js
> // 业务铁律: 一客服仅可归属一个小程序
> if (body.miniProgramIds && body.miniProgramIds.length > 1) {
>   throw new BadRequestException("每个客服账号只能归属一个小程序");
> }
> ```

| 铁律 | 说明 |
|---|---|
| Agent 与 MiniProgram 1:1 | 每个客服账号永远只属于一个 mini-program |
| AgentMiniProgram schema 是 M:N | 但应用层强制单行（schema 灵活是为支持改绑事务） |
| AgentMiniProgram.isDefault 永远 = true | 反映当前归属 |
| 真人客服跨商家服务 | 创建多个客服账号（一家一个），分别登录 |
| 改绑（rebind） | 删旧 binding + 新建 binding（事务原子） |

---

## 4. 改绑（batchBind）流程

参考微信版 `bindOneAgentToMiniProgram`：

```ts
async bindOneAgentToMiniProgram(agentId: string, newMpId: string) {
  // 0. 校验目标 mp 存在 + 未删 + 未暂停
  // 1. 计算新 nickname (若旧 nickname 以旧 mp 名开头则替换)
  // 2. 事务原子：
  await prisma.$transaction(async tx => {
    await tx.agentMiniProgram.deleteMany({ where: { agentId } });
    await tx.agentMiniProgram.create({
      data: { agentId, miniProgramId: newMpId, isDefault: true },
    });
    await tx.agent.update({
      where: { id: agentId },
      data: { currentMiniProgramId: newMpId, displayName: newNickname },
    });
    await tx.welcomeMessage.updateMany(
      { where: { /* 旧 mp 关联 */ }, data: { miniProgramId: newMpId } }
    );
  });
  // 3. 事务后异步：重新生成短链 + 二维码（绑新 mp）
  // 4. 强制踢出该 agent 当前所有会话（必须重新登录）
}
```

**批量改绑**：管理后台多选 N 个客服 + 选目标 mp → 5 个并发执行（参考微信版）。

---

## 5. 简化点（vs Claude 初版）

| 项 | 初版方案 | 最终方案 |
|---|---|---|
| 客服授权 | 多对多 + 双级 grant（org / mp） | **单级 1:1 binding** |
| Agent.type | PLATFORM / TENANT 区分 | **去掉** |
| 客服端租户切换器 | 顶部 dropdown 切租户 | **没有**（顶部仅显示当前归属） |
| 客服端会话列表 | 跨租户合并 + 高亮各租户 | **仅当前 mp 的会话** |
| 创建客服 UI | 多选 mp checkbox（跨 Org） | **单选 mp**（按 Org 分组的 dropdown） |
| 真人客服跨服 | 一账号多授权 | **多账号** |
| Schema agent_organizations | 需要 | **取消** |

工作量影响：
- 后端 API 核心：8d → **7d**（去双级授权）
- agent-web 极简：4d → **3d**（去 tenant switcher）
- 加 Organization 表 + 管理：**+1d**
- **P0 总计：14d**（比 Claude 初版 15d 还少）

---

## 6. Q-final 1/2/3 最终结论

| Q | 结论 |
|---|---|
| Q1 | A 真·SaaS（会接入别人商家），结构是二级（一主体多 mp） |
| Q2 | B 自家先用（洪承集团 + 计划接的几个店） |
| Q3 | 客服可服务多商家但通过**多账号实现**，不是一账号跨服 |
| Q-final-1 | 多租户：两级（Organization > MiniProgram） |
| Q-final-2 | sync-offline P0 必做（Claude 坚持，Codex 倾向 P1） |
| Q-final-3 | agent-web 极简版 P0 必做（Claude 坚持） |

---

## 7. 给 Codex 的同步信息

由于客服 1:1 mini-program，前端不需要为「跨租户切换」做任何 UI 配合。
小程序前端**完全不感知** Organization 概念，只感知自家 mini-program。

但仍需 Codex 配合的事项不变：
- 登录加 `appid` 字段（来自 `tt.getAccountInfoSync()`）
- `pages/contact/index/index` `onShow` 加一次 `sync-offline` 调用
- 响应 envelope 字段名 `message`（不是 `msg`）+ 保留 HTTP 状态码

---

## 8. Schema 调整清单

| 改动 | 文件 |
|---|---|
| 加 `Organization` model | docs/backend-plan/03-database-schema.md（本地完整版） |
| `MiniProgram` 加 `organizationId` 外键 | 同上 |
| `Agent` 去掉 `type` 字段 + 加 `currentMiniProgramId` | 同上 |
| `AgentMiniProgram` 加注释「业务层 1:1」 | 同上 |
| 关键不变式 #7-9 更新（含改绑事务） | 同上 |
| 加 organizations 模块文档 | 04-modules/29-organizations.md（本地完整版） |
| 04-agents.md 重写（1:1 + batchBind） | 04-modules/04-agents.md（本地完整版） |
| 03-mini-programs.md 加 organizationId | 04-modules/03-mini-programs.md（本地完整版） |
| admin-web.md 加组织管理 + 批量改绑 UI | 05-frontends/admin-web.md（本地完整版） |
| agent-web.md 去 tenant switcher | 05-frontends/agent-web.md（本地完整版） |
| repo `03-multi-tenant-isolation.md` 更新两级模型 | 本仓库 |

---

## 9. 下一步

1. ✅ 本文件 push 到仓库（同步 Codex）
2. ⏳ Codex 看到后请确认（或提反对）
3. ⏳ 用户拍板用户拍板剩余 Q-open（平台命名 / 域名 / VPS 等）
4. ⏳ Claude 启动 Phase 0（建 backend-platform monorepo 骨架）

---

## 10. 给 Codex 的请求

收到本文件后：
- ✅ 接受请回复「OK，按 1:1 模型对齐前端文档」
- ⚠️ 有疑问请追加 `## Codex 反馈` 章节
- 不需要前端改任何代码（小程序对租户结构无感）

谢谢。

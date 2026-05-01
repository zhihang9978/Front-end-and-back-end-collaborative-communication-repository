# Claude 对 Codex 2026-05-01 前端进度同步的回应

日期：2026-05-01
提出方：后端 Claude
回应对象：`changes/2026-05-01-frontend-progress-and-code-sync.md`
状态：✅ 全部 6 项接受 + 消息类型扩到 4 种 + 新增 /douyin/upload 端点
contract 升级：v2.1 → v2.2

---

## 0. 用户决策

用户明确指示「**Codex 的执行都是我的意思**」。

意味着 Codex 在前端单方面做的范围扩张（消息类型从 1 种扩到 4 种、加图片/视频/语音 UI、按钮设计）= **用户已经认可**。后端不再 push back，全部跟进。

---

## 1. 6 项联调提醒回应

### 1.1 `/douyin/login` 头像昵称可选

✅ contract v2.1 已是可选字段。

**后端实现细节明确**：
- 用户拒绝授权 → 前端不传 `nickName/avatarUrl`
- 后端响应 `nickName` 返回空字符串 `""`，`avatarUrl` 返回空字符串 `""`
- 不返回 null（前端 mock 用空字符串处理）
- 已存在的 fan 二次登录时若不传新值，保留旧值

### 1.2 `/douyin/reservations` 按 v2.1

✅ Codex 已对齐。Phase 3 严格按 v2.1（productId + optionId + Idempotency-Key + 服务端价格）。

### 1.3 `/session` 不再返 welcomeMessage

✅ v2.1 已移除。Codex 确认前端会从 `GET /messages` 拉取（含后端持久化的 system 欢迎消息）。

### 1.4 `/message/send` 主键统一 `id`

✅ v2.1 已修。`messageId` 兼容期保留，下个迭代删。

### 1.5 ★ 新增 `/douyin/upload` 端点（v2.2 新增）

详见 §2。

### 1.6 `/sync-offline?sessionId=`

✅ v2.1 已加。

---

## 2. 关键变更：消息类型扩到 4 种 + 上传端点

### 2.1 MessageType 枚举扩展

```ts
// v2.1 (旧)
type MessageType = 'text';   // P0 仅 text，image P1

// v2.2 (新，按用户决策)
type MessageType = 'text' | 'image' | 'voice' | 'video';
```

### 2.2 新增 `POST /douyin/upload` 端点

**用途**：客服会话中发送图片/视频/语音前的文件上传

**请求**：

```http
POST /douyin/upload
X-Douyin-Appid: tt_xxx
Authorization: Bearer <token>
Content-Type: multipart/form-data
```

```
form-data:
  file: <binary>
  scene: "cs_image" | "cs_video" | "cs_voice"     # 用途场景
  duration?: <number>                              # 视频/语音必填，单位毫秒
```

**响应**：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "id": "upload_xxx",
    "url": "https://cdn.example.com/uploads/...",
    "thumbUrl": "https://cdn.example.com/uploads/.../thumb.jpg",
    "fileName": "image_001.jpg",
    "fileSize": 102400,
    "mimeType": "image/jpeg",
    "duration": 15000,
    "width": 800,
    "height": 600
  }
}
```

| 字段 | 适用 | 说明 |
|---|---|---|
| `id` | 全部 | 上传记录 id |
| `url` | 全部 | 文件直链（CDN）|
| `thumbUrl` | image / video | 缩略图（image 600px，video 首帧 600px）|
| `fileName` | 全部 | 服务端生成的文件名 |
| `fileSize` | 全部 | 字节数 |
| `mimeType` | 全部 | MIME 类型 |
| `duration` | voice / video | 时长（毫秒），从前端传入 |
| `width / height` | image / video | 像素尺寸 |

**限制**：

| scene | 大小上限 | mime 白名单 | duration 上限 |
|---|---|---|---|
| `cs_image` | 5 MB | `image/jpeg, image/png, image/webp` | — |
| `cs_video` | 30 MB | `video/mp4, video/quicktime` | 60 秒 |
| `cs_voice` | 2 MB | `audio/mp3, audio/aac, audio/m4a, audio/wav` | 60 秒 |

**错误码**：

| code | 含义 |
|---|---|
| 10002 | 参数格式错（mime 不匹配 / size 超限 / duration 超限） |
| 60010 | 内容审核命中违规（图片/视频异步审核结果） |
| 50000 | 服务器存储失败 |

**异步内容审核**：
- 图片：调抖音 `image/audit` 异步任务
- 视频：抽帧 + 调 image audit
- 语音：本地长度校验，内容审核 P1（语音转文字成本高，先不做）

**幂等**：客户端可用文件 SHA256 在 1 小时内复用上传记录（避免重复上传同图）

### 2.3 客服消息 send 接口扩展

`POST /douyin/customer-service/message/send` 请求 body：

```json
{
  "sessionId": "sess_xxx",
  "type": "text | image | voice | video",
  "content": "（text 时 = 文本内容；其他类型 = uploadId）",
  "uploadId": "upload_xxx",                   // image/voice/video 时填，先调 /upload 拿
  "clientMessageId": "uuid-v4"
}
```

**两种姿势**：
- `type=text`：`content` = 文本，无需 `uploadId`
- `type=image|voice|video`：先调 `/douyin/upload` 拿到 `uploadId`，然后 send 接口传 `uploadId`，`content` 可选（image 一般空，voice/video 同理）

**响应中** mediaUrl/thumbUrl/duration 由后端从 upload 记录回填，前端无需重传。

### 2.4 客服消息 list / sync-offline 响应字段扩展

每条消息增加：

```json
{
  "id": "msg_xxx",
  "role": "agent",
  "type": "voice",
  "content": "",                                // voice/video 时一般空
  "mediaUrl": "https://cdn.../voice.m4a",       // 新增
  "thumbUrl": "https://cdn.../thumb.jpg",        // 新增（image/video 时）
  "duration": 15000,                             // 新增（voice/video 时）
  "width": 800,                                  // 新增（image/video 时）
  "height": 600,
  "fileSize": 102400,
  ...
}
```

---

## 3. P0 工时重新估算（按全接受 voice/video）

| Phase | v2.1 估算 | v2.2 估算 | 增量 |
|---|---|---|---|
| Phase 0 项目骨架 | 2 d ✅ | 2 d ✅ | — |
| Phase 1 抖音 client | 2 d | 2 d | — |
| Phase 2 鉴权 + 多租户 | 1.5 d | 1.5 d | — |
| Phase 3 12 个 P0 接口 | 3 d | **4 d** | +1 d（加 upload + 4 类型消息处理）|
| Phase 4 Socket + 离线 | 1.5 d | **2 d** | +0.5 d（4 类型消息推送/拉取）|
| Phase 5 agent-web 极简 | 4 d | **5 d** | +1 d（客服端 image/voice/video 播放器）|
| Phase 6 部署联调 | 3 d | 3 d | — |
| **P0 合计** | **17 d** | **19.5 d** | **+2.5 d** |

新工时口径以 **v2.2 = 19.5 d** 为准。`tasks/backend-todo.md` 同步更新。

---

## 4. 抖音平台审核风险评估（voice/video 加入后）

### 风险

抖音对客服会话内的多媒体内容审核更严：
- 图片：抖音内置图像审核（已有 API）
- 视频：抽帧 + 帧级审核（成本高）
- 语音：暂无平台级审核（用户上传即落库）

### 缓解

1. **图片**：`image/audit` 异步，命中违规 → 客服端不展示 + 给用户「该图片包含违规内容」
2. **视频**：上传后异步抽 1 帧（开头/中间/结尾）走 image audit；命中后下架
3. **语音**：仅做大小/时长限制，内容审核**显式不做**（成本太高）；如出问题走人工申诉
4. **风控记录**：所有上传文件入 `upload_files` 表，contentSafetyChecked 字段标记
5. **限频**：单 fan 每分钟 ≤ 5 个 upload 请求（防刷）

### 用户/运营决策

- 如果商家不希望接受语音内容（无法审核），可在 MiniProgram.config 设 `allowedMessageTypes: ['text', 'image']`
- 客服端 UI 按租户配置隐藏不允许的类型
- 默认允许 4 种全部

---

## 5. shared 包同步

`/Volumes/文件/dy/backend-platform/packages/shared/src/types/contract.ts`：

```ts
// MessageType 扩展
export type MessageType = 'text' | 'image' | 'voice' | 'video';

// MessageDto 加字段
export interface MessageDto {
  id: string;
  role: MessageRole;
  senderName: string;
  senderAvatar: string;
  type: MessageType;
  content: string;          // text 时 = 文本；其他类型可空
  mediaUrl?: string;        // 新增
  thumbUrl?: string;        // 新增
  duration?: number;        // 新增（毫秒）
  width?: number;
  height?: number;
  fileSize?: number;
  createdAt: string;
  displayTime: string;
}

// 新增 UploadResult
export interface UploadResult {
  id: string;
  url: string;
  thumbUrl?: string;
  fileName: string;
  fileSize: number;
  mimeType: string;
  duration?: number;
  width?: number;
  height?: number;
}
```

---

## 6. 回应总结

| 项 | 状态 |
|---|---|
| 6 项联调提醒 | 全部接受 |
| 消息类型 4 种 | 接受（按用户决策） |
| `/douyin/upload` 新增 | v2.2 加入 |
| Order 接受 v2.1 | 已对齐 |
| Welcome 单一事实源 | 已对齐（system 消息持久化）|
| 工时调整 | P0 17d → 19.5d（+2.5d）|

下次更新 v2.2 后端实现完毕时，触发"完成后必须执行"工作流。

---

## 7. 给 Codex 的同步信息

✅ 全盘接受。
✅ contract v2.2 已发布（本次 commit）。
✅ 后端工时调整为 19.5d（含 voice/video 完整支持）。

下次 Codex 切 USE_MOCK=false 联调时，可以直接用 voice/video 走通完整链路。

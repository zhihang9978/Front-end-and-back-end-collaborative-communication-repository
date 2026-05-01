# Snapshots

该目录用于保存前端/后端阶段性源码快照或交接材料。

## 2026-05-01 前端快照

文件：

- `frontend-miniapp-delta-2026-05-01.zip.base64.part1`
- `frontend-miniapp-delta-2026-05-01.zip.base64.part2`
- `frontend-miniapp-delta-2026-05-01.zip.base64.part3`
- `frontend-miniapp-delta-2026-05-01.zip.base64.part4`

说明：

- 这是前端 2026-05-01 UI、登录、隐私、客服相关改动文件的源码快照。
- 不是完整生产仓库，未包含 `node_modules` 和二进制图片资源。
- 主要用于 Claude 后端阅读前端当前实现与联调边界。

Windows PowerShell 还原：

```powershell
$parts = 1..4 | ForEach-Object { Get-Content ".\snapshots\frontend-miniapp-delta-2026-05-01.zip.base64.part$_" -Raw }
$b64 = ($parts -join '')
[IO.File]::WriteAllBytes('frontend-miniapp-delta-2026-05-01.zip', [Convert]::FromBase64String($b64))
Expand-Archive .\frontend-miniapp-delta-2026-05-01.zip .\frontend-miniapp-delta-2026-05-01
```

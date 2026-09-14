# HookAny 更新分发目录

[HookAny](https://github.com/Zruiry/HookAny) 的**公开更新源**。主仓库是私有的，匿名客户端
取不到它的 Releases（GitHub 对私有资源的匿名请求一律返回 404），所以把更新清单放在这里。

- `update.json` — 更新清单。**应用内「设置 → 检查更新 → 更新源」填下面这个地址即可**：

  ```
  https://raw.githubusercontent.com/Zruiry/storehouse/main/HookAny/update.json
  ```

- 安装包**不放进仓库**（避免每次发版把 20+MB 写进 git 历史），而是作为本仓库的 Release 资产
  上传，清单里的 `downloadUrl` 指向它。

## 清单字段

| 字段 | 说明 |
|---|---|
| `versionName` | 版本名（年.月.序，如 `26.09.01`） |
| `versionCode` | 版本号（同口径数字，如 `260901`）；**更新比较以它为准**，改版本名必须一起改 |
| `notes` | 更新说明（应用内弹层展示） |
| `downloadUrl` | 安装包地址（必须可匿名访问） |
| `size` / `sha256` | 大小与摘要；客户端下载后**强制校验 sha256**，不匹配即丢弃 |
| `downloadMirrors` | 备用下载地址数组，直连失败时按顺序尝试（可加对象存储等国内直连地址） |

> 后端解析同时支持本清单与本仓库式 `downloadMirrors`，也兼容 GitHub Releases 的
> `tag_name` / `assets[]` 格式（两种清单共用一个解析器）。

## 发版步骤

1. 在 HookAny 仓库改版本号、构建、发布 tag，得到 `HookAny-release-v<版本>.apk` 与它的 SHA-256；
2. 把该 APK 上传为本仓库的 Release 资产：

   ```powershell
   $env:GH_TOKEN = "<PAT>"
   py -3 tools\publish-release.py hookany-v<版本> <说明 md 路径> <apk 路径> --repo Zruiry/storehouse
   ```

3. 更新本目录的 `update.json`（版本名、版本号、说明、下载地址、大小、SHA-256）。

## 注意

- **安装包签名与开发期的 debug 包不同**（release 用自有密钥）：交叉安装会失败，需先卸载
  （卸载会清除应用数据，包括 Keystore 加密的 API Key）。
- `raw.githubusercontent.com` 在部分网络下会抖（设备侧实测 3 次得 200/000/200），
  因此应用侧建议同时配一个「备用更新源」；有对象存储时把它填进 `downloadMirrors` 更稳。

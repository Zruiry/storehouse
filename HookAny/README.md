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

3. 更新本目录的 `update.json`（版本名、版本号、说明、下载地址、大小、SHA-256、镜像列表）。

## 加速通道（国内网络）

`github.com` 在国内经常完全不可达（设备侧实测：首页与 Release 资产都超时/空响应），
因此清单里默认带有多个**聚合加速**通道，客户端按顺序挨个尝试，只要有一个通就能装上。

真机实测（Android 14 / Mi 11，同一个时刻取样）：

| 通道 | 清单（1KB） | 安装包（1MB 范围请求） |
|---|---|---|
| `raw.githubusercontent.com` 直连 | 200 / 0.7s（但会抖：3 次中 1 次失败） | **超时** |
| `gh-proxy.com` 前缀 | 200 / 0.8s | 206 / **2.6s** |
| `ghproxy.net` 前缀 | 200 / 1.3s | 206 / 6.4s |
| `ghfast.top` 前缀 | 超时 | — |
| `cdn.jsdelivr.net` | 200 / 1.7s | 301 后超时（**单文件有大小限制，不适合放 APK**） |
| `github.com` 直连（Release 资产） | — | **不可达** |

据此的默认配置：

- 应用侧「更新源」= 本目录 `update.json`；「备用更新源」= 同一清单经 `gh-proxy.com`、
  `ghproxy.net` 各取一份；「下载加速前缀」= `https://gh-proxy.com/ https://ghproxy.net/`
- 清单侧 `downloadUrl` = 目录内的 APK；`downloadMirrors` 依次是
  「两个加速前缀 × Release 资产」与「两个加速前缀 × 目录内 APK」

> 为什么不用 jsDelivr 当清单源：它对分支文件有约 12 小时缓存，缓存期内客户端会拿旧版本号，
> 表现为「明明发了新版却提示已是最新」。只有**透明代理**才适合放清单。

## 注意

- **安装包签名与开发期的 debug 包不同**（release 用自有密钥）：交叉安装会失败，需先卸载
  （卸载会清除应用数据，包括 Keystore 加密的 API Key）。
- 加速域名是第三方服务，寿命不可控；它们只是**兜底**，一旦有对象存储（OSS/COS/七牛）等
  国内直连地址，填进 `downloadMirrors` 更稳。

# HookAny 更新分发目录

[HookAny](https://github.com/Zruiry/HookAny) 的**公开更新源**。主仓库是私有的，匿名客户端
取不到它的 Releases（GitHub 对私有资源的匿名请求一律返回 404），所以把更新清单放在这里。

- `update.json` — 更新清单。**应用内「设置 → 检查更新 → 更新源」填下面这个地址即可**：

  ```
  https://raw.githubusercontent.com/Zruiry/storehouse/main/HookAny/update.json
  ```

- 安装包以**本目录内**的一份为准（清单 `downloadMirrors` 里有对应的 raw 地址）；
  另可作为本仓库的 Release 资产上传（清单 `downloadUrl` 指向它）——那是**可选加成**，
  只为提升国内可达性与下载速度，**不传也算发布成功**。目录内只放最新一个是为了不让 git
  历史随版本线性增长；旧版仍可从对应提交的 git 历史取到。

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

**发布成功的判据：把新版本传到本仓库，使应用内的「检查更新 → 下载 → 安装」可用**
（`update.json` 指向的安装包能匿名取到、`sha256` 与包逐字节一致）。

1. 在 HookAny 仓库改版本号、构建、发布 tag，得到 `HookAny-release-v<版本>.apk` 与它的 SHA-256；
2. 更新本目录的 `update.json`（版本名、版本号、说明、下载地址、大小、SHA-256、镜像列表），
   并把安装包放进本目录（**只留最新一个**），一并提交推送。清单里的 `sha256` 必须与放入的
   APK **逐字节一致**，发完可拉一次线上清单自检；
3. 验证：逐条请求清单里的地址，看到 `200` / `206` 即可（Release 未创建时前两条会 404，
   客户端会继续换源，见下条）。
4. **可选**：把该 APK 上传为本仓库的 Release 资产（tag 用 `hookany-v<版本>`）——这只是加速与
   备份，需要 PAT，没有令牌就不做：

   ```powershell
   $env:GH_TOKEN = "<PAT>"
   py -3 tools\publish-release.py hookany-v<版本> <说明 md 路径> <apk 路径> --repo Zruiry/storehouse
   ```

客户端在 `downloadUrl` → `downloadMirrors[]` 之间逐条尝试，**只有 SHA-256 不匹配才中止**，
普通 `404` 或网络错都会继续换源——所以「Release 资产没传」不影响更新可用。

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

- 应用侧顺序：**`gh-proxy.com` 取清单 → `ghproxy.net` → `raw` 直连兜底**（成功的源会被记住，
  下次先试它）；下载加速前缀同为 `https://gh-proxy.com/ https://ghproxy.net/`
- 清单侧 `downloadUrl` = 加速后的 Release 资产；`downloadMirrors` 依次是另一个加速域名、
  两条「加速 × 目录内 APK」、以及**不经第三方**的 `raw` 直连兜底

### 为什么清单也让加速通道优先

`raw` 由 Fastly 缓存 `max-age=300`，实测**刚发布后同一时刻** raw 仍返回旧版本，而加速通道
已返回新版本（给 URL 加随机查询串**绕不过**这个缓存）：

```
raw 直连（无查询串）   → 26.09.01   ← 陈旧
raw + 随机查询串       → 26.09.01   ← 查询串无效
gh-proxy 加速          → 26.09.02   ← 新鲜
```

所以主源用加速通道：既解决可达性，也避免「发了新版却提示已是最新」的 5 分钟窗口。

> 这也是不用 jsDelivr 当清单源的原因：它对分支文件有约 12 小时缓存，缓存期内必然拿到旧版本。

## 注意

- **安装包签名与开发期的 debug 包不同**（release 用自有密钥）：交叉安装会失败，需先卸载
  （卸载会清除应用数据，包括 Keystore 加密的 API Key）。
- 加速域名是第三方服务，寿命不可控；它们只是**兜底**，一旦有对象存储（OSS/COS/七牛）等
  国内直连地址，填进 `downloadMirrors` 更稳。

# MoxPush

[English](./README.md)

MoxPush 是 MoxChat 的自托管远程通知服务。它保存通知身份和设备 token，把 MoxChat 通知事件转发到 APNs，支持普通 alert token 和 PushKit VoIP token，并提供用于投递诊断的运维页面。

服务源码与目标规格由主 Mox 源码仓库维护，相关规格位于 `spec/notifications/moxpush/`。本仓库只提供部署说明与可分发产物。

## 从 GitHub 获取

本仓库提供部署文档、预编译二进制和懒猫 LPK，无需安装 Go 或自行编译。当前包版本为 `1.3.0`；源码修订、构建时间见 [构建信息](./BUILD-INFO.md)，文件摘要见 [SHA256SUMS](./SHA256SUMS)。

### 克隆整个发布仓库

安装 Git 后执行，Linux、macOS 和 Windows PowerShell 均适用：

```sh
git clone --depth 1 https://github.com/MoxChat/MoxPush.git
cd MoxPush
```

克隆后，在修改配置之前校验文件。Linux 使用：

```sh
sha256sum --check SHA256SUMS
```

macOS 使用：

```sh
shasum -a 256 --check SHA256SUMS
```

### 只下载当前平台的文件

Linux x64 示例（在新的部署目录中执行）：

```sh
mkdir moxpush-release
cd moxpush-release
curl -fL --retry 3 -o moxpush-linux-amd64 https://raw.githubusercontent.com/MoxChat/MoxPush/main/moxpush-linux-amd64
curl -fL --retry 3 -o SHA256SUMS https://raw.githubusercontent.com/MoxChat/MoxPush/main/SHA256SUMS
awk '$2 == "moxpush-linux-amd64" {print}' SHA256SUMS | sha256sum --check
chmod +x ./moxpush-linux-amd64
```

Linux arm64 将命令中的 `linux-amd64` 替换为 `linux-arm64`。macOS 将文件名替换为下表对应的 `darwin-*`，校验命令改用 `shasum -a 256 --check`。

Windows x64 可在新的目录中通过 PowerShell 下载并校验：

```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/MoxChat/MoxPush/main/moxpush-windows-amd64.exe" -OutFile ".\moxpush-windows-amd64.exe"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/MoxChat/MoxPush/main/SHA256SUMS" -OutFile ".\SHA256SUMS"
$expected = ((Get-Content .\SHA256SUMS | Select-String '  moxpush-windows-amd64\.exe$').Line -split '\s+')[0]
if ((Get-FileHash .\moxpush-windows-amd64.exe -Algorithm SHA256).Hash -ne $expected) { throw "SHA-256 校验失败" }
```

Windows arm64 将 `windows-amd64` 替换为 `windows-arm64`。单文件下载不会包含 `.env`，可按下方部署示例设置环境变量。只有校验通过后才启动或安装。

以上直链读取 `main` 分支。需要固定版本时，把所有下载地址中的 `main` 替换为同一个发布仓库提交 SHA；构建信息中的源码修订属于主源码仓库，不能用于这些下载地址。若下载期间分支更新导致校验失败，请固定同一提交后重新下载。

## 发布文件

通过上面的 GitHub 命令获取与你的部署目标匹配的文件：

| 目标 | 文件 |
| --- | --- |
| 懒猫微服 | `moxpush.lpk` |
| Linux x64 | `moxpush-linux-amd64` |
| Linux arm64 | `moxpush-linux-arm64` |
| macOS Intel | `moxpush-darwin-amd64` |
| macOS Apple Silicon | `moxpush-darwin-arm64` |
| Windows x64 | `moxpush-windows-amd64.exe` |
| Windows arm64 | `moxpush-windows-arm64.exe` |

## 懒猫微服部署

1. 使用已克隆的 `moxpush.lpk`，或在新的目录中直接下载并校验（Linux 示例；macOS 把 `sha256sum` 换成 `shasum -a 256`）：

```sh
curl -fL --retry 3 -o moxpush.lpk https://raw.githubusercontent.com/MoxChat/MoxPush/main/moxpush.lpk
curl -fL --retry 3 -o SHA256SUMS https://raw.githubusercontent.com/MoxChat/MoxPush/main/SHA256SUMS
awk '$2 == "moxpush.lpk" {print}' SHA256SUMS | sha256sum --check
```

2. 在懒猫应用界面安装，或使用 CLI：

```sh
lzc-cli app install moxpush.lpk
```

3. 打开分配到的 `moxpush` 子域名。
4. 检查 `https://<moxpush-host>/healthz`。

LPK 内包含 MoxPush 进程和 PostgreSQL 服务。它会刻意避免包含任何 `.p8` 私钥、APNs Team ID、APNs Key ID 或 Apple bundle identifier。需要推送投递时，请在部署环境中配置你自己的 Apple Developer APNs 凭据，通过运行时环境变量或挂载的私钥文件提供，私钥不要打入 LPK。

## APNs 要求

MoxPush 需要 Apple Developer APNs auth key：

- Team ID
- Key ID
- iOS bundle ID
- 可选的 VoIP bundle ID
- `.p8` 私钥内容或已挂载的 `.p8` 文件路径
- APNs 环境：`development` 或 `production`

这里的 APNs 环境必须和注册设备 token 的客户端构建环境一致。
不要把 `.p8` 私钥或 Apple 账号标识发布到公开发行包中。

## 通知隐私

文本通知预览由 MoxChat 客户端使用接收者已有的身份公钥加密。MoxPush 只转发不透明的 `encryptedPreview` 信封并请求可变通知，不解密也不生成预览明文；接收设备上的 Notification Service Extension 负责解密。无法解密的客户端会显示通用通知。

## Linux 部署

以下命令假定已进入二进制所在目录，且已安装并启动 PostgreSQL；SQL 在具备创建用户和数据库权限的 PostgreSQL 会话中执行。数据库密码 `change-me` 必须在 SQL 和连接串中同时替换。

先创建 PostgreSQL 数据库：

```sql
CREATE USER moxpush WITH PASSWORD 'change-me';
CREATE DATABASE moxpush OWNER moxpush;
```

启动二进制：

```sh
chmod +x ./moxpush-linux-amd64
export MOXPUSH_ADDR=:8983
export MOXPUSH_DB_DSN='postgres://moxpush:change-me@127.0.0.1:5432/moxpush?sslmode=disable'
export MOXPUSH_PUBLIC_BASE_URL='https://moxpush.example.com'
export MOXPUSH_APNS_TEAM_ID='YOUR_TEAM_ID'
export MOXPUSH_APNS_KEY_ID='YOUR_KEY_ID'
export MOXPUSH_APNS_BUNDLE_ID='com.example.moxchat'
export MOXPUSH_APNS_VOIP_BUNDLE_ID='com.example.moxchat.voip'
export MOXPUSH_APNS_PRIVATE_KEY_FILE='/secure/path/AuthKey_YOUR_KEY_ID.p8'
export MOXPUSH_APNS_ENVIRONMENT='production'
./moxpush-linux-amd64
```

arm64 主机使用 `moxpush-linux-arm64`。

## macOS 部署

使用匹配的 macOS 二进制：

```sh
chmod +x ./moxpush-darwin-arm64
export MOXPUSH_ADDR=:8983
export MOXPUSH_DB_DSN='postgres://moxpush:change-me@127.0.0.1:5432/moxpush?sslmode=disable'
export MOXPUSH_PUBLIC_BASE_URL='https://moxpush.example.com'
export MOXPUSH_APNS_TEAM_ID='YOUR_TEAM_ID'
export MOXPUSH_APNS_KEY_ID='YOUR_KEY_ID'
export MOXPUSH_APNS_BUNDLE_ID='com.example.moxchat'
export MOXPUSH_APNS_PRIVATE_KEY_FILE='/secure/path/AuthKey_YOUR_KEY_ID.p8'
export MOXPUSH_APNS_ENVIRONMENT='development'
./moxpush-darwin-arm64
```

如果 macOS 拦截下载的二进制，移除 quarantine 属性：

```sh
xattr -d com.apple.quarantine ./moxpush-darwin-arm64
```

## Windows 部署

先创建 PostgreSQL 数据库，然后在 PowerShell 中启动：

```powershell
$env:MOXPUSH_ADDR = ":8983"
$env:MOXPUSH_DB_DSN = "postgres://moxpush:change-me@127.0.0.1:5432/moxpush?sslmode=disable"
$env:MOXPUSH_PUBLIC_BASE_URL = "https://moxpush.example.com"
$env:MOXPUSH_APNS_TEAM_ID = "YOUR_TEAM_ID"
$env:MOXPUSH_APNS_KEY_ID = "YOUR_KEY_ID"
$env:MOXPUSH_APNS_BUNDLE_ID = "com.example.moxchat"
$env:MOXPUSH_APNS_PRIVATE_KEY_FILE = "C:\secure\AuthKey_YOUR_KEY_ID.p8"
$env:MOXPUSH_APNS_ENVIRONMENT = "development"
.\moxpush-windows-amd64.exe
```

Windows arm64 主机使用 `moxpush-windows-arm64.exe`。

## 环境变量

发布目录中已经包含默认 `.env` 文件，作为部署模板。MoxPush 读取进程环境变量；启动二进制前需要通过 shell 或部署工具加载该文件，例如 Linux/macOS 可使用 `set -a; . ./.env; set +a`。生产环境必须替换所有 APNs 占位值，且不要把真实 `.p8` 私钥或 Apple Developer 账号信息发布到公开仓库。

| 变量 | 必填 | 说明 |
| --- | --- | --- |
| `MOXPUSH_ADDR` | 否 | 通知 API、relay 回调、健康检查和运维状态页监听地址，默认 `:8983`。 |
| `MOXPUSH_DB_DSN` | 建议填写 | PostgreSQL 连接串，用于存储通知身份、设备 token、投递诊断、去重键和运维 IP 统计；不填时使用内存存储，重启后注册信息会丢失。 |
| `MOXPUSH_PUBLIC_BASE_URL` | 建议填写 | 外部可访问的 HTTPS 地址，用于校验 relay grant 和生成服务 URL。 |
| `MOXPUSH_APNS_TEAM_ID` | APNs 投递需要 | Apple Developer Team ID，必须属于 APNs auth key 和应用标识所在团队。 |
| `MOXPUSH_APNS_KEY_ID` | APNs 投递需要 | Apple Developer 中 `.p8` APNs auth key 对应的 Key ID。 |
| `MOXPUSH_APNS_BUNDLE_ID` | APNs 投递需要 | 普通 APNs alert topic，通常是 iOS app bundle ID，例如 `com.example.moxchat`。 |
| `MOXPUSH_APNS_VOIP_BUNDLE_ID` | VoIP 投递需要 | CallKit 来电使用的 PushKit VoIP APNs topic，必须和 app entitlement 一致；不填时默认 `<bundle id>.voip`。 |
| `MOXPUSH_APNS_PRIVATE_KEY` | APNs 投递需要 | `.p8` 私钥原文；仅在密钥管理系统以文本注入时使用，支持字面量 `\n` 换行。 |
| `MOXPUSH_APNS_PRIVATE_KEY_FILE` | APNs 投递需要 | 已挂载的 `.p8` 私钥文件路径；`MOXPUSH_APNS_PRIVATE_KEY` 为空时使用该文件。 |
| `MOXPUSH_APNS_ENVIRONMENT` | APNs 投递需要 | 默认 APNs 环境，取值 `development` 或 `production`，必须和 app 构建类型及已注册 token 环境一致。 |
| `MOXPUSH_AUTH_FAIL_WINDOW_MINUTES` | 否 | 按源 IP 统计认证失败或 relay 校验失败次数的时间窗口，默认 `30` 分钟。 |
| `MOXPUSH_AUTH_FAIL_BAN_MINUTES` | 否 | 源 IP 超过失败阈值后的临时封禁时长，默认 `30` 分钟。 |
| `MOXPUSH_AUTH_FAIL_BAN_THRESHOLD` | 否 | 统计窗口内允许的认证失败或 relay 校验失败次数，超过后封禁该源 IP，默认 `10`。 |

## 健康检查和运维

- `GET /healthz` 检查 HTTP 进程是否可达。
- `GET /` 打开运维状态页。
- `GET /demo/` 打开包内包含的通知 Demo 页面。
- `GET /status.json` 返回状态快照。
- `GET /status/stream` 返回实时状态流。

生产环境建议通过 HTTPS 暴露服务。MoxChat 客户端应填写外部可访问的推送中继地址，例如 `https://moxpush.example.com`。

## 更新已有部署

先备份数据库、持久化数据和运行配置，停止旧进程。真实配置和数据应放在发布仓库之外；仓库内 `.env` 仅为模板，避免更新时覆盖本地配置。对于通过 Git 克隆且工作区干净的部署：

```sh
git pull --ff-only
```

更新后重新校验 `SHA256SUMS`（Linux：`sha256sum --check SHA256SUMS`；macOS：`shasum -a 256 --check SHA256SUMS`），再使用原有运行配置启动新二进制。单文件部署重新下载同一提交下的二进制与校验文件，LPK 部署重新执行安装命令。文件校验失败时不要继续启动。

启动后在另一终端检查实际监听端口（默认 `8983`）：

```sh
curl -f http://127.0.0.1:8983/healthz
```

预期返回 HTTP 200。再通过公网服务地址检查 `/healthz`，并在 MoxChat 中验证对应功能。`/healthz` 只表明 HTTP 进程可达，不能替代数据库、Mesh、文件传输、SFU 媒体或 APNs 投递验证。

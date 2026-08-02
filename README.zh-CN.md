# MoxPush

[English](./README.md)

> 这是二进制发布仓库。源码、目标架构和通知规格维护在 Mox 主源码仓库；本文只说明当前打包版本的部署方式。

MoxPush 是 MoxChat 的自托管远程通知服务。它保存通知身份和设备 token，把 MoxChat 通知事件转发到 APNs，支持普通 alert token 和 PushKit VoIP token，并提供用于投递诊断的运维页面。

## 发布文件

从发布页下载与你的部署目标匹配的文件：

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

1. 下载 `moxpush.lpk`。
2. 在懒猫应用界面安装，或使用 CLI：

```sh
lzc-cli app install moxpush.lpk
```

3. 打开分配到的 `moxpush` 子域名。
4. 检查 `https://<moxpush-host>/healthz`。

LPK 内包含 MoxPush 进程和 PostgreSQL 服务。它会刻意避免包含任何 `.p8` 私钥、APNs Team ID、APNs Key ID 或 Apple bundle identifier。需要推送投递时，请在部署环境中配置你自己的 Apple Developer APNs 凭据，或用你自己的密钥重新构建包。

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

## Linux 部署

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

# MoxPush

[中文文档](./README.zh-CN.md)

MoxPush is the self-hosted remote notification demo service for MoxChat. It stores notification identities and device tokens, relays MoxChat notification events to APNs, supports alert and PushKit VoIP tokens, and exposes an operations page for delivery diagnostics.

## Release Files

Download the files from the release page that match your target platform:

| Target | File |
| --- | --- |
| Lazycat MicroServer | `moxpush.lpk` |
| Linux x64 | `moxpush-linux-amd64` |
| Linux arm64 | `moxpush-linux-arm64` |
| macOS Intel | `moxpush-darwin-amd64` |
| macOS Apple Silicon | `moxpush-darwin-arm64` |
| Windows x64 | `moxpush-windows-amd64.exe` |
| Windows arm64 | `moxpush-windows-arm64.exe` |

## Lazycat MicroServer Deployment

1. Download `moxpush.lpk`.
2. Install it from the Lazycat app UI, or with the CLI:

```sh
lzc-cli app install moxpush.lpk
```

3. Open the app at the assigned `moxpush` subdomain.
4. Check `https://<moxpush-host>/healthz`.

The LPK includes the MoxPush app process and a PostgreSQL service. It intentionally does not include any `.p8` private key, APNs Team ID, APNs Key ID, or Apple bundle identifier. For push delivery, configure the deployment with your own Apple Developer APNs credentials or rebuild the package with your own secrets.

## APNs Requirements

MoxPush needs an Apple Developer APNs auth key:

- Team ID
- Key ID
- iOS bundle ID
- Optional VoIP bundle ID
- `.p8` private key content or a path to the mounted `.p8` file
- APNs environment: `development` or `production`

Use the same APNs environment as the client build that registers device tokens.
Do not publish your `.p8` key or Apple account identifiers in a public release package.

## Linux Deployment

Create a PostgreSQL database:

```sql
CREATE USER moxpush WITH PASSWORD 'change-me';
CREATE DATABASE moxpush OWNER moxpush;
```

Run the binary:

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

Use `moxpush-linux-arm64` on arm64 hosts.

## macOS Deployment

Use the matching macOS binary:

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

If macOS blocks a downloaded binary, remove the quarantine attribute:

```sh
xattr -d com.apple.quarantine ./moxpush-darwin-arm64
```

## Windows Deployment

Create the PostgreSQL database first, then start MoxPush from PowerShell:

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

Use `moxpush-windows-arm64.exe` on Windows arm64 hosts.

## Environment Variables

This release directory includes a default `.env` file as a deployment template. MoxPush reads process environment variables; load the file with your shell or deployment tool before starting the binary, for example `set -a; . ./.env; set +a` on Linux/macOS. Replace all APNs placeholders before production use, and never publish a real `.p8` private key or Apple Developer account values.

| Variable | Required | Description |
| --- | --- | --- |
| `MOXPUSH_ADDR` | No | Network address for notification APIs, relay callbacks, health checks, and the operations page. Default: `:8983`. |
| `MOXPUSH_DB_DSN` | Recommended | PostgreSQL connection string used for notification identities, device tokens, delivery diagnostics, dedupe keys, and operations IP statistics. If omitted, storage is in memory and registrations are lost on restart. |
| `MOXPUSH_PUBLIC_BASE_URL` | Recommended | Externally reachable HTTPS URL used when validating relay grants and generating service URLs. |
| `MOXPUSH_APNS_TEAM_ID` | APNs delivery | Apple Developer Team ID that owns the APNs auth key and app identifiers. |
| `MOXPUSH_APNS_KEY_ID` | APNs delivery | APNs auth key ID shown in Apple Developer for the `.p8` key. |
| `MOXPUSH_APNS_BUNDLE_ID` | APNs delivery | Standard APNs alert topic, normally the iOS app bundle ID such as `com.example.moxchat`. |
| `MOXPUSH_APNS_VOIP_BUNDLE_ID` | VoIP delivery | PushKit VoIP APNs topic for CallKit calls. It must match the app entitlement; defaults to `<bundle id>.voip` when omitted. |
| `MOXPUSH_APNS_PRIVATE_KEY` | APNs delivery | Raw `.p8` private key content. Use this only when your secret manager injects the key as text; literal `\n` line breaks are supported. |
| `MOXPUSH_APNS_PRIVATE_KEY_FILE` | APNs delivery | Filesystem path to the mounted `.p8` private key. Used when `MOXPUSH_APNS_PRIVATE_KEY` is empty. |
| `MOXPUSH_APNS_ENVIRONMENT` | APNs delivery | Default APNs environment, `development` or `production`. It must match the app build and registered device token environment. |
| `MOXPUSH_AUTH_FAIL_WINDOW_MINUTES` | No | Time window used to count failed authentication or relay verification attempts per source IP. Default: `30`. |
| `MOXPUSH_AUTH_FAIL_BAN_MINUTES` | No | Temporary ban duration after a source IP exceeds the failure threshold. Default: `30`. |
| `MOXPUSH_AUTH_FAIL_BAN_THRESHOLD` | No | Number of failed authentication or relay verification attempts allowed in the window before banning the source IP. Default: `10`. |

## Health and Operations

- `GET /healthz` verifies that the HTTP process is reachable.
- `GET /` opens the operations/status page.
- `GET /demo/` opens the notification demo page when included by the package.
- `GET /status.json` returns a status snapshot.
- `GET /status/stream` streams live status updates.

Expose the service through HTTPS in production. MoxChat clients should use the externally reachable push relay URL, for example `https://moxpush.example.com`.

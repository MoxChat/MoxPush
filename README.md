# MoxPush

[中文文档](./README.zh-CN.md)

MoxPush is the self-hosted remote notification service for MoxChat. It stores notification identities and device tokens, relays authorized MoxChat notification candidates to APNs, supports alert and PushKit VoIP tokens, and exposes an operations page for delivery diagnostics.

Service source code and target specifications are maintained in the main Mox source repository under `spec/notifications/moxpush/`. This repository contains deployment documentation and distributable artifacts only.

## MoxChat Clients

- iOS: [Download on the App Store](https://apps.apple.com/us/app/moxchat/id6775016915)
- Web: [Open MoxChat](https://app.ponzs.com)

## Get the Files from GitHub

This repository provides deployment documentation, prebuilt binaries, and Lazycat LPK packages. You do not need Go or a local build. The current package version is `1.3.0`; see [build information](./BUILD-INFO.md) for the source revision and build timestamp, and [SHA256SUMS](./SHA256SUMS) for file digests.

### Clone the Release Repository

With Git installed, run these commands on Linux, macOS, or Windows PowerShell:

```sh
git clone --depth 1 https://github.com/MoxChat/MoxPush.git
cd MoxPush
```

Verify the files after cloning and before editing configuration. On Linux:

```sh
sha256sum --check SHA256SUMS
```

On macOS:

```sh
shasum -a 256 --check SHA256SUMS
```

### Download Only the Files for Your Platform

Linux x64 example, using a new deployment directory:

```sh
mkdir moxpush-release
cd moxpush-release
curl -fL --retry 3 -o moxpush-linux-amd64 https://raw.githubusercontent.com/MoxChat/MoxPush/main/moxpush-linux-amd64
curl -fL --retry 3 -o SHA256SUMS https://raw.githubusercontent.com/MoxChat/MoxPush/main/SHA256SUMS
awk '$2 == "moxpush-linux-amd64" {print}' SHA256SUMS | sha256sum --check
chmod +x ./moxpush-linux-amd64
```

For Linux arm64, replace `linux-amd64` with `linux-arm64`. On macOS, use the matching `darwin-*` filename from the table below and replace the checksum command with `shasum -a 256 --check`.

For Windows x64, download and verify the files in a new directory using PowerShell:

```powershell
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/MoxChat/MoxPush/main/moxpush-windows-amd64.exe" -OutFile ".\moxpush-windows-amd64.exe"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/MoxChat/MoxPush/main/SHA256SUMS" -OutFile ".\SHA256SUMS"
$expected = ((Get-Content .\SHA256SUMS | Select-String '  moxpush-windows-amd64\.exe$').Line -split '\s+')[0]
if ((Get-FileHash .\moxpush-windows-amd64.exe -Algorithm SHA256).Hash -ne $expected) { throw "SHA-256 verification failed" }
```

For Windows arm64, replace `windows-amd64` with `windows-arm64`. Individual binary downloads do not include `.env`; set environment variables as shown in the deployment examples below. Start or install the service only after verification succeeds.

These direct URLs read the `main` branch. To pin a version, replace `main` in every download URL with the same release repository commit SHA. The source revision in the build information belongs to the main source repository and cannot be used in these URLs. If the branch changes during download and verification fails, pin one commit and download the files again.

## Release Files

Use the GitHub commands above to obtain the files that match your target platform:

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

1. Use `moxpush.lpk` from your clone, or download and verify it in a new directory (Linux example; on macOS, replace `sha256sum` with `shasum -a 256`):

```sh
curl -fL --retry 3 -o moxpush.lpk https://raw.githubusercontent.com/MoxChat/MoxPush/main/moxpush.lpk
curl -fL --retry 3 -o SHA256SUMS https://raw.githubusercontent.com/MoxChat/MoxPush/main/SHA256SUMS
awk '$2 == "moxpush.lpk" {print}' SHA256SUMS | sha256sum --check
```

2. Install it from the Lazycat app UI, or with the CLI:

```sh
lzc-cli app install moxpush.lpk
```

3. Open the app at the assigned `moxpush` subdomain.
4. Check `https://<moxpush-host>/healthz`.

The LPK includes the MoxPush app process and a PostgreSQL service. It intentionally does not include any `.p8` private key, APNs Team ID, APNs Key ID, or Apple bundle identifier. For push delivery, provide your own Apple Developer APNs credentials through runtime environment variables or a mounted private-key file. Do not bundle the private key in an LPK.

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

## Notification Privacy

Text notification previews are encrypted by the MoxChat client with the recipient's existing identity public key. MoxPush forwards the opaque `encryptedPreview` envelope and requests a mutable notification; it does not decrypt or generate the preview text. The Notification Service Extension decrypts the envelope on the recipient device. Clients that cannot decrypt the envelope display a generic notification instead.

## Linux Deployment

These commands assume you are in the binary directory and PostgreSQL is installed and running. Run the SQL in a PostgreSQL session with permission to create users and databases. Replace `change-me` in both the SQL and the connection string with the same database password.

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

## Update an Existing Deployment

Back up the database, persistent data, and runtime configuration, then stop the old process. Keep real configuration and data outside the release repository; the included `.env` is a template, so updates should not overwrite your local configuration. For a Git clone with a clean working tree:

```sh
git pull --ff-only
```

Verify `SHA256SUMS` again (Linux: `sha256sum --check SHA256SUMS`; macOS: `shasum -a 256 --check SHA256SUMS`), then start the new binary with your existing runtime configuration. For individual downloads, download the binary and checksum file from the same commit again. For LPK deployments, run the installation command again. Do not start the service if file verification fails.

After startup, check the actual listening port from another terminal (default: `8983`):

```sh
curl -f http://127.0.0.1:8983/healthz
```

Expect HTTP 200. Then check `/healthz` through the public service URL and verify the corresponding feature in MoxChat. `/healthz` only confirms that the HTTP process is reachable; it does not replace database, mesh, file transfer, SFU media, or APNs delivery validation.

# TZ VPN

<p align="center">
  <strong>A simple, free and advertisement-free Android WireGuard VPN client.</strong>
</p>


https://github.com/tzalmamun/TZ-VPN-/releases/download/Apk/TZ.VPN_1.0.apk


<p align="center">
  ![Platform](https://img.shields.io/badge/platform-Android-green?logo=android)
  ![Language](https://img.shields.io/badge/language-Kotlin-purple?logo=kotlin)
  ![Protocol](https://img.shields.io/badge/protocol-WireGuard-blue)
  ![Ads](https://img.shields.io/badge/ads-none-brightgreen)
</p>

TZ VPN is an Android WireGuard client project. It is intended to let people manage their own WireGuard tunnel configurations on their Android devices.

**TZ VPN does not provide VPN servers, VPN credentials, or free internet access.** Users need a valid configuration and access to a working WireGuard server.

## Planned features

- Import a WireGuard `.conf` file using Android's system file picker
- Scan a QR code containing a WireGuard configuration
- Create and edit a tunnel manually
- Save, rename, and remove user-managed configurations
- Connect and disconnect a tunnel
- Show tunnel connection state
- Show locally measured upload and download traffic, where supported
- Optional foreground notification while a tunnel is active
- Clean, light interface inspired by the official WireGuard Android app
- No advertisements and no mandatory subscription

> Features are only available when implemented in the published build. Check the GitHub Releases notes for the current implementation status.

## Screenshots

Add real screenshots of the current TZ VPN build to this directory:

| Home | Add tunnel |
|---|---|
| `screenshots/home.png` | `screenshots/add-tunnel.png` |

| Tunnel details | Settings |
|---|---|
| `screenshots/details.png` | `screenshots/settings.png` |

## Requirements

- Android 7.0 (API 24) or later, if supported by the build
- A valid WireGuard configuration
- Internet access to reach the configured VPN endpoint

## Install

1. Open the [Releases](../../releases) page.
2. Download the APK attached to the release.
3. Install it on your Android device.
4. Import your own WireGuard configuration.
5. Approve Android's VPN permission prompt and connect.

Only install APKs published by a source you trust. Review release notes and checksums when provided.

## Build from source

This repository is a starter repository structure. Add the Android Studio project files before following these commands.

Once the Android project and Gradle wrapper are committed:

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/TZ-VPN.git
cd TZ-VPN
./gradlew assembleDebug
```

On Windows:

```powershell
.\gradlew.bat assembleDebug
```

The debug APK is normally generated at:

```text
app/build/outputs/apk/debug/app-debug.apk
```

## Data usage and quotas

Local tunnel traffic counters and provider billing quotas are different things.

- Local upload/download statistics may be measured by the client or tunnel backend.
- A provider's remaining quota or expiry date cannot be reliably inferred from a WireGuard `.conf` file.
- Quota and expiry information should only be displayed when supplied by a genuine, documented provider API or another verified source.
- TZ VPN must not fabricate usage, remaining data, or expiry information.

## Privacy and security

- Treat WireGuard configuration files as sensitive: they may contain private keys.
- Never commit personal `.conf` files, private keys, QR codes, credentials, or production secrets.
- Do not log private keys or upload configurations without clear user consent.
- Use Android's document picker rather than requesting broad storage access when possible.
- Review the app's permissions and privacy policy before publishing a release.

A VPN does not guarantee anonymity. The operator of the VPN server may be able to observe traffic metadata, and websites can still identify users through other means.

## Development

- Application ID: `com.tz.vpn`
- Language: Kotlin
- Platform: Native Android
- VPN protocol: WireGuard
- Intended minimum SDK: 24

Use the official WireGuard Android project and its published documentation as the source of truth for library integration and licensing. Verify dependency versions and APIs against the version used by the app.

## Contributing

Bug reports and improvements are welcome.

1. Open an issue describing the problem or proposed change.
2. Do not attach private VPN configurations, private keys, passwords, or personal data.
3. For code changes, create a branch and submit a pull request.

## License

The license for TZ VPN's original source code is described in [`LICENSE`](LICENSE). Third-party components retain their own licenses and notices. Do not assume that the license of this repository changes the license of WireGuard or other dependencies.

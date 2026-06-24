# TESTING-MRA

Over-the-air (OTA) update channel for **debug** builds of the Spaceti Meeting Room App (MRA).

> ⚠️ **This repository does not contain source code.** It exists only to host release artifacts (APKs). The app's source lives in [`spaceti/MRA`](https://github.com/spaceti/MRA). This is the *testing* counterpart of the production channel [`spaceti/MRAReleases`](https://github.com/spaceti/MRAReleases).

---

## What it's for

The Meeting Room App is a Compose Multiplatform Android application that runs on meeting-room tablets, typically unattended and locked into single-app / device-owner mode. Because the devices are locked down, the app updates itself rather than relying on the Play Store.

To do this, the running app periodically polls the GitHub Releases API of this repository:

```
GET https://api.github.com/repos/spaceti/TESTING-MRA/releases/latest
```

It then:

1. Reads the `tag_name` of the latest release (which is the Android **`versionCode`**, an integer).
2. Compares it against the `versionCode` currently installed on the device.
3. If the release is newer, finds the release asset whose content type is `application/vnd.android.package-archive` (the APK).
4. Downloads it from the asset's `browser_download_url`.
5. Installs it silently via Android `PackageInstaller` (device-owner privilege), or surfaces a manual-download prompt.

Update checks run once a day at a randomized night-time window (between 00:00 and 05:00, offset from the device's reboot time), so a new release published here will roll out to test devices automatically.

### Channels

| Build type | Releases repository | Configured in |
|------------|---------------------|---------------|
| `debug`    | `spaceti/TESTING-MRA` (**this repo**) | `shared/build.gradle.kts` → `GITHUB_URL` |
| `release`  | `spaceti/MRAReleases` | `shared/build.gradle.kts` → `GITHUB_URL` |

The URL is injected at build time as the `BuildConfig.GITHUB_URL` field and consumed in `KioskApi.fetchLatestRelease()`.

> **Migration note:** the debug channel previously pointed at `wuujcik/mraTest`. After moving to this repository, update the `debug` build type's `GITHUB_URL` in `shared/build.gradle.kts` to `"/repos/spaceti/TESTING-MRA/releases/latest"` and ship a new debug build so existing test devices start polling here.

---

## Publishing a release

Each GitHub Release in this repo represents one installable build. For the app to detect and install it, the release **must** follow these conventions.

### 1. Tag name = `versionCode` (integer)

The tag name is parsed with `tagName.toIntOrNull()` and compared numerically to the installed `versionCode`.

- ✅ `102`, `103`, `104`
- ❌ `v2.17.1`, `2.17.1`, `release-102` — a non-integer tag will break update detection.

The `versionCode` for a given build comes from `buildSrc/src/main/kotlin/ProjectSettings.kt` in the source repo (e.g. `versionCode = 102`). The tag here must match the `versionCode` baked into the APK you upload.

### 2. Attach the APK as a release asset

Upload the built APK as a binary asset on the release. The app selects the asset whose content type is:

```
application/vnd.android.package-archive
```

GitHub assigns this content type automatically to `.apk` files. Attach exactly one APK per release.

### 3. Use a human-readable release name

The release `name` (title) is shown to operators as the release label, so use the marketing version: `2.17.1 (102)`.

### 4. Publish it (not draft, not pre-release)

The `releases/latest` endpoint only returns the most recent **published**, non-draft, non-prerelease release. Drafts and pre-releases are ignored by the app.

### Checklist

- [ ] Tag = integer `versionCode` matching the APK (e.g. `102`)
- [ ] Exactly one `.apk` attached as an asset
- [ ] Descriptive release name (e.g. `2.17.1 (102)`)
- [ ] Version code is **higher** than the currently deployed build
- [ ] Release is published (not a draft / pre-release)

---

## How the app consumes a release (reference)

```
┌─────────────────────────────┐
│ Kiosk device (debug build)  │
└──────────────┬──────────────┘
               │  GET /repos/spaceti/TESTING-MRA/releases/latest
               ▼
        api.github.com  ──────────────►  this repository
               │
               │  { tag_name, name, assets: [{ content_type, browser_download_url, … }] }
               ▼
   tag_name.toInt() > installed versionCode ?
               │ yes
               ▼
   download APK asset → silent install (PackageInstaller, device owner)
```

Relevant source (in `spaceti/MRA`):

- `shared/build.gradle.kts` — sets `GITHUB_URL` per build type
- `shared/src/androidMain/kotlin/meetingRoomApp/data/network/apis/GithubApi.kt` — `KioskApi.fetchLatestRelease()`
- `shared/src/androidMain/kotlin/meetingRoomApp/data/common/models/GithubResponse.kt` — response/asset model
- `shared/src/androidMain/kotlin/meetingRoomApp/data/common/repositories/KioskRepository.kt` — version comparison, download, silent install
- `buildSrc/src/main/kotlin/ProjectSettings.kt` — `versionCode` / `versionName`

---

## Notes

- Tags must increase monotonically; if a release's `versionCode` is **lower** than what's installed, the app treats the device as up-to-date and does not downgrade.
- On every successful check the app replaces its locally stored "latest release" record with whatever GitHub currently reports, so a yanked or re-tagged release won't leave a stale row behind.
- Keep this channel separate from `MRAReleases` — anything published here ships only to debug/test devices.

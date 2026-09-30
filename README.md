# OpenMinis-Build

Builds an installable **OpenMinis** Android APK on GitHub's runners, and is the
place where **local modifications** to the app live until they are good enough
to go upstream.

Upstream: [OpenMinis/OpenMinis](https://github.com/OpenMinis/OpenMinis) (GPL-3.0).
This repository does **not** vendor the source — it holds the build recipe and
the patches, and the runner clones upstream at a pinned commit.

## Layout

| Path | What it is |
|---|---|
| `.github/workflows/android-apk.yml` | The whole pipeline: SDK/NDK setup, native sandbox deps, Gradle build, artifact + Release |
| `patches/*.patch` | Local changes applied to upstream before building, in filename order. Empty = build upstream as-is |
| `UPSTREAM_SHA` | Pinned in the workflow `env:`; bump it to move to a newer upstream |

## Getting an APK

Every push to `main`, or a manual run from **Actions → Android APK → Run
workflow**, produces:

- an `out/*.apk` **artifact** on the run page, and
- a **Release** tagged `build-<run number>` with the same APK attached
  (downloadable without unzipping).

The debug APK is signed with the standard Android debug key — the project's
`release` build type deliberately uses that same key (see upstream
`BUILDING.md`), so builds from this repo install over each other. Only
`arm64-v8a` is packaged, matching the app's `abiFilters`.

## Why the native deps are rebuilt here

The app ships a Linux sandbox (PRoot + Alpine rootfs) plus FFmpeg/LAME/rclone,
none of which are committed upstream — `BUILDING.md` builds them from source.
This workflow does exactly that, then caches the results keyed on the pinned
SHA + the workflow itself + `patches/`. The first run takes ~30–45 min; later
runs skip the heavy steps (Gradle is cached separately).

`deps/build_proot.sh` must produce all three of `libproot.so`,
`libproot-loader.so` and `libproot-loader32.so`. Without the loaders the APK
installs and then fails every shell command with `[Shell not running]`, so the
workflow asserts they exist *before* Gradle runs.

## Signing

Every build is signed with the **same key**, `keystore/minis-build.p12`
(throwaway self-signed, CN=OpenMinis local build, valid to 2056,
`patches/0001-pinned-signing.patch` makes Gradle use it):

```
SHA-256  8A:A1:78:4D:87:AA:1D:EC:CD:EC:EA:06:A0:19:45:85:6D:65:3E:A5:09:FC:0F:F0:4D:BF:3E:BB:62:ED:E7:BE
```

That matters because Android refuses to install an APK whose signer differs
from the installed one ("App not installed"). The default debug keystore is
generated per machine — and per CI runner — so leaving it in place made every
build a *different* app: nothing could be updated in place, ever. The workflow
now asserts the signer fingerprint on each APK before publishing.

**The official build is signed with the developer's key**, so moving from the
store version to this one requires uninstalling it once (and exporting its data
first — see upstream's backup/restore). After that, builds from here install
over each other normally.

## Changing the app

1. Add a `.patch` under `patches/` (e.g. `git -C <upstream clone> diff > 0001-thing.patch`).
2. Push — CI applies it, builds, and publishes a new Release.
3. A red run leaves the previous Release untouched, so the last good APK stays
   downloadable.

## Notes

- The workflow needs `contents: write` (set in the file) to publish Releases.
- `workflow_dispatch` takes `build_release=true` to also produce the minified
  release APK.
- Local development of the patches is done against a normal clone of upstream;
  nothing in this repo requires a local Android toolchain.

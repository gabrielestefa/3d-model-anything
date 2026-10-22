# Fork notes: gabrielestefa/3d-model-anything

Fork of [wesbos/3d-model-anything](https://github.com/wesbos/3d-model-anything). The goal is to run the **capture app** (scan + in-app 3D review) on an **iPhone 12 Pro**, using a **Windows 11** PC and no Mac.

- **Install guide:** [`docs/iphone-sideload.md`](docs/iphone-sideload.md).
- **This file:** the record of changes, status, gotchas and next steps, for humans and AI agents.

## Status

| Milestone | State |
|---|---|
| M1: cloud build of unsigned `.ipa` (GitHub Actions) | Written and lint-checked, **not yet run** |
| M2: sign and install via Sideloadly on Windows | Pending M1 |
| M3: first scan, in-app 3D review, backup to Files | Pending M2 |
| (optional) M4: view own reconstructed rooms in the web viewer | Not started |

## Changes from upstream

Only new files are added; upstream files are untouched, so **Sync fork** stays conflict-free.

| File | What | Why |
|---|---|---|
| `.github/workflows/build-ipa.yml` | Manual workflow on the `xcode-27` runner. Runs XcodeGen, then a Release `xcodebuild` with signing off and the bundle ID overridden, packages `Payload/` into a `.ipa`, and uploads it as an artifact. | No Mac available. Sideloadly does the signing. |
| `docs/iphone-sideload.md` | Windows install and 7-day refresh guide | User-facing steps |
| `FORK-NOTES.md` | This file | Project memory |

## How the build works

```
project.yml ──xcodegen──▶ HouseCapture.xcodeproj (gitignored, generated in CI)
           ──xcodebuild Release, generic iOS, CODE_SIGNING_ALLOWED=NO,
             PRODUCT_BUNDLE_IDENTIFIER=<input>──▶ HouseCapture.app
           ──zip Payload/──▶ HouseCapture-unsigned.ipa (artifact, 90 days)
```

The workflow fails if the `.app` is missing or its `CFBundleIdentifier` doesn't match the input. On failure, the full `xcodebuild.log` is uploaded.

## Relevant repo structure

| Path | Contents |
|---|---|
| `apps/house-capture/` | iOS app: SwiftUI + ARKit + RoomPlan + SceneKit. 29 Swift files, no package dependencies, no entitlements file. Spec in `project.yml`. |
| `apps/room-viewer/` | Three.js web viewer (Vite). Upstream deploys it to GitHub Pages. |
| `.agents/skills/scan-to-threejs/` | Agent skill that turns exported scans into Three.js scenes. Runs on the PC. |
| `demos/` | Reconstructed rooms. A new room goes in `demos/<id>/` and is registered in `demos/index.ts`. |
| `tools/audit-public.py` | Runs in `check.yml` on every push. It rejects macOS home-folder paths (Users + username), credentials, unlisted `data/` files and unreviewed demos. |

## Gotchas

- **SDK floor:** `HouseViewer.swift` calls `glassEffect` (iOS 26 SDK) behind `#available`. Build with Xcode 26 or newer. The deployment target is still iOS 17.
- **Bundle ID:** the app code never references it, so overriding it at build time is safe. Keep `com.gabrielestefanelli.housecapture` forever. A different ID installs as a separate, empty app.
- **Release only:** upstream notes that Debug disables Swift optimizations and makes scanning slow. Debug-only launch modes (`-selftest`, `-autotest`, `-diag`) are unavailable in this build.
- **Data lives in the app sandbox:** deleting the app loses the scans. Re-sign over the top instead.
- **Inherited workflows:** `pages.yml` fails on this fork until Pages is enabled, so disable it or enable Pages. `check.yml` is harmless.
- **actionlint 1.7.12** flags the `xcode-27` label as unknown. The label is newer than that linter release (GA 2026-10-15). If it's ever unavailable, use `macos-26` plus `xcode-select`.
- **`audit-public.py`:** new docs must not contain a macOS home-folder path (slash, `Users`, slash, a name, slash), not even as an example, or `check.yml` fails. This file tripped it once.
- **A14 device:** the app throttles depth and photo capture under thermal or memory pressure, which happens sooner than on newer phones.

## Remaining tasks

- [ ] Push these files to the fork (the session had no write access when they were written).
- [ ] Enable Actions on the fork and disable `Deploy viewer`.
- [ ] Run the workflow and confirm the `xcode-27` label and build succeed. Record the Xcode and SDK versions here.
- [ ] Install via Sideloadly and confirm Developer Mode, trust, camera permission and first scan.
- [ ] Optional: check whether Sideloadly's auto-refresh fits, to avoid the weekly manual re-sign.
- [ ] Optional M4: export a scan to the PC, run `scan-to-threejs`, and host your own viewer on Pages.

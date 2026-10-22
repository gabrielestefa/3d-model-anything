# Fork notes: gabrielestefa/3d-model-anything

Fork of [wesbos/3d-model-anything](https://github.com/wesbos/3d-model-anything). The goal is to run the **capture app** (scan + in-app 3D review) on an **iPhone 12 Pro**, using a **Windows 11** PC and no Mac.

- **Install guide:** [`docs/iphone-sideload.md`](docs/iphone-sideload.md).
- **This file:** the record of changes, status, gotchas and next steps, for humans and AI agents.

## Status

| Milestone | State |
|---|---|
| M1: cloud build of unsigned `.ipa` (GitHub Actions) | ✅ Done 2026-10-22: run #1 succeeded in 1m56s, artifact 4.8 MB (zip) |
| M2: sign and install on Windows | ✅ Done 2026-10-23 with **iloader** (Sideloadly blocked by Apple ID error -80046) |
| M3: first scan + in-app 3D review | ✅ Done 2026-10-23: one attic room scanned |
| M4: process a scan into an editable Three.js room | ✅ Quick test done 2026-10-23 in a Claude workspace (private, not in this repo); see *Processing a scan* |
| M5: repeatable local processing (PC) or reusable cloud route | Not started |

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

## Processing a scan (M4 quick test, 2026-10-23)

Route used: the room export (`.zip`) was uploaded to a Claude chat and processed in Claude's private Linux workspace. The room data, the room module and the result **are not in this public repo**, on purpose. The result was published as a private claude.ai page and also given to the user as a zip.

Steps (all from the target repo root, with the skill's scripts as `S`):
1. Extract the export into `data/captures-in/<room>-raw/` (gitignored).
2. `python3 $S/bundle_to_room.py <session> --room 0 -o data/demos/<room>.json`, then `prepare_photos.py` into `data/web-assets/<room>/photos`.
3. Measure what RoomPlan misses from `mesh/room-0.ply` (numpy probes). Here that was the vaulted ceiling heightfield and the real furniture tops and fronts.
4. Write `demos/<room>/index.ts` (a `RoomModule`) and register it first in `demos/index.ts`.
5. `npm run build` (runs `tsc`), then do matched-camera checks: sections (`view=e|n&cut=`) and photo views (`photo=N`) against the scan, fixing the largest differences first.
6. Private viewer: `VIEWER_BASE=./` build, keep only the reachable `assets/`, the base `textures/*.jpg` and the room's scan file, and publish as a claude.ai artifact.

Result for the quick test: walls, vault, floor and 10 furniture blockouts. Measured: wall plan, knee walls, ceiling (82% of cells), bed footprint, desk/drawer/cabinet tops and fronts. Approximate: furniture shapes, colors and lighting. Not modeled: monitor, floor lamp, small objects.

### Processing gotchas
- **PLY `inroom` flag clips vaults:** the app marks a vertex "in room" only up to about 1.7 m (RoomPlan wall height + margin). Above 1.8 m nothing is flagged, so an attic ceiling vanishes. Select by wall footprint plus a height window instead.
- **ARKit labels ceiling coves as "wall":** a heightfield from `class == ceiling` faces leaves the wall edges empty and the gap fill overshoots there (0.2–0.3 m too high). Use the upper envelope (p95) of all surfaces above the furniture per 10 cm cell instead (82% measured vs 46%).
- **RoomPlan merges neighbours:** a desk and the drawer unit beside it came out as one 2 m "table". Probe the mesh for top heights per x-bin and for front planes (`|nz| > 0.85`) to split them.
- **RoomPlan `front` is local +z:** objects against a wall often have `rotY = PI`. Build drawer fronts, handles and backrests relative to local +z, or they end up facing the wall.
- **HemisphereLight ground color tints ceilings:** downward-facing surfaces take the ground color. A dark wood-floor color turned a white vault brown. Use a pale bounce color.
- **Skill capture scripts are zsh plus `agent-browser`:** on Windows they need WSL. In Claude's sandbox, `agent-browser install` can't download Chrome. `@sparticuz/chromium` + `puppeteer-core` from npm gives headless Chrome with WebGL2 (SwiftShader), so `cap.sh` was ported to a small Node script.
- **Artifact hosting:** it doesn't serve `.bin` files, so the scan ships as base64 inside `.json`. Build with `VIEWER_BASE=./`, drop the office demo (`VITE_ONLY_UPPER_FLOOR=1` in the private copy) and hide the *Play* link, which needs `play.html` and the Rapier wasm.
- **Small objects (detail pass):** RoomPlan reports only large furniture. Find objects on each support surface as connected blobs of mesh vertices above the surface (2 cm grid, `scipy.ndimage.label`), taking footprint, top height and median scan color per blob. Model them as simple hints.
- **LiDAR misses thin and glossy things:** a monitor's black screen and a thin lamp pole barely register. Size them from the photos instead. A light source can be located by voting: project candidate 3D points into every posed photo and keep the point that lands on saturated pixels most often.
- **Reading photos against the model:** camera right = `forward × up`. Check left/right placement in a photo-matched render before trusting a photo for which side an object is on. Two items came out mirrored on the first try.
- **Privacy:** the fork is public, and `demos/<room>/`, `data/demos/<room>*` and `data/web-assets/<room>/` are not gitignored. Never commit them here. `audit-public.py` would also fail `check.yml` on them.

## Build log

| Run | Date | Result | Notes |
|---|---|---|---|
| #1 ([39858014234](https://github.com/gabrielestefa/3d-model-anything/actions/runs/39858014234)) | 2026-10-22 | ✅ success, 1m56s | `xcode-27` label accepted. Bundle-ID assert passed. Artifact expires 2027-01-20. Xcode/SDK version: see the *Show toolchain* step. |

## Gotchas

- **SDK floor:** `HouseViewer.swift` calls `glassEffect` (iOS 26 SDK) behind `#available`. Build with Xcode 26 or newer. The deployment target is still iOS 17.
- **Bundle ID:** the app code never references it, so overriding it at build time is safe. Keep `com.gabrielestefanelli.housecapture` forever. A different ID installs as a separate, empty app.
- **Release only:** upstream notes that Debug disables Swift optimizations and makes scanning slow. Debug-only launch modes (`-selftest`, `-autotest`, `-diag`) are unavailable in this build.
- **Data lives in the app sandbox:** deleting the app loses the scans. Re-sign over the top instead.
- **Inherited workflows:** `pages.yml` fails on this fork until Pages is enabled, so disable it or enable Pages. `check.yml` is harmless.
- **actionlint 1.7.12** flags the `xcode-27` label as unknown. The label is newer than that linter release (GA 2026-10-15). If it's ever unavailable, use `macos-26` plus `xcode-select`.
- **`audit-public.py`:** new docs must not contain a macOS home-folder path (slash, `Users`, slash, a name, slash), not even as an example, or `check.yml` fails. This file tripped it once.
- **Node 20 deprecation:** GitHub warns that `actions/checkout@v4` and `actions/upload-artifact@v4` target Node 20 and now run on Node 24. Builds still work, and upstream uses the same versions. Bump them only if they start failing.
- **Sideloadly auto refresh isn't cloud-based:** the PC daemon re-signs only while it can see the phone (USB, or same Wi-Fi with iTunes Wi-Fi sync and the screen on). About 7 days apart means the app expires. Source: sideloadly.io/faq.html.
- **Apple ID sign-in breakage (Oct 2026):** since 2026-10-12, Apple rejects sign-ins from free signing tools with "This action cannot be completed at this time" (-80045; -80046 reported from 2026-10-21). Developers suspect Apple now checks the device identity ("anisette") the tools send. Fixes ship per tool: iloader 2.3.7+ and SideInstaller 1.5.0+ are fixed; Sideloadly 0.71.0 reworked sign-in but doesn't claim a fix; AltStore and SideStore were unfixed as of 2026-10-20. Hit on the first install here with Trust done, so it isn't a pairing issue.
- **iloader works (2026-10-23):** where Sideloadly failed with -80046, iloader ("Import any IPA", uses the iTunes drivers) signed in and installed. A first -22406 ("Enter the correct password") was a mistyped password, which also proved that the sign-in block had been passed.
- **Fallback without third-party sign-in:** a paid Apple Developer account ($99/yr) lets CI sign the app and ship it through TestFlight (App Store Connect API key + distribution cert as secrets). No PC sign-in, builds valid 90 days. Not implemented.
- **Claude sandbox limits:** a Claude session can trigger runs and read run status, but it can't download Actions logs or artifacts (blocked by its network allowlist). Download from github.com instead.
- **A14 device:** the app throttles depth and photo capture under thermal or memory pressure, which happens sooner than on newer phones.

## Remaining tasks

- [x] Push these files to the fork.
- [x] Actions enabled on the fork (it was on by default).
- [x] Run the workflow: the `xcode-27` label works and the build succeeded.
- [ ] Decide on `Deploy viewer`: disable it, or enable Pages for M4. It fails on every push until then.
- [ ] Record the Xcode and SDK versions from the *Show toolchain* step of run #1.
- [x] Install (via iloader) and confirm Developer Mode, trust, camera permission and first scan.
- [ ] Find out how iloader re-signs before day 7 (manual re-install, or whether it has auto refresh). Sideloadly's daemon can't sign in right now.
- [x] M4 quick test: one room processed (private).
- [ ] M5: choose where processing lives. Options: (a) keep using a Claude workspace per room, (b) local on Windows (WSL for the zsh scripts + Node 22 + Python/Pillow), (c) a **private** repo for personal rooms (a fork of a public repo can't be private).
- [ ] Generalize the room-specific prep script (ceiling heightfield, scan export, mesh probes) into a reusable tool.
- [x] Detail pass test: small-object hints from mesh blobs + photo voting for the light (private).

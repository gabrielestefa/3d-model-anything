# Install the iPhone app without a Mac (Windows + Sideloadly)

Build the app in the cloud, sign it on Windows with a free Apple ID, run it on a LiDAR iPhone.

| Item | Value |
|---|---|
| Phone | iPhone 12 Pro (LiDAR, so scanning works) |
| Computer | Windows 11 (no Mac needed) |
| Bundle ID | `com.gabrielestefanelli.housecapture` (never change it) |
| Build | GitHub Actions workflow `build-ipa.yml`, unsigned Release `.ipa` |
| Signing | Sideloadly + free Apple ID (re-sign every 7 days) |

---

## 1. Build the .ipa (cloud)

1. **First time only:** open the fork's **Actions** tab and enable workflows. GitHub disables them on new forks.
2. **First time only:** disable the inherited **Deploy viewer** workflow (Actions → Deploy viewer → `⋯` → Disable workflow). It deploys to GitHub Pages, which isn't set up on this fork, so it fails on every push.
3. **Start the build:** Actions → **Build iPhone app (unsigned IPA)** → **Run workflow**. Keep the default bundle ID.
4. **Download:** when the run is green (about 5–15 min), download the `HouseCapture-unsigned-ipa-runN` artifact from the run page. It's a zip; extract `HouseCapture-unsigned.ipa` and keep it on the PC, because you'll reuse it every week.

Artifacts expire after 90 days. To get a fresh copy, re-run the workflow.

## 2. Prepare Windows (once)

1. **Apple software:** install **iTunes** and **iCloud** using the download links on [sideloadly.io](https://sideloadly.io). Use those links, not the Microsoft Store versions, because Sideloadly doesn't detect the phone through the Store versions.
2. **Sideloadly:** install it from **sideloadly.io** only. There are fake copies on other sites.
3. **Connect the phone:** plug the iPhone in with a cable, unlock it, and tap **Trust This Computer**.

## 3. Install on the iPhone

1. **Load the app:** open Sideloadly, drag `HouseCapture-unsigned.ipa` onto it, and enter your Apple ID.
   - Leave the bundle ID unchanged in the advanced options.
   - You can use a secondary Apple ID if you prefer not to give a third-party tool your main one.
2. **Sign and install:** click **Start** and enter the two-factor code when prompted.
3. **Developer Mode:** on the iPhone, go to Settings → Privacy & Security → **Developer Mode** → On, restart, and confirm. The toggle only appears after the first developer-signed app is installed.
4. **Trust your account:** Settings → General → **VPN & Device Management** → your Apple ID → **Trust**.
5. **Launch:** open **3D Model Anything** and allow camera access.

## 4. Every 7 days (free Apple ID)

The signature expires after 7 days. The app stays on the phone but won't open until you re-sign it. Your scans are kept.

### Automatic (Sideloadly daemon, recommended)

Turn on **auto refresh** when you sideload. The Sideloadly daemon on the PC then re-signs the app when it nears expiry, but only at moments when it can reach the phone:

- **The PC:** on, with the Sideloadly daemon running.
- **The phone:** plugged in by USB, **or** on the same Wi-Fi as the PC.
- **Wi-Fi refresh setup (once):** connect by USB and enable Wi-Fi sync in iTunes (device → Summary → *Sync with this iPhone over Wi-Fi*).
- **During a Wi-Fi refresh:** the phone's screen must be on. If the phone isn't detected, opening iTunes on the PC can help.

If the phone and PC don't meet for about 7 days (travel, PC off), the app expires. Re-sign it manually as below.

### Manual

1. Connect the iPhone to the PC.
2. In Sideloadly, use the **same** `.ipa`, Apple ID and bundle ID, then click **Start**. This installs over the existing app.

To install a newer build, use the same steps with the new `.ipa`. Sideloadly overwrites the old version and keeps it enrolled for auto refresh.

> ⚠️ **Never delete the app before re-signing.** The scans live inside the app's storage, and deleting the app deletes them.

**Free account limits:** 3 sideloaded apps active per device, and 10 new app IDs per 7 days. A paid Apple Developer account ($99/yr) extends the signature to 1 year.

## 5. Back up and move scans

- **On the phone:** open a room → *Share room (.zip)* → **Save to Files**, or browse Files → On My iPhone → **3D Model Anything**.
- **To the PC:** the app has iTunes File Sharing enabled, so its documents appear in iTunes on Windows (device → File Sharing). This is also how scans get to the PC for the `scan-to-threejs` reconstruction step.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Login failed (-80045/-80046): This action cannot be completed at this time` | Apple-side problem, not your password or Trust. Since 2026-10-12 Apple rejects sign-ins from many free sideloading tools. Update Sideloadly and try **once**; repeated attempts get throttled. Use the 2FA code from a trusted Apple device, not SMS. If it still fails, try **iloader** ("Import any IPA", sign-in fixed in v2.3.7+, use its built-in anisette server), or the paid-account route in FORK-NOTES. |
| Sideloadly doesn't see the phone | Make sure iTunes and iCloud are the non-Store versions, then unlock the phone, re-plug it and tap Trust. |
| "Untrusted Developer" on launch | Repeat step 3.4 (VPN & Device Management → Trust). |
| App icon present but won't open | The signature has expired. Re-sign as in step 4. |
| A new, empty app appeared next to the old one | It was signed with a different bundle ID. Re-sign with `com.gabrielestefanelli.housecapture`. |
| Workflow fails: no runner matches `xcode-27` | In `build-ipa.yml`, set `runs-on: macos-26` and add a step `sudo xcode-select -s /Applications/Xcode_26*.app`. Xcode 26 or newer is required. |
| Scanning slows or pauses on long scans | This is the app's thermal and memory protection, which kicks in sooner on the A14 chip. Finish the room and continue in a new scan. |

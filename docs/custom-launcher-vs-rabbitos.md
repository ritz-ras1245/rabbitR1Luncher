# Custom launcher / public docs vs stock rabbitOS

Contrast between **this codebase** (`rabbitR1Luncher` / CarrotOS) and **Rabbit’s public documentation**. Only pages that were actually opened for this write-up are linked. Anything not verified is labeled.

## What this repo is

From `README.md` in this tree:

- Compose HOME launcher for the Rabbit R1 (`com.r1.launcher`).
- Intended to ship as the single home app in **CarrotOS** (custom LineageOS 21 / Android 14 GSI), replacing Launcher3 under `/system/app/R1Launcher/`, with no status bar / nav bar / lock screen chrome.

From CarrotOS README (opened via raw GitHub for `khalifa007/carrotOS`):

- Single-app kiosk; launcher is the only home.
- Unauthenticated root shell on `127.0.0.1:1337`; README states only the bundled launcher runs and warns not to install untrusted third-party apps.

From this tree’s code (see [how-the-launcher-works.md](./how-the-launcher-works.md)):

- Hardcoded synthetic “cards” (Settings, OpenClaw, Messages, Terminal, Hermes, Meetings, Translator) plus discovered launcher activities.
- No stock rabbitOS “creations” stack, no Rabbit cloud agent UI in this source.

## License / reuse (can public docs or a custom launcher be based on this?)

**Not verified — no license file found.**

- There is **no** `LICENSE` / `COPYING` (or similarly named) file at the repo root in this checkout.
- `README.md` does not state an SPDX license in the portion read for this note.
- Therefore: **whether you may republish docs, relicense, or ship a derivative custom launcher for public support cannot be affirmed from this tree alone.** Treat redistribution and “official support” claims as blocked until the upstream author publishes clear terms. Forking on GitHub for private study is a separate question from publishing support docs or redistributing binaries.

Technical feasibility of a *custom* launcher based on the code (engineering, not legal): yes, the project builds as a normal Android app (`assembleRelease`) and can be sideloaded onto an existing CarrotOS device per README — but that is not a license grant.

## Stock rabbitOS (from Rabbit pages actually opened)

### Cards and navigation

Opened: [How to interact with your rabbit r1](https://www.rabbit.tech/support/article/use-rabbit-r1)

- Home → swipe/scroll opens a **card stack**.
- Tap a card to open; back exits; active cards can be dismissed.
- Settings, r-cade, quick settings (brightness/volume/camera/keyboard/lock), music controls, etc. are described as stock UX — not as Android `CATEGORY_LAUNCHER` tiles.

### Creations

Opened: [How to use r1 creations](https://www.rabbit.tech/support/article/how-to-use-r1-creations), [Creations gallery](https://www.rabbit.tech/creations)

- Creations are AI-generated mini-apps for r1; install via Creations card (QR or public tab); installed creations appear as cards.
- Documented technical limits include: small screen, limited performance/storage, one creation at a time, and **“creations cannot currently access speech-to-text (STT) or make anything with a hosted backend.”**

### Agents / OpenClaw / Hermes (stock path)

Opened: [How to use third-party agents on rabbit r1](https://www.rabbit.tech/support/article/agents-on-rabbit-r1), [Updates changelog](https://www.rabbit.tech/updates)

- Stock path uses **rabbit agent** on a computer (install via OS3 settings → rabbit agents), then r1 agent screens (swipe, refresh, PTT).
- OpenClaw, Hermes Agent, Claude Code are described as third-party; Rabbit does not support setting them up.
- Changelog notes OpenClaw protocol v4 via rabbit agent, Hermes on r1, terminal mode, creations gallery updates (rabbitOS 2.x / OS3 milestones).

Opened (marketing / product): [rabbit.tech/updates](https://www.rabbit.tech/updates) OS3 blurb — settings for r1 (linking, lost mode, magic gallery, magic voice, **developer mode**) move under OS3.

### Developer mode / Unlock

Attempted: [https://hole.rabbit.tech](https://hole.rabbit.tech) — **not verified** in this session (HTTP 403 / CloudFront block from the fetch environment). Do not document Unlock behavior from that page here. Device Unlock remains out of scope for this story.

### OpenClaw install script URL from household notes

Attempted: `https://rabbit.tech/r1-openclaw.sh` — **not verified** (fetch failed / 403). Do not treat that URL as live from this write-up.

## Contrast table

| Topic | Stock rabbitOS (public docs) | This launcher / CarrotOS |
|-------|------------------------------|---------------------------|
| Home UX | Card stack, Rabbit agent, creations, magic features | Clock home + apps list of Android launcher activities + fixed synthetics |
| Add capability without flashing | Creations (QR/gallery); connect computers via rabbit agent | No creations system in this repo; add APK (Real) or change Kotlin for new synthetics; configure OpenClaw/Hermes URLs on existing cards |
| OpenClaw | Via rabbit agent + r1 agent pages; Rabbit unsupported setup | First-class in-app OpenClaw QR/chat wired in Kotlin |
| Hermes | Via rabbit agent path in support article | First-class in-app Hermes config/chat |
| Hosted backend from “mini apps” | Creations: no hosted backend (per support article) | Full Android app: can use OkHttp/WebSocket to user-configured servers (subject to NSC TLS rules) |
| Kiosk | Not described as single-app AOSP kiosk in the pages opened | CarrotOS: single-app kiosk, hide system chrome |
| Official support | Rabbit support articles | Community ROM + community launcher; no Rabbit support implied |

## Answers to the ask

1. **Can a custom launcher be based on this repo?**  
   **Engineering:** yes as a starting point (Compose HOME app, clear structure). **Legal / public redistribution:** **unknown** — no license file verified. **Product:** a custom launcher on CarrotOS is the intended design of that ROM; on stock rabbitOS, replacing the official launcher was **not verified** from Rabbit’s public docs opened here (stock docs describe their card stack, not AOSP home replacement).

2. **Can public docs/support be based on this repo?**  
   You can write **accurate technical notes** about what the code does (as in this `docs/` set). Calling them official Rabbit support, or republishing substantial third-party code/docs without a license, is **not cleared**. Prefer linking upstream repos and Rabbit’s own pages for stock behavior.

3. **Stock vs this code for household LAN hooks**  
   Stock creations cannot carry a hosted backend (Rabbit support). This launcher *can* talk to networked backends (OpenClaw/Hermes/cloud APIs) but LAN cleartext is denied. Household HTTP services therefore align better with **TLS + OpenClaw/Hermes/custom panel** on CarrotOS than with stock creations — with the legal/kiosk caveats above.

## Sources opened for this note

- https://www.rabbit.tech/support/article/use-rabbit-r1
- https://www.rabbit.tech/support/article/how-to-use-r1-creations
- https://www.rabbit.tech/creations
- https://www.rabbit.tech/support/article/agents-on-rabbit-r1
- https://www.rabbit.tech/updates
- https://github.com/khalifa007/carrotOS (README via raw.githubusercontent.com)
- This repository’s `README.md`, `AppEntry.kt`, `LauncherActivity.kt`, `network_security_config.xml`

## Not verified

- Full text of https://hole.rabbit.tech (blocked)
- https://rabbit.tech/r1-openclaw.sh (blocked/failed)
- https://www.rabbit.tech/r1-user-guide (listed in the plan; not required after the support articles above covered cards/creations — **not opened** in this pass)
- Rabbit’s full user agreement / OpenClaw legal pages beyond what the agents support article summarizes
- Whether GitHub’s “public repo” default implies a license (it does not replace an explicit LICENSE)

# Household LAN projects vs this launcher / CarrotOS

Requirements-oriented mapping only. No patches. APIs that are not defined in **this** repository are marked **unknown** (or cited only at the level of a sibling project’s public README role — never invented endpoints).

Companion reading: [how-the-launcher-works.md](./how-the-launcher-works.md).

## Constraints from this repo + CarrotOS README

From **rabbitR1Luncher** (this tree):

- Apps list = discovered `CATEGORY_LAUNCHER` activities + fixed synthetics (Messages, OpenClaw, Terminal, Hermes, Translator, Meetings, Settings). No user-added URL card without a code change.
- Outbound cleartext denied except `localhost` / `127.0.0.1` / `10.0.2.2`. LAN `http`/`ws` needs TLS or an explicit NSC domain (resource change).
- First-class remote agent surfaces: **OpenClaw** (pair + chat) and **Hermes** (config URL + chat). Both are in-app panels, not generic browsers.

From **CarrotOS** README (fetched from `khalifa007/carrotOS` main; not vendored here):

- Custom Android 14 ROM with a **single-app kiosk** launcher baked in.
- “The R1 launcher is the only home — status bar, nav bar, and lock screen are hidden.”
- Unauthenticated root shell on `127.0.0.1:1337` for launcher hardware helpers; README says since only the bundled launcher runs, do **not** install untrusted third-party apps.
- Launcher source pointed at `khalifa007/rabbitR1Luncher`.

Implication: treating CarrotOS as the host OS, “just sideload another APK for a dashboard” conflicts with the published kiosk security caveat. Prefer hooks that use **existing** launcher surfaces (OpenClaw / Hermes / TLS HTTP clients you add in a future change) rather than random third-party apps.

## Shared hook patterns (requirements, not designs)

| Pattern | What would have to exist | Blockers in this tree |
|---------|--------------------------|------------------------|
| A. New synthetic `AppEntry` + panel or Intent | Code change to sealed class, `loadApps`, `AppsPanel`, `launchApp` | Explicitly out of scope for this story |
| B. Configure OpenClaw gateway on LAN | Gateway that speaks the protocol this app’s `GatewaySession` expects; reachable as `wss` (or `ws` only if NSC allows — today LAN cleartext `ws` is blocked) | Pairing/setup out of scope; exact protocol **unknown** beyond URL scheme helpers in this repo |
| C. Configure Hermes server URL | Server that matches Hermes client expectations + TLS (or loopback) | Hermes wire protocol **unknown** from this repo alone |
| D. Install another launcher activity APK | Appears as `AppEntry.Real` | Conflicts with CarrotOS “don’t install untrusted apps”; no generic WebView card |
| E. Use inbound companion `:8080` | Phone/browser → device | Wrong direction for “open Shelf Sense on R1”; does not browse LAN services |

## Per project

### Shelf Sense (inventory + receipt import; LAN HTTP/HTTPS)

- **In this launcher repo:** no Shelf Sense client, package name, or API references found → service contract **unknown** here.
- **Hook potential:** High priority household service, but **not** callable as a card today. A plain `http://<lan-host>:…` client call from the app would hit NSC cleartext denial unless the service is **HTTPS** (or NSC is changed — a resource edit).
- **Requirements (not patches):** Either (1) TLS endpoint the device trusts + a new in-app panel or OpenClaw/Hermes tool that calls it, or (2) an OpenClaw gateway on a LAN computer that already talks to Shelf Sense and is paired into the existing OpenClaw card, or (3) a dedicated `AppEntry` + UI (code change). CarrotOS does not document a “local services” card catalog.

### airplay-status (Pi AirPlay dashboard)

- **In this launcher repo:** no references → API **unknown**.
- **Hook potential:** Same as any LAN dashboard: no WebView/URL card; outbound cleartext blocked for LAN HTTP. Would need HTTPS + a UI surface (new card, Hermes/OpenClaw bridge, or trusted APK — last option fights CarrotOS guidance).

### haul-capture (Costco vision via household LLM gateway)

- **In this launcher repo:** no haul-capture references → API **unknown**.
- **Related hardware fact from handoff (not re-measured here):** r1 camera exists; OpenClaw already has camera panels. Whether haul-capture’s album/upload flow belongs on-device is a product decision outside this repo.
- **Hook potential:** Indirect only. Vision calls in sibling demos go through a household LLM gateway (Ollama-shaped) — that gateway’s HTTP API is **not** defined in this launcher tree. Requirements: a client that can capture/upload in the shape haul-capture expects, plus TLS or loopback path to the gateway; or OpenClaw tools on a paired computer that already runs haul-capture. Do not assume the Hermes or OpenClaw panels speak haul-capture’s protocol.

### Household LLM gateway

- **In this launcher repo:** no references → enroll/token/API **unknown** here.
- **Hook potential:** Natural backend for on-device features, but NSC blocks cleartext LAN HTTP. Requirements: HTTPS (or NSC domain exception — code/resources), plus either Hermes compatibility (**unknown**), OpenClaw tools on a host that already holds a gateway token, or a new panel. Do not invent `/v1/…` paths in this doc from the launcher tree.

### OpenClaw gateway the r1 can pair to

- **In this launcher repo:** first-class. `AppEntry.OpenClaw` → QR pairing or chat; `OpenClawPrefs.gatewayUrl` + `GatewaySession` URL normalization (`ws`/`wss`/`http`/`https`).
- **CarrotOS:** screenshots/docs show OpenClaw chat as a primary surface of the bundled launcher.
- **Hook potential:** Primary recommended path for “r1 talks to a computer that then calls household services.” Requirements: run a gateway the launcher can pair with; prefer **`wss`/`https`** for off-loopback hosts because cleartext LAN `ws`/`http` is NSC-blocked. Stock Rabbit OpenClaw setup and pairing scripts are **out of scope** for this story; Rabbit’s public support pages describe OpenClaw as user-set-up / unsupported by Rabbit (see contrast doc). Exact match between this app’s protocol and a given OpenClaw build: **unknown** without a live pairing test.

### cursor-usage-status

- **In this launcher repo:** no references → API **unknown**.
- **Hook potential:** Local usage dashboard (HTTP). Same pattern as airplay-status: needs TLS + a UI surface, or an OpenClaw/host-side tool. Not present as a card.

### always-on-ship

- **In this launcher repo:** no references → API **unknown**.
- **Hook potential:** Supervisor/kiosk for child dashboards on a server, not an R1 surface by itself. Could expose HTTPS status pages that a future card or OpenClaw tool opens; nothing in this repo wires it today.

## Summary

| Project | Hook from this launcher today | Hook from CarrotOS today | Gap (requirements) |
|---------|-------------------------------|--------------------------|--------------------|
| Shelf Sense | No | No dedicated card | TLS + UI or OpenClaw bridge; API unknown in this repo |
| airplay-status | No | No | Same |
| haul-capture | No | Camera exists for OpenClaw panels only | Client + gateway contract unknown here |
| Household LLM gateway | No direct | No | TLS + client; API unknown here |
| OpenClaw gateway | Yes (pair/chat) | Bundled launcher includes it | Prefer WSS/TLS; pairing setup out of scope |
| cursor-usage-status | No | No | Same as other HTTP dashboards |
| always-on-ship | No | No | Server-side; no R1 card |

**unknown** means: not verified from `rabbitR1Luncher` sources. Sibling READMEs on the workshop LAN describe those services’ own ports and roles, but those contracts must be confirmed in their repos before any implementation.

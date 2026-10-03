# How this launcher works

Understanding of `ritz-ras1245/rabbitR1Luncher` at commit `59bf707` (`release: 1.1.10`). Facts below are from the code in this tree. Leads that disagreed with the code were discarded.

Package: `com.r1.launcher`. Single `LauncherActivity` (Compose HOME), registered as HOME + LAUNCHER in `AndroidManifest.xml`.

## AppEntry (sealed)

```5:14:app/src/main/java/com/r1/launcher/AppEntry.kt
sealed class AppEntry {
    data class Real(val info: ResolveInfo) : AppEntry()
    object Settings : AppEntry()
    object OpenClaw : AppEntry()
    object Messages : AppEntry()
    object Terminal : AppEntry()
    object Hermes : AppEntry()
    object Meetings : AppEntry()
    object Translator : AppEntry()
}
```

There is no URL / bookmark / custom-card subtype. Adding a new synthetic card requires a new sealed member and matching `when` branches (UI + `launchApp`).

## How the card list is built

`loadApps()` runs off the main thread, queries installed launcher activities, filters, sorts, then appends hardcoded synthetics:

```825:851:app/src/main/java/com/r1/launcher/LauncherActivity.kt
    private fun loadApps() {
        // ...
        Thread {
            val main = Intent(Intent.ACTION_MAIN).addCategory(Intent.CATEGORY_LAUNCHER)
            val found = pm.queryIntentActivities(main, 0)
                .filter { it.activityInfo.packageName != ownPkg && it.activityInfo.packageName != "com.android.settings" }
                .sortedBy { it.loadLabel(pm).toString().lowercase(Locale.getDefault()) }
            ui.post {
                state.apps.clear()
                found.forEach { state.apps.add(AppEntry.Real(it)) }
                state.apps.add(AppEntry.Messages)
                state.apps.add(AppEntry.OpenClaw)
                state.apps.add(AppEntry.Terminal)
                state.apps.add(AppEntry.Hermes)
                state.apps.add(AppEntry.Translator)
                state.apps.add(AppEntry.Meetings)
                state.apps.add(AppEntry.Settings)
                state.appsLoaded = true
                // ...
            }
        }.start()
    }
```

Order on the apps panel:

1. Every other package with `ACTION_MAIN` + `CATEGORY_LAUNCHER` (label A–Z), excluding this package and `com.android.settings`
2. Then fixed synthetics: Messages, OpenClaw, Terminal, Hermes, Translator, Meetings, Settings (Settings last)

`AppsPanel` renders `state.apps` in a `LazyColumn` of `AppCard`s; tap calls `onAppClick(idx)` → `launchApp(idx)`. A decorative `FolderTray` (“carrotos” version label) sits under the list; it is not an `AppEntry` and is not launched.

## What selecting each AppEntry does

Dispatch is entirely in `launchApp`:

```1151:1218:app/src/main/java/com/r1/launcher/LauncherActivity.kt
    override fun launchApp(idx: Int) {
        when (val entry = state.apps.getOrNull(idx)) {
            is AppEntry.Real -> {
                val info = entry.info
                launchTone()
                runCatching {
                    val i = Intent().apply {
                        setClassName(info.activityInfo.packageName, info.activityInfo.name)
                        action = Intent.ACTION_MAIN
                        addCategory(Intent.CATEGORY_LAUNCHER)
                        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED
                    }
                    startActivity(i)
                    state.back()
                }
            }
            AppEntry.Settings -> {
                if (!ensureWriteSettingsGrant()) return
                seedSettingsLevels()
                state.openSettings()
                selectTone()
            }
            AppEntry.OpenClaw -> {
                selectTone()
                if (openClawPrefs.hasPairing()) {
                    refreshVoiceKeyState()
                    openClawStartSession()
                    state.openOpenClawChat()
                } else {
                    state.qrError = null
                    ensureCameraPerm()
                    state.openOpenClawQr()
                }
            }
            AppEntry.Messages -> {
                selectTone()
                state.openMessages()
                if (ensureSmsPerm()) loadSmsConversations()
                else state.smsError = "permission required"
            }
            AppEntry.Terminal -> {
                selectTone()
                state.openTerminal()
            }
            AppEntry.Hermes -> {
                selectTone()
                hydrateHermesStateFromPrefs()
                if (hermesPrefs.hasConfig()) {
                    state.openHermesChat()
                    hermesTestConnection()
                } else {
                    state.openHermesConfig(fromChat = false)
                }
            }
            AppEntry.Meetings -> {
                selectTone()
                transcriberOpen()
            }
            AppEntry.Translator -> {
                selectTone()
                hydrateTranslatorStateFromPrefs()
                // First run → wizard (pick source/target, set a key). After that
                // it goes straight to the focused translation screen.
                if (translatorPrefs.onboarded) state.openTranslator()
                else state.openTranslatorOnboarding()
            }
            null -> Unit
        }
    }
```

| AppEntry | Kind | Selecting it |
|----------|------|----------------|
| `Real` | External app | Builds an `Intent` with `setClassName(package, activity)`, `ACTION_MAIN`, `CATEGORY_LAUNCHER`, `FLAG_ACTIVITY_NEW_TASK \| FLAG_ACTIVITY_RESET_TASK_IF_NEEDED`, then `startActivity`. Leaves the apps panel via `state.back()`. |
| `Settings` | In-app panel | Requires `WRITE_SETTINGS` grant path; seeds levels; `state.openSettings()` → Compose settings stack (`Panel.SETTINGS` and children). Not Android Settings (`com.android.settings` is filtered out of the real list). |
| `OpenClaw` | In-app panel | If `openClawPrefs.hasPairing()`: refresh voice key, `openClawStartSession()`, `state.openOpenClawChat()`. Else: clear QR error, `ensureCameraPerm()`, `state.openOpenClawQr()`. Gateway URL lives in prefs; `GatewaySession` accepts `ws://` / `wss://` / `http://` / `https://` forms (see `openclaw/GatewaySession.kt`). |
| `Messages` | In-app panel | `state.openMessages()`; loads SMS conversations if SMS permission is granted. |
| `Terminal` | In-app panel | `state.openTerminal()` only (no external terminal app Intent here). |
| `Hermes` | In-app panel | Hydrate prefs; if `hermesPrefs.hasConfig()` open chat and test connection; else open Hermes config. User-supplied server URL + API key in prefs (`HermesPrefs` / `HermesConnection`) — not a free-form “app card.” |
| `Meetings` | In-app panel | `transcriberOpen()` → transcriber / meetings Compose panels. |
| `Translator` | In-app panel | Hydrate prefs; onboarded → translator UI; else onboarding. |

Panel enum (home, apps, and all in-app surfaces) is in `LauncherState.kt` (`enum class Panel { HOME, … }`). Synthetics navigate that enum; they do not start separate APK activities.

## Can a user add a URL or a card without a code change?

**No** for a new apps-list card or arbitrary URL tile:

- The sealed `AppEntry` set is fixed in source.
- `loadApps()` only discovers `CATEGORY_LAUNCHER` activities plus the hardcoded synthetics.
- There is no prefs-driven “bookmark card,” no user-editable apps list, and no WebView `AppEntry`.

**Partial exceptions (not general URL cards):**

- **OpenClaw** and **Hermes** let a user configure a **gateway / server URL** (and credentials) after opening those existing cards (QR or config UI). That does not add a new card; it configures the existing synthetic.
- **Installed third-party apps** with a launcher activity appear as `AppEntry.Real` after install + `loadApps()` refresh — that is “add an APK,” not “add a URL.” CarrotOS README separately warns against installing untrusted apps on the kiosk ROM (see sibling doc).

## Network cleartext policy (outbound client)

```1:23:app/src/main/res/xml/network_security_config.xml
<network-security-config>
    <!-- Default: require TLS for every OUTBOUND connection. ...
         NOTE: this policy governs the app's outbound *client* sockets only. The
         embedded web companion server (inbound NanoHTTPD on :8080) is NOT
         affected and still serves plain http to the phone on the LAN. -->
    <base-config cleartextTrafficPermitted="false" />
    <!-- Loopback exceptions: ... A backend reached over a LAN
         IP via http/ws (e.g. a self-hosted gateway on another box) must use TLS
         or be added here as an explicit <domain> entry. -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">localhost</domain>
        <domain includeSubdomains="true">127.0.0.1</domain>
        <domain includeSubdomains="true">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

Implications:

- Outbound `http://` / cleartext `ws://` to a **LAN host IP or hostname** is blocked by default.
- Cleartext is allowed only to `localhost`, `127.0.0.1`, and emulator host `10.0.2.2`.
- LAN backends for Hermes/OpenClaw (or any OkHttp/WebSocket client in this app) need **TLS**, or a **code/resource change** adding an explicit `<domain>` exception.
- Inbound companion server on **:8080** (`R1WebServer`, NanoHTTPD) is separate: phones on the LAN can still hit that plain HTTP server; the NSC does not govern inbound.

Manifest wires this via `android:networkSecurityConfig="@xml/network_security_config"` on the application element.

## Not verified here

- Exact OpenClaw / Hermes wire protocols beyond URL scheme handling and “pairing / hasConfig” gates (out of scope for this note).
- Runtime behavior on a flashed CarrotOS device (docs only; no flash).
- Whether AGENTS.md / CLAUDE.md match current `loadApps` order — those notes mentioned a `Claude` synthetic; **this tree has no `AppEntry.Claude`**. Prefer the Kotlin above.

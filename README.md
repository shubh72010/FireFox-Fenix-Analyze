#  Firefox Fenix — Deep Codebase Analysis

[![License: MPL 2.0](https://img.shields.io/badge/License-MPL_2.0-blue.svg)](https://opensource.org/licenses/MPL-2.0)
[![Fenix](https://img.shields.io/badge/Fenix-Android-FF7139?logo=firefox&logoColor=white)](https://github.com/mozilla-mobile/firefox-android)
[![GeckoView](https://img.shields.io/badge/GeckoView-Engine-00A7E0?logo=mozilla&logoColor=white)](https://mozilla.github.io/geckoview/)
[![Lines](https://img.shields.io/badge/Docs-5.2k%20lines-2ea44f)]()

> A comprehensive, code-backed analysis of the Firefox Fenix Android browser — architecture, subsystems, data flow, and implementation guidance. Designed as a developer handbook for understanding or rebuilding major parts of Fenix.

---

##  Structure

```
FirefoxAnalyze/
├── 00-overview/              Project map & module relationships
├── 01-architecture/          Architecture principles & project structure
├── 02-build-system/          Gradle build, variants, plugins
├── 03-startup/               Full startup timeline & session restore
├── 04-geckoview/             GeckoView integration & browser session lifecycle
├── 05-tabs/                  Tab management, tray, strips
├── 06-memory/                Memory pressure handling & caches
├── 07-navigation/            Navigation graph, deep links, Custom Tabs
├── 08-ui/                    Compose/Fragment hybrid, theming
├── 09-state-management/      Multi-store Redux architecture
├── 10-storage/               Places, Room, DataStore, JSON
├── 11-history-bookmarks/     History & bookmark storage (Places)
├── 12-downloads/             Download service, middleware, UI
├── 13-privacy-security/      TP, DoH, HTTPS-only, FPP, GPC
├── 14-permissions/           Permission request flow & bridges
├── 15-extensions/            WebExtensions, AMO, built-in add-ons
├── 16-passkeys-webauthn/     FIDO2/WebAuthn 4-layer architecture
├── 17-sync/                  Firefox Account, engines, push
├── 18-networking/            HTTP client, DoH, proxy, security protocols
├── 19-performance/           Startup metrics, profiling, completeness
├── 20-telemetry/             Glean, middleware, metric timing
├── 21-testing/               Unit, instrumented, e2e, benchmarks
├── 22-settings/              Preference system, custom types
├── 23-private-browsing/      Mode management, locking, themes
├── 24-implementation-notes/  Cross-cutting guidance & feature map
└── README.md                 You are here
```

---

##  Deep-Dives

| Topic | What's Covered |
|-------|----------------|
| **GeckoView Integration** | Runtime lifecycle, session delegates, process management, rendering pipeline, crash handling |
| **Browser Session** | `EngineSession` lifecycle, state save/restore, speculative sessions, crash recovery, process-kill resilience |
| **State Management** | Redux over `BrowserStore` + `AppStore` — 7 diagrams-worth of middleware chains, reducer patterns, async flows |
| **Tab Lifecycle** | Creation → lazy loading → suspension → eviction, tab tray/strips, session persistence, auto-close |
| **Memory Management** | `onTrimMemory` tiers, 5-level cache eviction strategy, `onLowMemory`, disabled auto-suspension (#25517) |
| **Passkeys / WebAuthn** | Full 4-layer architecture — Credential Manager / GMS bridge, `ConditionalMediation`, dual RP ID, activity-result dance, attestation |
| **Sync** | FxA state machine, push-registration, 10 sync engines, send tab protocol, collection sync vs incremental |
| **Private Browsing** | Mode middleware, per-session locking, `PrivateBrowsingStorage`, multi-window isolation, themed indicator |
| **Build System** | `app/build.gradle.kts`, AGP 8.x, `variantFilter`, GeckoView channel pinning, 6 `mach build` tiers |
| **Startup** | Timeline from `attachBaseContext` → `firstComposition`, splash → restore → warm, each phase timed via Glean |

---

##  Implementation Guidance

Each document answers: *If I had to implement this elsewhere, how would I do it?*

The **implementation guide** (`24-implementation-notes/implementation-guide.md`) synthesizes cross-cutting patterns:
- Adapter/Facade pattern for GeckoView abstraction
- Multi-store Redux with middleware chains
- Nested lifecycle scoping (Fenix → Session → View)
- Places-based storage with sync-native schemas
- Middleware-driven telemetry/analytics
- How to structure permissions, credentials, and downloads in another Android browser

The **feature map** (`24-implementation-notes/feature-map.md`) indexes every feature by domain, location, key classes, and migration path.

---

##  Each File Follows a Template

Every document covers these sections (when applicable):

```
Purpose
Where it lives
Main classes
How it works
Data flow
Lifecycle
Threading
Storage / Persistence
Integration points
Implementation notes (how to do it elsewhere)
Edge cases
Relevant paths
Related features
```

---

##  How to Use This

- **New contributors**: Start with `01-architecture/architecture.md` → `01-architecture/project-structure.md` → `03-startup/startup.md`
- **Adding a feature**: Check `24-implementation-notes/feature-map.md` for related subsystems, then read the relevant domain docs
- **Porting Fenix patterns**: Read `24-implementation-notes/implementation-guide.md` first, then the individual subsystem docs for code-level detail

---

##  Source Code

All analysis was performed against the [firefox-android](https://github.com/mozilla-mobile/firefox-android) repository at the Fenix layer (`mobile/android/fenix/`), android-components (`mobile/android/android-components/`), and GeckoView (`mobile/android/geckoview/`).

---

##  License

This documentation is provided under the [Mozilla Public License 2.0](https://opensource.org/licenses/MPL-2.0).

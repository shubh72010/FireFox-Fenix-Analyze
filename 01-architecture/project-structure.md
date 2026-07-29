# Project Structure

## Purpose
Complete directory map with descriptions of every major package in the Fenix app and android-components library.

## Fenix App Packages

### Top-level (`org/mozilla/fenix/`)
| File | Description |
|------|-------------|
| `FenixApplication.kt` | Application class, startup initialization, `onTrimMemory` |
| `HomeActivity.kt` | Single Activity, navigation host, intent processing |
| `NavHostActivity.kt` | Interface for activities with NavHost |
| `Config.kt` | Build configuration reading |
| `FeatureFlags.kt` | Boolean feature flags |
| `SecureFragment.kt` | Base Fragment with FLAG_SECURE |
| `AppRequestInterceptor.kt` | Engine request interceptor for app links |
| `BrowserDirection.kt` | Navigation direction for browser screen |

### Feature Packages
| Package | Description |
|---------|-------------|
| `addons/` | WebExtension add-ons management UI |
| `autofill/` | Android Autofill framework integration |
| `automotive/` | Android Auto support |
| `bookmarks/` | Bookmarks management UI and state |
| `browser/` | Browser screen, toolbar, session features |
| `collections/` | Tab collections (saved tab groups) |
| `components/` | Core wiring, service locator, app state management |
| `compose/` | Reusable Compose UI components |
| `crashes/` | Crash reporting integration |
| `customtabs/` | Custom tabs / trusted web activities |
| `datastore/` | Jetpack DataStore usage |
| `debugsettings/` | Developer/debug settings |
| `distributions/` | Distribution channel handling |
| `downloads/` | Download management UI and service |
| `exceptions/` | Site exceptions UI |
| `experiments/` | Nimbus experiment hooks |
| `extension/` | WebExtension Fenix integration |
| `gecko/` | GeckoView runtime provider |
| `historymetadata/` | History metadata service |
| `home/` | Home screen (Compose) |
| `intent/` | Intent processing |
| `ipprotection/` | IP Protection (Mozilla VPN) |
| `lifecycle/` | Lifecycle observers |
| `media/` | Media session integration |
| `messaging/` | In-app messaging system |
| `navigation/` | Navigation direction helpers |
| `nimbus/` | Nimbus experimentation SDK |
| `onboarding/` | First-run onboarding |
| `pbmlock/` | Private browsing lock |
| `perf/` | Performance monitoring |
| `push/` | Push notifications (AutoPush) |
| `search/` | Search engine management |
| `session/` | Session lifecycle management |
| `settings/` | Settings screens |
| `share/` | Share functionality |
| `shortcut/` | Home screen shortcuts |
| `snackbar/` | Snackbar system |
| `sync/` | Firefox Sync management |
| `tabgroups/` | Tab groups persistence and state |
| `tabhistory/` | Tab history UI |
| `tabstray/` | Tab tray (tab switcher) |
| `telemetry/` | Telemetry middleware |
| `theme/` | Theming system |
| `trackingprotection/` | Tracking protection |
| `translations/` | Page translations |
| `utils/` | Utilities |
| `wallpapers/` | Wallpaper system |
| `widget/` | Home screen widgets |
| `wifi/` | WiFi integration |

## Android-Components Library Structure

All under `mobile/android/android-components/`.

### Browser Components
| Component | Description |
|-----------|-------------|
| `browser/engine-gecko` | GeckoView Engine implementation |
| `browser/engine-gecko-beta` | GeckoView Beta Engine |
| `browser/engine-gecko-nightly` | GeckoView Nightly Engine |
| `browser/icons` | Website icon loading and caching |
| `browser/session-storage` | Tab session persistence |
| `browser/state` | BrowserState Redux store |
| `browser/thumbnails` | Tab thumbnail capture and storage |
| `browser/domains` | Domain autocomplete |
| `browser/storage-sync` | Places (history/bookmarks) storage |
| `browser/toolbar` | Browser toolbar component |
| `browser/engine-servo` | (Future) Servo engine implementation |

### Concept Components (Abstractions)
| Component | Description |
|-----------|-------------|
| `concept/engine` | Engine, EngineSession, EngineView abstractions |
| `concept/fetch` | HTTP client abstraction |
| `concept/storage` | Storage abstractions |
| `concept/toolbar` | Toolbar abstraction |
| `concept/awesomebar` | URL bar abstraction |
| `concept/push` | Push messaging abstraction |
| `concept/base` | Base abstractions (crash, sync) |

### Feature Components
| Component | Description |
|-----------|-------------|
| `feature/tabs` | Tab management use cases |
| `feature/session` | Session management, HistoryDelegate |
| `feature/search` | Search engine management |
| `feature/prompts` | Prompt dialogs (alerts, confirm, prompts) |
| `feature/sitepermissions` | Site permission prompts |
| `feature/downloads` | Download management |
| `feature/addons` | WebExtension add-ons management |
| `feature/autofill` | Android Autofill framework |
| `feature/media` | Media session feature |
| `feature/pwa` | Progressive Web Apps |
| `feature/readerview` | Reader view |
| `feature/contextmenu` | Context menus |
| `feature/findinpage` | Find in page |
| `feature/customtabs` | Custom tabs implementation |
| `feature/webauthn` | WebAuthn feature |
| `feature/syncedtabs` | Synced tabs display |
| `feature/top-sites` | Top sites (most visited) |
| `feature/recentlyclosed` | Recently closed tabs |
| `feature/webcompat` | Web compatibility features |
| `feature/webnotifications` | Web notification support |
| `feature/logins` | Login management |
| `feature/translations` | Page translations feature |
| `feature/ai` | AI feature controls |
| `feature/summarize` | Page summarization |

### Service Components
| Component | Description |
|-----------|-------------|
| `service/fxa` | Firefox Accounts integration |
| `service/glean` | Glean telemetry integration |
| `service/nimbus` | Nimbus experimentation |
| `service/pocket` | Pocket stories integration |
| `service/location` | Location service (MLS) |
| `service/mars` | MARS (top sites) integration |
| `service/merino` | Merino suggest API |
| `service/contile` | Contile ad integration |
| `service/digitalassetlinks` | Digital Asset Links verification |
| `service/sync-autofill` | Autofill sync (credit cards, addresses) |
| `service/sync-logins` | Login sync |
| `service/fxarelay` | Firefox Relay integration |

### Support Components
| Component | Description |
|-----------|-------------|
| `support/base` | Base utilities |
| `support/ktx` | Kotlin extensions |
| `support/locale` | Locale management |
| `support/rusthttp` | Rust HTTP configuration |
| `support/utils` | General utilities |
| `support/webextensions` | WebExtension support utilities |
| `support/remotesettings` | Remote Settings support |

### Library Components
| Component | Description |
|-----------|-------------|
| `lib/crash` | Crash reporting library |
| `lib/dataprotect` | Data protection (encrypted prefs) |
| `lib/state` | Redux store implementation |
| `lib/publicsuffixlist` | Public suffix list |
| `lib/integrity` | Play Integrity API |
| `lib/llm` | LLM/AI integration |
| `lib/ai` | AI feature controls storage |

## GeckoView Structure

All under `mobile/android/geckoview/`.

```
geckoview/
  src/main/java/org/mozilla/geckoview/
    GeckoRuntime.java        # Singleton process manager
    GeckoRuntimeSettings.java # Runtime configuration
    GeckoSession.java        # Single browsing session
    GeckoSessionSettings.java # Per-session settings
    GeckoView.java           # Android rendering View
    GeckoWebExecutor.java    # HTTP fetch via Gecko
    WebExtension.java        # WebExtension wrapper
    WebExtensionController.java # Extension lifecycle
    ContentBlockingController.java # Tracking protection
    StorageController.java   # Data storage management
    WebAuthnTokenManager.java # FIDO2/WebAuthn management
    PromptController.java    # Prompt routing
    Autocomplete.java        # Autofill integration
    CrashReporter.java       # Crash reporting
    MediaSession.java        # Media session support
    MemoryController.java    # Memory pressure handling
```

## Build Variants

Build types defined by `Config.channel`:
- **Debug**: Development, LeakCanary enabled, strict mode
- **Nightly**: Pre-release, unstable
- **Beta**: Release candidate
- **Release**: Production

Each variant can use different GeckoView engines (`engine-gecko`, `engine-gecko-beta`, `engine-gecko-nightly`), different Nimbus endpoints, and different AMO collection configurations.

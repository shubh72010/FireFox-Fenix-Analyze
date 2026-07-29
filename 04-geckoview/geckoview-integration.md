# GeckoView Integration

## Purpose
GeckoView is Mozilla's standalone web engine library for Android. Fenix wraps it through the android-components `Engine` abstraction layer. This document explains the complete integration architecture.

## Architecture Layers

```
Fenix App Code
    ↕ (Engine interface - concept/engine)
Android-Components Engine Bridge (browser/engine-gecko)
    ↕ (GeckoView Java API)
GeckoView Native Library (mobile/android/geckoview)
    ↕ (IPC - Gecko Child Process)
Gecko Rendering Engine (C++/Rust)
```

## Key Files by Layer

### Concept Layer (Engine Abstraction)
| File | Purpose |
|------|---------|
| `components/concept/engine/src/main/java/.../Engine.kt` | Engine interface |
| `components/concept/engine/src/main/java/.../EngineSession.kt` | Session abstraction |
| `components/concept/engine/src/main/java/.../EngineView.kt` | View abstraction |
| `components/concept/engine/src/main/java/.../Settings.kt` | Engine settings + DefaultSettings |
| `components/concept/engine/src/main/java/.../EngineSessionState.kt` | Serializable session state |

### Bridge Layer (GeckoEngine)
| File | Purpose |
|------|---------|
| `components/browser/engine-gecko/src/main/java/.../GeckoEngine.kt` | Engine impl, Settings → GeckoRuntimeSettings bridge |
| `components/browser/engine-gecko/src/main/java/.../GeckoEngineSession.kt` | EngineSession impl, GeckoSession delegates |
| `components/browser/engine-gecko/src/main/java/.../GeckoEngineView.kt` | EngineView impl, NestedGeckoView wrapper |
| `components/browser/engine-gecko/src/main/java/.../GeckoEngineSessionState.kt` | SessionState serialization |
| `components/browser/engine-gecko/src/main/java/.../NestedGeckoView.kt` | GeckoView with NestedScrollingChild |
| `components/browser/engine-gecko/src/main/java/.../GeckoWebExtension.kt` | WebExtension wrapper |
| `components/browser/engine-gecko/src/main/java/.../GeckoSitePermissionsStorage.kt` | Permission bridging |
| `components/browser/engine-gecko/src/main/java/.../GeckoViewFetchClient.kt` | HTTP client via GeckoWebExecutor |
| `components/browser/engine-gecko/src/main/java/.../GeckoAutocompleteStorageDelegate.kt` | Autofill bridge |
| `components/browser/engine-gecko/src/main/java/.../GeckoPromptDelegate.kt` | Prompt/alert bridging |

### Fenix Integration Layer
| File | Purpose |
|------|---------|
| `fenix/.../gecko/GeckoProvider.kt` | GeckoRuntime creation and caching |
| `fenix/.../components/Core.kt` | Engine, runtime, storage wiring |
| `fenix/.../components/TrackingProtectionPolicyFactory.kt` | TP → ContentBlocking.Settings |
| `fenix/.../browser/BaseBrowserFragment.kt` | EngineView rendering, feature wiring |
| `fenix/.../AppRequestInterceptor.kt` | Request interception (app links) |

## GeckoRuntime Initialization

### GeckoProvider.getOrCreateRuntime()
Singleton `@Synchronized` method that creates `GeckoRuntime` with:

1. **CrashHandler**: `CrashHandlerService::class.java`
2. **ExperimentDelegate**: Nimbus experiment integration
3. **ContentBlocking**: Comprehensive tracking protection via `TrackingProtectionPolicy.toContentBlockingSetting()`
4. **Console/Debug**: Conditional debug logging
5. **Extensions**: `extensionsProcessEnabled(true)`, `extensionsWebAPIEnabled(true)`
6. **Fission**: Web content isolation from Nimbus
7. **AutocompleteDelegate**: Bridges credit cards, addresses, and logins storage
8. **CrashPullDelegate**: Reports crash IDs to AppStore

### GeckoRuntimeSettings
The `GeckoRuntimeSettings` builder configures all runtime-level settings. Key groups:
- **ContentBlocking.Settings**: ETP level, categories, anti-tracking, cookie behavior, cookie banner handling, safe browsing, query stripping, bounce tracking
- **Debug/Remote**: Remote debugging, console output, about:config
- **Process**: Fission, isolated process, app zygote
- **Security**: Certificate transparency, post-quantum KEX, DoH autoselect

## Engine Interface Mapping

### Engine → GeckoRuntime
```
Engine.settings.javascriptEnabled        → runtime.settings.javaScriptEnabled
Engine.settings.trackingProtectionPolicy → runtime.settings.contentBlocking.*
Engine.settings.httpsOnlyMode            → runtime.settings.allowInsecureConnections
Engine.settings.preferredColorScheme     → runtime.settings.preferredColorScheme
Engine.settings.remoteDebuggingEnabled   → runtime.settings.remoteDebuggingEnabled
Engine.settings.loginAutofillEnabled     → runtime.settings.loginAutofillEnabled
// ... 50+ settings mapped in GeckoEngine.settings (anonymous subclass lines 1474-2076)
```

### EngineSession → GeckoSession
```
EngineSession.loadUrl(url)               → geckoSession.load(GeckoSession.Loader().uri(url))
EngineSession.reload()                   → geckoSession.reload()
EngineSession.goBack()/goForward()       → geckoSession.goBack()/goForward()
EngineSession.stopLoading()              → geckoSession.stop()
EngineSession.saveState()                → geckoSession.saveState()
EngineSession.restoreState(state)        → geckoSession.restoreState(state)
EngineSession.close()                    → geckoSession.close()
```

### EngineView → GeckoView
```
EngineView.render(session)               → geckoView.setSession(geckoSession)
EngineView.release()                     → geckoView.releaseSession()
EngineView.captureThumbnail()            → geckoView.capturePixels()
EngineView.setVerticalClipping(clip)     → geckoView.setVerticalClipping(clip)
```

## GeckoSession Delegates

Each `GeckoEngineSession` registers these delegates on its `GeckoSession`:

| Delegate | Purpose | Key Callbacks |
|----------|---------|---------------|
| NavigationDelegate | Page navigation | onLocationChange, onLoadRequest, onNewSession, onNavigationError |
| ProgressDelegate | Loading progress | onPageStart, onPageStop, onSecurityChange, onSessionStateChange |
| ContentDelegate | Page content events | onTitleChange, onCrash, onContextMenu, onFullScreen, onDownload, onCookieBannerBlocked |
| ContentBlockingDelegate | Tracking events | onContentBlocked, onContentLoaded (category + URI) |
| PermissionDelegate | Web permissions | onContentPermissionRequest, onMediaPermissionRequest, onAndroidPermissionsRequest |
| PromptDelegate | JS prompts | onAlert, onBeforeUnload, onButton, onChoice, onDateTime, onFile, onLoginSave, onShare, onWebAuthn |
| HistoryDelegate | Visit recording | onVisited, onTitleChanged, onPreviewImageChange, getVisited |
| MediaDelegate | Media playback | onMediaActionDelegate |
| MediaSessionDelegate | Media metadata | onMetadata, onPlaybackState, onPositionState, onFullscreen, onFeatures |
| ScrollDelegate | Scroll position | onScrollChanged |
| SelectionActionDelegate | Text selection | (default clipboard actions) |
| TranslationsSessionDelegate | Translations | (onTranslate, etc.) |

## Settings Flow (Fenix → Engine → GeckoView)

```
Settings.kt (SharedPreferences)
    ↕
TrackingProtectionPolicyFactory
    ↕
Core.kt DefaultSettings
    ↕
GeckoEngine constructor → apply { settings = defaultSettings }
    ↕ (GeckoEngine.settings anonymous class)
runtime.settings.* (GeckoRuntimeSettings)
```

### Session-Level Settings
```
GeckoEngineSession.settings.userAgentString → geckoSession.settings.userAgentOverride
GeckoEngineSession.settings.trackingProtection → geckoSession.settings.useTrackingProtection
```

## History Delegate Bridge

`HistoryDelegate` (`feature/session/HistoryDelegate.kt`) bridges GeckoView history callbacks to `PlacesHistoryStorage`:

```
GeckoSession.historyDelegate.onVisited(uri, flags)
    → HistoryTrackingDelegate.onVisited(uri, visit)
        → PlacesHistoryStorage.recordVisit(uri, visit)
```

Key filtering:
- Private mode visits are not recorded
- Error pages are not recorded
- Non-top-level loads (iframes) are not recorded
- App redirect URLs are filtered out

## WebExtension Support

```
GeckoEngine (WebExtensionRuntime)
  → runtime.webExtensionController.install/update/uninstall/list()
  → runtime.webExtensionController.setPromptDelegate()
  → runtime.webExtensionController.setDebuggerDelegate()
  → runtime.webExtensionController.setAddonManagerDelegate()
  → runtime.webExtensionController.setExtensionProcessDelegate()

GeckoWebExtension wraps org.mozilla.geckoview.WebExtension
  → MessageDelegate for background/content scripts
  → ActionDelegate for browser/page actions
  → TabDelegate / SessionTabDelegate for tab handling
```

Fenix initializes WebExtensions in `initializeWebExtensionSupport()`:
1. Sets up `GlobalAddonDependencyProvider`
2. Calls `WebExtensionSupport.initialize(engine, store, ...)` with callbacks for new tabs, close tabs, select tabs
3. Registers addon updater and supported addons checker

## Permissions Bridge

`GeckoSitePermissionsStorage` bridges GeckoView's StorageController with Fenix's on-disk storage:

- **GeckoView-managed**: ContentPermission types (geolocation, notifications, storage access, autoplay)
- **On-disk**: Other permissions (media, etc.) via `OnDiskSitePermissionsStorage`
- **Temporary permissions**: In-memory only ("don't remember my decision")

Permission storage is dual-write: `save()` updates GeckoView's StorageController AND on-disk storage.

## HTTP Client Bridge

`GeckoViewFetchClient` wraps `GeckoWebExecutor` to implement `concept-fetch` `Client`:
```
Client.fetch(Request) → GeckoWebExecutor.fetch(WebRequest) → WebResponse → Response
```

Used by: icons, location service, Merino (suggest), Pocket, ads, and other A-C components.

## Speculative Connections

`GeckoEngine` supports:
- `speculativeCreateSession()`: Pre-creates `GeckoEngineSession` with a ready `GeckoSession`
- `speculativeConnect(url)`: Pre-connects to a URL (DNS + TCP/TLS)
- `warmUp()`: Warms up the engine process

## Key Design Properties

1. **Singleton Runtime**: One `GeckoRuntime` per app, shared across all sessions
2. **Multi-Process**: GeckoView runs in a separate child process (content process)
3. **SurfaceView Rendering**: `GeckoView` uses `SurfaceView` (or `TextureView`) for compositing
4. **State Serialization**: `GeckoEngineSessionState` wraps `GeckoSession.SessionState` for save/restore
5. **Session Priority**: `updateSessionPriority(HIGH/DEFAULT)` hints resource allocation
6. **Fission**: Site isolation via `WebContentIsolationStrategy`
7. **Process Isolation**: Extension processes can be isolated via `extensionsProcessEnabled`

## Engine Switch Considerations

To replace GeckoView with another engine:
1. Implement `Engine`, `EngineSession`, `EngineView` interfaces
2. Create a Settings bridge from `DefaultSettings` → engine-specific config
3. Replace `GeckoViewFetchClient` with a `Client` impl for the new engine
4. Replace `GeckoSitePermissionsStorage` with a new `SitePermissionsStorage`
5. Replace `GeckoAutocompleteStorageDelegate` with platform-specific autofill
6. Replace `GeckoPromptDelegate` with new prompt handling
7. WebExtension support would need a new implementation (or be dropped)
8. WebAuthn/FIDO2 would need new handling
9. Content blocking maps differently per engine

The abstraction covers ~85% of the app; Gecko-specific details leak through `DefaultSettings` properties and `GeckoRuntime` references in `Core.kt`.

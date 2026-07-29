# Browser Session

## Purpose
A browser session represents a single browsing context (tab) lifecycle managed through the Engine/EngineSession abstraction. The session lifecycle covers creation, navigation, state persistence, suspension, and destruction.

## Session Lifecycle

```
createSession()
  → open(runtime)
    → register all delegates (navigation, progress, content, etc.)
      → loadUrl()
        → navigation events
          → saveState() / restoreState()
            → suspendSession() / close()
```

## EngineSession Interface

File: `A-C/.../concept/engine/EngineSession.kt`

### Key Methods
```kotlin
interface EngineSession {
    fun loadUrl(url: String, flags: LoadUrlFlags)
    fun loadUrl(url: String, headers: Map<String, String>)
    fun loadData(data: String, mimeType: String)
    
    fun goBack()
    fun goForward()
    fun reload()
    fun stopLoading()
    
    fun saveState(): EngineSessionState
    fun restoreState(state: EngineSessionState)
    
    fun close()
    
    suspend fun toggleRequestedTrackingProtection()
    
    // Settings
    val settings: SessionSettings
    
    // Observers
    fun register(object: Observer)
    fun unregister(object: Observer)
}
```

### Observer Callbacks
```kotlin
interface Observer {
    fun onLocationChange(url: String)
    fun onProgress(progress: Int)
    fun onLoadingStateChange(loading: Boolean)
    fun onNavigationStateChange(canGoBack: Boolean, canGoForward: Boolean)
    fun onSecurityChange(secure: Boolean, host: String?, issuer: String?)
    fun onTitleChange(title: String)
    fun onSessionStateChange(sessionState: EngineSessionState)
    fun onTrackerBlocked(tracker: Tracker)
    fun onTrackerLoaded(tracker: Tracker)
    fun onLongPress(request: HitResult)
    fun onCrash()
    fun onProcessKilled()
    fun onFirstContentfulPaint()
    fun onRecordingStateChanged(devices: List<RecordingDevice>)
    // ... more
}
```

## GeckoEngineSession Implementation

File: `A-C/.../engine-gecko/GeckoEngineSession.kt`

### Initialization
```kotlin
init {
    createGeckoSession()
        // Creates GeckoSession with private mode + context ID
        // Opens session on runtime
        // Registers all delegates
}
```

### Delegates Registered
| Delegate | Implementation | Key Purpose |
|----------|----------------|-------------|
| NavigationDelegate | `createNavigationDelegate()` | URL changes, load requests, new windows, errors |
| ProgressDelegate | `createProgressDelegate()` | Loading progress, security changes, page state |
| ContentDelegate | `createContentDelegate()` | Cookie banners, context menus, crashes, downloads, fullscreen |
| ContentBlockingDelegate | `createContentBlockingDelegate()` | Tracker events |
| PermissionDelegate | `createPermissionDelegate()` | Content, media, Android permissions |
| PromptDelegate | `GeckoPromptDelegate` | JS prompts, alerts, auth, file pickers |
| HistoryDelegate | `createHistoryDelegate()` | Visit recording |
| MediaDelegate | `createMediaDelegate()` | Media events |
| MediaSessionDelegate | `GeckoMediaSessionDelegate` | Media metadata, playback state |
| ScrollDelegate | `createScrollDelegate()` | Scroll position |
| SelectionActionDelegate | Default clipboard actions | Text selection |
| TranslationsSessionDelegate | `GeckoTranslateSessionDelegate` | Page translations |

### Session Priority
```kotlin
fun updateSessionPriority(priority: EngineSessionPriority) {
    geckoSession.setPriorityHint(priority.id)
}
// Priority.HIGH or Priority.DEFAULT
// Used by SessionPrioritizationMiddleware
```

## Session State

### EngineSessionState
```kotlin
interface EngineSessionState {
    val toJson: JSONObject
}
```

### GeckoEngineSessionState
Wraps `GeckoSession.SessionState`:
```kotlin
class GeckoEngineSessionState(internal val actualState: GeckoSession.SessionState) : EngineSessionState {
    override val toJson: JSONObject
        get() = JSONObject().apply {
            put("GECKO_STATE", actualState.toString())
        }
    
    companion object {
        fun fromJson(json: JSONObject): GeckoEngineSessionState? {
            val geckoState = json.optString("GECKO_STATE", null) ?: return null
            return GeckoEngineSessionState(GeckoSession.SessionState.fromString(geckoState))
        }
    }
}
```

### Save/Restore Flow
```
BrowserStore dispatches save state
  → EngineDelegateMiddleware → engineSession.saveState()
    → GeckoEngineSession.onSessionStateChange()
      → GeckoSession.SessionState
        → BrowserStateReducer saves to TabSessionState.engineState.engineSessionState

On restore:
  BrowserStore dispatches create engine session
    → CreateEngineSessionMiddleware
      → engine.createSession(private, contextId)
        → engineSession.restoreState(savedState)
          → geckoSession.restoreState(GeckoSession.SessionState)
```

## Session Features in BaseBrowserFragment

File: `fenix/.../browser/BaseBrowserFragment.kt`

### Features attached to the session
| Feature | Description |
|---------|-------------|
| `SessionFeature` | Ties session lifecycle to view lifecycle |
| `FullScreenFeature` | Fullscreen video delegate |
| `DownloadsFeature` | Download event handling |
| `PromptFeature` | Form/alert/confirm prompt handling |
| `SitePermissionsFeature` | Permission request handling |
| `ReaderViewFeature` | Reader mode toggle |
| `FindInPageFeature` | Find-in-page bar |
| `ContextMenuFeature` | Long-press context menus |
| `SwipeRefreshFeature` | Pull-to-refresh |
| `PictureInPictureFeature` | PiP support |
| `WebAuthnFeature` | FIDO2 activity result |

### Session Observation
```kotlin
// In BaseBrowserFragment
browserStore.stateFlow.map { state ->
    state.tabs.find { it.id == state.selectedTabId }
}.collect { tab ->
    tab?.let { renderTab(it) }
}

fun renderTab(tab: TabSessionState) {
    // Link EngineView to EngineSession
    engineView.render(tab.engineState.engineSession)
    
    // Update security indicator
    updateSecurityIcon(tab.content.securityInfo)
    
    // Update URL bar
    toolbar.url = tab.content.url
}
```

## Speculative Sessions

```kotlin
// Pre-create sessions for faster loading
GeckoEngine.speculativeCreateSession()
    → Pre-creates GeckoEngineSession with ready GeckoSession

GeckoEngine.clearSpeculativeSession()
    → Clears pre-created session cache
```

## Session Crashes

```kotlin
// Detected via EngineObserver
fun onCrash() {
    store.dispatch(CrashAction.SessionCrashedAction(sessionId))
}
// → Suspends session, sets crashed flag
// → User must manually restore or navigate away
// → CrashMiddleware prevents auto-recreation

fun onProcessKilled() {
    store.dispatch(KillEngineSessionAction(sessionId))
}
// → Adds to recentlyKilledTabs
// → Next navigation creates fresh session
// → If EngineSessionState exists, restores it
```

## Key Files

| File | Role |
|------|------|
| `A-C/.../concept/engine/EngineSession.kt` | Session abstraction |
| `A-C/.../concept/engine/EngineSessionState.kt` | Serializable state |
| `A-C/.../engine-gecko/GeckoEngineSession.kt` | GeckoView session impl |
| `A-C/.../engine-gecko/GeckoEngineSessionState.kt` | State serializer |
| `A-C/.../engine/EngineObserver.kt` | Session → Store bridge |
| `A-C/.../engine/middleware/CreateEngineSessionMiddleware.kt` | Session creation |
| `A-C/.../engine/middleware/SuspendMiddleware.kt` | Session suspend/kill |
| `A-C/.../engine/middleware/CrashMiddleware.kt` | Crash handling |
| `A-C/.../engine/middleware/LinkingMiddleware.kt` | Observer linking |
| `fenix/.../browser/BaseBrowserFragment.kt` | Session rendering + features |
| `fenix/.../browser/BaseBrowserFragment.kt` (line 1284) | WebAuthnFeature setup |

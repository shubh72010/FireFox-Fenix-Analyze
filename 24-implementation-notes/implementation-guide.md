# Implementation Guide

## Purpose
This guide explains how to build major Fenix features from scratch in another Android browser project. Each section covers the essential building blocks, state model, event flow, and implementation approach.

## 1. Building a Tab Management System

### Required Building Blocks
- Redux store (`Store<State, Action>` with middleware)
- Tab data model (ID, URL, title, private flag, engine session reference, timestamps)
- Tab use cases (add, remove, select, restore)
- Session persistence (JSON or database)

### State Model
```kotlin
data class TabState(
    val id: String,
    val url: String,
    val title: String,
    val isPrivate: Boolean,
    val lastAccess: Long,
    val engineSession: EngineSession?,
    val engineSessionState: EngineSessionState?,
)

data class BrowserState(
    val tabs: List<TabState>,
    val selectedTabId: String?,
)
```

### Key Implementation Steps
1. Create a Redux-style store for browser state
2. Implement tab use cases that dispatch actions
3. Connect use cases to UI (buttons, gestures, context menus)
4. Implement session persistence (serialize state to JSON on background/periodically)
5. Add middleware for cross-cutting concerns (telemetry, recently closed, thumbnails)
6. Build tab strip and tab tray UI

### Common Pitfalls
- Not persisting private tabs (security risk)
- Not filtering stale tabs on restore
- Not handling engine session lifecycle (creation, suspension, destruction)
- Race conditions between tab operations and session persistence

## 2. Building a GeckoView-like Wrapper

### Required Building Blocks
- `Engine` interface: createView(), createSession(), settings, name(), version
- `EngineSession` interface: loadUrl(), goBack(), goForward(), reload(), stop()
- `EngineView` interface: render(session), release(), captureThumbnail()
- Settings bridge: map your app's settings to engine-native settings

### Implementation Approach
```kotlin
class MyEngine(private val runtime: MyRuntime) : Engine {
    override fun createSession(private: Boolean, contextId: String?): EngineSession {
        return MyEngineSession(runtime, private)
    }
    
    override fun createView(context: Context): EngineView {
        return MyEngineView(context)
    }
}
```

### Key Decisions
- How to handle web rendering (SurfaceView vs TextureView vs WebView)
- Multi-process architecture (isolated renderer process)
- Content blocking integration
- Permission delegation
- State serialization for session restore

## 3. Building a Passkey System

### Required Building Blocks
- WebAuthn protocol handling (create credential, get assertion)
- Platform credential integration (Android Credential Manager + GMS FIDO2)
- Activity delegate for FIDO2 PendingIntent flow
- Origin validation and related-origin checking

### Two-Tier Platform Strategy
```
Primary (Android 14+):
  CredentialManager.createCredential() / getCredential()
    with TYPE_PUBLIC_KEY_CREDENTIAL

Fallback (all Android with GMS):
  Fido2PrivilegedApiClient.getRegisterPendingIntent() / getSignPendingIntent()
```

### Key Implementation Steps
1. Implement WebAuthn request handling (create/get)
2. Serialize WebAuthn options to JSON for Credential Manager API
3. Implement Activity delegate pattern for GMS PendingIntent
4. Implement decision tree: GMS non-discoverable → CM discoverable → GMS fallback
5. Implement origin validation + .well-known/webauthn fetch
6. Map platform errors to DOMException codes

## 4. Building Session Restore

### Required Building Blocks
- Serialization format (JSON recommended)
- Auto-save with periodic + background triggers
- Tab timeout filtering
- Engine session state serialization

### Implementation
```kotlin
class SessionStorage(val file: File) {
    fun save(state: BrowserState) {
        // Filter: only normal tabs, no crash pages
        // Serialize: tab URLs, titles, engine session state, timestamps
        // Write: AtomicFile for thread safety
    }
    
    fun restore(): List<RecoverableTab> {
        // Read JSON
        // Filter: remove stale tabs (by timeout)
        // Filter: remove invalid URLs
        // Return: list of recoverable tabs
    }
}

class AutoSave(private val storage: SessionStorage) {
    fun start(store: Store<BrowserState>) {
        // Periodically save (every 30s in foreground)
        // Save on app background
        // Save on tab change
    }
}
```

## 5. Building Memory Management

### Approach
- **Don't auto-suspend tabs**: Rely on the engine's internal memory management
- Clear icon/thumbnail caches on `onTrimMemory`
- Skip expensive operations (thumbnail capture) when memory is constrained
- Use session priority hints to guide engine resource allocation
- Handle content process kills gracefully (save state, reload on demand)

### Key Implementation Steps
1. Override `Application.onTrimMemory()` to clear caches
2. Implement session priority management (HIGH for active tab, DEFAULT for background)
3. Track recently killed tabs for crash recovery telemetry
4. Implement memory-aware thumbnail capture

## 6. Building a Settings System

### Architecture
```
Backend: SharedPreferences + DataStore
Wrapper: Typed Settings class with extension properties
UI: PreferenceFragmentCompat (traditional) or Compose (newer)
```

### Key Categories
1. Privacy/Security: Tracking protection, HTTPS-only, DoH, cookies
2. Appearance: Theme, toolbar position, font size
3. Tabs: Timeout, tab strip, view mode
4. Search: Engine selection, suggestions
5. Sync: Account, enabled engines
6. Downloads: Location, delete behavior
7. Accessibility: Auto-size, force zoom

## 7. Building Sync

### Required Building Blocks
- FxA/Sync protocol implementation (via Mozilla App Services or custom)
- Storage backends for each sync engine (history, bookmarks, passwords, tabs, autofill)
- Periodic sync scheduling
- Push notification integration for near-real-time sync

### Implementation Notes
- Use Mozilla App Services for Firefox Sync compatibility
- Supported engines: History, Bookmarks, Passwords, Tabs, Credit Cards, Addresses
- 4-hour periodic sync interval recommended
- Push enables faster sync triggers

## 8. Privacy Features Implementation

### Tracking Protection
- Implement multiple protection levels (Standard/Strict/Custom)
- Use platform content blocking API where available
- Categories: cookies, fingerprinters, cryptominers, tracking content, redirect trackers

### HTTPS-Only Mode
- Three levels: DISABLED, PRIVATE_ONLY, ENABLED
- Intercept navigation requests, upgrade HTTP to HTTPS
- Show error page if HTTPS upgrade fails

### DoH
- Support built-in providers (Cloudflare) and custom URLs
- Implement protection levels (Off/Default/Increased/Max)
- Validate provider URLs

## Key Design Principles to Follow

1. **Single Activity Architecture**: One activity, many fragments
2. **Redux State Management**: Multiple stores, middleware for side effects
3. **Engine Abstraction**: Interface + implementation pattern
4. **Lazy Initialization**: Defer work until needed
5. **Deferred Post-Startup**: Queue non-critical work after first frame
6. **Service Locator**: Central component access (or DI framework)
7. **Feature Flags**: Control feature rollout via experiments

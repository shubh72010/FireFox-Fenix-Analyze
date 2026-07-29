# Memory Management

## Purpose
Memory management in Fenix is split between Android-level (app process), GeckoView-level (content process), and store-level (middleware). A key design decision: Fenix explicitly disables automatic tab suspension and relies on GeckoView's internal memory management.

## Memory Pressure Detection

Two independent paths:

### 1. Android App Process (FenixApplication.onTrimMemory)
```
Android OS onTrimMemory(level)
  → FenixApplication.onTrimMemory(level)
      → recordCrashBreadcrumb (debug logging)
      → components.core.icons.onTrimMemory(level)  [clear icon cache]
      → store.dispatch(SystemAction.LowMemoryAction(level))
```

### 2. GeckoView Content Process (MemoryController)
```
Android OS onTrimMemory(level)
  → MemoryController.onTrimMemory(level)
      → TRANSLATION:
          TRIM_MEMORY_COMPLETE / RUNNING_CRITICAL → CRITICAL
          TRIM_MEMORY_BACKGROUND or higher → MODERATE
          otherwise → LOW (ignored)
      → GeckoAppShell.notifyObservers("memory-pressure", "low-memory"/"low-memory-ongoing")
      → Gecko nsIMemory internal GC/CC handling
```

The GeckoView path has a 10-second debounce: full `"low-memory"` at most once per 10 seconds, `"low-memory-ongoing"` within the window.

### 3. OS Snapshot Check
```kotlin
context.isOSOnLowMemory() → ActivityManager.MemoryInfo.lowMemory
```
Used by thumbnail capture to skip screenshots when memory is constrained.

## Disabled Automatic Tab Suspension

In `Core.kt`:
```kotlin
BrowserStore(
    middleware = middlewareList + EngineMiddleware.create(
        engine,
        trimMemoryAutomatically = false,  // KEY DECISION
    ),
)
```

This disables `TrimMemoryMiddleware` which would otherwise:
- On `TRIM_MEMORY_RUNNING_CRITICAL` or `TRIM_MEMORY_COMPLETE`
- Suspend non-selected tabs (LRU order, keeping MIN_ACTIVE_TABS=3)
- By dispatching `SuspendEngineSessionAction`

**Rationale**: Suspending and recreating engine sessions caused more OOMs and performance issues than letting GeckoView manage its own memory internally.

## What Actually Happens on Low Memory

1. **Icon Memory Cache cleared** via `BrowserIcons.onTrimMemory()`:
   - `IconMemoryCache` has two LRU caches:
     - `iconResourcesCache`: 1000 entry URL metadata cache
     - `iconBitmapCache`: 25 MB bitmap LruCache
   - Cleaned on RUNNING_LOW, RUNNING_CRITICAL, MODERATE, COMPLETE

2. **SystemAction.LowMemoryAction dispatched** → processed by `SystemReducer` (current no-op)

3. **GeckoView handles its own memory** internally via `MemoryController` → sends `"memory-pressure"` to Gecko engine

4. **Thumbnail capture skipped** if `isOSOnLowMemory()` returns true

## Session Prioritization

`SessionPrioritizationMiddleware` manages `EngineSession` priority:

- **Selected tab**: Priority set to `HIGH`
- **Deselected tab**: Checked for form data; if present, stays HIGH for 3-minute timeout, then reverts to DEFAULT
- **Background/foreground**: On app pause, checks selected tab for form data (without adjusting priority)
- **Session unlink**: Immediatly set to DEFAULT

This hints GeckoView to allocate more resources to the active tab.

## Tab Kill/Crash Recovery

### recentlyKilledTabs
`BrowserState.recentlyKilledTabs: LinkedHashSet<String>` (max 50 entries)

- **Added by**: `EngineStateReducer.killTab()` when `KillEngineSessionAction` processed
- **Removed by**: `EngineStateReducer.restoreTab()` when new `EngineSession` created
- **Purpose**: Track which tabs had content processes killed (not suspended)

### CrashDetection Flow
```
GeckoSession crash/process kill
  → EngineObserver.onCrash() / onProcessKilled()
      → dispatch(CrashAction.SessionCrashedAction)
      → dispatch(KillEngineSessionAction)
          → Reducer: add to recentlyKilledTabs, clear engineSession
          → SuspendMiddleware: close EngineSession
```

### CrashMiddleware
On `SessionCrashedAction`: dispatches `SuspendEngineSessionAction` to prevent automatic recreation. The crash flag persists until user manually restores.

### Telemetry Recovery Detection
`TelemetryMiddleware` checks `recentlyKilledTabs` when an engine session is created and records:
- `ContentProcessKill` (if ID in recentlyKilledTabs)
- `AppSessionRestore` (if app session restore)

## Cache Architecture

### Icon Memory Cache (`IconMemoryCache`)
| Cache | Type | Max Size | Cleaned On |
|-------|------|----------|------------|
| iconResourcesCache | LRU | 1000 entries | onTrimMemory |
| iconBitmapCache | LRU | 25 MB | onTrimMemory |

### Icon Disk Cache (`IconDiskCache`)
| Store | Max Size | Location |
|-------|----------|----------|
| iconResourcesStore | 10 MB | `noBackupFilesDir/mozac_browser_icons/` |
| iconDataStore | 100 MB | `noBackupFilesDir/mozac_browser_icons/` |

### Thumbnail Disk Cache (`ThumbnailDiskCache`)
| Instance | Max Size | Format |
|----------|----------|--------|
| sharedDiskCache | 100 MB | JPEG quality 90 |
| privateDiskCache | 100 MB | JPEG quality 90 (cleared on init) |

Location: `noBackupFilesDir/mozac_browser_thumbnails/`

## Thumbnail Memory Behavior

- `ContentState` does NOT store thumbnails in memory (no thumbnail field)
- Thumbnails are captured on first contentful paint via `BrowserThumbnails`
- Captures skipped if `isOSOnLowMemory()` returns true
- Saved to disk via `ThumbnailsMiddleware` (consumed there, not passed to reducer)
- Loaded on demand from disk when displayed

## Two-Level Memory Architecture

```
Android onTrimMemory()
  │
  ├──→ FenixApplication (app process)
  │      ├── Clear icon cache
  │      └── Dispatch LowMemoryAction (no-op)
  │
  └──→ MemoryController (content process)
         └──→ Gecko nsIMemory
                ├── Garbage Collection
                ├── Cycle Collection
                ├── Memory cache reduction
                └── Tab process management
```

Fenix deliberately relies on the content process path. If the content process IS killed, the app detects it via `EngineObserver.onProcessKilled()` and recovers gracefully.

## Key Files

| File | Role |
|------|------|
| `fenix/.../FenixApplication.kt` (line 759) | onTrimMemory entry point |
| `fenix/.../components/Core.kt` (line 416) | trimMemoryAutomatically = false |
| `A-C/.../engine/middleware/TrimMemoryMiddleware.kt` | Auto-suspend on low mem (disabled) |
| `A-C/.../engine/middleware/SuspendMiddleware.kt` | Unlink+close on suspend/kill |
| `A-C/.../engine/middleware/CrashMiddleware.kt` | Handle session crashes |
| `A-C/.../engine/middleware/SessionPrioritizationMiddleware.kt` | Session priority hints |
| `A-C/.../engine/EngineObserver.kt` (line 430) | Crash/process kill detection |
| `A-C/.../engine/EngineMiddleware.kt` | Middleware composition, trimMemory flag |
| `A-C/.../reducer/SystemReducer.kt` | LowMemoryAction handler (no-op) |
| `A-C/.../reducer/EngineStateReducer.kt` | KillTab, restoreTab |
| `A-C/.../state/BrowserState.kt` | recentlyKilledTabs field |
| `A-C/.../state/EngineState.kt` | engineSession, engineSessionState, crashed |
| `A-C/.../icons/utils/IconMemoryCache.kt` | 25 MB LRU bitmap cache |
| `A-C/.../icons/utils/IconDiskCache.kt` | 10 MB + 100 MB disk stores |
| `A-C/.../thumbnails/utils/ThumbnailDiskCache.kt` | 100 MB disk cache per mode |
| `A-C/.../thumbnails/ThumbnailsMiddleware.kt` | Thumbnail persistence on update |
| `A-C/.../thumbnails/BrowserThumbnails.kt` | Thumbnail capture (skipped on low mem) |
| `geckoview/.../MemoryController.java` | GeckoView process memory handling |
| `A-C/.../ktx/android/content/Context.kt` (line 73) | isOSOnLowMemory() check |

(A-C = `android-components` root)

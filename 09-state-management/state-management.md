# State Management

## Purpose
Fenix uses a multi-store Redux architecture for state management. There is no single global store; different domains have their own `Store<State, Action>` with middleware.

## Store Architecture

### Primary Stores

| Store | State Type | Scope | Location |
|-------|-----------|-------|----------|
| `BrowserStore` | `BrowserState` | Browser engine state (tabs, sessions, content) | A-C `browser/state` |
| `AppStore` | `AppState` | App-level UI state (home screen, mode, messaging) | Fenix `components/appstate` |
| `TabsTrayStore` | `TabsTrayState` | Tab tray UI | Fenix `tabstray/redux` |
| `DownloadUIStore` | `DownloadUIState` | Download list screen | Fenix `downloads/listscreen/store` |
| `IPProtectionStore` | `IPProtectionState` | IP Protection feature | A-C `feature/ipprotection` |
| `IPProtectionPromptStore` | `IPProtectionPromptState` | IPP onboarding prompt | Fenix `ipprotection/store` |
| `DohSettingsStore` | `DohSettingsState` | DoH settings | Fenix `settings/doh` |

### Store Pattern
All stores follow `mozilla.components.lib.state.Store<S, A>`:

```kotlin
class Store<S : State, A : Action>(
    initialState: S,
    reducer: (S, A) -> S,
    middleware: List<Middleware<S, A>> = emptyList(),
)
```

Key methods:
- `dispatch(action)` → middleware chain → reducer → new state
- `stateFlow: StateFlow<S>` → reactive observation
- `state: S` → synchronous snapshot

## BrowserStore (Global Browser State)

### Creation (Core.kt)
```kotlin
val store = BrowserStore(
    middleware = middlewareList + EngineMiddleware.create(engine, trimMemoryAutomatically = false)
)
```

### Middleware Stack (in order)
```
1. ProfileMarkerMiddleware        → Performance profiling
2. LogMiddleware                  → Debug logging
3. LastAccessMiddleware           → Updates lastAccess on tab selection
4. RecentlyClosedMiddleware       → Manages closed tabs list
5. DownloadMiddleware             → Download lifecycle
6. ReaderViewMiddleware           → Reader mode state
7. TelemetryMiddleware            → Telemetry events
8. ThumbnailsMiddleware           → Thumbnail persistence (consumed)
9. UndoMiddleware                 → Tab close undo delay
10. RegionMiddleware              → Geo-region detection
11. SearchMiddleware              → Search engine management
12. RecordingDevicesMiddleware    → Media recording indicators
13. PromptMiddleware              → Dialog/prompt management
14. AdsTelemetryMiddleware        → Search ad telemetry
15. LastMediaAccessMiddleware     → Media session tracking
16. HistoryMetadataMiddleware     → History metadata recording
17. ProtectionsDashboardMiddleware → Tracker blocking stats
18. SessionPrioritizationMiddleware → Engine session priority
19. SaveToPDFMiddleware           → PDF saving
20. FxSuggestFactsMiddleware      → Firefox Suggest telemetry
21. FileUploadsDirCleanerMiddleware → Temp file cleanup
22. DesktopModeMiddleware         → Per-site desktop mode
23. ApplicationSearchMiddleware   → App search integration
24. TranslationsMiddleware        → Page translations (lazy init)
25. StartupMiddleware             → Homepage tab after restore
26. AboutHomeMiddleware           → About:home handling
27. BrowserVisualCompletenessMiddleware → Signal visual completeness
28. TabGroupMiddleware            → Tab group operations
29. EngineMiddleware.create()     → Engine session lifecycle
```

### BrowserState
```kotlin
data class BrowserState(
    val tabs: List<TabSessionState>,
    val tabPartitions: Map<String, TabPartition>,
    val customTabs: List<CustomTabSessionState>,
    val closedTabs: List<TabState>,
    val selectedTabId: String?,
    val containers: Map<String, ContainerState>,
    val undoHistory: UndoHistoryState,
    val restoreComplete: Boolean,
    val desktopMode: Boolean,
    val recentlyKilledTabs: LinkedHashSet<String>,
    val search: SearchState,
    val downloads: DownloadState,
    val media: MediaState,
    val engine: EngineState,
    val protections: ProtectionsState,
)
```

## AppStore (Application UI State)

### Creation (Components.kt)
```kotlin
val appStore = AppStore(
    initialState = AppState(
        collections = ...,
        expandedCollections = emptySet(),
        topSites = ...,
        bookmarks = emptyList(),
        recentTabs = ...,
        // ... more initial state
    ).run { filterState(blocklistHandler) },
    middlewares = listOf(
        ProfileMarkerMiddleware,
        LogMiddleware,
        BlocklistMiddleware,
        PocketMiddleware,
        MessagingMiddleware,
        MetricsMiddleware,
        CrashReportingAppMiddleware,
        HomeTelemetryMiddleware,
        SetupChecklistPreferencesMiddleware,
        SetupChecklistTelemetryMiddleware,
        ReviewPromptMiddleware,
        AppVisualCompletenessMiddleware,
    ),
)
```

### AppState
```kotlin
data class AppState(
    val browsingMode: BrowsingMode,
    val orientation: OrientationMode,
    val topSites: List<TopSite>,
    val recentTabs: List<RecentTab>,
    val bookmarks: List<Bookmark>,
    val collections: List<TabCollection>,
    val pocketStories: List<PocketRecommendation>,
    val wallpaperState: WallpaperState,
    val messagingState: MessagingState,
    val searchState: SearchState,
    val blockedTrackers: BlockedTrackersState,
    val readerViewState: ReaderViewState,
    val findInPageState: FindInPageState,
    val crashReportingState: CrashReportingState,
    val snackbarState: SnackbarState,
    val setupChecklistState: SetupChecklistState?,
    val privateBrowsingLock: PrivateBrowsingLockState,
    val reviewPromptState: ReviewPromptState,
    val translationsState: TranslationsState,
    val voiceSearchState: VoiceSearchState,
    val lensState: LensState,
    val qrScannerState: QrScannerState,
    val shareState: ShareState,
    val shortcutState: ShortcutState,
    val webCompatState: WebCompatState,
    val contentRecommendationsState: ContentRecommendationsState,
    // ... more
)
```

### AppAction Hierarchy
```kotlin
sealed class AppAction {
    // Core actions
    data class BrowsingModeManagerModeChanged(val mode: BrowsingMode) : AppAction()
    data class OrientationChange(val orientation: OrientationMode) : AppAction()
    
    // Domain actions (each has own sealed hierarchy)
    sealed class MessagingAction : AppAction()
    sealed class WallpaperAction : AppAction()
    sealed class AppLifecycleAction : AppAction()
    sealed class ShortcutAction : AppAction()
    sealed class ShareAction : AppAction()
    sealed class SnackbarAction : AppAction()
    sealed class BookmarkAction : AppAction()
    sealed class FindInPageAction : AppAction()
    sealed class ReaderViewAction : AppAction()
    sealed class DownloadAction : AppAction()
    sealed class SearchAction : AppAction()
    sealed class ReviewPromptAction : AppAction()
    sealed class BlockedTrackersAction : AppAction()
    sealed class TranslationsAction : AppAction()
    sealed class SetupChecklistAction : AppAction()
    sealed class WebCompatAction : AppAction()
    sealed class PrivateBrowsingLockAction : AppAction()
    // ... more
}
```

### Reducer Pattern
`AppStoreReducer.reduce(state, action)` is a `when` expression delegating to sub-reducers:
```kotlin
fun reduce(state: AppState, action: AppAction): AppState = when (action) {
    is AppAction.MessagingAction -> messagingReducer(state, action)
    is AppAction.WallpaperAction -> wallpaperReducer(state, action)
    is AppAction.BookmarkAction -> bookmarkReducer(state, action)
    is AppAction.FinInPageAction -> findInPageReducer(state, action)
    // ... 15+ sub-reducers
    else -> state
}
```

## TabsTrayStore (Tab Tray)

### Middleware
```
1. TabsTrayTelemetryMiddleware   → Glean telemetry
2. TabSearchMiddleware           → Tab search filtering
3. TabSearchNavigationMiddleware → Navigate on search select
4. TabStorageMiddleware          → Transform BrowserState → TabsTrayState
5. TabManagerUiStateStorageMiddleware → Persist UI state
```

### TabStorageMiddleware (Key Component)
Observes `BrowserStore.stateFlow` and `TabGroupRepository.tabGroupDataFlow`, transforms them into `TabsTrayState` on every change.

## State Flow Patterns

### Fragment → Store Observation
```kotlin
// In Fragment
lifecycleScope.launch {
    store.stateFlow.collect { state ->
        // Update UI
    }
}

// Or for Compose
val state by store.stateFlow.collectAsState()
```

### Feature Bindings (AbstractBinding)
```kotlin
class TrackersBlockedFeature(
    private val browserStore: BrowserStore,
    private val appStore: AppStore,
) : AbstractBinding<TrackerBlockingState>() {
    override fun start() {
        // Observe store and dispatch to another store
    }
}
```

### ViewBoundFeatureWrapper
```kotlin
private val feature = ViewBoundFeatureWrapper<SomeFeature>()
// In onCreateView:
feature.set(SomeFeature(context), view, owner)
// Auto-stops on view destruction
```

## Middleware Pattern

### Structure
```kotlin
class MyMiddleware : Middleware<BrowserState, BrowserAction> {
    override fun invoke(
        context: MiddlewareContext<BrowserState, BrowserAction>,
        next: (BrowserAction) -> Unit,
        action: BrowserAction,
    ) {
        // Pre-processing
        if (action is RelevantAction) {
            // Side effect (telemetry, persistence, etc.)
        }
        // Continue chain
        next(action)
        // Post-processing
    }
}
```

### Usage Categories
| Category | Examples |
|----------|----------|
| Telemetry | TelemetryMiddleware, MetricsMiddleware, HomeTelemetryMiddleware |
| Persistence | ThumbnailsMiddleware, RecentlyClosedMiddleware |
| Navigation | TabSearchNavigationMiddleware, DownloadNavigationMiddleware |
| Engine | EngineDelegateMiddleware, SuspendMiddleware, CrashMiddleware |
| UI State | TabManagerUiStateStorageMiddleware, TabStorageMiddleware |
| Cross-feature | MessagingMiddleware, ReviewPromptMiddleware |

## State Observation in Compose

```kotlin
// BrowserStore observation
val currentTab by browserStore.observeAsComposableState { _, state ->
    state.tabs.find { it.id == state.selectedTabId }
}

// AppStore observation
val appState by appStore.stateFlow.collectAsState()

// Derived state
val tabStripState by browserStore.stateFlow.map { state ->
    state.toTabStripState()
}.collectAsState(initial = TabStripState.empty())
```

## Key Files

| File | Role |
|------|------|
| `A-C/.../lib/state/Store.kt` | Redux store implementation |
| `A-C/.../browser/state/store/BrowserStore.kt` | BrowserStore |
| `A-C/.../browser/state/state/BrowserState.kt` | BrowserState |
| `A-C/.../browser/state/action/BrowserAction.kt` | All browser actions |
| `A-C/.../browser/state/reducer/BrowserStateReducer.kt` | Main reducer |
| `A-C/.../browser/state/engine/EngineMiddleware.kt` | Engine middleware composition |
| `fenix/.../components/AppStore.kt` | AppStore |
| `fenix/.../components/appstate/AppState.kt` | AppState |
| `fenix/.../components/appstate/AppAction.kt` | AppAction |
| `fenix/.../components/appstate/AppStoreReducer.kt` | App reducer with sub-reducers |
| `fenix/.../tabstray/TabsTrayStore.kt` | TabsTrayStore |
| `fenix/.../tabstray/TabsTrayState.kt` | TabsTrayState |
| `fenix/.../tabstray/TabStorageMiddleware.kt` | BrowserState → TabsTrayState |

(A-C = `android-components` root)

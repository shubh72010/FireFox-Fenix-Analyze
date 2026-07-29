# Tab Management

## Purpose
The tab management system handles creation, selection, removal, persistence, and display of browser tabs. It spans across the Fenix codebase and android-components.

## Architecture

```
BrowserFragment (renders active tab)
    ↕ (observes selectedTabId)
BrowserStore (BrowserState) ← TabsUseCases → User actions
    ↕ (stateFlow)
TabStorageMiddleware → TabsTrayStore → TabManagementFragment UI
TabStripMiddleware → TabStrip composable
```

## Core State Model

### BrowserState
File: `android-components/.../browser/state/state/BrowserState.kt`

Key fields:
```kotlin
data class BrowserState(
    val tabs: List<TabSessionState>,        // All open tabs (normal + private)
    val tabPartitions: Map<String, TabPartition>, // Tab groups
    val customTabs: List<CustomTabSessionState>,
    val closedTabs: List<TabState>,           // Recently closed (via middleware)
    val selectedTabId: String?,
    val containers: Map<String, ContainerState>, // Contextual identities
    val undoHistory: UndoHistoryState,
    val restoreComplete: Boolean,
    val desktopMode: Boolean,
    val recentlyKilledTabs: LinkedHashSet<String>, // Max 50
)
```

### TabSessionState
File: `android-components/.../browser/state/state/TabSessionState.kt`

```kotlin
data class TabSessionState(
    val id: String,                             // UUID
    val content: ContentState,                  // URL, title, icon, private flag
    val trackingProtection: TrackingProtectionState,
    val engineState: EngineState,               // EngineSession + session state
    val parentId: String?,
    val lastAccess: Long,
    val lastVisibleAt: Long,
    val createdAt: Long,
    val contextId: String?,                     // Container identity
    val source: SessionState.Source,            // UserEntered, NewTab, Action, etc.
    val restored: Boolean,
)
```

### BrowsingMode
`BrowsingMode` enum: `Normal`, `Private` (distinguished by `TabSessionState.content.private: Boolean`).

## Tab Operations (TabsUseCases)

All in `android-components/.../feature/tabs/TabsUseCases.kt`:

| Use Case | Action Dispatched | Description |
|----------|------------------|-------------|
| `addTab` | `TabListAction.AddTabAction` + `EngineAction.LoadUrlAction` | Create new tab |
| `selectTab` | `TabListAction.SelectTabAction` | Switch to tab |
| `removeTab` | `TabListAction.RemoveTabAction` | Close single tab |
| `removeAllTabs` | `TabListAction.RemoveAllTabsAction` | Close all tabs |
| `removeNormalTabs` | `TabListAction.RemoveAllNormalTabsAction` | Close all normal tabs |
| `removePrivateTabs` | `TabListAction.RemoveAllPrivateTabsAction` | Close all private tabs |
| `undo` | `UndoAction.RestoreRecoverableTabs` | Undo tab closure |
| `restore` | `TabListAction.RestoreAction` | Restore from session |
| `selectOrAddTab` | `SelectTabAction` or `AddTabAction` + `LoadUrlAction` | Find or create |
| `duplicateTab` | `AddTabAction` (copy state) | Duplicate tab |
| `moveTabs` | `TabListAction.MoveTabsAction` | Reorder tabs |
| `migratePrivateTabUseCase` | Remove + Add | Move private → normal |

### AddNewTabUseCase Flow
1. `createTab()` constructs `TabSessionState` with URL, private flag, source, parent
2. Dispatches `TabListAction.AddTabAction(tab, select=true)` to `BrowserStore`
3. If `startLoading`: dispatches `EngineAction.LoadUrlAction(tab.id, url)`
4. Dispatches `ContentAction.UpdateIsSearchAction` if search result

### RestoreUseCase Flow
1. `storage.restore()` reads `RecoverableBrowserState` from JSON
2. Applies `tabTimeoutInMs` filter (discard stale tabs)
3. Dispatches `TabListAction.RestoreAction` → populates `BrowserState.tabs`
4. Dispatches `RestoreCompleteAction`

## Tab Rendering

### BaseBrowserFragment
File: `fenix/.../browser/BaseBrowserFragment.kt`

- Observes `BrowserStore.stateFlow` for `selectedTabId` changes
- Links `EngineView` to the selected tab's `EngineSession`
- Manages features: toolbar, fullscreen, downloads, prompts, permissions, reader view, find in page, context menus, PiP, media, translations

### BrowserFragment
Extends `BaseBrowserFragment` with additional features specific to full browsing (vs custom tabs).

## Tab Tray

The full-screen tab management UI is built in Compose:

```
TabManagementFragment
  └── TabsTrayStore (Redux, local to tray)
        ├── TabsTrayTelemetryMiddleware
        ├── TabSearchMiddleware
        ├── TabSearchNavigationMiddleware
        ├── TabStorageMiddleware (key: transforms BrowserState → TabsTrayState)
        └── TabManagerUiStateStorageMiddleware
  └── TabsTray composable
        ├── HorizontalPager
        │     ├── NormalTabsPage
        │     ├── PrivateTabsPage
        │     ├── SyncedTabsPage
        │     └── TabGroupsPage
        └── DefaultTabManagerController
```

### TabsTrayState
Key fields: `selectedPage`, `mode` (Normal/Select/DragAndDrop), `normalTabsState`, `inactiveTabs`, `privateBrowsing` (with lock state), `tabGroupState`, `sync`, `backStack`.

### TabStorageMiddleware
The central middleware that:
1. Observes `BrowserStore` tab data + `TabGroupRepository` data
2. Calls `transformTabData()` to build `TabsTrayState` from raw tab data
3. Handles tab group creation, editing, merge, delete, drag-and-drop, reorder

## Tab Strip

The horizontal tab strip above the toolbar (Compose):

```
TabStrip composable
  ├── LazyRow of TabStripCard items
  ├── TabCounterMenuItem dropdown
  └── toTabStripState() converter (BrowserState → TabStripState)
```

Key behavior:
- Shows favicon, title, close button per tab
- Close button visible on all tabs if ≤ 7, otherwise only on selected
- Long-press drag-and-drop reordering
- Auto-scroll to new/selected tabs
- Filtered by private/normal mode

## Tab Persistence

### SessionStorage
File: `android-components/.../browser/session-storage/src/main/java/.../SessionStorage.kt`

- Persists `BrowserState` to JSON file on disk (`mozilla_components_session_storage_<engine>.json`)
- Uses `AtomicFile` for thread-safe writes
- Only normal tabs are persisted (private tabs excluded)
- `save()` filters out `about:crashparent`
- `restore()` applies predicate to validate URLs

### AutoSave
File: `android-components/.../browser/session-storage/src/main/java/.../AutoSave.kt`

Configured in `FenixApplication.restoreBrowserState()`:
```kotlin
sessionStorage.autoSave(store)
    .periodicallyInForeground(interval = 30, unit = TimeUnit.SECONDS)
    .whenGoingToBackground()
    .whenSessionsChange()
```

### Serialization
- `BrowserStateWriter`: Serializes `BrowserState` to JSON
- `BrowserStateReader`: Deserializes JSON to `RecoverableBrowserState`
- `RecoverableTab`: Tab data + `EngineSessionState` for restoration

## Tab Groups

### State Model
```kotlin
BrowserState.tabPartitions: Map<String, TabPartition>
  TabPartition.id
  TabPartition.tabGroups: List<TabGroup>
    TabGroup.id, TabGroup.name, TabGroup.tabIds: Set<String>
```

### Persistence (Room Database)
- `TabGroupRepository`: Storage layer
- `StoredTabGroup`: Group metadata (id, title, theme, closed, lastModified)
- `TabGroupAssignment`: Maps tab IDs to group IDs
- `TabGroupOperationsDao`: CRUD operations
- `TabGroupDatabase`: Room database

### Creation Flow
1. User selects tabs → "Add to Group" UI
2. `TabGroupAction.SaveClicked` dispatched
3. `TabStorageMiddleware` creates group in `TabGroupRepository`
4. Tabs sequenced together via `MoveTabsUseCase`

## Private Tabs

- Flag: `TabSessionState.content.private: Boolean`
- Filtered via `normalTabs`/`privateTabs` selectors
- Never persisted to disk via `SessionStorage`
- Locked via `PrivateBrowsingLockFeature` (requires biometric/PIN on app background)
- `MigratePrivateTabUseCase` converts private → normal
- Separate removal actions: `RemoveAllNormalTabsAction`, `RemoveAllPrivateTabsAction`

## Related Middleware

| Middleware | Purpose |
|-----------|---------|
| `LastAccessMiddleware` | Updates `lastAccess` on tab selection |
| `RecentlyClosedMiddleware` | Manages recently closed tabs list |
| `UndoMiddleware` | Handles undo delay for tab closures |
| `SessionPrioritizationMiddleware` | Sets engine session priority (HIGH/DEFAULT) |
| `ThumbnailsMiddleware` | Persists thumbnails on update |
| `TelemetryMiddleware` | Records tab count telemetry |
| `DesktopModeMiddleware` | Manages per-site desktop mode |
| `TabGroupMiddleware` | Coordinates tab group operations |

## Data Flow for Opening a New Tab

```
1. User enters URL or clicks link
2. FenixBrowserUseCases.loadUrlOrSearch()
3. TabsUseCases.addTab.invoke(url, selectTab=true, startLoading=true)
4. AddNewTabUseCase:
   a. createTab() → TabSessionState
   b. dispatch(AddTabAction(tab))
   c. dispatch(LoadUrlAction(tab.id, url))
5. BrowserStore reducer: adds tab, updates selectedTabId
6. BrowserFragment observes: renders new session
7. TabStorageMiddleware observes: updates TabsTrayState
8. TabStrip observes: adds tab card
9. SessionStorage.AutoSave: persists state (eventually)
```

## Key Files

| File | Role |
|------|------|
| `fenix/.../components/UseCases.kt` | TabsUseCases, CustomTabsUseCases creation |
| `fenix/.../browser/browsingmode/BrowsingModeManager.kt` | Normal/Private mode tracking |
| `fenix/.../browser/tabstrip/TabStrip.kt` | Tab strip Compose UI |
| `fenix/.../tabstray/TabManagementFragment.kt` | Tab tray store and UI setup |
| `fenix/.../tabstray/TabStorageMiddleware.kt` | BrowserState → TabsTrayState transform |
| `fenix/.../tabstray/TabsTrayStore.kt` | Tabs tray Redux store |
| `fenix/.../tabgroups/TabGroupRepository.kt` | Room-backed tab group storage |
| `A-C/.../feature/tabs/TabsUseCases.kt` | All tab operation use cases |
| `A-C/.../browser/state/state/BrowserState.kt` | Global browser state |
| `A-C/.../browser/state/state/TabSessionState.kt` | Single tab state |
| `A-C/.../browser/state/store/BrowserStore.kt` | Redux store for browser state |
| `A-C/.../browser/session-storage/SessionStorage.kt` | JSON-based tab persistence |
| `A-C/.../browser/session-storage/AutoSave.kt` | Periodic auto-save |

(A-C = `android-components` root)

# Storage Layer

## Purpose
Fenix uses multiple persistence mechanisms: Places (SQLite via Rust), JSON files, SharedPreferences, Room, DataStore, and disk caches. Each serves a specific purpose with different performance and reliability characteristics.

## Storage Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Storage Systems                        │
├────────────┬───────────┬──────────┬──────────┬───────────┤
│  Places    │  Room     │  JSON    │  Shared  │DataStore  │
│  (Rust     │  (Tab     │  (Tab    │  Prefs   │(Pocket    │
│   SQLite)  │   Groups) │  Session)│ (Settings│ Stories,  │
│            │           │          │  + Core) │ AI, etc.) │
├────────────┼───────────┼──────────┼──────────┼───────────┤
│ History    │ Tab Group │ Browser  │ User     │ Feature   │
│ Bookmarks  │ Metadata  │ State    │ Prefs    │ Settings  │
│ Logins     │           │ (restore)│ Secret   │           │
│ Autofill   │           │          │ Settings │           │
│ Synced Tabs│           │          │          │           │
└────────────┴───────────┴──────────┴──────────┴───────────┘
```

## Places (Mozilla Application Services)

### What it stores
- **History**: Visited URLs, visits, page observations (title, preview image)
- **Bookmarks**: Folders, bookmarks, separators, mobile/desktop roots
- **Metadata**: History metadata (recent visits grouped by URL)

### Implementation
Built on SQLite via Rust (Mozilla Application Services megazord). Initialized through `AppServicesInitializer.init()`.

### Key Classes
```kotlin
PlacesHistoryStorage(context, crashReporter)  // History
PlacesBookmarksStorage(context)                 // Bookmarks
```

### Operations
- `recordVisit(uri, visitType)` - record page visit
- `recordObservation(uri, pageObservation)` - record title/preview
- `getVisited(uris)` - bulk visited check
- `warmUp()` - pre-warm storage (called after visual completeness)
- `registerStorageMaintenanceWorker()` - periodic maintenance via WorkManager

## Syncable Logins Storage

### What it stores
- Encrypted login credentials (username + password per origin)

### Implementation
`SyncableLoginsStorage(context, securePrefs)` backed by encrypted SQLite via Rust.

### Key Classes
```kotlin
SyncableLoginsStorage(context, lazySecurePrefs)
```

### Secure Preferences
```kotlin
SecureAbove22Preferences(
    context = context,
    name = "core_prefs",
    forceInsecure = !Config.channel.isDebug,
    crashReporting = crashReporter,
)
```

Uses Android KeyStore for encryption on API 23+, debug builds only actually encrypt. See: `https://github.com/mozilla-mobile/fenix/issues/19155`

## Autofill Storage

### What it stores
- Credit cards (encrypted)
- Addresses (encrypted)

### Implementation
`AutofillCreditCardsAddressesStorage(context, securePrefs)` - Rust-backed encrypted storage.

### Key Classes
```kotlin
AutofillCreditCardsAddressesStorage(context, lazySecurePrefs)
```

## Remote Tabs Storage

### What it stores
- Synced tabs from other Firefox devices

### Implementation
`RemoteTabsStorage(context, crashReporter)` - Rust-backed storage for the Tabs sync engine.

## Tab Groups (Room)

### What it stores
- Tab group metadata (name, theme, creation time)
- Tab-to-group assignments

### Implementation
Room database with:
```kotlin
@Database(entities = [StoredTabGroup::class, TabGroupAssignment::class], version = 1)
abstract class TabGroupDatabase : RoomDatabase() {
    abstract fun tabGroupOperationsDao(): TabGroupOperationsDao
}
```

### Key Classes
- `TabGroupRepository` - storage layer interface + implementation
- `StoredTabGroup` - group entity (id, title, theme color, closed flag, lastModified)
- `TabGroupAssignment` - tab-to-group mapping entity (tabId, groupId)
- `TabGroupOperationsDao` - CRUD DAO

## Browser Session Storage (JSON)

### What it stores
- Full `BrowserState` for tab restoration (normal tabs only)

### Implementation
`SessionStorage` serializes `BrowserState` to a JSON file on disk:
```
mozilla_components_session_storage_<engine>.json
```

### Key Classes
```kotlin
SessionStorage(context, engine, crashReporter)
  // Auto-save:
  sessionStorage.autoSave(store)
    .periodicallyInForeground(30, SECONDS)
    .whenGoingToBackground()
    .whenSessionsChange()

  // Restore:
  val recoverableState = storage.restore()  // Read JSON
```

### Serialization
- `BrowserStateWriter` - `BrowserState` → JSON
- `BrowserStateReader` - JSON → `RecoverableBrowserState`
- `RecoverableTab` - Tab data + `EngineSessionState`
- `AtomicFile` used for thread-safe writes

### What is NOT persisted
- Private tabs (never saved)
- `about:crashparent` pages filtered out

## SharedPreferences

### Usage locations
| Name | Contents | Access Pattern |
|------|----------|----------------|
| `FENIX_PREFERENCES` | All user settings | `Settings.kt` wrapper |
| `core_prefs` | Secure prefs (encrypted on debug) | `SecureAbove22Preferences` |
| `private_browsing_lock_prefs` | PBM lock settings | `DefaultPrivateBrowsingLockStorage` |
| Various feature prefs | Feature-specific settings | Direct |

### Key Settings Categories
- Privacy: tracking protection, cookie behavior, HTTPS-only, DoH
- Appearance: theme, toolbar position, font size
- Search: search engines, suggestions
- Tabs: tab timeout, tab view mode
- Sync: enabled engines, account info
- Downloads: download location, delete behavior
- Accessibility: auto-size, force zoom

## DataStore (Jetpack)

Used for newer feature-specific settings:

| DataStore | Contents |
|-----------|----------|
| `pocket_stories_selected_categories` | Selected Pocket content categories |
| `AIFeatureBlockStorage` | AI feature block state |
| `SummarizationSettings` | Page summarization preferences |
| `IPProtectionEligibility` | IPP eligibility state |
| `TranslationsSettings` | Translation enabled/disabled |
| `HomepageAsANewTabPreference` | Homepage as new tab setting |

## Disk Caches

### Icon Cache
| Cache | Location | Max Size |
|-------|----------|----------|
| Icon Memory Cache | In-memory LRU | 25 MB (bitmaps) + 1000 entries (resources) |
| Icon Disk Cache | `noBackupFilesDir/mozac_browser_icons/` | 10 MB (resources) + 100 MB (data) |

### Thumbnail Cache
| Instance | Location | Max Size |
|----------|----------|----------|
| sharedDiskCache | `noBackupFilesDir/mozac_browser_thumbnails/` | 100 MB |
| privateDiskCache | Same dir | 100 MB (cleared on init) |

Format: JPEG quality 90.

### ThumbnailStorage
```kotlin
ThumbnailStorage(context)
  .loadThumbnail(sessionId, isPrivate) → Bitmap?
  .saveThumbnail(sessionId, isPrivate, bitmap)
  .deleteThumbnail(sessionId)
  .clearThumbnails()
```

## File Uploads Directory Cleaner

```kotlin
FileUploadsDirCleaner { context.cacheDir }
```
Cleans temporary upload files. Runs after visual completeness and via middleware.

## Recently Closed Tabs Storage

```kotlin
RecentlyClosedTabsStorage(context, engine, crashReporter)
```
Persists recently closed tabs. Max 10 items (`RECENTLY_CLOSED_MAX`).

## Init Sequence

### Startup Order
1. SharedPreferences loaded (async)
2. Megazord (Rust) initialized
3. Places dependency provider initialized
4. Logins dependency provider initialized
5. Autofill dependency provider initialized
6. Session state restored from JSON
7. Downloads restored
8. After visual completeness: warmUp() all storages
9. Register maintenance workers

### Maintenance Workers
```kotlin
historyStorage.registerStorageMaintenanceWorker()    // Periodic Places maintenance
passwordsStorage.registerStorageMaintenanceWorker()  // Periodic Logins maintenance
autofillStorage.registerStorageMaintenanceWorker()   // Periodic Autofill maintenance
```

## Global Dependency Providers

These make storage accessible to WorkManager tasks and other processes:

```kotlin
GlobalPlacesDependencyProvider.initialize(historyStorage)
GlobalLoginsDependencyProvider.initialize(lazy { passwordsStorage })
GlobalAutofillDependencyProvider.initialize(lazy { autofillStorage })
GlobalSyncedTabsCommandsProvider.initialize(lazy { syncedTabsCommands })
GlobalFxSuggestDependencyProvider.initialize(fxSuggest.storage)
GlobalRemoteSettingsDependencyProvider.initialize(remoteSettingsService)
GlobalAddonDependencyProvider.initialize(addonManager, addonUpdater)
```

## Key Files

| File | Role |
|------|------|
| `A-C/.../browser/storage/sync/PlacesHistoryStorage.kt` | History storage |
| `A-C/.../browser/storage/sync/PlacesBookmarksStorage.kt` | Bookmarks storage |
| `A-C/.../service/sync/logins/SyncableLoginsStorage.kt` | Encrypted login storage |
| `A-C/.../service/sync/autofill/AutofillCreditCardsAddressesStorage.kt` | Autofill storage |
| `A-C/.../browser/storage/sync/RemoteTabsStorage.kt` | Synced tabs storage |
| `A-C/.../browser/session-storage/SessionStorage.kt` | JSON session persistence |
| `A-C/.../browser/session-storage/AutoSave.kt` | Periodic auto-save |
| `A-C/.../browser/thumbnails/storage/ThumbnailStorage.kt` | Thumbnail disk storage |
| `A-C/.../browser/icons/utils/IconDiskCache.kt` | Icon disk cache |
| `A-C/.../feature/recentlyclosed/RecentlyClosedTabsStorage.kt` | Closed tabs |
| `A-C/.../lib/dataprotect/SecureAbove22Preferences.kt` | Encrypted prefs |
| `fenix/.../utils/Settings.kt` | User preferences wrapper |
| `fenix/.../tabgroups/storage/repository/TabGroupRepository.kt` | Room DB tab groups |
| `fenix/.../components/Core.kt` | Storage creation + wiring |
| `fenix/.../FenixApplication.kt` (line 371-376) | Global dependency init |
| `support/base/AppServicesInitializer.kt` | Megazord/AppServices init |

(A-C = `android-components` root)

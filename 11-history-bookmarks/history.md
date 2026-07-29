# History

## Purpose
Browser history records visited URLs, visit timestamps, page titles, and preview images. Fenix uses Places (Mozilla Application Services) for history storage, which provides sync-compatible storage backed by SQLite via Rust.

## History Architecture

```
GeckoSession (navigation)
  → HistoryDelegate (A-C feature/session)
    → PlacesHistoryStorage (A-C browser/storage-sync)
      → Places Rust library (SQLite)
        → Sync (Firefox Account)
```

## HistoryDelegate

File: `A-C/.../feature/session/HistoryDelegate.kt`

```kotlin
class HistoryDelegate(
    private val historyStorage: Lazy<HistoryStorage>,
) : HistoryTrackingDelegate
```

### Callbacks
- `onVisited(uri, visitType, lastVisited, redirectSources)` → `historyStorage.recordVisit(uri, visit)`
- `onTitleChanged(result, title)` → `historyStorage.recordObservation(uri, PageObservation(title=title))`
- `onPreviewImageChange(uri, previewImageUrl)` → records observation with preview URL
- `getVisited(uris)` → `historyStorage.getVisited(uris)` for link coloring
- `shouldStoreUri(uri)` → filters out invalid URIs

### Filtering in GeckoEngineSession
- Private mode visits filtered out at the delegate level
- Error pages not recorded
- Non-top-level loads (iframes) not recorded
- App redirect URLs filtered out

## History Storage (Places)

### PlacesHistoryStorage
```kotlin
PlacesHistoryStorage(context, crashReporter)
```

### Key Operations
| Method | Purpose |
|--------|---------|
| `recordVisit(uri, visitType)` | Record a page visit |
| `recordObservation(uri, pageObservation)` | Record title + preview image |
| `getVisited(uris)` | Bulk check visited status |
| `getDetailedVisits(start, end)` | Get visit list for UI |
| `deleteVisit(uri, startTime)` | Remove single visit |
| `deleteVisitionsSince(timestamp)` | Clear history since time |
| `deleteAll()` | Clear all history |
| `warmUp()` | Pre-warm storage |
| `registerStorageMaintenanceWorker()` | Periodic Places maintenance |

## History UI

### HistoryFragment
File: `fenix/.../library/HistoryFragment.kt`

Displays visited pages grouped by time:
- Today
- Yesterday
- Last 7 days
- Last 30 days
- Older

Features: search, multi-select delete, open in tab, share, bookmark.

### PagedHistoryProvider
File: `fenix/.../components/history/PagedHistoryProvider.kt`

Wrapper for paged history data access.

## History Metadata

File: `fenix/.../historymetadata/`

### DefaultHistoryMetadataService
```kotlin
class DefaultHistoryMetadataService(storage: PlacesHistoryStorage) : HistoryMetadataService
```

Records metadata about browsing sessions:
- Duration on page
- Search queries associated with page
- Navigation patterns

### HistoryMetadataMiddleware
File: `fenix/.../historymetadata/HistoryMetadataMiddleware.kt`

BrowserStore middleware that records history metadata on navigation events.

### History Metadata Cleanup
```kotlin
historyMetadataService.cleanup(cutoffDate)
// 14-day cutoff (HISTORY_METADATA_MAX_AGE_IN_MS)
```

### Recently Visited
Shown on home screen via `RecentlyVisitedSection`. Uses history metadata to show pages from recent browsing sessions.

## History Sync

Synced via `SyncEngine.History`. Requires FxA account and history sync enabled.

## Delete Browsing Data

Included in the "Delete Browsing Data" flow:
```kotlin
historyStorage.deleteVisitionsSince(startTime)
historyStorage.deleteAll()
```

## Key Files

| File | Role |
|------|------|
| `A-C/.../feature/session/HistoryDelegate.kt` | GV → Storage bridge |
| `A-C/.../browser/storage/sync/PlacesHistoryStorage.kt` | History storage |
| `A-C/.../engine-gecko/GeckoEngineSession.kt` (line 1304) | History delegate creation |
| `fenix/.../library/HistoryFragment.kt` | History UI |
| `fenix/.../historymetadata/DefaultHistoryMetadataService.kt` | Metadata service |
| `fenix/.../historymetadata/HistoryMetadataMiddleware.kt` | Metadata recording |
| `fenix/.../components/history/PagedHistoryProvider.kt` | Paged access |
| `fenix/.../home/recentvisits/` | Recently visited on home |
| `fenix/.../settings/DeleteBrowsingDataFragment.kt` | Clear history UI |

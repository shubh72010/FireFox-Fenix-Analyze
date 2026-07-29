# Bookmarks

## Purpose
Bookmarks provide persistent storage of user-saved URLs organized in folders. Fenix uses Places (Mozilla Application Services) for bookmark storage, supporting mobile and desktop bookmark roots, folders, separators, and sync.

## Bookmarks Architecture

```
Bookmarks UI (Fenix)
  ↕
BookmarksUseCases (A-C feature/bookmarks)
  ↕
PlacesBookmarksStorage (A-C browser/storage-sync)
  ↕
Places Rust library (SQLite)
  ↕
Sync (Firefox Account)
```

## Bookmark Storage (Places)

### PlacesBookmarksStorage
```kotlin
PlacesBookmarksStorage(context)
```

### Key Operations
| Method | Purpose |
|--------|---------|
| `addItem(guid, url, title, position)` | Add bookmark |
| `addFolder(guid, parent, title, position)` | Add folder |
| `addSeparator(guid, parent, position)` | Add separator |
| `updateNode(guid, title, url)` | Update bookmark |
| `deleteNode(guid)` | Remove bookmark/folder |
| `getTree(guid)` | Get folder tree |
| `search(query, limit)` | Search bookmarks |
| `getBookmarks(limit, offset)` | Paged bookmarks |
| `warmUp()` | Pre-warm storage |

### Bookmark Roots
- `MobileRootGuid` - Mobile bookmarks
- `DesktopRootGuid` - Desktop bookmarks (from sync)
- `UnfiledRootGuid` - Unfiled bookmarks
- `ToolbarRootGuid` - Toolbar bookmarks
- `MenuRootGuid` - Menu bookmarks

## Bookmarks UI

### BookmarkFragment
File: `fenix/.../bookmarks/BookmarkFragment.kt`

Displays bookmarks in a tree structure:
- Expandable folders
- Search
- Multi-select operations
- Edit, delete, share

### Redux Architecture
Bookmark management uses Redux within the AppStore:

```kotlin
// Actions (BookmarksAction.kt)
sealed class BookmarkAction {
    data class Add(val bookmark: Bookmark)
    data class Remove(val guid: String)
    data class Update(val bookmark: Bookmark)
    data class Move(val guid: String, val toParent: String, val position: Int)
}

// State (AppState.bookmarks)
data class Bookmark(
    val guid: String,
    val title: String,
    val url: String?,
    val parent: String,
    val position: Int,
    val children: List<Bookmark>?,
)
```

### Middleware
| Middleware | Purpose |
|-----------|---------|
| `BookmarksMiddleware` | Core bookmark operations |
| `BookmarksSyncMiddleware` | Sync integration |
| `BookmarksTelemetryMiddleware` | Telemetry events |
| `BrowserToolbarSyncToBookmarksMiddleware` | Toolbar bookmark state sync |
| `PrivateBrowsingLockMiddleware` | Restrict in private mode |

### Sub-fragments
| Screen | Fragment | Purpose |
|--------|----------|---------|
| Bookmarks Root | `BookmarkFragment` | Full tree view |
| Add Folder | `AddFolderScreen` (Compose) | Create folder |
| Edit Folder | `EditFolderScreen` (Compose) | Rename folder |
| Select Folder | `SelectFolderScreen` (Compose) | Choose target folder |
| Edit Bookmark | `EditBookmarkFragment` | Edit title/URL |

## Bookmark Operations

### Adding a Bookmark
1. User taps bookmark icon in toolbar
2. `BrowserToolbarSyncToBookmarksMiddleware` dispatches `BookmarkAction.Add`
3. `BookmarksMiddleware` calls `PlacesBookmarksStorage.addItem()`
4. State updated in `AppStore`

### Import Bookmarks
`FenixBookmarkImporterError`, `FenixImporterEvent` handle importing bookmarks from other browsers.

### Desktop Folders
`DesktopFolders.kt` manages detection and display of desktop-originated bookmark folders.

## Bookmarks on Home Screen

`BookmarksSection` on the home screen shows recently modified bookmarks from `AppStore.bookmarks`.

## Bookmarks Sync

Synced via `SyncEngine.Bookmarks`. Bookmark state is automatically synced when Firefox Account is connected. Desktop bookmarks appear in `DesktopRootGuid` folder.

## Delete Browsing Data

Bookmarks are NOT cleared by "Delete Browsing Data" (they are permanent user data). Users must delete bookmarks individually or clear via bookmarks management screen.

## Key Files

| File | Role |
|------|------|
| `fenix/.../bookmarks/BookmarkFragment.kt` | Bookmark tree UI |
| `fenix/.../bookmarks/BookmarksAction.kt` | Redux actions |
| `fenix/.../bookmarks/BookmarksState.kt` | Redux state |
| `fenix/.../bookmarks/BookmarksStore.kt` | Bookmark store creation |
| `fenix/.../bookmarks/BookmarksReducer.kt` | State reducer |
| `fenix/.../bookmarks/BookmarksMiddleware.kt` | Core operations |
| `fenix/.../bookmarks/BookmarksSyncMiddleware.kt` | Sync integration |
| `fenix/.../bookmarks/BookmarksTelemetryMiddleware.kt` | Telemetry |
| `fenix/.../bookmarks/DesktopFolders.kt` | Desktop folder handling |
| `fenix/.../bookmarks/importer/` | Bookmark import |
| `fenix/.../bookmarks/ui/` | Compose UI screens |
| `fenix/.../components/bookmarks/BookmarksUseCase.kt` | Use cases |
| `A-C/.../browser/storage/sync/PlacesBookmarksStorage.kt` | Storage layer |

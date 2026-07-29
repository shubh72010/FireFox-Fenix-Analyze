# Downloads

## Purpose
Download management in Fenix spans a foreground service for download execution, Redux middleware for state tracking, and a full Compose-based download list screen.

## Architecture

```
GeckoView Download Request
  → ContentDelegate.onDownload()
    → EngineObserver
      → BrowserStore (DownloadState update)
        → DownloadMiddleware (manages lifecycle)
          → DownloadService (Foreground Service, downloads file)
          → DownloadSnackbar (UI notifications)
          → DownloadFragment (list screen)
```

## Download Flow

1. Page triggers download → `GeckoEngineSession.ContentDelegate.onDownload()`
2. `EngineObserver` creates `DownloadState` and dispatches to `BrowserStore`
3. `DownloadMiddleware` starts the download via `AbstractFetchDownloadService`
4. `DownloadService` runs in foreground, downloads file, reports progress
5. `DownloadState` updates through middleware → `BrowserStore`
6. `DownloadSnackbar` observes state and shows completion/cancellation snackbar
7. `DownloadFragment` shows all downloads with sort/filter/search

## DownloadService

```kotlin
class DownloadService : AbstractFetchDownloadService() {
    override val httpClient: Client = components.core.client
    override val store: BrowserStore = components.core.store
    override val notificationsDelegate = components.notificationsDelegate
    override val fileSizeFormatter = components.core.fileSizeFormatter
    override val downloadEstimator = components.core.downloadEstimator
    override val downloadFileUtils = DefaultDownloadFileUtils(
        context, downloadLocation = {
            DownloadLocationManager(settings, contentResolver).defaultLocation
        }
    )
    override val downloadFileWriter = DefaultDownloadFileWriter()
}
```

Extends `AbstractFetchDownloadService` from android-components which handles:
- File download from URL to device storage
- Foreground service notification with progress
- Pause/resume/cancel support
- Download estimation (time remaining, file size)

## DownloadMiddleware

Created in `Core.kt`:
```kotlin
DownloadMiddleware(
    applicationContext = context,
    downloadServiceClass = DownloadService::class.java,
    deleteFileFromStorage = { settings.deleteDownloadBehavior == DELETE_FROM_DEVICE },
    downloadFileUtils = DefaultDownloadFileUtils(...),
)
```

Intercepts `DownloadAction` in `BrowserStore` to manage download lifecycle.

## Download Use Cases

```kotlin
class DownloadUseCases(
    private val store: BrowserStore,
    private val downloadFileUtils: DownloadFileUtils,
) {
    val removeDownload: RemoveDownloadUseCase
    val removeAllDownloads: RemoveAllDownloadsUseCase
    val restoreDownloads: RestoreDownloadsUseCase
}
```

`restoreDownloads()` is called during app startup to rehydrate in-progress downloads.

## Download List Screen (Compose)

### Store
`DownloadUIStore` is a Redux-style store with 7 middleware:

| Middleware | Purpose |
|-----------|---------|
| DownloadUIMapperMiddleware | BrowserState → FileItem list |
| DownloadUIShareMiddleware | Share intent handling |
| DownloadTelemetryMiddleware | Glean metrics |
| DownloadDeleteMiddleware | Deletion with undo |
| DownloadsServiceCommunicationMiddleware | Pause/resume/cancel via service intents |
| DownloadNavigationMiddleware | Back/settings navigation |
| DownloadUIRenameMiddleware | File rename operations |

### State (DownloadUIState)
```kotlin
data class DownloadUIState(
    val items: List<FileItem>,
    val mode: UIMode,                    // Normal | Editing
    val searchQuery: String,
    val fileToRename: FileItem?,
    val dialogState: DialogState?,
    val deletionSnackbarState: SnackbarState?,
)
```

### FileItem Model
```kotlin
data class FileItem(
    val id: String,
    val url: String,
    val fileName: String,
    val filePath: String,
    val directoryPath: String,
    val contentType: ContentType,          // All, Image, Video, Document, Other
    val status: Status,                    // Initiated, Downloading, Paused, Cancelled, Failed, Completed
    val timeCategory: TimeCategory,        // IN_PROGRESS, TODAY, YESTERDAY, etc.
)
```

### Content Type Filters
All, Image, Video, Document, Other

### Time Categories
IN_PROGRESS, TODAY, YESTERDAY, LAST_7_DAYS, LAST_30_DAYS, OLDER

## Download Location

```kotlin
class DownloadLocationManager(
    settings: Settings,
    contentResolver: ContentResolver,
) {
    val defaultLocation: DownloadLocation
        // Uses settings.downloadDirectory
        // Default: Environment.DIRECTORY_DOWNLOADS
}
```

Configurable via settings. Defaults to standard Downloads directory.

## Delete Behavior

```kotlin
enum DeleteDownloadBehavior {
    DELETE_FROM_DEVICE,    // Actually delete the file
    REMOVE_FROM_LIST,      // Only remove from download list
}
```

## Key Files

| File | Role |
|------|------|
| `fenix/.../downloads/DownloadService.kt` | Foreground download service |
| `fenix/.../downloads/DownloadSnackbar.kt` | Download notifications |
| `fenix/.../downloads/DownloadsUtils.kt` | Download utilities |
| `fenix/.../downloads/listscreen/DownloadFragment.kt` | Download list fragment |
| `fenix/.../downloads/listscreen/store/DownloadUIStore.kt` | Downloads Redux store |
| `fenix/.../downloads/listscreen/middleware/` | 7 middleware implementations |
| `fenix/.../downloads/dialog/DownloadAppDialog.kt` | Handle with app dialog |
| `fenix/.../components/UseCases.kt` | DownloadUseCases creation |
| `fenix/.../components/Core.kt` (line 350) | DownloadMiddleware in BrowserStore |
| `fenix/.../FenixApplication.kt` (line 458) | restoreDownloads() |
| `fenix/.../settings/downloads/DownloadLocationManager.kt` | Location config |
| `A-C/.../feature/downloads/AbstractFetchDownloadService.kt` | Base download service |
| `A-C/.../feature/downloads/DownloadMiddleware.kt` | Download state middleware |

(A-C = `android-components` root)

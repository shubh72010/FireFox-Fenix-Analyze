# Permissions

## Purpose
Fenix handles two types of permissions: Android runtime permissions (camera, microphone, location) and website permissions (notifications, geolocation, storage access, autoplay, media). The site permissions system bridges GeckoView's storage with Fenix's on-disk persistence.

## Permission Architecture

```
Website requests permission (e.g., geolocation)
  → GeckoSession.PermissionDelegate.onContentPermissionRequest()
    → GeckoEngineSession (creates GeckoPermissionRequest)
      → EngineObserver (notifies observers)
        → SitePermissionsFeature (A-C, creates prompt dialog)
          → User allows/denies
            → GeckoSitePermissionsStorage.save() (dual: GV + disk)
```

## SitePermissionsFeature

Created in `BaseBrowserFragment.kt`:
```kotlin
val sitePermissionsFeature = ViewBoundFeatureWrapper<SitePermissionsFeature>()
// In initializeUI:
sitePermissionsFeature.set(
    SitePermissionsFeature(
        context = context,
        engine = components.core.engine,
        store = components.core.store,
        sessionId = sessionId,
        promptsStyling = PromptsStyling(...),
        onNeedToRequestPermissions = { activity.requestPermissions(...) },
        learnMoreUrlProvider = FenixSitePermissionLearnMoreUrlProvider,
    ),
    view,
    owner,
)
```

## Permission Types

### GeckoView-Managed (ContentPermission)
| Permission | GeckoView Storage | On-Disk Storage |
|-----------|------------------|-----------------|
| Geolocation | Yes | Yes |
| Notification | Yes | Yes |
| Storage Access | Yes | Yes |
| AutoPlay | Yes | Yes |
| Persistent Storage | Yes | Yes |
| Cross-Origin Storage Access | Yes | Yes |

### On-Disk Only (not GeckoView-managed)
| Permission | Storage |
|-----------|---------|
| Media (Camera/Mic) | OnDiskSitePermissionsStorage |
| Local Network Access | OnDiskSitePermissionsStorage |
| Local Device Access | OnDiskSitePermissionsStorage |

## GeckoSitePermissionsStorage (Dual Storage)

```kotlin
class GeckoSitePermissionsStorage(
    runtime: GeckoRuntime,
    private val onDiskStorage: SitePermissionsStorage,  // OnDiskSitePermissionsStorage
) : SitePermissionsStorage
```

### Operations
- `save()`: Updates GeckoView `StorageController.setPermission()` AND on-disk storage
- `saveTemporary()`: In-memory only (for "don't remember my decision")
- `findSitePermissionsBy()`: Queries both storages, merges into one `SitePermissions`
- `remove()`: Clears from both

### Status Mapping
| GeckoView | Fenix |
|-----------|-------|
| VALUE_ALLOW | ALLOWED |
| VALUE_DENY | BLOCKED |
| VALUE_PROMPT | NO_DECISION |

## PermissionStorage (Fenix Wrapper)

```kotlin
class PermissionStorage(context: Context) {
    private val geckoSitePermissionsStorage = GeckoSitePermissionsStorage(
        geckoRuntime,
        OnDiskSitePermissionsStorage(context)
    )
    // Provides:
    fun add(sitePermissions, private)
    fun findSitePermissionsBy(origin, private)
    fun updateSitePermissions(sitePermissions, private)
    fun getSitePermissionsPaged()  // DataSource.Factory for paged display
    fun deleteSitePermissions(sitePermissions)
    fun deleteAllSitePermissions()
}
```

All operations run on `Dispatchers.IO`.

## Permission Prompts Flow

1. Engine detects permission request → `PermissionDelegate.onContentPermissionRequest()`
2. `GeckoEngineSession` creates `GeckoPermissionRequest.Content`
3. `EngineObserver` dispatches to `SitePermissionsFeature`
4. `SitePermissionsFeature` checks stored permission:
   - If ALLOWED/BLOCKED with existing decision: auto-responds
   - If NO_DECISION: shows prompt dialog (Fenix-styled bottom sheet)
5. User responds → `SitePermissionsFeature` stores via `GeckoSitePermissionsStorage`
6. Response sent to GeckoView → to web page

### Media Permissions
`onMediaPermissionRequest()` → `GeckoPermissionRequest.Media` with video/audio sources. Separate prompt flow with source selection.

### Android Permissions
`onAndroidPermissionsRequest()` → `GeckoPermissionRequest.App` → `ActivityCompat.requestPermissions()`

## Learn More URLs

`FenixSitePermissionLearnMoreUrlProvider` maps permissions to help articles:
- `ContentCrossOriginStorageAccess` → MDN Storage Access API
- `ContentLocalNetworkAccess` / `ContentLocalDeviceAccess` → SUMO LNA article
- Others → null

## Site Permissions Settings

File: `fenix/.../settings/sitepermissions/`

Users can view and manage all stored permissions per site:
- `SiteSettingsFragment` - list of sites with permissions
- `SitePermissionsDetailsExceptionsFragment` - per-site permission details
- Permissions categories: camera, microphone, location, notification, storage, autoplay, LNA

## Delete Browsing Data

`DeleteBrowsingDataController` includes `PermissionStorage.deleteAllSitePermissions()` as part of clearing browsing data.

## Key Files

| File | Role |
|------|------|
| `fenix/.../components/PermissionStorage.kt` | Fenix permission storage wrapper |
| `fenix/.../browser/permissions/FenixSitePermissionLearnMoreUrlProvider.kt` | Help URL provider |
| `fenix/.../browser/BaseBrowserFragment.kt` (line 312, 1235) | SitePermissionsFeature setup |
| `fenix/.../components/Core.kt` (line 313, 662) | Storage creation |
| `fenix/.../settings/sitepermissions/` | Permission management UI |
| `A-C/.../engine-gecko/permission/GeckoSitePermissionsStorage.kt` | Dual storage bridge |
| `A-C/.../engine-gecko/GeckoEngineSession.kt` (line 1642) | PermissionDelegate creation |
| `A-C/.../feature/sitepermissions/SitePermissionsFeature.kt` | Permission prompt feature |
| `A-C/.../feature/sitepermissions/OnDiskSitePermissionsStorage.kt` | On-disk persistence |

(A-C = `android-components` root)

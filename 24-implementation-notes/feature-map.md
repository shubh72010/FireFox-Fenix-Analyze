# Feature Map

## Purpose
A comprehensive reference mapping every major feature in Firefox Fenix to its implementation location, key classes, and related documentation.

## How to Use
Features are grouped by domain. Each entry lists: feature name, implementation location, key classes, and cross-reference to other docs.

---

## Core Browser

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Browser Engine | A-C `browser/engine-gecko/` | GeckoEngine, GeckoEngineSession, GeckoEngineView | [GeckoView](../04-geckoview/geckoview-integration.md) |
| Browser State | A-C `browser/state/` | BrowserStore, BrowserState, TabSessionState | [State Management](../09-state-management/state-management.md) |
| Tab Management | A-C `feature/tabs/` | TabsUseCases, AddNewTabUseCase, RestoreUseCase | [Tab Management](../05-tabs/tab-management.md) |
| Session Storage | A-C `browser/session-storage/` | SessionStorage, AutoSave, BrowserStateWriter/Reader | [Tab Management](../05-tabs/tab-management.md) |
| Toolbar | A-C `browser/toolbar/` | BrowserToolbar, BrowserToolbarStore | [UI Architecture](../08-ui/ui-architecture.md) |
| Browsing Mode | `fenix/.../browser/browsingmode/` | BrowsingModeManager, DefaultBrowsingModeManager | [Tab Management](../05-tabs/tab-management.md) |
| Tab Strip | `fenix/.../browser/tabstrip/` | TabStrip, TabStripState, BrowserState.toTabStripState() | [Tab Management](../05-tabs/tab-management.md) |
| Tab Tray | `fenix/.../tabstray/` | TabManagementFragment, TabsTrayStore, TabStorageMiddleware | [Tab Management](../05-tabs/tab-management.md) |

## Navigation

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Navigation Graph | `fenix/app/src/main/res/navigation/` | nav_graph.xml, NavGraphDirections | [Navigation](../07-navigation/navigation.md) |
| Intent Processing | `fenix/.../intent/` | IntentProcessors, HomeDeepLinkIntentProcessor | [Navigation](../07-navigation/navigation.md) |
| Custom Tabs | `fenix/.../customtabs/` | ExternalAppBrowserActivity, ExternalAppBrowserFragment | [Navigation](../07-navigation/navigation.md) |

## Startup

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Application Init | `fenix/.../FenixApplication.kt` | FenixApplication, initializeFenixProcess() | [Startup](../03-startup/startup.md) |
| Components | `fenix/.../components/` | Components, Core, UseCases | [Architecture](../01-architecture/architecture.md) |
| Visual Completeness | `fenix/.../perf/PerformanceComponent.kt` | PerformanceComponent, RunWhenReadyQueue | [Performance](../19-performance/performance.md) |
| StartupTimeline | `fenix/.../perf/StartupTimeline.kt` | StartupTimeline | [Performance](../19-performance/performance.md) |
| Session Restore | `fenix/.../FenixApplication.kt` (line 444) | restoreBrowserState() | [Startup](../03-startup/startup.md) |
| Download Restore | `fenix/.../FenixApplication.kt` (line 458) | restoreDownloads() | [Downloads](../12-downloads/downloads.md) |

## State Management

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| BrowserStore | A-C `browser/state/store/` | BrowserStore, BrowserAction | [State Management](../09-state-management/state-management.md) |
| AppStore | `fenix/.../components/AppStore.kt` | AppStore, AppState, AppAction | [State Management](../09-state-management/state-management.md) |
| TabsTrayStore | `fenix/.../tabstray/TabsTrayStore.kt` | TabsTrayStore, TabsTrayState | [Tab Management](../05-tabs/tab-management.md) |
| Middleware | A-C `lib/state/` | Middleware, Store, AbstractBinding | [State Management](../09-state-management/state-management.md) |

## Memory

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| onTrimMemory | `fenix/.../FenixApplication.kt` (line 759) | FenixApplication.onTrimMemory() | [Memory](../06-memory/memory-management.md) |
| TrimMemoryMiddleware | A-C `browser/state/engine/middleware/` | TrimMemoryMiddleware (disabled) | [Memory](../06-memory/memory-management.md) |
| Session Prioritization | A-C `browser/state/engine/middleware/` | SessionPrioritizationMiddleware | [Memory](../06-memory/memory-management.md) |
| Icon Cache | A-C `browser/icons/utils/` | IconMemoryCache, IconDiskCache | [Memory](../06-memory/memory-management.md) |
| Thumbnail Cache | A-C `browser/thumbnails/` | ThumbnailStorage, ThumbnailDiskCache | [Memory](../06-memory/memory-management.md) |
| MemoryController | `geckoview/.../MemoryController.java` | MemoryController | [Memory](../06-memory/memory-management.md) |

## Privacy & Security

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Tracking Protection | `fenix/.../components/TrackingProtectionPolicyFactory.kt` | TrackingProtectionPolicyFactory | [Privacy](../13-privacy-security/privacy-and-security.md) |
| ETP Dashboard | `fenix/.../trackingprotection/` | ProtectionsDashboardFragment, TrackersBlockedFeature | [Privacy](../13-privacy-security/privacy-and-security.md) |
| HTTPS-Only | `fenix/.../settings/HttpsOnlyFragment.kt` | HttpsOnlyFragment, Settings.getHttpsOnlyMode() | [Privacy](../13-privacy-security/privacy-and-security.md) |
| DoH | `fenix/.../settings/doh/` | DohSettingsStore, DohSettingsProvider | [Privacy](../13-privacy-security/privacy-and-security.md) |
| Cookie Banner | `fenix/.../utils/Settings.kt` (line 1245) | Settings.getCookieBannerHandling() | [Privacy](../13-privacy-security/privacy-and-security.md) |
| Fingerprinting | `fenix/.../components/Core.kt` (line 222) | FxNimbus.features.fingerprintingProtection | [Privacy](../13-privacy-security/privacy-and-security.md) |
| IP Protection | `fenix/.../components/ipprotection/` | IPProtection, IPProtectionFeature | [Privacy](../13-privacy-security/privacy-and-security.md) |
| GPC | `fenix/.../utils/Settings.kt` | shouldEnableGlobalPrivacyControl | [Privacy](../13-privacy-security/privacy-and-security.md) |

## Permissions

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Permission Storage | `fenix/.../components/PermissionStorage.kt` | PermissionStorage | [Permissions](../14-permissions/permissions.md) |
| Site Permissions | A-C `feature/sitepermissions/` | SitePermissionsFeature, OnDiskSitePermissionsStorage | [Permissions](../14-permissions/permissions.md) |
| Gecko Permissions | A-C `engine-gecko/permission/` | GeckoSitePermissionsStorage | [Permissions](../14-permissions/permissions.md) |
| Permission Prompts | A-C `feature/prompts/` | PromptFeature, GeckoPromptDelegate | [Permissions](../14-permissions/permissions.md) |

## Passkeys / WebAuthn

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| JS API | `dom/webauthn/` | WebAuthnHandler.h/.cpp | [Passkeys](../16-passkeys-webauthn/passkeys-webauthn.md) |
| IPC | `dom/webauthn/PWebAuthnTransaction.ipdl` | WebAuthnTransactionParent | [Passkeys](../16-passkeys-webauthn/passkeys-webauthn.md) |
| Service | `dom/webauthn/nsIWebAuthnService.idl` | WebAuthnService, AndroidWebAuthnService | [Passkeys](../16-passkeys-webauthn/passkeys-webauthn.md) |
| Token Manager | `geckoview/.../WebAuthnTokenManager.java` | WebAuthnTokenManager | [Passkeys](../16-passkeys-webauthn/passkeys-webauthn.md) |
| Credential Manager | `geckoview/.../WebAuthnCredentialManager.java` | WebAuthnCredentialManager | [Passkeys](../16-passkeys-webauthn/passkeys-webauthn.md) |
| Fenix Integration | A-C `feature/webauthn/` | WebAuthnFeature | [Passkeys](../16-passkeys-webauthn/passkeys-webauthn.md) |
| Related Origin Prompt | A-C `feature/prompts/` | WebAuthnRelatedOriginDialogFragment | [Passkeys](../16-passkeys-webauthn/passkeys-webauthn.md) |

## Extensions

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Add-on Manager | A-C `feature/addons/` | AddonManager, DefaultAddonUpdater | [Extensions](../15-extensions/extensions.md) |
| AMO Integration | A-C `feature/addons/amo/` | AMOAddonsProvider | [Extensions](../15-extensions/extensions.md) |
| Extension UI | `fenix/.../addons/` | AddonsManagementFragment, AddonDetailsFragment | [Extensions](../15-extensions/extensions.md) |
| WebExtension Support | A-C `support/webextensions/` | WebExtensionSupport | [Extensions](../15-extensions/extensions.md) |
| GeckoWebExtension | A-C `engine-gecko/` | GeckoWebExtension | [Extensions](../15-extensions/extensions.md) |
| Built-in Extensions | `fenix/.../components/Core.kt` | icons, adsTelemetry, searchTelemetry | [Extensions](../15-extensions/extensions.md) |

## Sync

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| FxA Manager | A-C `service/fxa/manager/` | FxaAccountManager | [Sync](../17-sync/sync.md) |
| BackgroundServices | `fenix/.../components/BackgroundServices.kt` | BackgroundServices | [Sync](../17-sync/sync.md) |
| Synced Tabs | `fenix/.../sync/SyncedTabsIntegration.kt` | SyncedTabsStorage | [Sync](../17-sync/sync.md) |
| Send Tab | A-C `service/fxa/SendTabFeature.kt` | SendTabFeature | [Sync](../17-sync/sync.md) |
| Push Integration | `fenix/.../push/` | PushFxaIntegration, WebPushEngineIntegration | [Sync](../17-sync/sync.md) |

## Downloads

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Download Service | `fenix/.../downloads/DownloadService.kt` | DownloadService | [Downloads](../12-downloads/downloads.md) |
| Download List | `fenix/.../downloads/listscreen/` | DownloadFragment, DownloadUIStore | [Downloads](../12-downloads/downloads.md) |
| Download Middleware | A-C `feature/downloads/` | DownloadMiddleware | [Downloads](../12-downloads/downloads.md) |
| Location Manager | `fenix/.../settings/downloads/` | DownloadLocationManager | [Downloads](../12-downloads/downloads.md) |

## Storage

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Places History | A-C `browser/storage-sync/` | PlacesHistoryStorage | [Storage](../10-storage/storage-layer.md) |
| Places Bookmarks | A-C `browser/storage-sync/` | PlacesBookmarksStorage | [Storage](../10-storage/storage-layer.md) |
| Logins Storage | A-C `service/sync-logins/` | SyncableLoginsStorage | [Storage](../10-storage/storage-layer.md) |
| Autofill Storage | A-C `service/sync-autofill/` | AutofillCreditCardsAddressesStorage | [Storage](../10-storage/storage-layer.md) |
| Tab Groups (Room) | `fenix/.../tabgroups/storage/` | TabGroupRepository, TabGroupDatabase | [Tab Management](../05-tabs/tab-management.md) |
| Thumbnail Storage | A-C `browser/thumbnails/` | ThumbnailStorage | [Storage](../10-storage/storage-layer.md) |
| Icon Storage | A-C `browser/icons/` | BrowserIcons, IconDiskCache | [Storage](../10-storage/storage-layer.md) |

## UI

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| Home Screen | `fenix/.../home/` | HomeFragment, Homepage, HomepageState | [UI Architecture](../08-ui/ui-architecture.md) |
| Settings | `fenix/.../settings/` | SettingsFragment, Settings.kt | [Settings](../22-settings/settings.md) |
| Theming | `fenix/.../theme/` | FirefoxTheme, ThemeManager, Theme | [UI Architecture](../08-ui/ui-architecture.md) |
| Compose Components | `fenix/.../compose/` | Banner, Menu, Snackbar, ListItem | [UI Architecture](../08-ui/ui-architecture.md) |
| Menus | `fenix/.../components/menu/compose/` | MainMenu, MenuDialogBottomSheet | [UI Architecture](../08-ui/ui-architecture.md) |

## Performance

| Feature | Location | Key Classes | Docs |
|---------|----------|-------------|------|
| StartupTimeline | `fenix/.../perf/StartupTimeline.kt` | StartupTimeline | [Performance](../19-performance/performance.md) |
| Visual Completeness | `fenix/.../perf/PerformanceComponent.kt` | RunWhenReadyQueue | [Performance](../19-performance/performance.md) |
| Profile Markers | `fenix/.../perf/ProfileMarkerMiddleware.kt` | ProfileMarkerMiddleware | [Performance](../19-performance/performance.md) |
| lazyMonitored | `fenix/.../perf/lazyMonitored.kt` | lazyMonitored() | [Performance](../19-performance/performance.md) |

## Build

| Feature | Location | Key Docs |
|---------|----------|----------|
| Gradle Build | `fenix/build.gradle.kts` | [Build System](../02-build-system/build-system.md) |
| App Module | `fenix/app/build.gradle.kts` | [Build System](../02-build-system/build-system.md) |
| GeckoView Build | `mobile/android/geckoview/build.gradle` | [Build System](../02-build-system/build-system.md) |
| Variants | `fenix/config/*.yml` | [Build System](../02-build-system/build-system.md) |

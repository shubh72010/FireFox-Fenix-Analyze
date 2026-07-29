# Extensions (WebExtensions / Add-ons)

## Purpose
Firefox for Android supports WebExtensions, the cross-browser extension API. Add-ons are managed through a combination of GeckoView's extension controller, android-components support libraries, and Fenix-specific UI.

## Architecture

```
Fenix Add-ons UI
    ↕
A-C AddonManager + AddonUpdater + AMOAddonsProvider
    ↕
GlobalAddonDependencyProvider
    ↕
WebExtensionSupport.initialize()
    ↕
GeckoEngine (WebExtensionRuntime)
    ↕
GeckoView WebExtensionController
    ↕
Gecko WebExtension Engine (C++)
```

## Extension Lifecycle

### Initialization
In `FenixApplication.initializeWebExtensionSupport()`:
```kotlin
GlobalAddonDependencyProvider.initialize(addonManager, addonUpdater)
WebExtensionSupport.initialize(engine, store,
    onNewTabOverride = { engineSession, url, selected ->
        tabsUseCases.addTab(url, selected, engineSession)
    },
    onCloseTabOverride = { sessionId ->
        tabsUseCases.removeTab(sessionId)
    },
    onSelectTabOverride = { sessionId ->
        tabsUseCases.selectTab(sessionId)
    },
    onExtensionsLoaded = { extensions ->
        addonUpdater.registerForFutureUpdates(extensions)
        subscribeForNewAddonsIfNeeded(supportedAddonsChecker, extensions)
    },
    onUpdatePermissionRequest = addonUpdater::onUpdatePermissionRequest,
)
```

### Installation Sources
1. **AMO Collection**: From `AMOAddonsProvider` configured per build variant
2. **Built-in**: Bundled extensions (installed via `installBuiltInWebExtension`)
3. **Custom Collection**: Overrideable in Nightly/Beta via settings

### AMOAddonsProvider Configuration
```kotlin
// Priority order:
if (customCollectionOverrideEnabled && collectionConfigured) {
    // User-configured collection (Nightly/Beta debug settings)
    AMOAddonsProvider(context, client, collectionUser, collectionName)
} else if (BuildConfig.AMO_COLLECTION_* configured) {
    // Build-variant collection
    AMOAddonsProvider(context, client, serverURL, collectionUser, collectionName)
} else {
    // Default AMO collection
    AMOAddonsProvider(context, client)
}
```

Cache: 2 days max cache age.

## Add-on Management

### AddonManager
```kotlin
class AddonManager(
    store: BrowserStore,
    engine: Engine,
    addonsProvider: AddonsProvider,
    addonUpdater: AddonUpdater,
)
```
Manages installed add-ons, coordinates between store, engine, provider, and updater.

### AddonUpdater
```kotlin
DefaultAddonUpdater(context, Frequency(12, HOURS), notificationsDelegate)
```
Checks for updates every 12 hours.

### SupportedAddonsChecker
```kotlin
DefaultSupportedAddonsChecker(context, Frequency(12, HOURS))
```
Checks if installed add-ons are still supported. Registers for checks only when unsupported add-ons exist.

## UI Screens

| Screen | Fragment | Purpose |
|--------|----------|---------|
| Add-ons Manager | `AddonsManagementFragment` | List installed + recommended add-ons |
| Add-on Details | `AddonDetailsFragment` | Permissions, description, enable/disable |
| Installed Details | `InstalledAddonDetailsFragment` | Settings for installed add-on |
| Permissions Details | `AddonPermissionsDetailsFragment` | View add-on permissions |
| Add-on Settings | `AddonInternalSettingsFragment` | Internal/developer settings |
| Not Yet Supported | `NotYetSupportedAddonFragment` | Unsupported add-on info |
| Action Popup | `WebExtensionActionPopupFragment` | Browser/page action popup |

## Extension Process Management

### Extension Process Controller
```kotlin
class ExtensionsProcessDisabledForegroundController(context, engine, store)
class ExtensionsProcessDisabledBackgroundController(context, engine, store)
```
These handle UI and telemetry when extension processes are disabled.

### Unsupported Add-on Detection
```kotlin
val hasUnsupportedAddons = installedExtensions.any { it.isUnsupported() }
if (hasUnsupportedAddons) checker.registerForChecks()
else checker.unregisterForChecks()
```

## WebExtension Messaging

### GeckoWebExtension
```kotlin
class GeckoWebExtension(internal val nativeExtension: WebExtension) : WebExtension {
    fun registerBackgroundMessageHandler(session, name, delegate)
    fun registerContentMessageHandler(session, name, delegate, port)
    // Port-based communication via GeckoNativeWebExtension.Port and GeckoPort
    // Browser/Page actions via ActionDelegate
    // Tab handling via TabDelegate / SessionTabDelegate
}
```

### Message Flow
```
Content Script → runtime.sendMessage()
    → GeckoSession.webExtensionController.setMessageDelegate()
        → GeckoWebExtension.contentMessageDelegate
            → A-C WebExtension support
                → Fenix extension handler
```

## Built-in Extensions

Fenix ships with built-in WebExtensions:
1. **Icons**: Auto-load website icons (`icons.install()` in Core.kt)
2. **Ads Telemetry**: Partner link tracking (`adsTelemetry.install()`)
3. **Search Telemetry**: SERP interaction tracking (`searchTelemetry.install()`)
4. **WebCompat**: Web compatibility mitigations (`WebCompatFeature.install()`)

## Key Files

| File | Role |
|------|------|
| `fenix/.../FenixApplication.kt` (line 848) | WebExtension initialization |
| `fenix/.../addons/AddonsManagementFragment.kt` | Add-ons list UI |
| `fenix/.../addons/AddonDetailsBindingDelegate.kt` | Details view binding |
| `fenix/.../components/Components.kt` (line 181) | AMOAddonsProvider creation |
| `fenix/.../extension/` | Fenix extension integration |
| `A-C/.../feature/addons/AddonManager.kt` | Add-on lifecycle management |
| `A-C/.../feature/addons/update/DefaultAddonUpdater.kt` | Background updates |
| `A-C/.../feature/addons/migration/DefaultSupportedAddonsChecker.kt` | Compatibility checks |
| `A-C/.../feature/addons/amo/AMOAddonsProvider.kt` | AMO API client |
| `A-C/.../support/webextensions/WebExtensionSupport.kt` | Tab handling support |
| `A-C/.../engine-gecko/GeckoWebExtension.kt` | GeckoView extension wrapper |
| `A-C/.../concept/engine/webextension/WebExtension.kt` | Extension abstraction |
| `geckoview/.../WebExtensionController.java` | GV extension lifecycle |

(A-C = `android-components` root)

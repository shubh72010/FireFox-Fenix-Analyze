# Sync (Firefox Sync)

## Purpose
Firefox Sync synchronizes browser data across devices. Fenix integrates with Mozilla's Application Services (via `FxaAccountManager`) to support syncing history, bookmarks, passwords, open tabs, credit cards, and addresses.

## Architecture

```
Fenix
  ↕
BackgroundServices
  ├── FxaAccountManager (authentication)
  ├── SyncEngine (periodic sync)
  ├── SyncedTabsStorage (remote tabs)
  ├── SendTabFeature (tab sharing)
  └── CloseTabsFeature (remote close)
    ↕
Mozilla Application Services (Rust)
  ├── Places (history + bookmarks)
  ├── Logins (encrypted)
  ├── Autofill (credit cards + addresses)
  └── Sync Manager
    ↕
Firefox Sync Server
```

## BackgroundServices

File: `fenix/.../components/BackgroundServices.kt`

### Initialization
```kotlin
class BackgroundServices(
    context: Context,
    push: Push,
    settings: Settings,
    crashReporter: CrashReporting,
    lazyHistoryStorage: Lazy<PlacesHistoryStorage>,
    lazyBookmarksStorage: Lazy<PlacesBookmarksStorage>,
    lazyPasswordsStorage: Lazy<SyncableLoginsStorage>,
    lazyRemoteTabsStorage: Lazy<RemoteTabsStorage>,
    lazyAutofillStorage: Lazy<AutofillCreditCardsAddressesStorage>,
    strictMode: StrictModeManager,
)
```

### Account Manager Configuration
```kotlin
val accountManager = FxaAccountManager(
    context,
    serverConfig = FxaServer.config(context),
    deviceConfig = DeviceConfig(
        name = deviceName,
        type = DeviceType.MOBILE,
        capabilities = listOf(Capability.SEND_TAB, Capability.CLOSE_TABS),
    ),
    syncConfig = SyncConfig(
        supportedEngines = listOf(
            SyncEngine.History, SyncEngine.Bookmarks,
            SyncEngine.Passwords, SyncEngine.Tabs,
            SyncEngine.CreditCards, SyncEngine.Addresses
        ),
        periodicSyncConfig = PeriodicSyncConfig(periodicSyncInterval = 240),
    ),
    syncScopes = listOf(SCOPE_SYNC),
)
```

### Supported Sync Engines
| Engine | Storage | Toggle |
|--------|---------|--------|
| History | `PlacesHistoryStorage` | Fixed |
| Bookmarks | `PlacesBookmarksStorage` | Fixed |
| Passwords | `SyncableLoginsStorage` | User |
| Tabs | `RemoteTabsStorage` | User |
| Credit Cards | `AutofillCreditCardsAddressesStorage` | User |
| Addresses | Same storage (addresses part) | User (`isAddressSyncEnabled`) |

## FxA Authentication Flow

### Initialization
```kotlin
accountManager.start()  // In MainScope
```

### Account Observer Callbacks
- `onAuthenticated(account, type)` → records Glean auth type, sets `signedInFxaAccount = true`
- `onLoggedOut()` → clears account state
- `onAuthError()` → handles auth errors

### Auth Types (Telemetry)
Signin, Signup, Pairing, Recovered, OtherExternal, Existing

## Push Notification Integration

```kotlin
FxaPushSupportFeature(autoPushFeature, accountManager)  // Enables push for FxA
SendTabFeature(accountManager, notificationManager)       // Show received tab notifications
CloseTabsFeature(receiver, notificationObserver)           // Handle remote close commands
```

### Push Flow
```
Firefox Sync Server
  → Push Service
    → AutoPush Feature
      → FxaPushSupportFeature
        → accountManager.processPushMessage()
          → Sync triggers
```

## Synced Tabs

### Storage
```kotlin
SyncedTabsStorage(
    accountManager,
    store,
    remoteTabsStorage,
)
```

### Autocomplete
```kotlin
SyncedTabsAutocompleteProvider(store, accountManager, remoteTabsStorage)
```
Provides URL autocomplete suggestions from synced tabs.

### Recent Synced Tab
`RecentSyncedTabFeature` shows the most recent tab synced from another device on the home screen.

### Send Tab
- `SendTabFeature`: Shows notification for tabs received from other devices
- `SyncedTabsCommands`: Manages command queues for send/close tab operations
- Flush scheduler with undo-delay-aware timing

## Sync Store

`SyncStore` tracks sync state across all engines. Telemetry records `ClientAssociation.uid` on account UID changes.

## Global Dependency Providers

```kotlin
GlobalSyncedTabsCommandsProvider.initialize(lazy { syncedTabsCommands })
GlobalLoginsDependencyProvider.initialize(lazy { passwordsStorage })
GlobalAutofillDependencyProvider.initialize(lazy { autofillStorage })
```

These make storage accessible to sync workers and other processes.

## Settings Integration

- `SyncEnginesStorage`: Checks which engines are enabled
- `SyncPreferenceView`: Settings UI for sync configuration
- `AccountSettingsFragment`: Account management UI
- `AutofillSettingFragment`: Controls credit card + address sync toggles

## Key Files

| File | Role |
|------|------|
| `fenix/.../components/BackgroundServices.kt` | Sync infrastructure setup |
| `fenix/.../sync/SyncedTabsIntegration.kt` | Synced tabs lifecycle |
| `fenix/.../sync/SyncedTabsAutocompleteProvider.kt` | URL autocomplete from synced tabs |
| `fenix/.../home/recentsyncedtabs/RecentSyncedTabFeature.kt` | Home screen recent tab |
| `fenix/.../settings/SyncPreferenceView.kt` | Sync preferences UI |
| `fenix/.../settings/account/AccountSettingsFragment.kt` | Account management |
| `fenix/.../push/PushFxaIntegration.kt` | Push → FxA bridge |
| `fenix/.../push/WebPushEngineIntegration.kt` | Web Push → Engine bridge |
| `A-C/.../service/fxa/manager/FxaAccountManager.kt` | Account lifecycle |
| `A-C/.../service/fxa/SendTabFeature.kt` | Received tab notifications |
| `A-C/.../feature/syncedtabs/SyncedTabsStorage.kt` | Remote tabs data |
| `A-C/.../feature/syncedtabs/commands/` | Tab command management |

(A-C = `android-components` root)

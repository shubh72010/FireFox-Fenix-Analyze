# Architecture

## Purpose
Fenix is a single-Activity, multi-Fragment Android browser app that uses a layered architecture with Redux-inspired state management, abstracted browser engine, and service-locator-style dependency management.

## High-Level Layers

### 1. UI Layer
- **Single Activity**: `HomeActivity` extends `LocaleAwareAppCompatActivity`, hosts a `NavHostFragment` for all navigation.
- **Fragments**: Traditional `Fragment`-based navigation with XML layout or Compose UI. `BaseBrowserFragment` is the core content display fragment.
- **Compose**: Newer screens (Home, Tab Tray, Settings sections) use full Compose via `ComposeView`.
- **Theming**: `FirefoxTheme` composable wraps `AcornTheme` with three palettes: light, dark, private. Theme switching triggers `activity.recreate()`.

### 2. State Management Layer
- **BrowserStore**: Global Redux store holding `BrowserState` (tabs, custom tabs, containers, desktop mode, etc.). Managed by android-components library.
- **AppStore**: Fenix-level Redux store holding `AppState` (browsing mode, top sites, collections, bookmarks, recent tabs, messaging, wallpapers, etc.).
- **Domain Stores**: `TabsTrayStore`, `DownloadUIStore`, `IPProtectionStore`, `DohSettingsStore` - scoped to specific features.

### 3. Feature Layer
- **UseCases**: Encapsulate business operations (e.g. `TabsUseCases`, `CustomTabsUseCases`, `SearchUseCases`).
- **Middleware**: Intercept Redux actions for side effects (telemetry, persistence, navigation).
- **Features**: `AbstractBinding` and `ViewBoundFeatureWrapper` pattern for tying feature logic to view lifecycle.
- **Controllers/Interactors**: Mediate between UI and business logic (e.g. `TabManagerController`, `SessionControlInteractor`).

### 4. Engine Abstraction Layer
- **Engine interface** (`concept/engine`): Defines `Engine`, `EngineSession`, `EngineView` abstractions.
- **GeckoEngine** (`browser/engine-gecko`): Implements Engine interface wrapping GeckoView.
- **DefaultSettings**: Engine-agnostic settings data class, mapped to GeckoView settings at initialization.

### 5. GeckoView Layer
- **GeckoRuntime**: Singleton process manager for the Gecko rendering engine.
- **GeckoSession**: Per-tab browsing session.
- **GeckoView**: Android `SurfaceView`-based rendering widget.
- **GeckoProvider**: Fenix's runtime management, creates and configures `GeckoRuntime`.

### 6. Storage/Sync Layer
- **Places**: SQLite-based history and bookmarks storage (via Mozilla Application Services).
- **SyncableLoginsStorage**: Encrypted login storage.
- **AutofillCreditCardsAddressesStorage**: Autofill data storage.
- **RemoteTabsStorage**: Synced tabs data.
- **SessionStorage**: JSON-based tab session persistence.

## Data Flow

```
User Action
    |
    v
UI (Fragment/Compose)
    |
    v
Controller/Interactor
    |
    v
UseCase
    |
    v
Store.dispatch(Action)
    |
    v
Middleware chain (telemetry, persistence, etc.)
    |
    v
Reducer -> new State
    |
    v
Store.stateFlow -> UI recomposition
```

## Navigation Architecture

```
HomeActivity
  └── NavHostFragment (R.id.container)
        └── nav_graph.xml
              ├── startupFragment (start destination)
              ├── onboardingFragment
              ├── homeFragment 
              ├── browserFragment / BaseBrowserFragment
              ├── externalAppBrowserFragment (custom tabs/PWAs)
              ├── settingsFragment (nested graphs for sub-settings)
              ├── tabManagementFragment
              ├── bookmarkFragment
              ├── historyFragment
              └── 50+ other fragment destinations
```

Navigation uses AndroidX Navigation with `NavGraphDirections` for typed safe args. Global actions allow navigation from any destination.

## Component Architecture (Service Locator)

`Components` in `components/Components.kt` is a service locator holding all application components:

```
Components
  ├── core (Core) - Engine, Store, Storage, Runtime
  ├── backgroundServices (BackgroundServices) - FxA, Sync
  ├── useCases (UseCases) - All use cases
  ├── settings (Settings) - User preferences
  ├── analytics (Analytics) - Crash reporter, metrics
  ├── nimbus (NimbusComponents) - Experiments
  ├── appStore (AppStore<AppState, AppAction>)
  ├── intentProcessors (IntentProcessors)
  ├── push (Push)
  ├── performance (PerformanceComponent)
  ├── fxSuggest (FxSuggest)
  ├── ipProtection (IPProtection)
  └── ... (30+ more components)
```

## Key Architectural Decisions

1. **Single Activity**: All screens are Fragments within one Activity. This simplifies lifecycle management, intent handling, and theme switching.

2. **Redux Stores**: Multiple Redux stores (not one global store) to avoid coupling unrelated state. BrowserStore and AppStore are the two primary stores.

3. **Engine Abstraction**: The `Engine` interface in `concept/engine` allows theoretical replacement of GeckoView. In practice, Fenix is tightly coupled to GeckoView through `GeckoRuntime`, `GeckoViewFetchClient`, and `GeckoSitePermissionsStorage`.

4. **Disabled Automatic Tab Suspension**: Fenix explicitly disables `TrimMemoryMiddleware` (passes `trimMemoryAutomatically = false`) to rely on GeckoView's internal memory management.

5. **Service Locator over DI**: Fenix uses explicit `Components` service locator rather than a DI framework. android-components internally uses Hilt.

6. **JSON Session Persistence**: Tab state is serialized to JSON via `SessionStorage` rather than using a database.

7. **Compose Adoption**: Newer screens are built in Compose, older screens remain Fragment-based. The home screen, tab tray, and settings search are Compose.

## Tight Coupling Points

These would be difficult to replace if switching to a different engine:
- `GeckoRuntime` usage in `Core.kt` and `GeckoProvider.kt`
- `GeckoViewFetchClient` for HTTP fetches
- `GeckoSitePermissionsStorage` for permission persistence
- Gecko-specific `DefaultSettings` properties (fission, fingerprinting protection, safe browsing modes)
- WebExtension support via GeckoView's `WebExtensionController`
- WebAuthn via GeckoView's FIDO2 bridge

## Reusable Patterns for Other Browser Projects

1. Redux store architecture for browser state
2. Middleware pattern for cross-cutting concerns (telemetry, persistence)
3. AbstractBinding for reactive state-to-view binding
4. UseCases layer for business logic encapsulation
5. `concept/engine` abstraction pattern
6. Session auto-save with periodic and background triggers
7. Intent processor chain for external intent handling
8. Lazy component initialization for fast startup

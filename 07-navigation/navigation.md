# Navigation

## Purpose
Fenix uses a single-Activity architecture with AndroidX Navigation for Fragment-based navigation. All screens are Fragments within `HomeActivity`.

## Navigation Infrastructure

### NavHostActivity
Interface implemented by `HomeActivity`. Provides the `NavHostFragment` and `navController`.

### NavGraph
File: `fenix/app/src/main/res/navigation/nav_graph.xml` (~1905 lines)

**Start destination**: `startupFragment`

**Key Destinations**:
| Fragment ID | Class | Purpose |
|-------------|-------|---------|
| startupFragment | StartupFragment | Splash/loading |
| onboardingFragment | OnboardingFragment | First-run flow |
| homeFragment | HomeFragment | Home screen |
| browserFragment | BrowserFragment | Web content |
| externalAppBrowserFragment | ExternalAppBrowserFragment | Custom tabs / PWAs |
| settingsFragment | SettingsFragment | Main settings |
| bookmarkFragment | BookmarkFragment | Bookmarks library |
| historyFragment | HistoryFragment | History library |
| downloadsFragment | DownloadFragment | Downloads list |
| tabManagementFragment | TabManagementFragment | Tab tray |
| addonsManagementFragment | AddonsManagementFragment | Add-ons manager |
| trackingProtectionFragment | TrackingProtectionFragment | ETP settings |

**Nested Graphs**:
| Graph ID | Content |
|----------|---------|
| addons_management_graph | Add-ons → details → permissions |
| search_engine_graph | Search engine settings |
| nimbus_experiment_graph | Nimbus experiments UI |
| autofill_graph | Credit cards, addresses |
| savedLogins | Saved logins (inside auth gate) |
| menu_graph | Main menu (bottom sheet) |
| trust_panel_graph | Site info / trust panel |
| translations_settings_graph | Translation settings |
| site_permissions_exceptions_graph | Site permissions exceptions |

**Global Actions**: 50+ `action_global_*` destinations for navigation from anywhere in the app.

### Navigation Directions
```kotlin
// Global directions (auto-generated from nav_graph.xml)
NavGraphDirections.actionGlobalHome()
NavGraphDirections.actionGlobalBrowser(url, private, ...)
NavGraphDirections.actionGlobalSettings()

// Fragment-specific directions
val directions = NavGraphDirections.actionGlobalBookmarkFragment()
navController.navigate(directions)
```

## Navigation Flow

### App Launch
```
HomeActivity.onCreate()
  ├── Set browsing mode + theme
  ├── Install splash screen
  ├── If onboarding needed → navigate to onboardingFragment
  ├── Navigate to homeFragment
  └── If cold start with URL intent → navigate to browserFragment
```

### Browser Navigation
```
User clicks link / opens URL
  → FenixBrowserUseCases.loadUrlOrSearch()
    → TabsUseCases.addTab(url)
      → BrowserStore state update
        → BaseBrowserFragment observes selectedTabId change
          → Renders new tab's EngineSession in EngineView
```

### Screen Transitions
- Standard: Fragment transactions via NavController
- Settings: Slide-in/slide-out animations (`R.anim.slide_in_right`, etc.)
- Dialogs: `BottomSheetDialogFragment` for menus, site info, etc.
- Compose screens: Full Compose inside Fragment via `ComposeView`

## Intent Processing

### External Intents
Handled by `IntentProcessors` chain (in `components/IntentProcessors.kt`):

```kotlin
val externalSourceIntentProcessors = listOf(
    HomeDeepLinkIntentProcessor,
    SpeechProcessingIntentProcessor,
    AssistIntentProcessor,
    StartSearchIntentProcessor,
    OpenBrowserIntentProcessor,
    OpenSpecificTabIntentProcessor,
    OpenPasswordManagerIntentProcessor,
    OpenRecentlyClosedIntentProcessor,
)
```

Each processor checks if it can handle the intent, and if so, processes it and returns true.

## Custom Tab Navigation

`ExternalAppBrowserActivity` (extends `HomeActivity`) handles:
- Custom tabs from other apps
- Progressive Web Apps (PWAs)
- Trusted Web Activities (TWAs)

Uses `ExternalAppBrowserFragment` (extends `BaseBrowserFragment`) with additional features:
- `CustomTabColorsBinding` - apply custom tab toolbar colors
- `CustomTabWindowFeature` - multi-window support
- `WebAppHideToolbarFeature` - auto-hide toolbar for PWAs
- `ManifestUpdateFeature` - PWA manifest updates

## Browser Direction

`BrowserDirection` enum guides navigation decisions:
- `HOME` → homeFragment
- `EXTERNAL_APP` → externalAppBrowserFragment

## Back Navigation

- **Tab history**: Uses GeckoView's `goBack()`/`goForward()` for page history
- **Tab tray back**: Managed by `TabsTrayState.backStack: List<TabManagerNavDestination>`
- **Settings back**: Standard AndroidX navigation back stack
- **Custom tab back**: Finishes `ExternalAppBrowserActivity` on back if no page history

## Key Files

| File | Role |
|------|------|
| `fenix/.../HomeActivity.kt` | Nav host activity, intent processing |
| `fenix/.../NavHostActivity.kt` | NavHost interface |
| `fenix/.../BrowserDirection.kt` | Browser navigation direction |
| `fenix/.../GlobalDirections.kt` | Global navigation actions |
| `fenix/.../navigation/` | Navigation helpers |
| `fenix/.../intent/IntentProcessors.kt` | Intent processor chain |
| `fenix/.../components/IntentProcessors.kt` | IntentProcessor creation |
| `fenix/.../components/IntentProcessorType.kt` | Processor type enum |
| `fenix/.../customtabs/ExternalAppBrowserActivity.kt` | Custom tab activity |
| `fenix/app/src/main/res/navigation/nav_graph.xml` | Navigation graph |

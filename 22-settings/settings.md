# Settings

## Purpose
Fenix's settings system uses Android `PreferenceFragmentCompat` for most screens, with custom preference types for browser-specific controls. Settings are persisted in `SharedPreferences` (wrapped by `Settings.kt`) and newer features use Jetpack `DataStore`.

## Settings Architecture

```
SettingsFragment (PreferenceFragmentCompat)
  ├── Sub-settings (PrefrenceFragmentCompat)
  │     ├── Search → nested search_engine_graph
  │     ├── Account → nested autofill_graph
  │     ├── Tracking Protection
  │     ├── Private Browsing
  │     ├── Customize → Home, Toolbar, Tabs
  │     ├── Accessibility
  │     ├── Delete Browsing Data
  │     └── ... (20+ sub-screens)
  ├── Secret Settings (debug builds)
  └── Settings Search (Compose)
        └── DefaultFenixSettingsIndexer
```

## Settings.kt Wrapper

File: `fenix/.../utils/Settings.kt`

A class wrapping `SharedPreferences` with typed property accessors:

```kotlin
class Settings(private val context: Context) {
    private val prefs = context.getSharedPreferences(FENIX_PREFERENCES, MODE_PRIVATE)

    // Privacy
    val shouldUseTrackingProtection: Boolean
    val shouldUseHttpsOnly: Boolean
    val shouldEnableGlobalPrivacyControl: Boolean
    val shouldUseCookieBanner: Boolean
    
    // Appearance
    val shouldUseLightTheme: Boolean
    val shouldUseDarkTheme: Boolean
    val toolbarPosition: ToolbarPosition
    
    // Tabs
    val getTabTimeout: TabTimeout
    val isTabStripEnabled: Boolean
    
    // Search
    val shouldShowSearchSuggestions: Boolean
    val showSponsoredSuggestions: Boolean
    
    // Sync
    var signedInFxaAccount: Boolean
    
    // Downloads
    val downloadDirectory: String
    val deleteDownloadBehavior: DeleteDownloadBehavior
    
    // Hundreds of other properties...
}
```

## Custom Preference Types

| Type | Class | Purpose |
|------|-------|---------|
| Switch | `FenixSwitchPreference` | Toggle with Fenix styling |
| Radio | `RadioButtonPreference` | Single selection |
| Switch+Caption | `SwitchWithCaptionPreference` | Toggle with description |
| Toolbar | `ToolbarShortcutPreference` | Toolbar shortcut selector |
| Compose | `ComposePreference` | Embed Compose in preference screen |
| Text Size | `ComposeTextSizePreference` | Font size slider |
| IPP | `IPProtectionPreference` | IP Protection toggle |
| Sync | `SyncPreferenceView` | Sync status and control |
| Cookie Banner | `CustomCBHSwitchPreference` | Cookie banner handling |

## Navigation Graph

Settings has its own navigation with slide animations:
```xml
<action android:id="@+id/action_settingsFragment_to_searchFragment"
    app:enterAnim="@anim/slide_in_right"
    app:exitAnim="@anim/slide_out_left"
    app:popEnterAnim="@anim/slide_in_left"
    app:popExitAnim="@anim/slide_out_right" />
```

## Settings Search

File: `fenix/.../settings/settingssearch/`

```kotlin
class DefaultFenixSettingsIndexer(context, providers) : SettingsIndexer
```

Additional providers:
- `DataChoicesSearchProvider`
- `AIControlsSearchProvider`
- `PageSummariesSettingsSearchProvider`
- `FirefoxLabsSettingsSearchProvider`

Allows searching across all settings screens from a single search bar.

## Secret Settings (Debug)

File: `fenix/.../debugsettings/`

Debug builds have a `SecretSettingsFragment` accessible from settings:
- Feature toggles
- Nimbus experiment overrides
- Remote Settings server selection
- AMO collection overrides
- Secret setting overrides from `local.properties`

## Major Settings Categories

| Category | Fragment | Key Settings |
|----------|----------|--------------|
| Search | `SearchEngineFragment` | Engine selection, suggestions, shortcut |
| Account | `AccountSettingsFragment` | Sync, sign in/out |
| Privacy | `TrackingProtectionFragment` | TP mode, cookie behavior |
| Security | `HttpsOnlyFragment`, `DohSettingsFragment` | HTTPS-only, DoH |
| Private Browsing | `PrivateBrowsingFragment` | Lock, quick access |
| Customize | `HomeSettingsFragment` | Top sites, pocket, wallpaper |
| Toolbar | `ToolbarSettingsFragment` | Position, tab strip |
| Tabs | `TabsSettingFragment` | Tab timeout, view mode |
| Accessibility | `AccessibilityFragment` | Font size, zoom, auto-size |
| Downloads | `DownloadSettingsFragment` | Location, delete behavior |
| Delete Data | `DeleteBrowsingDataFragment` | Clear all browsing data |
| IP Protection | Settings sub-screen (Nimbus-enabled) | VPN proxy, locations |
| Add-ons | `AddonsManagementFragment` | Extension management |
| About | `AboutFragment` | Version, licenses |

## DataStore Settings

Newer features use Jetpack DataStore:
| DataStore | Settings |
|-----------|----------|
| `pocket_stories_selected_categories` | Pocket content preferences |
| `AIFeatureBlockStorage` | AI control blocking |
| `SummarizationSettings` | AI page summary preferences |
| `TranslationsSettings` | Translation enable/disable |
| `HomepageAsANewTabPreference` | Homepage on new tab |

## Settings Applied to Engine

At startup, all security/privacy settings from `Settings` are applied to the engine in `Core.kt`'s `DefaultSettings`. Changes during runtime go through:
1. `settingsUseCases.updateTrackingProtection(policy)`
2. `sessionUseCases.reload()`

## Key Files

| File | Role |
|------|------|
| `fenix/.../utils/Settings.kt` | Main settings wrapper |
| `fenix/.../settings/SettingsFragment.kt` | Main settings screen |
| `fenix/.../settings/settingssearch/DefaultFenixSettingsIndexer.kt` | Settings search |
| `fenix/.../debugsettings/SecretSettingsFragment.kt` | Debug settings |
| `fenix/.../settings/sitepermissions/` | Site permission settings |
| `fenix/.../settings/doh/` | DoH settings store |
| `fenix/.../settings/account/` | Account settings |
| `fenix/.../settings/downloads/` | Download settings |
| `fenix/.../onboarding/FenixOnboarding.kt` | First-run settings |

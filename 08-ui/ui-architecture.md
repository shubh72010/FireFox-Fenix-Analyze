# UI Architecture

## Purpose
Fenix uses a hybrid UI approach: Fragments for screen structure, Compose for newer screen content, and traditional XML for settings. The browser chrome (toolbar, URL bar) uses the Acorn design system from android-components.

## UI Layer Structure

```
HomeActivity
  └── NavHostFragment
        └── current Fragment
              ├── HomeFragment (Compose-based)
              ├── BrowserFragment (XML + EngineView)
              ├── TabManagementFragment (Compose-based)
              ├── SettingsFragment (PreferenceFragmentCompat)
              └── ... (50+ other fragments)
```

## Compose Usage

### Compose Integration Pattern
```kotlin
class SomeFragment : Fragment() {
    override fun onCreateView(inflater, container, savedInstanceState): View {
        return ComposeView(requireContext()).apply {
            setViewCompositionStrategy(
                ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
            )
            setContent {
                FirefoxTheme {
                    // Composable content
                }
            }
        }
    }
}
```

### Shared Compose Components (`compose/` directory)
| File | Component |
|------|-----------|
| `Banner.kt` | Two-button messaging banner |
| `Menu.kt` | Dropdown menu |
| `Favicon.kt` | Favicon image loader |
| `Image.kt` | Network image loader |
| `BetaLabel.kt` | Beta badge |
| `ScrollIndicator.kt` | Horizontal scroll indicator |
| `SwipeToDismissBox2.kt` | Swipe-to-dismiss gesture |
| `DismissibleItemBackground.kt` | Swipe reveal background |
| `ThumbnailCard.kt` | Tab thumbnail card |
| `ClickableSubstringLink.kt` | Clickable text spans |
| `HorizontalFadingEdgeBox.kt` | Fading edge container |
| `compose/snackbar/` | Compose snackbar system |
| `compose/list/` | List items, expandable headers |
| `compose/settings/` | Settings section headers |

### Menus (Compose-based)
File: `fenix/.../components/menu/compose/`

- `MainMenu.kt` - Root menu composable
- `MenuDialogBottomSheet.kt` - Bottom sheet container
- `MenuScaffold.kt` - Menu layout
- `MenuItem.kt` - Individual menu items
- `MenuGroup.kt` - Menu item groups
- Sub-menus: Extensions, Library, IP Protection, Add-ons, Summarization, Settings

## Theming System

### Theme Enum
```kotlin
enum class Theme { Light, Dark, Private }
```

### ThemeProvider
`FenixApplication` implements `ThemeProvider`:
```kotlin
override val currentTheme: Theme
    get() = when (browsingModeManager.mode) {
        BrowsingMode.Normal -> themeManager.currentTheme
        BrowsingMode.Private -> Theme.Private
    }
```

### FirefoxTheme Composable
```kotlin
@Composable
fun FirefoxTheme(theme: Theme = Theme.fromResources(), content: @Composable () -> Unit) {
    val colorScheme = when (theme) {
        Theme.Light -> lightColorPalette + acornLightColorScheme()
        Theme.Dark -> darkColorPalette + acornDarkColorScheme()
        Theme.Private -> privateColorPalette + acornPrivateColorScheme()
    }
    AcornTheme(colors = colorScheme) { content() }
}
```

### ThemeManager
`DefaultThemeManager`:
- Stores current `BrowsingMode` (Normal/Private)
- On mode change calls `activity.recreate()` to apply new theme
- Manages status bar + navigation bar colors via `applyStatusBarTheme()`
- Applies `R.style.NormalTheme` or `R.style.PrivateTheme` to activity

## Browser Chrome (Toolbar)

### BaseBrowserFragment
File: `fenix/.../browser/BaseBrowserFragment.kt` (~2683 lines)

The main content display fragment. Renders:
- `EngineView` (GeckoView) for web content
- `BrowserToolbar` (from A-C `browser/toolbar`) for URL bar + navigation
- `TabStrip` (Compose) above toolbar
- Various overlays: find in page, reader view, fullscreen, prompts

### Features Registered in BaseBrowserFragment
| Feature | Description |
|---------|-------------|
| `SessionFeature` | Session lifecycle management |
| `FullScreenFeature` | Fullscreen video handling |
| `DownloadsFeature` | Download management |
| `PromptFeature` | Form/alert/confirm dialogs |
| `SitePermissionsFeature` | Permission request dialogs |
| `ReaderViewFeature` | Reader mode |
| `FindInPageFeature` | Find in page bar |
| `ContextMenuFeature` | Long-press context menus |
| `SwipeRefreshFeature` | Pull-to-refresh |
| `PictureInPictureFeature` | PiP support |
| `MediaSessionFeature` | Media playback controls |
| `WebAuthnFeature` | WebAuthn activity result |

### Toolbar Position
Configurable: TOP or BOTTOM (via `Settings.toolbarPosition`)

### Home Toolbar
File: `fenix/.../home/toolbar/`

Compose-based toolbar on home screen with:
- URL bar / search bar
- Navigation buttons (back, forward, refresh)
- Tab counter / tab strip
- Menu button

## Home Screen (Compose)

### HomeFragment
File: `fenix/.../home/HomeFragment.kt`

Uses `ComposeView` as root. Composes `FirefoxTheme` → `Scaffold` with toolbar + `Homepage`.

### Homepage Sections
```
Homepage
  ├── Banner Card (Privacy notice / Nimbus message)
  ├── Header (browsing mode toggle)
  ├── Top Sites (shortcuts grid/pager)
  ├── Privacy Report Card (tracker count + animation)
  ├── Setup Checklist (onboarding tasks)
  ├── Recent Tabs
  ├── Recent Synced Tab
  ├── Bookmarks (recent)
  ├── Recently Visited (history metadata)
  ├── Collections
  └── Pocket Stories (content recommendations)
```

### HomepageState
Sealed class:
- `Normal`: All sections above
- `Private`: Simplified private browsing description

## Settings

### SettingsFragment
Extends `PreferenceFragmentCompat` (traditional XML preferences) with:
- Custom preference types: `FenixSwitchPreference`, `RadioButtonPreference`, `SwitchWithCaptionPreference`, `ComposePreference`, `IPProtectionPreference`, `SyncPreference`
- 20+ sub-settings screens (Accessibility, Search, Account, Autofill, Tracking Protection, Private Browsing, etc.)
- `SettingsSearchFragment` for searching settings
- `SecretSettingsFragment` for debug/developer settings

## Custom Tabs UI

### ExternalAppBrowserActivity
Subclass of `HomeActivity` for:
- Chrome Custom Tabs
- Progressive Web Apps (standalone mode)
- Trusted Web Activities

Uses `ExternalAppBrowserFragment` with custom tab-specific features:
- `CustomTabColorsBinding` - apply toolbar colors from external app
- `WebAppHideToolbarFeature` - auto-hide toolbar scroll
- `WebAppSiteControlsFeature` - site controls badge
- `ManifestUpdateFeature` - PWA manifest updates

## Secure Fragments

`SecureFragment` base class sets `FLAG_SECURE` on the window (prevents screenshots/recording):
- `SavedLoginsFragment`
- `CreditCardsManagementFragment`
- `CreditCardEditorFragment`
- `AddressEditorFragment`

## Key Design Patterns

1. **Fragment + Compose hybrid**: Fragments own the lifecycle; Compose handles rendering via `ComposeView`
2. **ViewBoundFeatureWrapper**: Ties feature lifecycle to view lifecycle (start in `onViewCreated`, stop in `onDestroyView`)
3. **AbstractBinding**: Reactive state observation + UI updates for features
4. **Controller/Interactor pattern**: Separates UI callbacks from business logic (e.g., `TabManagerController` + `TabManagerInteractor`)
5. **Acorn Design System**: Shared design language across Mozilla Android products

## Key Files

| File | Role |
|------|------|
| `fenix/.../HomeActivity.kt` | Main activity, navigation host |
| `fenix/.../browser/BaseBrowserFragment.kt` | Core browser fragment |
| `fenix/.../browser/BrowserFragment.kt` | Full browsing fragment |
| `fenix/.../customtabs/ExternalAppBrowserFragment.kt` | Custom tab fragment |
| `fenix/.../home/HomeFragment.kt` | Home screen fragment |
| `fenix/.../home/ui/Homepage.kt` | Home screen composable |
| `fenix/.../theme/FirefoxTheme.kt` | Compose theme wrapper |
| `fenix/.../theme/ThemeManager.kt` | Theme + status bar management |
| `fenix/.../compose/` | Shared Compose components |
| `fenix/.../components/menu/compose/` | Menu composables |
| `fenix/.../SecureFragment.kt` | Secure window base class |

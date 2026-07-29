# Private Browsing

## Purpose
Private browsing prevents the browser from storing history, cookies, site data, or form information. Fenix implements private browsing as a first-class browsing mode with separate state, themed UI, locking, and strict data isolation.

## Private Mode Architecture

```
BrowsingModeManager (tracks Normal vs Private)
  → BrowsingMode.Private
    → Private theme applied (activity.recreate())
    → New tabs created with content.private = true
    → Engine sessions opened in private mode
    → History, cookies, site data NOT persisted
```

## Key Characteristics

| Characteristic | Normal Mode | Private Mode |
|---------------|-------------|--------------|
| Tab flag | `content.private = false` | `content.private = true` |
| Theme | Light/Dark | Private (purple/dark) |
| Session persistence | Saved to JSON | Never saved |
| History recording | Yes | No |
| Tab state restoration | Yes (app restart) | No |
| Lockable | No | Yes (biometric/PIN) |
| Multi-select in tray | Yes | No (enforced by telemetry) |

## BrowsingModeManager

File: `fenix/.../browser/browsingmode/BrowsingModeManager.kt`

```kotlin
enum class BrowsingMode { Normal, Private }

interface BrowsingModeManager {
    val mode: BrowsingMode
    val isPrivate: Boolean
    fun onModeChange(mode: BrowsingMode)
}
```

`DefaultBrowsingModeManager`:
- Persists mode in `Settings.lastKnownMode`
- Notifies `AppStore` via `AppAction.BrowsingModeManagerModeChanged`
- Mode switching triggers `activity.recreate()` to reload theme

## Private Theme

- `Theme.Private` in `FirefoxTheme` composable
- `privateColorPalette`: purple/dark color scheme
- `R.style.PrivateTheme` applied to activity
- Status bar and navigation bar themed appropriately

## Private Tab Lifecycle

### Creation
```kotlin
AddNewTabUseCase.invoke(url, private = true)
  → createTab(url, private = true, source, ...)
    → TabSessionState(content.private = true)
      → dispatch(AddTabAction(tab))
```

### Session
```kotlin
GeckoEngineSession(privateMode = true)
  → GeckoSessionSettings.Builder().usePrivateMode(true)
    → GeckoView creates session with private browsing enabled
      → No history recording, no cookie persistence
```

### Persistence
Private tabs are explicitly excluded from `SessionStorage.save()`:
```kotlin
// SessionStorage only persists normalTabs
state.normalTabs  // Does not include private tabs
```

### Closure
Separate action: `RemoveAllPrivateTabsAction`
When closing last private tab with pending private downloads → warning dialog shown.

## Private Browsing Lock

### Feature
`PrivateBrowsingLockFeature` (in `pbmlock/`) implements `DefaultLifecycleObserver`:

1. **`onStop()`**: If private tabs exist and activity is not changing config/disconnected from custom tab:
   - `dispatch(PrivateBrowsingLockAction.Update(isLocked = true))`

2. **Auto-unlock**: When all private tabs are closed, lock is automatically released

3. **Navigation intercept**: Blocks access to private tab content until unlocked

### Unlock Flow
```
UnlockPrivateTabsFragment
  → ActivityResultLauncher (device credentials)
    → BiometricUtils.bindBiometricsCredentialsPromptOrShowWarning()
      → Success: dispatch(Update(isLocked = false))
      → Failure: record telemetry, navigate away
```

### Navigation Origins
```kotlin
enum class NavigationOrigin { TABS_TRAY, HOME_PAGE, TAB, CUSTOM_TAB }
// Determines behavior on leaving without auth:
//   CUSTOM_TAB → finish activity
//   Others → navigate to normal mode
```

### UI
- `UnlockPrivateTabsScreen` (Compose): Logo, unlock button, leave button
- `UnlockPrivateTabsTrayScreen`: Simplified for tab tray context
- `BlackScreenOverlay`: Obscures content behind lock screen

## Private Download Warning

When closing the last private tab while private downloads are active:
- Warning dialog shown
- User can cancel closure or proceed

## Private Mode Home Screen

`HomepageState.Private` is a simplified home screen with:
- Private browsing description
- No top sites, recent tabs, bookmarks, or pocket stories
- No tracking protection dashboard

## Secure Window Flag

Private mode automatically applies `FLAG_SECURE` via the `PrivateBrowsingLockFeature` lifecycle observer:
```kotlin
// Prevents screenshots in recent apps
window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)
```

## Key Files

| File | Role |
|------|------|
| `fenix/.../browser/browsingmode/BrowsingModeManager.kt` | Mode tracking |
| `fenix/.../pbmlock/PrivateBrowsingLockFeature.kt` | Lock lifecycle |
| `fenix/.../pbmlock/UnlockPrivateTabsFragment.kt` | Unlock UI |
| `fenix/.../pbmlock/UnlockPrivateTabsScreen.kt` | Compose unlock screen |
| `fenix/.../pbmlock/BlackScreenOverlay.kt` | Content obscuring |
| `fenix/.../theme/Theme.kt` | Private theme palette |
| `fenix/.../components/appstate/privatebrowsinglock/` | Lock state + reducer |
| `fenix/.../components/appstate/AppState.kt` | privateBrowsingLock field |
| `A-C/.../browser/state/state/TabSessionState.kt` | `content.private` flag |
| `A-C/.../browser/state/selector/Selectors.kt` | `normalTabs`/`privateTabs` selectors |
| `A-C/.../browser/session-storage/SessionStorage.kt` | Private tabs excluded from save |
| `A-C/.../feature/tabs/TabsUseCases.kt` | RemoveAllPrivateTabsUseCase |

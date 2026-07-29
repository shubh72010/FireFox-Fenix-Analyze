# Privacy and Security

## Purpose
Fenix implements multiple layers of privacy and security features: tracking protection, private browsing, DNS-over-HTTPS, HTTPS-only mode, cookie banner handling, fingerprinting protection, IP Protection, and more. Most are controlled by Nimbus experiments.

## Tracking Protection

### Architecture
`TrackingProtectionPolicyFactory` builds the engine's tracking protection policy from user settings.

### Modes
| Mode | Policy | Description |
|------|--------|-------------|
| Standard | `TrackingProtectionPolicy.recommended()` + TCP | Blocks known trackers, Total Cookie Protection |
| Strict | `TrackingProtectionPolicy.strict()` + TCP | More aggressive blocking |
| Custom | `TrackingProtectionPolicy.select()` | User-configurable per category |
| None | `TrackingProtectionPolicy.none()` | No blocking |

### Custom Categories
- Cookies (5 levels: `ACCEPT_NONE` to `ACCEPT_FIRST_PARTY_AND_ISOLATE_OTHERS`)
- Fingerprinters
- Cryptominers
- Tracking content
- Redirect trackers
- Suspected fingerprinters

### Policy → GeckoView Mapping
`TrackingProtectionPolicy.toContentBlockingSetting()` converts the policy to `ContentBlocking.Settings.Builder`:
- ETP level + categories
- Anti-tracking policy
- Cookie behavior (normal + private)
- Cookie purging
- Safe browsing (standard + V5 + real-time)
- Strict social tracking
- Cookie banner handling (reject, detect-only, global rules, sub-frames)
- Query parameter stripping
- Bounce tracking protection
- Allow lists (baseline TP + convenience TP)

### Dashboard
- `ProtectionsDashboardFragment` (`BottomSheetDialogFragment`) shows global protection state
- `TrackersBlockedFeature` (AbstractBinding) observes `BrowserStore` for tracker events
- Data flows: GeckoView → ContentBlockingDelegate → EngineObserver → BrowserStore → AppStore (blocked trackers state)
- Weekly tracker data stored in `ProtectionsStorage`

### Key Files
| File | Role |
|------|------|
| `fenix/.../components/TrackingProtectionPolicyFactory.kt` | Policy builder |
| `fenix/.../trackingprotection/TrackersBlockedFeature.kt` | Tracker event observer |
| `fenix/.../trackingprotection/TrackerBuckets.kt` | Tracker categorization |
| `fenix/.../settings/TrackingProtectionFragment.kt` | Settings UI |
| `A-C/.../engine-gecko/ext/TrackingProtectionPolicy.kt` | Policy → ContentBlocking mapping |
| `A-C/.../feature/protection/dashboard/` | Protection dashboard UI |

## HTTPS-Only Mode

### Levels
| Setting | Behavior |
|---------|----------|
| DISABLED | No HTTPS upgrade |
| PRIVATE_ONLY | HTTPS upgrade in private tabs only |
| ENABLED | HTTPS upgrade in all tabs |

Applied via `engine.settings.httpsOnlyMode`. When a site doesn't support HTTPS, GeckoView shows an error page.

### Key Files
`fenix/.../settings/HttpsOnlyFragment.kt`, `fenix/.../utils/Settings.kt` (lines 1220-1235, 2407-2415)

## DNS-over-HTTPS (DoH)

### Architecture
Redux store: `DohSettingsStore` with `DohSettingsMiddleware` and `DohSettingsProvider`.

### Protection Levels
| Level | Description |
|-------|-------------|
| Off | Default DNS |
| Default | Cloudflare (built-in) |
| Increased | Increased protection |
| Max | Max protection |

### Key Files
| File | Role |
|------|------|
| `fenix/.../settings/doh/DohSettingsStore.kt` | DoH state store |
| `fenix/.../settings/doh/DohSettingsProvider.kt` | Read/write DoH settings to engine |
| `fenix/.../settings/doh/DohUrlValidator.kt` | Custom provider URL validation |
| `fenix/.../components/Core.kt` (line 184-187, 207) | Engine defaults |

## Cookie Banner Handling

### Settings
- `REJECT_ALL` when enabled
- `DISABLED` when off
- Detect-only mode
- Global rules + sub-frame support

### Control
Nimbus experiment `cookieBanners` provides a map of `CookieBannersSection` to integer values for each setting.

### Key Files
`fenix/.../utils/Settings.kt` (lines 1245-1266), `fenix/.../settings/cookiebannerhandling/CustomCBHSwitchPreference.kt`

## Fingerprinting Protection

### Two Layers
| Layer | Control |
|-------|---------|
| Standard FPP | `FxNimbus.features.fingerprintingProtection` |
| Baseline FPP | `FxNimbus.features.baselineFpp` |

Each has enabled flags for normal + private browsing, plus overrides for specific fingerprinting surfaces. Fdlibm math is a separate flag for JS fingerprinting resistance.

## IP Protection (Mozilla VPN)

### Architecture
```kotlin
IPProtection(
    engine = engine,
    browserStore = browserStore,
    syncStore = syncStore,
    authSources = IPProtectionAuthSources(fxaAccountManager, integrityClient),
    appStore = appStore,
    settings = settings,
)
```

### Components
- `IPProtectionStore` - Redux store with middleware (telemetry, snackbar, preferences)
- `IPProtectionFeature` - Core feature from A-C managing proxy connections
- `IPProtectionStorageSynchronizer` - Syncs eligibility state
- `FenixIPProtectionEligibilityStorage` - Combines region, Nimbus flags, secret override

### Auth Sources
- FxA account (primary auth)
- Google Play Integrity (device verification)

### Key Files
| File | Role |
|------|------|
| `fenix/.../components/ipprotection/IPProtection.kt` | Top-level component |
| `fenix/.../components/ipprotection/IPProtectionTelemetryMiddleware.kt` | Glean metrics |
| `fenix/.../settings/ipprotection/IPProtectionLocationsFragment.kt` | Location settings |
| `fenix/.../ipprotection/ui/IPProtectionBottomSheetFragment.kt` | Onboarding UI |

## Private Browsing Lock

`PrivateBrowsingLockFeature` locks private tabs when the app goes to background:
1. On `onStop()`: if private tabs exist, lock via `AppAction.PrivateBrowsingLockAction.Update(isLocked = true)`
2. On resume: show unlock screen requiring biometric/PIN
3. Auto-unlocks when all private tabs closed
4. Biometric fallback to system PIN/password

### Key Files
| File | Role |
|------|------|
| `fenix/.../pbmlock/PrivateBrowsingLockFeature.kt` | Lifecycle observer, lock logic |
| `fenix/.../pbmlock/UnlockPrivateTabsFragment.kt` | Auth UI |
| `fenix/.../pbmlock/UnlockPrivateTabsScreen.kt` | Compose unlock screen |
| `fenix/.../components/appstate/privatebrowsinglock/` | Lock state + reducer |

## Global Privacy Control (GPC)

When enabled, sends the `Sec-GPC` header to signal Do Not Sell preference.
```kotlin
engine.settings.globalPrivacyControlEnabled = settings.shouldEnableGlobalPrivacyControl
```

## Security Settings Applied to Engine

All applied in `Core.kt` `DefaultSettings` construction:

| Setting | Source | Effect |
|---------|--------|--------|
| `httpsOnlyMode` | User setting | HTTPS enforcement |
| `dohSettingsMode` | User setting | DoH mode |
| `globalPrivacyControlEnabled` | User setting | GPC header |
| `certificateTransparencyMode` | Nimbus | CT enforcement |
| `postQuantumKeyExchangeEnabled` | Nimbus | PQ crypto |
| `enterpriseRootsEnabled` | User setting | Third-party root certs |
| `safeBrowsingV5Enabled` | Nimbus | Safe Browsing V5 |
| `safeBrowsingRealTimeEnabled` | Nimbus | Real-time SB lookups |
| `crliteChannel` | Nimbus | CRLite channel |
| `bannedPorts` | Nimbus | Blocked network ports |
| `cookieBehavior*` | Nimbus + User | 3rd-party cookie blocking |
| `fingerprintingProtection*` | Nimbus | FPP overrides |
| `lnaBlockingEnabled` | User | Local Network Access |

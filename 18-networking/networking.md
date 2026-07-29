# Networking

## Purpose
Fenix uses a dual networking approach: GeckoView for web content loading and GeckoViewFetchClient for HTTP requests from the app layer. DNS, security, and proxy settings are configured through the engine.

## Architecture

```
Web Content (GeckoView)
  → GeckoView's internal networking stack (Necko)
    → HTTP/2, HTTP/3 (QUIC), TLS 1.3
    → DNS-over-HTTPS (configurable)
    → Proxy support (IP Protection)

App Layer (Fenix components)
  → GeckoViewFetchClient (concept-fetch Client)
    → GeckoWebExecutor
      → Same Necko stack
        → Icon loading, Merino, Pocket, MARS, Location Service
```

## HTTP Client

### GeckoViewFetchClient
```kotlin
class GeckoViewFetchClient(
    context: Context,
    runtime: GeckoRuntime,
    private val maxReadTimeOut: Pair<Long, TimeUnit> = Pair(5, TimeUnit.MINUTES),
) : Client()
```

- Wraps `GeckoWebExecutor`
- 5-minute read timeout
- Supports: anonymous fetch, private fetch, no-redirects, OHTTP
- Used by `Core.client` for all A-C component network requests

### Usage
| Component | What It Fetches |
|-----------|----------------|
| BrowserIcons | Website favicons |
| LocationService | Mozilla Location Service |
| MerinoManifestProvider | Merino suggest manifest |
| PocketStoriesService | Content recommendations |
| MacTopSitesProvider | Top sites tiles |
| AMOAddonsProvider | Add-on list |
| RemoteSettingsService | Remote configuration |

## DNS Configuration

### DoH (DNS-over-HTTPS)
Configured through `DefaultSettings`:
```kotlin
defaultSettings = DefaultSettings(
    dohSettingsMode = settings.getDohSettingsMode(),
    dohProviderUrl = settings.dohProviderUrl,
    dohDefaultProviderUrl = settings.dohDefaultProviderUrl,
    dohExceptionsList = settings.dohExceptionsList,
)
```

### DoH Autoselect
```kotlin
defaultSettings.dohAutoselectEnabled = FxNimbus.features.doh.value().autoselectEnabled
```

## Security Protocol Settings

Applied in `DefaultSettings`:
| Setting | GeckoView Property | Source |
|---------|-------------------|--------|
| HTTPS-Only | `allowInsecureConnections` | User setting |
| Certificate Transparency | `certificateTransparencyMode` | Nimbus |
| Post-Quantum Key Exchange | `postQuantumKeyExchangeEnabled` | Nimbus |
| CRLite | `crliteChannel` | Nimbus |
| Safe Browsing V5 | `safeBrowsingV5Enabled` | Nimbus |
| Safe Browsing Real-Time | `safeBrowsingRealTimeEnabled` | Nimbus |
| Enterprise Roots | `enterpriseRootsEnabled` | User setting |

## Proxy (IP Protection)

IP Protection routes traffic through Mozilla's VPN proxy:
```kotlin
IPProtectionFeature(engine, browserStore, syncStore, authSources)
```
Uses GeckoView's proxy configuration to route selected traffic through the VPN tunnel.

## Cookie Behavior

Three-level cookie blocking:
| Setting | Nimbus Feature | Values |
|---------|---------------|--------|
| Standard blocking | `thirdPartyCookieBlocking` | enabledNormal, enabledPrivate |
| Opt-in Partitioning | Same feature | Partitioning per-site |
| Cookie Banner Handling | `cookieBanners` | REJECT_ALL, DETECT_ONLY, DISABLED |

## Cache

Fenix does not configure a separate HTTP cache - GeckoView manages its own network cache internally. See [Storage Layer](../10-storage/storage-layer.md) for app-level caches (icons, thumbnails).

## Content Blocking Network Effects

Tracking protection directly affects networking:
- Content blocking categories block/allow network requests to tracking domains
- Total Cookie Protection isolates cookies per-site
- Query parameter stripping removes tracking parameters from URLs
- Bounce tracking protection prevents redirect tracking
- Referrer tracking protection controls the Referer header

## Banned Ports

```kotlin
defaultSettings.bannedPorts = FxNimbus.features.networkingBannedPorts.value().bannedPortList
```

## User Agent

Set by GeckoView. Not overridden by Fenix (uses default GeckoView UA string).

## Key Files

| File | Role |
|------|------|
| `A-C/.../engine-gecko/fetch/GeckoViewFetchClient.kt` | HTTP Client impl |
| `A-C/.../concept/fetch/Client.kt` | Client abstraction |
| `fenix/.../components/Core.kt` (line 293) | Client creation |
| `fenix/.../components/Core.kt` (line 184-207) | Network settings in DefaultSettings |
| `fenix/.../settings/doh/` | DoH settings store |
| `fenix/.../components/ipprotection/IPProtection.kt` | Proxy management |
| `geckoview/.../GeckoWebExecutor.java` | GeckoView HTTP executor |
| `A-C/.../concept/engine/Settings.kt` | Engine settings model |

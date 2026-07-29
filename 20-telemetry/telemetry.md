# Telemetry

## Purpose
Fenix uses Mozilla's Glean telemetry system for product metrics and uses Adjust for attribution. All telemetry is opt-in (requires user consent during onboarding).

## Glean Integration

### Initialization
```kotlin
fun initializeGlean(context, logger, isTelemetryEnabled, client) {
    Glean.initialize(context)
    // Metrics ping scheduled on app start
}
```

Called in `maybeInitializeGlean()`:
- Only called after user has completed onboarding (`userHasBeenOnboarded()`)
- If onboarding disabled (local builds): called immediately
- If no data-upload consent: deferred until consent obtained

### Metrics Pings
Glean sends metrics pings containing:
- Startup metrics (RAM, screen size, default browser status, etc.)
- Feature state (top sites count, bookmarks count, installed add-ons, etc.)
- Configuration (privacy settings, search engine, etc.)

### Telemetry Consent
Controlled by: `Settings.isTelemetryEnabled`, `Settings.isMarketingTelemetryEnabled`, `Settings.isDailyUsagePingEnabled`

## TelemetryMiddleware (BrowserStore)

File: `fenix/.../telemetry/TelemetryMiddleware.kt`

Records browser-level telemetry:

| Metric | Trigger |
|--------|---------|
| `Events.normalAndPrivateUriCount` | Page load completion |
| `openTabsCount` | Tab add/remove/restore |
| `openPrivateTabsCount` | Private tab add/remove |
| `EngineMetrics.tabKilled` | Engine session kill |
| `EngineMetrics.reloaded` | Engine session recreate (ContentProcessKill or AppSessionRestore) |
| `Urlbar.engagement` / `Urlbar.abandonment` | Awesome bar interaction |
| `Translations.*` | Translation events |
| `Addons.extensionsProcessUiRetry/Disable` | Extension process UI |

## Telemetry Systems Overview

| System | Purpose | Location |
|--------|---------|----------|
| Glean | Core product metrics | `components/GleanHelper.kt` |
| Adjust | Campaign attribution | `components/metrics/` |
| Glean SDK | Built-in metrics (build, os, etc.) | Automatic |
| Nimbus | Experiment enrollment | `nimbus/` |
| Profiler markers | Performance profiling | `perf/ProfileMarkerMiddleware` |

## Metrics Middleware (AppStore)

`MetricsMiddleware` records app-level telemetry dispatched as `AppAction`:
- Navigation events
- Feature interactions
- Setup checklist completion

## Telemetry Middleware by Feature

| Feature | Middleware | Metrics Prefix |
|---------|-----------|----------------|
| Downloads | `DownloadTelemetryMiddleware` | `Downloads.*` |
| IP Protection | `IPProtectionTelemetryMiddleware` | `Vpn.*` |
| IPP Prompt | `IPProtectionPromptTelemetryMiddleware` | Prompt events |
| Bookmarks | `BookmarksTelemetryMiddleware` | `Bookmarks.*` |
| Tab Tray | `TabsTrayTelemetryMiddleware` | `TabsTray.*` |
| Home Screen | `HomeTelemetryMiddleware` | `Home.*` |
| Setup Checklist | `SetupChecklistTelemetryMiddleware` | `Setup.*` |
| Review Prompt | `ReviewPromptMiddleware` | `ReviewPrompt.*` |
| Add-ons | Glean labeled metrics | `Addons.*` |
| Autofill | Glean labeled metrics | `Logins.*`, `CreditCards.*`, `Addresses.*` |
| Sync | `TelemetryAccountObserver` | `SyncAuth.*`, `ClientAssociation.*` |
| Search | `SearchMiddleware` | `Search.*` |
| Ads | `AdsTelemetryMiddleware` | Search ad click-through |

## Startup Metrics

File: `fenix/.../FenixApplication.kt` (line 914)

`setStartupMetrics()` runs on `Dispatchers.IO` and records:
- `Metrics.defaultBrowser` - is Firefox default?
- `Metrics.distributionId` - distribution channel
- `Metrics.ramMoreThanThreshold` - device RAM > 1GB
- `Metrics.deviceTotalRam` - total RAM
- `Metrics.isLargeDevice` - tablet vs phone
- `Metrics.adjustCampaign/AdGroup/Creative/Network` - attribution
- `Metrics.searchWidgetInstalled` - widget state
- `Metrics.hasOpenTabs` / `tabsOpenCount`
- `Metrics.hasTopSites` / `topSitesCount`
- `Addons.hasInstalledAddons` / `installedAddons`
- `Preferences.*` - 20+ preference states
- `SearchDefaultEngine.*` / `SearchDefaultEngineForPrivate.*` - engine info
- `AndroidAutofill.supported` / `enabled` - autofill state
- `UserAiSummarize.*` - AI summary feature state

## Storage Stats Metrics

```kotlin
StorageStatsMetrics.report(context)
```
Records the size of Places database, logins storage, and other storage files.

## Application Exit Info

On Android 11+ (API 30+):
```kotlin
ApplicationExitInfoMetrics.recordProcessExits(context)
```
Records why the app process exited previously, helping diagnose crashes and OOM kills.

## Crash Telemetry

File: `fenix/.../crashes/`

- `CrashReporter` from android-components
- Breadcrumbs recorded for debugging
- `CrashReportingAppMiddleware` in AppStore

## Key Files

| File | Role |
|------|------|
| `fenix/.../components/GleanHelper.kt` | Glean initialization |
| `fenix/.../telemetry/TelemetryMiddleware.kt` | Browser-level telemetry |
| `fenix/.../components/metrics/MetricsMiddleware.kt` | AppStore metrics |
| `fenix/.../components/metrics/MetricsServiceHelper.kt` | Metrics service helper |
| `fenix/.../components/metrics/AdjustMetricsService.kt` | Adjust attribution |
| `fenix/.../FenixApplication.kt` (line 914) | Startup metrics |
| `fenix/.../perf/StorageStatsMetrics.kt` | Storage size metrics |
| `fenix/.../perf/ApplicationExitInfoMetrics.kt` | Exit reason metrics |
| `fenix/.../perf/ProfileMarkerMiddleware.kt` | Profiler markers |
| `fenix/.../experiments/` | Nimbus experiment integration |

# Performance

## Purpose
Fenix has extensive performance instrumentation and optimization. The performance system focuses on startup measurement, visual completeness tracking, profiling markers, and deferred initialization.

## Startup Performance

### Timeline
```kotlin
StartupTimeline.onApplicationInit()  // Earliest possible timestamp (init block)
    → PerfStartup.applicationOnCreate.accumulateSamples()  // End of onCreate
    → visualCompletenessQueue: tasks deferred until first frame
```

### PerformanceComponent
```kotlin
class PerformanceComponent {
    val visualCompletenessQueue = RunWhenReadyQueue()
    // visualCompletenessQueue.isReady() set when first frame rendered
}
```

### Startup Phases
| Phase | Time | Work |
|-------|------|------|
| Application init | Process start | Timestamp |
| Content providers | Before onCreate | EmojiCompat, library init |
| Application.onCreate | 0-500ms | Engine, Nimbus, Glean, Storage init |
| Activity.onCreate | +100ms | Navigation, theme, toolbar |
| First frame | +200ms | Initial UI |
| Visual completeness | +500ms | UI first displayed |
| Post-visual | +1-5s | Storage warmup, maintenance, wallpapers |

### Visual Completeness Queue Tasks (Post-First-Frame)
```
1. Storage warmup (history, bookmarks, logins, autofill, top sites)
2. Storage stats metrics
3. Engine warmup
4. Increment launch count
5. Restore locale
6. Register storage maintenance workers
7. Warm up Play Integrity client
8. Nimbus fetch experiments
9. FxSuggest ingestion
10. Download wallpapers
11. Collect process exit info
```

## Lazy Initialization

All components in `Components.kt` use `lazyMonitored`:
```kotlin
val core by lazyMonitored { Core(context, ...) }
```

`lazyMonitored` wraps Kotlin `lazy()` with performance monitoring (logs initialization time). Components are only constructed when first accessed, not at app startup.

## Performance Markers

### ProfileMarkerMiddleware
```kotlin
ProfileMarkerMiddleware(
    markerName = "BrowserStore",
    profiler = engine.profiler,
)
```
Registered for both `AppStore` and `BrowserStore`. Records profiler markers for every store dispatch.

### ProfilerMarkerFactProcessor
```kotlin
ProfilerMarkerFactProcessor.create { engine.profiler }.register()
```
Listens for `Facts` (event system) and records them as profiler markers.

## Memory-Related Performance

See [Memory Management](../06-memory/memory-management.md) for:
- `onTrimMemory` handling
- Icon cache clearing
- Thumbnail capture skipping on low memory
- Session prioritization middleware

## Build Performance

- **Debug builds**: StrictMode enabled, LeakCanary activated
- **Development**: `local.properties` secret setting overrides for feature toggling

## Performance Metrics (Telemetry)

| Metric | Source | Purpose |
|--------|--------|---------|
| `PerfStartup.applicationOnCreate` | FenixApplication | Duration of onCreate |
| `Metrics.*` | setStartupMetrics() | Device RAM, screen size, state snapshots |
| `StorageStatsMetrics` | Post-visual | Storage size |
| `ApplicationExitInfoMetrics` | Post-visual | Previous process exit reasons |
| `EngineMetrics.tabKilled` | TelemetryMiddleware | Content process kills |
| `EngineMetrics.reloaded` | TelemetryMiddleware | Tab reload after kill |

## StrictModeManager

```kotlin
class StrictModeManager(
    isDebug: Boolean,
    components: Components,
    manufacturerChecker: BuildManufacturerChecker,
)
```

Enables `StrictMode` on debug builds to detect:
- Disk reads/writes on main thread
- Network access on main thread
- Resource leaks

With manufacturer-specific exclusions (some OEMs trigger false positives).

## Threading for Performance

| Dispatcher | Usage |
|-----------|-------|
| `Dispatchers.Main` | UI, store dispatches, engine calls |
| `Dispatchers.IO` | Storage warmup, metrics, file ops, networking |
| `Dispatchers.Default` | CPU-intensive operations |
| `GlobalScope` | Fire-and-forget background tasks (launch count, locale restore, suggest ingestion) |

## Key Files

| File | Role |
|------|------|
| `fenix/.../perf/StartupTimeline.kt` | Startup timestamps |
| `fenix/.../perf/PerformanceComponent.kt` | Visual completeness queue |
| `fenix/.../perf/AppStartReasonProvider.kt` | Process start reason |
| `fenix/.../perf/StartupActivityLog.kt` | Startup activity log |
| `fenix/.../perf/StartupStateProvider.kt` | Combined startup state |
| `fenix/.../perf/StorageStatsMetrics.kt` | Storage size metrics |
| `fenix/.../perf/ApplicationExitInfoMetrics.kt` | Exit reason metrics |
| `fenix/.../perf/MarkersActivityLifecycleCallbacks.kt` | Activity lifecycle markers |
| `fenix/.../perf/StrictModeManager.kt` | StrictMode configuration |
| `fenix/.../perf/lazyMonitored.kt` | Monitored lazy delegate |
| `fenix/.../components/FenixApplication.kt` | Startup orchestration |
| `fenix/.../components/StartupMiddleware.kt` | Post-restore actions |
| `fenix/.../components/AppVisualCompletenessMiddleware.kt` | Visual completeness signal |
| `fenix/.../components/BrowserVisualCompletenessMiddleware.kt` | Visual completeness signal |

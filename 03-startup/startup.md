# Startup

## Purpose
Startup is the most critical performance path in Fenix. The code is carefully structured to defer non-essential work until after the UI is visible, and to parallelize initialization across multiple threads.

## Startup Timeline

### Phase 1: Process Start (Before Application.onCreate)
```
Zygote -> ActivityThread -> ContentProviders -> Application constructor
                                                      |
                                           StartupTimeline.onApplicationInit()
                                                      | (timestamp recorded in init block)
```

### Phase 2: Application.onCreate
```
FenixApplication.onCreate()
  └── initializeFenixProcess()
        ├── Log.addSink(FenixLogSink)
        ├── CrashReporter.registerDeferredInitializer
        ├── runOnlyInMainProcess {
        │     ├── Load SharedPreferences (async)
        │     ├── Apply secret settings overrides (debug)
        │     ├── setupEarlyMain()
        │     └── setupPostMegazord()
        │   }
```

### Phase 3: setupEarlyMain()
```
setupEarlyMain()
  ├── setupCrashReporting()                # CrashReporter.install()
  ├── setupMegazordInitial()               # AppServicesInitializer.init()
  ├── initializeNimbus()                   # Nimbus SDK init, FxNimbus.initialize()
  ├── ProfilerMarkerFactProcessor.register()
  ├── initializeEmojiCompat()              # (async on Main)
  ├── components.core.engine               # Triggers engine lazy init
  │     └── GeckoEngine constructor
  │           └── GeckoProvider.getOrCreateRuntime()
  ├── maybeInitializeGlean()
  ├── components.core.store                # Triggers BrowserStore lazy init
  ├── setStartupMetrics()                  # (async on IO)
  ├── setupMegazordNetwork()               # RustHttpConfig (async, awaited)
  ├── setDayNightTheme()
  ├── initializeWebExtensionSupport()
  ├── GlobalPlacesDependencyProvider.initialize()
  ├── GlobalLoginsDependencyProvider.initialize()
  ├── GlobalAutofillDependencyProvider.initialize()
  ├── GlobalSyncedTabsCommandsProvider.initialize()
  ├── initializeRemoteSettingsSupport()
  ├── restoreBrowserState()               # (async on Main)
  ├── restoreDownloads()                  # Restore download state
  ├── restoreMessaging()                  # Restore messaging state
  └── await(megazordDeferred)            # Wait for app-services network init
```

### Phase 4: setupPostMegazord()
```
setupPostMegazord()
  ├── setupLeakCanary()
  ├── startMetricsIfEnabled()             # Adjust, marketing
  ├── setupPush()                         # AutoPush initialization
  ├── maybeSetupIPProtection()            # IPP feature + storage sync
  ├── GlobalFxSuggestDependencyProvider.initialize()
  ├── visibilityLifecycleCallback = VisibilityLifecycleCallback
  ├── registerActivityLifecycleCallbacks()
  ├── appStartReasonProvider.registerInAppOnCreate()
  ├── startupActivityLog.registerInAppOnCreate()
  ├── appLinkIntentLaunchTypeProvider.registerInAppOnCreate()
  ├── initVisualCompletenessQueueAndQueueTasks()
  ├── ProcessLifecycleOwner observers
  └── wallpaperUseCases.fetchCurrentWallpaperUseCase()
```

### Phase 5: Visual Completeness Queue

These tasks run after the first frame is rendered:

```
queueInitStorageAndServices
  ├── historyStorage.warmUp()
  ├── bookmarksStorage.warmUp()
  ├── passwordsStorage.warmUp()
  ├── autofillStorage.warmUp()
  ├── topSitesStorage.getTopSites()    # Pre-populate cache
  ├── historyMetadataService.cleanup()
  ├── FxSuggest ingestion start/stop
  ├── fileUploadsDirCleaner.cleanUploadsDirectory()
  ├── Delete Pocket DB if needed
  ├── Account manager kick-off
  └── RelayFeatureIntegration.start()

queueMetrics                → StorageStatsMetrics.report()
queueEngineWarmup           → engine.warmUp()
queueIncrementNumberOfAppLaunches → settings.numberOfAppLaunches++
queueRestoreLocale          → useCases.localeUseCases.restore()
queueStorageMaintenance     → register periodic storage maintenance workers
queueIntegrityClientWarmUp  → integrityClient.warmUp()
queueNimbusFetchInForeground → nimbus fetch experiments
queueSuggestIngest          → fxSuggest.storage.runStartupIngestion()
queueDownloadWallpapers     → download configured wallpapers
queueCollectProcessExitInfo → record ApplicationExitInfo
```

## Session Restoration

### restoreBrowserState()
```
FenixApplication.restoreBrowserState() (launched on Dispatchers.Main)
  ├── tabsUseCases.restore(sessionStorage, tabTimeout)
  │     ├── storage.restore() → reads JSON from disk
  │     ├── Filter tabs by tabTimeout
  │     └── dispatch(RestoreAction)
  └── sessionStorage.autoSave(store)
        ├── periodicallyInForeground(30s)
        ├── whenGoingToBackground()
        └── whenSessionsChange()
```

The `restore()` method:
1. Reads `RecoverableBrowserState` from `SessionStorage` (JSON file on disk)
2. Applies `tabTimeoutInMs` filter - tabs inactive longer than threshold are discarded
3. Dispatches `TabListAction.RestoreAction` to populate `BrowserState.tabs`
4. Dispatches `RestoreCompleteAction` to signal restore completion

### Session Auto-Save
`AutoSave` writes browser state to `SessionStorage`:
- Periodically every 30 seconds while in foreground
- When app goes to background
- When sessions change (tabs added/removed)

Only normal (non-private) tabs are persisted. Private tabs are never saved.

## Startup Performance Optimization

1. **Lazy initialization**: All components use `lazyMonitored` (a monitored lazy delegate) to defer construction until first access.
2. **Deferred post-Megazord work**: Heavy initialization is queued behind visual completeness.
3. **Parallel initialization**: Engine, Glean, Nimbus, and storage are initialized concurrently.
4. **Early timestamp**: `StartupTimeline.onApplicationInit()` records the earliest possible startup timestamp in the `init` block.
5. **Performance markers**: `ProfileMarkerMiddleware` and `ProfilerMarkerFactProcessor` enable profiling.
6. **Off-thread I/O**: SharedPreferences loading, storage warming, and metric recording are done on `Dispatchers.IO`.

## Key Startup Files

| File | Role |
|------|------|
| `FenixApplication.kt` | Main startup orchestrator |
| `Components.kt` | Service locator, lazy component initialization |
| `Core.kt` | Engine, store, storage creation |
| `StartupTimeline.kt` | Startup timestamp tracking |
| `PerformanceComponent.kt` | Visual completeness queue |
| `StartupMiddleware.kt` | Adds homepage tab after restore |
| `AppStartReasonProvider.kt` | Determines why the app started |
| `StartupActivityLog.kt` | Startup activity logging |
| `StartupStateProvider.kt` | Combined startup state |
| `StartupCrashActivity.kt` | Crash-on-startup recovery |
| `StartupFragment.kt` | Initial splash/loading screen |
| `AppVisualCompletenessMiddleware.kt` | Signals visual completeness to AppStore |
| `BrowserVisualCompletenessMiddleware.kt` | Signals visual completeness to BrowserStore |
| `lazyMonitored.kt` | Lazy delegate with performance monitoring |

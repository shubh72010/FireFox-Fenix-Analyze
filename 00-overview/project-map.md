# Firefox Fenix Project Map

## Purpose
A high-level map of the Firefox Fenix Android codebase structure, showing every major module, its location, purpose, and key dependencies.

## Repository Structure

### Top-Level Mobile Android Layout
```
mobile/android/
  fenix/                  # Firefox for Android (the browser app)
  android-components/     # Shared browser component library (A-C)
  geckoview/              # GeckoView engine library (GV)
  geckoview_example/      # Reference app for raw GeckoView usage
  focus-android/           # Firefox Focus product
  components/             # Additional shared components
  docs/                   # Project documentation
  gradle/                 # Gradle build infrastructure
  installer/              # Installer-related files
  locales/                # Localization files
  modules/                # Gradle module definitions
  branding/               # Branding resources per channel
  annotations/            # Custom annotations
  config/                 # Build configuration
  themes/                 # Theme resources
  test_infra/             # Test infrastructure
  test_runner/            # Test runner
```

### Fenix Module Structure (`mobile/android/fenix/`)
```
fenix/
  app/                    # Main application module
    src/main/
      java/org/mozilla/fenix/
        addons/               # Add-ons management
        android/              # Android system integration
        autofill/             # Android Autofill framework integration
        automotive/           # Android Auto support
        bindings/             # View/state bindings
        biometricauthentication/ # Biometric auth
        bookmarks/            # Bookmarks feature
        browser/              # Browser screen & features
        collections/          # Tab collections
        components/           # Service locator & core wiring
        compose/              # Shared Compose components
        crashes/              # Crash reporting
        customtabs/           # Custom tabs (Chrome Custom Tabs)
        datastore/            # Jetpack DataStore usage
        debugsettings/        # Debug settings
        distributions/        # Distribution handling
        downloads/            # Downloads feature
        e2e/                  # End-to-end tests
        exceptions/           # Site exceptions
        experiments/          # Nimbus experiment integration
        ext/                  # Extension functions
        extension/            # WebExtension support in Fenix
        gecko/                # GeckoView provider
        historymetadata/      # History metadata
        home/                 # Home screen
        iconpicker/           # Icon picker UI
        intent/               # Intent processing
        ipprotection/         # IP Protection feature
        library/              # Library (history, bookmarks, downloads)
        lifecycle/            # Lifecycle observers
        media/                # Media playback
        messaging/            # In-app messaging
        microsurvey/          # Micro surveys
        navigation/           # Navigation helpers
        nimbus/               # Nimbus SDK integration
        onboarding/           # First-run onboarding
        pbmlock/              # Private browsing lock
        perf/                 # Performance monitoring
        push/                 # Push notifications
        reviewprompt/         # Play Store review prompts
        search/               # Search functionality
        selection/            # Text selection features
        session/              # Session lifecycle
        settings/             # Settings screens
        share/                # Share functionality
        shortcut/             # Home screen shortcuts
        snackbar/             # Snackbar system
        splashscreen/         # Splash screen
        startupCrash/          # Startup crash handling
        summarization/        # Page summarization (AI)
        sync/                 # Firefox Sync integration
        tabgroups/            # Tab groups feature
        tabhistory/           # Tab history
        tabstray/             # Tab tray (tab switcher)
        telemetry/            # Telemetry middleware
        termsofuse/           # Terms of Use
        theme/                # Theming system
        trackingprotection/   # Tracking protection
        translations/         # Page translations
        utils/                # Utilities
        wallpapers/           # Wallpaper system
        webcompat/            # Web compatibility
        whatsnew/             # What's New
        wifi/                 # WiFi integration
        widget/              # Home screen widgets
  benchmark/              # Benchmarking module
  build.gradle            # Project-level build config
  settings.gradle         # Module includes
  gradle.properties       # Gradle properties
  plugins/                # Custom Gradle plugins
  tools/                  # Developer tools
  config/                 # Build configuration
  docs/                   # Fenix-specific docs
  certificates/           # Certificate files
```

## Architecture Layers

```
┌────────────────────────────────────────────────┐
│                   UI Layer                      │
│  Fragments, Compose Screens, Toolbar, Menus     │
├────────────────────────────────────────────────┤
│              State Management                   │
│  AppStore (Redux), BrowserStore (Redux),        │
│  TabsTrayStore, DownloadUIStore, etc.          │
├────────────────────────────────────────────────┤
│              Feature Layer                      │
│  UseCases, Features, Controllers, Middleware   │
├────────────────────────────────────────────────┤
│           Browser Engine (Abstraction)          │
│  Engine interface, EngineSession, EngineView    │
├────────────────────────────────────────────────┤
│         GeckoView Implementation                │
│  GeckoEngine, GeckoEngineSession, GeckoView     │
├────────────────────────────────────────────────┤
│          Storage / Sync Layer                   │
│  Places (History, Bookmarks), Logins, Sync,    │
│  Autofill, Remote Tabs                         │
└────────────────────────────────────────────────┘
```

## Key Design Patterns

1. **Service Locator**: `Components` class in `components/Components.kt` provides access to all application components.

2. **Redux Architecture**: Used for `BrowserStore`, `AppStore`, `TabsTrayStore`, `DownloadUIStore`, `IPProtectionStore`, and others. Each has a `Store<S, A>` with middleware and reducers.

3. **AbstractBinding**: Many features use `AbstractBinding<T>` to observe state flows and react to changes.

4. **Feature Wrappers**: `ViewBoundFeatureWrapper` ties feature lifecycle to view lifecycle.

5. **Middleware**: Cross-cutting concerns (telemetry, persistence, navigation) are handled by middleware in the Redux pipeline.

## Key Dependencies

| Library | Purpose |
|---------|---------|
| GeckoView | Web engine |
| android-components | Browser component library |
| Mozilla App Services | Sync, storage (Places, Logins) |
| Nimbus | Experimentation framework |
| Glean | Telemetry |
| Acorn Design System | UI component library |
| Jetpack Compose | Modern UI toolkit |
| AndroidX Navigation | Fragment-based navigation |
| Room | Local database (tab groups) |
| WorkManager | Background work |
| Hilt (via A-C) | Dependency injection (internal) |

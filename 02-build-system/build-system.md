# Build System

## Purpose
Fenix uses Gradle with Kotlin DSL for the Android app. The build system manages multiple modules, build variants (Debug/Nightly/Beta/Release), GeckoView engine selection, and integration with Mozilla's Application Services (megazord).

## Project Layout

```
settings.gradle.kts  → Includes: app, benchmark, mozilla-detekt-rules, plugins
build.gradle.kts     → Root project config
gradle.properties    → Android SDK paths, Kotlin options, memory settings
```

## Fenix Module Build

File: `fenix/app/build.gradle.kts`

### Key Plugins
- `com.android.application`
- `kotlin-android`
- `kotlin-kapt` (annotation processing)
- `com.google.devtools.ksp` (KSP for Room)
- `androidx.navigation.safeargs.kotlin`
- `org.mozilla.telemetry.glean-gradle-plugin`
- `org.mozilla.appservices-glean` (auto-stripping on CI)
- `org.mozilla.apilint` (GeckoView API compatibility)

### Dependencies
```kotlin
dependencies {
    // GeckoView (variant-specific)
    implementation(BrowserEngineGecko)
    // or:
    implementation(BrowserEngineGeckoBeta)
    // or:
    implementation(BrowserEngineGeckoNightly)
    
    // Android-Components (via local publication or Maven)
    implementation(project(":browser-state"))
    implementation(project(":feature-tabs"))
    // ... 50+ A-C components
    
    // Application Services (megazord)
    implementation(Megazord)
    
    // AndroidX
    implementation(AndroidX.compose.*)
    implementation(AndroidX.navigation.*)
    implementation(AndroidX.room.*)
    
    // Third-party
    implementation(Kotlin.coroutines.*)
    implementation(Kotlin.serialization)
    implementation(Coil)
    implementation(Acorn)  // Design system
}
```

## Build Variants

| Variant | Config.channel | Description |
|---------|---------------|-------------|
| `debug` | Debug | Development, LeakCanary, StrictMode |
| `nightly` | Nightly | Daily builds, unstable GeckoView |
| `beta` | Beta | Release candidate |
| `release` | Release | Production |

Each variant can use different:
- GeckoView engine version
- Nimbus server (experiment endpoints)
- AMO collection (add-on list)
- Remote Settings server
- Adjust token (attribution)

## GeckoEngine Version Selection

```kotlin
object BrowserEngineGecko : Dependency("org.mozilla.components", "browser-engine-gecko", "latest")
object BrowserEngineGeckoBeta : Dependency("org.mozilla.components", "browser-engine-gecko-beta", "latest")
object BrowserEngineGeckoNightly : Dependency("org.mozilla.components", "browser-engine-gecko-nightly", "latest")
```

## Build Configuration Files

| File | Purpose |
|------|---------|
| `fenix/build.gradle.kts` | Fenix project build config |
| `fenix/app/build.gradle.kts` | Main app module |
| `fenix/settings.gradle.kts` | Module includes |
| `fenix/gradle.properties` | Gradle properties |
| `fenix/config/*.yml` | Build configuration per variant |
| `fenix/gradle/wrapper/gradle-wrapper.properties` | Gradle version |

## Custom Gradle Plugins

Located in `fenix/plugins/`:
- Custom plugin for build optimization
- Detekt rules configuration

## Detekt (Static Analysis)

`fenix/mozilla-detekt-rules/` contains custom detekt rules for Fenix-specific code quality checks.

## Benchmark Module

`fenix/benchmark/` contains performance benchmark tests using Android Benchmark library.

## Release Structure

### APK/AAB Build
```
fenix/app/build/outputs/
  apk/
    debug/      → fenix-debug.apk
    nightly/    → fenix-nightly.apk
    beta/       → fenix-beta.apk
    release/    → fenix-release.aab (Google Play)
```

### ProGuard / R8
ProGuard rules are configured for release builds. The megazord library requires specific keep rules.

## Build Optimization

- Gradle configuration cache enabled
- Kotlin compile avoidance
- Parallel builds
- Lazy component initialization avoids early class loading

## Development Tools

- `./mach build` from Firefox root builds the Android app
- `./gradlew tasks` shows available Gradle tasks
- `./gradlew assembleDebug` builds debug APK
- `./gradlew lint` runs lint checks
- `./gradlew detekt` runs detekt analysis

## Continuous Integration

- CircleCI configuration in `fenix/.circleci/config.yml`
- Jenkins configuration for Mozilla CI
- Taskcluster integration for full Firefox build pipeline

## Key Files

| File | Role |
|------|------|
| `fenix/build.gradle.kts` | Root build config |
| `fenix/app/build.gradle.kts` | App module build |
| `fenix/settings.gradle.kts` | Module includes |
| `fenix/gradle.properties` | Android SDK, Kotlin, memory settings |
| `fenix/config/*.yml` | Channel-specific config |
| `fenix/plugins/` | Custom Gradle plugins |
| `fenix/mozilla-detekt-rules/` | Custom detekt rules |
| `mobile/android/geckoview/build.gradle` | GeckoView library build |

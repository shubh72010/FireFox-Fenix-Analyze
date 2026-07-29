# Testing

## Purpose
Fenix uses a multi-layered testing strategy: unit tests, UI tests, end-to-end tests, and integration tests. The testing infrastructure is split between Fenix-specific tests and android-components tests.

## Test Structure

```
fenix/
  app/src/
    test/          → Unit tests (JVM, Robolectric)
    androidTest/   → Instrumented tests (on-device)
  benchmark/       → Performance benchmarks
  e2e/             → End-to-end tests (app/src/main/java/org/mozilla/fenix/e2e/)
```

## Unit Tests

### Framework
- **JUnit 4/5**: Test framework
- **Mockito/MockK**: Mocking
- **Robolectric**: Android framework mocking for JVM tests
- **Turbine**: Kotlin Flow testing
- **Coroutines Test**: `runTest`, `TestScope`, `TestDispatcher`

### Test Patterns
```kotlin
@RunWith(AndroidJUnit4::class)
@Config(sdk = [Build.VERSION_CODES.R])
class SomeUseCaseTest {
    @get:Rule val coroutineRule = MainCoroutineRule()
    
    private val store = BrowserStore()
    private val useCase = SomeUseCase(store)
    
    @Test
    fun `GIVEN some state WHEN action THEN expected result`() {
        // Given
        store.dispatch(SomeAction(setupState))
        
        // When
        useCase.invoke()
        
        // Then
        assertEquals(expected, store.state.someField)
    }
}
```

### What Is Tested
- Use cases (`TabsUseCases`, `SearchUseCases`, etc.)
- Reducers (`AppStoreReducer`, `BrowserStateReducer`, sub-reducers)
- Middleware (telemetry, persistence, navigation)
- State transformations (e.g., `BrowserState.toTabStripState()`)
- Data layer (repositories, DAOs)

## Instrumented Tests

### Framework
- **AndroidX Test**: `ActivityScenario`, `FragmentScenario`
- **Espresso**: UI interaction
- **Compose UI Test**: `ComposeTestRule`
- **MockWebServer**: Network mocking

### Test Patterns
```kotlin
@RunWith(AndroidJUnit4::class)
class SettingsFragmentTest {
    @get:Rule val activityRule = ActivityScenarioRule(HomeActivity::class.java)
    
    @Test
    fun navigate_to_settings() {
        // Navigate to settings
        onView(withId(R.id.settings_button)).perform(click())
        // Verify
        onView(withText("Settings")).check(matches(isDisplayed()))
    }
}
```

## End-to-End Tests

Located in `fenix/app/src/main/java/org/mozilla/fenix/e2e/`:
- Full app flow tests
- Startup → navigation → browsing → settings → shutdown
- Use `ActivityScenario` for lifecycle testing

## android-components Tests

Each A-C component has its own test suite:
```
components/browser/state/src/test/
components/feature/tabs/src/test/
components/browser/engine-gecko/src/test/
// ... per-component tests
```

## Benchmark Tests

`fenix/benchmark/` contains Android Benchmark tests for performance measurement:
- Startup time
- Tab creation time
- Page load time
- Database operations

## Test Configuration

### Robolectric Config
```kotlin
@Config(
    sdk = [Build.VERSION_CODES.R],
    application = TestFenixApplication::class,
)
```

### Test Application
```kotlin
open class TestFenixApplication : FenixApplication() {
    override fun initializeFenixProcess() {
        // Skip all native initialization
        // No Nimbus, Glean, Gecko, or Megazord
    }
    
    override val components by lazy { TestComponents(this) }
}
```

## CI Test Execution

Tests run on:
- **Pull Requests**: Unit tests + lint
- **Nightly**: Full test suite + instrumented tests
- **Release**: All tests + benchmark comparisons

Test commands:
```bash
./gradlew test                              # Unit tests
./gradlew connectedAndroidTest              # Instrumented tests
./gradlew benchmark                          # Performance benchmarks
./mach test --auto                          # Mozilla CI test suite
```

## Code Coverage

- JaCoCo for code coverage reporting
- Coverage aggregated per module
- Minimum coverage thresholds enforced on CI

## Linting

| Tool | Purpose | Configuration |
|------|---------|---------------|
| detekt | Kotlin static analysis | `fenix/mozilla-detekt-rules/` |
| Android Lint | Android-specific checks | `fenix/app/lint.xml` |
| ktlint | Kotlin formatting | `.editorconfig` |

## Test Fixtures

Common test utilities:
- `MainCoroutineRule` - Coroutine test rule
- `MockBrowserStore` - Pre-configured mock store
- `FenixTestApplication` - No-init test application
- `RobolectricHelpers` - Common Robolectric setup

## Key Files

| File | Role |
|------|------|
| `fenix/app/src/test/` | Unit tests |
| `fenix/app/src/androidTest/` | Instrumented tests |
| `fenix/e2e/` | End-to-end tests |
| `fenix/benchmark/` | Performance benchmarks |
| `fenix/mozilla-detekt-rules/` | Custom detekt rules |
| `A-C/*/src/test/` | Component unit tests |
| `A-C/*/src/androidTest/` | Component instrumented tests |

# Passkeys / WebAuthn

## Purpose
Firefox for Android implements the W3C WebAuthn specification (FIDO2) across four layers: DOM API (C++), IPC, GeckoView Java bridge, and Android platform APIs. It supports both discoverable (resident) and non-discoverable credentials via two platform backends: Android 14+ Credential Manager and Google Play Services FIDO2.

## Architecture Layers

```
JS: navigator.credentials.create()/get()
    ↕ (DOM API)
C++: WebAuthnHandler (content process)
    ↕ (IPDL IPC)
C++: WebAuthnTransactionParent (parent process)
    ↕ (XPCOM nsIWebAuthnService)
C++: AndroidWebAuthnService (JNI bridge)
    ↕ (JNI @WrapForJNI)
Java: WebAuthnTokenManager (GV module)
    ↕ (Dispatch Logic)
    ├──→ WebAuthnCredentialManager (Android 14+ Credential Manager API)
    └──→ GMS FIDO2 PrivilegedApiClient (Google Play Services)
        ↕ (Activity result delegation)
Java: WebAuthnFeature (A-C → Fenix Activity)
```

## Complete Request/Response Lifecycle

### navigator.credentials.create() (Registration)

#### Step 1: Content Process (JS → C++)
- Website calls `navigator.credentials.create({ publicKey: { ... } })`
- `WebAuthnHandler::MakeCredential()` in `dom/webauthn/WebAuthnHandler.cpp`
- Validates params, creates `WebAuthnTransactionChild` IPC actor
- Sends `RequestRegister` IPDL message to parent process

#### Step 2: Parent Process (IPC → nsIWebAuthnService)
- `WebAuthnTransactionParent::RecvRequestRegister()`
- Validates `IsWebAuthnAllowedInContext()` and `IsValidRpId()`
- Fetches `https://{rpId}/.well-known/webauthn` via `WebAuthnRelatedOriginFetcher.sys.mjs`
- If cross-origin RP ID: shows `WebAuthnRelatedOriginPrompt` to user
- If confirmed: calls `mWebAuthnService->MakeCredential()`

#### Step 3: Service Routing
- `WebAuthnService::MakeCredential()` → `DefaultService()->MakeCredential()`
- On Android: `DefaultService()` = `AndroidWebAuthnService`

#### Step 4: JNI Bridge (C++ → Java)
- `AndroidWebAuthnService::MakeCredential()` packs params into `GeckoBundle`s
- Calls `WebAuthnTokenManager.webAuthnMakeCredential()` via `@WrapForJNI`

#### Step 5: Token Manager Dispatching
```java
WebAuthnTokenManager.webAuthnMakeCredential(bundles, byteBuffers, callback)
  ├── WebAuthnCredentialManager.makeCredential(request)  // Android 14+ primary
  │     ├── CreateCredentialRequest(UPDATE_DOMAIN_CAPABILITY)
  │     ├── Bundle: request JSON, clientDataHash, origin
  │     └── CredentialManager.createCredential(req, null, executor, receiver)
  │           ├── onResult: parse response JSON → MakeCredentialResponse
  │           └── onError(NOT_SUPPORTED_ERR): → GMS fallback
  │
  └── GMS FIDO2 fallback:
        ├── getRequestOptionsForMakeCredential() → PublicKeyCredentialCreationOptions
        ├── BrowserPublicKeyCredentialCreationOptions with origin
        └── Fido2PrivilegedApiClient.getRegisterPendingIntent()
              → PendingIntent
              → GeckoRuntime.startActivityForResult(intent)
```

#### Step 6: Activity Delegate (System UI)
```
GeckoRuntime.startActivityForResult(intent)
  → GeckoActivityDelegate (A-C bridge)
      → WebAuthnFeature (Fenix) 
          → activity.startIntentSenderForResult(intent, requestCode)
              → OS shows system UI (biometric/passkey selection)
                  → onActivityResult → callbackRef → GeckoResult
```

#### Step 7: Response Propagation
```
GeckoResult → MozPromise → AndroidWebAuthnService
  → WebAuthnService → WebAuthnTransactionParent
      → ContinueWithRegister() (after optional attestation consent)
          → IPDL → WebAuthnHandler
              → PublicKeyCredential DOM object → JS Promise resolves
```

### navigator.credentials.get() (Authentication)

Same flow with key differences at Step 5:

```java
WebAuthnTokenManager.webAuthnGetAssertion(bundles, byteBuffers, callback)
  │
  ├── hasCredentialInGMS(rpId, allowList)?
  │     ├── true (non-discoverable match found in GMS):
  │     │     → GMS FIDO2 getAssertion()
  │     │         → Fido2PrivilegedApiClient.getSignPendingIntent()
  │     │         → PendingIntent → system UI
  │     │
  │     └── false (check Credential Manager):
  │           ├── prepareGetCredential() → CredentialManager.prepareGetCredential()
  │           │     → handle != null: getCredential(handle) → system UI
  │           │         → OnResult: parse assertion response JSON
  │           │     → handle == null: → GMS FIDO2 fallback
  │           │
  │           └── GMS path: same as above
```

## Decision Tree for GetAssertion

```
webAuthnGetAssertion()
  |
  +-- hasCredentialInGMS(rpId, allowList)?
  |     |
  |     +-- true --> GMS FIDO2 getAssertion()
  |     |
  |     +-- false --> prepareGetCredential()
  |                    |
  |                    +-- handle != null --> Credential Manager getCredential()
  |                    |
  |                    +-- handle == null --> GMS FIDO2 fallback
```

`hasCredentialInGMS()` logic:
- Android < 14: Always returns true (always use GMS)
- Non-official builds: Returns false (always use Credential Manager first)
- Android 14+ official: Queries `fidoClient.getCredentialList(rpId)`, filters out discoverable credentials (handled by CM), returns true if any non-discoverable match exists

## Credential Manager Integration

File: `geckoview/.../WebAuthnCredentialManager.java`

Uses Android `android.credentials.CredentialManager` directly (not Jetpack wrapper):

### MakeCredential
```java
CreateCredentialRequest request = new CreateCredentialRequest(
    TYPE_PUBLIC_KEY_CREDENTIAL,    // "androidx.credentials.TYPE_PUBLIC_KEY_CREDENTIAL"
    requestBundle,                 // JSON + metadata
    new CreateCredentialRequest.UiCapabilities.Builder()
        .setAlwaysSendAppInfoToProvider(true)
        .build()
);
manager.createCredential(request, null, executor, outcomeReceiver);
```

### GetAssertion (Two-Phase)
```java
// Phase 1: Check availability
PrepareGetCredentialRequest prepareRequest = new PrepareGetCredentialRequest(
    TYPE_PUBLIC_KEY_CREDENTIAL,
    requestBundle
);
PendingGetCredentialHandle handle = manager.prepareGetCredential(prepareRequest, null, executor)
    .get();  // blocks, returns null if no credentials available

// Phase 2: Get credential (shows UI)
GetCredentialRequest getRequest = new GetCredentialRequest(handle);
manager.getCredential(getRequest, null, executor, outcomeReceiver);
```

### Error Mapping
| Platform Error | DOM Error | Description |
|---------------|-----------|-------------|
| TYPE_USER_CANCELED | ABORT_ERR | User cancelled |
| TYPE_NO_CREATE_OPTIONS | NOT_SUPPORTED_ERR | No options → try GMS |
| TYPE_NO_CREDENTIAL | NOT_SUPPORTED_ERR | No credential → try GMS |
| SecurityException | NOT_SUPPORTED_ERR | Permission denied → try GMS |

## Resident (Discoverable) Keys

Mapping from WebAuthn `residentKey` to platform equivalents:

| residentKey | GMS requireResidentKey | GMS residentKeyRequirement |
|-------------|----------------------|---------------------------|
| "required" | true | RESIDENT_KEY_REQUIRED |
| "preferred" | false | RESIDENT_KEY_PREFERRED |
| "discouraged" | false | RESIDENT_KEY_DISCOURAGED |

Gated by `security.webauthn.webauthn_enable_android_fido2_residentkey` pref.

- **Discoverable credentials**: Stored on authenticator, discoverable without allow list. True "passkeys" that sync. Handled by Credential Manager.
- **Non-discoverable credentials**: Require allow list. Handled by GMS FIDO2.

## Biometric Authentication

**For WebAuthn**: Biometric auth is entirely delegated to the platform (Android Credential Manager system UI or Google Play Services FIDO2 UI). Firefox does NOT show its own biometric prompt for WebAuthn.

**For Autofill (separate from WebAuthn)**: Fenix has its own `DeviceCredentialAuthenticator` using `KeyguardManager.createConfirmDeviceCredentialIntent()` and `BiometricAuthenticator` using AndroidX `BiometricPrompt`. This is for unlocking saved logins/credit cards, not for WebAuthn.

## Related Origin Check

When RP ID doesn't match the caller origin:
1. `WebAuthnRelatedOriginFetcher.sys.mjs` fetches `https://{rpId}/.well-known/webauthn`
2. If caller origin is in the list: prompt shown for user confirmation
3. On Android: `GeckoSession.WebAuthnRelatedOriginPrompt` → `GeckoPromptDelegate` → `WebAuthnRelatedOriginDialogFragment`

## Error Handling

### Error Propagation Chain
```
Android (CM/GMS error)
  → WebAuthnUtils.Exception(message)
    → WebAuthnTokenManager (completes GeckoResult exceptionally)
      → AndroidWebAuthnService (MozPromise rejects with AndroidWebAuthnError)
        → WebAuthnService / WebAuthnTransactionParent (maps to nsresult)
          → WebAuthnHandler (rejects JS Promise with DOMException)
```

### Error Code Mapping
| GMS Error String | DOM Error |
|-----------------|-----------|
| SECURITY_ERR | SecurityError |
| NOT_SUPPORTED_ERR | NotSupportedError |
| INVALID_STATE_ERR | InvalidStateError |
| NOT_ALLOWED_ERR | NotAllowedError |
| ABORT_ERR | AbortError |
| UNKNOWN_ERR | UnknownError |
| TIMEOUT_ERR | TimeoutError |
| DATA_ERR | DataError |
| NETWORK_ERR | NetworkError |

### Cancellation
- User cancels system UI → `TYPE_USER_CANCELED` → `ABORT_ERR`
- `WebAuthnHandler::CancelTransaction()` → `RequestCancel` IPC → `CancelTransaction()` → rejects pending promises

## Security Architecture

1. **Origin validation**: `IsWebAuthnAllowedInContext()` + `IsValidRpId()` before any platform call
2. **Related origin enforcement**: `.well-known/webauthn` check + user confirmation
3. **Attestation consent**: Controlled by `security.webauthn.always_allow_direct_attestation` pref (default: anonymized)
4. **Credential Manager permissions**: Catches `SecurityException`, falls back to GMS FIDO2
5. **Feature check**: `PackageManager.FEATURE_CREDENTIALS` (some OEMs disable Credential Manager)
6. **GMS whitelisting**: Official builds use `Fido2PrivilegedApiClient`; non-official use `Fido2ApiClient` (requires Digital Asset Links)
7. **Thread safety**: `AtomicBoolean` prevents double-completion; `ThreadUtils.runOnUiThread()` for UI ops

## Key Files

| File | Role |
|------|------|
| `dom/webauthn/WebAuthnHandler.h/.cpp` | Content process WebAuthn handler |
| `dom/webauthn/WebAuthnTransactionParent.cpp` | Parent process IPC + origin validation |
| `dom/webauthn/PWebAuthnTransaction.ipdl` | IPDL protocol definition |
| `dom/webauthn/WebAuthnService.cpp/.h` | Service routing + attestation consent |
| `dom/webauthn/nsIWebAuthnService.idl` | XPCOM interface |
| `dom/webauthn/AndroidWebAuthnService.cpp/.h` | Android JNI bridge (C++ side) |
| `dom/webauthn/WebAuthnRelatedOriginFetcher.sys.mjs` | .well-known/webauthn fetcher |
| `geckoview/.../WebAuthnTokenManager.java` | GMS + Credential Manager orchestrator |
| `geckoview/.../WebAuthnCredentialManager.java` | Android 14+ Credential Manager API |
| `geckoview/.../util/WebAuthnUtils.java` | Response parsing, data classes |
| `geckoview/.../GeckoRuntime.java` | ActivityDelegate + startActivityForResult |
| `A-C/.../feature/webauthn/WebAuthnFeature.kt` | Fenix Activity delegate for FIDO2 |
| `A-C/.../engine-gecko/activity/GeckoActivityDelegate.kt` | ActivityDelegate bridge |
| `A-C/.../concept/engine/activity/ActivityDelegate.kt` | startIntentSenderForResult interface |
| `A-C/.../prompts/GeckoPromptDelegate.kt` | WebAuthn-related-origin prompt bridge |
| `A-C/.../prompts/PromptFeature.kt` | Creates WebAuthnRelatedOriginDialogFragment |
| `A-C/.../prompts/WebAuthnRelatedOriginDialogFragment.kt` | BottomSheet for related origin |

(A-C = `android-components` root)

## Implementation Guide

To implement passkeys in another browser:

1. **Two-tier platform strategy**: Try Android Credential Manager first (Android 14+), fall back to GMS FIDO2
2. **JSON-based protocol**: Serialize WebAuthn options to JSON matching spec structure for Credential Manager
3. **Activity delegation**: Implement `ActivityDelegate` pattern for GMS PendingIntent flow
4. **Decision tree**: Check GMS for non-discoverable creds first, then CM for discoverable
5. **Origin validation**: Implement `.well-known/webauthn` check per spec
6. **Error mapping**: Map platform errors to DOMException codes
7. **Biometric delegation**: Let platform handle biometrics (do NOT implement custom biometric prompts)
8. **Credential Manager constants**: Replicate `TYPE_PUBLIC_KEY_CREDENTIAL` and bundle keys from `androidx.credentials` (or use Jetpack library)

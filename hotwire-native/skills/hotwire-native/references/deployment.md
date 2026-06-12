# Deployment — Device URLs, TestFlight, Play Internal Testing

## Which base URL, when

`localhost` on a physical device resolves to the device itself, not your machine.
Use the right URL per target:

| Target | `baseURL` |
|---|---|
| iOS simulator | `http://localhost:3000` |
| Android emulator | `http://10.0.2.2:3000` (the emulator's alias for host machine) |
| Physical device (either platform) | `http://<your-LAN-IP>:3000` |
| Production build | `https://your-app.example.com` |

```bash
networksetup -getinfo Wi-Fi        # find your Mac's LAN IP
bin/rails server -b 0.0.0.0        # bind to all interfaces so devices on the LAN can connect
```

Without `-b 0.0.0.0`, Rails only listens on loopback and physical devices get a blank
screen / connection error.

## Switching URLs per build

### iOS — `#if DEBUG`

```swift
#if DEBUG
let baseURL = URL(string: "http://localhost:3000")!     // or LAN IP for device debugging
#else
let baseURL = URL(string: "https://your-app.example.com")!
#endif
```

For local ports also add an ATS exception:
`NSAppTransportSecurity → NSAllowsLocalNetworking = YES` (DEBUG only).

### Android — `buildConfigField` per build type

```kotlin
// app/build.gradle.kts
buildTypes {
    debug {
        buildConfigField("String", "BASE_URL", "\"http://10.0.2.2:3000\"")
    }
    release {
        buildConfigField("String", "BASE_URL", "\"https://your-app.example.com\"")
    }
}
buildFeatures {
    buildConfig = true   // REQUIRED — BuildConfig.BASE_URL won't compile without it
}
```

```kotlin
val baseURL = BuildConfig.BASE_URL
```

`buildConfig = true` missing from `buildFeatures` is the classic
"`BuildConfig.BASE_URL` unresolved reference" cause.

## TestFlight (iOS) — minimum pre-archive steps

> Store-submission steps below are docs/experience-sourced, not re-verified against the
> current App Store Connect flow this cycle.

Do these **before** the first archive attempt, or the archive/upload fails:

1. **1024×1024 PNG app icon** in `Assets → AppIcon` — archiving without it errors out.
2. **`ITSAppUsesNonExemptEncryption` = `NO`** in `Info.plist` — otherwise you answer
   the encryption compliance questionnaire manually on every single upload.
3. **Bundle ID in Xcode matches the App Store Connect app record exactly.**
4. Signing: enable "Automatically manage signing" (Signing & Capabilities) with your
   team selected.

Then: App Store Connect → register the App ID → create the app → Xcode
`Product → Archive` → Distribute to App Store Connect → wait for processing →
TestFlight tab → add internal testers (up to 100, no App Review required).

## Play Internal Testing (Android) — minimums

1. **Identity verification in Play Console takes days — start it the moment you create
   the developer account** ($25 one-time fee). It blocks every upload until approved.
2. **Keystore**: generate it once, back it up **outside git** — losing it means you can
   never update the app under the same listing. Don't commit it; don't commit its
   passwords.
3. `buildConfig = true` set (see above) so release builds compile.
4. Upload format is a **signed App Bundle (`.aab`)**: Build → Generate Signed App
   Bundle, then upload to the Internal testing track (up to 100 testers, no review).

## Internal-testing comparison

| | TestFlight | Play Internal Testing |
|---|---|---|
| Account cost | $99/year | $25 one-time |
| Identity check | Immediate | Manual, can take days |
| Max testers (internal) | 100 | 100 |
| Review required | No | No |
| Upload artifact | Xcode archive | Signed `.aab` |

## Quick tells

| Symptom | Likely cause |
|---|---|
| Blank screen on device, fine on simulator | `localhost` baseURL on device, or Rails not bound to `0.0.0.0` |
| Archive fails immediately | Missing 1024×1024 app icon |
| Upload asks encryption questions every time | `ITSAppUsesNonExemptEncryption` not set in `Info.plist` |
| `BuildConfig.BASE_URL` unresolved | `buildConfig = true` missing |
| Play upload blocked | Identity verification still pending |

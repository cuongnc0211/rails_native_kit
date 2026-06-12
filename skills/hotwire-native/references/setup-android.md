# Android Setup

How to bootstrap an Android app with `hotwire-native-android`. Content here is docs-sourced
(hotwire-native-android 1.2.8) — not yet verified in a real build. iOS-verified counterpart: `setup-ios.md`.

## Step 1 — Add the Gradle dependency

In `app/build.gradle.kts`:

```kotlin
dependencies {
    implementation("dev.hotwire:navigation:1.2.8")
}
```

Check [github.com/hotwired/hotwire-native-android/releases](https://github.com/hotwired/hotwire-native-android/releases)
for the current version before adding.

## Step 2 — Enable BuildConfig generation

If you plan to switch `baseURL` per build variant (debug / release), you need `BuildConfig`:

```kotlin
// app/build.gradle.kts
android {
    buildFeatures {
        buildConfig = true   // required — omitting it causes a compile error for BuildConfig.BASE_URL
    }

    buildTypes {
        debug {
            buildConfigField("String", "BASE_URL", "\"http://10.0.2.2:3000\"")
        }
        release {
            buildConfigField("String", "BASE_URL", "\"https://your-app.com\"")
        }
    }
}
```

`10.0.2.2` is the Android emulator's loopback address for the Mac host. On a physical device on the
same network, use your Mac's LAN IP instead.

| Context | URL |
|---|---|
| Emulator | `http://10.0.2.2:3000` |
| Physical device (local dev) | `http://<Mac-LAN-IP>:3000` |
| Production | Full HTTPS domain |

## Step 3 — Create an Application subclass

Global Hotwire configuration (path config, fragment registration) belongs in `Application.onCreate()`,
which fires before any Activity or Fragment:

```kotlin
// DemoApplication.kt
class DemoApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        Hotwire.loadPathConfiguration(
            context = this,
            location = PathConfiguration.Location(
                remoteFileUrl = "${BuildConfig.BASE_URL}/configurations/android_v1.json"
            )
        )
        // Register native fragment destinations here later (see native-screens-android.md)
    }
}
```

Register it in `AndroidManifest.xml`:

```xml
<application
    android:name=".DemoApplication"
    ...>
```

## Step 4 — Create MainActivity

`HotwireActivity` is the single Activity entry point. Keep it thin:

```kotlin
// MainActivity.kt
class MainActivity : HotwireActivity() {
    override fun navigatorConfigurations() = listOf(
        NavigatorConfiguration(
            name       = "main",
            navigatorHostId = R.id.main_nav_host,
            startLocation   = BuildConfig.BASE_URL
        )
    )
}
```

Add the corresponding `FragmentContainerView` to your activity layout:

```xml
<!-- res/layout/activity_main.xml -->
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/main_nav_host"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

## Step 5 — Android wildcard rule in path config (required)

Unlike iOS, Android requires an explicit wildcard rule that maps all URLs to the default web fragment.
**This rule must be first** — Android matches rules in order, and a catch-all placed later would shadow
specific rules above it.

```json
{
  "settings": {},
  "rules": [
    {
      "patterns": [".*"],
      "properties": {
        "uri": "hotwire://fragment/web",
        "pull_to_refresh_enabled": true
      }
    }
  ]
}
```

iOS does not need this wildcard — it falls back to the web view controller automatically. See
`path-configuration.md` for the full JSON structure.

## Common gotchas

| Symptom | Cause | Fix |
|---|---|---|
| `BuildConfig.BASE_URL` compile error | `buildConfig = true` missing | Add to `buildFeatures` block |
| App crashes navigating to native Fragment | Fragment class not registered | `Hotwire.registerFragmentDestinations(...)` in `Application.onCreate()` |
| Emulator can't reach Rails server | Wrong `baseURL` | Use `10.0.2.2` not `localhost` |
| All URLs fall through to wrong screen | Missing or mis-ordered wildcard rule | `".*"` wildcard must be the **first** rule in `android_v1` |

## What comes next

- Path configuration JSON rules → `path-configuration.md`
- Tab bar and multiple navigators → `navigation-and-tabs.md`
- Native Compose screens → `native-screens-android.md`
- Bridge components → `bridge-components-android.md`

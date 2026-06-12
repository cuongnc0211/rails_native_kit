# iOS Setup

How to bootstrap an iOS app with `hotwire-native-ios`. The critical difference from older guides:
the modern Xcode **App** template uses the SwiftUI lifecycle — you must switch it to UIKit manually.

## Step 1 — Add the Swift package

In Xcode: **File → Add Package Dependencies…**

```
https://github.com/hotwired/hotwire-native-ios
```

After resolving, go to **General → Frameworks, Libraries, and Embedded Content** and confirm
`HotwireNative` is listed for your target. If it is missing, add it there — the package resolves but
does **not** link automatically.

✅ Verified (hotwire-native-ios 1.2.2, Xcode 17): forgetting to link is the most common reason the
import compiles but the app crashes at runtime.

## Step 2 — Switch to UIKit lifecycle

The Xcode **App** template generates `XxxApp.swift` + `ContentView.swift` (SwiftUI lifecycle).
`Navigator` and `HotwireTabBarController` need a UIKit `SceneDelegate`. Do this once:

1. Delete `XxxApp.swift` and `ContentView.swift`.
2. Create `AppDelegate.swift` with `@main` and a `configurationForConnecting` method that assigns
   the `SceneDelegate` class — this replaces the Info.plist scene-manifest approach (no plist edits
   needed):

```swift
// AppDelegate.swift
import HotwireNative
import UIKit

let baseURL = URL(string: "http://localhost:3000")!

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        Hotwire.loadPathConfiguration(from: [
            .server(baseURL.appending(path: "configurations/ios_v1.json"))
        ])
        Hotwire.registerBridgeComponents([]) // add component classes here later
        return true
    }

    func application(_ application: UIApplication,
        configurationForConnecting connectingSceneSession: UISceneSession,
        options: UIScene.ConnectionOptions) -> UISceneConfiguration {
        let config = UISceneConfiguration(name: "Default", sessionRole: connectingSceneSession.role)
        config.delegateClass = SceneDelegate.self
        return config
    }
}
```

3. Create `SceneDelegate.swift`:

```swift
// SceneDelegate.swift
import HotwireNative
import UIKit

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private let navigator = Navigator(
        configuration: .init(name: "main", startLocation: baseURL)
    )

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }
        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = navigator.rootViewController
        window?.makeKeyAndVisible()
        navigator.start()
    }
}
```

✅ Verified (hotwire-native-ios 1.2.2, Pave-AI build):
- `Navigator(configuration:)` — init requires `configuration`; the no-arg `Navigator()` was removed.
- `navigator.start()` handles the initial load via `startLocation`. Do **not** call `navigator.route(baseURL)` separately.
- Tab bar apps: replace `Navigator` with `HotwireTabBarController` (see `navigation-and-tabs.md`).

## Step 3 — Allow localhost traffic (debug only)

The iOS simulator blocks plain HTTP by default (App Transport Security). Add to `Info.plist`:

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsLocalNetworking</key>
    <true/>
</dict>
```

This allows `localhost` and `.local` only. Production traffic over HTTPS is unaffected.

## Step 4 — Set the production URL

```swift
// SceneDelegate.swift (or a shared constants file)
#if DEBUG
let baseURL = URL(string: "http://localhost:3000")!
#else
let baseURL = URL(string: "https://your-app.com")!
#endif
```

| Context | URL |
|---|---|
| Simulator | `http://localhost:3000` |
| Physical device (local dev) | `http://<Mac-IP>:3000` (find in System Settings → Wi-Fi) |
| Production | Full HTTPS domain |

## Optional: useful Hotwire config flags

```swift
// in AppDelegate.application(_:didFinishLaunchingWithOptions:)
#if DEBUG
Hotwire.config.debugLoggingEnabled = true
#endif
Hotwire.config.backButtonDisplayMode = .minimal  // hides "Back" label text
Hotwire.config.showDoneButtonOnModals = true
Hotwire.config.applicationUserAgentPrefix = "DemoApp/1.0;"
```

The user-agent prefix appended here appears in every web view request — useful for server-side
`hotwire_native_app?` detection alongside custom app version checks.

## What comes next

- Path configuration endpoint on Rails → `rails-integration.md`
- Path config JSON rules → `path-configuration.md`
- Tab bar with multiple navigators → `navigation-and-tabs.md`
- Bridge components registration → `bridge-components-ios.md`

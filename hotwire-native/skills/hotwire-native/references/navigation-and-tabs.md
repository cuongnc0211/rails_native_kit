# Navigation and Tabs

How Hotwire Native handles navigation between pages, and how to set up a multi-tab app.

## Navigation model

Each `Navigator` manages two sessions (main stack + modal stack) and a `WKWebView`. It intercepts
Turbo link taps, resolves them against path configuration, and either pushes a new web view controller
or presents a modal.

The path config `context` property drives the choice:

| `context` value | Result |
|---|---|
| _(absent)_ | Push on the main nav stack |
| `"modal"` | Present as a modal sheet |

```json
{ "patterns": ["/new$", "/edit$"], "properties": { "context": "modal", "pull_to_refresh_enabled": false } }
```

`pull_to_refresh_enabled: false` on modals is Android-only (iOS ignores it), but setting it on both
platforms is harmless and correct.

## Tab bar setup (iOS)

✅ Verified (hotwire-native-ios 1.2.2, Pave-AI build):

Replace the single `Navigator` in `SceneDelegate` with `HotwireTabBarController`:

```swift
// SceneDelegate.swift
import HotwireNative, UIKit

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private let tabBarController = HotwireTabBarController()  // no navigatorDelegate needed for simple cases

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession,
               options connectionOptions: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }
        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = tabBarController
        window?.makeKeyAndVisible()
        tabBarController.load(HotwireTab.all)
    }
}
```

Define tabs in a separate model file — one file to change when tabs change:

```swift
// Models/Tabs.swift
extension HotwireTab {
    static let all: [HotwireTab] = [.home, .profile]
    static let home    = HotwireTab(title: "Home",    image: UIImage(systemName: "house")!,                url: baseURL)
    static let profile = HotwireTab(title: "Profile", image: UIImage(systemName: "person.crop.circle")!,  url: baseURL.appending(path: "profile"))
}
```

`selectedImage` is optional — omit it to reuse `image` for both states. Each tab gets its own
`Navigator`; tab switches are instant after the first load. Route programmatically:

```swift
tabBarController.activeNavigator.route(someURL)
```

## Tab bar setup (Android)

(Docs-sourced — not yet verified in a real build.)

```kotlin
// MainActivity.kt
private lateinit var bottomNavigationController: HotwireBottomNavigationController
override fun navigatorConfigurations() = mainTabs.navigatorConfigurations

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    if (savedInstanceState == null) {
        bottomNavigationController = HotwireBottomNavigationController(
            this, findViewById(R.id.bottom_nav)
        )
        bottomNavigationController.load(mainTabs, 0)
    }
}
```

```kotlin
val mainTabs = listOf(
    HotwireBottomTab(
        title = "Home",
        iconResId = android.R.drawable.ic_menu_today,
        configuration = NavigatorConfiguration("home", R.id.home_nav_host, BuildConfig.BASE_URL)
    )
)
```

Layout: one `FragmentContainerView` per tab — `android:id` must match `navigatorHostId`.

## Custom navigation handling (iOS)

Implement `NavigatorDelegate.handle(proposal:from:)` to intercept navigation and return a custom
view controller for specific paths:

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate, NavigatorDelegate {
    private let tabBarController = HotwireTabBarController()   // delegate set below

    func scene(_ scene: UIScene, ...) {
        // ...
        tabBarController.load(HotwireTab.all)
        // wire delegate after load so activeNavigator exists
    }

    func handle(proposal: VisitProposal, from navigator: Navigator) -> ProposalResult {
        switch proposal.viewController {
        case "map":
            return .acceptCustom(MapViewController(url: proposal.url, navigator: navigator))
        default:
            return .accept   // default: show web view
        }
    }
}
```

`proposal.viewController` comes from the path config `view_controller` property (iOS only). See
`native-screens-ios.md`.

## Snapshot cache

Each `Navigator` keeps a Turbo snapshot per URL. Tab switches are instant after the first load.
To force a fresh server load after a mutation (e.g. after a form submit):

```swift
navigator.session.clearSnapshotCache()
```

## Sizing guidance

- 2–5 tabs → `HotwireTabBarController` / `HotwireBottomNavigationController`.
- 6+ tabs → reconsider IA. iOS shows "More" overflow automatically; Android has no built-in overflow.

# Native Screens — iOS

When to drop from HTML to a fully native SwiftUI screen, and how to wire it up via path config.

## When to go native

Keep the bar high — every native screen adds deploy coordination (server API + client code must ship
together). Default to HTML. Go native only for:

| Trigger | Examples |
|---|---|
| Native API | Camera, biometrics, ARKit, HealthKit |
| Maps | MapKit at full interaction fidelity |
| Home / launch screen | Speed, offline cache (HEY, Basecamp pattern) |

If only a single control needs to feel native (a submit button, a menu), use a bridge component
instead (`bridge-components-ios.md`) — much less code.

## Step 1 — Add a path config rule

```json
{ "patterns": ["/dashboard"], "properties": { "view_controller": "dashboard" } }
```

The `view_controller` string is read by your `NavigatorDelegate` on iOS only.

## Step 2 — Handle in NavigatorDelegate

✅ Verified (hotwire-native-ios 1.2.2): `proposal.viewController` reads `properties.viewController`
from path config. Return `.acceptCustom(vc)` to replace the default web view:

```swift
// SceneDelegate.swift or wherever you hold NavigatorDelegate
extension SceneDelegate: NavigatorDelegate {
    func handle(proposal: VisitProposal, from navigator: Navigator) -> ProposalResult {
        switch proposal.viewController {
        case "dashboard":
            return .acceptCustom(DashboardViewController(url: proposal.url, navigator: navigator))
        default:
            return .accept
        }
    }
}
```

`ProposalResult` cases: `.accept` (default web VC), `.acceptCustom(UIViewController)`, `.reject`.

For tab bar apps, pass `navigatorDelegate` when creating `HotwireTabBarController`:
```swift
HotwireTabBarController(navigatorDelegate: self)
```

## Step 3 — Build the view controller

A native screen that hosts a SwiftUI view:

```swift
// DashboardViewController.swift
import HotwireNative, SwiftUI, UIKit

final class DashboardViewController: UIViewController {
    private let url: URL
    private let navigator: Navigator

    init(url: URL, navigator: Navigator) {
        self.url = url
        self.navigator = navigator
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        let hosting = UIHostingController(rootView: DashboardView(url: url))
        addChild(hosting)
        view.addSubview(hosting.view)
        hosting.view.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            hosting.view.topAnchor.constraint(equalTo: view.topAnchor),
            hosting.view.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            hosting.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hosting.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
        ])
        hosting.didMove(toParent: self)
    }
}
```

## Step 4 — Serve JSON from Rails

```ruby
# app/controllers/dashboards_controller.rb
def show
  @stats = current_user.stats
  respond_to do |format|
    format.html
    format.json { render json: { streak: @stats.streak, completed: @stats.completed } }
  end
end
```

The native screen fetches `url.appendingPathExtension("json")` — same Rails controller, different
format. The HTML response continues to work for web users.

## Key constraints

- **Deploy coordination**: the `view_controller` string in path config and the Swift `case` must match.
  A path config update that references a `view_controller` value the current app binary doesn't handle
  will fall back to `.accept` (web view) — safe, but silent.
- **No `NavigatableView` protocol in 1.2.2**: this protocol does not exist in the current SDK.
  Use plain `UIViewController` + `UIHostingController` as shown above.
- **Passing `navigator`**: the view controller receives `navigator` at init time so it can trigger
  further navigation (e.g. push a detail page from within a native screen).

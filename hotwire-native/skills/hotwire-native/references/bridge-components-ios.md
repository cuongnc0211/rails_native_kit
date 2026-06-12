# Bridge Components — iOS

✅ Verified (hotwire-native-ios 1.2.2, Pave-AI build). The entire Swift section reflects 1.2.x API.
**Three APIs changed between older SDK previews and 1.2.x** — the patterns below are correct.

## What bridge components are

Progressive enhancement for a single HTML element: keep the HTML working as a fallback, layer a
native control on top. Less code than a full native screen; no deploy coordination beyond ensuring
the component is registered before the user hits that page.

The JS + HTML sides are identical for iOS and Android. Only the Swift/Kotlin implementation differs.

## Critical 1.2.x API notes

### 1. No `didUpdateBarButton` delegate method

`delegate?.bridgeComponent(self, didUpdateBarButton: button, for: .right)` does **not exist** in
1.2.2. Get the host view controller via `delegate?.destination as? UIViewController` and set
`navigationItem` directly.

### 2. `jsonData` replaces `message.metadata` for payload

`Message.metadata` in 1.2.2 is a typed struct exposing only `url` — not a dictionary. The bridge
payload arrives in `message.jsonData: String`. The `data<T>()` generic decoder is `internal` (not
callable from app code). Decode `jsonData` yourself.

### 3. Swift 6: `name` must be `nonisolated`

Plain `override class var name` is treated as main-actor-isolated in Swift 6, conflicting with the
base class. Compiler error: _"Main actor-isolated class property 'name' has different actor isolation
from protocol requirement."_ Fix: `override nonisolated class var name: String`.

## Minimal working example

```swift
// BridgeComponents/ButtonComponent.swift
import HotwireNative, UIKit

final class ButtonComponent: BridgeComponent {
    override nonisolated class var name: String { "button" }   // ← nonisolated (Swift 6)

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController             // ← no didUpdateBarButton
    }

    override func onReceive(message: Message) {
        guard message.event == "connect", let vc = viewController else { return }

        let data = decode(message.jsonData)                    // ← jsonData, not metadata
        let action = UIAction { [weak self] _ in self?.reply(to: "connect") }

        let item: UIBarButtonItem
        if let symbol = data?.symbol, let image = UIImage(systemName: symbol) {
            item = UIBarButtonItem(image: image, primaryAction: action)
        } else {
            item = UIBarButtonItem(title: data?.title ?? "Action", primaryAction: action)
        }

        var items = vc.navigationItem.rightBarButtonItems ?? []
        items.append(item)                                     // ← rightBarButtonItems (plural)
        vc.navigationItem.rightBarButtonItems = items
    }

    private func decode(_ jsonData: String) -> ButtonData? {
        guard let raw = jsonData.data(using: .utf8) else { return nil }
        return try? JSONDecoder().decode(ButtonData.self, from: raw)
    }
}

private struct ButtonData: Decodable {
    let symbol: String?
    let title: String?
}
```

**Always use `rightBarButtonItems` (plural) and append** — assigning to the singular
`rightBarButtonItem` silently overwrites any existing button (e.g. a back button override or another
component's button already placed there).

## Registration

```swift
// AppDelegate.application(_:didFinishLaunchingWithOptions:)
Hotwire.registerBridgeComponents([ButtonComponent.self])
```

Register all components before `navigator.start()` or `tabBarController.load(_:)`.

## The JS side (shared with Android)

```javascript
// app/javascript/controllers/bridge/button_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "button"   // matches BridgeComponent.name

  connect() {
    super.connect()
    this.send("connect", { symbol: "person.crop.circle", title: "Profile" }, () => {
      // native button was tapped — e.g. follow the underlying link
      this.bridgeElement.click()
    })
  }
}
```

Pin `@hotwired/hotwire-native-bridge` via importmap (use esm.sh):
```
pin "@hotwired/hotwire-native-bridge", to: "https://esm.sh/@hotwired/hotwire-native-bridge@1.2.2"
```

## reply(to:) constraints

`reply(to: event)` replies to the **last received message** for that event name. Two distinct buttons
sharing one component name and the `"connect"` event will both reply to whichever connected last.

Solutions:
- **Distinct component names** per button (safest)
- One button → native action sheet (single `"connect"` event)
- One button that navigates to a hub page (simplest; what shipped in the Pave-AI build)

## HTML markup

```erb
<button
  data-controller="bridge--button"
  data-bridge-symbol="person.crop.circle"
  data-bridge-title="Profile">
  Profile  <%# shown if bridge doesn't connect (older app / web browser) %>
</button>
```

Hide the HTML fallback only after bridge `connect` succeeds — progressive enhancement means older
app versions still see the HTML button.

# Bridge Components — Protocol & Shared JS/HTML Side

Bridge components swap a single HTML control (a button, a menu, a form submit) for a
native equivalent — without rewriting the screen. One JS controller + one HTML attribute
set powers **both** iOS and Android; only the native class differs per platform.

Native sides: see `bridge-components-ios.md` and `bridge-components-android.md`.

## The message protocol

A two-way channel between the web view and native code:

| Direction | API | Typical use |
|---|---|---|
| JS → native | `this.send(event, data, callback)` | "I'm here, render a native button titled X" |
| Native → JS | `reply(to: event)` (Swift) / `reply(event)` (Kotlin) | "User tapped the native button — run your callback" |

Each message carries an `event` name and a JSON payload built from the data you pass
plus `data-bridge-*` attributes on the element.

## JS side (Stimulus)

`@hotwired/hotwire-native-bridge` (npm latest: **1.2.2**) ships `BridgeComponent`,
a Stimulus `Controller` subclass:

```javascript
// app/javascript/controllers/form_component_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "form"          // must match the native component's name

  connect() {
    super.connect()
    const element = this.bridgeElement                  // BridgeElement wrapper
    const title = element.bridgeAttribute("title")      // reads data-bridge-title
    this.send("connect", { title }, () => {
      this.element.click()           // native button tapped → submit the form
    })
  }
}
```

Useful `BridgeComponent` properties: `this.enabled` (native app supports this
component), `this.bridgeElement` (wraps `this.element` with `.title`,
`.bridgeAttribute()`, `.click()`, …).

### Importmap pin (importmap-rails apps)

✅ Verified (hotwire-native-ios 1.2.2, Rails 8.1)

```ruby
# config/importmap.rb
pin "@hotwired/hotwire-native-bridge", to: "https://esm.sh/@hotwired/hotwire-native-bridge@1.2.2"
```

If a newly added bridge controller never fires with no error, check for a stale
`public/assets` manifest first (Propshaft silently drops the file) — see
`rails-integration.md`.

## HTML side

```erb
<%= form.submit "Save",
      data: { controller: "form-component", bridge_title: "Save" } %>
```

- `data-controller` activates the Stimulus controller (its Stimulus identifier).
- `data-bridge-*` attributes carry the payload (`data-bridge-title`, custom keys).
- `data-controller-optout-ios` / `data-controller-optout-android` disable the
  component on one platform.

## Stimulus identifier ≠ bridge component name

✅ Verified (hotwire-native-ios 1.2.2, Rails 8.1)

The HTML `data-controller` value is the **Stimulus identifier** — it only has to match
the controller's registered name. What the native side matches is **`static component`**.
They are independent: a flat file `form_component_controller.js` with
`static component = "form"` works fine. You do **not** need a `bridge--` folder prefix;
that's a file-organization convention, not a protocol requirement.

## Progressive enhancement rule

The HTML fallback must keep working on the web and in older app versions that don't
register the component. Hide it **only after** the bridge connects:

```javascript
connect() {
  super.connect()
  this.send("connect", { title: this.bridgeElement.title }, () => {
    this.element.click()
  })
  if (this.enabled) this.element.hidden = true   // hide fallback only when bridged
}
```

Never hide the element with plain `hotwire_native_app?` server-side checks for bridged
controls — a native user on an app version without the component would see nothing.

## Build checklist

- [ ] HTML: `data-controller="<stimulus-id>"` + `data-bridge-*` payload attributes
- [ ] JS: `BridgeComponent` subclass, `static component = "<name>"`,
      `this.send("connect", payload, callback)` in `connect()`
- [ ] Fallback: HTML element stays functional; hidden only after bridge connects
- [ ] iOS: Swift `BridgeComponent` subclass with matching `name`; register via
      `Hotwire.registerBridgeComponents([FormComponent.self])` → `bridge-components-ios.md`
- [ ] Android: Kotlin `BridgeComponent` + `BridgeComponentFactory("form", ::FormComponent)`
      registered in your `Application` class → `bridge-components-android.md`
- [ ] Test on web first — the page must work with the bridge entirely absent

## Design constraints (learned in a real build)

✅ Verified (hotwire-native-ios 1.2.2, Rails 8.1)

- **`reply(to: event)` replies to the LAST received message for that event.** Two
  buttons sharing one component name and the same `"connect"` event both reply to
  whichever connected last — wrong action fires. Use distinct component names per
  button, or one button driving a native action sheet, or one button that simply
  navigates to a hub page (simplest, lowest risk).
- New pages get the native control for free: add the `data-*` attributes to the HTML;
  the JS and native code never change.

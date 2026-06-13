---
name: hotwire-native
description: "Build iOS/Android apps powered by a Rails backend using Hotwire Native. Use when working on a Hotwire Native project: iOS/Android setup, path configuration rules, web vs. bridge component vs. native screen decisions, Swift/Kotlin implementations, push notifications, or deployment. Includes verified real-world gotchas for hotwire-native-ios 1.2.2 that prevent the most common compile/runtime errors."
license: MIT
metadata:
  version: "0.1.0"
---

# Hotwire Native Skill

**Verified versions:** hotwire-native-ios 1.2.2 · hotwire-native-android 1.2.8 (docs-sourced) · @hotwired/hotwire-native-bridge 1.2.2 · Rails 8.1

---

## Decision framework

```
One element on screen feels wrong → Bridge Component
Entire screen needs native (maps, camera, home screen) → Native Screen
Everything else → stay Web (change server-side, ship nothing)
```

**Path config controls behavior without an app release:**

| Need | Path config property |
|---|---|
| Modal (new/edit forms) | `context: "modal"` + `pull_to_refresh_enabled: false` |
| iOS native screen | `view_controller: "map"` → handle in `NavigatorDelegate` |
| Android native Fragment | `uri: "hotwire://fragment/native/map"` → register in `Application` |
| Android default web | `".*"` wildcard → `uri: "hotwire://fragment/web"` — **must be first rule** |

iOS has no wildcard requirement — it defaults to web. Android requires the wildcard explicitly, always first.

---

## Fast-path checklist — new iOS app for an existing Rails app

1. **Rails audit** — `turbo-rails` gem present? `/new`, `/edit` routes? HTTPS prod URL? Session cookie auth?
2. **Xcode** — Create App target → delete `XxxApp.swift` + `ContentView.swift` → add `hotwire-native-ios` package → **link to target** (General → Frameworks) → add `AppDelegate` + `SceneDelegate` (see `setup-ios.md`)
3. **ATS** — Add `NSAllowsLocalNetworking = YES` to `Info.plist` for simulator HTTP
4. **Rails path config endpoint** — inherit from `ActionController::Base`, not `ApplicationController` (see `rails-integration.md`)
5. **Tab bar** — swap `Navigator` for `HotwireTabBarController` + `HotwireTab.all` (see `navigation-and-tabs.md`)
6. **Hide web chrome** — `unless hotwire_native_app?` around nav/footer partials in Rails
7. **Bridge component** — JS controller + Swift `BridgeComponent` + register in `AppDelegate` (see `bridge-components-ios.md`)
8. **Push notifications** — Noticed gem + `NotificationTokenComponent` (see `push-notifications.md`)
9. **Deploy** — App icon, `ITSAppUsesNonExemptEncryption`, bundle ID (see `deployment.md`)

---

## Reference file routing table

Load these on demand — SKILL.md loads first, reference files load when the question matches.

| Question / task | Load |
|---|---|
| Prepare Rails app for native clients | `rails-integration.md` |
| Path config JSON structure, rule ordering | `path-configuration.md` |
| iOS project bootstrap, UIKit lifecycle, ATS | `setup-ios.md` |
| Android project bootstrap, Gradle, wildcard rule | `setup-android.md` |
| Tab bar, modal vs push, snapshot cache | `navigation-and-tabs.md` |
| iOS native SwiftUI screen via path config | `native-screens-ios.md` |
| Android native Compose screen via path config | `native-screens-android.md` |
| Bridge component shared concepts (JS + HTML side) | `bridge-components.md` |
| iOS `BridgeComponent` Swift implementation | `bridge-components-ios.md` |
| Android `BridgeComponent` Kotlin implementation | `bridge-components-android.md` |
| APNs / FCM, Noticed gem, push token storage | `push-notifications.md` |
| TestFlight, Play Internal Testing, device URLs | `deployment.md` |
| Symptom → cause diagnosis | `troubleshooting.md` |

---

## Top gotchas (iOS, verified in real builds)

**1. No-arg `Navigator()` removed** — init requires `configuration`:
```swift
Navigator(configuration: .init(name: "main", startLocation: baseURL))
```
Call `navigator.start()` — not `navigator.route(baseURL)`.

**2. Modern Xcode App template uses SwiftUI lifecycle** — no `SceneDelegate` exists.
Delete `XxxApp.swift` + `ContentView.swift`, create `AppDelegate` with `@main` + `configurationForConnecting` that assigns `SceneDelegate.self` programmatically.

**3. No `didUpdateBarButton` delegate in 1.2.2** — get the host VC via:
```swift
delegate?.destination as? UIViewController
```
Then set `navigationItem.rightBarButtonItems` directly.

**4. Bridge payload is in `jsonData`, not `message.metadata`** — decode manually:
```swift
JSONDecoder().decode(ButtonData.self, from: Data(message.jsonData.utf8))
```

**5. Swift 6: `name` override must be `nonisolated`**
```swift
override nonisolated class var name: String { "button" }
```

**6. Propshaft stale `public/assets` silently kills new bridge controllers** — if a new JS file never loads: `rm -rf public/assets` (gitignored) and restart.

**7. `allow_browser versions: :modern` returns 406 for path config fetch** — fix: inherit path config controller from `ActionController::Base`.

**8. Append to `rightBarButtonItems` (plural)** — assigning to the singular overwrites existing buttons.

**9. iOS auto-zooms on `font-size < 16px` inputs — zoom persists after Turbo redirect** — iOS WKWebView zooms into any focused input smaller than 16px. Unlike Safari (which resets on navigation), the zoom state survives Turbo's soft navigation, so the page after login/form-submit appears zoomed in until a hard refresh. Fix: add a global CSS rule so inputs are never smaller than 16px:
```css
@supports (-webkit-touch-callout: none) {
  input, textarea, select {
    font-size: max(16px, 1em);
  }
}
```
`max(16px, 1em)` preserves larger sizes; `@supports (-webkit-touch-callout: none)` limits the rule to iOS/Safari only.

**10. Emoji render as "?" boxes in WKWebView (iOS 26+, unresolved)** — Custom `--font-sans` overrides Tailwind's default stack which included `"Apple Color Emoji"`. Restore it:
```css
@theme {
  --font-sans: "Your Font", ui-sans-serif, system-ui, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Noto Color Emoji";
}
```
**Known issue:** adding emoji fonts back is unconfirmed to fully fix iOS 26 WKWebView — emoji may still render as "?". For static UI, use inline SVG instead of emoji. For dynamic/AI-generated content that may contain emoji, a reliable solution (e.g. Twemoji img replacement) is still needed.

# Troubleshooting

Symptom → likely cause → fix. Entries marked ✅ were hit and resolved in a real build
(hotwire-native-ios 1.2.2, Rails 8.1, Propshaft + importmap, Swift 6 / Xcode 17+).

## Golden workflow: confirm SDK signatures before writing Swift

The SDK API drifts between versions, and most compile/runtime surprises are signature drift.
Before writing native code against `hotwire-native-ios`, grep the **resolved package source**
Xcode already downloaded:

```bash
find ~/Library/Developer/Xcode/DerivedData -path "*SourcePackages/checkouts/hotwire-native-ios*"
grep -rn "public func\|public init\|public var" <that-path>/Source
```

✅ This catches every API mismatch in one pass instead of compile round-trips. Examples it has
caught: `Navigator` now requires a `configuration:` init param (with `startLocation` replacing a
manual route call), and `Message.data<T>()` being `internal` (decode `message.jsonData` yourself).

## Launch and navigation

| Symptom | Likely cause | Fix |
|---|---|---|
| Blank screen on launch | `baseURL` unreachable — server not running, wrong IP, or ATS blocking plain HTTP | Start the server; simulator uses `http://localhost:3000`, Android emulator `http://10.0.2.2:3000`, physical device your LAN IP + `bin/rails server -b 0.0.0.0`. iOS dev: add ATS exception `NSAllowsLocalNetworking = YES` |
| Modal opens as a push (or vice versa) | Path config rule not matching — regex typo, or rule ordering (later rules override earlier) | Test the regex against the URL path; remember properties merge top-to-bottom with later rules winning |
| Android: every screen broken / crash on navigate | Missing wildcard `".*"` rule with `uri: "hotwire://fragment/web"` as the **first** rule | Add it first; specific rules below override per-key |
| Android: crash on URL meant for a native fragment | Fragment class not registered for the `hotwire://fragment/native/...` URI | Register the destination in your Application class before navigating |
| Path config seems ignored entirely | Config endpoint failing, or config loaded after navigation started | See the 406 entry below; load path config in `AppDelegate` / `Application.onCreate` before the navigator starts |
| Tab bar shows a blank first tab | Tabs never loaded, or loaded before navigation was ready | Call the tab controller's `load(...)` after setup completes |

## Rails-side failures ✅

| Symptom | Likely cause | Fix |
|---|---|---|
| Path config fetch returns **406** | Rails 8 `allow_browser versions: :modern` rejects the native URLSession/OkHttp user agent; it's an anonymous before_action you can't `skip_before_action` by name | Make the configurations controller inherit `ActionController::Base` instead of `ApplicationController` (also bypasses app auth — keep the endpoint public) |
| New Stimulus/bridge controller never loads; native bridge button silently missing; **no error anywhere** | Stale `public/assets/.manifest.json` → Propshaft resolves from the manifest even in dev → `Propshaft::MissingAssetError` on the new file → importmap silently drops it | Diagnose: `ActionController::Base.helpers.asset_path("controllers/your_controller.js")` in console — `MissingAssetError` confirms it. Fix: `rm -rf public/assets`, restart server |
| Native users logged out every app launch | Session cookie cleared on app kill; remember cookie only set behind an opt-in checkbox | Set the long-lived signed/encrypted cookie unconditionally when `hotwire_native_app?` (see `rails-integration.md`) |
| ✅ Page looks zoomed in after login / form submit; hard refresh restores it | iOS WKWebView auto-zooms to any focused input with `font-size < 16px` (e.g. Tailwind `text-sm` = 14px). Unlike browser Safari, the zoom state persists across Turbo soft navigations — so the redirect target renders zoomed until a full reload | Add one global CSS rule: `@supports (-webkit-touch-callout: none) { input, textarea, select { font-size: max(16px, 1em); } }` — `max(16px, 1em)` prevents zoom without changing visible size on desktop |
| ⚠️ Emoji render as "?" placeholder boxes (iOS 26+ WKWebView) | Custom font stack overrides Tailwind's default which included `"Apple Color Emoji"`; iOS 26 WKWebView also appears to have a breaking change in emoji rendering | Add emoji fonts back to the stack: `"Apple Color Emoji", "Segoe UI Emoji", "Noto Color Emoji"` as trailing entries in `--font-sans`. **Unconfirmed fix on iOS 26** — for static UI use inline SVG; for AI-generated content (plans, exercises, notes) a Twemoji-style img replacement is the reliable long-term fix |

## Bridge components

| Symptom | Likely cause | Fix |
|---|---|---|
| Native shows the HTML fallback instead of native control | JS never sent `connect` — component not registered natively, JS controller didn't load, or component name mismatch | Register via `Hotwire.registerBridgeComponents([...])`; check the Propshaft entry above; the native `name` must equal the JS `static component`, **not** the Stimulus `data-controller` id |
| ✅ Swift 6 compile error: "Main actor-isolated class property 'name' has different actor isolation" | Base `BridgeComponent.name` is nonisolated; plain override becomes actor-isolated | `override nonisolated class var name: String { "button" }` |
| ✅ Compile error on `message.metadata?["title"]` | 1.2.2 `Metadata` is a typed struct exposing only `url`; the generic `data<T>()` decoder is `internal` | Decode `message.jsonData` yourself: `try? JSONDecoder().decode(MyData.self, from: Data(message.jsonData.utf8))` |
| ✅ No `didUpdateBarButton`-style delegate callback exists | That API isn't in 1.2.2 | Get the hosting screen via `delegate?.destination as? UIViewController` and set `navigationItem` directly |
| ✅ Adding a bar button removes an existing one | `navigationItem.rightBarButtonItem` (singular) overwrites | Use `rightBarButtonItems` (plural) and append |
| ✅ Tapping button A triggers button B's action | `reply(to: event)` replies to the **last received** message for that event; two buttons sharing one component name + event cross-wire | Use distinct component names per button, a single button → native action sheet, or one button that navigates to a hub page |
| Bridge JS import fails | `@hotwired/hotwire-native-bridge` not pinned | `bin/importmap pin @hotwired/hotwire-native-bridge` (or vendor from esm.sh) |

## Build and deploy

| Symptom | Likely cause | Fix |
|---|---|---|
| ✅ Xcode App template has no SceneDelegate to wire `Navigator` into | Modern template uses the SwiftUI lifecycle | Delete `XxxApp.swift`/`ContentView.swift`; add a `@main` `AppDelegate` returning a `UISceneConfiguration` whose `delegateClass` is your SceneDelegate — no Info.plist scene-manifest edits needed |
| ✅ `HotwireNative` module not found despite the package resolving | Package added but not linked to the app target | Target → General → Frameworks, Libraries… → add `HotwireNative` |
| Android: `BuildConfig.BASE_URL` won't compile | `buildConfig = true` missing under `buildFeatures` | Add it in `build.gradle.kts` |
| iOS archive fails immediately | Missing 1024×1024 app icon | Add it to `Assets → AppIcon` before the first archive |
| Every TestFlight upload asks about encryption | `ITSAppUsesNonExemptEncryption` not declared | Set it to `NO` in `Info.plist` (if you only use standard HTTPS) |
| Push works locally, fails in production (or vice versa) | Sandbox vs production APNs routing not switched per environment | With Noticed: `config.development = Rails.env.local?` in the iOS delivery block |
| Duplicate push notifications | Duplicate token rows | Use `find_or_create_by!` when registering device tokens |
| Notification tap opens app home, not the relevant screen | No deep-link path in the payload | Include a `path` in the APNs custom payload and route it through the Navigator on tap |

## Debugging order when "the native thing just doesn't show up" ✅

1. Rails console: `asset_path(...)` for the JS controller → rules out the stale-manifest trap
2. Browser: load the page on desktop, check the console for the Stimulus controller connecting
3. `curl -A "Hotwire Native iOS" <config-url>` → rules out the 406 / auth gate
4. Grep the resolved SDK source (golden workflow above) → rules out API-signature drift
5. Only then suspect your native code

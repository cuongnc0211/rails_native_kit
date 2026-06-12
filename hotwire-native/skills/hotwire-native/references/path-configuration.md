# Path Configuration

A server-hosted JSON file that maps URL patterns to native navigation behavior (modals,
pull-to-refresh, native screen routing). Think of it as the native counterpart of
`config/routes.rb`: change the JSON on the server and every installed app picks up the new
behavior on next launch — no app release, no store review.

## JSON anatomy

```json
{
  "settings": {},
  "rules": [
    { "patterns": [".*"],
      "properties": { "uri": "hotwire://fragment/web", "pull_to_refresh_enabled": true } },

    { "patterns": ["/new$", "/edit$"],
      "properties": { "context": "modal", "pull_to_refresh_enabled": false } },

    { "patterns": ["/items/\\d+/map"],
      "properties": { "view_controller": "map" } }
  ]
}
```

- `settings` — free-form app-level config (feature flags, custom values). Not interpreted by
  Hotwire Native itself; your app code reads it.
- `rules` — array of `{ patterns, properties }`. `patterns` are **regex strings** matched
  against the URL path. ✅ Verified (hotwire-native-ios 1.2.2): this shape is current.

## Rule ordering: rules merge, later rules win

Rules are evaluated top-to-bottom against the URL. **All matching rules apply; properties merge,
and rules later in the array override earlier ones for the same key.** (Official docs: entries
are read sequentially; earlier rules can be overwritten by later ones.)

Practical consequences:

| Rule | Why |
|---|---|
| Put the broad/default rule **first** | It sets defaults; specific rules below override per-key |
| **Android: the wildcard `".*"` rule must be the first rule** | It must set `uri: "hotwire://fragment/web"` and default `pull_to_refresh_enabled` for every URL; without it Android has no fragment to route to |
| iOS needs no wildcard | URLs without a matching rule default to a plain web screen push |
| A regex typo fails silently | The URL falls through to default push behavior — check the regex first when a modal opens as a push |

## Common properties

| Property | Platform | Effect |
|---|---|---|
| `context: "modal"` | both | Present from the bottom instead of pushing onto the stack |
| `pull_to_refresh_enabled: true/false` | Android | Toggle pull-to-refresh; set `false` on modal rules to avoid gesture conflicts |
| `uri: "hotwire://fragment/web"` | Android | Route to the default web fragment (the wildcard default) |
| `uri: "hotwire://fragment/native/<name>"` | Android | Route to a registered native fragment |
| `view_controller: "<name>"` | iOS | Route to a native view controller — handle in the Navigator delegate's route decision |

**Modal convention:** match Rails form routes with `["/new$", "/edit$"]`. Modals suit narrowly
scoped, dismissible tasks; after a successful form submit + redirect, the modal is dismissed and
the destination is pushed.

**Native screen routing** is split per platform: iOS reads `view_controller` and your Navigator
delegate returns the native controller; Android reads `uri` and routes to the fragment class you
registered for it. Both platforms ignore the other's key, so one rule can carry both — or keep
separate `ios_v1` / `android_v1` payloads (recommended).

## Loading the config

```swift
// iOS — AppDelegate, before the Navigator starts
Hotwire.loadPathConfiguration(from: [
  .server(baseURL.appending(path: "configurations/ios_v1.json"))
])
```
✅ Verified (hotwire-native-ios 1.2.2): `Hotwire.loadPathConfiguration(from: [.server(url)])`
is the current API.

```kotlin
// Android — Application/MainActivity onCreate, before navigation starts
Hotwire.loadPathConfiguration(
  context = this,
  location = PathConfiguration.Location(remoteFileUrl = "$baseURL/configurations/android_v1.json")
)
```

Note: `hotwire-native-android` ≥ 1.2.7 exposes the path-configuration **loading state as an
observable**, so you can defer first navigation until the remote config has been applied
(docs-sourced, not yet verified in a real build).

## Server-deployed config = behavior changes without an app release

Because the JSON lives on your server:
- Make `/new` open as a modal tomorrow by editing the controller response — installed apps pick
  it up on next launch.
- Feature-flag native screens: return different rules based on user attributes or flags in the
  Rails action.
- Caveat: the config is fetched on launch — already-running sessions won't see changes until
  restart (the SDK also caches the last good config for offline launches).

## Versioned endpoints for backward compatibility

Serve `ios_v1.json` / `android_v1.json` and bump the version **only for breaking shape changes**:

```ruby
get "configurations/ios_v1", to: "configurations#ios_v1"
get "configurations/ios_v2", to: "configurations#ios_v2"  # newer app builds point here
```

Old app binaries keep fetching `v1` forever — never remove or repurpose a deployed version.
Additive changes (new rules, new properties old clients ignore) can stay within a version;
renaming `settings` keys your app code reads, or restructuring rules in ways old native code
mishandles, requires a new version that only new builds reference.

Serve the endpoint from a controller inheriting `ActionController::Base` — Rails 8's
`allow_browser` guard 406s the native fetch otherwise. Details in `rails-integration.md`.

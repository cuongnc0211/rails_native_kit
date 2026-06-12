# Rails Integration

How to prepare a Rails app to serve Hotwire Native clients. Most of the work is server-side —
the native shells stay thin.

## Detecting native requests: `hotwire_native_app?`

Provided by `turbo-rails`. Returns `true` when the request user agent contains "Hotwire Native"
(or the legacy "Turbo Native"). Available in controllers and views.

```erb
<% if hotwire_native_app? %>
  <%# native-only markup %>
<% end %>
```

It is a user-agent sniff: reliable for UI adaptation, trivially spoofable. **Never** use it for
authorization decisions.

## Hiding web chrome (top nav, footer)

Native apps bring their own navigation (nav bar, tab bar), so the web header/footer must go.
Three approaches — pick by caching needs:

| Approach | Same HTML for web & native? | Use when |
|---|---|---|
| `unless hotwire_native_app?` in ERB | No (fragments HTTP cache) | Authenticated, uncached pages — simplest |
| `native.css` utility classes | Yes | Pages served from HTTP/CDN cache |
| Tailwind custom variant | Yes | Tailwind projects |

✅ Verified (hotwire-native-ios 1.2.2, Rails 8.1, Propshaft + importmap): for authenticated pages
that are never HTTP-cached, plain `unless hotwire_native_app?` around the nav partials is enough.
Reserve the CSS approaches for cache-sensitive pages.

**`native.css` approach** — conditionally link a stylesheet only for native requests:

```erb
<%# layouts/application.html.erb %>
<% if hotwire_native_app? %>
  <%= stylesheet_link_tag "native", "data-turbo-track": "reload" %>
<% end %>
```

```css
/* app/assets/stylesheets/native.css */
.d-hotwire-native-none  { display: none  !important; }
.d-hotwire-native-block { display: block !important; }
```

**Tailwind variant** — set a marker on `<html>`, then define a variant:

```erb
<html data-hotwire-native="<%= hotwire_native_app? %>">
```

```css
/* Tailwind v4 — app.css */
@custom-variant hotwire-native ([data-hotwire-native="true"] &);
```

```js
// Tailwind v3 — tailwind.config.js
plugins: [plugin(({ addVariant }) => addVariant('hotwire-native', '[data-hotwire-native="true"] &'))]
```

Usage: `<nav class="hotwire-native:hidden">…</nav>`.

## Persistent login for native apps

✅ Verified (hotwire-native-ios 1.2.2, Rails 8.1). Native session cookies are cleared when the
app process is killed. If your app only sets a long-lived remember cookie behind an opt-in
"remember me" checkbox, native users must re-login on every launch. Always persist for native:

```ruby
# on successful sign-in
if remember_me || hotwire_native_app?
  cookies.signed[:remember_user_id] = {
    value: user.id, expires: 30.days, httponly: true, same_site: :lax
  }
end
```

(With `cookies.permanent.encrypted[:user_id]` style auth this is already covered — the point is:
the cookie that survives restarts must be set unconditionally for native clients.)

## Path-configuration endpoint

Serve the path config JSON from a versioned route (see `path-configuration.md` for content):

```ruby
# config/routes.rb
get "configurations/ios_v1",     to: "configurations#ios_v1"
get "configurations/android_v1", to: "configurations#android_v1"
```

✅ Verified (Rails 8.1): Rails 8's `allow_browser versions: :modern` returns **406** for the native
config fetch — the URLSession/OkHttp user agent fails the `browser` gem check. `allow_browser`
registers an anonymous-lambda `before_action`, so you cannot `skip_before_action` it by name.
Fix: inherit from `ActionController::Base` directly, which also bypasses app-wide auth:

```ruby
class ConfigurationsController < ActionController::Base  # NOT ApplicationController
  def ios_v1 = render json: { settings: {}, rules: [...] }
end
```

Keep this endpoint public — it must work before login.

## CSRF: when to skip, when never

`skip_before_action :verify_authenticity_token` is correct **only** when all of these hold:
- The endpoint is JSON-only and called from native HTTP code (e.g. a push-token POST) — native
  URLSession/OkHttp requests carry no CSRF token
- Auth is still enforced (`before_action :authenticate_user!`) via the shared session cookie —
  the web view and native HTTP layer share the cookie jar

**Never** skip CSRF on HTML endpoints or on any endpoint without another auth layer. Form
submissions inside the web view go through Turbo and carry the CSRF token normally — they need
no exemption.

## Propshaft gotcha: stale `public/assets` kills new JS silently

✅ Verified (Rails 8.1, Propshaft + importmap) — highest-impact gotcha in this file. If a
leftover `public/assets/.manifest.json` exists (committed by accident, or left from a local
precompile), Propshaft resolves assets from that manifest **even in development**. Any newly
added JS file — e.g. a new Stimulus/bridge controller — raises `Propshaft::MissingAssetError`,
importmap silently drops it, and the controller never loads. Symptom: a bridge component's
native button just never appears, with no visible error.

Diagnose in `rails console`:

```ruby
ActionController::Base.helpers.asset_path("controllers/nav_button_controller.js")
# Propshaft::MissingAssetError → stale manifest is the culprit
```

Fix (dev): `rm -rf public/assets` (gitignored) and restart the server. Production precompile is
unaffected.

## Horizontal overflow masquerades as "auto-zoom"

✅ Verified (hotwire-native-ios 1.2.2). Content a few pixels wider than the viewport gets clipped
in the web view (e.g. the right edge of the nav disappears) — easily misdiagnosed as the web view
zooming. Fix the overflowing element, or guard with `overflow-x-hidden` on `<body>`. It's a
server-side change: users just pull-to-refresh, no app rebuild.

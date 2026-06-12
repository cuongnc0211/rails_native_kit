# Push Notifications — APNs/FCM with the Noticed Gem

Architecture: **Rails (Noticed) → APNs / FCM → device**. Rails never talks to devices
directly — it hands the notification to Apple's or Google's service, addressed by a
device token the app registered earlier. Physical devices required; simulators cannot
receive APNs/FCM pushes.

## Token registration flow

1. A bridge component on a signed-in page sends `"connect"` to native.
2. Native requests notification permission; on grant, the OS returns a device token.
3. The app POSTs the token to Rails, reusing the web session cookie for auth.
4. Rails stores it on a `NotificationToken` model.

### Bridge component (JS — shared across platforms)

```javascript
// app/javascript/controllers/notification_token_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "notification-token"
  connect() {
    super.connect()
    this.send("connect")
  }
}
```

```erb
<%# render only when signed in — Rails guards auth, not Swift/Kotlin %>
<% if user_signed_in? %>
  <meta data-controller="notification-token">
<% end %>
```

The iOS side calls `UNUserNotificationCenter.requestAuthorization` then
`registerForRemoteNotifications()`; the token arrives in
`AppDelegate.application(_:didRegisterForRemoteNotificationsWithDeviceToken:)`.
Android receives tokens via `FirebaseMessagingService.onNewToken`.

### Rails endpoint — idempotent by design

Registration fires on **every app launch**, so the create must be an upsert:

```ruby
class NotificationTokensController < ApplicationController
  before_action :authenticate_user!
  skip_before_action :verify_authenticity_token   # JSON from native; auth via session cookie

  def create
    current_user.notification_tokens.find_or_create_by!(token_params)
    head :created
  end

  private

  def token_params = params.require(:notification_token).permit(:token, :platform)
end
```

Without `find_or_create_by!` you accumulate duplicate token rows → duplicate
notifications per device.

## Sending — Noticed notifier

```ruby
# app/notifiers/comment_notifier.rb
class CommentNotifier < ApplicationNotifier
  required_param :comment

  deliver_by :ios do |config|
    config.device_tokens = -> {
      recipient.notification_tokens.where(platform: :iOS).pluck(:token)  # platform filter!
    }
    config.format = ->(apn) {
      apn.alert = "New comment on your post"
      apn.custom_payload = { path: post_path(params[:comment].post) }    # deep link
    }
    credentials = Rails.application.credentials.ios
    config.bundle_identifier = credentials.bundle_identifier
    config.key_id            = credentials.key_id
    config.team_id           = credentials.team_id
    config.apns_key          = credentials.apns_key
    config.development       = Rails.env.local?    # sandbox vs production APNs
  end
end

# CommentNotifier.with(comment: comment).deliver(post.user)
```

### The two settings that bite

- **`config.development = Rails.env.local?`** routes between the APNs sandbox (debug
  builds) and production APNs (TestFlight/App Store builds). Hardcode it wrong and
  pushes work on your machine but silently fail in production — or vice versa.
- **Platform column filtering**: the `device_tokens` lambda must scope to
  `platform: :iOS` for the `:ios` delivery method. Sending Android FCM tokens to APNs
  fails delivery. Mirror this with a `deliver_by :fcm` block scoped to `:android`.

## Credentials

| Platform | Credential | Where |
|---|---|---|
| iOS / APNs | `.p8` auth key + key ID + team ID + bundle ID (Apple Developer → Keys) | Rails encrypted credentials (`bin/rails credentials:edit`) |
| Android / FCM | Firebase service-account JSON (Firebase Console) | Rails encrypted credentials |

Never put the `.p8` contents in plain env vars or git. If the key is missing or
malformed, the APNs client Noticed uses — the **`apnotic`** gem — raises at send time.

## Failure tells

| Missing / wrong | Symptom |
|---|---|
| `config.development` mismatched to build | Works locally, dead in production (or reverse) |
| `.p8` key absent from encrypted credentials | `apnotic` errors when the notifier delivers |
| No `find_or_create_by!` | Duplicate tokens → duplicate notifications |
| No `platform` filter in `device_tokens` | Android tokens hit APNs → delivery failures |
| No `custom_payload` path | Tapping the notification opens the app home, not the content |
| Testing on simulator | Nothing arrives — APNs/FCM need real devices |

## Deep linking

Include a path in the payload (`apn.custom_payload = { path: ... }`); on launch from a
notification tap, read it and route the navigator to `baseURL + path`. Path
configuration rules then decide how that screen presents (modal, native, web).

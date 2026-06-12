# rails-native-kit — Claude Code Plugin Marketplace

Battle-tested Claude Code skills for Rails developers building mobile apps with
[Hotwire Native](https://native.hotwired.dev) and deploying with [Kamal](https://kamal-deploy.org).

**Unique value:** verified real-world gotchas from actual builds — API deltas, Swift 6
quirks, and Rails/Propshaft traps the official docs don't cover. Not a docs mirror.

---

## Plugins

| Plugin | What it covers | Always-on cost |
|--------|---------------|----------------|
| [`hotwire-native`](#hotwire-native) | iOS/Android setup, path config, native screens, bridge components, push notifications, deployment | ~113 tok |
| [`kamal`](#kamal) | Zero-downtime deployments, accessory setup, environment config, rollbacks | ~72 tok |

---

## Install

### Step 1 — Add the marketplace

```bash
claude plugin marketplace add cuongnc0211/rails_native_kit
```

### Step 2 — Install the plugin(s) you need

```bash
# Hotwire Native skill
claude plugin install hotwire-native@rails-native-kit

# Kamal skill
claude plugin install kamal@rails-native-kit
```

Restart your Claude Code session to activate.

### Local development / offline

```bash
claude plugin marketplace add /path/to/rails_native_kit --scope user
claude plugin install hotwire-native@rails-native-kit
```

---

## hotwire-native

**Verified against:** hotwire-native-ios 1.2.2 · hotwire-native-android 1.2.8 (docs-sourced) · Rails 8.1

Topics covered:

- iOS and Android project setup (UIKit lifecycle, Gradle, ATS)
- Path configuration — JSON anatomy, ordering rules, server-deployed behavior
- Rails integration — `hotwire_native_app?`, hiding web chrome, persistent auth, CSRF
- Navigation — tab bars, modal vs push, snapshot cache
- Native screens — SwiftUI via `UIHostingController`, Jetpack Compose via `HotwireFragment`
- Bridge components — JS ↔ Swift/Kotlin message protocol, progressive enhancement
- Push notifications — APNs/FCM, Noticed gem, token idempotency
- Deployment — TestFlight, Play Internal Testing, device URL switching
- Troubleshooting — symptom → cause table

```
hotwire-native/
├── SKILL.md             # entry point — decision framework, routing table, top gotchas
└── references/          # 13 task-focused files loaded on demand
    ├── rails-integration.md
    ├── path-configuration.md
    ├── setup-ios.md
    ├── setup-android.md
    ├── navigation-and-tabs.md
    ├── native-screens-ios.md
    ├── native-screens-android.md
    ├── bridge-components.md
    ├── bridge-components-ios.md
    ├── bridge-components-android.md
    ├── push-notifications.md
    ├── deployment.md
    └── troubleshooting.md
```

Content marked `✅ Verified (hotwire-native-ios x.y.z, Rails x.y)` was confirmed in a
real build. Unmarked content is docs-sourced.

---

## kamal

Topics covered: zero-downtime deployments, accessory setup, environment configuration, rollbacks.

---

## Further reading

- [Hotwire Native official docs](https://native.hotwired.dev)
- [hotwire-native-ios on GitHub](https://github.com/hotwired/hotwire-native-ios)
- [hotwire-native-android on GitHub](https://github.com/hotwired/hotwire-native-android)
- *Hotwire Native for Rails Developers* by Joe Masilotti (PragProg) — recommended book

---

## License

MIT — see [LICENSE](LICENSE).

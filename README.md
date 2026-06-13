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
| [`kamal`](#kamal) | deploy.yml config, Kamal Proxy/SSL, secrets, accessories, zero-downtime deploys, rollbacks, CI/CD | ~90 tok |

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

**Verified against:** the official Kamal docs (kamal-deploy.org) + `basecamp/kamal` source · Kamal 2.11.x · Kamal Proxy

Topics covered:

- Concepts — push model, deploy loop, Kamal Proxy, locks, hooks, terminology
- Getting started — install, `kamal init`, minimal config, first `setup`/`deploy`
- Configuration — `deploy.yml` servers/roles, registry, builder, asset bridging, SSH
- Secrets & env — `env.clear` vs `env.secret`, `.kamal/secrets`, secret managers, tags
- Kamal Proxy — TLS/Let's Encrypt, host routing, health checks, timeouts, buffering
- Accessories — Postgres/Redis, scheduled jobs (cron role), Docker networking
- Volumes & data — named volumes, bind mounts, `directories:`/`files:`
- Destinations — staging vs production, `deploy.<dest>.yml`, per-destination secrets
- Server provisioning — non-root user, UFW, fail2ban, swap
- Maintenance — rollbacks, locks, `app exec`, logs, aliases, prune
- CI/CD — GitHub Actions, kamal-less build, `--skip-push`, build cache
- Backups — database dumps, volume backups, restore
- Troubleshooting — symptom → cause → fix tables, debugging commands

```
kamal/
├── SKILL.md             # entry point — decision framework, routing table, top gotchas
└── references/          # 13 task-focused files loaded on demand
    ├── concepts.md
    ├── getting-started.md
    ├── configuration.md
    ├── secrets-and-env.md
    ├── kamal-proxy.md
    ├── accessories.md
    ├── volumes-and-data.md
    ├── destinations.md
    ├── server-provisioning.md
    ├── maintenance.md
    ├── ci-cd.md
    ├── backups.md
    └── troubleshooting.md
```

All Kamal content is docs-sourced (verified against the official docs); a real-deploy
verification pass is planned, after which confirmed content will carry `✅ Verified` markers.

---

## Further reading

- [Hotwire Native official docs](https://native.hotwired.dev)
- [hotwire-native-ios on GitHub](https://github.com/hotwired/hotwire-native-ios)
- [hotwire-native-android on GitHub](https://github.com/hotwired/hotwire-native-android)
- *Hotwire Native for Rails Developers* by Joe Masilotti (PragProg) — recommended book
- [Kamal official docs](https://kamal-deploy.org)
- [kamal on GitHub](https://github.com/basecamp/kamal) · [kamal-proxy on GitHub](https://github.com/basecamp/kamal-proxy)

---

## License

MIT — see [LICENSE](LICENSE).

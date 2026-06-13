# Changelog

## kamal v0.1.0 — 2026-06-13

Initial `kamal` skill. Original content distilled from the Kamal tool's own behavior and
re-verified against the official Kamal docs and `basecamp/kamal` source — no third-party
prose carried over.

### Content

- `SKILL.md` — decision framework, 9-step first-deploy checklist, routing table (13 files),
  9 top gotchas, verified-against statement
- 13 reference files in `references/`:
  - `concepts.md` — push model, components, deploy loop, zero-downtime cutover, locks, hooks
  - `getting-started.md` — prerequisites, install, `kamal init`, minimal config, `setup`/`deploy`
  - `configuration.md` — `deploy.yml` servers/roles, registry, builder, asset bridging, SSH
  - `secrets-and-env.md` — `env.clear` vs `env.secret`, `.kamal/secrets`, secret managers, tags
  - `kamal-proxy.md` — `proxy:` block, TLS/Let's Encrypt, healthcheck, timeouts, buffering
  - `accessories.md` — Postgres/Redis, roles/hosts, networking, scheduled jobs (cron role)
  - `volumes-and-data.md` — named volumes, bind mounts, `directories:`/`files:`
  - `destinations.md` — `deploy.<dest>.yml`, merge behavior, per-destination secrets
  - `server-provisioning.md` — non-root user, UFW, fail2ban, swap, Docker
  - `maintenance.md` — rollbacks, locks, `app exec`, logs, aliases, prune
  - `ci-cd.md` — GitHub Actions, kamal-less build, `--skip-push`, registry/gha cache
  - `backups.md` — database dumps via `accessory exec`, volume backups, restore
  - `troubleshooting.md` — symptom → cause → fix tables, debugging commands

### Verified against

- Kamal 2.11.x — the official Kamal docs (kamal-deploy.org) + `basecamp/kamal` source
- Kamal Proxy (`basecamp/kamal-proxy`)
- Docs-sourced only: no real production deploy verified yet. A real-deploy verification pass
  is planned, after which confirmed content will carry `✅ Verified` markers.

### Re-verification trigger

On the next Kamal minor, re-check: the `proxy:` schema (ssl/healthcheck/buffering/timeouts),
`env.secret`/`.kamal/secrets` mechanism, `kamal secrets` adapters, builder cache syntax, and
the `kamal proxy`/`prune`/`lock` subcommands.

---

## hotwire-native v0.1.0 — 2026-06-11

Initial public release. Original skill built from official docs, SDK source, and verified real-world builds.

### Content

- `SKILL.md` — decision framework, 9-step iOS fast-path checklist, routing table (13 files),
  8 top gotchas, verified-versions statement
- 13 reference files in `references/`:
  - `rails-integration.md` — `hotwire_native_app?`, hiding chrome, persistent auth, CSRF, Propshaft gotcha
  - `path-configuration.md` — JSON anatomy, ordering rules, server-deployed config, versioned endpoints
  - `setup-ios.md` — UIKit lifecycle fix, Navigator 1.2.2 API, ATS, config flags
  - `setup-android.md` — Gradle dependency, BuildConfig, HotwireActivity, wildcard rule
  - `navigation-and-tabs.md` — HotwireTabBarController, modal vs push, snapshot cache
  - `native-screens-ios.md` — view_controller path config, handle(proposal:), UIHostingController
  - `native-screens-android.md` — uri registration, HotwireFragment, Compose screen
  - `bridge-components.md` — shared JS/HTML protocol, progressive enhancement, importmap pin
  - `bridge-components-ios.md` — 1.2.2 API (no didUpdateBarButton, jsonData, nonisolated name)
  - `bridge-components-android.md` — BridgeComponent + MenuProvider, BridgeComponentFactory
  - `push-notifications.md` — Noticed gem, token idempotency, APNs/FCM config, deep linking
  - `deployment.md` — device URLs, baseURL switching, TestFlight/Play Internal Testing minimums
  - `troubleshooting.md` — symptom → cause table, golden debugging workflow

### Verified against

- `hotwire-native-ios` 1.2.2 (iOS content verified in a real build — Pave-AI app)
- `hotwire-native-android` 1.2.8 (Android content docs-sourced; real-build verification planned for v1.1)
- `@hotwired/hotwire-native-bridge` 1.2.2
- Rails 8.1, Propshaft + importmap, Swift 6 / Xcode 17+

### Re-verification trigger

When `hotwire-native-ios` 1.3.0 final ships, re-check: bridge bar-button API, Navigator
init, `HotwireTabBarController` init, and any deprecated symbols flagged in Xcode.

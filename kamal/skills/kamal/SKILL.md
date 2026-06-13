---
name: kamal
description: "Deploy Rails (and other Dockerized) apps with Kamal 2. Use when setting up Kamal, writing config/deploy.yml, configuring Kamal Proxy/SSL, managing secrets and env, setting up accessories (Postgres/Redis), zero-downtime deploys, rollbacks, multi-destination, CI/CD, or debugging a failed deploy. Verified against the official Kamal docs (Kamal 2.11.x)."
license: MIT
metadata:
  version: "0.1.0"
---

# Kamal Skill

**Verified against:** the official Kamal docs (kamal-deploy.org) + `basecamp/kamal` source · Kamal 2.11.x · Kamal Proxy. Docs-sourced — not yet re-verified against a real production deploy.

---

## Decision framework

```
Where does this setting belong?
  Non-secret app/runtime value      → env.clear in deploy.yml
  Credential / token / key           → env.secret + .kamal/secrets (gitignored)
  TLS, host routing, health, timeout → proxy: block in deploy.yml
  Per-environment difference         → config/deploy.<dest>.yml (kamal deploy -d <dest>)

Which command?
  First time on a host  → kamal setup     (installs Docker, boots everything)
  Ship a new version    → kamal deploy     (build, push, rolling zero-downtime swap)
  Same image, faster    → kamal redeploy   (skips bootstrap/registry steps)
  Revert                → kamal rollback   (switch back to a kept prior container)
  Inspect / operate     → kamal app logs | app exec | details | audit | proxy logs
```

Kamal pushes over SSH from your machine/CI; servers pull the image from a registry. There is no control plane — `deploy.yml` is the whole truth (see concepts.md).

---

## Fast-path checklist — first deploy of a Dockerized app

1. **Prereqs** — key-based SSH to each server works, Docker locally, a registry, a `Dockerfile`, and an app health endpoint (`/up`) that returns 200 without redirecting (see getting-started.md)
2. **Install + init** — `gem install kamal` → `kamal init` (writes `config/deploy.yml`, `.kamal/secrets`, `.kamal/hooks/*.sample`)
3. **Secrets** — gitignore `.kamal/secrets`; put a registry **access token** there as `KAMAL_REGISTRY_PASSWORD` (see secrets-and-env.md)
4. **deploy.yml core** — `service`, `image`, `servers.web`, `registry`, `builder.arch` (match the server!), `proxy.host` + `ssl: true` (see configuration.md)
5. **Harden the host** — non-root deploy user, UFW, swap; set `ssh.user` accordingly (see server-provisioning.md)
6. **Accessories** — add Postgres/Redis under `accessories:` if the app needs them (see accessories.md)
7. **Bootstrap** — `kamal setup` (one time), then `kamal deploy` for every release
8. **TLS in Rails** — `force_ssl`/`assume_ssl`, and exclude `/up` from the SSL redirect or the probe 301s and the deploy fails (see kamal-proxy.md)
9. **Day-2** — rollbacks, locks, logs, console (see maintenance.md); automate via GitHub Actions (see ci-cd.md)

---

## Reference file routing table

Load on demand — SKILL.md loads first, reference files load when the question matches.

| Question / task | Load |
|---|---|
| How Kamal works, push model, deploy loop, hooks, terminology | `concepts.md` |
| Install, `kamal init`, minimal config, first `setup`/`deploy` | `getting-started.md` |
| `config/deploy.yml` keys: servers, roles, registry, builder, assets | `configuration.md` |
| `env.clear` vs `env.secret`, `.kamal/secrets`, secret managers, tags | `secrets-and-env.md` |
| Kamal Proxy: TLS/SSL, host routing, healthcheck, timeouts, buffering | `kamal-proxy.md` |
| Postgres/Redis accessories, scheduled jobs (cron role) | `accessories.md` |
| Persisting data: volumes, bind mounts, `directories:`/`files:` | `volumes-and-data.md` |
| Staging vs production, `deploy.<dest>.yml`, per-destination secrets | `destinations.md` |
| Server hardening before Kamal: user, UFW, fail2ban, swap | `server-provisioning.md` |
| Rollbacks, locks, `app exec`, logs, aliases, prune | `maintenance.md` |
| Deploy from GitHub Actions, kamal-less build, `--skip-push`, cache | `ci-cd.md` |
| Database dumps, volume backups, restore | `backups.md` |
| Symptom → cause → fix; debugging commands | `troubleshooting.md` |

---

## Top gotchas

**1. `builder.arch` is required and must match the servers.** Build on a different CPU arch and containers fail at runtime with `exec format error`. Set it explicitly:
```yaml
builder:
  arch: amd64   # or arm64 — match your hosts
```
Kamal 2 uses the `docker-container` driver by default: no caching, no multiarch unless you configure it.

**2. Exclude the health-check path from the SSL redirect.** With `ssl: true`, a forced HTTP→HTTPS redirect makes `/up` return 301, the proxy probe never sees 200, and the deploy aborts:
```ruby
config.assume_ssl = true
config.ssl_options = { redirect: { exclude: ->(r) { r.path == "/up" } } }
```

**3. `ssl: true` needs a real resolvable host.** Let's Encrypt cannot issue a certificate for a bare IP — set `proxy.host` to a domain with DNS pointing at the server, port 443 open.

**4. Listing a name in `.kamal/secrets` alone does nothing.** A secret is only exported if its name also appears under `env.secret` (or `registry.password` / `builder.secrets`). And `.kamal/secrets` must be gitignored — it resolves your real credentials (see secrets-and-env.md).

**5. Kamal Proxy is per-server, not a load balancer.** It does gapless cutover on one host. For a multi-server `web` role, put a managed load balancer in front of the hosts.

**6. `service` must never be renamed.** It prefixes container/network/env-file names on the host; renaming orphans the running containers instead of replacing them.

**7. Kamal deploys committed git state.** Uncommitted working-tree changes are not built or shipped. Commit first.

**8. Never force-release a live deploy lock.** `kamal lock release` during an in-flight deploy corrupts state. Check `kamal lock status` + `kamal audit` first (see concepts.md, maintenance.md).

**9. Set `asset_path` to avoid mid-deploy asset 404s.** Old and new containers run side by side during cutover; without bridged, content-hashed assets, in-flight requests hit missing fingerprints (see configuration.md).

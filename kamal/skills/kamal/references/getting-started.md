# Getting Started with Kamal

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

From zero to a deployed app: prerequisites, install, `kamal init`, a minimal `config/deploy.yml`, and the `kamal setup` → `kamal deploy` flow.

## Prerequisites

| Need | Who provides it | Notes |
|---|---|---|
| SSH access to each server | You | Kamal assumes key-based SSH already works; root by default, or set `ssh.user` |
| Docker locally / in CI | You | Used to build and push the image |
| A container registry | You | Docker Hub, GHCR, ECR, etc. — servers pull from here |
| A Dockerfile in the app root | You | App should listen on a known port and log to STDOUT |
| Secrets (`.kamal/secrets`) | You | Holds `KAMAL_REGISTRY_PASSWORD` and friends; never commit |

Kamal installs Docker on the servers for you during `kamal setup`. Everything else on the host — firewall, users, swap — is your job (see server-provisioning.md).

Your app should expose a health endpoint (Rails ships `/up`), log to STDOUT so `kamal app logs` works, and listen on the port the proxy targets (port 80 by default, overridable via `proxy.app_port`).

## Install

Ruby projects — install the gem (add it to the Gemfile for CI so `bundle exec kamal` works):

```bash
gem install kamal
kamal version
```

No Ruby? Run it dockerized with a shell alias (macOS shown; the image is `ghcr.io/basecamp/kamal:latest`):

```bash
alias kamal='docker run -it --rm -v "${PWD}:/workdir" \
  -v "/run/host-services/ssh-auth.sock:/run/host-services/ssh-auth.sock" \
  -e SSH_AUTH_SOCK="/run/host-services/ssh-auth.sock" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/basecamp/kamal:latest'
```

On Linux mount the agent socket via `${SSH_AUTH_SOCK}:/ssh-agent` and set `-e "SSH_AUTH_SOCK=/ssh-agent"` instead.

## kamal init

Run in the project root:

```bash
kamal init
```

This generates:

| File | Purpose |
|---|---|
| `config/deploy.yml` | Main configuration (heavily commented) |
| `.kamal/secrets` | Secret references; pulls from ENV / files / a manager — gitignore it |
| `.kamal/hooks/*.sample` | Sample lifecycle hook scripts; rename to activate |

Add `.kamal/secrets` to `.gitignore` immediately — it's the bridge to your real credentials (see secrets-and-env.md).

## Minimal config/deploy.yml

Strip the generated comments down to a working core:

```yaml
service: web
image: my-user/web

servers:
  web:
    - 192.168.0.1

proxy:
  ssl: true
  host: app.example.com
  # app_port: 3000   # only if your app isn't on port 80

registry:
  username: my-user
  password:
    - KAMAL_REGISTRY_PASSWORD

builder:
  arch: amd64        # match your servers; arm64 is the other option
```

Keep `service` hostname-safe (hyphens, not underscores). `arch` is required — `amd64` or `arm64`. Kamal 2 uses the `docker-container` build driver by default; it doesn't cache or build multiarch, so for a fast single-arch build set `builder.arch` to your server's architecture and build on a matching machine (Kamal docs). See configuration.md for env, volumes, accessories, and destinations.

Point the registry password at a secret entry:

```bash
# .kamal/secrets
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD   # read from your shell/CI env
# or: KAMAL_REGISTRY_PASSWORD=$(cat .registry-token)
```

Use a registry **access token**, never your account password.

## First Deploy

`kamal setup` is the one-time bootstrap; `kamal deploy` is every run after.

```bash
kamal setup    # installs Docker on hosts, boots accessories, then deploys
```

`setup` connects over SSH, installs Docker if missing, logs into the registry, builds and pushes the image, starts Kamal Proxy, boots the app, health-checks it, and switches traffic.

Once the server is bootstrapped, redeploy with:

```bash
kamal deploy                 # build, push, rolling zero-downtime switch
kamal deploy -d staging      # deploy a named destination (config/deploy.staging.yml)
kamal redeploy               # faster: skips bootstrap/registry-login steps
kamal app logs -f            # tail running logs
kamal app exec -i "bin/rails console"
```

Kamal builds from your committed git state — uncommitted changes are not deployed. If `setup` wedges, `kamal remove` tears the app down so you can rerun cleanly.

Next: harden the host (see server-provisioning.md); proxy/TLS (see kamal-proxy.md); accessories like Postgres/Redis (see accessories.md); when things break (see troubleshooting.md).

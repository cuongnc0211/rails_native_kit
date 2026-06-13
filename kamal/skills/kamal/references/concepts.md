# Kamal Concepts

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

How Kamal works and the vocabulary you need. Kamal deploys a containerized app to one or more servers over SSH, with zero-downtime switchover handled by Kamal Proxy — no Kubernetes, no orchestrator, no agents.

## The Push Model

You run Kamal on your laptop or in CI. It opens SSH connections to your servers and tells each one what to do: pull the image, run a container, switch traffic. The servers pull the image from a registry; Kamal never streams the image over SSH.

There is no control plane. Kamal holds no server-side state beyond what's running in Docker — it only knows what `config/deploy.yml` describes at the moment you run a command. Remove a server from the file and its containers keep running until you stop them by hand.

## Core Components

| Component | Role |
|---|---|
| SSHKit | Ruby library Kamal uses to run Docker commands on each host over SSH |
| Docker | Builds the image locally/in CI, runs containers on the servers |
| Registry | Stores images; servers pull from it (Docker Hub, GHCR, ECR, etc.) |
| Kamal Proxy | Per-host reverse proxy container; does gapless traffic switching and TLS |

Kamal 2 ships **Kamal Proxy** (the `basecamp/kamal-proxy` image), not Traefik. The proxy routes by host header, so several apps can share one server, each claiming its own domain (Kamal docs).

## Terminology

| Term | Meaning |
|---|---|
| service | App name; becomes the container-name prefix and image tag prefix |
| image | Registry path for the built image, e.g. `my-user/web` |
| server | A host (VM or bare metal) that runs containers |
| role | Named group of servers running the same command, e.g. `web`, `job` |
| primary role | The role health-checked and fronted by the proxy; `web` by default |
| accessory | Long-running side service (Postgres, Redis) with its own lifecycle (see accessories.md) |
| destination | A named environment variant, e.g. `staging` via `-d staging` |
| version | The git SHA Kamal tags the image with for each deploy |

## The Deploy Loop

Each `kamal deploy` runs roughly this sequence:

1. Acquire the deploy lock on the first host.
2. Run `pre-build`, build the image, push it to the registry.
3. Ensure Kamal Proxy is running on each web host.
4. Pull the new image and boot a new container alongside the old one.
5. Wait for the readiness check; the proxy switches traffic to the new container.
6. Stop the old container, prune old images.
7. Release the lock, run `post-deploy`.

## Zero-Downtime Switchover

Per host this is effectively blue-green: the old container keeps serving until the new one is healthy, then Kamal Proxy redirects traffic in one step — in-flight requests drain on the old one. Across the fleet it's a rolling restart; tune batch size and pacing with `boot` (see configuration.md):

```yaml
boot:
  limit: 25%   # restart at most a quarter of hosts at a time
  wait: 2      # seconds between batches
```

Asset bridging keeps both versions' fingerprinted assets on disk during the swap so in-flight requests don't 404 (set `asset_path`).

## Web Barrier

In a multi-role setup, non-primary roles (like `job`) won't boot until a `web` container passes its check. Only the primary role is health-checked through the proxy; other roles rely on Docker's container state / `HEALTHCHECK`.

## Deploy Locks

Before mutating a deploy, Kamal takes a lock on the first host so two deploys can't race. CI should expect this — a queued deploy waits or fails on a held lock.

```bash
kamal lock status
kamal lock acquire -m "Investigating an incident"
kamal lock release
```

Release a lock by hand only after confirming no deploy is in flight; forcing it during one corrupts state.

## Hooks

Drop executable scripts in `.kamal/hooks/` named for a lifecycle point. `kamal init` writes `.sample` versions you rename. Available hooks:

`docker-setup` · `pre-connect` · `pre-build` · `pre-deploy` · `post-deploy` · `pre-app-boot` · `post-app-boot` · `pre-proxy-reboot` · `post-proxy-reboot`

Common uses: `pre-build` for a clean-tree check, `pre-deploy` to run migrations, `post-deploy` for a Slack ping. Kamal injects context as env vars — `KAMAL_VERSION`, `KAMAL_HOSTS`, `KAMAL_COMMAND`, `KAMAL_DESTINATION`, `KAMAL_PERFORMER`, `KAMAL_SERVICE` and more — so hooks can act on the deploy in progress.

```bash
#!/bin/bash -e
# .kamal/hooks/pre-deploy — run migrations before the new version takes traffic
kamal app exec -p -q "bin/rails db:migrate"
```

Skip all hooks for a one-off run with `--skip-hooks`.

Next: install and first deploy (see getting-started.md); proxy/TLS detail (see kamal-proxy.md); server prep (see server-provisioning.md).

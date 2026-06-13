# Troubleshooting

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

Symptom → likely cause → fix, grouped by area. Debug in layers: app logs → proxy
logs → container state → host resources → audit trail
(see maintenance.md, kamal-proxy.md, secrets-and-env.md).

## Debugging commands

```bash
kamal app logs -f                 # follow app logs (start here)
kamal app logs -g "ERROR" -s 30m  # filter by pattern + time window
kamal proxy logs                  # proxy routing / TLS errors
kamal audit                       # recent Kamal commands run per server
kamal details                     # all containers (app + accessories + proxy)
kamal app version                 # version running on each host
kamal app containers -q           # running + old containers (rollback targets)

# on the host, via kamal server exec:
kamal server exec "docker ps -a"                       # all containers
kamal server exec "docker logs <id>"                   # exited container output
kamal server exec "docker inspect <id> --format '{{json .State.Health}}'"
kamal server exec "free -h && df -h"                   # memory + disk pressure
```

## Deploy fails / health check failing

| Symptom | Likely cause | Fix |
|---|---|---|
| Deploy aborts at health check | App boots but `/up` never returns 200 in time | `kamal app logs -s 10m` for the boot error; confirm the health-check path returns 200 without a redirect (see kamal-proxy.md) |
| Health check times out, app is slow to start | Default `deploy_timeout` too short, or OOM during the dual-container window | Increase `deploy_timeout`; check `kamal server exec "free -h"`, add swap |
| Assets 404 mid-deploy | `asset_path` not set, old/new containers serving mismatched fingerprints | Set `asset_path` in deploy.yml so both versions can serve assets during the switch |
| New container never appears | Build or push failed before deploy | See build failures below; check `kamal audit` |

## Registry auth errors

| Symptom | Likely cause | Fix |
|---|---|---|
| `denied` / `unauthorized` on push or pull | Registry password missing or wrong | Confirm `KAMAL_REGISTRY_PASSWORD` is set and `.kamal/secrets` resolves it (see secrets-and-env.md) |
| Works locally, fails in CI | Secret not passed to the runner | Add it as an Actions secret and into the job `env` (see ci-cd.md) |
| `--skip-push` deploy can't find image | Image lacks the `service=<name>` label | Add `labels: service=<name>` to the external build step (see ci-cd.md) |

## Proxy 502 / 503

| Symptom | Likely cause | Fix |
|---|---|---|
| 502 Bad Gateway | App container not listening on the expected port, or crashed | `kamal app logs -f`; confirm the app listens on the proxy's `app_port` (see kamal-proxy.md) |
| 503 Service Unavailable | No healthy target — deploy in progress, or all containers down | `kamal details`; `kamal proxy logs`; redeploy or `kamal app boot` |
| Routes to the wrong app | Stale proxy registration | `kamal proxy logs`; as a last resort `kamal proxy reboot` |
| TLS / cert errors | Let's Encrypt challenge failing or wrong host | `kamal proxy logs`; verify the `host` and that ports 80/443 are reachable |

## Container won't boot

| Symptom | Likely cause | Fix |
|---|---|---|
| Container exits immediately after deploy | Missing env var or secret | `kamal server exec "docker logs <id>"` (works on exited containers); confirm every `env.secret` key is in `.kamal/secrets` |
| `KeyError` / nil config at boot | Secret referenced in deploy.yml but not provided | Check `.kamal/secrets` resolves all referenced names (see secrets-and-env.md) |
| Boots locally, dies on server | Image arch mismatch | Set `builder.arch: amd64` (or match the host) |

## SSH / connection issues

| Symptom | Likely cause | Fix |
|---|---|---|
| `Connection refused` / timeout to host | Wrong IP, SSH port, or firewall | Test `ssh app@<host>` manually; check the `ssh` block in deploy.yml |
| `Permission denied (publickey)` | Key not authorized on the server | Add the public key to the server's `authorized_keys`; in CI load it via ssh-agent (see ci-cd.md) |
| Docker permission errors over SSH | Deploy user not in the `docker` group | `usermod -aG docker <user>` on the host |

## Build failures

| Symptom | Likely cause | Fix |
|---|---|---|
| `cache export not supported` | Buildx not set up | Add `docker/setup-buildx-action` before deploy (see ci-cd.md) |
| Rebuilds everything every push | No build cache configured | Add `builder.cache.type: registry` (see ci-cd.md) |
| `exec format error` at runtime | Built for wrong CPU arch | Set `builder.arch` to match the server |

## Lock stuck

| Symptom | Likely cause | Fix |
|---|---|---|
| Deploy refuses to start: "locked" | Previous deploy crashed holding the lock | `kamal lock status` to inspect; check `kamal audit`; then `kamal lock release` |
| Lock reappears immediately | Another deploy (CI) is genuinely running | Wait for it; don't release a live lock (see maintenance.md) |

## Debugging a failed deploy, step by step

```bash
kamal app containers -q                       # 1. find the failed container's SHA
kamal server exec "docker ps -a | grep <sha>" # 2. locate it (may have exited)
kamal server exec "docker logs <id>"          # 3. read its logs (works post-exit)
kamal server exec "docker inspect <id> --format '{{json .State.Health}}'"  # 4. health
kamal server exec "free -h && df -h"          # 5. rule out OOM / disk full
kamal audit                                   # 6. confirm command order/timing
```

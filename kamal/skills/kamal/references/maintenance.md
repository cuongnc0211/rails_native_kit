# Maintenance

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

Day-2 operations: rolling back, holding deploy locks, running one-off commands,
reading logs, pruning old artifacts, and opening a Rails console.

## Rollback

`kamal rollback` re-runs the standard rolling deploy against an already-pushed
image, so it is zero-downtime as long as the schema is still compatible.

```bash
kamal app containers -q          # list running + old container versions (SHAs)
kamal rollback <SHA>             # re-deploy a previous version (full tag also works)
```

Retention: old containers from previous deploys are kept on the host as rollback
targets, then cleaned up after ~3 days the next time you `kamal deploy`. `kamal
prune` keeps the last 5 deployed containers and images (see Prune below). Don't
wait indefinitely to decide on a rollback.

If a rollback also needs a schema change, hold a lock and migrate first:

```bash
kamal lock acquire -m "rollback with db migration"
kamal app exec -p --reuse "bin/rails db:rollback STEP=1"
kamal rollback <SHA>
kamal lock release
```

## Deploy locks

Unsafe-to-run-concurrently commands take an atomic lock (a directory under
`.kamal/` on the primary host). Acquire one manually to protect a maintenance
window, or release one left stale by a crashed deploy.

```bash
kamal lock status                       # holder, timestamp, version, message
kamal lock acquire -m "debugging issue" # -m/--message is required
kamal lock release
```

A stuck lock after a failed deploy is the usual cause of "deploy won't start"
(see troubleshooting.md) — check `kamal audit` before releasing.

## Running commands

`kamal app exec` runs a command in the app image. By default it runs on every
server in a fresh container — use `-p`/`--primary` for single-run tasks and
`--reuse` to run inside the already-running container.

```bash
kamal app exec "ps aux"                       # all servers, new container
kamal app exec -p "bin/rails about"           # primary only
kamal app exec -p --reuse "bin/rails db:migrate"  # inside running container
kamal app exec -i "bin/rails console"         # -i/--interactive (TTY)
kamal app exec -i --reuse bash                # interactive shell in running container
```

`kamal server exec` runs on the host itself, outside any container:

```bash
kamal server exec "uptime"
kamal server exec "apt-get update && apt-get upgrade -y"   # host security updates
```

`kamal accessory exec` targets an accessory container (see accessories.md and
backups.md):

```bash
kamal accessory exec postgres -i --reuse "psql -U app app_production"
```

## Console & dbconsole

`kamal app exec -i --reuse "bin/rails console"` (and `bin/rails dbconsole`) open
interactive sessions in the running container. Turn them into aliases (below) so
they become `kamal console` / `kamal dbc`.

## Logs

`kamal app logs` reads container logs. Same flags apply to
`kamal accessory logs <name>`.

```bash
kamal app logs                       # recent lines, all app servers
kamal app logs -f                    # -f/--follow (tail)
kamal app logs -n 200                # -n/--lines: last N lines
kamal app logs -g "ERROR"            # -g/--grep: filter
kamal app logs -g "<request-id>" -s 1h   # -s/--since: time window
kamal app logs -r web                # -r/--roles
kamal app logs -h 192.168.0.1        # -h/--hosts
```

`kamal app logs -g "<request-id>" -s 1h` traces one request during an incident.

## Log driver configuration

Kamal passes `logging` straight through to Docker's `--log-driver` /`--log-opt`.
Set it root-level or per-role. Configure rotation so a chatty service can't fill
the disk:

```yaml
# config/deploy.yml
logging:
  driver: json-file
  options:
    max-size: 100m     # size per log file before rotation
    max-file: 5        # number of rotated files kept
```

A per-role `logging:` block (nested under a role in `servers:`) is merged on top
of the root config. Other valid drivers: `local`, `journald`, `awslogs`,
`gcplogs`, `none`.

## Aliases

Define reusable shortcuts under `aliases` in deploy.yml. Names allow lowercase
letters, numbers, dashes, underscores. Invoke as `kamal <alias>`.

```yaml
# config/deploy.yml
aliases:
  console: app exec -i --reuse "bin/rails console"
  dbc:     app exec -i --reuse "bin/rails dbconsole"
  shell:   app exec -i --reuse "bash"
  logs:    app logs -f
```

Then run `kamal console`, `kamal shell`, `kamal logs`.

## Prune

`kamal prune` reclaims disk by removing old containers and images beyond the
retention window (last 5 deployed containers + their images by default).

```bash
kamal prune all          # unused images and stopped containers
kamal prune containers   # stopped containers except the last N (default 5)
kamal prune images       # unused images
```

`kamal deploy` prunes automatically, so manual pruning is mainly for recovering
disk between deploys (see troubleshooting.md for "disk full").

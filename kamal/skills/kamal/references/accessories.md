# Accessories

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

Accessories are long-running auxiliary services — databases (PostgreSQL, MySQL),
caches/queues (Redis), search, etc. — that run as Docker containers alongside
your app but with their own lifecycle. Kamal does not build accessory images;
it pulls public images. Accessories are not touched by `kamal deploy`; you
manage them with `kamal accessory` commands.

## The `accessories:` block

```yaml
# config/deploy.yml
accessories:
  postgres:
    image: postgres:17
    host: db.example.com          # one host; or hosts:/roles:/tags: (below)
    port: "127.0.0.1:5432:5432"   # bind to localhost only — see Networking
    env:
      clear:
        POSTGRES_USER: demo
        POSTGRES_DB: demo_production
      secret:
        - POSTGRES_PASSWORD       # pulled from .kamal/secrets (see secrets-and-env.md)
    files:
      - config/postgres/production.conf:/etc/postgresql/postgresql.conf
      - db/init.sql:/docker-entrypoint-initdb.d/setup.sql
    directories:
      - data:/var/lib/postgresql/data   # persisted (see volumes-and-data.md)

  redis:
    image: redis:7
    roles:
      - web                       # deploy to every server in the web role
    port: "127.0.0.1:6379:6379"
    cmd: redis-server --appendonly yes
    directories:
      - data:/data
```

## Targeting hosts

| Key | Targets |
|---|---|
| `host` | a single server |
| `hosts` | a list of servers |
| `roles` | every server in the named role(s) |
| `tags` | servers matching tag(s) |

## Supported keys

`image`, `host`/`hosts`/`roles`/`tags`, `port`, `cmd`, `env`, `files`,
`directories`, `volumes`, `options` (raw `docker run` args), `labels`,
`network` (defaults to `kamal`), `registry`, and `proxy`.

## `kamal accessory` commands

```bash
kamal accessory boot postgres      # create + start the accessory
kamal accessory reboot postgres    # stop, remove, recreate (e.g. image bump)
kamal accessory restart postgres   # restart the existing container
kamal accessory stop postgres      # stop without removing
kamal accessory start postgres     # start a stopped accessory
kamal accessory details postgres   # show container info
kamal accessory logs postgres -f   # stream logs
kamal accessory exec postgres "psql -U demo demo_production" --interactive
kamal accessory remove postgres    # stop + remove the container
# Omit the name (e.g. `kamal accessory boot`) to act on all accessories.
```

## Docker networking between app and accessory

Accessories and the app join the `kamal` Docker network. The accessory is
reachable by the hostname `<service>-<accessory>` — e.g. an app named `demo`
with a `postgres` accessory resolves at `demo-postgres`. Reference it in
connection URLs:

```bash
DATABASE_URL=postgres://demo:PASSWORD@demo-postgres/demo_production
REDIS_URL=redis://demo-redis:6379/0
```

When you publish a `port`, bind it to `127.0.0.1` (e.g. `"127.0.0.1:5432:5432"`).
A bare `"5432:5432"` is published on all interfaces and bypasses UFW, exposing
the database to the internet.

## Scheduled / cron jobs

Kamal has no separate cron accessory. The official approach is a dedicated
`servers:` role that runs `cron` in the foreground using the app image, so all
app code and tasks are available:

```yaml
# config/deploy.yml
servers:
  cron:
    hosts:
      - 192.168.0.1
    cmd: bash -c "(env && cat config/crontab) | crontab - && cron -f"
```

```cron
# config/crontab (baked into the image)
0 0 * * * cd /rails && bin/rails db:cleanup >> /proc/1/fd/1 2>&1
```

The `(env && cat config/crontab)` prefix copies the container's environment
into the crontab, since cron does not inherit it. Run only one cron role
host to avoid duplicate job runs. A job-queue scheduler (e.g. Solid Queue
recurring tasks, Sidekiq-cron) can replace the cron role entirely.

## Gotchas

- `kamal deploy` does not update accessories — bump the image and run
  `kamal accessory reboot <name>`.
- Moving an accessory to a new host does not remove the old container; remove
  it first (`kamal accessory remove`).
- A stateful accessory without a `directories`/`volumes` mount loses all data
  on reboot (see volumes-and-data.md).
- Back up accessory databases on a schedule (see backups.md).

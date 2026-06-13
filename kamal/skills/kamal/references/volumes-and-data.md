# Volumes and Persisting Data

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

Container filesystems are ephemeral: every deploy replaces the container, so
anything written inside it — uploaded files, SQLite databases, generated
assets — is gone on the next deploy. To survive deploys, data must live on the
host (or a Docker-managed volume) and be mounted into the container. Kamal
gives you three directives for this: `volumes`, `directories`, and `files`.

## `volumes:` — named volumes and bind mounts

The top-level `volumes:` key mounts storage into the app container. Each entry
is a `host:container` (or `name:container`) string:

```yaml
# config/deploy.yml
volumes:
  - "demo_storage:/rails/storage"        # named Docker volume (auto-created)
  - "/var/lib/demo/uploads:/rails/uploads"  # bind mount (host path must exist)
```

| Form | Meaning | Best for |
|---|---|---|
| `"name:/path"` | named Docker volume, created on first use, lives under `/var/lib/docker/volumes/` | DB data, portable storage |
| `"/host/path:/path"` | bind mount to an absolute host path that must already exist | block storage, pre-provisioned dirs |

Named volumes are managed by Docker and are easier to move/back up. Bind mounts
give you a known host path (useful when attaching cloud block storage). For
`volumes:`, the host side must be a volume name or an **absolute** path —
relative paths are not allowed here.

## `directories:` — auto-created bind mounts

`directories:` works like a bind mount but Kamal creates the host directory if
it is missing. A relative path is scoped under the service directory
(`/home/<user>/.kamal/<service>/...`); an absolute path is used as-is. This is
the usual choice for app storage and accessory data:

```yaml
# app-level
directories:
  - storage:/rails/storage            # relative → auto-created under service dir

# accessory-level (see accessories.md)
accessories:
  postgres:
    directories:
      - data:/var/lib/postgresql/data
```

## `files:` — single-file mounts

`files:` mounts one file from the repo into the container — config files, init
scripts, certificates. The host file is read at deploy time and pushed to the
server:

```yaml
accessories:
  postgres:
    files:
      - config/postgres/production.conf:/etc/postgresql/postgresql.conf
      - db/init.sql:/docker-entrypoint-initdb.d/setup.sql
```

## Where common data should live

| Data | Directive | Mount example |
|---|---|---|
| Active Storage / uploads | `directories` or `volumes` | `storage:/rails/storage` |
| SQLite database | `volumes` (named) | `demo_db:/rails/db` |
| Postgres/MySQL data | accessory `directories` | `data:/var/lib/postgresql/data` |
| DB config / init SQL | `files` | `config/...:/etc/...` |

SQLite and uploads break most often: if `db/production.sqlite3` or
`storage/` is not mounted out of the container, it is recreated empty on every
deploy and all data is lost.

## Permissions

The app container runs as a non-root user (UID/GID `1000` in the default Rails
image). Auto-created `directories` are owned correctly, but a pre-existing bind
mount may need ownership fixed so the container can write:

```bash
sudo mkdir -p /var/lib/demo/uploads
sudo chown -R 1000:1000 /var/lib/demo/uploads
```

## Gotchas

- Container FS is ephemeral — never trust in-container paths to persist across
  deploys. Mount everything you need to keep.
- Relative paths only work in `directories:`, not `volumes:`.
- Back up named volumes and bind mounts with the service stopped, or use the
  database's own dump tooling to avoid copy-on-write corruption (see backups.md).
- A stateful accessory with no mount loses its data on `kamal accessory reboot`
  (see accessories.md).

See maintenance.md for migrating data between hosts.

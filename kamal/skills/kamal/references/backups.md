# Backups

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

Kamal deploys containers; it does **not** back up your data. Persistent data lives
in accessory databases and named volumes, and protecting it is your job. Three
things to back up: database contents, named volumes, and a copy held off-server
(see accessories.md, maintenance.md).

## Database dumps

Run the database's own dump tool inside the accessory container with
`kamal accessory exec <name> --reuse`, and redirect stdout to a local file. Use
`--reuse` to run in the live container (not a throwaway one) and `-i` so output
streams back.

PostgreSQL:

```bash
kamal accessory exec postgres --reuse -i \
  "pg_dump -U app -d app_production" > app_production.sql
```

MySQL / MariaDB:

```bash
kamal accessory exec mysql --reuse -i \
  "mysqldump -u root -p\"$MYSQL_ROOT_PASSWORD\" app_production" > app_production.sql
```

Notes:

- `pg_dump`/`mysqldump` are the right tools — they take a consistent snapshot even
  while the database serves traffic. Never tar a live database's data directory.
- The dump runs inside the accessory container, so the client binary and the
  database version always match.
- Quote the inner command so the redirect (`>`) applies to your local shell, not
  the remote one.

Redis persists differently — enable AOF in the accessory `cmd`
(`redis-server --appendonly yes`) and back up the `appendonly.aof` file from its
volume using the volume method below.

## Named volume backups

Volumes (uploads, `storage/`, certs) aren't in a database dump. Archive them with
`tar` from a throwaway container that mounts the same volumes, then pull the
archive off-server.

```bash
# on the host (via kamal server exec or SSH)
docker run --rm \
  --volumes-from demo-web \
  -v "$PWD/backup:/backup" \
  ubuntu \
  tar -czf /backup/storage.tar.gz -C /rails/storage .
```

## Copying off-server

A backup on the same VM dies with the VM. Pull it down or push it to object
storage.

```bash
scp app@192.168.0.1:/backup/storage.tar.gz ./backups/
scp app@192.168.0.1:~/app_production.sql ./backups/

# or straight to S3-compatible storage
aws s3 cp ./backups/app_production.sql s3://demo-backups/$(date +%F)/
```

## Automated scheduled backups

For unattended, encrypted, off-site backups, run a backup tool as its own
accessory that uploads to S3 on a schedule:

```yaml
# config/deploy.yml
accessories:
  pg-backup:
    image: eeshugerman/postgres-backup-s3:16
    host: 192.168.0.1
    env:
      clear:
        SCHEDULE: "@daily"
        BACKUP_KEEP_DAYS: "14"
        S3_REGION: us-east-1
        S3_BUCKET: demo-backups
        POSTGRES_HOST: demo-postgres
        POSTGRES_DATABASE: app_production
        POSTGRES_USER: app
      secret:
        - POSTGRES_PASSWORD
        - PASSPHRASE          # GPG encryption key for backups at rest
```

```bash
kamal accessory boot pg-backup
```

## Restore

Always test restores — an untested backup is not a backup.

PostgreSQL:

```bash
cat app_production.sql | \
  kamal accessory exec postgres --reuse -i \
  "psql --set ON_ERROR_STOP=on -U app -d app_production"
```

MySQL:

```bash
cat app_production.sql | \
  kamal accessory exec mysql --reuse -i \
  "mysql -u root -p\"$MYSQL_ROOT_PASSWORD\" app_production"
```

Volume restore (extract into the mounted volume):

```bash
docker run --rm \
  --volumes-from demo-web \
  -v "$PWD/backup:/backup" \
  ubuntu \
  bash -c "cd /rails/storage && tar -xzf /backup/storage.tar.gz"
```

Combine logical dumps (point-in-time, portable) with full VM snapshots from your
cloud provider so you also capture config, certs, and filesystem state.

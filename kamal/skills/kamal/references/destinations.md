# Destinations

Destinations let one repo deploy to multiple environments — typically staging and production — by layering a small per-destination config file over the shared base.

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

## How it works

The base `config/deploy.yml` holds everything common. A destination file `config/deploy.<destination>.yml` holds only what differs, and is **merged over** the base when you pass `-d <destination>`.

```
config/
├── deploy.yml                 # shared base
├── deploy.staging.yml         # staging overrides
└── deploy.production.yml      # production overrides
```

The `-d` (`--destination`) flag selects the file: `-d staging` loads `deploy.staging.yml`. Without `-d`, only `deploy.yml` is used.

## Base config

Keep the registry, image, builder, and shared env here.

```yaml
# config/deploy.yml
service: demo
image: dockeruser/demo
registry:
  username:
    - DOCKER_USERNAME
  password:
    - KAMAL_REGISTRY_PASSWORD
builder:
  arch: amd64
  cache:
    type: registry
    image: dockeruser/demo:build-cache
    options: mode=max
env:
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
```

## Destination overrides

Specify only the diffs — usually hosts and the proxy hostname.

```yaml
# config/deploy.staging.yml
servers:
  web:
    hosts:
      - 192.168.0.10
proxy:
  host: staging.example.com
  ssl: true
```

```yaml
# config/deploy.production.yml
servers:
  web:
    hosts:
      - 192.168.0.1
      - 192.168.0.2
proxy:
  host: app.example.com
  ssl: true
```

## Merge behavior

The destination file is deep-merged on top of the base. Maps merge key by key; a scalar or list in the destination replaces the base value at that key. So `proxy.host` above adds to the inherited proxy block, while a `servers.web.hosts` list fully replaces any base hosts. Define shared structure once in the base and override the leaves per destination.

## Registry and host differences

Because the destination file can set any key, a destination can target a different registry, build arch, or builder host — useful when staging and production live in separate clouds:

```yaml
# config/deploy.production.yml
registry:
  server: <id>.dkr.ecr.us-east-1.amazonaws.com
  username: AWS
  password:
    - AWS_ECR_PASSWORD
builder:
  remote: ssh://docker@192.168.0.20
```

## Per-destination secrets

Each destination resolves secrets from `.kamal/secrets.<destination>`, with values shared across all destinations placed in `.kamal/secrets-common`. Both stay out of git (see secrets-and-env.md).

```bash
# .kamal/secrets-common  — shared by every destination
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD

# .kamal/secrets.staging
DATABASE_URL=$STAGING_DATABASE_URL
RAILS_MASTER_KEY=$(cat config/credentials/staging.key)

# .kamal/secrets.production
DATABASE_URL=$PRODUCTION_DATABASE_URL
RAILS_MASTER_KEY=$(cat config/credentials/production.key)
```

## Commands

Pass `-d` to every command for a non-default destination — it is not remembered between invocations.

```bash
kamal setup -d staging          # first-time provision + deploy (see getting-started.md)
kamal deploy -d staging
kamal deploy -d production
kamal app logs -d production
kamal rollback -d production
```

In hooks, the selected destination is exposed as the `KAMAL_DESTINATION` environment variable, so a single hook script can branch per environment (see ci-cd.md).

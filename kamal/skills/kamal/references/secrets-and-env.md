# Secrets and Environment Variables

Kamal separates environment variables into non-secret values that live in config and secret values that are resolved at deploy time from a local, uncommitted file.

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

## clear vs secret

```yaml
env:
  clear:
    RAILS_ENV: production
    DB_HOST: 192.168.0.2
    RAILS_LOG_TO_STDOUT: "true"
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
    - REDIS_URL
```

| | `env.clear` | `env.secret` |
|---|---|---|
| Form | key → value map | list of variable names |
| Where the value lives | inline in `deploy.yml` | resolved from `.kamal/secrets` |
| Visibility | appears in `kamal config` / logs | redacted from output |
| Delivery to container | passed directly to `docker run` | written to a host env file, then loaded |

Because secret values are written to an env file on the host (not embedded in the `docker run` invocation), they don't leak into process listings or deploy logs.

## The .kamal/secrets file

A dotenv-format file loaded locally during deploy. It is **never committed** and **never copied to the server** — only the resolved values for keys listed under `env.secret` are sent. Add it to `.gitignore`:

```
.kamal/secrets
.kamal/secrets.*
```

Each line can be a literal, a variable reference, or a shell command substitution:

```bash
# .kamal/secrets

# Variable reference (value comes from the shell / CI environment)
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD

# Command substitution — read from a file
RAILS_MASTER_KEY=$(cat config/master.key)

# Compose a value from another command
DATABASE_URL="postgres://demo:$(cat tmp/db_pw)@192.168.0.2/demo_production"
```

Only names that also appear in `env.secret` (or `registry.password`, `builder.secrets`) are exported to the deploy. Listing a name in `.kamal/secrets` alone does nothing.

## Secret manager adapters

`kamal secrets fetch` pulls one or more secrets from a manager in a single call; `kamal secrets extract` picks one value out of that result. This keeps round-trips (and biometric prompts) to one per deploy.

```bash
# Fetch several secrets at once, then extract each
SECRETS=$(kamal secrets fetch --adapter 1password \
  --account my-team --from Vault/Item \
  KAMAL_REGISTRY_PASSWORD RAILS_MASTER_KEY)

KAMAL_REGISTRY_PASSWORD=$(kamal secrets extract KAMAL_REGISTRY_PASSWORD $SECRETS)
RAILS_MASTER_KEY=$(kamal secrets extract RAILS_MASTER_KEY $SECRETS)
```

Supported `--adapter` values: `1password`, `lastpass`, `bitwarden`, `bitwarden-sm`, `aws_secrets_manager`, `doppler`, `gcp`, `passbolt`.

| Subcommand | Purpose |
|---|---|
| `kamal secrets fetch` | retrieve secrets from a manager (returns JSON) |
| `kamal secrets extract` | pull a single value out of fetch output |
| `kamal secrets print` | print resolved secrets — debugging only |

Any manager with a CLI works without an adapter via plain command substitution, e.g. `FOO=$(op read op://Vault/Item/field)`.

## How secrets reach containers

1. `kamal deploy` loads `.kamal/secrets` locally and resolves every referenced name.
2. Values for `env.secret` keys are written to a host env file under the service's app directory.
3. The container is started with that env file loaded, so the variables appear in the process environment.
4. Changing a secret value and re-running `kamal deploy` re-pushes the env file and restarts the containers — no separate command needed.

## Tags: grouping env per role or host

Tag hosts (see configuration.md), then attach env to each tag. Tagged env applies only to hosts carrying that tag and merges over the base `env`.

```yaml
servers:
  worker:
    hosts:
      - 192.168.0.3: fast
      - 192.168.0.4: [ recurring, batch ]
    cmd: bundle exec sidekiq

env:
  clear:
    QUEUE: default
  tags:
    fast:
      QUEUE: fast            # clear value shorthand
    recurring:
      clear:
        QUEUE: recurring
      secret:
        - RECURRING_API_KEY  # tag-scoped secret
```

Tags are only honored under the top-level `env`, not inside a role block.

See destinations.md for per-environment secrets files (`.kamal/secrets.<destination>` and the shared `.kamal/secrets-common`).

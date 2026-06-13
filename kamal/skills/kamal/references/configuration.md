# Configuration

The `config/deploy.yml` file is the single source of truth for a Kamal deployment. This is the anatomy of its top-level keys.

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

## Required keys

```yaml
service: demo                 # names containers, networks, env files on the host
image: dockeruser/demo        # repository the image is pushed to / pulled from
servers:
  - 192.168.0.1               # shorthand: all hosts in the implicit `web` role
```

`service` must be stable — renaming it orphans the running containers.

## Servers and roles

A role is a named group of hosts running the same image with a shared startup command. The `web` role is special: it receives traffic from kamal-proxy (see kamal-proxy.md). Other roles run unexposed (workers, schedulers, custom APIs).

```yaml
servers:
  web:
    hosts:
      - 192.168.0.1
      - 192.168.0.2
    options:                  # passed through to `docker run`
      cpus: 2
      memory: 2g
    labels:
      app: demo
  worker:
    hosts:
      - 192.168.0.3
    cmd: bundle exec sidekiq  # overrides the image CMD for this role
    proxy: false              # non-web roles default to no proxy anyway
    logging:
      driver: json-file
      options:
        max-size: 10m
```

Kamal runs one container per role per host — scale by adding hosts or processes inside the image, not replicas.

## Host tags

Tag hosts to attach role/host-specific env (see secrets-and-env.md):

```yaml
servers:
  worker:
    hosts:
      - 192.168.0.3: fast
      - 192.168.0.4: [ recurring, batch ]
    cmd: bundle exec sidekiq
```

## Registry

`username`/`password` accept literals or single-item lists referencing secrets from `.kamal/secrets`.

```yaml
registry:
  server: registry.example.com    # omit for Docker Hub
  username:
    - DOCKER_USERNAME
  password:
    - KAMAL_REGISTRY_PASSWORD
```

| Provider | server | username |
|---|---|---|
| Docker Hub | _(omit)_ | your user |
| GHCR | `ghcr.io` | GitHub user |
| AWS ECR | `<id>.dkr.ecr.<region>.amazonaws.com` | `AWS` |
| GCP Artifact Registry | `<region>-docker.pkg.dev` | `_json_key_base64` |

Validate with `kamal registry login`.

## Builder

Controls image construction. `arch` takes a single value or a list for multi-arch.

```yaml
builder:
  arch: amd64                 # or [ amd64, arm64 ]
  context: .                  # build context (default: .)
  dockerfile: Dockerfile      # alternate path if non-standard
  target: production          # multi-stage build target
  args:                       # build-time ARGs (non-secret)
    RUBY_VERSION: "3.3.6"
  secrets:                    # build-time secrets, sourced from .kamal/secrets
    - GITHUB_TOKEN
  cache:
    type: registry            # registry | gha
    image: dockeruser/demo:build-cache
    options: mode=max
  remote: ssh://docker@192.168.0.10   # build on a remote Docker host
  local: true                 # also keep a local builder for matching arch
```

Build secrets are consumed in the Dockerfile via mounts:

```dockerfile
RUN --mount=type=secret,id=GITHUB_TOKEN \
    bundle install
```

| Cache `type` | Stored in | Best for |
|---|---|---|
| `registry` | a separate image tag | CI/CD, remote builders |
| `gha` | GitHub Actions cache | GitHub Actions runners only (see ci-cd.md) |

## Environment, labels, SSH

`env` splits into non-secret (`clear`) and secret values — see secrets-and-env.md for the full mechanism.

```yaml
env:
  clear:
    RAILS_LOG_TO_STDOUT: "true"
  secret:
    - RAILS_MASTER_KEY

labels:
  service: demo               # Docker labels for log filtering / scraping

ssh:
  user: deploy                # non-root deploy user
  port: 22
  proxy: deploy@bastion.example.com   # jump host
  # proxy_command: aws ssm start-session ...   # custom tunnel (e.g. SSM)
  keys: [ ~/.ssh/id_demo ]
  keys_only: true             # ignore ~/.ssh/config, use `keys` only
```

## Asset bridging

During a deploy old and new containers run briefly side by side. If the app serves fingerprinted assets directly, in-flight requests can 404 against the wrong version. `asset_path` makes Kamal extract both versions' assets into a shared host directory and bind-mount the merged set into both containers.

```yaml
asset_path: /rails/public/assets   # path inside the image where assets live
```

Filenames must be content-hashed (Rails/Propshaft and Webpack do this by default) so old names stay valid. To serve static files from the app, set `RAILS_SERVE_STATIC_FILES=true` in `env.clear`. A CDN or Thruster is preferable to serving assets through the app container.

See destinations.md to layer per-environment overrides on top of this base file.

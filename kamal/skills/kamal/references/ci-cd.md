# CI/CD

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

Deploying from CI (GitHub Actions): SSH key access to servers, registry auth via
secrets, a build cache so each push doesn't rebuild from scratch, and an optional
build-outside-Kamal pattern with `--skip-push` (see secrets-and-env.md,
configuration.md).

## What CI needs

To run `kamal deploy` from a runner you must provide:

1. An SSH private key the runner uses to reach your servers.
2. Registry credentials so the runner can push the built image.
3. App secrets (e.g. `RAILS_MASTER_KEY`) that `.kamal/secrets` reads from env.

Store all of these as GitHub Actions repository secrets, never in the repo.

## SSH key setup

The simplest path uses the `webfactory/ssh-agent` action to load the key into an
agent for the job:

```yaml
- uses: webfactory/ssh-agent@v0.9.0
  with:
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```

Generate a dedicated deploy key locally and add the public half to each server's
`~/.ssh/authorized_keys`; put the private half in `SSH_PRIVATE_KEY`.

## Registry auth via secrets

In deploy.yml the registry password references an env var name (see
secrets-and-env.md):

```yaml
# config/deploy.yml
registry:
  username: my-registry-user
  password:
    - KAMAL_REGISTRY_PASSWORD
```

`.kamal/secrets` resolves it from the runner's environment:

```bash
# .kamal/secrets
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD
RAILS_MASTER_KEY=$RAILS_MASTER_KEY
```

## Build cache (do this from day one)

A registry-backed build cache avoids a cold rebuild on every push. Configure the
builder once and set up Buildx in the workflow:

```yaml
# config/deploy.yml
builder:
  arch: amd64
  cache:
    type: registry
    image: my-registry-user/demo:build-cache
    options: mode=max
```

Add `docker/setup-buildx-action` to the job before deploying, otherwise BuildKit
cache export fails.

## GitHub Actions workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    env:
      DOCKER_BUILDKIT: 1
    steps:
      - uses: actions/checkout@v4

      - name: Set up Buildx
        uses: docker/setup-buildx-action@v3

      - name: Load SSH key
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true

      - name: Deploy
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.KAMAL_REGISTRY_PASSWORD }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
        run: bundle exec kamal deploy --version=${{ github.sha }}
```

`--version=${{ github.sha }}` pins the deploy to the exact commit, so the running
container tag matches the source you tested.

## Build-outside-Kamal with --skip-push

To decouple the image build from Kamal — e.g. reuse GitHub Actions cache
(`type=gha`) or a shared org build step — build and push the image yourself, then
have Kamal pull and run it with `--skip-push`. The image **must** carry the
`service=<your-service>` label, or Kamal won't recognise it.

```yaml
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: my-registry-user/demo:${{ github.sha }}
          labels: service=demo          # required — Kamal matches on this
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy without rebuilding
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.KAMAL_REGISTRY_PASSWORD }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
        run: bundle exec kamal deploy --skip-push --version=${{ github.sha }}
```

`--skip-push` tells Kamal the image is already in the registry: it skips the local
build/push and goes straight to pulling and the rolling restart.

## Multiple environments

Use destinations for staging vs production (see configuration.md):

```bash
kamal deploy -d staging --version=${{ github.sha }}
```

Tip: keep the deploy job gated on a green test job (`needs: test`) so a failing
build never reaches your servers.

# Kamal Proxy

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

Kamal Proxy is the lightweight reverse proxy that ships with Kamal 2, replacing
Traefik from Kamal 1. It runs as a single Docker container per server, handles
zero-downtime traffic cutover, terminates TLS (auto Let's Encrypt), and buffers
requests/responses. You configure it entirely from the `proxy:` block in
`deploy.yml` — there are no proxy config files (see configuration.md).

## The `proxy:` block

```yaml
# config/deploy.yml
proxy:
  host: demo.example.com        # single host the proxy routes to this app
  # hosts:                      # or multiple
  #   - demo.example.com
  #   - www.example.com
  app_port: 3000                # container port the app listens on (default: 80)
  ssl: true                     # auto Let's Encrypt TLS (default: false)
  healthcheck:
    path: /up                   # default: /up
    interval: 1                 # seconds between probes (default: 1)
    timeout: 5                  # per-probe timeout in seconds (default: 5)
  response_timeout: 30          # seconds to wait for the app (default: 30)
  buffering:
    requests: true              # default: true
    responses: true             # default: true
    max_request_body: 40_000_000  # bytes; default: 1GB
    max_response_body: 0          # 0 = unlimited (default)
    memory: 1_000_000             # bytes kept in memory before disk spill (default: 1MB)
  forward_headers: true         # forward X-Forwarded-*; default false when ssl:true, else true
```

## Key settings

| Key | Default | When to change |
|---|---|---|
| `app_port` | `80` | App listens on 3000/8080/etc. |
| `ssl` | `false` | Enable HTTPS via Let's Encrypt (needs `host` + DNS pointing at the server) |
| `ssl_redirect` | `true` | Set `false` to stop forcing HTTP→HTTPS |
| `healthcheck.path` | `/up` | App exposes a different readiness endpoint |
| `healthcheck.interval` | `1` | Reduce probe frequency |
| `healthcheck.timeout` | `5` | Slow health endpoint |
| `response_timeout` | `30` | Long-running requests (reports, exports) |
| `buffering.max_request_body` | `1GB` | Cap upload size |
| `forward_headers` | conditional | Behind another upstream proxy/CDN |

`deploy_timeout` and `drain_timeout` are top-level keys (not under `proxy:`)
that govern cutover timing:

```yaml
deploy_timeout: 30   # max seconds for new containers to pass healthcheck (default: 30)
drain_timeout: 30    # seconds to let in-flight requests finish (default: 30)
```

## TLS and Let's Encrypt

`ssl: true` provisions and renews a certificate automatically. Requirements:
a public `host`, port 443 reachable, and the app trusting the proxy's TLS
termination. In Rails:

```ruby
# config/environments/production.rb
config.force_ssl  = true
config.assume_ssl = true   # the proxy terminates TLS; app sees plain HTTP
```

Exclude the health check path from the SSL redirect, or the probe gets a 301
and the deploy fails:

```ruby
config.ssl_options = { redirect: { exclude: ->(r) { r.path == "/up" } } }
```

## Zero-downtime cutover

On each deploy the proxy keeps routing to the running container while the new
one boots. It probes the new container's `healthcheck.path` once per `interval`
until it returns `200` (or `deploy_timeout` elapses and the deploy aborts).
Once healthy, all new traffic is switched to the new container, then the proxy
drains in-flight requests from the old one (up to `drain_timeout`) before it is
stopped. No connections are dropped.

## `kamal proxy` commands

```bash
kamal proxy boot            # install/start the proxy container on all servers
kamal proxy reboot          # stop, remove, and start a fresh proxy container
kamal proxy reboot --rolling # one server at a time (avoid simultaneous downtime)
kamal proxy restart         # restart the existing proxy container
kamal proxy start           # start a stopped proxy container
kamal proxy stop            # stop the proxy container
kamal proxy details         # show proxy container info per server
kamal proxy logs -f         # stream proxy logs
kamal proxy remove          # remove the proxy container and image
```

## Gotchas

- Forgetting to exclude `/up` from the SSL redirect → 301 on the probe →
  deploy fails. Exclude it in both `ssl_options` and `host_authorization`.
- Short `deploy_timeout` on low-RAM servers: both containers run during cutover,
  swap thrashing slows boot, the probe times out. Raise the timeout.
- The proxy is per-server only — it does not load-balance across servers. Put a
  managed load balancer in front for multi-server web roles.
- `ssl: true` needs a single resolvable `host`; it cannot issue a cert for a
  bare IP address.

See troubleshooting.md for failed-cutover and certificate diagnosis.

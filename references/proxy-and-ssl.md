# Kamal 2 — Proxy & SSL Reference

> This covers `kamal-proxy`, the built-in reverse proxy that replaced Traefik in Kamal 2.  
> For Traefik (Kamal 1.x), see `kamal-v1.md`.

## Table of Contents

1. [What kamal-proxy does](#what-kamal-proxy-does)
2. [Full proxy: configuration block](#full-proxy-configuration-block)
3. [Automatic SSL with Let's Encrypt](#automatic-ssl-with-lets-encrypt)
4. [Multiple hostnames](#multiple-hostnames)
5. [Custom TLS certificates](#custom-tls-certificates)
6. [Streaming, SSE, WebSockets](#streaming-sse-websockets)
7. [Per-role proxy](#per-role-proxy)
8. [Path prefix routing](#path-prefix-routing)
9. [Proxy runtime tuning](#proxy-runtime-tuning)
10. [Proxy CLI commands](#proxy-cli-commands)
11. [Gotchas](#gotchas)

---

## What kamal-proxy does

`kamal-proxy` is a lightweight reverse proxy that runs as a Docker container on each server on ports 80 and 443. It:

- Routes incoming HTTP(S) requests to the correct app container
- Performs zero-downtime traffic switching during deploys
- Manages Let's Encrypt certificate provisioning and renewal
- Handles connection draining when old containers stop
- Supports multiple apps on the same server (Kamal 2 feature)

Zero-downtime sequence:
```
new container starts
    ↓
proxy polls healthcheck endpoint (default: GET /up every 1s)
    ↓
healthcheck passes → proxy routes all new requests to new container
    ↓
in-flight requests to old container drain (up to drain_timeout seconds)
    ↓
old container stops
```

---

## Full proxy: configuration block

```yaml
proxy:
  # Routing
  host: app.example.com          # hostname (required for SSL; also used for routing)
  app_port: 3000                 # container's listening port (default: 80)

  # SSL
  ssl: true                      # auto Let's Encrypt (default: false)
  ssl_redirect: true             # redirect HTTP → HTTPS (default: true when ssl: true)
  forward_headers: true          # forward X-Forwarded-For, X-Forwarded-Proto (REQUIRED with ssl: true)

  # Healthcheck
  healthcheck:
    path: /up                    # default: /up
    interval: 1                  # seconds between polls (default: 1)
    timeout: 5                   # seconds per request (default: 5)

  # Timeouts (also set at top-level)
  response_timeout: 30           # max seconds to wait for app response (default: 30)

  # Request/response buffering
  # Disable for streaming, SSE, WebSockets
  buffering:
    requests: true               # default: true
    responses: true              # default: true
    max_request_body: 40000000   # bytes (default: ~1 GB)
    max_response_body: 0         # bytes, 0 = unlimited
    memory: 2000000              # bytes (default: 1 MB)
```

Top-level deployment timeout settings (sibling of `proxy:`, not inside it):
```yaml
deploy_timeout: 30      # total seconds to wait for new container to pass healthchecks (default: 30)
drain_timeout: 30       # seconds to drain in-flight requests before forcibly stopping old container (default: 30)
readiness_delay: 0      # extra seconds to wait before starting healthchecks (default: 0)
```

---

## Automatic SSL with Let's Encrypt

The simplest way to get HTTPS — kamal-proxy handles certificate issuance and auto-renewal:

```yaml
proxy:
  host: app.example.com    # must match your DNS
  ssl: true
  forward_headers: true    # critical! see Gotchas
```

**Prerequisites:**
- DNS A record pointing `app.example.com` to your server's IP (must be live before `kamal setup`)
- Port 80 and 443 open on the server's firewall
- Only one server (Let's Encrypt is per-server; see Multiple hostnames for multi-server)

Certificates are auto-renewed before expiry. No manual intervention needed.

---

## Multiple hostnames

### Single server, multiple domains

```yaml
proxy:
  hosts:
    - app.example.com
    - www.example.com
    - api.example.com
  ssl: true
  forward_headers: true
```

### Multiple servers

With multiple servers, deploy each as a separate service or configure your DNS/load balancer to distribute between them. kamal-proxy on each server manages its own certificate.

---

## Custom TLS certificates

Use when you have certificates from an external CA, need wildcard certs, or can't use Let's Encrypt for compliance reasons.

```yaml
proxy:
  host: app.example.com
  ssl:
    certificate_pem: SSL_CERTIFICATE_PEM    # name of secret in .kamal/secrets
    private_key_pem: SSL_PRIVATE_KEY_PEM    # name of secret in .kamal/secrets
  forward_headers: true
```

`.kamal/secrets`:
```bash
SSL_CERTIFICATE_PEM=$(cat /path/to/certificate.pem)
SSL_PRIVATE_KEY_PEM=$(cat /path/to/private-key.pem)
```

> ⚠️ Custom certificates are NOT auto-renewed. You must update the secrets and redeploy before expiry.

---

## Streaming, SSE, WebSockets

Buffering must be disabled for long-lived connections:

```yaml
proxy:
  buffering:
    requests: false
    responses: false
```

Also ensure `response_timeout` is high enough (or 0 for no limit):
```yaml
proxy:
  response_timeout: 0     # no timeout for streaming responses
  buffering:
    requests: false
    responses: false
```

---

## Per-role proxy

Different roles can have different proxy settings:

```yaml
servers:
  web:
    hosts: [192.168.0.1]
    proxy:
      app_port: 3000
      ssl: true
      host: app.example.com
      forward_headers: true
  api:
    hosts: [192.168.0.2]
    proxy:
      app_port: 4000
      ssl: true
      host: api.example.com
      forward_headers: true
  worker:
    hosts: [192.168.0.3]
    # no proxy: block — workers don't need a proxy
```

---

## Path prefix routing

Route requests to a specific app based on URL prefix:

```yaml
proxy:
  host: example.com
  ssl: true
  path_prefix: /api           # only /api/* requests reach this app
  strip_path_prefix: true     # strip /api prefix before forwarding (default: true)
  # path_prefixes:            # multiple prefixes
  #   - /api
  #   - /v2
```

---

## Proxy runtime tuning

Advanced container configuration for kamal-proxy itself:

```yaml
proxy:
  run:
    http_port: 80              # default: 80
    https_port: 443            # default: 443
    log_max_size: 10m          # default: 10m
    debug: false               # default: false
    publish: true              # default: true (publish ports to host)
    metrics_port: 9090         # expose Prometheus metrics
    bind_ips:
      - 0.0.0.0                # default: all interfaces
    repository: basecamp/kamal-proxy    # default
    version: latest            # pin a specific kamal-proxy version
    options:
      memory: 256m
      cpus: "0.5"
```

Persist proxy config across reboots:
```bash
kamal proxy boot_config set --http-port 80 --https-port 443 --log-max-size 50m
kamal proxy boot_config get    # inspect current config
kamal proxy boot_config reset  # restore defaults
```

---

## Proxy CLI commands

```bash
kamal proxy details           # show proxy container status
kamal proxy logs              # show proxy logs
kamal proxy logs -f           # follow proxy logs
kamal proxy logs -n 100       # last 100 lines
kamal proxy reboot            # restart proxy (brief downtime)
kamal proxy reboot --rolling  # staged restart (less downtime)
kamal proxy start             # start stopped proxy
kamal proxy stop              # stop proxy
kamal proxy restart           # restart proxy
kamal proxy remove            # remove proxy container and image
```

---

## Gotchas

### 1. `forward_headers` is NOT automatic with SSL

When `ssl: true`, the proxy terminates TLS before forwarding to your app. Without `forward_headers: true`, your app sees:
- `X-Forwarded-For`: missing → app sees proxy's IP instead of client IP
- `X-Forwarded-Proto`: missing → app thinks requests are plain HTTP → secure cookies break, HTTPS redirects loop

**Always add `forward_headers: true` when `ssl: true`.**

### 2. DNS must be live before `kamal setup` (for auto-SSL)

Let's Encrypt validates domain ownership via HTTP challenge. If DNS isn't pointing to the server yet when you run `kamal setup`, certificate issuance will fail.

### 3. Outdated kamal-proxy version

```
Error: kamal-proxy version X.Y.Z is too old, run kamal proxy reboot
```

Fix:
```bash
kamal proxy reboot
```

### 4. Port 80/443 conflicts

If another service (e.g. nginx, another app, an old Traefik from v1) is already using port 80 or 443, kamal-proxy will fail to start. Stop and remove the conflicting service first.

### 5. App not listening on the right port

`proxy.app_port` (default: 80) must match the port your container actually listens on. If your app listens on port 3000 but you didn't set `app_port: 3000`, all healthchecks will fail.

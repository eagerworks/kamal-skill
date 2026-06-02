# Kamal 1.9.x — Reference & Upgrade Guide

> This file documents **Kamal 1.9.x** (the last stable v1 release series). If you're on Kamal 2, refer to the other reference files.

## Table of Contents

1. [Detecting v1](#detecting-v1)
2. [Installation](#installation)
3. [config/deploy.yml — Traefik block](#configdeployyml--traefik-block)
4. [healthcheck in v1](#healthcheck-in-v1)
5. [env and secrets in v1](#env-and-secrets-in-v1)
6. [SSL in v1 (via Traefik)](#ssl-in-v1-via-traefik)
7. [v1-only CLI commands](#v1-only-cli-commands)
8. [Hooks in v1](#hooks-in-v1)
9. [Builder in v1](#builder-in-v1)
10. [Upgrading from v1 to v2](#upgrading-from-v1-to-v2)

---

## Detecting v1

If an existing repo has any of these, it's Kamal v1:
- `traefik:` key in `config/deploy.yml`
- `.env` or `.env.erb` file at the project root (not `.kamal/secrets`)
- `kamal env push` in CI scripts

```bash
kamal version    # prints e.g. "1.9.2"
```

---

## Installation

```bash
gem install kamal                   # installs latest (currently v2.x)
gem install kamal --version 1.9.2   # pin to specific v1 release
kamal version                       # verify
```

Initialize a new project:
```bash
kamal init
# Creates: config/deploy.yml, .env (not .kamal/secrets), .kamal/hooks/
```

Using Docker (no Ruby needed):
```bash
alias kamal="docker run -it --rm \
  -v '${PWD}:/workdir' \
  -v '${SSH_AUTH_SOCK}:/ssh-agent' \
  -e 'SSH_AUTH_SOCK=/ssh-agent' \
  ghcr.io/basecamp/kamal:v1.9.2 \
  kamal"
```

---

## config/deploy.yml — Traefik block

In v1, the reverse proxy is **Traefik** (not kamal-proxy). The `traefik:` config block uses raw Traefik CLI args and labels.

### Minimal Traefik config (HTTP only)

```yaml
traefik:
  image: traefik:v2.10       # default: traefik:v2.10
  host_port: "80"            # port on the host (default: 80)
```

### Full Traefik config with all options

```yaml
traefik:
  image: traefik:v2.10
  host_port: "80"            # host-side port (default: 80)
  publish: true              # publish port to host (default: true)
  env:
    SOME_ENV_VAR: value
  labels:
    traefik.http.routers.catchall.entryPoints: http
    traefik.http.routers.catchall.rule: PathPrefix(`/`)
    traefik.http.routers.catchall.service: unavailable
    traefik.http.routers.catchall.priority: "1"
    traefik.http.services.unavailable.loadbalancer.server.port: "0"
  args:
    entryPoints.http.address: ":80"
    entryPoints.http.forwardedHeaders.insecure: true
    accesslog: true
    accesslog.format: json
  options:
    memory: 512m
    cpus: "1"
```

### Key differences from Kamal 2's `proxy:`

| Aspect | v1 `traefik:` | v2 `proxy:` |
|---|---|---|
| Configuration | Raw Traefik args + labels | High-level keys (`host`, `ssl`, `app_port`) |
| SSL | Traefik ACME certresolver or external | `ssl: true` (native Let's Encrypt) |
| App port | `healthcheck.port` (default: 3000) | `proxy.app_port` (default: 80) |
| Forward headers | `args.entryPoints.http.forwardedHeaders.insecure: true` | `forward_headers: true` |
| Networking | Host network | Isolated Docker network |

---

## healthcheck in v1

```yaml
healthcheck:
  path: /up                  # endpoint to check (default: /)
  port: "3000"               # app port (default: 3000 — different from v2!)
  cmd: "curl -f http://localhost:3000/up"   # custom curl command
  interval: 10s              # Docker healthcheck interval (default: 10s)
  max_attempts: 7            # retries before failure (default: 7)
  cord: /tmp/kamal-cord      # cord file for zero-downtime (default: /tmp/kamal-cord; set false to disable)
  log_lines: 50              # lines logged on failure (default: 50)
```

> ⚠️ **Default port is `3000`** in v1 (vs `80` in v2). The `cord` key is a v1-specific zero-downtime mechanism.

In v1, `curl` must be available inside the container (used for healthchecks).

---

## env and secrets in v1

### env block (same structure as v2)

```yaml
env:
  clear:
    NODE_ENV: production
    DB_HOST: 192.168.0.5
  secret:
    - DATABASE_PASSWORD
    - SESSION_SECRET
```

### How secrets are resolved in v1

Secrets listed under `env.secret` are resolved from your **local shell environment** and the `.env` file — NOT from `.kamal/secrets` (that's v2).

**`.env` file** (project root, gitignored):
```bash
# .env — plain dotenv format
KAMAL_REGISTRY_PASSWORD=your-registry-token
DATABASE_PASSWORD=your-db-password
SESSION_SECRET=your-session-secret
```

**`.env.erb` template** (v1 only) — ERB is rendered into `.env` via `kamal envify`:
```erb
KAMAL_REGISTRY_PASSWORD=<%= ENV["KAMAL_REGISTRY_PASSWORD"] %>
DATABASE_PASSWORD=<%= `aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text`.strip %>
```

Render:
```bash
kamal envify    # renders .env.erb → .env
```

Push secrets to servers:
```bash
kamal env push    # reads .env and pushes to all servers
```

Remove secrets from servers:
```bash
kamal env delete
```

> ⚠️ In v2, `.env` / `kamal env push` / `kamal envify` are all removed. Use `.kamal/secrets` instead.

---

## SSL in v1 (via Traefik)

v1 has no native Let's Encrypt support. SSL is configured manually via Traefik's ACME certresolver or by using an external TLS terminator.

### Option A: Let's Encrypt via Traefik ACME

```yaml
traefik:
  args:
    entryPoints.web.address: ":80"
    entryPoints.websecure.address: ":443"
    certificatesResolvers.letsencrypt.acme.email: "you@example.com"
    certificatesResolvers.letsencrypt.acme.storage: "/letsencrypt/acme.json"
    certificatesResolvers.letsencrypt.acme.httpChallenge: true
    certificatesResolvers.letsencrypt.acme.httpChallenge.entryPoint: web
  labels:
    traefik.http.routers.myapp.entrypoints: websecure
    traefik.http.routers.myapp.rule: Host(`app.example.com`)
    traefik.http.routers.myapp.tls: "true"
    traefik.http.routers.myapp.tls.certresolver: letsencrypt
    # HTTP → HTTPS redirect
    traefik.http.routers.myapp-redirect.entrypoints: web
    traefik.http.routers.myapp-redirect.rule: Host(`app.example.com`)
    traefik.http.routers.myapp-redirect.middlewares: redirect-to-https
    traefik.http.middlewares.redirect-to-https.redirectscheme.scheme: https
  options:
    volumes:
      - "/letsencrypt:/letsencrypt"   # persist ACME certificates across reboots
```

### Option B: Cloudflare as TLS terminator

Simpler — Cloudflare handles TLS, Kamal handles plain HTTP. Traefik runs on port 80 only. In Cloudflare: set SSL/TLS mode to "Flexible" or "Full". No Traefik config changes needed.

---

## v1-only CLI commands

These commands exist in v1 but are **removed in v2**:

### `kamal traefik`

```bash
kamal traefik boot          # start Traefik on servers
kamal traefik reboot        # stop → remove → start Traefik (brief downtime)
kamal traefik reboot --rolling   # staged restart
kamal traefik start         # start existing stopped Traefik container
kamal traefik stop          # stop Traefik
kamal traefik restart       # restart Traefik
kamal traefik details       # show Traefik container details
kamal traefik logs          # show Traefik logs
kamal traefik remove        # remove Traefik container + image
```

v2 equivalent: `kamal proxy <subcommand>` (same subcommand names).

### `kamal env`

```bash
kamal env push     # push .env secrets to all servers
kamal env delete   # remove secrets file from all servers
```

v2 equivalent: secrets are pushed automatically on every `kamal deploy` from `.kamal/secrets`.

### `kamal envify`

```bash
kamal envify    # render .env.erb → .env
```

No v2 equivalent (use `.kamal/secrets` with `$(command)` interpolation instead).

---

## Hooks in v1

Same mechanism as v2 (`.kamal/hooks/`, no extension, executable, non-zero exits abort). KAMAL_* env vars are the same.

**v1 hook names:**

| Hook | Description |
|---|---|
| `docker-setup` | After Docker is confirmed available on each server |
| `pre-connect` | Before SSH connections (auto-loads `.env` secrets) |
| `pre-build` | Before Docker image build |
| `pre-deploy` | After build/push, before containers swap |
| `post-deploy` | After traffic switches to new containers |
| `pre-traefik-reboot` | Before Traefik is rebooted |
| `post-traefik-reboot` | After Traefik is rebooted |

**v1 vs v2 hook differences:**

| v1 | v2 | Notes |
|---|---|---|
| `pre-traefik-reboot` | `pre-proxy-reboot` | Renamed |
| `post-traefik-reboot` | `post-proxy-reboot` | Renamed |
| *(none)* | `pre-app-boot` | New in v2 |
| *(none)* | `post-app-boot` | New in v2 |

---

## Builder in v1

v1 builder has a different structure (note: `multiarch`, `local`/`remote` as nested maps):

```yaml
builder:
  multiarch: true                    # enable multiarch builds (default: true)
  local:
    arch: arm64                      # local machine arch
    host: /var/run/docker.sock
  remote:
    arch: amd64                      # remote builder arch
    host: ssh://docker@builder.example.com
  cache:
    type: registry
    options: mode=max
    image: myapp-build-cache
  context: .
  dockerfile: Dockerfile
  target: production
  args:
    NODE_VERSION: "20"
  secrets:
    - GITHUB_TOKEN
  ssh: default=$SSH_AUTH_SOCK
```

v2 equivalent:
```yaml
builder:
  arch:
    - amd64
    - arm64
  remote: ssh://docker@builder.example.com
  cache:
    type: registry
    options: mode=max
```

---

## Upgrading from v1 to v2

### Before you start

- **Upgrade to v1.9.x first** (enables `kamal downgrade` as a safety net)
- Back up your `config/deploy.yml` and `.env`
- Test your upgraded config with `kamal config` before touching servers

### Step-by-step

**1. Install Kamal 2:**
```bash
gem install kamal            # installs latest (v2.x)
kamal version                # verify: "2.x.y"
```

**2. Update `config/deploy.yml`:**

| v1 | v2 |
|---|---|
| `traefik:` block | Remove entirely; add `proxy:` block |
| `healthcheck.port: "3000"` | `proxy.app_port: 3000` |
| `healthcheck.path: /up` | `proxy.healthcheck.path: /up` |
| `healthcheck.interval: 10s` | `proxy.healthcheck.interval: 1` (seconds, integer) |
| *(no equivalent)* | `proxy.ssl: true` (if using SSL) |
| *(no equivalent)* | `proxy.forward_headers: true` |
| `builder.multiarch`, `builder.local`, `builder.remote` | `builder.arch`, `builder.remote` |

Example conversion:

**v1:**
```yaml
traefik:
  host_port: "80"

healthcheck:
  path: /up
  port: "3000"
  interval: 10s
```

**v2:**
```yaml
proxy:
  app_port: 3000
  ssl: true
  host: app.example.com
  forward_headers: true
  healthcheck:
    path: /up
    interval: 1
```

**3. Migrate secrets: `.env` → `.kamal/secrets`:**

```bash
# Copy values from .env to .kamal/secrets
cp .env .kamal/secrets
# Edit .kamal/secrets to use the correct format:
# Remove .env-specific comments/headers
# KAMAL_REGISTRY_PASSWORD=your-token  (same format — just different file)

# Update .gitignore
echo ".kamal/secrets*" >> .gitignore
echo "!.kamal/secrets.example" >> .gitignore

# Remove .env from git tracking (if it was tracked)
git rm --cached .env
```

**4. Update hooks** (if any):

- Rename `.kamal/hooks/pre-traefik-reboot` → `.kamal/hooks/pre-proxy-reboot`
- Rename `.kamal/hooks/post-traefik-reboot` → `.kamal/hooks/post-proxy-reboot`

**5. Remove Dockerfile `EXPOSE` if it was `3000` and you want to use the v2 default `80`:**

Or set `proxy.app_port: 3000` to keep port 3000 — either works.

**6. Validate:**
```bash
kamal config          # should show no errors and correct values
```

**7. Run the upgrade:**
```bash
kamal upgrade          # single server or all at once
# or
kamal upgrade --rolling   # staged (for multi-server fleets)
```

What `kamal upgrade` does:
1. Removes Traefik container
2. Creates the `kamal` Docker network
3. Reboots all containers in the new network
4. Starts kamal-proxy

**8. Rollback if needed (v1.9.x only):**
```bash
kamal downgrade          # reverts to Traefik, v1 networking
kamal downgrade --rolling
```

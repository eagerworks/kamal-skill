# Kamal 2 — config/deploy.yml Reference

> This file covers **Kamal 2.x** only. For Kamal 1.x (Traefik-based), see `kamal-v1.md`.

## Table of Contents

1. [Minimal required config](#minimal-required-config)
2. [service](#service)
3. [image](#image)
4. [servers](#servers)
5. [registry](#registry)
6. [env](#env)
7. [proxy](#proxy)
8. [builder](#builder)
9. [accessories](#accessories)
10. [ssh](#ssh)
11. [volumes & labels & logging](#volumes-labels-logging)
12. [asset_path](#asset_path)
13. [boot](#boot)
14. [aliases](#aliases)
15. [Global deployment settings](#global-deployment-settings)
16. [Destinations (multi-environment)](#destinations-multi-environment)
17. [Full annotated example](#full-annotated-example)

---

## Minimal required config

```yaml
service: myapp
image: myuser/myapp

servers:
  - 192.168.0.1

registry:
  username: myuser
  password:
    - KAMAL_REGISTRY_PASSWORD
```

`.kamal/secrets`:
```
KAMAL_REGISTRY_PASSWORD=your-registry-token
```

---

## service

Container name prefix. Must be alphanumeric, hyphens, underscores. Used to name containers (`myapp-web-v<sha>`), volumes, network, etc.

```yaml
service: myapp
```

---

## image

Docker image name **without** registry hostname or tag. Tag is set automatically from the git commit SHA (or `VERSION` env var).

```yaml
image: myuser/myapp
# For GHCR: ghcr.io prefix goes in registry.server, not here
# image: myapp    (just the name, no registry prefix)
```

---

## servers

### Simple list (all are `web` role)

```yaml
servers:
  - 192.168.0.1
  - 192.168.0.2
```

### Named roles

```yaml
servers:
  web:
    hosts:
      - 192.168.0.1
      - 192.168.0.2
  worker:
    hosts:
      - 192.168.0.3
    cmd: node worker.js       # override container command
  scheduler:
    hosts:
      - 192.168.0.4
    cmd: node scheduler.js
```

### Per-role options

```yaml
servers:
  web:
    hosts:
      - 192.168.0.1
    env:
      clear:
        WEB_CONCURRENCY: "4"
    labels:
      team: frontend
    options:
      memory: 2g
      cpus: 2
    logging:
      driver: json-file
      options:
        max-size: 100m
        max-file: "3"
    proxy:
      app_port: 3000     # per-role proxy port
      ssl: true
      host: app.example.com
  worker:
    hosts:
      - 192.168.0.2
    cmd: node worker.js
    options:
      memory: 4g
```

### Host tags (per-host env vars)

```yaml
servers:
  web:
    - 192.168.0.1: primary
    - 192.168.0.2: [replica, eu]

env:
  tags:
    primary:
      clear:
        ROLE: primary
    replica:
      clear:
        ROLE: replica
        READ_ONLY: "true"
```

`primary_role` (default: `web`) — controls which host runs primary-only commands (`-p`/`--primary`):

```yaml
primary_role: web
```

---

## registry

### Docker Hub (default if `server` omitted)

```yaml
registry:
  username: myuser
  password:
    - KAMAL_REGISTRY_PASSWORD    # value read from .kamal/secrets
```

### GitHub Container Registry (GHCR)

```yaml
registry:
  server: ghcr.io
  username: github-username
  password:
    - GITHUB_TOKEN    # PAT with write:packages scope
```

### AWS ECR (token expires every 12h — use ERB)

```yaml
registry:
  server: 123456789012.dkr.ecr.us-east-1.amazonaws.com
  username: AWS
  password: <%= `aws ecr get-login-password --region us-east-1` %>
```

### Google Artifact Registry

```yaml
registry:
  server: us-docker.pkg.dev
  username: _json_key_base64
  password:
    - GAR_KEY_BASE64    # base64-encoded service account JSON key
```

**Rules:**
- `password:` is **always an array** of secret variable names. A plain string will fail.
- Use access tokens / PATs — never your personal password.
- Never commit registry credentials to version control.

---

## env

```yaml
env:
  clear:                   # committed to repo, visible in container
    NODE_ENV: production
    PORT: "3000"
    DB_HOST: 192.168.0.5
  secret:                  # names only — values come from .kamal/secrets
    - DATABASE_PASSWORD
    - SESSION_SECRET
    - API_KEY
```

### Aliased secrets (env var name ≠ secret key)

```yaml
env:
  secret:
    - DB_PASS:PROD_DATABASE_PASSWORD    # container gets DB_PASS, value from PROD_DATABASE_PASSWORD
```

### Per-role env (inline under `servers:`)

```yaml
servers:
  worker:
    hosts: [192.168.0.3]
    env:
      clear:
        QUEUE: default,mailer
      secret:
        - WORKER_CONCURRENCY
```

### Tag-scoped env

```yaml
env:
  tags:
    primary:
      clear: { LEADER: "true" }
    replica:
      secret: [REPLICA_TOKEN]
```

**Env variable precedence** (later wins):
root clear → root secret → role clear → role secret → tag clear → tag secret

---

## proxy

Controls `kamal-proxy` (the built-in reverse proxy, replaces Traefik in v2).

```yaml
proxy:
  host: app.example.com      # single hostname (required for SSL)
  # hosts:                   # OR multiple hostnames
  #   - app.example.com
  #   - www.example.com
  app_port: 3000             # internal port your container listens on (default: 80)
  ssl: true                  # auto Let's Encrypt cert; requires host + DNS + ports 80+443 open
  ssl_redirect: true         # HTTP→HTTPS redirect (default: true)
  forward_headers: true      # pass X-Forwarded-For, X-Forwarded-Proto (REQUIRED when ssl: true)
  response_timeout: 60       # seconds to wait for app response (default: 30)
  healthcheck:
    path: /up                # default: /up
    interval: 1              # seconds between polls (default: 1)
    timeout: 5               # seconds per request (default: 5)
  buffering:
    requests: true           # set false for SSE/streaming/WebSocket upload
    responses: true          # set false for SSE/streaming
    max_request_body: 40000000   # bytes (default: 1 GB)
    max_response_body: 0         # bytes, 0 = unlimited
    memory: 2000000              # bytes (default: 1 MB)
  path_prefix: /api          # or path_prefixes: [/api, /webhooks]
  strip_path_prefix: true    # default: true
```

### Custom TLS certificates

```yaml
proxy:
  host: app.example.com
  ssl:
    certificate_pem: SSL_CERTIFICATE_PEM    # secret name
    private_key_pem: SSL_PRIVATE_KEY_PEM    # secret name
```

`.kamal/secrets`:
```
SSL_CERTIFICATE_PEM=$(cat /path/to/certificate.pem)
SSL_PRIVATE_KEY_PEM=$(cat /path/to/private-key.pem)
```

See `proxy-and-ssl.md` for full proxy documentation.

---

## builder

```yaml
builder:
  arch:
    - amd64         # build target(s)
    - arm64
  driver: docker-container    # default; use 'docker' for single-arch local-only
  context: .                  # build context (default: tmp clone of HEAD)
  dockerfile: Dockerfile      # default: Dockerfile
  target: production          # multi-stage build target
  args:
    NODE_VERSION: "20"
    BUILD_ENV: production
  secrets:
    - GITHUB_TOKEN            # build-time secrets (docker build --secret)
    - NPM_TOKEN
  cache:
    type: registry            # or: local
    options: mode=max
    image: myapp-build-cache  # optional custom cache image name
```

### Remote builder (cross-arch via SSH — required for Apple Silicon → amd64)

```yaml
builder:
  remote: ssh://user@builder.example.com
  # local: false    # uncomment to always build on remote, never locally
  arch:
    - amd64
    - arm64
```

See `builders.md` for full builder documentation.

---

## accessories

Accessories are side services (databases, caches, queues) managed by Kamal but independent of app deploys.

```yaml
accessories:
  db:
    image: postgres:16
    host: 192.168.0.5        # single host
    # hosts: [...]           # multiple hosts
    # roles: [worker]        # hosts belonging to a role
    port: "127.0.0.1:5432:5432"    # bind to localhost only!
    env:
      clear:
        POSTGRES_DB: myapp_production
        POSTGRES_USER: myapp
      secret:
        - POSTGRES_PASSWORD
    directories:
      - postgres-data:/var/lib/postgresql/data    # named volume
    files:
      - config/postgres/postgresql.conf:/etc/postgresql/postgresql.conf
    options:
      restart: unless-stopped
      memory: 4g
      shm-size: 256m

  cache:
    image: redis:7-alpine
    hosts:
      - 192.168.0.5
    port: "127.0.0.1:6379:6379"
    cmd: redis-server --maxmemory 1gb --maxmemory-policy allkeys-lru
    directories:
      - redis-data:/data
    options:
      restart: unless-stopped
```

**Accessory boot commands:**
```bash
kamal accessory boot db          # first time
kamal accessory reboot db        # apply image/config changes (brief downtime)
kamal accessory logs db -f
kamal accessory exec db -i "psql -U myapp myapp_production"
```

> ⚠️ **Always bind ports to `127.0.0.1:PORT:PORT`**, not `PORT:PORT`. Bare port mapping exposes the service publicly.

---

## ssh

```yaml
ssh:
  user: deploy        # default: root
  port: 22            # default: 22
  proxy: user@bastion.example.com    # jump/bastion host
  keys:
    - ~/.ssh/id_ed25519
    - ~/.ssh/deploy_key
  keys_only: true     # ignore ssh-agent, use only listed keys
  log_level: debug    # debug|info|warn|error|fatal (default: fatal)
```

---

## volumes, labels, logging

Top-level (applied to all roles):

```yaml
volumes:
  - "/host/path:/container/path"
  - "/data:/app/data:ro"

labels:
  app: myapp
  environment: production

logging:
  driver: json-file
  options:
    max-size: 100m
    max-file: "3"
```

---

## asset_path

Prevents 404s during rolling deploys by syncing static assets between old and new container versions.

```yaml
asset_path: /app/public       # path inside the container where assets live
```

See `builders.md` for how asset bridging works.

---

## boot

Controls rolling deploy behavior:

```yaml
boot:
  limit: 2          # servers to deploy at once (fixed number or "25%")
  wait: 10          # seconds to wait between batches
```

---

## aliases

Custom command shortcuts (`kamal <alias>`):

```yaml
aliases:
  shell: app exec --interactive --reuse "sh"
  console: app exec --interactive --reuse "node repl.js"
  logs: app logs --follow
  db-console: app exec --interactive --reuse "psql -U myapp"
  migrate: app exec "node migrate.js"
```

Usage:
```bash
kamal shell
kamal logs --grep ERROR
kamal migrate
```

---

## Global deployment settings

```yaml
retain_containers: 5         # how many old containers to keep per host (default: 5)
deploy_timeout: 30           # seconds to wait for healthchecks (default: 30)
drain_timeout: 30            # seconds to drain in-flight requests (default: 30)
readiness_delay: 0           # extra wait before healthchecks start (default: 0)
require_destination: false   # require -d flag (default: false)
```

---

## Destinations (multi-environment)

Base config: `config/deploy.yml`  
Destination override: `config/deploy.<destination>.yml`

Override files are **deep-merged** on top of the base — only set keys that differ.

`config/deploy.staging.yml`:
```yaml
servers:
  web:
    - 10.0.0.1

proxy:
  host: staging.example.com
  ssl: true

env:
  clear:
    NODE_ENV: staging
```

Usage:
```bash
kamal deploy -d staging
kamal rollback abc123 -d staging
kamal app logs -d staging
```

**Secrets per destination:**  
`.kamal/secrets` (base) + `.kamal/secrets.<destination>` (e.g. `.kamal/secrets.staging`)  
Destination secrets are merged on top of the base secrets.

---

## Full annotated example

```yaml
service: myapp
image: ghcr.io/myorg/myapp

servers:
  web:
    hosts:
      - 192.168.0.1
      - 192.168.0.2
    env:
      clear:
        WEB_WORKERS: "4"
  worker:
    hosts:
      - 192.168.0.3
    cmd: node worker.js
    env:
      clear:
        QUEUE: default,email
    options:
      memory: 2g

registry:
  server: ghcr.io
  username: myorg
  password:
    - KAMAL_REGISTRY_PASSWORD

builder:
  arch:
    - amd64
  cache:
    type: registry
    options: mode=max

env:
  clear:
    NODE_ENV: production
    DB_HOST: 192.168.0.5
    DB_PORT: "5432"
    REDIS_URL: redis://127.0.0.1:6379
  secret:
    - DATABASE_PASSWORD
    - SESSION_SECRET

proxy:
  host: app.example.com
  ssl: true
  forward_headers: true
  app_port: 3000
  healthcheck:
    path: /healthz
    interval: 1
    timeout: 5
  response_timeout: 30

ssh:
  user: deploy

volumes:
  - "/data/uploads:/app/uploads"

asset_path: /app/public

boot:
  limit: "50%"
  wait: 10

retain_containers: 5
deploy_timeout: 60
drain_timeout: 30

accessories:
  db:
    image: postgres:16
    host: 192.168.0.5
    port: "127.0.0.1:5432:5432"
    env:
      clear:
        POSTGRES_DB: myapp_production
        POSTGRES_USER: myapp
      secret:
        - POSTGRES_PASSWORD
    directories:
      - postgres-data:/var/lib/postgresql/data
    options:
      restart: unless-stopped
      memory: 4g
  cache:
    image: redis:7-alpine
    host: 192.168.0.5
    port: "127.0.0.1:6379:6379"
    directories:
      - redis-data:/data
    options:
      restart: unless-stopped

aliases:
  shell: app exec --interactive --reuse "sh"
  logs: app logs --follow
  migrate: app exec "node migrate.js"
```

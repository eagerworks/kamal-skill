# Kamal 2 — Deployment Workflows

## Table of Contents

1. [First-time setup](#first-time-setup)
2. [deploy vs redeploy vs app boot](#deploy-vs-redeploy-vs-app-boot)
3. [Zero-downtime mechanics](#zero-downtime-mechanics)
4. [Rolling deploys](#rolling-deploys)
5. [Rollback](#rollback)
6. [Multi-server / multi-role](#multi-server--multi-role)
7. [Multiple environments (destinations)](#multiple-environments-destinations)

---

## First-time setup

### Prerequisites

- **Local machine:** Ruby 3.0+, Docker, SSH key pair
- **Servers:** SSH access (root or sudo user; key-based), curl/wget, internet connectivity (Docker is auto-installed by Kamal)
- **Registry:** Account on Docker Hub, GHCR, ECR, GAR, or similar
- **DNS:** Point your domain to the server IP before `kamal setup` if using `ssl: true`
- **Firewall:** Port 22 (SSH), 80 (HTTP), 443 (HTTPS if using SSL) open

### Step-by-step

```bash
# 1. Install Kamal
gem install kamal

# 2. Scaffold inside your project
cd myapp
kamal init
# Creates:
#   config/deploy.yml
#   .kamal/secrets
#   .kamal/hooks/ (sample hook scripts)

# 3. Configure (use assets/deploy.yml as a reference)
#    Required: service, image, servers[0], registry.username, registry.password
vim config/deploy.yml

# 4. Add secrets to .kamal/secrets (NEVER commit this file)
echo "KAMAL_REGISTRY_PASSWORD=your-registry-token" >> .kamal/secrets
echo "DATABASE_PASSWORD=your-db-password" >> .kamal/secrets

# 5. Add .kamal/secrets* to .gitignore
echo ".kamal/secrets*" >> .gitignore
echo "!.kamal/secrets.example" >> .gitignore

# 6. Verify merged config (also shows secrets — careful in shared terminals)
kamal config

# 7. Bootstrap servers + first deploy
#    - Installs Docker on servers if missing
#    - Boots all accessories (databases, redis, etc.)
#    - Pushes image and starts app
#    - Starts kamal-proxy
kamal setup
```

---

## deploy vs redeploy vs app boot

### `kamal deploy` (normal ongoing deploy)

Full deploy sequence:
1. Acquire deploy lock
2. Run `pre-connect` hook
3. Check SSH connections
4. Run `pre-build` hook
5. Build Docker image + push to registry (or `--skip-push` to skip)
6. Boot kamal-proxy (if not running)
7. Push env/secrets to servers
8. Run `pre-deploy` hook
9. Detect and stop stale containers
10. Run `pre-app-boot` hook
11. Boot new app containers with zero-downtime swap
12. Run `post-app-boot` hook
13. Run `post-deploy` hook
14. Prune old containers and images
15. Release deploy lock

```bash
kamal deploy
kamal deploy --skip-push             # use existing image (skip build+push)
kamal deploy -r web                  # deploy only web role
kamal deploy -r web,worker           # deploy specific roles
kamal deploy -h 192.168.0.1          # deploy to one host
kamal deploy -d staging              # deploy to staging destination
kamal deploy -H                      # skip all hooks
```

### `kamal redeploy` (faster, for in-place refreshes)

Skips: server bootstrapping, proxy start, pruning, registry login, lock.  
Includes: build → push → swap containers.

Use when: infra is running and you just need to refresh the app.

```bash
kamal redeploy
kamal redeploy --skip-push
```

### `kamal app boot` (manual container swap)

Most targeted — just boots new containers without build/push/prune.

```bash
kamal app boot
kamal app boot --version abc123d     # boot a specific image version
kamal app boot -r worker             # boot only a specific role
```

---

## Zero-downtime mechanics

Kamal achieves zero downtime through **kamal-proxy**:

1. **New container boots** alongside the old one (both run briefly in parallel)
2. **Healthchecks** — kamal-proxy polls the new container's health endpoint:
   - Default: GET `/up` every **1 second**, 5-second timeout per request
   - Continues polling until `deploy_timeout` (default: 30s) is reached
3. **Traffic switch** — once the new container passes healthchecks, all new requests route to it
4. **Drain** — in-flight requests to the old container finish naturally
5. **Old container stops** — after `drain_timeout` (default: 30s), remaining connections are closed
6. Old container is stopped and eventually pruned

### Tuning healthchecks

```yaml
proxy:
  healthcheck:
    path: /healthz           # your health endpoint (must return 200–399)
    interval: 1              # seconds between polls (default: 1)
    timeout: 5               # seconds per request (default: 5)

deploy_timeout: 60           # total seconds to wait for healthchecks to pass (default: 30)
readiness_delay: 5           # extra seconds before healthchecks start (default: 0)
drain_timeout: 30            # seconds to drain in-flight requests (default: 30)
```

### Healthcheck endpoint requirements

Your app MUST expose a health endpoint that:
- Returns HTTP 200–399 when healthy
- Responds quickly (< 1 second ideal)
- Does NOT require authentication

```javascript
// Example: Node/Express
app.get('/healthz', (req, res) => res.status(200).json({ status: 'ok' }))

// Example: Fastify
fastify.get('/healthz', async () => ({ status: 'ok' }))
```

---

## Rolling deploys

Roll out across a large fleet in batches to reduce risk.

```yaml
boot:
  limit: 2          # deploy N servers at a time (or "25%" for a percentage)
  wait: 10          # seconds to pause between batches
```

Examples:
```yaml
boot:
  limit: "25%"      # 25% of fleet per batch; minimum 1
  wait: 15
```

### What happens

1. Divide servers into batches of `limit`
2. Deploy first batch (boot → healthcheck → traffic switch)
3. Wait `wait` seconds
4. Deploy next batch
5. Repeat until all servers updated

If any host fails, the deploy stops (remaining hosts stay on old version).

### Monitoring a rolling deploy

```bash
# In another terminal:
kamal app version          # shows current version on each host
kamal app logs -f -p       # follow logs on primary
kamal details              # full status snapshot
```

---

## Rollback

### Find available versions

```bash
kamal app containers       # lists all containers per host with version tags; current is marked
kamal audit                # deployment history with timestamps
kamal app version          # currently running version
```

Output example from `kamal app containers`:
```
myapp-web-abc123d  (current)   running  2h ago
myapp-web-def456e              stopped  6h ago
myapp-web-789ghij              stopped  1d ago
```

### Roll back

```bash
kamal rollback abc123d                # roll back to a specific version hash
kamal rollback abc123d -r web         # roll back only the web role
kamal rollback abc123d -h 192.168.0.2 # roll back a specific host
```

Rollback runs the same hook sequence as deploy (`pre-deploy`, `pre-app-boot`, `post-app-boot`, `post-deploy`).

### Retention

Old containers are retained per `retain_containers` (default: 5). Once pruned, you cannot roll back to that version without rebuilding the image.

```yaml
retain_containers: 10    # keep more versions (useful for long QA cycles)
```

---

## Multi-server / multi-role

### Defining roles

```yaml
servers:
  web:
    hosts:
      - 192.168.0.1
      - 192.168.0.2
  worker:
    hosts:
      - 192.168.0.3
      - 192.168.0.4
    cmd: node worker.js
  scheduler:
    hosts:
      - 192.168.0.5
    cmd: node scheduler.js
```

### Targeting roles/hosts

```bash
kamal deploy -r web                           # deploy only web role
kamal deploy -r web,worker                    # multiple roles
kamal app exec -r worker "node queue-stats.js"
kamal app logs -r worker -f
kamal app stop -r scheduler                   # stop one role only
```

### Primary host

The primary host is the first host in `primary_role` (default: `web`). Used for `-p` / `--primary` targeting. Commands like `kamal app exec -p "db-migrate"` run only on this host.

```yaml
primary_role: web    # default
```

---

## Multiple environments (destinations)

### File structure

```
config/
  deploy.yml              # production (or base)
  deploy.staging.yml      # staging overrides
  deploy.eu.yml           # EU region overrides
.kamal/
  secrets                 # production secrets (base)
  secrets.staging         # staging secrets (merged on top of base)
  secrets.eu              # EU secrets
```

### Destination override file (only set what differs)

`config/deploy.staging.yml`:
```yaml
servers:
  web:
    - 10.0.0.10

proxy:
  host: staging.example.com
  ssl: true

env:
  clear:
    NODE_ENV: staging
    LOG_LEVEL: debug
```

### Usage

```bash
kamal deploy -d staging
kamal deploy -d eu
kamal setup -d staging
kamal rollback abc123 -d staging
kamal app logs -d staging -f
kamal config -d staging           # inspect merged config
```

### Tips

- Use `require_destination: true` in the base config to force the `-d` flag (prevents accidental production deploys)
- `KAMAL_DESTINATION` env var is available in hook scripts
- Destination secrets file: `.kamal/secrets.<destination>` (staging → `.kamal/secrets.staging`)

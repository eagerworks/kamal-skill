---
name: kamal
description: >
  Expert guide for setting up and managing Kamal deployments — the Basecamp tool for zero-downtime
  container deployments to bare-metal servers and VMs. Use this skill whenever the user: mentions
  Kamal, kamal-proxy, or Traefik-based deployments; wants to deploy a Dockerized app to a VPS, VM,
  or bare-metal server; is writing or editing a config/deploy.yml or .kamal/secrets file; asks about
  zero-downtime deploys, rolling deploys, rollbacks, or deploy hooks; needs to set up a Docker
  registry, builder, or SSH connection for deployments; asks about Kamal accessories (databases,
  Redis, etc.); is troubleshooting a failed deploy, healthcheck error, or lock issue; is upgrading
  from Kamal 1 to Kamal 2; or wants to run commands inside a running container. Also use when
  the user says things like "deploy my app without downtime", "deploy to my own server", or
  "set up SSL for my self-hosted app" — even if they don't name Kamal.
---

# Kamal Deployment Skill

Kamal is an open-source tool (by Basecamp/37signals) that deploys any Dockerized app to bare-metal servers or cloud VMs with zero downtime. It builds a Docker image, pushes it to a registry, pulls it on your servers, and hot-swaps traffic via a built-in proxy — no Kubernetes, no complex PaaS needed.

## Version Detection — Do This First

Before generating any config or running any command, determine which version is in play:

```bash
kamal version    # prints e.g. "2.10.1" or "1.9.2"
```

If Kamal isn't installed yet, infer from the repo:
- `traefik:` key in `config/deploy.yml` + `.env`/`.env.erb` file → **Kamal 1.x** → read `references/kamal-v1.md`
- `proxy:` key in `config/deploy.yml` + `.kamal/secrets` file → **Kamal 2.x**
- No existing config → **default to Kamal 2.x** (current; install with `gem install kamal`)

**Key v1 → v2 differences at a glance:**

| Area | Kamal 1.9.x | Kamal 2.x |
|---|---|---|
| Proxy | Traefik (`traefik:` block, raw labels/args) | kamal-proxy (`proxy:` block, `host`/`ssl`/`app_port`) |
| SSL | Manual Traefik ACME config or Cloudflare | Native Let's Encrypt (`ssl: true`) |
| Secrets/env | `.env` / `.env.erb` + `kamal env push` | `.kamal/secrets` + pushed on every deploy |
| Apps per host | Single-app oriented | Multiple apps on one server |
| Networking | Host network | Isolated custom Docker network |
| Default app port | `3000` | `80` |
| Hooks | `pre/post-traefik-reboot` | `pre/post-proxy-reboot` + `pre/post-app-boot` |
| Key commands | `kamal traefik`, `kamal env push`, `kamal envify` | `kamal proxy`, `kamal secrets` |

→ **If working with Kamal 1.x:** read `references/kamal-v1.md` for all v1-specific syntax.

---

## Reference Files (read these on demand)

| Task | Read |
|---|---|
| Writing or editing `config/deploy.yml` | `references/configuration.md` |
| Full CLI command list + flags | `references/commands.md` |
| First-time setup, deploy, redeploy, rollback, rolling deploys | `references/workflows.md` |
| `.kamal/secrets`, vault adapters, deploy hooks | `references/secrets-and-hooks.md` |
| `proxy:` block, Let's Encrypt SSL, custom certs | `references/proxy-and-ssl.md` |
| Build strategies, remote builder, multiarch, asset bridging | `references/builders.md` |
| Healthcheck failures, lock errors, logs, debugging | `references/troubleshooting.md` |
| Everything Kamal 1.9.x + upgrading to v2 | `references/kamal-v1.md` |

Copyable starter templates live in `assets/`:
- `assets/deploy.yml` — annotated v2 starter config
- `assets/deploy.v1.yml` — annotated v1 starter config  
- `assets/secrets.example` — `.kamal/secrets` template
- `assets/hooks/` — sample pre-connect, pre-deploy, post-deploy hooks

---

## First-Time Setup (Kamal 2)

```bash
# 1. Install
gem install kamal          # requires Ruby 3.0+

# 2. Scaffold
kamal init                 # creates config/deploy.yml, .kamal/secrets, .kamal/hooks/

# 3. Edit config/deploy.yml — see assets/deploy.yml for an annotated starter
#    Minimum required: service, image, servers, registry

# 4. Fill in .kamal/secrets — see assets/secrets.example
echo "KAMAL_REGISTRY_PASSWORD=your-token" >> .kamal/secrets
# NEVER commit .kamal/secrets — add to .gitignore:
echo ".kamal/secrets*" >> .gitignore
echo "!.kamal/secrets.example" >> .gitignore

# 5. Validate merged config (shows resolved values including secrets — careful in shared terminals)
kamal config

# 6. Bootstrap servers + first deploy (installs Docker if needed, boots accessories, deploys app)
kamal setup
```

**Server prerequisites:** SSH access (key-based, root or sudo user), curl/wget, internet access. Kamal installs Docker automatically via `kamal setup` or `kamal server bootstrap`.

---

## Core Management Cheatsheet

```bash
# Deploy (build + push + zero-downtime swap)
kamal deploy
kamal deploy --skip-push          # skip build, use existing image
kamal deploy -r web               # target a specific role
kamal deploy -h 192.168.0.1       # target a specific host
kamal redeploy                    # faster: skips proxy start, prune, registry login

# Rollback
kamal app containers              # list versions (current is marked)
kamal rollback <VERSION>          # e.g. kamal rollback abc123d

# Logs
kamal app logs -f                 # follow live logs
kamal app logs --since 30m        # last 30 minutes
kamal app logs --grep ERROR       # filter by pattern

# Run commands in the container
kamal app exec -i "bash"          # interactive shell
kamal app exec -i --reuse "bash"  # reuse existing container (faster)
kamal app exec "db-migrate-command"

# App status
kamal app version                 # currently running version
kamal app containers              # containers on all hosts
kamal details                     # all containers + proxy details

# Accessories (databases, redis, etc.)
kamal accessory boot mysql
kamal accessory reboot mysql      # apply config/image changes
kamal accessory logs mysql -f
kamal accessory exec mysql -i "mysql -u root -p"

# Proxy
kamal proxy logs                  # proxy logs
kamal proxy reboot                # restart kamal-proxy (brief interruption)
kamal proxy details

# Deploy lock (release if stuck after a crash)
kamal lock status
kamal lock release

# Audit & history
kamal audit                       # deployment history from servers
```

---

## Critical Gotchas

1. **Never commit secrets.** Add `.kamal/secrets*` (and allow `!.kamal/secrets.example`) to `.gitignore`. The file holds plaintext credentials.

2. **`password:` is always an array of secret names — not a string.**
   ```yaml
   # ✅ correct
   registry:
     password:
       - KAMAL_REGISTRY_PASSWORD
   # ❌ wrong — will fail
   registry:
     password: "my-token"
   ```

3. **Bind accessory ports to localhost.** Use `"127.0.0.1:5432:5432"` not `"5432:5432"` — the latter exposes your database to the internet.

4. **SSL + forward_headers.** When `ssl: true`, the proxy does NOT forward `X-Forwarded-For` / `X-Forwarded-Proto` by default. Add `forward_headers: true` or your app won't see the real client IP / will think it's on HTTP.

5. **Rollback requires the old container to exist.** Old containers are retained by default (5 most recent). Once pruned, you can't roll back to that version without rebuilding.

6. **ECR tokens expire every 12 hours.** Use the ERB trick in registry config (see `references/configuration.md` → Registry).

7. **`kamal setup` vs `kamal deploy`.** `setup` is for first time — it installs Docker, boots accessories, then deploys. `deploy` is for subsequent releases and assumes infra is ready.

8. **Disable buffering for streaming/WebSockets.** Add `buffering: { requests: false, responses: false }` under `proxy:` for SSE/WebSocket endpoints.

9. **v1 users:** The default app port in v1 was `3000`; v2 defaults to `80`. When upgrading, update your `Dockerfile`/`proxy.app_port` accordingly.

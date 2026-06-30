# Kamal 2 — Complete Command Reference

> This file covers **Kamal 2.x** only. For Kamal 1.x commands (including `kamal traefik`, `kamal env push`, `kamal envify`), see `kamal-v1.md`.

## Global Flags

These flags apply to almost every command:

| Flag | Short | Description |
|---|---|---|
| `--verbose` | `-v` | Detailed logging |
| `--quiet` | `-q` | Errors only |
| `--primary` | `-p` | Run only on the primary host |
| `--hosts=HOSTS` | `-h` | Comma-separated host list (supports `*` wildcard) |
| `--roles=ROLES` | `-r` | Comma-separated role list |
| `--config-file=PATH` | `-c` | Config file path (default: `config/deploy.yml`) |
| `--destination=DEST` | `-d` | Destination (e.g. `staging`, `production`) |
| `--skip-hooks` | `-H` | Skip all hooks |
| `--version=VERSION` | | Target a specific app version |

---

## Top-Level Commands

| Command | Description |
|---|---|
| `kamal init` | Scaffold `config/deploy.yml`, `.kamal/secrets`, and sample hooks |
| `kamal setup` | **First-time bootstrap**: installs Docker, boots accessories, deploys app |
| `kamal deploy` | **Full deploy**: build → push → zero-downtime swap → prune |
| `kamal redeploy` | **Fast deploy**: skips proxy boot, prune, registry login |
| `kamal rollback [VERSION]` | Roll back to a previous version |
| `kamal details` | Show all container details (app + proxy + accessories) |
| `kamal audit` | Show deployment history from servers |
| `kamal config` | Print merged resolved config (includes secrets — use carefully) |
| `kamal docs [SECTION]` | Show built-in configuration documentation |
| `kamal version` | Print installed Kamal version |
| `kamal remove` | Remove app, proxy, accessories, and registry session from servers |
| `kamal upgrade` | Migrate from Kamal 1.x to 2.x |
| `kamal help [COMMAND]` | Show help |

### `kamal deploy` flags

```bash
kamal deploy                     # full deploy
kamal deploy --skip-push         # skip build; pull existing image
kamal deploy -r web              # deploy only the web role
kamal deploy -h 192.168.0.1      # deploy to a specific host
kamal deploy -H                  # skip hooks
kamal deploy -d staging          # deploy to staging destination
kamal deploy -v                  # verbose output
```

---

## `kamal app` — Application Container Management

| Command | Description |
|---|---|
| `kamal app boot` | Boot (or reboot) the app on servers |
| `kamal app start` | Start existing stopped containers |
| `kamal app stop` | Stop running app containers |
| `kamal app details` | Show app container details |
| `kamal app exec [CMD]` | Run a command inside the app container |
| `kamal app logs` | Show app logs |
| `kamal app images` | List app images on servers |
| `kamal app containers` | List app containers (and versions) on servers |
| `kamal app version` | Show currently running version |
| `kamal app stale_containers` | Detect stale/orphaned containers |
| `kamal app remove` | Remove app containers and images |

### `kamal app exec` flags

```bash
kamal app exec "node -e 'console.log(process.env)'"
kamal app exec -i "sh"                    # -i/--interactive: open a shell
kamal app exec -i --reuse "sh"           # reuse existing container (faster, no new spawn)
kamal app exec -p "node migrate.js"      # run on primary only
kamal app exec -r worker "node queue.js" # run in worker role
kamal app exec -h 192.168.0.1 "command" # run on specific host
kamal app exec --version abc123 "sh"     # run in a specific version's container
```

### `kamal app logs` flags

```bash
kamal app logs -f                   # follow (live tail)
kamal app logs --since 30m          # last 30 minutes (e.g. 30m, 1h, 2h)
kamal app logs -n 100               # last 100 lines
kamal app logs --grep "ERROR"       # filter lines
kamal app logs --grep error --grep-options "-i"   # case-insensitive
kamal app logs -T                   # skip timestamps
kamal app logs -q                   # no host prefixes
kamal app logs -r web               # filter by role
kamal app logs -h 192.168.0.1       # filter by host
kamal app logs -p                   # primary only (required with -f on multi-host)
```

> ⚠️ `--follow` / `-f` requires `--primary` or `--hosts` on multi-host setups.

---

## `kamal accessory` — Side Service Management

Accessories (databases, caches, etc.) are managed independently from the app. They do **not** get zero-downtime restarts.

| Command | Description |
|---|---|
| `kamal accessory boot [NAME]` | Boot a new accessory (use `all` for all accessories) |
| `kamal accessory reboot [NAME]` | Stop → remove → start (applies image/config changes) |
| `kamal accessory start [NAME]` | Start an existing stopped accessory |
| `kamal accessory stop [NAME]` | Stop a running accessory |
| `kamal accessory restart [NAME]` | Restart an accessory |
| `kamal accessory details [NAME]` | Show container details |
| `kamal accessory logs [NAME]` | Show accessory logs |
| `kamal accessory exec [NAME] [CMD]` | Run a command in the accessory |
| `kamal accessory remove [NAME]` | Remove container, image, and data directory |

```bash
kamal accessory boot db
kamal accessory boot all          # boot all accessories
kamal accessory reboot redis      # apply new redis image or config
kamal accessory logs db -f
kamal accessory exec db -i "psql -U myapp"
kamal accessory remove db         # ⚠️ removes data directory too
```

---

## `kamal proxy` — Proxy Management

| Command | Description |
|---|---|
| `kamal proxy boot` | Boot kamal-proxy on servers |
| `kamal proxy reboot` | Stop → remove → start proxy (brief downtime) |
| `kamal proxy start` | Start existing stopped proxy |
| `kamal proxy stop` | Stop running proxy |
| `kamal proxy restart` | Restart proxy |
| `kamal proxy details` | Show proxy container details |
| `kamal proxy logs` | Show proxy logs |
| `kamal proxy remove` | Remove proxy container and image |
| `kamal proxy boot_config get` | Show current proxy boot config |
| `kamal proxy boot_config set` | Update proxy boot config |
| `kamal proxy boot_config reset` | Reset proxy boot config to defaults |

```bash
kamal proxy reboot                          # restart after config change
kamal proxy reboot --rolling               # staged restart (less downtime)
kamal proxy logs -f
kamal proxy boot_config set --http-port 80 --https-port 443 --log-max-size 50m
```

---

## `kamal build` — Image Build Management

| Command | Description |
|---|---|
| `kamal build deliver` | Build, push to registry, then pull on all servers |
| `kamal build push` | Build and push image to registry |
| `kamal build pull` | Pull image from registry onto servers |
| `kamal build create` | Create build infrastructure |
| `kamal build details` | Show build infrastructure details |
| `kamal build dev` | Build from working dir (tagged `dirty`), push to local Docker store |
| `kamal build remove` | Tear down build infrastructure |

---

## `kamal registry` — Registry Authentication

| Command | Description |
|---|---|
| `kamal registry login` | Log in to registry (locally and on servers) |
| `kamal registry logout` | Log out of registry (locally and on servers) |
| `kamal registry setup` | Set up registry access |
| `kamal registry remove` | Remove registry access |

---

## `kamal server` — Server Management

| Command | Description |
|---|---|
| `kamal server bootstrap` | Install Docker on servers (via get.docker.com) |
| `kamal server exec [CMD]` | Run a shell command directly on servers |

```bash
kamal server bootstrap            # install Docker on all servers
kamal server exec -i "/bin/bash"  # interactive shell on all servers
kamal server exec -p "df -h"      # check disk space on primary
```

---

## `kamal secrets` — Secrets Management

Integrates with external secret managers. Values are fetched and injected into the environment locally before deploying.

| Command | Description |
|---|---|
| `kamal secrets fetch [KEYS...]` | Fetch secrets from a vault |
| `kamal secrets extract KEY` | Extract a single secret from a batch fetch result |
| `kamal secrets print` | Print resolved secrets (debugging) |

```bash
# 1Password
kamal secrets fetch --adapter 1password --account myaccount --from "Kamal/production" DB_PASSWORD SESSION_SECRET

# AWS Secrets Manager
kamal secrets fetch --adapter aws_secrets_manager --from "prod/myapp"

# Pattern for .kamal/secrets file
SECRETS=$(kamal secrets fetch --adapter 1password --account myaccount --from "Kamal/production")
DATABASE_PASSWORD=$(kamal secrets extract DATABASE_PASSWORD $SECRETS)
```

Supported adapters: `1password`, `lastpass`, `bitwarden`, `bitwarden-sm`, `aws_secrets_manager`, `doppler`, `gcp`, `passbolt`

---

## `kamal lock` — Deploy Lock

Kamal uses a distributed lock to prevent concurrent deploys. If a deploy crashes, the lock may need manual release.

| Command | Description |
|---|---|
| `kamal lock acquire -m "MESSAGE"` | Manually acquire the deploy lock |
| `kamal lock release` | Release a stuck deploy lock |
| `kamal lock status` | Show current lock status |

```bash
kamal lock status       # check if locked
kamal lock release      # release after crashed deploy
```

---

## `kamal prune` — Cleanup

| Command | Description |
|---|---|
| `kamal prune all` | Remove unused images and stopped containers |
| `kamal prune images` | Remove unused images only |
| `kamal prune containers` | Remove old stopped containers (keeps last N per `retain_containers`) |

---

## Targeting Tips

```bash
# By role
kamal app logs -r web
kamal deploy -r web,worker

# By host
kamal app exec -h "192.168.0.*" "command"    # wildcard
kamal app exec -h 192.168.0.1,192.168.0.2 "command"

# Primary only (first host in primary_role)
kamal app exec -p "node migrate.js"
kamal app logs -p -f

# Specific version
kamal app exec --version abc123d "sh"
```

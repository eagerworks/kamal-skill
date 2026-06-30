# Kamal 2 — Troubleshooting Guide

## Quick Diagnostic Checklist

```bash
kamal app version        # what's currently running?
kamal app containers     # all containers + their states on all hosts
kamal app logs -f        # live logs
kamal proxy details      # is kamal-proxy running?
kamal proxy logs         # proxy errors
kamal details            # full snapshot: app + proxy + accessories
kamal config             # resolved config (sanity check your YAML + secrets)
kamal audit              # deployment history
```

---

## Common Issues

### 1. Healthcheck failure: container not ready

```
Error: Container failed to become healthy (attempt 30/30)
Error: Healthcheck::Error: container not ready after 30 seconds
```

**Causes and fixes:**

**a) App is slow to start:**
```yaml
deploy_timeout: 120        # increase total wait time
readiness_delay: 10        # add initial wait before polling starts
```

**b) Wrong healthcheck path:**
```yaml
proxy:
  healthcheck:
    path: /healthz         # must match your app's health endpoint
```
Verify: `kamal app exec --reuse "curl -f http://localhost:3000/healthz"`

**c) Wrong app_port:**
```yaml
proxy:
  app_port: 3000           # must match what your container EXPOSE / listens on
```
Verify: `kamal app exec --reuse "ss -tlnp"` or `"netstat -tlnp"`

**d) App is crashing on startup:**
```bash
kamal app logs --since 5m
kamal app exec -i "sh"     # try to start container manually and inspect
```

**e) App needs environment variables that aren't set:**
```bash
kamal config               # check that all secret values are populated
kamal secrets print        # see what secrets resolved to
```

---

### 2. Deploy lock is stuck

```
Error: LockError: Deploy lock found — another deploy may be running
```

This happens when a previous deploy crashed without releasing the lock.

```bash
kamal lock status           # confirm lock is stale
kamal lock release          # release the lock
```

⚠️ Only release if you're certain no other deploy is actively running.

---

### 3. kamal-proxy version too old

```
Error: kamal-proxy version X.Y.Z is too old, run kamal proxy reboot
```

```bash
kamal proxy reboot          # brief downtime (~seconds)
```

For zero-downtime proxy upgrade (rolling reboot):
```bash
kamal proxy reboot --rolling
```

---

### 4. Registry authentication failure

```
Error: unauthorized: authentication required
Error: denied: access forbidden
```

Fixes:
```bash
kamal registry login        # re-authenticate registry on all hosts

# Check your secret value
kamal secrets print | grep KAMAL_REGISTRY_PASSWORD

# Verify the token/PAT has the right permissions:
# Docker Hub: "Read & Write" scope
# GHCR: "write:packages" scope
# ECR: "ecr:GetLoginPassword" IAM permission
```

ECR tokens expire every 12 hours — use ERB to auto-refresh:
```yaml
registry:
  server: 123456789012.dkr.ecr.us-east-1.amazonaws.com
  username: AWS
  password: <%= `aws ecr get-login-password --region us-east-1` %>
```

---

### 5. SSH connection failure

```
Error: SSH authentication failed
Error: Connection refused (connect(2) for "192.168.0.1" port 22)
```

Diagnose:
```bash
ssh user@192.168.0.1                    # test direct connection
ssh-add -l                              # check loaded SSH keys
ssh-add ~/.ssh/id_ed25519              # load your key

# In deploy.yml:
ssh:
  user: deploy       # verify correct user
  port: 22           # verify correct port
  keys:
    - ~/.ssh/id_ed25519
```

Copy key to server:
```bash
ssh-copy-id deploy@192.168.0.1
```

---

### 6. Port 80/443 already in use

```
Error: failed to bind port 80: address already in use
```

Something else is listening on port 80 or 443 (e.g. nginx, old Traefik from v1).

```bash
# On the server:
kamal server exec "ss -tlnp | grep -E '80|443'"
kamal server exec "docker ps"   # check for stale containers

# Stop old Traefik (if upgrading from v1):
kamal server exec "docker stop kamal-traefik 2>/dev/null; docker rm kamal-traefik 2>/dev/null"
```

---

### 7. Image not found / push failed

```
Error: image not found: myuser/myapp:abc123d
Error: push access denied, repository does not exist
```

```bash
kamal build push               # rebuild and push
kamal registry login           # re-auth registry
kamal build details            # check builder configuration
```

---

### 8. Docker not installed on server

```
Error: docker: command not found
```

```bash
kamal server bootstrap         # auto-installs Docker on all servers
# or
kamal setup                    # also auto-installs Docker
```

---

### 9. Out of disk space

```
Error: no space left on device
```

```bash
kamal prune all                           # remove old containers + images
kamal server exec "df -h"                 # check disk usage
kamal server exec "docker system prune -f"  # aggressive cleanup
```

Reduce retained containers (default: 5):
```yaml
retain_containers: 2
```

---

### 10. Accessory boot failure

```
Error: accessory "db" failed to start
```

```bash
kamal accessory details db     # container state
kamal accessory logs db        # startup errors
kamal accessory remove db      # remove and try again
kamal accessory boot db
```

Common cause: volume permissions or wrong env vars. Check:
```bash
kamal accessory exec db -i "sh"    # inspect inside the container
```

---

## Debugging Techniques

### Verbose mode

Add `-v` / `--verbose` to any command for detailed SSH + Docker output:

```bash
kamal deploy -v
kamal setup -v
kamal app boot -v
```

### Inspect config

```bash
kamal config                    # full resolved config + secrets
kamal config -d staging         # for a specific destination
```

### Run commands inside containers

```bash
kamal app exec -i "sh"                    # interactive shell (new container)
kamal app exec -i --reuse "sh"           # attach to existing container
kamal app exec -p "env | sort"           # dump all env vars on primary
kamal app exec "node -e 'console.log(process.env.DATABASE_URL)'"
```

### Inspect the running container

```bash
kamal app details                        # docker inspect output
kamal app containers                     # all containers + status
kamal app stale_containers               # find orphaned containers
```

### Network debugging

```bash
# Test connectivity to an accessory from the app
kamal app exec --reuse "nc -zv 192.168.0.5 5432"    # test DB connectivity
kamal app exec --reuse "curl -f http://localhost:3000/healthz"
```

### Server-level debugging

```bash
kamal server exec "docker ps -a"              # all containers on servers
kamal server exec "docker network ls"         # Docker networks
kamal server exec "docker logs kamal-proxy"   # proxy logs via docker directly
kamal server exec -i "bash"                   # interactive shell on all servers
```

---

## Reading Logs

```bash
# App logs
kamal app logs -f                        # follow (live)
kamal app logs --since 30m              # last 30 minutes
kamal app logs -n 200                   # last 200 lines
kamal app logs --grep "ERROR"           # filter
kamal app logs --grep error --grep-options "-i"    # case-insensitive
kamal app logs -r web                   # specific role
kamal app logs -h 192.168.0.1          # specific host
kamal app logs -p -f                    # primary only (required for -f on multi-host)
kamal app logs -T                       # no timestamps
kamal app logs -q                       # no host prefixes (clean output)

# Proxy logs
kamal proxy logs -f
kamal proxy logs -n 100

# Accessory logs
kamal accessory logs db -f
kamal accessory logs redis --since 1h
```

---

## Still stuck?

1. Check `kamal audit` for the deployment history — does the last deploy show success?
2. Run with `-v` flag for verbose SSH + Docker output
3. SSH directly to the server and run `docker logs <container-id>` to see raw container output
4. Check `kamal config` to verify secrets are being populated correctly
5. If upgrading from v1, see `references/kamal-v1.md` → "Upgrading to v2" section

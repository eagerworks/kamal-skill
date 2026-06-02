# Kamal 2 — Secrets & Hooks

## Table of Contents

1. [.kamal/secrets file](#kamal-secrets-file)
2. [env block (clear vs secret)](#env-block-clear-vs-secret)
3. [External secret managers](#external-secret-managers)
4. [Builder secrets](#builder-secrets)
5. [Hooks overview](#hooks-overview)
6. [Available hooks](#available-hooks)
7. [Hook environment variables](#hook-environment-variables)
8. [Sample hook scripts](#sample-hook-scripts)

---

## .kamal/secrets file

`.kamal/secrets` is a **dotenv-format** file that holds secrets on your local machine. It is read before every deploy and secrets are pushed (encrypted over SSH) to servers.

```bash
# Format: KEY=value
KAMAL_REGISTRY_PASSWORD=your-registry-token
DATABASE_PASSWORD=super-secret-db-password
SESSION_SECRET=your-session-secret

# Reference shell environment variables
KAMAL_REGISTRY_PASSWORD=$GITHUB_TOKEN

# Execute shell commands (e.g. read from a file)
RAILS_MASTER_KEY=$(cat config/master.key)
APP_SECRET=$(cat .app_secret)
```

### Security rules

```bash
# ALWAYS gitignore the secrets file
echo ".kamal/secrets*" >> .gitignore
echo "!.kamal/secrets.example" >> .gitignore   # allow an example template

# Restrict permissions
chmod 600 .kamal/secrets

# Commit a template for reference
cp .kamal/secrets .kamal/secrets.example
# Then scrub all real values in .kamal/secrets.example
```

### Destination-specific secrets

Secrets are merged: base → destination.

```
.kamal/secrets             # base (production or shared)
.kamal/secrets.staging     # staging overrides
.kamal/secrets.eu          # EU region overrides
```

When you run `kamal deploy -d staging`, Kamal loads both `.kamal/secrets` and `.kamal/secrets.staging`.

### Debugging

```bash
kamal secrets print        # print resolved secrets (⚠️ shows plaintext — use carefully)
kamal config               # shows full resolved config including secret values
```

---

## env block (clear vs secret)

```yaml
env:
  clear:            # in-plain-text, committed to repo
    NODE_ENV: production
    PORT: "3000"
    DB_HOST: 192.168.0.5
  secret:           # variable NAMES only — values from .kamal/secrets
    - DATABASE_PASSWORD
    - SESSION_SECRET
    - API_KEY
```

**Secret variables are NOT logged, NOT visible in `docker inspect`.**

### Aliased secrets

Map a different env var name in the container to a differently-named secret:

```yaml
env:
  secret:
    - DB_PASS:PROD_DATABASE_PASSWORD    # container sees DB_PASS; value comes from PROD_DATABASE_PASSWORD
    - CACHE_TOKEN:REDIS_AUTH_TOKEN
```

### Precedence (later wins)

root clear → root secret → role clear → role secret → tag clear → tag secret

---

## External secret managers

Use `kamal secrets fetch` to pull secrets from a vault into your local environment, then reference them in `.kamal/secrets`:

```bash
# .kamal/secrets — using 1Password
SECRETS=$(kamal secrets fetch --adapter 1password --account myaccount --from "Kamal/production")
DATABASE_PASSWORD=$(kamal secrets extract DATABASE_PASSWORD $SECRETS)
SESSION_SECRET=$(kamal secrets extract SESSION_SECRET $SECRETS)
KAMAL_REGISTRY_PASSWORD=$(kamal secrets extract KAMAL_REGISTRY_PASSWORD $SECRETS)
```

### Supported adapters

| Adapter | `--adapter` flag |
|---|---|
| 1Password | `1password` |
| Bitwarden | `bitwarden` |
| Bitwarden Secrets Manager | `bitwarden-sm` |
| LastPass | `lastpass` |
| AWS Secrets Manager | `aws_secrets_manager` |
| Doppler | `doppler` |
| Google Secret Manager | `gcp` |
| Passbolt | `passbolt` |
| HashiCorp Vault | `vault` |

### Examples

```bash
# AWS Secrets Manager — fetch individual keys
kamal secrets fetch \
  --adapter aws_secrets_manager \
  --from prod/myapp \
  DATABASE_PASSWORD SESSION_SECRET

# In .kamal/secrets:
DATABASE_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id prod/db \
  --query SecretString \
  --output text)

# Google Secret Manager
DATABASE_PASSWORD=$(gcloud secrets versions access latest --secret="db-password")

# HashiCorp Vault
DATABASE_PASSWORD=$(vault kv get -field=password secret/myapp/db)

# Azure Key Vault
DATABASE_PASSWORD=$(az keyvault secret show \
  --name db-password \
  --vault-name myapp-kv \
  --query value -o tsv)
```

---

## Builder secrets

Build-time secrets (mounted at build time via `docker build --secret`, NOT baked into layers):

```yaml
builder:
  secrets:
    - GITHUB_TOKEN
    - NPM_TOKEN
```

In your Dockerfile:
```dockerfile
RUN --mount=type=secret,id=GITHUB_TOKEN \
    GITHUB_TOKEN=$(cat /run/secrets/GITHUB_TOKEN) \
    npm install --prefer-offline
```

Builder secrets come from `.kamal/secrets`, same as runtime secrets.

---

## Hooks overview

Hooks are scripts in `.kamal/hooks/` that run at specific points in the deploy lifecycle.

**Rules:**
- Filename = hook name, **no file extension** (`pre-deploy`, not `pre-deploy.sh`)
- Must be **executable** (`chmod +x .kamal/hooks/*`)
- Non-zero exit code **aborts** the deploy
- Run locally on the machine running Kamal (not on remote servers)
- Only `pre-connect` auto-loads `.kamal/secrets`; other hooks may need to source it manually

```bash
ls .kamal/hooks/
# pre-connect  pre-build  pre-deploy  post-deploy  ...

chmod +x .kamal/hooks/*
```

Skip all hooks:
```bash
kamal deploy -H    # or --skip-hooks
```

Control output verbosity (`config/deploy.yml`):
```yaml
hooks_output: verbose      # or: quiet
# Per-hook:
hooks_output:
  pre-deploy: verbose
  post-deploy: quiet
```

---

## Available hooks

All hooks available in Kamal 2.x (canonical, from kamal-deploy.org):

| Hook | When it runs |
|---|---|
| `docker-setup` | After Docker is confirmed available on each server |
| `pre-connect` | Before SSH connections are established (auto-loads secrets) |
| `pre-build` | Before the Docker image build begins |
| `pre-deploy` | After build/push, before containers swap |
| `pre-app-boot` | Just before new app containers are started |
| `post-app-boot` | Just after new app containers start (before traffic switches) |
| `post-deploy` | After traffic has switched to new containers |
| `pre-proxy-reboot` | Before kamal-proxy is rebooted |
| `post-proxy-reboot` | After kamal-proxy is rebooted |

---

## Hook environment variables

All hooks receive these `KAMAL_*` environment variables:

| Variable | Description |
|---|---|
| `KAMAL_RECORDED_AT` | UTC ISO 8601 timestamp of the deploy |
| `KAMAL_PERFORMER` | Local user running Kamal |
| `KAMAL_SERVICE` | Service name (from `service:` in deploy.yml) |
| `KAMAL_VERSION` | Full version being deployed (git SHA or VERSION env) |
| `KAMAL_SERVICE_VERSION` | Abbreviated service + version |
| `KAMAL_HOSTS` | Comma-separated list of target hosts |
| `KAMAL_COMMAND` | Kamal command being run (e.g. `deploy`) |
| `KAMAL_SUBCOMMAND` | Subcommand if applicable (optional) |
| `KAMAL_DESTINATION` | Destination name if `-d` was used (optional) |
| `KAMAL_ROLE` | Role being targeted (optional) |

Build-related hooks (`pre-build`, `post-build`) also receive:
- `KAMAL_RECORDED_IMAGE` — full image reference including tag
- `KAMAL_REGISTRY` — registry hostname

---

## Sample hook scripts

### `pre-deploy` — Slack/webhook notification

```bash
#!/usr/bin/env bash
set -euo pipefail

WEBHOOK="${SLACK_DEPLOY_WEBHOOK:-}"
if [[ -n "$WEBHOOK" ]]; then
  curl -s -X POST "$WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{
      \"text\": \"🚀 Deploying *${KAMAL_SERVICE}* version \`${KAMAL_VERSION}\` to ${KAMAL_DESTINATION:-production}\",
      \"channel\": \"#deployments\"
    }"
fi
```

### `post-deploy` — Run database migrations

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Running migrations for ${KAMAL_SERVICE} v${KAMAL_VERSION}..."
kamal app exec --primary "node migrate.js"    # replace with your migration command
echo "Migrations complete."
```

### `pre-connect` — Check required env vars

```bash
#!/usr/bin/env bash
# pre-connect auto-loads .kamal/secrets, so secrets are available here
set -euo pipefail

required_vars=(
  "KAMAL_REGISTRY_PASSWORD"
  "DATABASE_PASSWORD"
  "SESSION_SECRET"
)

missing=0
for var in "${required_vars[@]}"; do
  if [[ -z "${!var:-}" ]]; then
    echo "ERROR: Required secret '${var}' is not set in .kamal/secrets"
    missing=1
  fi
done

if [[ $missing -eq 1 ]]; then
  echo "Set missing secrets in .kamal/secrets and retry."
  exit 1
fi

echo "All required secrets are present ✓"
```

### `post-deploy` — Health verification

```bash
#!/usr/bin/env bash
set -euo pipefail

HOST="https://${KAMAL_SERVICE}.example.com"
echo "Verifying deployment at ${HOST}..."

for i in 1 2 3; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${HOST}/healthz" || echo "000")
  if [[ "$STATUS" == "200" ]]; then
    echo "Health check passed (attempt ${i}) ✓"
    exit 0
  fi
  echo "Attempt ${i}: got HTTP ${STATUS}, retrying in 5s..."
  sleep 5
done

echo "Health check failed after 3 attempts — investigate with: kamal app logs -f"
exit 1
```

# kamal-skill

A Claude Code skill for setting up and managing [Kamal](https://kamal-deploy.org) deployments — covering Kamal 2.x and 1.9.x.

## What it covers

- Zero-downtime deploys, rolling deploys, and rollbacks
- First-time setup (`kamal setup`) through day-to-day operations
- `kamal-proxy` configuration, Let's Encrypt SSL, and custom certificates
- Secrets management (`.kamal/secrets`, vault adapters: 1Password, AWS, GCP, Vault, and more)
- Accessories (databases, Redis, etc.)
- Deploy hooks with sample scripts
- Remote builders and multiarch builds (Apple Silicon → amd64)
- Asset bridging to prevent 404s during rolling deploys
- Multi-environment deployments (destinations)
- Full Kamal 1.9.x syntax + step-by-step v1 → v2 upgrade guide

## Repo structure

```
SKILL.md                        # hub: version detection, setup workflow, cheatsheet, gotchas
references/
  configuration.md              # complete deploy.yml key reference with full example
  commands.md                   # every CLI command, subcommand, and flag
  workflows.md                  # setup / deploy / rollback / rolling / multi-server
  secrets-and-hooks.md          # .kamal/secrets, vault adapters, hooks + sample scripts
  proxy-and-ssl.md              # kamal-proxy, Let's Encrypt, streaming, per-role proxy
  builders.md                   # build strategies, remote builder, multiarch, asset bridging
  troubleshooting.md            # 10 common issues, debugging techniques, log commands
  kamal-v1.md                   # Kamal 1.9.x syntax + v1→v2 upgrade procedure
assets/
  deploy.yml                    # annotated v2 starter config (framework-agnostic)
  deploy.v1.yml                 # annotated v1 starter config (Traefik block)
  secrets.example               # .kamal/secrets template with vault adapter patterns
  hooks/                        # executable pre-connect, pre-deploy, post-deploy samples
evals/
  evals.json                    # eval test cases
```

## Usage

### Claude Code

Copy (or symlink) the skill into your Claude skills directory:

```bash
cp -r /path/to/kamal-skill ~/.claude/skills/kamal

# or as a symlink (changes in the repo reflect immediately):
ln -s /path/to/kamal-skill ~/.claude/skills/kamal
```

Claude will then load and apply the skill automatically whenever you mention Kamal, `deploy.yml`, zero-downtime container deploys, rolling deploys, deploying a Dockerized app to a VPS, and more — even if you don't say "Kamal" explicitly.

### Outside Claude Code

The content is plain markdown — it works in any Claude interface or with any LLM:

| Interface | How to use |
|---|---|
| **Claude.ai** | Paste `SKILL.md` + the relevant `references/*.md` into your conversation |
| **Claude API** | Include the files as system-prompt context |
| **Any LLM / standalone docs** | The `references/` files are plain Kamal documentation — use them directly |

## Kamal version support

The skill defaults to **Kamal 2.x** (current). Full Kamal 1.9.x syntax — including the `traefik:` block, `.env`-based secrets, `kamal env push`, and the complete v1→v2 upgrade procedure — is in [`references/kamal-v1.md`](references/kamal-v1.md).

## License

MIT © [Eagerworks](https://eagerworks.com)

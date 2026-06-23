# kamal-skill

A portable agent skill for setting up and managing [Kamal](https://kamal-deploy.org) deployments — covering Kamal 2.x and 1.9.x. Works with Claude Code, Cursor, GitHub Copilot, Codex, Amp, and any agentic coding tool that can read markdown files.

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

## Install

The universal pattern is the same for every tool:

1. **Vendor the skill** into your project (or a global location).
2. **Add a small pointer** in your tool's rules/instructions file so the agent knows to load `SKILL.md` when Kamal is relevant.

The agent opens `references/*.md` files on demand, so the pointer stays tiny while the full knowledge base is always available.

### Claude Code

**Global install** (available in every project):

```bash
# copy:
cp -r /path/to/kamal-skill ~/.claude/skills/kamal

# or symlink (changes in the repo reflect immediately):
ln -s /path/to/kamal-skill ~/.claude/skills/kamal
```

**Project-level install** (committed to the repo, shared with your team):

```bash
mkdir -p .claude/skills
cp -r /path/to/kamal-skill .claude/skills/kamal
# or: ln -s /path/to/kamal-skill .claude/skills/kamal
```

Claude Code reads the `name` and `description` frontmatter in `SKILL.md` and loads the skill automatically — no extra pointer file needed.

### Cursor

First, vendor the skill into your project:

```bash
git submodule add https://github.com/your-org/kamal-skill tools/kamal-skill
# or: cp -r /path/to/kamal-skill tools/kamal-skill
```

Then create `.cursor/rules/kamal.mdc`:

```markdown
---
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
alwaysApply: false
---

Read @tools/kamal-skill/SKILL.md for Kamal deployment guidance.
Load the matching file from @tools/kamal-skill/references/ on demand as needed.
```

Cursor matches the `description` to incoming requests and loads the rule automatically — no need to mention Kamal explicitly.

### GitHub Copilot

Vendor the skill first (same as above):

```bash
git submodule add https://github.com/your-org/kamal-skill tools/kamal-skill
# or: cp -r /path/to/kamal-skill tools/kamal-skill
```

**Option A — always-on** (applies to every file): append to `.github/copilot-instructions.md`:

```markdown
## Kamal deployments

When helping with Kamal deployments, read `tools/kamal-skill/SKILL.md` for version detection,
setup workflow, a command cheatsheet, and critical gotchas. Load the relevant file from
`tools/kamal-skill/references/` on demand (configuration, commands, workflows, secrets-and-hooks,
proxy-and-ssl, builders, troubleshooting, kamal-v1).
```

**Option B — path-scoped** (triggers only for Kamal files): create `.github/instructions/kamal.instructions.md`:

```markdown
---
applyTo: "**/deploy.yml,**/.kamal/**,**/kamal/**"
---

When helping with Kamal deployments, read `tools/kamal-skill/SKILL.md` for version detection,
setup workflow, a command cheatsheet, and critical gotchas. Load the relevant file from
`tools/kamal-skill/references/` on demand (configuration, commands, workflows, secrets-and-hooks,
proxy-and-ssl, builders, troubleshooting, kamal-v1).
```

### Codex, Amp, and other AGENTS.md-compatible tools

Vendor the skill (same as above), then add a section to your project's `AGENTS.md`:

```markdown
## Kamal deployments

When helping with Kamal deployments, read `tools/kamal-skill/SKILL.md` for version detection,
setup workflow, a command cheatsheet, and critical gotchas. Load the relevant file from
`tools/kamal-skill/references/` on demand (configuration, commands, workflows, secrets-and-hooks,
proxy-and-ssl, builders, troubleshooting, kamal-v1).
```

### Any LLM — Claude.ai, API, or standalone

The content is plain markdown and works in any interface:

| Interface | How to use |
|---|---|
| **Claude.ai** | Paste `SKILL.md` + the relevant `references/*.md` into your conversation |
| **Claude API** | Include the files as system-prompt context |
| **Any LLM / standalone docs** | The `references/` files are plain Kamal documentation — use them directly |

## How it stays in sync

All tool integrations point at the same `SKILL.md` and `references/` files. Update the skill once (pull the submodule, copy the new version, etc.) and every tool integration picks up the changes automatically — there is no per-tool content to keep in sync.

The progressive-disclosure design means each pointer is small (a few lines), while the full knowledge base — eight reference files and annotated config assets — is always available for the agent to open on demand.

## Kamal version support

The skill defaults to **Kamal 2.x** (current). Full Kamal 1.9.x syntax — including the `traefik:` block, `.env`-based secrets, `kamal env push`, and the complete v1→v2 upgrade procedure — is in [`references/kamal-v1.md`](references/kamal-v1.md).

## License

MIT © [Eagerworks](https://eagerworks.com)

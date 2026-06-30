# skills

Agent skills maintained by [Eagerworks](https://eagerworks.com) — portable, progressive-disclosure knowledge bases that teach coding agents how to do a specific job well.

Each skill is plain markdown and works with Claude Code, Cursor, GitHub Copilot, Codex, Amp, and any agentic tool that reads the [Agent Skills](https://www.skills.sh) format. Install them with one command via the [skills.sh CLI](https://www.skills.sh).

## Available skills

| Skill | What it does |
|---|---|
| [**kamal**](skills/kamal/) | Zero-downtime [Kamal](https://kamal-deploy.org) deployments (v2.x + 1.9.x): first-time setup, deploys, rollbacks, rolling deploys, `kamal-proxy` + Let's Encrypt SSL, secrets & vault adapters, accessories, builders/multiarch, troubleshooting, and the v1→v2 upgrade. |

## Install

Install with the [skills.sh CLI](https://www.skills.sh) — one command, no manual file copying or pointer files to write. It works with Claude Code and 70+ other agents (Cursor, GitHub Copilot, Codex, Amp, and more).

```bash
# install a specific skill
npx skills add eagerworks/skills --skill kamal

# or pick interactively from the collection
npx skills add eagerworks/skills

# or install every skill in the collection
npx skills add eagerworks/skills --all
```

A few notes:

- **Agent detection** — the CLI auto-detects the coding agents you have installed. If none are detected, it prompts you to pick which ones to install to.
- **Target specific tools** with `-a` / `--agent` (repeatable):

  ```bash
  npx skills add eagerworks/skills --skill kamal -a claude-code -a cursor
  ```

  Supported slugs include `claude-code`, `cursor`, `github-copilot`, `codex`, and `amp` (plus dozens more).
- **Global vs project** — installs into your project by default. Add `-g` / `--global` to make it available in every project.
- **No pointer files** — agents with native Agent Skills support load each skill's `SKILL.md` automatically from its frontmatter `description`, and open `references/*.md` on demand. Nothing else to wire up.

<details>
<summary><strong>Manual install (advanced)</strong> — for tools without skills.sh support, or if you prefer to vendor the files yourself</summary>

The universal pattern is the same for every tool, and every skill lives under `skills/<name>/`:

1. **Vendor the skill** into your project (or a global location).
2. **Add a small pointer** in your tool's rules/instructions file so the agent knows to load the skill's `SKILL.md` when it's relevant.

The agent opens `references/*.md` files on demand, so the pointer stays tiny while the full knowledge base is always available. All integrations point at the same `SKILL.md` and `references/` files, so updating the vendored copy updates every integration at once.

The examples below use the **kamal** skill — swap `kamal` for any other skill in the collection.

### Claude Code

**Global install** (available in every project):

```bash
# copy:
cp -r /path/to/skills/skills/kamal ~/.claude/skills/kamal

# or symlink (changes in the repo reflect immediately):
ln -s /path/to/skills/skills/kamal ~/.claude/skills/kamal
```

**Project-level install** (committed to the repo, shared with your team):

```bash
mkdir -p .claude/skills
cp -r /path/to/skills/skills/kamal .claude/skills/kamal
# or: ln -s /path/to/skills/skills/kamal .claude/skills/kamal
```

Claude Code reads the `name` and `description` frontmatter in `SKILL.md` and loads the skill automatically — no extra pointer file needed.

### Cursor

First, vendor the collection into your project:

```bash
git submodule add https://github.com/eagerworks/skills tools/eagerworks-skills
# or: cp -r /path/to/skills tools/eagerworks-skills
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

Read @tools/eagerworks-skills/skills/kamal/SKILL.md for Kamal deployment guidance.
Load the matching file from @tools/eagerworks-skills/skills/kamal/references/ on demand as needed.
```

Cursor matches the `description` to incoming requests and loads the rule automatically — no need to mention Kamal explicitly.

### GitHub Copilot

Vendor the collection first (same as above):

```bash
git submodule add https://github.com/eagerworks/skills tools/eagerworks-skills
# or: cp -r /path/to/skills tools/eagerworks-skills
```

**Option A — always-on** (applies to every file): append to `.github/copilot-instructions.md`:

```markdown
## Kamal deployments

When helping with Kamal deployments, read `tools/eagerworks-skills/skills/kamal/SKILL.md` for version detection,
setup workflow, a command cheatsheet, and critical gotchas. Load the relevant file from
`tools/eagerworks-skills/skills/kamal/references/` on demand (configuration, commands, workflows, secrets-and-hooks,
proxy-and-ssl, builders, troubleshooting, kamal-v1).
```

**Option B — path-scoped** (triggers only for Kamal files): create `.github/instructions/kamal.instructions.md`:

```markdown
---
applyTo: "**/deploy.yml,**/.kamal/**,**/kamal/**"
---

When helping with Kamal deployments, read `tools/eagerworks-skills/skills/kamal/SKILL.md` for version detection,
setup workflow, a command cheatsheet, and critical gotchas. Load the relevant file from
`tools/eagerworks-skills/skills/kamal/references/` on demand (configuration, commands, workflows, secrets-and-hooks,
proxy-and-ssl, builders, troubleshooting, kamal-v1).
```

### Codex, Amp, and other AGENTS.md-compatible tools

Vendor the collection (same as above), then add a section to your project's `AGENTS.md`:

```markdown
## Kamal deployments

When helping with Kamal deployments, read `tools/eagerworks-skills/skills/kamal/SKILL.md` for version detection,
setup workflow, a command cheatsheet, and critical gotchas. Load the relevant file from
`tools/eagerworks-skills/skills/kamal/references/` on demand (configuration, commands, workflows, secrets-and-hooks,
proxy-and-ssl, builders, troubleshooting, kamal-v1).
```

</details>

### Any LLM — Claude.ai, API, or standalone

The content is plain markdown and works in any interface:

| Interface | How to use |
|---|---|
| **Claude.ai** | Paste a skill's `SKILL.md` + the relevant `references/*.md` into your conversation |
| **Claude API** | Include the files as system-prompt context |
| **Any LLM / standalone docs** | The `references/` files are plain documentation — use them directly |

## Repository structure

Each skill is a self-contained directory under `skills/<name>/` — this is the boundary the skills.sh CLI ships, so everything the agent needs lives inside it:

```
skills/
  kamal/                        # one skill (what skills.sh installs)
    SKILL.md                    # agent entrypoint: when to use it + a lean hub
    references/                 # in-depth docs, loaded on demand
    assets/                     # copyable starter files / templates
    README.md                   # human-facing overview of the skill
evals/
  kamal/
    evals.json                  # per-skill eval cases (repo-level harness, not shipped)
```

## Adding a new skill

1. Create `skills/<name>/SKILL.md` with `name` and `description` frontmatter (the `description` is what agents match against — make it specific about when to use the skill).
2. Add `references/` and `assets/` inside `skills/<name>/` as needed; keep `SKILL.md` lean and push depth into `references/`.
3. Add eval cases at `evals/<name>/evals.json`.
4. Add a row to the **Available skills** table above.

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for authoring conventions.

## How it stays in sync

Every tool reads the same `SKILL.md` and `references/` files. Re-run `npx skills add eagerworks/skills --skill <name>` to pull the latest version (or, on a manual install, update your vendored copy) and every agent picks up the changes — there is no per-tool content to keep in sync.

The progressive-disclosure design means the agent loads only `SKILL.md` up front, while the full knowledge base is always available to open on demand.

## License

MIT © [Eagerworks](https://eagerworks.com)

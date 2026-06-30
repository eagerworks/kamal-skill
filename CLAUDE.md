# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`eagerworks/skills` is a **collection of portable markdown agent skills** — not a runnable library. There is no build, lint, or compile step. The "product" is documentation that teaches coding agents (Claude Code, Cursor, Copilot, Codex, Amp, any [skills.sh](https://www.skills.sh)-compatible tool) how to do a specific job well. Contributions are almost always edits to markdown files.

## Architecture

Two parallel trees, intentionally kept separate (see `docs/decision-records/0001-evals-separate-from-skills.md`):

```
skills/<name>/      # SHIPPED to users — the skills.sh CLI copies this whole dir
  SKILL.md          # agent entrypoint: frontmatter `description` (match trigger) + lean hub
  references/*.md    # in-depth docs, loaded by the agent ON DEMAND
  assets/           # copyable starter files / templates (e.g. deploy.yml)
  README.md         # human-facing overview
evals/<name>/       # NOT shipped — repo-level test harness
  evals.json        # question/answer + expectations pairs that verify skill quality
```

The boundary is the rule: **anything users should receive goes inside `skills/<name>/`; anything purely for developing or validating the skill stays out of it.** `evals/<name>/` mirrors the skill name. The skills.sh CLI ships the entire directory containing a `SKILL.md`, which is why evals must live outside it.

### Progressive disclosure (the core design principle)

`SKILL.md` is loaded up front, so it must stay **lean** — it's a hub: when-to-use frontmatter, a version-detection step, a table pointing to `references/*.md`, and critical gotchas. Depth belongs in `references/*.md`, which the agent opens only when relevant. When editing, push detail down into `references/` rather than growing `SKILL.md`.

## The kamal skill (the only skill so far)

Deploys Dockerized apps with Kamal. **Version-aware**: defaults to **Kamal 2.x**; all Kamal 1.9.x content lives *exclusively* in `references/kamal-v1.md`. `SKILL.md` opens with a version-detection step (`kamal version`, or infer from `traefik:`/`.env` → v1 vs `proxy:`/`.kamal/secrets` → v2). When adding version-specific examples, mark them (`# Kamal 2.x only` / `# Kamal 1.x only`) and never mix v1 syntax into the v2 references.

## Authoring conventions (from CONTRIBUTING.md)

- **Good/bad config**: use `✅ correct` / `❌ wrong` to contrast valid and invalid config.
- **No real secrets**: placeholders only (`your-token`, `ghcr.io/your-org/your-app`, `192.168.0.1`, `your.domain.com`). The skill teaches secret hygiene — examples must model it.
- **Code blocks for everything**: fence all commands/config with the right language tag (`bash`, `yaml`, `ruby`).
- Keep `SKILL.md`'s frontmatter `description` specific about *when* to use the skill — agents match against it.

## Adding a new skill

1. `skills/<name>/SKILL.md` with `name` + `description` frontmatter.
2. `references/` and `assets/` inside `skills/<name>/`; keep `SKILL.md` lean.
3. `skills/<name>/README.md` human overview.
4. `evals/<name>/evals.json` with cases.
5. Add a row to the **Available skills** table in `README.md`.

## Commits

[Conventional Commits](https://www.conventionalcommits.org/), matching existing history:
- `docs:` — content changes to `SKILL.md`, `references/`, `assets/`
- `feat:` — new reference files, new asset templates, significant new coverage
- `fix:` — corrections to wrong/outdated information
- `chore:` — eval cases, project-level housekeeping

## skill-creator (vendored tooling)

`.agents/skills/skill-creator/` is a vendored skill (tracked in `skills-lock.json`, sourced from `anthropics/skills`) used to author/eval/optimize skills. It contains the only executable code in the repo (Python under `scripts/` and `eval-viewer/`). Don't hand-edit it as project content — it's a managed dependency.

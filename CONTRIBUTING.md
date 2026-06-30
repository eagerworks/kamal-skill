# Contributing to eagerworks/skills

`eagerworks/skills` is a collection of portable markdown agent skills — not a runnable library. Contributions improve the quality and accuracy of the guidance agents receive. Every improvement, no matter how small, benefits anyone who uses these skills.

## What you can contribute

- **Content fixes** — corrections to any skill's `SKILL.md` or `references/*.md` file (outdated commands, wrong flags, missing gotchas)
- **New reference content** — coverage gaps in an existing skill
- **Annotated assets** — additions or updates to a skill's `assets/` (config templates, executable samples)
- **Eval cases** — new test cases under `evals/<skill>/evals.json` that verify a skill answers questions correctly
- **A new skill** — see [Adding a new skill](#adding-a-new-skill) below
- **Clarity** — rewriting confusing sections, fixing typos, improving code comments

## Project structure

See [`README.md`](README.md) for the full layout. Each skill is a self-contained directory under `skills/<name>/`, and the key design principle (illustrated here with `kamal`) is the same for every skill:

- **`skills/<name>/SKILL.md`** is the hub and agent entrypoint — keep it lean. If something is detailed or niche, it belongs in `references/`.
- **`skills/<name>/references/*.md`** files are loaded on demand — each covers a single domain in depth. Add detail here rather than expanding `SKILL.md`.
- **`skills/<name>/assets/`** holds copyable starter files — templates and executable samples.
- **`skills/<name>/README.md`** is the human-facing overview of the skill.
- **`evals/<name>/evals.json`** is the repo-level test harness — question/answer pairs used to verify quality (kept outside `skills/<name>/` so it isn't shipped to users).

## Adding a new skill

1. Create `skills/<name>/SKILL.md` with `name` and `description` frontmatter. The `description` is what agents match against — be specific about *when* to use the skill.
2. Add `references/` and `assets/` inside `skills/<name>/` as needed; keep `SKILL.md` lean and push depth into `references/`.
3. Add a `skills/<name>/README.md` human overview.
4. Add eval cases at `evals/<name>/evals.json`.
5. Add a row to the **Available skills** table in [`README.md`](README.md).

## Content guidelines

**Version accuracy.** The skill defaults to Kamal 2.x. Kamal 1.9.x content belongs exclusively in `skills/kamal/references/kamal-v1.md`. When adding examples that only apply to one version, mark them clearly:

```yaml
# Kamal 2.x only
proxy:
  ssl: true

# Kamal 1.x only (traefik block)
traefik:
  args:
    entryPoints.web.address: ":80"
```

**Good/bad config convention.** Use `✅ correct` / `❌ wrong` to contrast valid and invalid config — consistent with the existing gotchas section in `SKILL.md`.

**No real secrets.** Use placeholder values in all examples — `your-token`, `ghcr.io/your-org/your-app`, `192.168.0.1`, `your.domain.com`. Never include real credentials, registry tokens, or server addresses. The skill itself teaches users to keep secrets out of version control; examples must model that.

**Code blocks for everything.** Wrap all commands, config snippets, and file content in fenced code blocks with the appropriate language tag (`bash`, `yaml`, `ruby`, etc.).

## Testing your changes

### Eval cases

Each skill's `evals/<name>/evals.json` (e.g. `evals/kamal/evals.json`) contains question/answer pairs for automated evaluation. When your change alters skill behavior or adds new coverage, add a corresponding eval case:

```json
{
  "question": "How do I run a one-off command inside a running Kamal 2 container?",
  "answer": "Use `kamal app exec -i \"bash\"` for an interactive shell, or `kamal app exec \"your-command\"` for a non-interactive run. Add `--reuse` to reuse the existing container instead of starting a new one."
}
```

### Manual sanity check

Load the updated skill in an agent (Claude Code, Cursor, etc.) and ask questions that exercise your changes. Verify the agent cites correct commands and config for the right Kamal version.

## Commit & PR process

1. **Branch from `main`:**
   ```bash
   git checkout -b docs/fix-rolling-deploy-example
   ```

2. **Commit with [Conventional Commits](https://www.conventionalcommits.org/)** — matching the project's existing history:
   - `docs:` — content changes to `SKILL.md`, `references/`, or `assets/`
   - `feat:` — new reference files, new asset templates, significant new coverage
   - `fix:` — corrections to wrong or outdated information
   - `chore:` — eval cases, project-level housekeeping

   ```bash
   git commit -m "docs: add missing --skip-push flag to cheatsheet"
   ```

3. **Open a PR against `main`** on [`eagerworks/skills`](https://github.com/eagerworks/skills). In the PR description, briefly explain what changed and why — a sentence or two is fine.

4. **Reference issues** when applicable: `Closes #12` or `See #8`.

## Reporting bugs or requesting content

Open a [GitHub issue](https://github.com/eagerworks/skills/issues) — whether it's a factual error, missing coverage, an outdated command, or a feature request.

## License

By contributing, you agree that your changes will be released under the [MIT License](LICENSE) that covers this project.

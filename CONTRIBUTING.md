# Contributing to kamal-skill

`kamal-skill` is a portable markdown knowledge base — not a runnable library. Contributions improve the quality and accuracy of the deployment guidance agents receive when working with Kamal. Every improvement, no matter how small, benefits anyone who uses the skill.

## What you can contribute

- **Content fixes** — corrections to `skills/kamal/SKILL.md` or any `skills/kamal/references/*.md` file (outdated commands, wrong flags, missing gotchas)
- **New reference content** — coverage gaps in configuration options, commands, or workflows
- **Annotated assets** — additions or updates to `skills/kamal/assets/deploy.yml`, `skills/kamal/assets/deploy.v1.yml`, `skills/kamal/assets/secrets.example`, or the hook samples in `skills/kamal/assets/hooks/`
- **Eval cases** — new test cases in `evals/evals.json` that verify the skill answers questions correctly
- **Clarity** — rewriting confusing sections, fixing typos, improving code comments

## Project structure

See [`README.md`](README.md) for the full layout. The key design principle:

- **`skills/kamal/SKILL.md`** is the hub — version detection, setup workflow, command cheatsheet, and critical gotchas. Keep it lean. If something is detailed or niche, it belongs in `references/`.
- **`skills/kamal/references/*.md`** files are loaded on demand — each covers a single domain in depth. Add detail here rather than expanding `SKILL.md`.
- **`skills/kamal/assets/`** holds copyable starter files — annotated config templates and executable hook samples.
- **`evals/evals.json`** is the repo-level test harness — a list of question/answer pairs used to verify skill quality (not shipped with the skill).

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

`evals/evals.json` contains question/answer pairs for automated evaluation. When your change alters skill behavior or adds new coverage, add a corresponding eval case:

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

3. **Open a PR against `main`** on [`eagerworks/kamal-skill`](https://github.com/eagerworks/kamal-skill). In the PR description, briefly explain what changed and why — a sentence or two is fine.

4. **Reference issues** when applicable: `Closes #12` or `See #8`.

## Reporting bugs or requesting content

Open a [GitHub issue](https://github.com/eagerworks/kamal-skill/issues) — whether it's a factual error, missing coverage, an outdated command, or a feature request.

## License

By contributing, you agree that your changes will be released under the [MIT License](LICENSE) that covers this project.

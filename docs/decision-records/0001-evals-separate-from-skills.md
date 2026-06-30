# 1. Keep `evals/` separate from `skills/`

- **Status:** Accepted
- **Date:** 2026-06-30

## Context

The skills.sh CLI (`vercel-labs/skills`) installs a skill by copying the **entire
directory that contains its `SKILL.md`**. For a skill at `skills/<name>/`, that means
every file under `skills/<name>/` — `SKILL.md`, `references/`, `assets/`, `README.md`,
and anything else — is shipped to the user's agent directory
(`.claude/skills/<name>/`, `.agents/skills/<name>/`, etc.).

We maintain eval cases (`evals.json`) to verify each skill answers questions
correctly. These are a **development/test harness**: they are useful to contributors
and CI, but they are noise — or worse, confusing — inside an end user's installed
skill. An agent scanning the skill directory should see only the knowledge base, not
our test fixtures.

If evals lived at `skills/<name>/evals/`, they would be bundled into every install.

## Decision

Keep evals **outside** the shipped skill directory, at the repository root, namespaced
per skill:

```
skills/
  kamal/            # shipped to users by skills.sh
    SKILL.md
    references/
    assets/
    README.md
evals/
  kamal/            # NOT shipped — repo-level test harness
    evals.json
```

The rule: anything that should reach users goes inside `skills/<name>/`; anything that
is purely for developing or validating the skill stays out of it. `evals/<name>/`
mirrors the skill name so the mapping stays obvious as more skills are added.

## Consequences

- **Installed skills stay clean** — users receive only the knowledge base, no test
  fixtures.
- **Per-skill evals scale** — each new skill adds `evals/<name>/`, parallel to
  `skills/<name>/`.
- Tooling that runs evals must look under `evals/<name>/`, not inside the skill
  directory.
- This generalizes: any future non-shipped, skill-specific tooling (fixtures,
  generation scripts, benchmarks) should follow the same "outside `skills/`" rule.

## Related

- The decision to nest each skill under `skills/<name>/` (rather than a root-level
  `SKILL.md`) so the CLI ships the full bundle — see the project README's repository
  structure section.

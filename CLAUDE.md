# claude-skill-commit

Claude Code skills for `/commit` and `/commit-act`. Each skill is a
self-contained SKILL.md in its own subdirectory.

## Structure

- `commit/SKILL.md` — `/commit` skill (local pre-commit detection, make fallback)
- `commit-act/SKILL.md` — `/commit-act` skill (CI-faithful checks via act)
- `Plan-multi-commit-support.md` — design doc for future `/commit-plan` skill

## No build system

No Makefile, no dependencies, no tests. Changes are tested by invoking `/commit` in a target repo with staged changes.

## Conventions

- Conventional Commits format
- GPL-3.0 license

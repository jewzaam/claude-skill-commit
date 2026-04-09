# claude-skill-commit

Claude Code skill for `/commit`. Single-file skill — all behavior is defined in `SKILL.md`.

## Structure

- `SKILL.md` — skill definition (the only file that matters)
- `Plan-multi-commit-support.md` — design doc for future `/commit-plan` skill

## No build system

No Makefile, no dependencies, no tests. Changes are tested by invoking `/commit` in a target repo with staged changes.

## Conventions

- Conventional Commits format
- GPL-3.0 license

# commit skill

Shared Claude Code and Codex skill for creating commits from staged changes. Claude Code invokes it with `/commit`; Codex invokes it explicitly with `$commit`. `SKILL.md` at repo root plus bundled helper scripts under `scripts/`.

## Structure

- `SKILL.md` — shared skill instructions. Claude Code and Codex invoke the orchestrator and report its result.
- `agents/openai.yaml` — Codex UI metadata and explicit-only invocation policy.
- `scripts/commit` — the commit orchestrator. It invokes `scripts/util/detect-checks.sh` for validation, asks a read-only harness for a message only after validation passes, and performs the final commit.
- `scripts/util/detect-checks.sh` — the original standalone validation utility, retained unchanged for workflows that need its detailed act diagnostics and machine-readable output. It is deliberately below `scripts/` so only `scripts/commit` is exposed when that directory is put on `PATH`.
- `references/commit-message-prompt.md` — the full commit-message guidance consumed by the configured harness.

## No build system

No Makefile, no dependencies, no tests. Changes are tested by invoking `scripts/commit` in a target repo with staged changes. It is the skill's only non-trivial logic, and it writes to `.tmp-commit-skill/` in whichever cwd you run it from.

## Conventions

- Conventional Commits format — rules are supplied to the harness from `references/commit-message-prompt.md`; `SKILL.md` contains only execution instructions.
- **Validation cascade.** `scripts/util/detect-checks.sh` runs act over pull-request jobs first, then a literal `make check` target, then bare mode. Its detailed act behavior and machine-readable output are intentionally retained there rather than duplicated in `scripts/commit`.
- **Makefile fallback: `check` only.** The probe looks for a literal `check:` target because the user's `~/source/standards/` convention reserves `check` for non-mutating validation. There is no fallback to `test` or the default target.
- **Log dir: `.tmp-commit-skill/`.** Repo-root-relative. The detector wipes and recreates it at the start of every invocation, then leaves it for inspection. A `.gitignore` with `*` keeps its contents out of `git status`.
- **Harness configuration.** Set `COMMIT_HARNESS=claude|codex`, `COMMIT_MODEL`, and `COMMIT_EFFORT` when invoking `scripts/commit`; the defaults are `claude`, `sonnet`, and `medium`.
- GPL-3.0 license

## Upstream specs tracked by this skill

The rules in `references/commit-message-prompt.md` are snapshots of upstream specs, not independent inventions. If upstream changes, flag it as a revision task for this repo — the skill does not auto-follow upstream.

- **Conventional Commits 1.0.0** — <https://www.conventionalcommits.org/en/v1.0.0/>
  - Source of title structure, scope syntax, `!` / `BREAKING CHANGE:` footer semantics, and footer token format encoded in `references/commit-message-prompt.md`.
  - On upstream version bump or grammar change: review the diff, update the prompt reference, and note the new version there.
- **Angular Commit Message Guidelines** — <https://github.com/angular/angular/blob/main/contributing-docs/commit-message-guidelines.md>
  - Source of the type list (8 types: feat, fix, build, ci, docs, perf, refactor, test). The Conventional Commits spec mandates only `feat` and `fix`; the full type list follows Angular's current convention, not the defunct AngularJS convention or `@commitlint/config-conventional`.
  - On upstream change to Angular's type list: review the corresponding section in `references/commit-message-prompt.md`.

# claude-skill-commit

Claude Code skill for `/commit`. `SKILL.md` at repo root plus bundled helper scripts under `scripts/`.

## Structure

- `SKILL.md` — `/commit` skill: at skill-load time it calls `scripts/detect-checks.sh` via a `!` injection, reads the machine-readable result, and then either commits the staged changes or routes user consent via `AskUserQuestion` for fail / unmatched / bare cases. Claude does not run any validation commands directly — the script is the engine.
- `scripts/detect-checks.sh` — the validation engine. Enforces repo-root invocation as a precondition, bootstraps `.tmp-commit-skill/`, then cascades through three modes:
  - **act mode** when `act` is installed and `act pull_request --list` returns ≥1 job. Executes every listed job via act with the security-flags template. No `gh` / rulesets dependency — works on private repos without a paid plan and on repos with non-GitHub remotes.
  - **make mode** when a `Makefile` declares a literal `check:` target. Executes `make check`.
  - **bare mode** when neither applies. Runs no validation; emits a REASON explaining what was tried.
  Full per-job stdout+stderr is written to `.tmp-commit-skill/` (wiped at the start of every invocation, persists afterwards so the user can inspect post-hoc). The machine-readable summary on stdout is documented in the `Validation (auto-detected)` section of `SKILL.md`. Path resolved via `${CLAUDE_PLUGIN_ROOT}`, which Claude Code sets for plugin-installed skills.

## No build system

No Makefile, no dependencies, no tests. Changes are tested by invoking `/commit` in a target repo with staged changes. When editing `scripts/detect-checks.sh`, run it standalone from a repo root (`bash scripts/detect-checks.sh` from inside a target repo) to validate output before committing — it is the skill's only non-trivial logic, and it writes to `.tmp-commit-skill/` in whichever cwd you run it from.

## Conventions

- Conventional Commits format — rules are encoded inline in `SKILL.md` so the skill is self-contained at runtime.
- **Makefile fallback: `check` only.** The script's make-mode probe looks for a literal `check:` target because the user's `~/source/standards/` convention reserves `check` for non-mutating validation. Default targets in user repos may run mutating operations like `format` (which rewrites files), so auto-running them would be unsafe. The fallback is deliberately narrow: no auto-fallback to `test`, no default-target unpacking. Repos that want something else under `/commit` should add a `check:` target.
- **Log dir: `.tmp-commit-skill/`.** Repo-root-relative. Wiped at the start of every `/commit` invocation, persists after. A `.gitignore` with `*` is dropped inside so contents stay out of `git status`. Users who want to hide the directory itself should add `.tmp-commit-skill/` to their repo root `.gitignore` — the skill does not auto-edit user gitignores.
- GPL-3.0 license

## Upstream specs tracked by this skill

The inlined rules in `SKILL.md` are snapshots of upstream specs, not independent inventions. If upstream changes, flag it as a revision task for this repo — the skill does not auto-follow upstream.

- **Conventional Commits 1.0.0** — <https://www.conventionalcommits.org/en/v1.0.0/>
  - Source of the type list, title structure, scope syntax, `!` / `BREAKING CHANGE:` footer semantics, and footer token format encoded in the `Conventional Commits rules` section of `SKILL.md`.
  - On upstream version bump or grammar change: review the diff, update the inlined rules, and note the new version in `SKILL.md`.

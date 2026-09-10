# commit skill

Shared Claude Code and Codex skill for creating commits from staged changes. Claude Code invokes it with `/commit`; Codex invokes it explicitly with `$commit`. `SKILL.md` at repo root plus bundled helper scripts under `scripts/`.

## Structure

- `SKILL.md` — shared skill instructions. Claude Code and Codex invoke the orchestrator and report its result.
- `agents/openai.yaml` — Codex UI metadata and explicit-only invocation policy.
- `scripts/commit` — the commit orchestrator. It invokes `scripts/util/detect-checks.sh` for validation, asks a read-only harness for a message only after validation passes, and performs the final commit.
- `scripts/util/detect-checks.sh` — the original standalone validation utility, retained unchanged for workflows that need its detailed act diagnostics and machine-readable output. It is deliberately below `scripts/` so only `scripts/commit` is exposed when that directory is put on `PATH`.
- `hooks/` — the attribution hook and its registration, for both harnesses. `record-attribution.py` is the hook; `register.claude.json` and `register.codex.json` are its registration as data; `test_record_attribution.py` is its self-check. Not invoked by the skill. See `attribution-plan.md` for the design and what is still open.
- `references/commit-message-prompt.md` — the full commit-message guidance consumed by the configured harness.

## No build system

No Makefile and no dependencies. `scripts/commit` is tested by invoking it in a target repo with staged changes; it writes to `.tmp-commit-skill/` in whichever cwd you run it from.

The one automated check is `python3 hooks/test_record_attribution.py`. It drives the hook the way a harness does — a JSON payload on stdin — so the Claude and Codex payload shapes are part of what is tested rather than an internal API. Run it after touching the hook. It sets `GIT_CONFIG_GLOBAL`/`GIT_CONFIG_SYSTEM` to `/dev/null`, because a host that signs commits by ssh would otherwise fail the test's own commits and a sandbox has no `ssh-keygen` at all.

## Conventions

- Conventional Commits format — rules are supplied to the harness from `references/commit-message-prompt.md`; `SKILL.md` contains only execution instructions.
- **Validation cascade.** `scripts/util/detect-checks.sh` runs act over pull-request jobs first, then a literal `make check` target, then bare mode. Its detailed act behavior and machine-readable output are intentionally retained there rather than duplicated in `scripts/commit`.
- **Makefile fallback: `check` only.** The probe looks for a literal `check:` target because the user's `~/source/standards/` convention reserves `check` for non-mutating validation. There is no fallback to `test` or the default target.
- **Log dir: `.tmp-commit-skill/`.** Repo-root-relative. The detector wipes and recreates it at the start of every invocation, then leaves it for inspection. A `.gitignore` with `*` keeps its contents out of `git status`.
- **Harness configuration.** Set `COMMIT_HARNESS=claude|codex`, `COMMIT_MODEL`, and `COMMIT_EFFORT` when invoking `scripts/commit`; the defaults are `claude`, `sonnet`, and `medium`.
- **`COMMIT_HARNESS`/`COMMIT_MODEL` are the trailer *fallback*, not its source.** They name whoever wrote the commit message, which is not whoever wrote the code. The authors come from `.commit-attribution/record`, written by `hooks/record-attribution.py`, one `<Tool>|<Model>` line per model that changed a file here; `scripts/commit` dedups them into one `Assisted-by:` line each and deletes the file only after the commit succeeds. No record means the old single-label behavior, which is the intended degradation — never worse than before.
- **Models are recorded as the harness reports them** — `Assisted-by: Claude Code (claude-opus-5)`, not `Claude Opus 5`. A friendly-name table is wrong on every model release and nothing detects when it goes stale. The user's global `CLAUDE.md` states the same rule, so the two agree.
- **`.commit-attribution/` must not move under `.tmp-commit-skill/`.** That directory is wiped at the start of every invocation, by `scripts/commit` and by `detect-checks.sh` independently, and the record has to survive both to accumulate across sessions. Its own `.gitignore` of `*` is what keeps it invisible to git — load bearing for the purge, which fires on an empty `git status --porcelain`: a visible record directory would make every repo look dirty and the purge would never run.
- **The commit is not an author.** After a successful commit `scripts/commit` leaves `.commit-attribution/skip-next`, and the hook consumes it to skip exactly one record. Without it the shell call that ran the commit is recorded against whatever is still uncommitted — `scripts/commit` commits only what is staged, so that is the common case, and it is precisely the misattribution this mechanism exists to remove when the code is written in one harness and committed in another. Written only if `.commit-attribution/` already exists: no hook installed means nothing to suppress, and creating it here would leave an untracked directory behind.
- **Record on `PostToolUse`, purge on `Stop`.** The hook appends an author line for every tool call that could change a file, then deletes the record for any repo whose tree comes out clean. Recording is credulous on purpose and the purge is what makes it honest: fingerprinting the tree around every call costs a git walk per call *and* still mis-handles a change that is made and then reverted, which moves the fingerprint twice and leaves attribution attached to whatever is committed later. The cost of this direction is that a model which only ran a shell command in an already-dirty repo is credited too — no signal available at hook time separates those, and an extra `Assisted-by` line is more visible and less wrong than a missing one.
- **The skill does not install anything.** `register.claude.json` and `register.codex.json` are the registration, as data. Each config repo's `reconcile` globs `skills/*/hooks/register.<harness>.json` and merges what it finds, so the skill stays the only place the registration is written down and neither config repo holds a copy to drift. Both fragments point at `~/.claude/skills/commit/...` — including the Codex one, because that is where the skill is installed regardless of which harness runs the hook.
- **Each registration ends with `# KEEP: <reason>`.** Config-copying tooling strips hooks it cannot vouch for; that shell comment is the hook declaring it is deliberate. `openshell-sandbox`'s `strip-settings.py` honours it, which is what gets this hook into a sandbox — the case that needs it most, since work authored there is committed on the host. It has to be a comment on the command, not a JSON key: Claude Code drops unrecognised keys nested inside hook objects when it rewrites `settings.json`, so a key silently vanishes and the hook is stripped on the next copy. Hook commands run through a shell, so the comment never reaches this script — a dummy argument would have.
- GPL-3.0 license

## Upstream specs tracked by this skill

The rules in `references/commit-message-prompt.md` are snapshots of upstream specs, not independent inventions. If upstream changes, flag it as a revision task for this repo — the skill does not auto-follow upstream.

- **Conventional Commits 1.0.0** — <https://www.conventionalcommits.org/en/v1.0.0/>
  - Source of title structure, scope syntax, `!` / `BREAKING CHANGE:` footer semantics, and footer token format encoded in `references/commit-message-prompt.md`.
  - On upstream version bump or grammar change: review the diff, update the prompt reference, and note the new version there.
- **Angular Commit Message Guidelines** — <https://github.com/angular/angular/blob/main/contributing-docs/commit-message-guidelines.md>
  - Source of the type list (8 types: feat, fix, build, ci, docs, perf, refactor, test). The Conventional Commits spec mandates only `feat` and `fix`; the full type list follows Angular's current convention, not the defunct AngularJS convention or `@commitlint/config-conventional`.
  - On upstream change to Angular's type list: review the corresponding section in `references/commit-message-prompt.md`.

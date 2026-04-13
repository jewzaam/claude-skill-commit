---
name: commit
description: Run validation locally for staged changes via act or `make check`, then write a Conventional Commits message and commit. When no validation engine applies, commit directly. Invoked explicitly by the user via /commit — do not trigger it automatically.
disable-model-invocation: true
allowed-tools: Bash(git diff --staged), Bash(git commit -m *), Bash(~/.claude/skills/commit/scripts/detect-checks.sh)
---

# Commit Skill

Run validation locally for staged changes, then commit with a Conventional Commits message. Validation is chosen and executed deterministically by `scripts/detect-checks.sh`, which cascades through three modes: **act** over every `pull_request`-triggered job in the repo's `.github/workflows/`, **`make check`** for repos that declare validation via Makefile (the user's `~/source/standards/` non-mutating convention), or **bare** for repos that declare neither (no validation, commit directly). The skill itself never decides what to run and never orchestrates validation — the script is the engine. Claude only reads the result, surfaces the log directory to the user for analysis on failure, writes the commit message, and commits.

Full per-run logs are always written to `.tmp-commit-skill/` at the repo root (the path is emitted as `LOGDIR:` in the script output). The log dir is wiped at the start of every `/commit` invocation and persists after, so the user can inspect raw act or `make check` output without re-running anything.

## Current Working Directory (auto-detected)

!`pwd`

## Staged files (auto-detected)

!`git diff --staged --name-only || true`

## Recent commits (auto-detected)

!`git log -3 --oneline || true`

## Working tree state (auto-detected)

!`git status --short || true`

## Validation (auto-detected)

The injection below calls `scripts/detect-checks.sh`, which is the canonical validation engine. The script enforces repo-root invocation, bootstraps `.tmp-commit-skill/`, cascades through act → make → bare, executes the selected validation, writes full stdout+stderr per job to the log directory, and emits a machine-readable summary on stdout.

**Output contract** — the skill parses these exact shapes. Any failing job tail is delimited by `---` and truncated to the last 30 lines. Full logs live in `$LOGDIR`.

```
MODE: act
RESULT: pass | fail
LOGDIR: .tmp-commit-skill/
DOCKER_HOST: <resolved-uri>
CHECK: <job-id> PASS
CHECK: <job-id> FAIL
---
<last 30 lines of act-<job-id>.log>
---
```

```
MODE: make
TARGET: check
RESULT: pass | fail
LOGDIR: .tmp-commit-skill/
---
<last 30 lines of make-check.log on fail>
---
```

```
MODE: bare
LOGDIR: .tmp-commit-skill/
REASON: <one-line explanation of what was tried and why nothing applied>
--- DIAGNOSTIC ---
<optional multi-line actionable context: platform, what was tried, concrete TO FIX commands>
--- END DIAGNOSTIC ---
```

**RESULT semantics (act mode):** `pass` = every job act listed ran and passed; `fail` = at least one job failed. There is no unmatched/partial case because the script enumerates jobs directly from `act pull_request --list` rather than matching against an external list of required checks — every job act knows about is a job the script runs.

**DIAGNOSTIC block:** only appears in bare mode and only when the script has actionable fix context (typically: `DOCKER_HOST` could not be resolved for act, with platform-specific commands to start the podman socket or machine). The block is opaque to the skill — surface it to the user as informational output so they have a paste-ready fix list if they want to enable act for the repo.

**`MODE: untested`:** written to `$LOGDIR/summary.txt` before the script acquires its global lock. Prevents stale data from a previous run from being misread if the script blocks waiting for another `/commit` to finish. The skill never reads `summary.txt` — it parses the script's stdout. This sentinel exists for external tooling only.

**Exit codes:** `0` when validation passed or bare mode was entered; `1` when validation failed (any act job or `make check` returned non-zero) OR when the script cannot dispatch at all (not a git repo, invoked from a subdirectory, git broken). On validation failure the summary is still written with per-job details so the skill can parse and report; only the exit code changes.

!`~/.claude/skills/commit/scripts/detect-checks.sh`

The script is the canonical source for matching rules, prerequisite checks, log-dir layout, and the output contract. Read `scripts/detect-checks.sh` if anything is unclear at runtime — do not re-derive it.

## Conventional Commits rules

Commit messages follow the rules below. These rules are the canonical source at execution time — do not fetch any external spec.

### Structure

```
<type>[optional scope][!]: <description>

[optional body]

[optional footer(s)]
```

- **Title line:** `<type>[optional scope][!]: <description>`, total length ≤72 characters.
- **Blank line** separates title from body. Another blank line separates body from footers.

### Types

Only these types are valid. Pick the most specific one that fits.

- **feat** — a new feature (MINOR in SemVer)
- **fix** — a bug fix (PATCH in SemVer)
- **docs** — documentation-only changes
- **style** — whitespace, formatting, semicolons, no code-behavior change
- **refactor** — code change that neither fixes a bug nor adds a feature
- **perf** — performance improvement
- **test** — adding or correcting tests
- **build** — build system, dependency, packaging changes
- **ci** — CI configuration and scripts
- **chore** — maintenance that doesn't fit another type; do not use as a catch-all when a more specific type applies

### Scope (optional)

A noun in parentheses immediately after the type, describing the section of the codebase touched:

- `feat(parser): add regex fallback`
- `fix(auth): handle expired refresh tokens`

Use a scope when one clearly applies. Omit it for cross-cutting changes.

### Description

- Imperative mood: "add", "fix", "remove" — not "added", "adds", "adding".
- Lowercase first letter unless it's a proper noun.
- No trailing period.
- Concrete and specific. "fix: correctness bug" is not acceptable; "fix: drop stale cache entries on TTL expiry" is.

### Breaking changes

A breaking change is signaled two ways, and you may use either or both:

1. **`!` after type/scope**, before the colon: `feat!: drop Python 3.9 support` or `refactor(api)!: rename /users to /accounts`.
2. **`BREAKING CHANGE:` footer** — literal uppercase, colon, space, description. Must appear as a footer, not in the body.

Either form bumps MAJOR in SemVer. Using both is allowed: `!` signals at a glance, the footer documents the impact.

### Body (optional)

- Separated from title by one blank line.
- Free-form prose or bullets describing what changed and why.
- Focus on logical changes, not line-by-line diffs. Do not mention tests (assumed).

### Footers (optional)

- Separated from body (or title, if no body) by one blank line.
- Format: `Token: value` or `Token #value`.
- Token is kebab-case (`Signed-off-by`, `Reviewed-by`, `Refs`, `Closes`). The only space-containing token is `BREAKING CHANGE`.
- Multiple footers allowed, one per line.
- `Assisted-by: Claude Code (<model>)` is a footer on every commit this skill creates.

### Example

```
feat(api)!: replace session cookies with bearer tokens

- Issue bearer token on login response
- Validate Authorization header in middleware
- Remove cookie-parser dependency

BREAKING CHANGE: clients must now send Authorization: Bearer <token>
Refs: #482
Assisted-by: Claude Code (Claude Opus 4.6)
```

## Process

1. **Step 1 — Check for staged changes:**
   Look at the **Staged files (auto-detected)** section above.
   - If the section is empty (no filenames listed), tell the user "Nothing is staged" and STOP.
   - If filenames are listed, → proceed to Step 2.

2. **Step 2 — Gather full diff:**
   Run a single Bash call:
   - `git diff --staged` — see actual changes
   - → Proceed to Step 3.

3. **Step 3 — Read validation result:**
   Look at the **Validation (auto-detected)** section above.
   - If the section shows a single diagnostic line (not a git repo, invoked from a subdirectory, git broken), surface that message to the user and STOP. Do not commit.
   - If the script blocks on the global lock, the bash call goes to background. Wait for the completion event, then parse the stdout output normally.
   - Otherwise, parse the `MODE:`, `RESULT:`, and `LOGDIR:` lines, plus any `CHECK:` / `TARGET:` / `REASON:` lines for the relevant mode. Proceed to Step 4.

4. **Step 4 — Resolve consent:**
   Route on `MODE` and `RESULT`. User-facing questions route through `AskUserQuestion` — never use plain-text prompts for decisions. Every question body includes the `LOGDIR` path so the user knows where the full raw output lives.

   - **`MODE: act, RESULT: pass`** → proceed to Step 5. No question.
   - **`MODE: make, RESULT: pass`** → proceed to Step 5. No question.
   - **`RESULT: fail`** (any mode) → report the failure to the user and **STOP**. Do not commit. Do not ask whether to continue — there is no "commit anyway" path. The report must include: the mode, each failing `CHECK:` job id, a ≤5-line synthesis of each failure tail, the `LOGDIR` path for full logs, and the `DOCKER_HOST` value if present. The user must either fix the failing check or remove it from the workflow before /commit will commit anything.
   - **`MODE: bare`** → proceed to Step 5. No validation was available; commit directly. No user consent needed.

   When reporting failures, do not paste raw act or `make check` logs — the tail synthesis in the injection is already trimmed for context, and the user can open `$LOGDIR` themselves for the full output.

5. **Step 5 — Write commit message and commit:**
   Follow the **Conventional Commits rules** section above exactly. Use your actual model name from system context in the `Assisted-by` footer.

   ```bash
   git commit -m "$(cat <<'EOF'
   feat: add retry logic for API timeouts

   - Add exponential backoff to HTTP client
   - Set max retries to 3

   Assisted-by: Claude Code (<your-model-name>)
   EOF
   )"
   ```

   After the commit succeeds, tell the user the commit was created and mention the `LOGDIR` path once (so they can audit what ran before the next `/commit` invocation wipes it). Then STOP. Do not suggest additional commits, do not show post-commit output.

## Command hygiene

These constraints exist because the user's environment uses hooks to inspect Bash commands before execution.

- **No `;` chaining** — Semicolons defeat hook-based command inspection because the shell expands the full pipeline before hooks can evaluate individual commands. Use `&&` for dependent sequencing or separate Bash tool calls.
- **No `echo $?`** — The Bash tool already reports exit codes. Redundant and adds noise.

## Scope constraints

- **No `git add`** — This skill only commits what is already staged. The user controls staging.
- **No `git push`** — Pushing is a separate, explicit decision the user makes.
- **No `--amend`** — Amending rewrites history and can destroy prior commits, especially after a failed hook where the commit didn't actually happen.

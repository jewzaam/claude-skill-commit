---
name: commit
description: Commit staged changes with concise messages and proper attribution. This skill is invoked explicitly by the user via /commit — do not trigger it automatically.
disable-model-invocation: true
allowed-tools: Bash(git commit -m *)
---

# Commit Skill

## Current Working Directory (auto-detected)

!`pwd`

## Staged files (auto-detected)

!`git diff --staged --name-only || true`

## Recent commits (auto-detected)

!`git log -3 --oneline || true`

## Working tree state (auto-detected)

!`git status --short || true`

## Pre-commit check: format-check (auto-detected)

!`make format-check 2>/dev/null || true`

## Pre-commit check: lint (auto-detected)

!`make lint 2>/dev/null || true`

## Pre-commit check: typecheck (auto-detected)

!`make typecheck 2>/dev/null || true`

## Pre-commit check: markdown-lint (auto-detected)

!`make markdown-lint 2>/dev/null || true`

## Pre-commit check: links (auto-detected)

!`make links 2>/dev/null || true`

## Permitted commands

Most context is gathered by auto-detected sections above. The only Bash tool calls this skill needs are:

- `git diff --staged` — see actual changes (too large to auto-detect)
- `git commit` — create the commit

Nothing else. No `git add`, `git push`, `git commit --amend`, `make`, or any other commands.

## Why pre-commit checks exist

The goal is to minimize failures caught after pushing, without introducing significant delay at commit time. CI pipelines catch everything — lint, typecheck, tests, coverage — but discovering a failure after push slows down the review cycle. Fast, local checks (lint, typecheck, markdown-lint, link validation) catch the most common issues cheaply. Unit tests and coverage are assumed to be expensive operations that CI handles; running them here would add unacceptable delay for marginal benefit.

## Process

1. **Step 1 — Determine working directory and scope:**
   - Use the **Current Working Directory (auto-detected)** section above. No arguments accepted.
   - → Proceed to Step 2.

2. **Step 2 — Check for staged changes:**
   Look at the **Staged files (auto-detected)** section above.
   - If the section is empty (no filenames listed), tell the user "Nothing is staged" and STOP. Do not proceed.
   - If filenames are listed, → proceed to Step 2b.

3. **Step 2b — Gather full diff:**
   Run a single Bash call:
   - `git diff --staged` — see actual changes (commit style and working tree state are already in auto-detected sections above)
   - → Proceed to Step 3.

4. **Step 3 — Check pre-commit results:**
   Look at the **Pre-commit check** auto-detected sections above. For each one:
   - Empty output → target doesn't exist or passed cleanly. Proceed.
   - Non-empty output with violation details → tell the user what failed and STOP. Do not commit.
   - → If all checks are clean, proceed to Step 4.

5. **Step 4 — Write commit message and commit:**
   - **Title:** [Conventional Commits](https://www.conventionalcommits.org/) format, 72 characters or less, imperative mood, no period
     - Prefix with the appropriate type: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`
   - **Body:** Bulleted list of changes (one bullet per logical change)
   - Focus on what changed, not implementation details. Don't mention tests (assumed).
   - Reference issues or PRs when the branch name contains an obvious reference (e.g., `fix/123`, `PROJ-456`).
   - Use your actual model name from system context.
   ```bash
   git commit -m "$(cat <<'EOF'
   feat: add retry logic for API timeouts

   - Add exponential backoff to HTTP client
   - Set max retries to 3

   Assisted-by: Claude Code (<your-model-name>)
   EOF
   )"
   ```
   - → Proceed to Step 5.

6. **Step 5 — STOP.**
   Do not suggest additional commits or show post-commit output. The user will inspect the result themselves if needed.

## Command hygiene

These constraints exist because the user's environment uses hooks to inspect Bash commands before execution.

- **No `;` chaining** — Semicolons defeat hook-based command inspection because the shell expands the full pipeline before hooks can evaluate individual commands. Use `&&` for dependent sequencing or separate Bash tool calls.
- **No `echo $?`** — The Bash tool already reports exit codes. Redundant and adds noise.

## Scope constraints

- **No `git add`** — This skill only commits what is already staged. The user controls staging.
- **No `git push`** — Pushing is a separate, explicit decision the user makes.
- **No `--amend`** — Amending rewrites history and can destroy prior commits, especially after a failed hook where the commit didn't actually happen.

## Examples

### Nothing staged
```
User: /commit
Step 1: auto-detected directory
Step 2: Staged files section is empty
Tell user: "Nothing is staged." → STOP
```

### Lint failure (blocking)
```
User: /commit
Step 1: auto-detected directory
Step 2: Staged files section shows filenames → Step 2b
Step 2b: git diff --staged → Step 3
Step 3: Pre-commit check: lint section shows violations → STOP
```

### Clean commit
```
User: /commit
Step 1: auto-detected directory
Step 2: Staged files section shows filenames → Step 2b
Step 2b: git diff --staged → Step 3
Step 3: All pre-commit check sections are empty → Step 4
Step 4: Staged changes are docs → type is "docs"
        Write: "docs: clarify retry configuration" with body, git commit → Step 5
Step 5: STOP
```

---
name: commit
description: Commit staged changes with concise messages and proper attribution. This skill is invoked explicitly by the user via /commit — do not trigger it automatically.
allowed-tools: Bash
---

# Commit Skill

## Permitted commands

This skill only needs `git` and `make`. Specifically:

- `git status`, `git diff --staged`, `git log` — understand what's staged and the commit style
- `git commit` — create the commit
- `make -p -n` piped through `grep` — discover available targets
- `make <targets>` — run pre-commit checks

Nothing else. No `git add`, `git push`, `git commit --amend`, `make format`, or any other commands.

## Why pre-commit checks exist

The goal is to minimize failures caught after pushing, without introducing significant delay at commit time. CI pipelines catch everything — lint, typecheck, tests, coverage — but discovering a failure after push slows down the review cycle. Fast, local checks (lint, typecheck, markdown-lint, link validation) catch the most common issues cheaply. Unit tests and coverage are assumed to be expensive operations that CI handles; running them here would add unacceptable delay for marginal benefit.

## Process

1. **Step 1 — Determine working directory and scope:**
   - If no arguments: work in current directory, commit all staged changes
   - If argument provided (submodule name): check if it's a submodule, cd into it, and scope all work to that submodule only — do not touch the parent repository or other submodules
   - → Proceed to Step 2.

2. **Step 2 — Gather context (parallel):**
   Run all three commands as parallel Bash tool calls in a single message:
   - `git status` — see staged files
   - `git diff --staged` — see actual changes
   - `git log -3 --oneline` — understand commit style
   - If nothing is staged (`git status` output has no "Changes to be committed"), tell the user and STOP.
   - → Proceed to Step 3.

3. **Step 3 — Discover available pre-commit targets:**
   Run a single Bash call:
   ```bash
   make -p -n | grep -E "^(format-check|lint|typecheck|markdown-lint|links):"
   ```
   - If the exit code is non-zero (`make` not installed or no Makefile), → skip to Step 5.
   - The output lines are the targets that exist. Collect them.
   - If none exist, → skip to Step 5.
   - → Proceed to Step 4.

4. **Step 4 — Run pre-commit checks (blocking):**
   Run all targets that had successful probes in Step 3 in a single `make` call (e.g., `make format-check lint typecheck`). Don't suppress stderr; the full output (stdout + stderr) provides useful context for diagnosing failures.
   - Only violation-based checks belong here — never run targets that modify files (like `format` or `fix`), because they change staged content and create a confusing mismatch between what was staged and what's on disk.
   - If the command exits non-zero: tell the user what failed and STOP — do not commit.
   - The working tree state after checks is irrelevant — only exit codes matter. Don't run `git diff`, don't check for unstaged changes, don't suggest `git add`, don't comment on a dirty working tree.
   - → Proceed to Step 5.

5. **Step 5 — Write commit message and commit:**
   - **Title:** Less than 80 characters, imperative mood, no period
   - **Body:** Bulleted list of changes (one bullet per logical change)
   - Focus on what changed, not implementation details. Don't mention tests (assumed).
   - Use your actual model name from system context.
   ```bash
   git commit -m "$(cat <<'EOF'
   Short title (< 80 chars)

   - First change
   - Second change
   - Third change

   Assisted-by: Claude Code (<your-model-name>)
   EOF
   )"
   ```
   - → Proceed to Step 6.

6. **Step 6 — STOP.**
   Do not check the parent repository, suggest additional commits, or show post-commit output. The user will inspect the result themselves if needed.

## Command hygiene

These constraints exist because the user's environment uses hooks to inspect Bash commands before execution.

- **No `;` chaining** — Semicolons defeat hook-based command inspection because the shell expands the full pipeline before hooks can evaluate individual commands. Use `&&` for dependent sequencing or separate Bash tool calls.
- **No `echo $?`** — The Bash tool already reports exit codes. Redundant and adds noise.

## Scope constraints

- **No `git add`** — This skill only commits what is already staged. The user controls staging.
- **No `git push`** — Pushing is a separate, explicit decision the user makes.
- **No `--amend`** — Amending rewrites history and can destroy prior commits, especially after a failed hook where the commit didn't actually happen.

## Examples

### Scoped commit to submodule
```
User: /commit standards
Step 1: cd standards
Step 2 (parallel): git status, git diff --staged, git log -3 --oneline → staged changes exist → Step 3
Step 3: make -p -n | grep ... → lint exists → Step 4
Step 4: make lint → exit 0 → Step 5
Step 5: write message, git commit (with attribution) → Step 6
Step 6: STOP
```

### Nothing staged
```
User: /commit
Step 1: current directory
Step 2 (parallel): git status, git diff --staged, git log -3 --oneline → no staged changes
Tell user: "Nothing is staged." → STOP
```

### Lint failure (blocking)
```
User: /commit
Step 1: current directory
Step 2 (parallel): git status, git diff --staged, git log -3 --oneline → staged changes exist → Step 3
Step 3: make -p -n | grep ... → format-check, lint, typecheck all exist → Step 4
Step 4: make format-check lint typecheck → exit 1 (lint violations) → STOP
```

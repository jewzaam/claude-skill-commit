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
- `make -n <target>` — probe whether a Makefile target exists
- `make <target>` — run a pre-commit check

Nothing else. No `git add`, `git push`, `git commit --amend`, `make format`, or any other commands.

## Why pre-commit checks exist

The goal is to minimize failures caught after pushing, without introducing significant delay at commit time. CI pipelines catch everything — lint, typecheck, tests, coverage — but discovering a failure after push slows down the review cycle. Fast, local checks (lint, typecheck) catch the most common issues cheaply. Unit tests and coverage are assumed to be expensive operations that CI handles; running them here would add unacceptable delay for marginal benefit.

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

3. **Step 3 — Probe pre-commit targets (parallel):**
   Run all three probes as parallel Bash tool calls in a single message:
   - `make -n format-check`
   - `make -n lint`
   - `make -n typecheck`
   - If any probe's output contains "command not found" (meaning `make` isn't installed), → skip to Step 6.
   - A probe that fails for other reasons (e.g., "No rule to make target") means that specific target doesn't exist but `make` is available — only run targets whose probes succeeded.
   - If no probes succeeded, → skip to Step 6.
   - → Proceed to Step 4.

4. **Step 4 — Format check (blocking):**
   Only if the `format-check` probe succeeded in Step 3. Otherwise → skip to Step 5.
   - Run `make format-check`
   - If it exits non-zero: tell the user which files need formatting, suggest running `make format` to fix them, and STOP — do not commit.
   - If it passes, → proceed to Step 5.

5. **Step 5 — Lint and typecheck (parallel, advisory):**
   Run all existing targets (from successful probes in Step 3) as parallel Bash tool calls in a single message. The Bash tool reports exit codes directly in its response — check success/failure from that. Don't suppress stderr; the full output (stdout + stderr) provides useful context for diagnosing failures.
   - Only violation-based checks belong here (lint errors, type errors) — never run targets that modify files (like `format` or `fix`), because they change staged content and create a confusing mismatch between what was staged and what's on disk.
   - If any target exits non-zero:
     - Tell the user which target(s) failed
     - Explain that committing without addressing the failures will likely cause upstream PR checks to fail, slowing down the review cycle
     - Ask: "Do you want to continue with the commit anyway, or stop to address these first?"
     - If user says stop, STOP — do not commit
     - If user says continue, → proceed to Step 6
   - If all targets pass (or none had successful probes), → proceed to Step 6.
   - The working tree state after checks is irrelevant — only exit codes matter. Don't run `git diff`, don't check for unstaged changes, don't suggest `git add`, don't comment on a dirty working tree.

6. **Step 6 — Write commit message:**
   - **Title:** Less than 80 characters, imperative mood, no period
   - **Body:** Bulleted list of changes (one bullet per logical change)
   - Focus on what changed, not implementation details. Don't mention tests (assumed).
   - → Proceed to Step 7.

7. **Step 7 — Commit with attribution:**
   Use your actual model name from system context.
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
   - → Proceed to Step 8.

8. **Step 8 — STOP.**
   Do not check the parent repository, suggest additional commits, or show post-commit output. The user will inspect the result themselves if needed.

## Command hygiene

These constraints exist because the user's environment uses hooks to inspect Bash commands before execution.

- **No `;` chaining** — Semicolons defeat hook-based command inspection because the shell expands the full pipeline before hooks can evaluate individual commands. Use `&&` for dependent sequencing or separate Bash tool calls.
- **No `echo $?`** — The Bash tool already reports exit codes. Redundant and adds noise.
- **One command per Bash call when possible** — Keeps each invocation inspectable and its result unambiguous.

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
Step 3 (parallel): make -n format-check → no such target, make -n lint → exists, make -n typecheck → no such target → Step 5
Step 4: skipped (no format-check) → Step 5
Step 5: make lint → exit 0 → Step 6
Step 6–7: write message, git commit (with attribution) → Step 8
Step 8: STOP
```

### Nothing staged
```
User: /commit
Step 1: current directory
Step 2 (parallel): git status, git diff --staged, git log -3 --oneline → no staged changes
Tell user: "Nothing is staged." → STOP
```

### Lint failure, user continues
```
User: /commit
Step 1: current directory
Step 2 (parallel): git status, git diff --staged, git log -3 --oneline → staged changes exist → Step 3
Step 3 (parallel): make -n format-check → exists, make -n lint → exists, make -n typecheck → exists → Step 4
Step 4: make format-check → exit 0 → Step 5
Step 5 (parallel): make lint → exit 1, make typecheck → exit 0
Tell user: "lint failed — committing without fixing will likely fail CI. Continue or stop?"
User: "continue" → Step 6
Step 6–7: write message, git commit (with attribution) → Step 8
Step 8: STOP
```

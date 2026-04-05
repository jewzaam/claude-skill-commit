---
name: commit
description: Commit staged changes with concise messages and proper attribution. This skill is invoked explicitly by the user via /commit — do not trigger it automatically.
disable-model-invocation: true
allowed-tools: Bash(git commit -m *)
---

# Commit Skill

## Current Working Directory (auto-detected)

!`pwd`

## Pre-Commit Targets (auto-detected)

!`make -p -n | grep -E "^(format-check|lint|typecheck|markdown-lint|links):" || true`

## Staged files (auto-detected)

!`git diff --staged --name-only || true`

## Recent commits (auto-detected)

!`git log -3 --oneline || true`

## Working tree state (auto-detected)

!`git status --short || true`

## Submodule context (auto-detected)

!`git submodule foreach 'echo "=== staged files ===" && git diff --staged --name-only && echo "=== recent commits ===" && git log -3 --oneline && echo "=== working tree ===" && git status --short && echo "=== pre-commit targets ===" && make -p -n 2>/dev/null | grep -E "^(format-check|lint|typecheck|markdown-lint|links):" || true' 2>/dev/null || true`

## Permitted commands

Most context is gathered by auto-detected sections above. The only Bash tool calls this skill needs are:

- `git diff --staged` — see actual changes (too large to auto-detect)
- `make <targets>` — run pre-commit checks
- `git commit` — create the commit

Nothing else. No `git add`, `git push`, `git commit --amend`, `make format`, `make -p -n`, or any other commands.

## Why pre-commit checks exist

The goal is to minimize failures caught after pushing, without introducing significant delay at commit time. CI pipelines catch everything — lint, typecheck, tests, coverage — but discovering a failure after push slows down the review cycle. Fast, local checks (lint, typecheck, markdown-lint, link validation) catch the most common issues cheaply. Unit tests and coverage are assumed to be expensive operations that CI handles; running them here would add unacceptable delay for marginal benefit.

## Process

1. **Step 1 — Determine working directory and scope:**
   - If no arguments: use the **Current Working Directory (auto-detected)** section above
   - If argument provided (submodule name): check if it's a submodule, cd into it, and scope all work to that submodule only — do not touch the parent repository or other submodules. Use the **Submodule context (auto-detected)** section for that submodule's staged files, targets, commit style, and working tree state.
   - → Proceed to Step 2.

2. **Step 2 — Check for staged changes:**
   Look at the **Staged files (auto-detected)** section above.
   - If the section is empty (no filenames listed), tell the user "Nothing is staged" and STOP. Do not proceed.
   - If filenames are listed, → proceed to Step 2b.

3. **Step 2b — Gather full diff:**
   Run a single Bash call:
   - `git diff --staged` — see actual changes (commit style and working tree state are already in auto-detected sections above)
   - → Proceed to Step 3.

4. **Step 3 — Check available pre-commit targets:**
   Look at the **Pre-Commit Targets (auto-detected)** section above.
   - If the section is empty (no targets listed), → skip to Step 5.
   - The listed targets are what exist. Collect them.
   - → Proceed to Step 4.

5. **Step 4 — Run pre-commit checks (blocking):**
   Run all targets from Step 3 in a single `make` call (e.g., `make format-check lint typecheck`). Don't suppress stderr; the full output (stdout + stderr) provides useful context for diagnosing failures.
   - Only violation-based checks belong here — never run targets that modify files (like `format` or `fix`), because they change staged content and create a confusing mismatch between what was staged and what's on disk.
   - If the command exits non-zero: tell the user what failed and STOP — do not commit.
   - The working tree state after checks is irrelevant — only exit codes matter. Don't run `git diff`, don't check for unstaged changes, don't suggest `git add`, don't comment on a dirty working tree.
   - → Proceed to Step 5.

6. **Step 5 — Write commit message and commit:**
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

7. **Step 6 — STOP.**
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
Step 1: cd standards — use Submodule context section for "standards"
Step 2: Submodule staged files show filenames → Step 2b
Step 2b: git diff --staged → Step 3
Step 3: Submodule pre-commit targets show lint → Step 4
Step 4: make lint → exit 0 → Step 5
Step 5: write message, git commit (with attribution) → Step 6
Step 6: STOP
```

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
Step 3: Pre-Commit Targets section shows format-check, lint, typecheck → Step 4
Step 4: make format-check lint typecheck → exit 1 (lint violations) → STOP
```

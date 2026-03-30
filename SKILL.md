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

The goal is to minimize failures caught after pushing, without introducing significant delay at commit time. CI pipelines catch everything — lint, typecheck, tests, coverage — but discovering a failure after push slows down the review cycle. Fast, local checks (lint, typecheck, markdown-lint, link validation) catch the most common issues cheaply. Unit tests and coverage are assumed to be expensive operations that CI handles; running them here would add unacceptable delay for marginal benefit.

## Process

1. **Determine working directory and scope:**
   - If no arguments: work in current directory, commit all staged changes
   - If argument provided (submodule name): check if it's a submodule, cd into it, and scope all work to that submodule only — do not touch the parent repository or other submodules
   - If nothing is staged (`git status` shows no "Changes to be committed"), tell the user and stop. Do not proceed.

2. **Review staged changes:**
   - `git status` — see staged files
   - `git diff --staged` — see actual changes
   - `git log -3 --oneline` — understand commit style

3. **Run pre-commit checks:**

   **a. Format check (blocking):**
   - Check if `format-check` target exists by running `make -n format-check` as its own Bash call. If `make` itself isn't available (command not found), skip pre-commit checks entirely.
   - If it exists, run `make format-check`
   - If it exits non-zero: tell the user which files need formatting, suggest running `make format` to fix them, and **stop — do not commit**
   - If it passes (or the target does not exist), proceed

   **b. Lint, typecheck, and documentation checks (advisory):**
   - For each target in `[lint, typecheck, markdown-lint, links]`, probe whether it exists by running `make -n <target>` as its own Bash call.
   - Run each existing target as its own Bash call. The Bash tool reports exit codes directly in its response — check success/failure from that. Don't suppress stderr; the full output (stdout + stderr) provides useful context for diagnosing failures.
   - Only violation-based checks belong here (lint errors, type errors) — never run targets that modify files (like `format` or `fix`), because they change staged content and create a confusing mismatch between what was staged and what's on disk.
   - If any target exits non-zero:
     - Tell the user which target(s) failed
     - Explain that committing without addressing the failures will likely cause upstream PR checks to fail, slowing down the review cycle
     - Ask: "Do you want to continue with the commit anyway, or stop to address these first?"
     - If user says stop, halt — do not commit
     - If user says continue, proceed
   - If all targets pass (or none exist), proceed silently

   **General rules for pre-commit checks:**
   - The working tree state after checks is irrelevant — only exit codes matter. Don't run `git diff`, don't check for unstaged changes, don't suggest `git add`, don't comment on a dirty working tree.

4. **Write commit message:**
   - **Title:** Less than 80 characters, imperative mood, no period
   - **Body:** Bulleted list of changes (one bullet per logical change)
   - Focus on what changed, not implementation details. Don't mention tests (assumed).

5. **Commit with attribution:**
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

6. **After commit:**
   - STOP. Do not check the parent repository, suggest additional commits, or show post-commit output. The user will inspect the result themselves if needed.

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
Skill:
  - cd standards
  - git status → staged changes exist
  - git diff --staged, git log -3 --oneline
  - make -n lint → exists, make lint → exit 0
  - make -n typecheck → no such target, skip
  - git commit (with message + attribution)
  - STOP
```

### Nothing staged
```
User: /commit
Skill:
  - git status → no staged changes
  - Tell user: "Nothing is staged."
  - STOP
```

### Lint failure, user continues
```
User: /commit
Skill:
  - git status → staged changes exist
  - git diff --staged, git log -3 --oneline
  - make lint → exit 1 (lint violations reported in output)
  - Tell user: "lint failed — committing without fixing will likely fail CI. Continue or stop?"
  - User: "continue"
  - git commit (with message + attribution)
  - STOP
```

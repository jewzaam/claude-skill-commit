---
name: commit
description: Commit staged changes with concise messages and proper attribution
allowed-tools: Bash
---

# Commit Skill

## Process

1. **Determine working directory and scope:**
   - If no arguments: work in current directory, commit all staged changes
   - If argument provided (submodule name):
     - Check if it's a submodule and cd into it
     - **SCOPE IS LIMITED TO THIS SUBMODULE ONLY**
     - Do NOT commit changes in parent repository or other submodules

2. **Review staged changes:**
   - `git status` - see staged files
   - `git diff --staged` - see actual changes
   - `git log -3 --oneline` - understand commit style

3. **Run pre-commit checks:**
   - For each target in `[lint, typecheck]`, check if it exists: `make -n <target> 2>/dev/null`
   - Run each existing target and note the exit code
   - If any target exited non-zero:
     - Tell the user which target(s) failed
     - Explain that committing without addressing the failures will likely cause upstream PR checks to fail, slowing down the review cycle
     - Ask: "Do you want to continue with the commit anyway, or stop to address these first?"
     - If user says stop, halt — do not commit
     - If user says continue, proceed with the commit
   - If all targets pass (or no Makefile / no matching targets), proceed silently
   - Use exit codes only to determine success/failure — do NOT use `git diff` or check for unstaged changes
   - Do NOT suggest `git add` or imply unstaged changes need to be staged
   - Never frame a dirty working tree as problematic

4. **Write commit message:**
   - **Title:** Less than 80 characters, imperative mood, no period
   - **Body:** Bulleted list of changes (one bullet per logical change)
   - When mentioning functionality changes, don't mention tests (assumed)
   - Focus on what changed, not implementation details

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
   - If a specific submodule was requested, STOP HERE
   - Do NOT check for or report staged changes in parent repository
   - Do NOT suggest or perform additional commits

## Critical Rules

- **NEVER run `git add`** - only commit what's already staged
- **NEVER push** - user does this explicitly
- **NEVER use `--amend`**
- **STRICT SCOPE:** When a submodule is specified, commit ONLY in that submodule and STOP. Never commit other changes, even if staged in parent repository.

## Examples

### Correct: Scoped commit to submodule
```bash
User: /commit standards
Skill:
  - cd standards
  - git commit <changes in standards submodule>
  - git log -n1
  - git status (in standards submodule)
  - STOP (do not look at parent repository)
```

### Incorrect: Scope creep (NEVER DO THIS)
```bash
User: /commit standards
Skill:
  - cd standards
  - git commit <changes in standards>
  - cd .. (to parent)
  - git commit <changes in parent>  ❌ WRONG!
```

### Correct: No arguments (commit all in current location)
```bash
User: /commit
Skill:
  - Stay in current location
  - git commit <all staged changes>
  - git log -n1
  - git status
```

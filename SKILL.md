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

3. **Write commit message:**
   - **Title:** Less than 80 characters, imperative mood, no period
   - **Body:** Bulleted list of changes (one bullet per logical change)
   - When mentioning functionality changes, don't mention tests (assumed)
   - Focus on what changed, not implementation details

4. **Commit with attribution:**
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

5. **After commit:**
   - Run `git log -n1` to show the actual commit message
   - Run `git status` in the CURRENT LOCATION ONLY
   - If a specific submodule was requested, STOP HERE
   - Do NOT check for or report staged changes in parent repository
   - Do NOT suggest or perform additional commits

## Critical Rules

- **NEVER run `git add`** - only commit what's already staged
- **NEVER push** - user does this explicitly
- **NEVER use `--amend`** unless explicitly requested
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

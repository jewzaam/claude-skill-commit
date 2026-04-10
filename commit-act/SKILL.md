---
name: commit-act
description: Run required PR checks via act in a container, then commit staged changes. Invoked explicitly by the user via /commit-act — do not trigger it automatically.
disable-model-invocation: true
allowed-tools: Bash(DOCKER_HOST=unix:///run/user/*/podman/podman.sock act pull_request -P ubuntu-latest=docker.io/catthehacker/ubuntu:act-22.04@sha256:d83455c10c9a31c9c944a4c5628360c6c374983fa6616bd2439ab88b05ae2046 --container-daemon-socket=- --privileged=false --insecure-secrets=false *), Bash(git diff --staged), Bash(git commit -m *)
---

# Commit-Act Skill

Run the actual CI workflow locally in a container using act. Fully deterministic — the GitHub API tells us what checks are required, act runs those jobs. No model judgment about what to run.

## Current Working Directory (auto-detected)

!`pwd`

## Staged files (auto-detected)

!`git diff --staged --name-only || true`

## Recent commits (auto-detected)

!`git log -3 --oneline || true`

## Working tree state (auto-detected)

!`git status --short || true`

## Required PR checks (auto-detected)

!`gh api repos/$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)/rules/branches/$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo main) --jq '.[] | select(.type == "required_status_checks") | .parameters.required_status_checks[].context' 2>/dev/null || echo "UNAVAILABLE"`

## Available act jobs (auto-detected)

!`DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock act --list -P ubuntu-latest=docker.io/catthehacker/ubuntu:act-22.04@sha256:d83455c10c9a31c9c944a4c5628360c6c374983fa6616bd2439ab88b05ae2046 --container-daemon-socket=- --privileged=false --insecure-secrets=false 2>&1 || echo "act not available"`

## Permitted commands

Context is gathered by auto-detected sections above. The Bash tool calls this skill needs are:

- `act pull_request ...` — run CI jobs locally (must include all security flags, see Command template below)
- `git diff --staged` — see actual changes (too large to auto-detect)
- `git commit` — create the commit

Nothing else. No `git add`, `git push`, `git commit --amend`, or any other commands.

## Process

1. **Step 1 — Check for staged changes:**
   Look at the **Staged files (auto-detected)** section above.
   - If the section is empty (no filenames listed), tell the user "Nothing is staged" and STOP.
   - If filenames are listed, → proceed to Step 2.

2. **Step 2 — Gather full diff:**
   Run a single Bash call:
   - `git diff --staged` — see actual changes
   - → Proceed to Step 3.

3. **Step 3 — Validate prerequisites:**
   Check the **Required PR checks** and **Available act jobs** auto-detected sections above.
   - If required checks show "UNAVAILABLE": tell the user `gh` could not query required checks (not authenticated, repo is a fork without rulesets, or no rulesets configured) and STOP.
   - If required checks section is empty (no check names listed): tell the user no required status checks are configured for this repo and STOP.
   - If act jobs show "act not available": tell the user act is not installed or Podman socket is not active and STOP.
   - → If all prerequisites are met, proceed to Step 4.

4. **Step 4 — Run required checks via act:**
   For each required check name from the **Required PR checks** section, find the matching job in the **Available act jobs** section. Construct the act command using the exact template below.

   If no required check names match any act job (naming convention mismatch), tell the user which checks could not be matched and STOP. At least one job must match to proceed.

   Run each matching job. Report pass/fail for each.
   - → Proceed to Step 5.

5. **Step 5 — Handle failures:**
   - All passed → proceed to Step 6.
   - Failures detected → report which jobs failed and why. If failures appear to be Docker-related (container socket errors, Docker daemon not found), note these are expected with `--container-daemon-socket=-`. Ask the user whether to continue with the commit or stop.
   - User says stop → STOP.
   - User says continue → proceed to Step 6.

6. **Step 6 — Write commit message and commit:**
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
   - → Proceed to Step 7.

7. **Step 7 — STOP.**
   Do not suggest additional commits or show post-commit output. The user will inspect the result themselves if needed.

## Command template

The act command MUST always use this exact flag order to match the allowed-tools pattern:

```bash
DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock \
act pull_request \
  -P ubuntu-latest=docker.io/catthehacker/ubuntu:act-22.04@sha256:d83455c10c9a31c9c944a4c5628360c6c374983fa6616bd2439ab88b05ae2046 \
  --container-daemon-socket=- \
  --privileged=false \
  --insecure-secrets=false \
  --reuse \
  -W .github/workflows/<workflow-file> \
  -j <job-name>
```

Security flags are non-negotiable. The `--container-daemon-socket=-` flag disables Docker socket access inside the container. Jobs that require Docker (testcontainers, docker-compose, podman-compose) will fail — this is by design per security standards.

## Command hygiene

These constraints exist because the user's environment uses hooks to inspect Bash commands before execution.

- **No `;` chaining** — Semicolons defeat hook-based command inspection because the shell expands the full pipeline before hooks can evaluate individual commands. Use `&&` for dependent sequencing or separate Bash tool calls.
- **No `echo $?`** — The Bash tool already reports exit codes. Redundant and adds noise.

## Scope constraints

- **No `git add`** — This skill only commits what is already staged. The user controls staging.
- **No `git push`** — Pushing is a separate, explicit decision the user makes.
- **No `--amend`** — Amending rewrites history and can destroy prior commits, especially after a failed hook where the commit didn't actually happen.

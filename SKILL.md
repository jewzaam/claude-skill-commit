---
name: commit
description: Create a commit from staged changes after running the repository's validation checks. Use only when explicitly requested; do not invoke automatically.
---

# Commit

Use this skill only when the user explicitly asks to commit staged changes.

Run the installed skill's `scripts/commit` from the target repository root. The script gathers fresh git state, invokes `scripts/util/detect-checks.sh`, displays validation output, asks the configured read-only harness for a commit message, validates it, and performs the commit.

Do not run validation separately. Do not generate the commit message yourself. Do not run `git commit`, `git add`, `git push`, or `git commit --amend` yourself. The script is the authority for all of those workflow decisions.

Report the script's output to the user. If it exits non-zero, report the failure and log path, then stop. If it succeeds, report the created commit and log path, and stop.

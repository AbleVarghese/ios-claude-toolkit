---
name: ios-commit
description: Stage, commit, and optionally push with safety checks
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
---

Perform a safe commit using the heimdall-git skill.

1. Load the heimdall-git skill: read ${CLAUDE_PLUGIN_ROOT}/skills/heimdall-git/SKILL.md
2. Follow the skill's pre-operation and pre-commit phase gates:
   - Verify branch with `git branch --show-current`
   - Review changes with `git status` and `git diff --stat`
   - Stage specific files (never `git add -A`)
   - Generate conventional commit message from diff analysis
   - Verify no secrets or .env files staged
3. Commit with the generated message
4. Ask the user if they want to push

---
name: ios-review
description: Run a comprehensive iOS code review on changed files
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
---

Run a comprehensive iOS code review using the ios-review skill.

1. First, load the ios-review skill by reading ${CLAUDE_PLUGIN_ROOT}/skills/ios-review/SKILL.md
2. Identify changed files: `git diff --name-only HEAD~1..HEAD` or `git diff --name-only main..HEAD`
3. Follow the skill's 5-phase review protocol: Triage → Architecture → Concurrency → Security → Report
4. Follow ALL phase gates — do not skip any phase
5. Produce the structured review report as the required output artifact

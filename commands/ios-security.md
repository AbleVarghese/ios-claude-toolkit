---
name: ios-security
description: Run a security audit on the iOS project
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
---

Run a comprehensive security audit using the cipher-security skill.

1. Load the cipher-security skill: read ${CLAUDE_PLUGIN_ROOT}/skills/cipher-security/SKILL.md
2. Also read ${CLAUDE_PLUGIN_ROOT}/skills/cipher-security/references/fintech-compliance.md and ${CLAUDE_PLUGIN_ROOT}/skills/cipher-security/references/ios-security-patterns.md
3. Follow the skill's phase sequence: Secrets Scan FIRST → STRIDE → Analysis → Findings → Remediation
4. Follow ALL phase gates — secrets scan must complete before any other analysis
5. Produce the structured security assessment as the required output artifact

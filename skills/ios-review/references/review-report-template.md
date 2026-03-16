# Code Review Report Template

Use this template to structure all review output. Fill in every section.

---

```markdown
# Code Review Report

**Date**: YYYY-MM-DD
**Reviewer**: Claude AI
**Files Reviewed**: [count]
**Risk Profile**: [X HIGH, Y MEDIUM, Z LOW risk files]

---

## Summary

| Severity | Count |
|----------|-------|
| BLOCKER  | 0     |
| CRITICAL | 0     |
| WARNING  | 0     |
| NOTE     | 0     |

---

## Findings

| # | File:Line | Issue | Severity | Fix |
|---|-----------|-------|----------|-----|
| 1 | `Path/File.swift:42` | [Description] | BLOCKER | [Concrete fix] |
| 2 | `Path/File.swift:87` | [Description] | CRITICAL | [Concrete fix] |
| 3 | `Path/File.swift:15` | [Description] | WARNING | [Concrete fix] |

---

## Phase Results

### Phase 1: Triage
- HIGH: [list files]
- MEDIUM: [list files]
- LOW: [list files]

### Phase 2: Architecture Audit
- Layer violations: [count] found
- DI compliance: [PASS/FAIL]
- Dependency direction: [PASS/FAIL]

### Phase 3: Concurrency Audit
- @MainActor coverage: [PASS/FAIL]
- Sendable compliance: [PASS/FAIL]
- Data race risk: [NONE/LOW/HIGH]

### Phase 4: Security & Quality
- Force unwraps: [count]
- Secrets exposure: [NONE/FOUND]
- Error handling: [PASS/FAIL]

---

## Verdict

**Merge Status**: [APPROVED / BLOCKED / APPROVED WITH CONDITIONS]

**Blockers** (must fix before merge):
- [List BLOCKER findings by number]

**Required fixes** (should fix before merge):
- [List CRITICAL findings by number]

**Recommended improvements** (document for future):
- [List WARNING findings by number]

**Estimated fix time**: [X hours/minutes]

---

## Notes
[Any additional context, patterns observed, or suggestions for future work]
```

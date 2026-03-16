# Differential Security Review Report Template

Generate a comprehensive markdown report file using this template.

```markdown
# Differential Security Review Report

## Summary
- **Scope**: [PR #/commit range]
- **Files Analyzed**: X of Y changed files
- **Risk Level**: [CRITICAL/HIGH/MEDIUM/LOW]
- **Confidence**: [HIGH/MEDIUM/LOW]
- **Coverage Limitations**: [what was NOT reviewed and why]

## Findings

### Finding 1: [Title]
- **Severity**: CRITICAL/HIGH/MEDIUM/LOW
- **File**: path/to/file.swift:42
- **Risk Type**: [Auth/Crypto/Value Transfer/etc.]
- **Description**: [What is the issue]
- **Evidence**: [Git history, line numbers, code snippets]
- **Attack Scenario**: [How could this be exploited]
- **Recommendation**: [Specific fix with code example]
- **Blast Radius**: [X callers affected]
- **Test Coverage**: [Covered/Not Covered]

## Positive Observations
[Security improvements found in the diff]

## Test Coverage Gaps
[Changed code paths without tests]

## Recommendations
[Prioritized action items]

## Methodology
[What was analyzed, what was skipped, confidence level]
```

---
name: differential-review
description: "This skill should be used when the user asks to 'security review a PR', 'differential review', 'review code changes for security', 'check for security regressions', 'blast radius analysis', or mentions security-focused code review. Provides adaptive security-focused differential review of PRs, commits, and diffs with blast radius calculation, test coverage checks, and adversarial analysis."
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
---

# Differential Security Review

Security-focused code review for PRs, commits, and diffs.

Source: [github.com/trailofbits/skills](https://github.com/trailofbits/skills)

## Core Principles

1. **Risk-First**: Focus on auth, crypto, value transfer, external calls
2. **Evidence-Based**: Every finding backed by git history, line numbers, attack scenarios
3. **Adaptive**: Scale to codebase size (SMALL/MEDIUM/LARGE)
4. **Honest**: Explicitly state coverage limits and confidence level
5. **Output-Driven**: Always generate comprehensive markdown report file

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|-----------------|----------------|-----------------|
| "Small PR, quick review" | Heartbleed was 2 lines | Classify by RISK, not size |
| "I know this codebase" | Familiarity breeds blind spots | Build explicit baseline context |
| "Git history takes too long" | History reveals regressions | Never skip Phase 1 |
| "Blast radius is obvious" | Transitive callers get missed | Calculate quantitatively |
| "No tests = not my problem" | Missing tests = elevated risk rating | Flag in report, elevate severity |
| "Just a refactor, no security impact" | Refactors break invariants | Analyze as HIGH until proven LOW |
| "I'll explain verbally" | No artifact = findings lost | Always write report |

---

## Quick Reference

### Codebase Size Strategy

| Codebase Size | Strategy | Approach |
|---------------|----------|----------|
| SMALL (<20 files) | DEEP | Read all deps, full git blame |
| MEDIUM (20-200) | FOCUSED | 1-hop deps, priority files |
| LARGE (200+) | SURGICAL | Critical paths only |

### Risk Level Triggers

| Risk Level | Triggers |
|------------|----------|
| HIGH | Auth, crypto, external calls, value transfer, validation removal |
| MEDIUM | Business logic, state changes, new public APIs |
| LOW | Comments, tests, UI, logging |

---

## Workflow Overview

```
Pre-Analysis -> Phase 0: Triage -> Phase 1: Code Analysis -> Phase 2: Test Coverage
    |              |                    |                        |
Phase 3: Blast Radius -> Phase 4: Deep Context -> Phase 5: Adversarial -> Report
```

---

## Phase 0: Triage

Classify each changed file by risk level:

```bash
# Get changed files
git diff --name-only HEAD~1..HEAD
# Or for a PR
git diff --name-only main..HEAD
```

**Risk classification:**
- **HIGH**: Files touching auth, crypto, payments, validation, access control, external calls
- **MEDIUM**: Business logic, state management, API endpoints, data models
- **LOW**: Comments, documentation, UI styling, test files, logging

## Phase 1: Code Analysis

For each HIGH and MEDIUM risk file:

1. **Read the diff** carefully, line by line
2. **Check git blame** on removed security-relevant code:
   ```bash
   git log --oneline -10 -- path/to/file.swift
   git blame path/to/file.swift
   ```
3. **Trace data flow** from inputs to outputs
4. **Identify trust boundaries** crossed by the change

## Phase 2: Test Coverage

For each changed file, verify:

```bash
# Find related test files
find . -name "*Tests.swift" | xargs grep -l "ClassName"

# Check if new code paths have tests
grep -n "func test" path/to/related/Tests.swift
```

Flag any HIGH risk changes without corresponding test coverage.

## Phase 3: Blast Radius

Calculate how many callers are affected:

```bash
# Find all callers of changed function
grep -rn "functionName" --include="*.swift" .

# Count affected files
grep -rl "functionName" --include="*.swift" . | wc -l
```

| Blast Radius | Callers | Action |
|-------------|---------|--------|
| LOW | 0-5 | Standard review |
| MEDIUM | 6-20 | Check all callers |
| HIGH | 21-50 | Sample callers + elevate risk |
| CRITICAL | 50+ | Full impact analysis required |

## Phase 4: Deep Context

For HIGH risk changes, gather additional context:

- **Git history**: Was this code previously fixed for security? Check commit messages for "fix", "CVE", "security", "vulnerability"
- **Related issues**: Are there open issues related to this code?
- **Documentation**: Does architecture documentation mention security constraints?
- **Dependencies**: Do changed dependencies have known vulnerabilities?

## Phase 5: Adversarial Analysis

For HIGH risk findings, model attacker perspective:

1. **Attacker Goal**: What could an attacker achieve by exploiting this?
2. **Attack Surface**: What inputs does the attacker control?
3. **Exploit Scenario**: Step-by-step exploitation
4. **Impact**: Data breach, financial loss, privilege escalation, DoS
5. **Exploitability**: How difficult is exploitation? (EASY/MEDIUM/HARD)

See `references/vulnerability-patterns.md` for common and iOS-specific vulnerability pattern tables.

Use `references/report-template.md` for the Phase 6 report output format.

---

## Quality Checklist

Before delivering:

- [ ] All changed files analyzed
- [ ] Git blame on removed security code
- [ ] Blast radius calculated for HIGH risk
- [ ] Attack scenarios are concrete (not generic)
- [ ] Findings reference specific line numbers + commits
- [ ] Report file generated
- [ ] User notified with summary

---

## Red Flags (Stop and Investigate)

**Immediate escalation triggers:**
- Removed code from "security", "CVE", or "fix" commits
- Access control modifiers removed (e.g., `private` -> `public`, `internal` -> `open`)
- Validation removed without replacement
- External calls added without checks
- High blast radius (50+ callers) + HIGH risk change

These patterns require adversarial analysis even in quick triage.

---

## When NOT to Use This Skill

- **Greenfield code** (no baseline to compare)
- **Documentation-only changes** (no security impact)
- **Formatting/linting** (cosmetic changes)
- **User explicitly requests quick summary only** (they accept risk)

For these cases, use standard code review instead.

---

## Tips for Best Results

**Do:**
- Start with git blame for removed code
- Calculate blast radius early to prioritize
- Generate concrete attack scenarios
- Reference specific line numbers and commits
- Be honest about coverage limitations
- Always generate the output file

**Don't:**
- Skip git history analysis
- Make generic findings without evidence
- Claim full analysis when time-limited
- Forget to check test coverage
- Miss high blast radius changes
- Output report only to chat (file required)

## References

- [Trail of Bits Skills](https://github.com/trailofbits/skills)
- [Trail of Bits Blog](https://blog.trailofbits.com/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
- `references/vulnerability-patterns.md` -- Common + iOS/Swift vulnerability patterns
- `references/report-template.md` -- Phase 6 report output template

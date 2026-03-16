---
name: ios-review
description: "Check my code, is this correct, what do you think of this, review my changes, pre-merge check, PR feedback, look over my implementation. Comprehensive iOS code review covering architecture, SwiftUI patterns, concurrency, security, and accessibility. Use for code quality checks, architecture violation audits, pre-merge review, or any request for feedback on Swift/SwiftUI code. Trigger on 'does this look right', 'anything wrong here', 'review before I merge'. For implementing fixes, use forge-development. For security-focused PR review, use trailofbits-differential-review."
user-invocable: true
allowed-tools:
  - Read
  - Grep
  - Glob
---

# iOS Code Review — 5-Phase Review Protocol

Perform comprehensive iOS-specific code review covering architecture compliance, SwiftUI patterns, concurrency safety, security, and accessibility. Every finding is specific: file, line, issue, fix.

> **Iron Law**: "Every review finding must cite a specific file, line number, and concrete fix — never generic advice."

> **Project Discovery**: Before executing, determine project-specific values (project name, scheme, bundle ID, target, architecture layers, ViewModel base class, dependency flow) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

---

## Pre-Review Gate: Is This a Review or a Refactor?

```
START: You've been asked to review code.

Q1: Are you being asked to READ and ASSESS code?
├── YES → This is a review. Proceed with the 5-phase protocol below.
└── NO →
    Q2: Are you being asked to CHANGE or FIX code?
    ├── YES → This is implementation work. Use `forge-development` skill.
    └── UNCLEAR → Ask the user. Do not guess.

CRITICAL GATE: During review, you MAY NOT:
  - Edit files (review is read-only)
  - Refactor code (that's a separate task)
  - Implement fixes (report them; owner decides priority)
  - Expand scope beyond changed files (unless investigating a finding)
```

**Why this gate exists**: Review scope creep is the most common way reviews become multi-hour refactoring sessions. A review produces a report. Fixing the findings is a separate task with its own prioritization.

---

## Phase 1: Triage — Classify Changed Files by Risk

Before reading code, classify every changed file. This determines review depth.

### Risk Classification

| Risk Level | What It Touches | Review Depth | Examples |
|-----------|----------------|-------------|---------|
| **HIGH** | Auth, crypto, payments, keychain, token handling, PII | Line-by-line, every path | `AuthService.swift`, `KeychainManager.swift`, `PaymentViewModel.swift` |
| **MEDIUM** | Business logic, state management, data models, navigation | Full method review, all error paths | `TransferService.swift`, `UserProfileViewModel.swift`, `AppRouter.swift` |
| **LOW** | Tests, UI styling, documentation, asset catalogs | Scan for anti-patterns | `TransferServiceTests.swift`, `Colors.swift`, `README.md` |

### Triage Process

1. List all changed files (from diff or file list)
2. Classify each file as HIGH / MEDIUM / LOW
3. Order review: HIGH files first, then MEDIUM, then LOW
4. Budget time: 60% on HIGH, 30% on MEDIUM, 10% on LOW

```
Example Triage Output:

HIGH RISK (review line-by-line):
  - Sources/Services/AuthService.swift (auth logic)
  - Sources/Infrastructure/KeychainAdapter.swift (credential storage)

MEDIUM RISK (full method review):
  - Sources/ViewModels/TransferViewModel.swift (state management)
  - Sources/Domain/TransferService.swift (business logic)

LOW RISK (scan for anti-patterns):
  - Tests/TransferServiceTests.swift (test coverage)
  - Sources/Views/TransferDetailView.swift (UI only)
```

### Phase Gate: TRIAGE
**Cannot proceed to ANALYSIS until:**
- [ ] EVERY changed file is listed with risk classification (HIGH/MEDIUM/LOW)
- [ ] No file is unclassified
- [ ] HIGH files are identified for line-by-line review

**If you classified everything as LOW**: Recheck. PRs that touch auth, payments, or data models are never LOW. If everything genuinely is LOW, document why.

---

## Phase 2: Architecture Audit

Check for structural violations that undermine the entire codebase. These are always BLOCKER or CRITICAL severity.

### Checks

1. **Layer violations** — Does any file import from a layer it shouldn't?
   - Domain imports SwiftUI/UIKit? → BLOCKER
   - Presentation imports Infrastructure directly? → CRITICAL
   - Infrastructure imports Presentation? → BLOCKER

2. **Dependency injection compliance** — Are all dependencies protocol-based and injected?
   - `.shared` singleton access? → CRITICAL
   - Concrete type used instead of protocol? → CRITICAL
   - Missing init-based injection? → CRITICAL

3. **Dependency direction** — Do all arrows point inward?
   - Verify: Presentation → Domain ← Infrastructure
   - Any outward dependency is a violation

4. **Import analysis** — Grep for suspicious imports:
   ```
   Grep for: import UIKit in Domain/
   Grep for: import SwiftUI in Domain/
   Grep for: import (InfrastructureModule) in Presentation/
   Grep for: .shared in ViewModels/
   ```

### Architecture Findings Template

```
FINDING: Layer violation — Domain imports SwiftUI
FILE: Sources/Domain/Entities/User.swift:3
SEVERITY: BLOCKER
CURRENT: import SwiftUI
FIX: Remove SwiftUI import. Use platform-agnostic types (String for color hex, Foundation.URL for URLs).
     Map to SwiftUI types in Presentation layer extension.
```

---

## Phase 3: Concurrency Audit

Swift 6 strict concurrency violations cause data races that crash at runtime. These are often invisible without careful review.

### Checks

1. **@MainActor on ViewModels** — Every ViewModel class must be `@MainActor`
   ```
   Grep for: class.*ViewModel.*ObservableObject
   Verify each has @MainActor annotation
   Missing? → BLOCKER
   ```

2. **@Published mutation isolation** — `@Published` properties must only be mutated from `@MainActor` context
   ```
   Look for: @Published var modifications inside Task {} without @MainActor
   Found? → BLOCKER (data race)
   ```

3. **Sendable conformance** — Types crossing concurrency domains must be Sendable
   ```
   Check: Are entities passed between actors marked Sendable?
   Check: Are closures passed to Task {} marked @Sendable?
   Missing conformance? → CRITICAL
   ```

4. **Actor isolation** — Mutable shared state must be protected
   ```
   Look for: var properties on classes without actor isolation
   Look for: mutable state accessed from multiple Task {} blocks
   Found? → BLOCKER (data race)
   ```

5. **@unchecked Sendable audit** — Each usage must have a justification comment
   ```
   Grep for: @unchecked Sendable
   Missing justification? → CRITICAL
   Wrapping mutable state without synchronization? → BLOCKER
   ```

### Concurrency Findings Template

```
FINDING: Missing @MainActor on ViewModel
FILE: Sources/ViewModels/TransferViewModel.swift:5
SEVERITY: BLOCKER
CURRENT: final class TransferViewModel: ObservableObject {
FIX: @MainActor final class TransferViewModel: ObservableObject {
REASON: @Published properties are mutated in async methods. Without @MainActor,
        these mutations are data races under Swift 6 strict concurrency.
```

---

## Phase 4: Security & Quality Scan

### Security Checks

| Check | Pattern to Find | Severity | Fix |
|-------|----------------|----------|-----|
| Force unwraps | `!` (not `!=`) | CRITICAL | Replace with `guard let` or `if let` |
| Hardcoded secrets | API keys, tokens, passwords in source | BLOCKER | Move to environment/config, load at runtime |
| PII in logs | `print()`, `logger.info()` with user data | BLOCKER | Remove PII, log only correlation IDs |
| UserDefaults for secrets | `UserDefaults` storing tokens/passwords | BLOCKER | Use Keychain via KeychainAdapter |
| Try without catch | `try?` silently discarding errors | WARNING | Handle error explicitly or document why ignored |
| Force cast | `as!` | CRITICAL | Use `as?` with guard or if-let |
| Implicitly unwrapped optionals | `var x: Type!` | CRITICAL | Use `Type?` with proper unwrapping |

### Quality Checks

| Check | Pattern to Find | Severity | Fix |
|-------|----------------|----------|-----|
| Completion handlers | `completionHandler:`, `callback:`, `{ result in` | CRITICAL | Convert to async/await |
| Retain cycles | `self.` in closures without `[weak self]` | WARNING | Add `[weak self]` or verify no cycle exists |
| View body complexity | `body` > 30 lines or > 3 nesting levels | WARNING | Extract subviews with @ViewBuilder |
| Missing error handling | `catch { }` (empty catch) | CRITICAL | Handle or log the error |
| Hardcoded strings | User-facing strings not in localization | WARNING | Use `String(localized:)` |
| File size | > 300 lines | WARNING | Split by concern |
| N+1 queries | Loop containing async call per item | WARNING | Batch request or use TaskGroup |

### Security Findings Template

```
FINDING: Sensitive data stored in UserDefaults
FILE: Sources/Services/AuthService.swift:47
SEVERITY: BLOCKER
CURRENT: UserDefaults.standard.set(token, forKey: "auth_token")
FIX: Use Keychain via KeychainAdapter:
     try await keychainAdapter.store(token, for: .authToken)
REASON: UserDefaults is not encrypted. Tokens stored there are readable by
        any process with file system access to the app's container.
```

---

### Phase Gate: ANALYSIS
**Cannot proceed to REPORT until:**
- [ ] Every HIGH file has been read line-by-line
- [ ] Every finding follows format: `[File:Line] | [Issue] | [Severity] | [Fix]`
- [ ] Every reviewed file has at least ONE observation (finding OR explicit "no issues — checked X, Y, Z")

**Hard Stop**: Generic findings are not findings. "Code looks clean" is not an observation. "Checked TransferService.swift:42-89 for actor isolation — all @MainActor usage correct, no cross-actor access" IS an observation. Be specific or go back.

**Counter**: If you found zero issues across 5+ changed files, you almost certainly missed something. Return to the highest-risk file and look again — specifically for: missing error handling, force unwraps, concurrency violations, missing accessibility labels.

---

## Phase 5: Report Generation

Compile all findings into a structured report. The report is the deliverable of a review.

### Report Format

```markdown
# Code Review Report

**Reviewed**: [date]
**Files**: [count] files reviewed
**Risk Profile**: [X HIGH, Y MEDIUM, Z LOW]

## Summary

| Severity | Count |
|----------|-------|
| BLOCKER  | X     |
| CRITICAL | Y     |
| WARNING  | Z     |
| NOTE     | W     |

## Findings

| # | File:Line | Issue | Severity | Fix |
|---|-----------|-------|----------|-----|
| 1 | `AuthService.swift:47` | Token stored in UserDefaults | BLOCKER | Use KeychainAdapter |
| 2 | `TransferViewModel.swift:5` | Missing @MainActor | BLOCKER | Add @MainActor annotation |
| 3 | `User.swift:3` | Domain imports SwiftUI | BLOCKER | Remove import, use Foundation types |
| 4 | `PaymentView.swift:82` | Force unwrap on optional | CRITICAL | Use guard let |
| 5 | `TransferService.swift:15` | Empty catch block | CRITICAL | Log error or propagate |

## Verdict

- **MERGE BLOCKED**: [X] BLOCKER findings must be fixed first
- **Estimated fix time**: [estimate based on findings]
- **Recommended priority**: Fix blockers → Fix criticals → Address warnings in next PR
```

### Phase Gate: REPORT
**Report must include ALL of these sections (missing = incomplete):**
- [ ] Summary table: `| # | File:Line | Issue | Severity | Fix |`
- [ ] Files reviewed count and risk breakdown
- [ ] At least one finding per HIGH-risk file (or documented "no issues with evidence")
- [ ] Recommendation: APPROVE / REQUEST CHANGES / BLOCK (with justification)

**Hard Stop**: If you're about to write "LGTM" or "looks good" without the summary table — STOP. That is not a code review. Return to ANALYSIS.

### Severity Definitions

| Severity | Definition | Merge Impact |
|----------|-----------|-------------|
| **BLOCKER** | Will cause crashes, data loss, security breach, or data race at runtime | Cannot merge. Must fix. |
| **CRITICAL** | Significant code quality issue, architectural violation, or potential bug | Should fix before merge. Rare exceptions with owner approval. |
| **WARNING** | Improvement opportunity, minor code quality issue, or tech debt | Document for future. Does not block merge. |
| **NOTE** | Style preference, minor suggestion, or informational observation | Optional. |

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|---------------|-----------------|
| "It's just a small PR" | Small PRs with auth/payment changes are the highest-risk reviews. Size doesn't determine risk. | Triage by risk, not size. Small HIGH-risk PRs get full review. |
| "I know this code already" | Familiarity breeds blindness. You miss changes precisely because you think you know the file. | Read the actual diff. Every line. No skimming. |
| "The tests pass so it's fine" | Tests verify behavior, not architecture, security, or concurrency safety. Passing tests with data races still crash in production. | Tests passing is necessary but not sufficient. Run all 5 phases. |
| "I'll do a deeper review next time" | Next time never comes. Every PR merged with known issues is tech debt at compound interest. | Review to completion now. If time-constrained, reduce scope, don't reduce depth. |
| "The linter would catch that" | Linters don't catch architecture violations, data races, business logic errors, or security issues. | Linters are Phase 0. Human review covers what automation cannot. |
| "It's test code, doesn't need review" | Bad tests give false confidence. Flaky tests waste hours. Tests without assertions pass but prove nothing. | Review tests for: assertions present, no shared state, no sleep, proper mocking. |

---

## Red Flags — HARD STOP

These are BLOCKING conditions, not suggestions. If ANY red flag is triggered, you CANNOT continue the current review until the violation is remediated. Proceeding past a red flag invalidates all subsequent findings.

- **Reviewing without reading the full diff**: You must read every changed line. Skimming is not reviewing.
- **Skipping concurrency checks**: Data races are the most common production crash in Swift 6. Never skip Phase 3.
- **Generic "looks good" with no specific findings**: Every review must produce at least one finding or explicit confirmation per phase. "LGTM" without evidence is not a review.
- **Missing force unwrap detection**: Force unwraps are the most common crash cause. Every `!` must be accounted for.
- **Scope creep into refactoring**: If you start editing files during review, stop. Write the finding, move on.

---

## When NOT to Use This Skill

- **Implementing new features or fixing bugs** → Use `forge-development`
- **Performance profiling with Instruments** → Use `prometheus-performance`
- **Accessibility-only audit** → Use `athena-accessibility`
- **Security-focused penetration review** → Use `cipher-security`
- **Git operations during or after review** → Use `heimdall-git`

---

## Decision Framework: Review Workflow

```
START: Code review requested.

Phase 1: TRIAGE
├── List all changed files
├── Classify each as HIGH / MEDIUM / LOW risk
└── Order: HIGH first, then MEDIUM, then LOW

Phase 2: ARCHITECTURE AUDIT
├── Check layer violations (imports crossing boundaries)
├── Check DI compliance (protocol-based injection)
├── Check dependency direction (inward only)
└── Record findings with file:line:severity:fix

Phase 3: CONCURRENCY AUDIT
├── @MainActor on all ViewModels?
├── @Published mutations isolated to main actor?
├── Sendable conformance for cross-actor types?
├── @unchecked Sendable justified?
└── Record findings with file:line:severity:fix

Phase 4: SECURITY & QUALITY SCAN
├── Force unwraps, force casts?
├── Secrets in source code?
├── PII in logs?
├── Keychain vs UserDefaults for sensitive data?
├── Error handling (no empty catches, no bare try?)
├── View body complexity?
└── Record findings with file:line:severity:fix

Phase 5: REPORT
├── Compile findings table
├── Summarize by severity
├── Determine merge verdict
└── Estimate fix time
```

---

## Scale Mode (50+ Changed Files)

When reviewing large scopes (codebase-wide audits, major refactors, 50+ files), the standard per-file triage breaks down. Switch to module-level analysis:

### Module-Level Triage
Instead of classifying every file individually, classify by module/directory:
```
| Module | File Count | Risk | Review Depth |
|--------|-----------|------|-------------|
| Auth/ | 12 files | HIGH | Grep patterns + sample 3 files |
| Payments/ | 8 files | HIGH | Grep patterns + sample 3 files |
| UI/Components/ | 25 files | LOW | Grep patterns only |
```

### Grep-First Workflow
For scale reviews, use automated pattern detection before manual review:
```bash
# Architecture violations
grep -rn "import Supabase\|import Stripe" --include="*.swift" Sources/Domain/ Sources/Presentation/

# Force unwraps
grep -rn '!' --include="*.swift" Sources/ | grep -v '//' | grep -v 'IBOutlet'

# Missing @MainActor on ViewModels
grep -rln "ObservableObject" --include="*.swift" Sources/ | xargs grep -L "@MainActor"

# Concurrency violations
grep -rn "\.shared" --include="*.swift" Sources/
```

### Scale Review Report
For 50+ files, the report focuses on **patterns found** rather than per-file findings:
- Pattern 1: [X occurrences of Y across Z modules]
- Pattern 2: ...
- Sampled files reviewed in detail: [list]
- Modules requiring deeper review: [list]

### When NOT to Use Scale Mode
- PRs under 50 files — use standard per-file triage
- Security-focused reviews — use trailofbits-differential-review regardless of size

---

## Quality Gates (Before Marking Complete)

- [ ] Every changed file has been read in full (not skimmed)
- [ ] All files classified by risk level (HIGH/MEDIUM/LOW)
- [ ] Architecture audit completed — imports and DI checked
- [ ] Concurrency audit completed — @MainActor, Sendable, actor isolation checked
- [ ] Security scan completed — force unwraps, secrets, PII, keychain usage checked
- [ ] Every finding has: file path, line number, severity, concrete fix
- [ ] Report generated with summary table and merge verdict
- [ ] No generic "looks good" — specific evidence for each pass/fail
- [ ] Scope maintained — no files edited during review

---

## Self-Audit (Before Declaring Review Complete)

Answer each question. If ANY answer is "no" or "unsure" — go back and fix it.

1. Did I complete every phase (TRIAGE, ARCHITECTURE, CONCURRENCY, SECURITY/QUALITY, REPORT)?
2. Did I read every HIGH-risk file line-by-line? (List them)
3. Does every finding cite a specific file and line number with a concrete fix?
4. Did I skip any checks because they "seemed unnecessary"? (If yes, go back)
5. Would a senior iOS engineer find gaps in this review? (Be honest)
6. Did I follow the Iron Law? (Every finding has file:line:severity:fix — cite evidence)

---

## Required Output Artifact

Every code review using this skill must produce a **Review Report** containing:
1. **Triage table** — every changed file classified by risk level
2. **Findings table** — `| # | File:Line | Issue | Severity | Fix |`
3. **Severity summary** — counts by BLOCKER / CRITICAL / WARNING / NOTE
4. **Verdict** — APPROVE / REQUEST CHANGES / BLOCK with justification
5. **Files reviewed count** — with risk breakdown

A review without a findings table is not a review. A review that says "LGTM" without evidence is not a review. Missing any section above = INCOMPLETE.

---

## Cross-Skill References

| Skill | When to Use |
|-------|-------------|
| `forge-development` | When findings need to be fixed — switch from review to implementation mode |
| `prometheus-performance` | When review uncovers performance concerns that need profiling |
| `athena-accessibility` | When review finds missing VoiceOver labels or Dynamic Type issues |
| `cipher-security` | When review finds security issues requiring deeper security analysis |
| `heimdall-git` | After review is complete — commit the review report or manage branches |
| `documentation` | When review findings affect project documentation that needs updating |

---

## References

- **Review Checklist**: `references/review-checklist.md` — detailed checklist organized by category with severity levels
- **Common Violations**: `references/common-violations.md` — top 30 iOS violations with bad/good examples and fixes
- **Report Template**: `references/review-report-template.md` — markdown template for structured review output

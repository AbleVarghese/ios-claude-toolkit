---
name: sentry-code-review
description: "Sentry flagged something, Sentry bot commented, Sentry comments on PR, fix what Sentry found. Review and resolve Sentry bot findings on GitHub PRs — bug prediction comments covering security, performance, error handling, type safety, and concurrency. Trigger when: Sentry has commented on a PR, user shares a PR with Sentry feedback, 'address Sentry review', or 'what did Sentry find'. For production crashes from Sentry monitoring (not PR comments), use sentry-find-bugs instead."
user-invocable: true
allowed-tools:
  - Read
  - Edit
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# Sentry PR Code Review

> **Iron Law**: "Verify every Sentry finding against the actual current code before acting -- findings may be stale or false positives."

Review and resolve code issues identified by Sentry's bug prediction bot in GitHub pull requests. Sentry's static analysis catches real bugs, but it also produces false positives. Your job is to distinguish the two, fix the real ones, and document your reasoning for dismissals.

Source: [github.com/getsentry/agent-skills](https://github.com/getsentry/agent-skills)

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|-----------------|----------------|-----------------|
| "Sentry flags too many things" | Volume doesn't mean low quality. Even if 30% are false positives, the 70% that are real bugs justify the review. | Evaluate each finding individually. Never dismiss in bulk. |
| "This is a false positive, I'm sure" | Confidence without verification is guessing. The code may have changed since you last looked at it. | Read the actual current code at the flagged location. Verify against the finding's specific claim. See `references/false-positive-guide.md`. |
| "The finding is low severity, skip it" | Low severity findings compound. A "minor" missing nil check becomes a production crash when an API changes its response format. | Fix low severity findings if the fix is straightforward (<5 min). Document and track if the fix requires architecture changes. |
| "I already know what the fix is" | Knowing the fix without reading the finding risks fixing the wrong thing. Sentry may have identified a subtlety you missed. | Read the full finding description, including the AI prompt and analysis sections. Then implement your fix. |
| "Sentry doesn't understand Swift well" | Sentry's Swift analysis is imperfect but catches real patterns: retain cycles, actor isolation violations, optional chaining gaps. | Verify the finding against Swift semantics. Dismiss with documented reasoning if it's a language misunderstanding. |
| "The PR is already approved, don't block it" | Sentry findings after approval indicate the reviewer may have missed something. Merging known issues creates tech debt on purpose. | Evaluate severity. Critical/High findings block merge. Medium/Low can be tracked as follow-up with owner agreement. |

---

## Red Flags -- HARD STOP

These are not warnings. Each is a BLOCKING condition that prevents further progress until addressed.

- **BLOCKED: Dismissing findings without reading the actual code** -- opening the file and reading the flagged lines is non-negotiable. Sentry provides file paths and line numbers; use them. STOP. Read the code at the exact file and line number before making any judgment.
- **BLOCKED: Fixing symptoms instead of root cause** -- if Sentry flags a nil check issue, don't just add `guard let`. STOP. Ask why the value could be nil at that point and fix the data flow.
- **BLOCKED: Not verifying the fix addresses the specific Sentry finding** -- after implementing a fix, re-read the finding and confirm your change resolves its specific concern. If you cannot map your fix to the finding, STOP.
- **BLOCKED: Bulk-dismissing findings as false positives** -- each finding requires individual evaluation. If you're dismissing more than 50% as false positives, STOP, slow down and re-examine your dismissal criteria.
- **BLOCKED: Ignoring concurrency findings** -- actor isolation and Sendable violations are the most commonly dismissed Swift findings, but they're also the most dangerous (data races cause heisenbugs). STOP. These require analysis, not dismissal.

---

## When NOT to Use This Skill

- **Investigating production crashes from Sentry monitoring** -- use `sentry-find-bugs` instead, which is designed for analyzing runtime errors with stack traces, breadcrumbs, and event data.
- **Performing a broad security audit** -- use `cipher-security` for application-level security posture assessment.
- **The PR has no Sentry bot comments** -- if there are no Sentry findings, use `ios-review` or `trailofbits-differential-review` for general or security-focused code review.
- **Performance profiling** -- use `prometheus-performance`. However, if Sentry flags a performance-related issue (N+1 queries, main thread work), this skill handles the finding.

---

## Behavioral Enforcement

### Phase Gate: CODE VERIFICATION (For Every Finding)
**Cannot act on ANY Sentry finding until:**
- [ ] Read the ACTUAL current code at the flagged file and line number
- [ ] Verified the finding is still relevant (code may have changed since Sentry flagged it)
- [ ] Checked if Swift's type system already prevents the flagged issue

**Hard Stop**: If about to dismiss a finding as "false positive" without reading the code -- STOP. Read the code. The finding may be stale (true), but you can't know that without looking.

**Hard Stop**: If about to implement Sentry's suggested fix verbatim without understanding it -- STOP. Sentry suggestions are starting points, not solutions. Understand the context before applying.

### Counter-Based Escalation
- **Finding #1-5**: Normal. Evaluate each individually.
- **Finding #6-10**: Pause. Are there patterns? Are multiple findings pointing to the same architectural issue?
- **Finding #11+**: STOP. This PR likely has systemic issues. Consider whether the PR should be split, refactored, or sent back for design review before fixing individual findings.

---

## Decision Framework

```
Sentry commented on a PR
|
+-- Fetch all Sentry comments (Phase 1)
|
+-- For each finding:
|   |
|   +-- Read the actual code at the flagged location
|   |
|   +-- Is the finding about current code?
|   |   +-- NO (code already changed) --> Mark as STALE, skip
|   |   +-- YES --> Continue evaluation
|   |
|   +-- Is the finding accurate?
|   |   +-- Check references/false-positive-guide.md for known false positive patterns
|   |   +-- Is the flagged path actually reachable?
|   |   +-- Does Swift's type system already prevent the issue?
|   |   |
|   |   +-- FALSE POSITIVE --> Document reasoning, mark as FALSE POSITIVE
|   |   +-- TRUE POSITIVE --> Continue to severity assessment
|   |
|   +-- What severity?
|       +-- CRITICAL: Security vulnerability, data loss, crash in happy path
|       |   --> Fix immediately, block merge until fixed
|       +-- HIGH: Crash in edge case, data corruption, concurrency bug
|       |   --> Fix before merge if possible, escalate if complex
|       +-- MEDIUM: Error handling gap, missing validation, code smell
|       |   --> Fix if <5 min, otherwise create tracked follow-up
|       +-- LOW: Style, minor improvement, defensive programming
|           --> Fix if trivial, otherwise note for future
|
+-- Generate summary report (Phase 4)
```

---

## Invoke This Skill When

- User asks to "review Sentry comments" or "fix Sentry issues" on a PR
- User shares a PR URL/number mentioning Sentry feedback
- User asks to "address Sentry review" or "resolve Sentry findings"
- User wants to find PRs with unresolved Sentry comments

---

## Phase 1: Fetch Sentry Comments

Retrieve all comments from the Sentry bot on the target PR using GitHub's API:

```bash
# Fetch PR review comments, filtering for sentry[bot]
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments | \
  jq '[.[] | select(.user.login == "sentry-bot" or .user.login == "github-actions[bot]")]'

# Also check issue comments (review comments are separate)
gh api repos/{owner}/{repo}/issues/{pr_number}/comments | \
  jq '[.[] | select(.user.login == "sentry-bot")]'
```

If no PR specified, find recent PRs with Sentry comments:

```bash
# List recent PRs
gh pr list --limit 20 --json number,title,url
```

---

## Phase 2: Parse and Classify Findings

Extract structured data from Sentry's markdown comments:

| Field | What to Extract | Why It Matters |
|-------|----------------|----------------|
| **Bug Description** | Short summary of the identified issue | Tells you what to look for in the code |
| **Severity** | Critical / High / Medium / Low | Determines fix priority and merge-blocking status |
| **Confidence** | How certain Sentry is about the finding | Low confidence = higher false positive likelihood |
| **Analysis** | Detailed explanation of the potential bug | Contains the specific claim to verify |
| **Suggested Fix** | Sentry's recommended code change | Starting point, but verify it's correct for your codebase |
| **AI Prompt** | Context for understanding the issue | Additional context about the detection methodology |

### Severity Calibration

Sentry's severity ratings are a starting point, not gospel. Calibrate based on context:

**When Sentry says High but actual risk is lower:**
- The finding is in dead code or an unreachable path
- Swift's type system already prevents the issue (e.g., non-optional flagged as potential nil)
- The finding is about test code, not production code
- The issue requires conditions that are architecturally impossible in your app

**When Sentry says Low but actual risk is higher:**
- The finding is in authentication, authorization, or payment code (any severity in security-critical code is HIGH)
- The finding involves concurrency (data races are hard to detect and high impact)
- The finding is about error handling in a path that processes financial transactions
- The finding has high blast radius (many callers affected)

---

## Phase 3: Verify and Implement Fixes

For each identified issue:

### Step 1: Read the Actual Code

```bash
# Read the file at the flagged location
# Always read surrounding context (at least 20 lines above and below)
```

Do not rely on Sentry's code snippet -- it may be from an older commit. Read the current state of the file.

### Step 2: Verify the Finding

Check against `references/false-positive-guide.md` for known false positive patterns. Then ask:

1. **Is the flagged code path actually reachable?** Trace callers backward. If no execution path reaches this code with the problematic state, it's likely a false positive.
2. **Does Swift's type system already prevent the issue?** If Sentry flags a nil check issue but the variable is non-optional, it's a false positive.
3. **Has the code been modified since Sentry analyzed it?** If the PR has new commits after the Sentry comment, the finding may be stale.

### Step 3: Implement the Fix

For true positives, implement considering:

- **Edge cases**: nil, empty collections, malformed inputs, unexpected API responses
- **Error propagation**: Does the caller handle errors from this function? Will your fix change the error contract?
- **Type safety**: Can types be narrowed or validated earlier in the pipeline?
- **Concurrency**: Race conditions, actor isolation, Sendable conformance
- **Performance implications**: Will the fix add latency, memory, or CPU usage?

```swift
// EXAMPLE: Sentry flags missing nil check on API response decoding

// BAD: Just adding a nil check (treats symptom)
func loadUser() async {
    let response = try? await api.fetchUser()
    guard let user = response else { return } // Silently fails
    self.user = user
}

// GOOD: Fix with proper error handling and context
func loadUser() async throws {
    do {
        let response = try await api.fetchUser()
        guard let userData = response.data else {
            logger.warning("load_user_empty_response", [
                "status_code": response.statusCode,
                "endpoint": "fetchUser"
            ])
            throw AppError.emptyResponse(endpoint: "fetchUser")
        }
        self.user = try UserProfile(from: userData)
    } catch let decodingError as DecodingError {
        logger.error("load_user_decode_failure", [
            "error": decodingError.localizedDescription
        ])
        throw AppError.decodingFailed(underlying: decodingError)
    }
}
```

### Step 4: Verify Fix Addresses Finding

After implementing, re-read the Sentry finding and confirm:
- [ ] The specific concern raised by Sentry is addressed
- [ ] The fix doesn't introduce new issues (nil safety, error handling, concurrency)
- [ ] The fix is consistent with codebase patterns

---

## Phase 4: Report Results

Provide a summary table:

```markdown
| # | File | Issue | Severity | Sentry Confidence | Status | Notes |
|---|------|-------|----------|-------------------|--------|-------|
| 1 | TransferService.swift:42 | Missing nil check | High | High | FIXED | Added guard + error propagation |
| 2 | PaymentHandler.swift:89 | Unhandled error | Medium | Medium | FIXED | Added do-catch with specific error types |
| 3 | WalletView.swift:15 | Unused variable | Low | Low | FALSE POSITIVE | Used in SwiftUI body property |
| 4 | AuthService.swift:201 | Race condition | Critical | High | NEEDS REVIEW | Requires architecture decision -- escalating |

### False Positive Justifications
- **#3 WalletView.swift:15**: Sentry flagged `balance` as unused, but it is referenced inside
  the SwiftUI `body` computed property. Sentry's analysis does not fully model SwiftUI's
  declarative property access pattern. (See false-positive-guide.md: "SwiftUI body property")

### Escalations
- **#4 AuthService.swift:201**: Race condition between token refresh and API call requires
  architectural decision on whether to serialize requests or implement a token refresh lock.
  Presenting options to owner.
```

---

## Common Issue Categories

### Type Safety
- Missing null/nil checks before force unwrap
- Unsafe type casting without validation
- Optional chaining gaps where a nil intermediate value causes silent failure
- Generic type constraints that are too permissive

```swift
// Sentry flags: Potential force unwrap crash
// BEFORE
let name = user.profile!.displayName

// AFTER: Guard with meaningful error
guard let profile = user.profile else {
    logger.error("missing_user_profile", ["user_id": user.id.uuidString])
    throw AppError.missingProfile(userId: user.id)
}
let name = profile.displayName
```

### Error Handling
- Swallowed errors (empty catch blocks)
- Missing error boundaries at async boundaries
- Errors not propagated to callers
- Missing retry logic for transient failures
- `try?` suppressing important error information

```swift
// Sentry flags: Swallowed error in catch block
// BEFORE
do {
    try await processPayment(amount)
} catch {
    // TODO: handle error
}

// AFTER: Proper error handling
do {
    try await processPayment(amount)
} catch let error as PaymentError {
    logger.error("payment_failed", [
        "type": error.type.rawValue,
        "amount": amount.description
    ])
    throw error // Propagate to caller
} catch {
    logger.error("payment_unexpected_error", [
        "error": error.localizedDescription
    ])
    throw PaymentError.unexpected(underlying: error)
}
```

### Concurrency (Swift-specific)
- Missing `@MainActor` on UI-mutating code
- `Sendable` conformance violations
- Data races in shared mutable state
- Actor isolation boundary crossings
- Task cancellation not handled

```swift
// Sentry flags: Actor-isolated property modified from non-isolated context
// BEFORE
class ViewModel: ObservableObject {
    @Published var items: [Item] = []

    func loadItems() async {
        let fetched = try? await api.fetchItems()
        items = fetched ?? [] // Data race: modifying @Published from background
    }
}

// AFTER: Proper actor isolation
@MainActor
final class ViewModel: ObservableObject {
    @Published var items: [Item] = []

    func loadItems() async {
        let fetched = try? await api.fetchItems()
        // Now safe: entire class is MainActor-isolated
        items = fetched ?? []
    }
}
```

### Input Validation
- Permissive inputs accepted without sanitization
- Missing bounds checking on arrays/collections
- Unvalidated external data (API responses, user input)
- Deep link parameters used without sanitization

### Configuration
- Missing environment variables with no fallback
- Incorrect default paths or URLs
- Hardcoded values that should be configurable

---

## Automated Fix Verification

After implementing fixes, run these verification steps:

```bash
# 1. Build succeeds
xcodebuild build -scheme {SCHEME} -destination 'platform=iOS Simulator,name=iPhone 16'

# 2. Tests pass
xcodebuild test -scheme {SCHEME} -destination 'platform=iOS Simulator,name=iPhone 16'

# 3. No new warnings introduced
xcodebuild build -scheme {SCHEME} 2>&1 | grep -c "warning:"
```

For each fix, also verify manually:
- [ ] The fix compiles without errors
- [ ] Existing tests still pass
- [ ] The fix handles the specific case Sentry identified
- [ ] The fix doesn't change the public API contract (unless intentional)

---

## Integration with Security Review

When Sentry findings touch security-sensitive code, cross-reference with `trailofbits-differential-review` methodology:

- **Auth code findings**: Verify the fix doesn't weaken authentication
- **Crypto code findings**: Verify the fix doesn't introduce crypto weaknesses
- **Input validation findings**: Verify the fix applies at the trust boundary
- **Data handling findings**: Verify no PII leakage in error messages or logs

---

## Quality Gates (Before Marking Complete)

- [ ] All Sentry findings fetched and cataloged
- [ ] Each finding individually verified against current code (not dismissed in bulk)
- [ ] False positives documented with specific reasoning (not "I think it's fine")
- [ ] True positives fixed with root cause addressed (not just symptom)
- [ ] Fixes verified to address the specific Sentry concern
- [ ] No new warnings or errors introduced by fixes
- [ ] Summary report generated with status for every finding
- [ ] Escalations documented for findings requiring owner decision

---

## Self-Audit Protocol

Before declaring this skill's execution complete, verify ALL of the following:

- [ ] **Phase Gate: Code Verification** -- For every finding, did you read the ACTUAL current code at the flagged location? (Not Sentry's snippet -- the current file.)
- [ ] **No bulk dismissals** -- Did you evaluate each finding individually? If >50% dismissed as false positive, did you slow down and re-examine?
- [ ] **False positive reasoning documented** -- For every false positive, is there a specific, written justification? (Not "I think it's fine" -- a concrete reason like "Swift's type system prevents this because the variable is non-optional.")
- [ ] **True positive fixes verified** -- For every fix, did you re-read the Sentry finding and confirm your change addresses the specific concern?
- [ ] **Root cause, not symptom** -- For each fix, did you address WHY the issue exists, not just prevent the symptom?
- [ ] **Concurrency findings given extra scrutiny** -- Did you analyze (not dismiss) every actor isolation and Sendable finding?
- [ ] **Summary report produced** -- Is the findings table complete with status for every finding?
- [ ] **Escalations documented** -- For findings requiring owner decision, did you document the options and trade-offs?

**If any checkbox is unchecked, you are not done.** Go back and complete it.

---

## Cross-Skill References

- `sentry-find-bugs` -- For investigating production crashes and runtime errors from Sentry monitoring (not PR comments)
- `trailofbits-differential-review` -- For security-focused differential review of the same PR
- `cipher-security` -- When Sentry findings reveal security concerns requiring broader audit
- `prometheus-performance` -- When Sentry flags performance-related issues (N+1, main thread work)
- `ios-review` -- For general code review beyond Sentry-specific findings

---

## References

- False positive identification guide: `references/false-positive-guide.md`
- [Sentry Agent Skills](https://github.com/getsentry/agent-skills)
- [Sentry Documentation](https://docs.sentry.io/)
- [Sentry Bug Prediction](https://docs.sentry.io/product/issues/issue-details/autofix/)

---
name: sentry-find-bugs
description: "App is crashing, production error, users are reporting issues, investigate this error. Production issue analysis using Sentry — stack traces, breadcrumbs, root cause hypothesis, and fix verification. Trigger on: 'why is this crashing', 'Sentry issue', 'debug production bug', crash reports, issue IDs (like 'PROJECT-123'), 'users can't log in', 'something broke in production', 'exception in the logs', or triaging Sentry backlog. Requires Sentry MCP server. For Sentry bot PR comments, use sentry-code-review instead."
user-invocable: true
allowed-tools:
  - Read
  - Edit
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# Fix Sentry Issues

> **Iron Law**: "Never propose a fix without first verifying the root cause against actual Sentry event data. Guessing wastes everyone's time."

Discover, analyze, and fix production issues using Sentry's full debugging capabilities. Every fix must trace back to concrete evidence in the Sentry event -- stack frames, breadcrumbs, tags, or trace spans.

Source: [github.com/getsentry/agent-skills](https://github.com/getsentry/agent-skills)

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|-----------------|----------------|-----------------|
| "The crash only happened once" | One crash = one user who lost trust. Rare crashes often indicate race conditions that are hard to reproduce but affect many users silently. | Investigate fully. Check if the crash signature matches known patterns in `references/crash-patterns.md`. |
| "I can tell from the error message what's wrong" | Error messages describe symptoms, not causes. `unexpectedly found nil` tells you nothing about WHY it was nil. | Read the full stack trace, breadcrumbs, and context before hypothesizing. |
| "Let me just add a try-catch" | Catching an error without understanding the cause hides the bug. The crash moves downstream or manifests as data corruption instead. | Fix the root cause. Add error handling only after understanding what went wrong and why. |
| "This is a third-party crash, not our problem" | Your app called the third-party code. Your input, your timing, your configuration triggered it. | Trace backward from the crash to YOUR code that invoked the third-party library. Fix the invocation. |
| "I'll just add a nil check" | A nil check prevents the crash but doesn't fix why the value was nil. You're papering over a logic error. | Find where the value should have been set. Fix the logic that failed to set it. Then add the nil check as defense-in-depth. |
| "The user probably did something weird" | Users do what users do. If your app crashes from user behavior, the bug is in your app, not the user. | Reproduce the user's journey from breadcrumbs. Handle the input gracefully. |
| "We should wait for more data" | More crashes = more users affected. The data you have now is sufficient for investigation. | Investigate immediately. You can refine the fix if new data emerges. |

---

## Red Flags -- HARD STOP

These are not warnings. Each is a BLOCKING condition that prevents further progress until addressed.

- **BLOCKED: Proposing a fix without reading the full stack trace** -- every frame matters. The crash may originate 5 frames deep from where the exception was thrown. STOP. Read ALL frames in YOUR code before proposing anything.
- **BLOCKED: Treating symptoms instead of root cause** -- if you're adding nil checks or try-catch without understanding WHY the value was nil or WHY the operation threw, STOP and investigate deeper. Write down your hypothesis before writing code.
- **BLOCKED: Ignoring breadcrumb context** -- breadcrumbs reveal the user journey that led to the crash. STOP. Read breadcrumbs chronologically. Reconstruct the user journey.
- **BLOCKED: Not checking for related issues** -- the crash you're looking at may be a symptom of a broader problem. STOP. Search for related Sentry issues with similar stack traces or error types before fixing in isolation.
- **BLOCKED: Fixing code without verifying the fix addresses the Sentry event data** -- after implementing a fix, walk through the Sentry event step by step and confirm your fix would have prevented it. If you cannot trace the fix back to the event, STOP.
- **BLOCKED: Skipping regression check** -- verify your fix doesn't break other code paths. Check callers, check tests, check related functionality. Shipping a fix that breaks something else is not a fix.

---

## When NOT to Use This Skill

- **Sentry bot commented on a PR with static analysis findings** -- use `sentry-code-review` instead, which is designed for PR-level Sentry feedback.
- **Broad security audit not tied to a specific crash** -- use `cipher-security` for security posture assessment.
- **Performance issues without crashes** -- use `prometheus-performance` for profiling. However, if performance issues cause watchdog kills or OOM crashes that appear in Sentry, this skill applies.
- **No Sentry MCP server configured** -- this skill requires the Sentry MCP integration. Without it, use traditional debugging approaches.
- **The issue is a build error or compile-time problem** -- Sentry tracks runtime issues. Build problems need `forge-development`.

---

## Behavioral Enforcement

### Phase Gate: EVIDENCE (Before ANY Fix)
**Cannot propose ANY fix until:**
- [ ] Full stack trace read (not just the top frame -- ALL frames in YOUR code)
- [ ] Error message quoted verbatim
- [ ] Breadcrumbs reviewed (what happened BEFORE the crash)
- [ ] Related issues checked (is this a known pattern?)

**Proof**: Quote the exact error message and the top 3 stack frames from your code.

**Hard Stop**: If you're about to propose a fix and you haven't quoted the error message -- STOP. You're guessing. Read the stack trace first.

### Phase Gate: HYPOTHESIS (Before Coding)
**Cannot start coding a fix until:**
- [ ] Written hypothesis: "I believe [X] caused this because [Y], as evidenced by [Z]"
- [ ] Alternative hypothesis considered: "It could also be [A] because [B]"
- [ ] Hypothesis explains ALL the evidence (not just part of it)

**Why write it down**: A written hypothesis forces clarity. "Something is wrong with the auth" is not a hypothesis. "The refresh token expires during background fetch because the token rotation doesn't account for time zones, as shown by the breadcrumb 'tokenRefresh: UTC-5 mismatch'" IS a hypothesis.

### Fix Attempt Tracking (Counter-Based Escalation)
- **Fix #1**: Normal. Implement and test.
- **Fix #2**: Pause. Why didn't #1 work? Write down what you learned.
- **Fix #3**: STOP. You likely have the wrong root cause. Return to EVIDENCE phase. Re-read the stack trace. Consider: is this a symptom of a deeper architectural issue?
- **Still stuck after #3**: ESCALATE to human partner with: evidence collected, hypotheses tried, fixes attempted, what you learned.

---

## Decision Framework

```
User reports a production issue
|
+-- Has a Sentry issue ID?
|   +-- YES --> Phase 1: Fetch issue directly via sentry_get_issue
|   +-- NO --> Phase 1: Search recent unresolved issues
|
+-- Issue fetched, what type?
|   +-- EXC_BAD_ACCESS --> Memory issue (see references/crash-patterns.md: Memory section)
|   |   +-- Dangling pointer? Use-after-free? Buffer overflow?
|   |   +-- Check for force unwraps, unowned references, unsafe pointer usage
|   |
|   +-- EXC_BAD_INSTRUCTION --> Swift runtime trap
|   |   +-- Force unwrap nil? Failed precondition? Actor isolation violation?
|   |   +-- Check for !, preconditionFailure, fatalError
|   |
|   +-- Signal SIGKILL --> System killed the app
|   |   +-- Watchdog timeout? OOM? Background task expiry?
|   |   +-- Check memory usage, main thread work, background task duration
|   |
|   +-- Swift Error / NSException --> Logic error
|   |   +-- Index out of range? Invalid argument? Decoding failure?
|   |   +-- Trace data flow, check input validation
|   |
|   +-- Network Error --> Connectivity or server issue
|       +-- Timeout? SSL error? DNS failure?
|       +-- Check certificate pinning, server health, retry logic
|
+-- Root cause identified?
    +-- YES --> Phase 5: Implement fix with tests
    +-- NO --> Gather more context: traces, replays, related issues
```

---

## Invoke This Skill When

- User asks to "fix Sentry issues" or "resolve Sentry errors"
- User wants to "debug production bugs" or "investigate exceptions"
- User mentions issue IDs, error messages, or asks about recent failures
- User wants to triage or work through their Sentry backlog

## Prerequisites

- Sentry MCP server configured and connected
- Access to the Sentry project/organization

---

## Phase 1: Issue Discovery

Use Sentry MCP to find issues. Confirm with user which issue(s) to fix before proceeding.

| Search Type | MCP Call |
|-------------|----------|
| Recent unresolved | `sentry_search_issues` query: `"is:unresolved"` sort: `"date"` |
| Specific error type | `sentry_search_issues` query: `"is:unresolved error.type:TypeError"` |
| By ID | `sentry_get_issue` issue_id: `"PROJECT-123"` |
| High frequency | `sentry_search_issues` query: `"is:unresolved times_seen:>100"` sort: `"freq"` |
| Recent regression | `sentry_search_issues` query: `"is:unresolved is:regression"` sort: `"date"` |

---

## Phase 2: Deep Issue Analysis

Gather ALL available context for each issue:

| Data Source | MCP Call | Extract |
|-------------|----------|---------|
| **Core Error** | `sentry_get_issue` | Exception type/message, full stack trace, file paths, line numbers, function names |
| **Event Details** | `sentry_get_event` | Breadcrumbs, tags, custom context, request data |
| **Trace** (if available) | `sentry_get_trace` | Parent transaction, spans, DB queries, API calls, error location |
| **Replay** (if available) | `sentry_get_replay` | User actions, UI state, network requests |

### Reading the Stack Trace

See `references/stack-trace-guide.md` for the complete guide. Key principles:

1. **Read top-down**: The top frame is where the crash occurred. Walk DOWN to find YOUR code.
2. **Identify your frames**: Filter out system frameworks (`UIKitCore`, `SwiftUI`, `libdispatch`). Your code is where the bug is.
3. **Note the crash address**: `EXC_BAD_ACCESS` at address `0x0` means nil dereference. Non-zero invalid addresses suggest use-after-free or buffer overflow.
4. **Check async continuations**: In Swift concurrency, look for `swift_task_` frames and continuation frames. The actual call site may be in a different continuation.
5. **Demangle Swift names**: Sentry usually demangles automatically. If you see `$s...` prefixes, use `swift demangle` to decode them.

### Analyzing Breadcrumbs

Breadcrumbs reconstruct the user's journey before the crash. Read them chronologically:

```
BREADCRUMB ANALYSIS TEMPLATE:

1. List breadcrumbs in chronological order
2. For each breadcrumb, note:
   - Category (ui, navigation, http, console, user)
   - Message/data
   - Timestamp (time between breadcrumbs matters)
3. Identify the trigger:
   - What was the LAST user action before the crash?
   - What was the LAST network response before the crash?
   - Did any breadcrumb show an error or unexpected state?
4. Reconstruct the scenario:
   - "User navigated to X, tapped Y, received response Z, then crashed when..."
```

---

## Phase 3: Root Cause Hypothesis

Before touching code, document:

1. **Error Summary**: One sentence describing what went wrong
2. **Immediate Cause**: The direct code path that threw
3. **Root Cause Hypothesis**: Why the code reached this state
4. **Supporting Evidence**: Breadcrumbs, traces, or context supporting this
5. **Alternative Hypotheses**: What else could explain this? Why is yours more likely?

### Pre-Mortem Pattern Matching

Before deep investigation, check if the crash signature matches a known pattern. See `references/crash-patterns.md` for the full catalog. Common matches:

| Crash Signature | Usually Means | First Check |
|----------------|---------------|-------------|
| `unexpectedly found nil while unwrapping` | Force unwrap on nil optional | Find the `!` operator in the crashing line |
| `Index out of range` | Array access without bounds check | Check collection size vs index value |
| `Actor-isolated property accessed` | Swift concurrency violation | Check for missing `await` or wrong actor context |
| `EXC_BAD_ACCESS KERN_INVALID_ADDRESS` at `0x0` | Nil pointer dereference (usually Obj-C interop) | Check bridging, `@objc` methods, delegate patterns |
| `EXC_BAD_ACCESS` at non-zero address | Use-after-free or dangling pointer | Check `unowned` references, unsafe pointers |
| `Terminated due to memory pressure` | OOM kill | Check for memory leaks, large allocations, image handling |
| `scene watchdog timeout` | Main thread blocked > 10s | Check for synchronous network/IO/crypto on main thread |

Challenge yourself: Is this a symptom of a deeper issue? Check for similar errors elsewhere, related issues, or upstream failures in traces.

---

## Phase 4: Code Investigation

| Step | Actions |
|------|---------|
| **Locate Code** | Read every file in stack trace from top down |
| **Trace Data Flow** | Find value origins, transformations, assumptions, validations |
| **Error Boundaries** | Check for try/catch -- why didn't it handle this case? |
| **Related Code** | Find similar patterns, check tests, review recent commits (`git log`, `git blame`) |
| **Timing Analysis** | If the crash is intermittent, check for race conditions, async ordering issues |

### Swift-Specific Investigation Techniques

```swift
// 1. FORCE UNWRAP CRASH: Find the nil source
// Sentry shows: fatal error: Unexpectedly found nil in UserService.swift:42
// Don't just add a guard. Ask: WHY is it nil?

// Check the data flow backward:
// - Where is the value set? (initialization, assignment)
// - What conditions could prevent it from being set? (early return, error path, timing)
// - Is there a race condition? (value set on background thread, read on main)

// 2. ACTOR ISOLATION CRASH: Find the boundary crossing
// Sentry shows: Actor-isolated property 'balance' accessed from non-isolated context
// Check:
// - Is the caller running on the expected actor?
// - Is the property correctly isolated?
// - Should the call use `await`?

// 3. DECODE FAILURE: Find the mismatch
// Sentry shows: DecodingError.keyNotFound("status")
// Check:
// - What does the actual API response look like? (check breadcrumbs for HTTP response)
// - Is the API version correct?
// - Is the key optional in some responses but required in the model?
```

---

## Phase 5: Implement Fix

Before writing code, confirm your fix will:
- [ ] Handle the specific case that caused the error
- [ ] Not break existing functionality
- [ ] Handle edge cases (null, undefined, empty, malformed)
- [ ] Provide meaningful error messages
- [ ] Be consistent with codebase patterns

**Fix priority:** Prefer input validation > try/catch, graceful degradation > hard failures, specific > generic handling, root cause > symptom fixes.

### Regression Prevention

After implementing the fix, add these safeguards to prevent the same crash from returning:

1. **Add a test that reproduces the exact Sentry conditions**: Use the breadcrumbs and context to create a test case that would have caught this crash.
2. **Add defensive checks at the boundary**: Even after fixing the root cause, add a guard at the crash site as defense-in-depth.
3. **Add monitoring**: If the crash involved a complex data flow, add logging (non-PII) at key checkpoints so future issues are easier to diagnose.
4. **Check for similar patterns**: Search the codebase for the same anti-pattern. If you found one force unwrap bug, there may be ten more.

```swift
// EXAMPLE: Regression prevention for a force unwrap crash
// BAD: Just fixing the immediate crash
func processUser() {
    guard let user = currentUser else { return } // Crash prevented but silently fails
    // ...
}

// GOOD: Fix root cause + add defense-in-depth + add monitoring
func processUser() async throws {
    // Root cause fix: ensure user is loaded before this method is called
    guard let user = currentUser else {
        // Defense-in-depth: handle the "impossible" case gracefully
        logger.error("process_user_nil_user", [
            "screen": currentScreen,
            "auth_state": authState.description
        ])
        throw AppError.userNotLoaded
    }
    // ...
}
```

---

## Phase 6: Verification Audit

Complete before declaring fixed:

| Check | Questions |
|-------|-----------|
| **Evidence** | Does fix address exact error message? Handle data state shown? Prevent ALL events? |
| **Regression** | Could fix break existing functionality? Other code paths affected? Backward compatible? |
| **Completeness** | Similar patterns elsewhere? Related Sentry issues? Add monitoring/logging? |
| **Self-Challenge** | Root cause or symptom? Considered all event data? Will handle if occurs again? |
| **Test Coverage** | Test reproduces exact Sentry conditions? Edge cases covered? |

---

## Phase 7: Report Results

Format:
```
## Fixed: [ISSUE_ID] - [Error Type]
- Error: [message], Frequency: [X events, Y users], First/Last: [dates]
- Root Cause: [one paragraph]
- Evidence: Stack trace [key frames], breadcrumbs [actions], context [data]
- Fix: File(s) [paths], Change [description]
- Regression Prevention: [what safeguards were added]
- Verification: [ ] Exact condition [ ] Edge cases [ ] No regressions [ ] Tests [y/n]
- Follow-up: [additional issues, monitoring, related code]
```

---

## Common Bug Patterns

See `references/crash-patterns.md` for the complete catalog of 30+ iOS crash patterns with stack trace signatures, root causes, and fix approaches.

### iOS / Swift Quick Reference

| Pattern | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| Force unwrap crash | `unexpectedly found nil` | Optional not validated | Guard/if-let + fix why it was nil |
| Main thread violation | UI freeze, watchdog kill | Heavy work on main thread | Move to background queue / actor |
| Retain cycle | Memory growth, dealloc never called | Strong reference cycle | Weak/unowned references |
| Actor isolation | `Actor-isolated property accessed from non-isolated context` | Missing actor annotation | Add `@MainActor` or `await` |
| Sendable violation | Concurrency warning/crash | Non-sendable type crossing boundary | Conform to `Sendable` or use `@unchecked Sendable` |
| Index out of range | `Array index out of range` | Missing bounds check | Use safe subscript or guard |
| Keychain error | Auth state lost | Keychain access denied | Check entitlements, error handling |
| Decode failure | `DecodingError.keyNotFound` | API response mismatch | Make field optional or add CodingKeys |
| Task cancellation | `CancellationError` | View dismissed while task running | Check `Task.isCancelled`, use `.task` modifier |

### General Patterns

| Pattern | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| TypeError | `Cannot read property of null` | Missing null check | Validate data flow |
| Promise Rejection | Unhandled async error | Missing error boundary | Add try/catch at async boundary |
| Network Error | Request timeout, CORS | Server/config issue | Retry logic, proper headers |
| Rate Limit | 429 responses | Too many requests | Implement throttling/backoff |
| Memory/Performance | Slow spans, N+1 queries | Inefficient data access | Optimize queries, add caching |

---

## Quality Gates (Before Marking Complete)

- [ ] Full stack trace read and understood (not just the top frame)
- [ ] Breadcrumbs analyzed to reconstruct user journey
- [ ] Root cause identified with supporting evidence from Sentry event data
- [ ] Fix addresses root cause, not just symptoms
- [ ] Similar patterns searched for across codebase
- [ ] Related Sentry issues checked for common root cause
- [ ] Test added that reproduces the exact Sentry conditions
- [ ] Regression check: no existing functionality broken
- [ ] Report generated with evidence, fix description, and verification checklist

---

## Required Output Artifact

Every execution of this skill MUST produce a structured fix report:

```
## Fixed: [ISSUE_ID] - [Error Type]
**Error**: [message verbatim], Frequency: [X events, Y users], First/Last: [dates]

### Evidence Collected
- **Error message**: "[exact message quoted]"
- **Stack trace (your code frames)**:
  1. [file:line] [function]
  2. [file:line] [function]
  3. [file:line] [function]
- **Breadcrumbs**: [chronological user journey]
- **Related issues**: [related Sentry IDs or "none found"]

### Hypothesis
- **Primary**: "I believe [X] caused this because [Y], as evidenced by [Z]"
- **Alternative considered**: "[A] because [B] -- ruled out because [C]"

### Fix
- **File(s)**: [paths]
- **Change**: [description]
- **Fix attempt #**: [1/2/3]

### Regression Prevention
- [safeguards added]

### Verification
- [ ] Fix addresses exact Sentry event data
- [ ] Edge cases handled
- [ ] No regressions introduced
- [ ] Tests added [yes/no -- if no, why]
- [ ] Similar patterns searched across codebase

### Follow-up
- [additional issues, monitoring, related code]
```

**Incomplete reports are not accepted.** Every section must be filled.

---

## Self-Audit Protocol

Before declaring this skill's execution complete, verify ALL of the following:

- [ ] **Phase Gate: Evidence** -- Did you quote the exact error message? Can you list the top 3 stack frames in YOUR code?
- [ ] **Phase Gate: Hypothesis** -- Did you write a specific hypothesis with evidence? Did you consider an alternative?
- [ ] **Fix attempt tracking** -- Which attempt number is this? If #2+, did you document what you learned from previous attempts?
- [ ] **Hard Stops checked** -- Did you read the full stack trace? Did you review breadcrumbs? Did you check related issues?
- [ ] **Root cause, not symptom** -- Is your fix addressing WHY the value was nil / WHY the operation threw, not just preventing the crash?
- [ ] **Regression check** -- Did you verify the fix doesn't break callers, tests, or related functionality?
- [ ] **Output artifact produced** -- Is the fix report complete with all sections filled?

**If any checkbox is unchecked, you are not done.** Go back and complete it.

---

## Cross-Skill References

- `sentry-code-review` -- For handling Sentry bot comments on PRs (static analysis findings, not production crashes)
- `cipher-security` -- When a production crash reveals a security vulnerability
- `trailofbits-differential-review` -- For security review of the fix before merging
- `prometheus-performance` -- When the crash is caused by performance issues (OOM, watchdog timeout)
- `forge-development` -- For implementing the fix following project development patterns

---

## Quick Reference: MCP Tools

| Tool | Purpose |
|------|---------|
| `sentry_search_issues` | Find issues by query |
| `sentry_get_issue` | Get issue details by ID |
| `sentry_get_event` | Get specific event details |
| `sentry_get_trace` | Get distributed trace |
| `sentry_get_replay` | Get session replay |
| `sentry_list_projects` | List available projects |
| `sentry_get_project` | Get project details |

## References

- Stack trace reading guide: `references/stack-trace-guide.md`
- iOS crash pattern catalog: `references/crash-patterns.md`
- [Sentry Agent Skills](https://github.com/getsentry/agent-skills)
- [Sentry Documentation](https://docs.sentry.io/)
- [Sentry Issue Search Syntax](https://docs.sentry.io/product/sentry-basics/search/)
- [Sentry MCP Integration](https://docs.sentry.io/product/integrations/mcp/)

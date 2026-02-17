---
name: sentry-find-bugs
description: "This skill should be used when the user asks to 'fix Sentry errors', 'debug production issues', 'investigate exceptions', 'resolve Sentry bugs', 'triage Sentry backlog', or mentions Sentry issue IDs. Provides methodical analysis of stack traces, breadcrumbs, traces, and context to identify root causes and implement fixes."
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

Discover, analyze, and fix production issues using Sentry's full debugging capabilities.

Source: [github.com/getsentry/agent-skills](https://github.com/getsentry/agent-skills)

## Invoke This Skill When

- User asks to "fix Sentry issues" or "resolve Sentry errors"
- User wants to "debug production bugs" or "investigate exceptions"
- User mentions issue IDs, error messages, or asks about recent failures
- User wants to triage or work through their Sentry backlog

## Prerequisites

- Sentry MCP server configured and connected
- Access to the Sentry project/organization

## Phase 1: Issue Discovery

Use Sentry MCP to find issues. Confirm with user which issue(s) to fix before proceeding.

| Search Type | MCP Call |
|-------------|----------|
| Recent unresolved | `sentry_search_issues` query: `"is:unresolved"` sort: `"date"` |
| Specific error type | `sentry_search_issues` query: `"is:unresolved error.type:TypeError"` |
| By ID | `sentry_get_issue` issue_id: `"PROJECT-123"` |

## Phase 2: Deep Issue Analysis

Gather ALL available context for each issue:

| Data Source | MCP Call | Extract |
|-------------|----------|---------|
| **Core Error** | `sentry_get_issue` | Exception type/message, full stack trace, file paths, line numbers, function names |
| **Event Details** | `sentry_get_event` | Breadcrumbs, tags, custom context, request data |
| **Trace** (if available) | `sentry_get_trace` | Parent transaction, spans, DB queries, API calls, error location |
| **Replay** (if available) | `sentry_get_replay` | User actions, UI state, network requests |

## Phase 3: Root Cause Hypothesis

Before touching code, document:

1. **Error Summary**: One sentence describing what went wrong
2. **Immediate Cause**: The direct code path that threw
3. **Root Cause Hypothesis**: Why the code reached this state
4. **Supporting Evidence**: Breadcrumbs, traces, or context supporting this
5. **Alternative Hypotheses**: What else could explain this? Why is yours more likely?

Challenge yourself: Is this a symptom of a deeper issue? Check for similar errors elsewhere, related issues, or upstream failures in traces.

## Phase 4: Code Investigation

| Step | Actions |
|------|---------|
| **Locate Code** | Read every file in stack trace from top down |
| **Trace Data Flow** | Find value origins, transformations, assumptions, validations |
| **Error Boundaries** | Check for try/catch -- why didn't it handle this case? |
| **Related Code** | Find similar patterns, check tests, review recent commits (`git log`, `git blame`) |

## Phase 5: Implement Fix

Before writing code, confirm your fix will:
- [ ] Handle the specific case that caused the error
- [ ] Not break existing functionality
- [ ] Handle edge cases (null, undefined, empty, malformed)
- [ ] Provide meaningful error messages
- [ ] Be consistent with codebase patterns

**Fix priority:** Prefer input validation > try/catch, graceful degradation > hard failures, specific > generic handling, root cause > symptom fixes.

**Add tests** reproducing exact Sentry event conditions and verifying edge cases.

## Phase 6: Verification Audit

Complete before declaring fixed:

| Check | Questions |
|-------|-----------|
| **Evidence** | Does fix address exact error message? Handle data state shown? Prevent ALL events? |
| **Regression** | Could fix break existing functionality? Other code paths affected? Backward compatible? |
| **Completeness** | Similar patterns elsewhere? Related Sentry issues? Add monitoring/logging? |
| **Self-Challenge** | Root cause or symptom? Considered all event data? Will handle if occurs again? |

## Phase 7: Report Results

Format:
```
## Fixed: [ISSUE_ID] - [Error Type]
- Error: [message], Frequency: [X events, Y users], First/Last: [dates]
- Root Cause: [one paragraph]
- Evidence: Stack trace [key frames], breadcrumbs [actions], context [data]
- Fix: File(s) [paths], Change [description]
- Verification: [ ] Exact condition [ ] Edge cases [ ] No regressions [ ] Tests [y/n]
- Follow-up: [additional issues, monitoring, related code]
```

## Common Bug Patterns

### iOS / Swift Specific

| Pattern | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| Force unwrap crash | `unexpectedly found nil` | Optional not validated | Guard/if-let |
| Main thread violation | UI freeze, watchdog kill | Heavy work on main thread | Move to background queue |
| Retain cycle | Memory growth, dealloc never called | Strong reference cycle | Weak/unowned references |
| Actor isolation | `Actor-isolated property accessed from non-isolated context` | Missing actor annotation | Add `@MainActor` or `await` |
| Sendable violation | Concurrency warning/crash | Non-sendable type crossing boundary | Conform to `Sendable` or use `@unchecked Sendable` |
| Index out of range | `Array index out of range` | Missing bounds check | Use `safe` subscript or guard |
| Keychain error | Auth state lost | Keychain access denied | Check entitlements, error handling |

### General Patterns

| Pattern | Symptoms | Root Cause | Fix |
|---------|----------|------------|-----|
| TypeError | `Cannot read property of null` | Missing null check | Validate data flow |
| Promise Rejection | Unhandled async error | Missing error boundary | Add try/catch at async boundary |
| Network Error | Request timeout, CORS | Server/config issue | Retry logic, proper headers |
| Rate Limit | 429 responses | Too many requests | Implement throttling/backoff |
| Memory/Performance | Slow spans, N+1 queries | Inefficient data access | Optimize queries, add caching |

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

- [Sentry Agent Skills](https://github.com/getsentry/agent-skills)
- [Sentry Documentation](https://docs.sentry.io/)
- [Sentry Issue Search Syntax](https://docs.sentry.io/product/sentry-basics/search/)
- [Sentry MCP Integration](https://docs.sentry.io/product/integrations/mcp/)

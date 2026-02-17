---
name: sentry-code-review
description: "This skill should be used when the user asks to 'review Sentry comments', 'fix Sentry issues', 'address Sentry review', 'resolve Sentry findings', or mentions Sentry PR feedback. Provides systematic code review methodology for issues detected by Sentry's bug prediction, covering security, performance, and error handling."
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

Review and resolve code issues identified by Sentry's bug prediction bot in GitHub pull requests.

Source: [github.com/getsentry/agent-skills](https://github.com/getsentry/agent-skills)

## Invoke This Skill When

- User asks to "review Sentry comments" or "fix Sentry issues" on a PR
- User shares a PR URL/number mentioning Sentry feedback
- User asks to "address Sentry review" or "resolve Sentry findings"
- User wants to find PRs with unresolved Sentry comments

## Workflow

### Phase 1: Fetch Sentry Comments

Retrieve all comments from the Sentry bot on the target PR using GitHub's API:

```bash
# Fetch PR comments, filtering for sentry[bot]
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

### Phase 2: Parse Issue Details

Extract structured data from Sentry's markdown comments:

| Field | What to Extract |
|-------|----------------|
| **Bug Description** | Short summary of the identified issue |
| **Severity** | Critical / High / Medium / Low |
| **Confidence** | How certain Sentry is about the finding |
| **Analysis** | Detailed explanation of the potential bug |
| **Suggested Fix** | Sentry's recommended code change |
| **AI Prompt** | Context for understanding the issue |

### Phase 3: Verify and Implement Fixes

For each identified issue:

1. **Read the actual code** at the flagged file and line number
2. **Verify the issue exists** in the current code state (it may have been fixed already)
3. **Assess context** to determine if this is a true positive or false positive
4. **Implement the fix** considering:
   - Edge cases (null, empty, malformed inputs)
   - Error propagation (does the caller handle errors properly?)
   - Type safety (can types be narrowed or validated?)
   - Concurrency (race conditions, actor isolation in Swift)
   - Performance implications of the fix

### Phase 4: Report Results

Provide a summary table:

```markdown
| # | File | Issue | Severity | Status | Notes |
|---|------|-------|----------|--------|-------|
| 1 | TransferService.swift:42 | Missing nil check | High | FIXED | Added guard statement |
| 2 | PaymentHandler.swift:89 | Unhandled error | Medium | FIXED | Added do-catch |
| 3 | WalletView.swift:15 | Unused variable | Low | FALSE POSITIVE | Used in SwiftUI body |
| 4 | AuthService.swift:201 | Race condition | Critical | NEEDS REVIEW | Requires architecture decision |
```

## Common Issue Categories

### Type Safety
- Missing null/nil checks before force unwrap
- Unsafe type casting without validation
- Optional chaining gaps

### Error Handling
- Swallowed errors (empty catch blocks)
- Missing error boundaries at async boundaries
- Errors not propagated to callers
- Missing retry logic for transient failures

### Input Validation
- Permissive inputs accepted without sanitization
- Missing bounds checking on arrays/collections
- Unvalidated external data (API responses, user input)

### Configuration
- Missing environment variables with no fallback
- Incorrect default paths or URLs
- Hardcoded values that should be configurable

### Concurrency (Swift-specific)
- Missing `@MainActor` on UI-mutating code
- `Sendable` conformance violations
- Data races in shared mutable state
- Actor isolation boundary crossings

## Security Review Checklist

When reviewing code changes, also check for:

- [ ] No secrets or API keys in code
- [ ] Input validation on all external data
- [ ] SQL injection prevention (parameterized queries)
- [ ] Authentication checks on protected endpoints
- [ ] Authorization checks (user can only access their own data)
- [ ] Rate limiting on sensitive operations
- [ ] Audit logging for financial transactions
- [ ] Secure data handling (no PII in logs)

## Performance Review Checklist

- [ ] No N+1 query patterns
- [ ] Appropriate use of caching
- [ ] No unnecessary work on main thread (iOS)
- [ ] Memory management (no retain cycles, proper weak references)
- [ ] Efficient collection operations (avoid repeated iterations)
- [ ] Lazy loading for expensive resources

## References

- [Sentry Agent Skills](https://github.com/getsentry/agent-skills)
- [Sentry Documentation](https://docs.sentry.io/)
- [Sentry Bug Prediction](https://docs.sentry.io/product/issues/issue-details/autofix/)

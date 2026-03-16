# Documentation Examples — GOOD vs BAD

Side-by-side examples with annotations explaining why the good version is better.

---

## STATUS.md Updates

### BAD Example

```markdown
## Recent Changes

### 2026-03-15 Session
- Worked on transfers feature
- Made good progress
- Fixed some bugs
- Tests are looking good
- About 85% done with the feature
```

**Why it's bad**:
- "Worked on" is vague — what specifically changed?
- "Made good progress" is subjective — progress toward what?
- "Fixed some bugs" — which bugs? Are they in the backlog?
- "Tests are looking good" — how many pass? How many exist? What's the coverage?
- "About 85% done" — measured how? Is this verified or a gut feeling?

### GOOD Example

```markdown
## Recent Changes

### 2026-03-15 Session — Transfer Validation Implementation
- **Completed**: TransferService recipient validation with balance check (closes BACK-042)
- **Completed**: 6 unit tests for TransferService (all passing)
- **In Progress**: TransferViewModel loading states — wired to service, error handling remaining
- **Discovered**: Currency conversion rounding issue when converting between EUR/USD (filed as BACK-067, P2)
- **Metrics**: 47 tests passing, 0 failing, 0 skipped. Build time: 12s.
- **Blocker**: None currently active
```

**Why it's good**:
- Specific items with backlog references (BACK-042, BACK-067)
- Completion vs in-progress clearly distinguished
- New discovery documented with priority
- Metrics are concrete numbers, not feelings
- Blocker status explicit (even when there are none)

---

## BACKLOG.md Entries

### BAD Example

```markdown
### BACK-042: Transfer Feature
- Need to implement transfers
- Should handle errors
- Add tests
```

**Why it's bad**:
- "Need to implement transfers" — what part? The entire feature? One endpoint?
- "Should handle errors" — which errors? What's the expected behavior?
- "Add tests" — what tests? What coverage is acceptable?
- No acceptance criteria — how do you know when this is done?
- No priority, no size estimate, no assignee

### GOOD Example

```markdown
### BACK-042: Implement Recipient Validation for Transfers
**Priority**: P1 | **Size**: M (4-6 hours) | **Status**: In Progress (60%)

**Context**: Users can currently initiate transfers without validating the recipient exists or has an active account. This has caused 3 failed transfers in the last week (Sentry #4521, #4528, #4534).

**Acceptance Criteria**:
1. TransferService validates recipient exists before executing transfer
2. TransferService validates recipient account is active (not suspended/closed)
3. TransferService validates sender has sufficient balance
4. All three validations return specific error types (not generic errors)
5. ViewModel shows appropriate user-facing error messages for each case
6. Unit tests cover: valid transfer, nonexistent recipient, suspended recipient, insufficient balance

**Completed**:
- [x] TransferService.execute() with recipient validation
- [x] Domain errors: recipientNotFound, recipientSuspended, insufficientFunds
- [x] Unit tests for all 4 scenarios (6 tests, all passing)

**Remaining**:
- [ ] TransferViewModel error state handling
- [ ] Error message localization (3 strings)
- [ ] Integration test with mock API
```

**Why it's good**:
- Clear title that describes the specific work item
- Priority, size, and status with percentage
- Context explains WHY this work exists (links to real incidents)
- Acceptance criteria define DONE with specific, verifiable conditions
- Completed and remaining items show exactly where things stand
- Someone can pick this up mid-stream and know exactly what's left

---

## Architecture Decision Records (ADRs)

### BAD Example

```markdown
# ADR-005: Use Supabase

## Decision
We're using Supabase for the database.

## Consequences
- It's a good choice
- Easy to use
```

**Why it's bad**:
- No context — why was this decision needed?
- No alternatives considered — was anything else evaluated?
- "It's a good choice" is not a consequence — it's cheerleading
- No negative consequences listed (every decision has trade-offs)
- No status — is this proposed, accepted, or superseded?

### GOOD Example

```markdown
# ADR-005: Use Supabase for Backend Services

## Status
Accepted (2026-01-15)

## Context
The app needs a backend for authentication, database, and real-time subscriptions. The team has 1 iOS developer (no dedicated backend developer). Self-hosted solutions (raw PostgreSQL + custom auth) would require backend expertise we don't have. We need a managed solution that provides auth, database, and real-time out of the box.

## Decision
Use Supabase (managed PostgreSQL + GoTrue auth + Realtime) as the backend platform.

## Consequences

### Positive
- Zero backend code to maintain — Supabase handles auth, DB, and realtime
- PostgreSQL foundation provides full SQL capabilities and migration support
- Row Level Security (RLS) enforces authorization at the database level
- Swift SDK available with async/await support
- Free tier sufficient for development; Pro tier ($25/mo) sufficient for launch

### Negative
- Vendor lock-in — migrating away requires rewriting auth and realtime layers
- Supabase Swift SDK is less mature than Firebase — expect occasional gaps
- Real-time subscriptions require careful RLS policy design to avoid data leaks
- Cold start latency on free tier (2-3 second delay after inactivity)

## Alternatives Considered

1. **Firebase** — Rejected because: NoSQL (Firestore) doesn't support relational queries needed for transfer tracking. Real-time rules language is complex. Vendor lock-in is deeper.

2. **Self-hosted PostgreSQL + Vapor** — Rejected because: Requires backend development expertise the team doesn't have. Infrastructure management overhead is high for a 1-person team.

3. **AWS Amplify** — Rejected because: Complex configuration, steep learning curve, AppSync adds unnecessary GraphQL layer for our REST-shaped queries.
```

**Why it's good**:
- Context explains the situation and constraints driving the decision
- Decision is clear and specific
- Both positive AND negative consequences listed (honest assessment)
- Alternatives listed with specific, honest reasons for rejection
- Status and date recorded for historical tracking

---

## CHANGELOG Entries

### BAD Example

```markdown
## v1.2.0
- Updated stuff
- Fixed things
- Improved performance
```

**Why it's bad**:
- "Updated stuff" tells the user nothing
- "Fixed things" — which things?
- "Improved performance" — how much? What was slow?
- No categorization (Added/Changed/Fixed/Security)

### GOOD Example

```markdown
## v1.2.0 - 2026-03-15

### Added
- Scheduled transfers: Set up recurring transfers with daily, weekly, or monthly frequency
- Transfer history search: Find past transfers by recipient name, amount, or date

### Changed
- Transfer confirmation screen now shows estimated arrival time
- Recipient list sorted by most recently used (was alphabetical)

### Fixed
- Transfer amount rounding error when converting between EUR and USD (#4521)
- Back button on transfer detail screen occasionally navigated to wrong screen (#4498)

### Security
- Auth tokens now refresh 5 minutes before expiry (was refreshing after expiry, causing brief unauthorized state)
```

**Why it's good**:
- User-facing language (not implementation details)
- Categorized by type (Added/Changed/Fixed/Security)
- Specific descriptions that help users understand what changed
- Issue references for traceability
- Ordered by user impact (most important first within each category)

---

## Anti-Patterns Summary

| Anti-Pattern | Why It Fails | Fix |
|-------------|-------------|-----|
| Vague language ("worked on", "some bugs") | Reader gains no actionable information | Use specific items with IDs and metrics |
| Missing metrics ("tests look good") | Cannot verify progress or catch regressions | Include counts: "47 passing, 0 failing" |
| Aspirational status ("about 85% done") | Unverified estimates mislead planning | Verify: count completed vs total acceptance criteria |
| One-sided ADRs (only pros, no cons) | Hides risk from decision-makers | Every decision has trade-offs. List both sides. |
| Implementation details in CHANGELOG | Users don't understand "refactored DI container" | Describe user-facing impact: "App launches 2 seconds faster" |
| Duplicate information across documents | Which one is correct when they disagree? | One fact, one document. Cross-reference everything else. |

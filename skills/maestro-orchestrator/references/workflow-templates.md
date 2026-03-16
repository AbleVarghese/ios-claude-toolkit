# Workflow Templates — Detailed Step-by-Step

Each template provides: numbered steps, agent type, skill reference, dependencies, and expected outputs.

---

## Template 1: Feature Development

**Use when**: Implementing a new feature from requirements to committed code.
**Estimated agents**: 3-4
**Estimated time**: 2-4 hours

### Step 1: Plan & Design
- **Agent type**: Plan (read-only)
- **Skill reference**: forge-development (architecture patterns)
- **Dependencies**: None (first step)
- **Prompt essentials**: Analyze requirements, identify all files that need creation or modification, design the approach, list all tests that need to be written, identify any schema/migration needs
- **Expected output**: Implementation plan document with:
  - List of files to create/modify
  - Test list with naming convention (test_method_when_should)
  - Architecture diagram if new layer interactions are introduced
  - Schema changes if any
  - Risk assessment (what could go wrong)

### Step 2a: Write Tests (parallel)
- **Agent type**: general-purpose
- **Run in background**: true
- **Skill reference**: forge-development (TDD patterns, RED phase)
- **Dependencies**: Step 1 output (test list)
- **Prompt essentials**: Write all tests from the plan. Tests MUST fail. Use Arrange-Act-Assert pattern. Use factories for test data. Follow naming convention.
- **Expected output**: Test files with all tests written and confirmed failing

### Step 2b: Prepare Infrastructure (parallel)
- **Agent type**: general-purpose
- **Run in background**: true
- **Skill reference**: Project-specific database/API skill
- **Dependencies**: Step 1 output (schema changes)
- **Prompt essentials**: Create database migrations, update API types, add new protocol definitions in Domain layer
- **Expected output**: Migration files, updated protocol definitions
- **Note**: Only run this step if the plan identifies infrastructure changes needed

### Step 3: Implement
- **Agent type**: general-purpose
- **Run in background**: false (blocking — this is the critical path)
- **Skill reference**: forge-development (GREEN phase)
- **Dependencies**: Steps 2a AND 2b must complete
- **Prompt essentials**: Implement minimum code to make all tests from Step 2a pass. Follow architecture layer rules. Use protocol-based DI. Apply @MainActor to ViewModels.
- **Expected output**: Implementation files with all tests passing
- **Verification**: Run test suite, confirm 0 failures

### Step 4: Review
- **Agent type**: Explore (read-only)
- **Run in background**: false
- **Skill reference**: ios-review (full 5-phase protocol)
- **Dependencies**: Step 3
- **Prompt essentials**: Review all files changed in Steps 2-3. Run all 5 phases: triage, architecture audit, concurrency audit, security scan, report generation.
- **Expected output**: Review report with findings table
- **Decision point**: If BLOCKER findings exist → Step 5 must fix them before proceeding

### Step 5: Fix & Refactor
- **Agent type**: general-purpose
- **Run in background**: false
- **Skill reference**: forge-development (REFACTOR phase)
- **Dependencies**: Step 4 (review findings)
- **Prompt essentials**: Address all BLOCKER and CRITICAL findings from review. Refactor for cleanliness. Keep all tests green.
- **Expected output**: Clean code with 0 BLOCKER findings, all tests passing

### Step 6: Commit & Document
- **Agent type**: general-purpose
- **Run in background**: false
- **Skills**: heimdall-git (commit), documentation (STATUS.md, BACKLOG.md update)
- **Dependencies**: Step 5
- **Prompt essentials**: Stage all changed files. Commit with conventional message. Update STATUS.md with completed work. Update BACKLOG.md work item status.
- **Expected output**: Git commit hash, updated documentation

---

## Template 2: Bug Fix Pipeline

**Use when**: Fixing a reported bug with unknown root cause.
**Estimated agents**: 2-3
**Estimated time**: 1-2 hours

### Step 1: Investigate
- **Agent type**: Explore
- **Run in background**: false (need result before proceeding)
- **Skill reference**: None specific — general investigation
- **Dependencies**: Bug report (Sentry issue, user report, failing test)
- **Prompt essentials**: Find root cause. Check error logs, Sentry issues, database state. Trace the code path that triggers the bug. Identify the exact file and line where the fault originates.
- **Expected output**: Root cause analysis with:
  - File:line where the bug originates
  - What's happening (actual behavior)
  - What should happen (expected behavior)
  - Why it's happening (root cause)
  - Blast radius (what else might be affected)

### Step 2: Reproduce
- **Agent type**: general-purpose
- **Run in background**: false
- **Skill reference**: forge-development (TDD — RED phase)
- **Dependencies**: Step 1 (root cause)
- **Prompt essentials**: Write a test that demonstrates the bug. The test MUST FAIL to confirm the bug exists. Name: `test_method_whenBugCondition_shouldExpectedBehavior`.
- **Expected output**: Failing test that proves the bug exists
- **Critical**: If you can't write a failing test, the bug is not well-understood. Go back to Step 1.

### Step 3: Fix
- **Agent type**: general-purpose
- **Run in background**: false
- **Skill reference**: forge-development (GREEN phase)
- **Dependencies**: Step 2 (failing test)
- **Prompt essentials**: Implement the minimum fix to make the test pass. Do not refactor unrelated code. Do not fix "nearby" issues.
- **Expected output**: Fix applied, failing test now passes, all other tests still pass

### Step 4: Verify
- **Agent type**: Explore
- **Run in background**: false
- **Skill reference**: ios-review (focused review of changed files)
- **Dependencies**: Step 3
- **Prompt essentials**: Review the fix for: side effects, regression risk, concurrency safety, security implications. Run full test suite.
- **Expected output**: Verification report: fix is safe to merge or list of concerns

### Step 5: Commit & Document
- **Agent type**: general-purpose
- **Run in background**: false
- **Skills**: heimdall-git, documentation
- **Dependencies**: Step 4 (verification passed)
- **Prompt essentials**: Commit with `fix(scope): description`. Reference the bug ID if available. Update STATUS.md. Update BACKLOG.md if the bug was a tracked item.
- **Expected output**: Git commit, updated docs

---

## Template 3: Code Review Workflow

**Use when**: Reviewing a large changeset that benefits from parallel review tracks.
**Estimated agents**: 2-3
**Estimated time**: 30-60 minutes

### Step 1a: Architecture + Concurrency Review (parallel)
- **Agent type**: Explore
- **Run in background**: true
- **Skill reference**: ios-review Phases 2-3
- **Prompt essentials**: Run Phase 2 (architecture audit: layer violations, DI compliance, dependency direction) and Phase 3 (concurrency audit: @MainActor, Sendable, actor isolation) on all changed files.
- **Expected output**: Findings table with [File:Line | Issue | Severity | Fix]

### Step 1b: Security + Quality Review (parallel)
- **Agent type**: Explore
- **Run in background**: true
- **Skill reference**: ios-review Phase 4
- **Prompt essentials**: Run Phase 4 (security scan: force unwraps, secrets, PII, keychain; quality checks: error handling, view complexity, tests) on all changed files.
- **Expected output**: Findings table with [File:Line | Issue | Severity | Fix]

### Step 2: Compile Report
- **Agent type**: Plan
- **Run in background**: false
- **Skill reference**: ios-review Phase 5 (report generation)
- **Dependencies**: Steps 1a AND 1b
- **Prompt essentials**: Merge findings from both review agents. Deduplicate. Sort by severity. Generate summary table. Determine merge verdict.
- **Expected output**: Complete review report (use review-report-template.md format)

---

## Template 4: Emergency Response (P0)

**Use when**: Production is broken, users are affected, immediate action needed.
**Estimated agents**: 1-2 (sequential only)
**Estimated time**: 30-90 minutes

**Emergency rules**:
- NO parallel agents — clarity over speed
- Every step verified before proceeding
- Owner notified at Step 1
- Document everything as you go

### Step 1: Triage (you, not an agent)
- Capture: error message, affected users, time started, blast radius
- Classify: data loss? Security breach? Feature broken? Performance degraded?
- Notify: owner with incident summary
- **Output**: Incident summary

### Step 2: Contain
- **Agent type**: general-purpose
- **Run in background**: false
- **Prompt essentials**: Apply immediate containment. Options: toggle feature flag OFF, revert last deployment, disable affected endpoint, add rate limiting.
- **Expected output**: Containment applied. Bleeding stopped.
- **Verify**: Confirm users are no longer experiencing the issue (or impact is reduced)

### Step 3: Root Cause
- **Agent type**: Explore
- **Run in background**: false
- **Prompt essentials**: Investigate. Check error logs, Sentry, database state, recent deployments. Find the exact cause.
- **Expected output**: Root cause with file:line reference

### Step 4: Fix
- **Agent type**: general-purpose
- **Run in background**: false
- **Skill reference**: forge-development (focused TDD)
- **Prompt essentials**: Write a test that reproduces the bug. Fix it. Verify all other tests pass.
- **Expected output**: Fix + test, all green

### Step 5: Verify & Deploy
- **Agent type**: Explore
- **Run in background**: false
- **Prompt essentials**: Review fix for safety. Run full test suite. Verify containment can be removed.
- **Expected output**: Verification passed. Safe to deploy.

### Step 6: Postmortem
- **Agent type**: Plan
- **Run in background**: false
- **Skill reference**: documentation
- **Prompt essentials**: Document: what happened, when, who was affected, root cause, how it was fixed, how to prevent recurrence. Create action items.
- **Expected output**: Postmortem document

---

## Template 5: Migration Workflow

**Use when**: Changing database schema, API contracts, or data format in a way that requires careful coordination.
**Estimated agents**: 2-3
**Estimated time**: 2-4 hours

### Step 1: Plan
- **Agent type**: Plan
- **Prompt essentials**: Analyze migration scope. What tables/columns change? What queries are affected? What services need updating? Design rollback strategy.
- **Expected output**: Migration plan with rollback procedure

### Step 2: Backup Verification
- **Agent type**: general-purpose
- **Dependencies**: Step 1
- **Prompt essentials**: Verify recent backup exists. If not, create one. Document backup location and timestamp.
- **Expected output**: Backup confirmed with location and timestamp

### Step 3: Implement Migration
- **Agent type**: general-purpose
- **Dependencies**: Step 2
- **Prompt essentials**: Write migration files. Update affected repositories, services, and DTOs. Follow project's migration conventions.
- **Expected output**: Migration files, updated service code

### Step 4a: Test Migration (parallel)
- **Agent type**: general-purpose
- **Run in background**: true
- **Dependencies**: Step 3
- **Prompt essentials**: Apply migration to test database. Verify data integrity — all existing data preserved, new schema correct, queries return expected results.
- **Expected output**: Test report — all data integrity checks pass

### Step 4b: Test Rollback (parallel)
- **Agent type**: general-purpose
- **Run in background**: true
- **Dependencies**: Step 3
- **Prompt essentials**: Apply migration, then apply rollback. Verify database returns to pre-migration state. No data loss.
- **Expected output**: Rollback test report — state fully restored

### Step 5: Apply
- **Agent type**: general-purpose
- **Dependencies**: Steps 4a AND 4b passing
- **Prompt essentials**: Apply migration to target environment. Monitor for errors.
- **Expected output**: Migration applied successfully

### Step 6: Verify
- **Agent type**: Explore
- **Dependencies**: Step 5
- **Prompt essentials**: Verify migration success. Check data integrity, run application tests, verify no broken queries or endpoints.
- **Expected output**: Verification report — migration successful or list of issues

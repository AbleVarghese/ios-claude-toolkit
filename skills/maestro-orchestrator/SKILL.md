---
name: maestro-orchestrator
description: "Do these in parallel, split this into tasks, coordinate multiple things, too big for one agent. Multi-agent workflow coordination for complex development tasks — parallel execution, SDLC templates, dependency management, and escalation protocols. Trigger when: orchestrating multi-step workflows, running parallel reviews, managing emergency P0 response, structured bug fix pipelines, or the task involves 3+ files across different concerns. Also for 'handle all of this', 'do everything at once'. For single-skill work, invoke that skill directly instead."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: true
---

# MAESTRO — Workflow Orchestration & Multi-Agent Coordination

Coordinate multi-step development workflows, manage parallel agent execution, handle dependency resolution between tasks, and run escalation protocols for critical issues.

> **Iron Law**: "Independent tasks run in parallel. Dependent tasks run sequentially. Never guess — verify the dependency."

---

## Task Dependency Analysis

Before assigning any work to agents, you must analyze task dependencies. Getting this wrong means agents either block on each other (wasted time) or produce conflicting changes (wasted work).

### Dependency Decision Tree

```
START: You have tasks A and B.

Q1: Does Task B need the OUTPUT of Task A?
├── YES → Sequential. B runs AFTER A completes.
│   Examples:
│   - "Write tests" then "implement code" (tests define the interface)
│   - "Investigate bug" then "fix bug" (need root cause first)
│   - "Design schema" then "write migration" (migration needs the schema)
│
└── NO →
    Q2: Does Task B modify the SAME FILES as Task A?
    ├── YES → Sequential OR use worktree isolation.
    │   Q3: Can you use separate worktrees?
    │   ├── YES → Parallel with worktree isolation.
    │   └── NO → Sequential. Same-file parallel edits cause merge conflicts.
    │
    └── NO →
        Q3: Does Task B depend on the same EXTERNAL STATE as Task A?
        (same database table, same API endpoint, same configuration)
        ├── YES → Careful. May need sequential if state changes affect both.
        └── NO → PARALLEL. These tasks are independent. Run them simultaneously.
```

### Phase Gate: DEPENDENCY ANALYSIS
**Cannot spawn agents until:**
- [ ] All tasks are listed with clear deliverables
- [ ] Dependency graph is determined: which tasks need output from other tasks?
- [ ] Parallel vs sequential is decided based on actual dependencies (not assumption)
- [ ] No two agents will edit the same file

**Hard Stop**: If two agents would edit the same file — RESTRUCTURE. Overlapping file edits cause merge conflicts and lost work. Split the work so each file has exactly one owner.

### Common Dependency Patterns

| Task A | Task B | Dependency? | Run |
|--------|--------|------------|-----|
| Write tests | Implement feature | YES — tests define the interface | Sequential: A → B |
| Review security | Review performance | NO — different concerns, read-only | Parallel |
| Update schema | Write API handler | YES — handler needs new schema | Sequential: A → B |
| Write service tests | Write view tests | NO — different files, different concerns | Parallel |
| Fix auth bug | Update docs | YES — docs should reflect the fix | Sequential: A → B |
| Investigate bug | Review unrelated PR | NO — completely independent | Parallel |

---

## Agent Tool Syntax & Invocation

### Agent Types

| `subagent_type` | Capabilities | When to Use |
|-----------------|-------------|-------------|
| `general-purpose` | Read, Write, Edit, Bash, Grep, Glob | Implementation work, bug fixes, refactoring |
| `Explore` | Read, Grep, Glob (read-only) | Investigation, code analysis, review |
| `Plan` | Read, Grep, Glob (read-only) | Planning, architecture analysis, strategy |

### Invocation Pattern

```
Agent tool call parameters:
  - prompt: Clear, specific instruction with all context the agent needs
  - subagent_type: "general-purpose" | "Explore" | "Plan"
  - run_in_background: true (for genuinely independent tasks) | false (when you need the result)
```

### Phase Gate: AGENT LAUNCH
**Each agent MUST have before spawning:**
- [ ] Specific task description (not vague — exact deliverable)
- [ ] Output path (where to save results)
- [ ] Success criteria (how to know it worked)
- [ ] Failure handling (what to do if it fails)

**Counter**: If an agent fails, understand WHY before respawning. At 2 failures for the same task — the task definition is wrong. Restructure.

### Writing Effective Agent Prompts

Every agent prompt must include:

1. **What to do**: Specific, concrete task description
2. **Where to look**: File paths, directories, modules
3. **What to produce**: Expected output format
4. **Constraints**: What NOT to do, boundaries, skill references

```
Example prompt (good):
"Review all Swift files in Sources/Services/ for Swift 6 strict concurrency violations.
Check for: missing @MainActor on ViewModels, @Published mutations outside MainActor,
missing Sendable conformance on types passed between actors.
Use the ios-review skill's Phase 3 (Concurrency Audit) checklist.
Output: table with columns [File:Line, Issue, Severity, Fix].
Do NOT edit any files — this is a review, not a fix."

Example prompt (bad):
"Check the code for issues."
→ Too vague. What code? What issues? What output format?
```

### Background vs Foreground Decision

```
Q1: Do you need the result of this agent BEFORE the next step?
├── YES → Foreground (run_in_background: false)
│   You'll wait for the result before proceeding.
│   Examples: investigation before fix, plan before implementation
│
└── NO →
    Q2: Is this agent genuinely independent of other work?
    ├── YES → Background (run_in_background: true)
    │   You'll be notified when it completes.
    │   Examples: parallel reviews, independent feature work
    │
    └── NO → Foreground. If there's any dependency, wait for it.
```

---

## Workflow Templates

### Template 1: Feature Development

```
WORKFLOW: New Feature Implementation
ESTIMATED AGENTS: 3-4
ESTIMATED TIME: 2-4 hours

Step 1: PLAN (foreground, Plan agent)
  Task: Analyze requirements, identify affected files, design approach
  Skill: forge-development (architecture patterns)
  Output: Implementation plan with file list and test list
  Dependencies: None (first step)

Step 2: PARALLEL PHASE
  Step 2a: WRITE TESTS (background, general-purpose agent)
    Task: Write failing tests for all acceptance criteria
    Skill: forge-development (TDD patterns)
    Input: Test list from Step 1
    Output: Test files with all tests failing (RED phase)
    Dependencies: Step 1 output

  Step 2b: PREPARE INFRASTRUCTURE (background, general-purpose agent)
    Task: Create/update database migrations, API contracts
    Skill: Project-specific infrastructure skill
    Input: Schema changes from Step 1
    Output: Migration files, updated API types
    Dependencies: Step 1 output
    Note: Only if schema/API changes are needed

Step 3: IMPLEMENT (foreground, general-purpose agent)
  Task: Write implementation code to make tests pass
  Skill: forge-development (GREEN phase)
  Input: Failing tests from 2a, infrastructure from 2b
  Output: Implementation code with all tests passing
  Dependencies: Steps 2a AND 2b must complete first

Step 4: REVIEW (foreground, Explore agent)
  Task: Review implementation for quality, security, concurrency
  Skill: ios-review (5-phase protocol)
  Input: All changed files from Steps 2-3
  Output: Review report with findings
  Dependencies: Step 3

Step 5: FIX & REFACTOR (foreground, general-purpose agent)
  Task: Address review findings, refactor for cleanliness
  Skill: forge-development (REFACTOR phase)
  Input: Review findings from Step 4
  Output: Clean, reviewed code with all tests passing
  Dependencies: Step 4

Step 6: COMMIT & DOCUMENT (foreground, general-purpose agent)
  Task: Commit changes, update project documentation
  Skills: heimdall-git, documentation
  Dependencies: Step 5
```

### Template 2: Bug Fix Pipeline

```
WORKFLOW: Bug Fix
ESTIMATED AGENTS: 2-3
ESTIMATED TIME: 1-2 hours

Step 1: INVESTIGATE (foreground, Explore agent)
  Task: Find root cause by analyzing error reports, logs, and code
  Output: Root cause analysis with file:line reference
  Dependencies: None

Step 2: REPRODUCE (foreground, general-purpose agent)
  Task: Write a test that reproduces the bug (must FAIL)
  Skill: forge-development (RED phase — the test proves the bug exists)
  Input: Root cause from Step 1
  Output: Failing test
  Dependencies: Step 1

Step 3: FIX (foreground, general-purpose agent)
  Task: Implement minimum fix to make the test pass
  Skill: forge-development (GREEN phase)
  Input: Failing test from Step 2
  Output: Passing test + fix
  Dependencies: Step 2

Step 4: VERIFY (foreground, Explore agent)
  Task: Review fix for side effects, run full test suite
  Skill: ios-review (focused on the changed files)
  Output: Verification report
  Dependencies: Step 3

Step 5: COMMIT & DOCUMENT (foreground, general-purpose agent)
  Task: Commit with "fix(scope): description", update STATUS.md
  Skills: heimdall-git, documentation
  Dependencies: Step 4
```

### Template 3: Code Review Workflow

```
WORKFLOW: Comprehensive Code Review
ESTIMATED AGENTS: 2-3 (parallel)
ESTIMATED TIME: 30-60 minutes

Step 1: PARALLEL REVIEW PHASE
  Step 1a: ARCHITECTURE + CONCURRENCY REVIEW (background, Explore agent)
    Task: ios-review Phases 2-3 (architecture audit, concurrency audit)
    Output: Findings table for architecture and concurrency

  Step 1b: SECURITY + QUALITY REVIEW (background, Explore agent)
    Task: ios-review Phase 4 (security scan, quality checks)
    Output: Findings table for security and quality

Step 2: COMPILE REPORT (foreground, Plan agent)
  Task: Merge findings from 1a and 1b into unified report
  Skill: ios-review Phase 5 (report generation)
  Input: Findings from Steps 1a and 1b
  Output: Complete review report with merge verdict
  Dependencies: Steps 1a AND 1b
```

### Template 4: Emergency Response (P0)

```
WORKFLOW: P0 Emergency
ESTIMATED AGENTS: 1-2
ESTIMATED TIME: 30-90 minutes

⚠️ EMERGENCY RULES:
  - No parallel agents. Sequential only. Clarity over speed.
  - Every step verified before proceeding.
  - Owner notified at Step 1.

Step 1: TRIAGE (immediate, foreground)
  Task: Capture current state — error details, affected users, blast radius
  Output: Incident summary with severity assessment
  Action: Notify owner

Step 2: CONTAIN (foreground, general-purpose agent)
  Task: Apply immediate containment (feature flag, disable endpoint, revert)
  Output: Containment applied, bleeding stopped
  Dependencies: Step 1

Step 3: ROOT CAUSE (foreground, Explore agent)
  Task: Investigate root cause — logs, Sentry, code analysis
  Output: Root cause with file:line reference
  Dependencies: Step 2

Step 4: FIX (foreground, general-purpose agent)
  Task: Implement targeted fix with test
  Skill: forge-development (focused TDD)
  Output: Fix + test, all tests passing
  Dependencies: Step 3

Step 5: VERIFY (foreground, Explore agent)
  Task: Review fix, run full test suite, verify containment can be removed
  Output: Verification report
  Dependencies: Step 4

Step 6: POSTMORTEM (foreground, Plan agent)
  Task: Document what happened, why, how it was fixed, how to prevent recurrence
  Skill: documentation
  Output: Postmortem document
  Dependencies: Step 5
```

### Template 5: Migration Workflow

```
WORKFLOW: Data/Schema Migration
ESTIMATED AGENTS: 2-3
ESTIMATED TIME: 2-4 hours

Step 1: PLAN (foreground, Plan agent)
  Task: Analyze migration scope, identify affected tables/queries/services
  Output: Migration plan with rollback strategy

Step 2: BACKUP VERIFICATION (foreground, general-purpose agent)
  Task: Verify backup exists and is recent, or create one
  Dependencies: Step 1

Step 3: IMPLEMENT MIGRATION (foreground, general-purpose agent)
  Task: Write migration files, update affected services
  Dependencies: Step 2

Step 4: PARALLEL VERIFICATION
  Step 4a: TEST MIGRATION (background, general-purpose agent)
    Task: Run migration against test database, verify data integrity
  Step 4b: TEST ROLLBACK (background, general-purpose agent)
    Task: Test rollback procedure, verify it restores previous state

Step 5: APPLY (foreground, general-purpose agent)
  Task: Apply migration to target environment
  Dependencies: Steps 4a AND 4b passing

Step 6: VERIFY (foreground, Explore agent)
  Task: Verify migration success — data integrity, no broken queries
  Dependencies: Step 5
```

---

## Agent Failure Handling

### What Happens When an Agent Fails

Agents can fail for several reasons:
- Task too vague (agent doesn't know what to do)
- File conflicts (agent tried to edit a file another agent is editing)
- Runtime error (build failure, test failure)
- Timeout (task took too long)

### Failure Response Protocol

```
AGENT FAILURE DETECTED

Q1: What type of failure?
├── VAGUE TASK → Rewrite the prompt with more specific instructions.
│   Include: exact file paths, expected output format, constraints.
│
├── FILE CONFLICT → This means two agents edited the same file.
│   Q2: Can the changes be merged?
│   ├── YES → Merge manually, continue.
│   └── NO → Roll back one agent's changes. Re-run sequentially.
│
├── BUILD/TEST FAILURE → The agent's code doesn't work.
│   Q3: Is the failure in the agent's changes or pre-existing?
│   ├── AGENT'S CHANGES → Re-run with corrective instructions.
│   └── PRE-EXISTING → Fix the pre-existing issue first, then retry.
│
└── TIMEOUT → Task was too large for one agent.
    Split into smaller sub-tasks. Re-assign.
```

### Escalation to Human

Escalate to the owner when:
- Two retry attempts fail for the same task
- Agent produces output that contradicts project architecture
- Security-sensitive decision needed (credentials, access, deployment)
- Ambiguous requirement that could go multiple ways

---

## Workload Estimation

### When to Use Agents

| Situation | Agents Needed | Reasoning |
|-----------|--------------|-----------|
| Single file change | 0 (do it yourself) | Agent overhead isn't worth it |
| 2-3 file changes, one concern | 0-1 | Manageable without coordination |
| Multi-file feature, single concern | 1-2 | One implements, one reviews |
| Multi-file feature, multiple concerns | 2-4 | Separate concerns, parallel work |
| Cross-cutting change (e.g., rename across codebase) | 1 with worktree | Single agent, clean isolation |
| Emergency P0 | 1-2 (sequential) | Clarity matters more than speed |

### Complexity Thresholds

```
SIMPLE (0 agents — do it yourself):
  - Single file edit
  - Documentation update
  - Simple bug fix with known cause
  - Configuration change

MODERATE (1-2 agents):
  - Feature with 3-5 files changed
  - Bug fix requiring investigation
  - Code review of 5-10 files

COMPLEX (2-4 agents):
  - Feature spanning multiple modules
  - Migration affecting multiple services
  - Comprehensive review of large PR
  - Parallel test + implementation work

OVER-ENGINEERED (too many agents):
  - Don't spawn 5+ agents. Coordination overhead exceeds benefit.
  - If you need 5+ parallel tasks, batch them into 2-3 agent groups.
```

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|---------------|-----------------|
| "I'll do it all sequentially, it's simpler" | Sequential execution of independent tasks wastes time. If two agents can review security and performance simultaneously, running them in sequence doubles the wait. | Analyze dependencies. If tasks are independent, run them in parallel. |
| "Parallel agents are too complex to coordinate" | Coordination is only complex when dependencies are unclear. The dependency decision tree makes it mechanical. | Use the dependency decision tree. It removes ambiguity. |
| "I can keep track in my head" | You can't. When 3 agents return results, you need a structured plan to merge their outputs. | Use workflow templates. They define the merge points. |
| "One agent can handle all of this" | One agent doing a 6-step sequential workflow takes 6x longer than 3 agents doing parallelizable work. | Split into independent tasks. Parallelize what you can. |
| "Let me just spawn a bunch of agents" | More agents means more coordination overhead, more potential conflicts, more failure points. | Match agent count to task complexity. Simple tasks need zero agents. |

---

## Red Flags — HARD STOP

These are BLOCKING conditions, not suggestions. If ANY red flag is triggered, you CANNOT continue orchestration until the violation is remediated. Proceeding past a red flag invalidates all subsequent agent work.

- **Spawning agents that edit the same files**: This will cause merge conflicts. Either use worktree isolation or run them sequentially.
- **No dependency verification before parallelizing**: You must verify independence before running tasks in parallel. Use the dependency decision tree.
- **No error handling for agent failures**: Every agent invocation must have a plan for what happens if it fails. "Retry" is not a plan — "retry with more specific instructions" is.
- **Spawning too many agents for a simple task**: If the task is simple enough to do yourself in 5 minutes, don't coordinate agents. The overhead costs more than the work.
- **Agent prompts without clear deliverables**: Every agent must know what output to produce. "Look at the code" is not a deliverable. "Produce a review report with findings table" is.

---

## When NOT to Use This Skill

- **Single-file changes or simple edits** → Do the work directly, no orchestration needed
- **Writing tests or implementation code** → Use `forge-development` directly
- **Code review of a small changeset** → Use `ios-review` directly
- **Git operations** → Use `heimdall-git` directly
- **Documentation updates** → Use `documentation` directly

**Rule of thumb**: If the work can be done in one skill invocation, call that skill directly. Use Maestro only when you need to coordinate MULTIPLE skill invocations across MULTIPLE steps.

---

## Decision Framework: Orchestration Selection

```
START: You have a complex task.

Q1: Can this be done in a single step with one skill?
├── YES → Use that skill directly. No orchestration needed.
└── NO →
    Q2: How many independent concerns are involved?
    ├── 1 concern → Sequential steps, one agent at a time.
    ├── 2-3 concerns → Parallel agents where independent, sequential where dependent.
    └── 4+ concerns → Group into 2-3 batches. Parallelize within batches.

    Q3: Select workflow template:
    ├── New feature → Template 1: Feature Development
    ├── Bug fix → Template 2: Bug Fix Pipeline
    ├── Code review → Template 3: Code Review Workflow
    ├── Emergency → Template 4: Emergency Response
    ├── Migration → Template 5: Migration Workflow
    └── Custom → Build from dependency analysis
```

---

## Quality Gates (Before Marking Complete)

- [ ] All task dependencies verified before agent assignment
- [ ] No agents editing the same files without worktree isolation
- [ ] Every agent prompt includes: what, where, output format, constraints
- [ ] Every agent has a failure handling plan
- [ ] All agent outputs collected and verified
- [ ] Final deliverable assembled from agent outputs
- [ ] No orphaned agents (all background agents completed or handled)
- [ ] Documentation updated to reflect completed work

---

## Self-Audit (Before Declaring Orchestration Complete)

Answer each question. If ANY answer is "no" or "unsure" — go back and fix it.

1. Did I verify task dependencies before parallelizing? (Cite the dependency analysis)
2. Did any two agents edit the same file? (If yes, were merge conflicts resolved?)
3. Did every agent have a specific deliverable and success criteria?
4. Did any agent fail? If so, did I understand why before retrying? (How many retry attempts?)
5. Are all background agents completed? (No orphaned agents)
6. Is the final deliverable assembled from all agent outputs? (Nothing missing?)

---

## Required Output Artifact

Every orchestration using this skill must produce:
1. **Task dependency graph** — showing which tasks depend on which
2. **Agent assignment table** — task, agent type, status, output
3. **Execution log** — order of execution, parallel vs sequential decisions
4. **Final deliverable** — assembled from all agent outputs
5. **Failure log** — any failures, root causes, and resolutions (if applicable)

An orchestration without a dependency graph is guesswork. An orchestration with orphaned agents is incomplete.

---

## Cross-Skill Handoff Protocol

When one skill produces output that another skill needs as input, use this structured format:

### Handoff Format
```markdown
## Handoff: [source-skill] → [target-skill]

### Context
- Task: [what was done]
- Findings: [count] items at [severity levels]

### Structured Findings
| # | File:Line | Finding | Severity | Action Needed |
|---|-----------|---------|----------|--------------|
| 1 | ... | ... | ... | ... |

### Recommended Next Steps
1. [specific action for target skill]
2. ...
```

### Common Handoff Chains
| Chain | When | Handoff Content |
|-------|------|----------------|
| ios-review → forge-development | Review finds issues to fix | Findings table with file:line:fix |
| cipher-security → trailofbits | Broad audit finds PR-specific concerns | Threat model + specific files to review |
| sentry-find-bugs → forge-development | Root cause identified, fix needed | Hypothesis + evidence + affected files |
| forge-development → ios-review | Implementation complete, needs review | Files changed + architecture decisions |
| forge-development → heimdall-git | Code ready to commit | Files to stage + commit scope |

### Handoff Gate
Before handing off, verify:
- [ ] Findings are in structured format (not just narrative)
- [ ] Each finding has enough context for the receiving skill to act
- [ ] The receiving skill is identified correctly (check "When NOT to Use" sections)

---

## Cross-Skill References

| Skill | When to Use |
|-------|-------------|
| `forge-development` | For implementation agents — writing tests, code, refactoring |
| `ios-review` | For review agents — architecture, concurrency, security audits |
| `heimdall-git` | For commit/push agents — version control operations |
| `documentation` | For documentation agents — STATUS.md, BACKLOG.md, CHANGELOG |
| `prometheus-performance` | For performance analysis agents |
| `cipher-security` | For security-focused review agents |

---

## Escalation Matrix

| Severity | Response Time | Action |
|----------|---------------|--------|
| P0 Critical | Immediate | Stop all other work. Use Emergency Response template. Notify owner. |
| P1 High | Within current session | Prioritize next. Use Bug Fix template. |
| P2 Medium | Next available slot | Add to BACKLOG.md. Schedule in upcoming session. |
| P3 Low | When convenient | Document only. Batch with other P3 items. |

---

## References

- **Agent Patterns**: `references/agent-patterns.md` — complete agent invocation guide, isolation modes, error handling
- **Workflow Templates**: `references/workflow-templates.md` — detailed templates with step-by-step agent assignments

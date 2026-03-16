# Agent Patterns — Complete Guide

## Agent Tool Invocation

### Parameters Reference

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `prompt` | string | Yes | The instruction for the agent. Must be specific and complete. |
| `subagent_type` | string | No | `"general-purpose"` (default), `"Explore"`, or `"Plan"` |
| `run_in_background` | boolean | No | `true` for non-blocking execution, `false` (default) for blocking |

### subagent_type Selection Guide

#### `general-purpose` — Full Capability Agent

**Capabilities**: Read, Write, Edit, Bash, Grep, Glob (all tools)

**When to use**:
- Implementing code (writing new files, editing existing ones)
- Running tests or build commands
- Fixing bugs (requires editing code)
- Applying refactoring changes
- Running git operations
- Any task that requires modifying files

**When NOT to use**:
- Pure investigation (use Explore instead — lower risk of accidental edits)
- Planning and analysis (use Plan instead)

#### `Explore` — Read-Only Investigation Agent

**Capabilities**: Read, Grep, Glob (read-only tools)

**When to use**:
- Code review (must not edit files during review)
- Bug investigation (finding root cause without fixing)
- Architecture analysis (mapping dependencies, finding violations)
- Security audit (scanning for vulnerabilities)
- Understanding unfamiliar code (exploration before implementation)

**When NOT to use**:
- Tasks that require changing files (use general-purpose)
- Tasks that require running commands (use general-purpose)

**Why Explore over general-purpose for reviews**: Explore agents literally cannot edit files. This prevents scope creep where a "review" accidentally becomes a "refactor." The skill's Read-only constraint enforces the ios-review Iron Law: reviews produce reports, not changes.

#### `Plan` — Analysis and Strategy Agent

**Capabilities**: Read, Grep, Glob (read-only tools)

**When to use**:
- Breaking down a complex feature into implementation steps
- Analyzing trade-offs between architectural approaches
- Designing test strategies before writing tests
- Estimating effort for a set of tasks
- Creating implementation plans with file lists and dependency graphs

**When NOT to use**:
- Tasks that require executing the plan (use general-purpose)
- Tasks that need to verify something by running it (use general-purpose)

---

## Background vs Foreground Execution

### Foreground (run_in_background: false or omitted)

The orchestrator waits for the agent to complete before proceeding.

**Use when**:
- The next step depends on this agent's output
- You need to inspect the result before continuing
- The task is on the critical path
- Sequential dependency exists

### Background (run_in_background: true)

The agent runs independently. You receive a notification when it completes.

**Use when**:
- The agent's work is genuinely independent of other work
- You have other tasks to start while this one runs
- The result isn't needed immediately
- Multiple independent reviews can run simultaneously

**Important**: You do NOT need to poll or sleep-wait for background agents. The system notifies you when they complete.

---

## Isolation Modes

### Default Isolation (No Worktree)

All agents work in the same working directory. This is fine when:
- Agents read different files (no conflict possible)
- Only one agent writes at a time (sequential writes)
- Agents write to completely different files (no overlap)

### Worktree Isolation

When two agents need to edit files in the same directory tree, use git worktrees to give each agent its own copy of the codebase.

**When to use worktrees**:
- Two agents need to edit files that might overlap
- Parallel implementation of features that touch shared infrastructure
- Testing two different approaches to the same problem

**How it works**:
1. Create a worktree: `git worktree add ../worktree-name branch-name`
2. Agent A works in the main directory
3. Agent B works in `../worktree-name`
4. Merge results when both complete

**When NOT to use worktrees**:
- One agent is read-only (Explore/Plan) — no conflict possible
- Agents edit completely non-overlapping files
- Simple sequential workflows

---

## Error Handling Patterns

### Agent Failure Notifications

When a background agent fails, you receive a notification with:
- The error message
- The agent's partial output (if any)
- The agent's state at time of failure

### Retry Strategy

```
ATTEMPT 1: Original prompt
  Failed? →

ATTEMPT 2: More specific prompt
  - Add explicit file paths
  - Clarify expected output format
  - Add constraints that were ambiguous
  - Reference specific skill sections
  Failed? →

ATTEMPT 3: Simplified scope
  - Split the task into smaller parts
  - Remove the part that's causing failure
  - Handle that part manually
  Failed? →

ESCALATE: Report to owner
  - Include all three failure messages
  - Include your analysis of why it's failing
  - Propose manual workaround
```

### Handling Conflicting Outputs

When two agents produce contradictory results:

```
Q1: Are they reviewing the same code with different findings?
├── YES → Both findings are likely valid. Include both in the report.
└── NO →
    Q2: Are they proposing different implementations?
    ├── YES → Evaluate both against the project's architecture.
    │         Use the one that fits. Document why the other was rejected.
    └── NO →
        Q3: Is one agent's output based on stale information?
        ├── YES → Use the more current output. Re-run the stale agent if needed.
        └── NO → Escalate to owner. Present both outputs with your analysis.
```

---

## Anti-Patterns

### Anti-Pattern 1: Too Many Agents

```
❌ BAD: Spawning 6 agents for a simple bug fix
  - Agent 1: Read the error log
  - Agent 2: Find the file
  - Agent 3: Read the file
  - Agent 4: Understand the bug
  - Agent 5: Write the fix
  - Agent 6: Test the fix

✅ GOOD: 1-2 agents for a bug fix
  - Agent 1 (Explore): Investigate root cause, output file:line and analysis
  - Agent 2 (general-purpose): Write test + fix (TDD), output passing test
```

**Rule**: If each agent's task takes less than 2 minutes, the coordination overhead exceeds the work. Do it yourself.

### Anti-Pattern 2: Agents Without Clear Deliverables

```
❌ BAD: "Look at the authentication code and see if there are any issues."
  - What's an "issue"? How should it be reported? What severity levels?

✅ GOOD: "Review Sources/Services/AuthService.swift and Sources/ViewModels/AuthViewModel.swift
  for Swift 6 concurrency violations. Check for: missing @MainActor, Sendable violations,
  @Published mutations outside MainActor. Output a table with columns:
  [File:Line | Issue | Severity | Fix]. Use severity levels: BLOCKER, CRITICAL, WARNING."
```

### Anti-Pattern 3: Agents With Overlapping Scope

```
❌ BAD:
  Agent 1: "Review TransferService.swift for quality issues"
  Agent 2: "Review TransferService.swift for bugs"
  → Both agents will find the same things. Wasted work.

✅ GOOD:
  Agent 1: "Review TransferService.swift for concurrency issues (Phase 3 of ios-review)"
  Agent 2: "Review TransferService.swift for security issues (Phase 4 of ios-review)"
  → Different checklists, different findings, no overlap.
```

### Anti-Pattern 4: Fire and Forget

```
❌ BAD: Spawn 3 background agents, immediately report "done"
  → You haven't verified any agent completed, let alone produced correct output.

✅ GOOD: Spawn background agents → Wait for notifications → Collect outputs →
  Verify each output → Merge into final deliverable → Report with evidence.
```

---

## Prompt Engineering for Agents

### Template for Agent Prompts

```
[TASK]: What the agent should do (1-2 sentences, specific)

[SCOPE]: Which files/directories to work with (explicit paths)

[SKILL REFERENCE]: Which skill's protocol to follow (e.g., "ios-review Phase 3")

[OUTPUT FORMAT]: What the output should look like (table, report, code, etc.)

[CONSTRAINTS]:
  - What NOT to do (e.g., "Do NOT edit files")
  - Boundaries (e.g., "Only review files in Sources/Services/")
  - References (e.g., "Use the review-checklist.md for severity levels")
```

### Example: Feature Implementation Agent

```
Implement the TransferService.execute() method to pass the failing tests in
Tests/Services/TransferServiceTests.swift.

Files to modify: Sources/Services/TransferService.swift
Files to read (for context): Sources/Domain/Protocols/TransferServiceProtocol.swift,
  Sources/Domain/Entities/Transfer.swift, Sources/Domain/Errors/TransferError.swift

Follow the forge-development skill's GREEN phase: write minimum code to make tests pass.
Ensure @MainActor is NOT on the service (it's not a ViewModel).
Ensure Sendable conformance on all types crossing concurrency boundaries.

Output: The completed TransferService.swift file with all tests passing.

Constraints:
  - Do NOT modify test files
  - Do NOT add features beyond what tests require
  - Do NOT use force unwraps or completion handlers
```

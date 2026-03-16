---
name: documentation
description: "Session ending, what changed today, update project status, record this decision. SSOT document management for STATUS.md, BACKLOG.md, CHANGELOG, and ADR maintenance. Trigger at session end, after code changes, when wrapping up work, or when the user asks 'what did we do', 'log this', 'update the docs', 'note this decision', or wants to capture progress. Also handles cross-document consistency validation and documentation freshness checks. For git commit/push operations, use heimdall-git instead."
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
user-invocable: false
---

# Documentation & SSOT Management

Automate session-end SSOT document updates, maintain cross-document consistency, generate changelog entries and ADRs, and enforce documentation freshness. Templates live in `references/templates.md`; enforcement protocol in `references/enforcement.md`; examples in `references/examples.md`.

> **Iron Law**: "Documentation reflects reality — never aspirational state. If you haven't verified it, don't write it."

> **Project Discovery**: Before executing, determine the project's SSOT documents, their locations, and update conventions from CLAUDE.md or project documentation. Common SSOT documents include STATUS.md, BACKLOG.md, CHANGELOG.md, ARCHITECTURE.md, and API.md — but each project may use different names or structures.

---

## Document Hierarchy

```
TIER 1 - Immutable Foundation (owner approval required):
    Project foundation document (e.g., PROJECT.md)
    Architecture decision records (once accepted)

TIER 2 - Live State (updated every session):
    Status tracking, backlog, architecture docs, API docs

TIER 3 - AI Instructions:
    CLAUDE.md, .claude/skills/*/SKILL.md

TIER 4 - Historical:
    Archives, completed decision records, old changelogs
```

**Why tiers matter**: Tier 1 documents require owner approval to change because they define project direction. Tier 2 documents change every session and must always reflect current reality. Confusing the tiers — treating a Tier 2 doc as Tier 1 (never updating it) or treating a Tier 1 doc as Tier 2 (changing it without approval) — creates drift and confusion.

---

## Single Source of Truth Rules

> **Note**: Identify the project's specific SSOT document mapping from CLAUDE.md. Common pattern:

| Document | Is SSOT For | Update Frequency |
|----------|-------------|-----------------|
| STATUS.md (or equivalent) | Current project state, metrics, blockers | Every session |
| BACKLOG.md (or equivalent) | All work items, priorities, assignments | Every session |
| ARCHITECTURE.md | System design, layer rules, tech decisions | When architecture changes |
| API.md | Service contracts, endpoint specs | When APIs change |
| CHANGELOG.md | Version history, user-facing changes | Every release |

**Rules**:
- No duplicates. One document per concern.
- Cross-reference instead of copy. If STATUS.md mentions a backlog item, link to BACKLOG.md — don't reproduce the description.
- If a fact exists in two places and they disagree, the SSOT document for that concern wins.

---

## Documentation Quality Decision Tree

```
START: You need to update documentation.

Q1: What kind of update is this?
├── NEW FACT (something that didn't exist before)
│   Q2: Does a document already cover this concern?
│   ├── YES → Add the fact to the existing document. Do NOT create a new doc.
│   └── NO → Is this a recurring concern that needs tracking?
│       ├── YES → Create a new SSOT document. Register it in the hierarchy.
│       └── NO → Add to the most relevant existing document under "Notes" section.
│
├── UPDATE (changing existing information)
│   Q3: Has the underlying reality actually changed?
│   ├── YES → Update the document to reflect the new reality.
│   │   Verify the new value by checking code/tests/builds — don't trust memory.
│   └── NO → Do not update. The documentation is correct as-is.
│       If you THINK it should change but reality hasn't, you may have a bug.
│
└── CORRECTION (fixing wrong documentation)
    Q4: Which is correct — the code or the documentation?
    ├── CODE is correct → Fix the documentation to match reality.
    └── DOCUMENTATION is correct → The code has a bug. File it in BACKLOG.md.
        Do NOT change the documentation to match broken code.
```

---

## Session-End Protocol (MANDATORY)

This protocol runs at the end of every session. It is not optional.

### Step 1: Read Current State

```
1. Parse status document current metrics (completion %, test counts, etc.)
2. Parse backlog document current work items and their statuses
3. Identify all work completed this session (from git log and memory)
4. Collect any new issues or blockers discovered
```

### Phase Gate: VERIFICATION (Before Writing Any Metric)
**Cannot write a percentage, count, or status metric until:**
- [ ] The actual value has been checked (run test suite, count files, check build)
- [ ] The value matches reality, not aspiration

**Hard Stop**: If you're about to write "85% complete" without having verified that number — STOP. Run the check. Report the real number even if it's lower than expected. Aspirational documentation is worse than no documentation — it creates false confidence.

### Step 2: Verify Before Writing

**This is where most documentation failures happen.** Before writing a single update:

```
For EACH metric you're about to update:
  - Is this number verified? (Did you run the test suite? Did you count the files?)
  - Or is this number estimated? (Are you guessing based on feel?)

VERIFIED → Write it.
ESTIMATED → Run the verification command. Then write the verified number.
UNKNOWN → Write "Unknown — needs verification" with a note about what to check.

NEVER write a number you haven't verified. "~85% complete" is honest.
"87.3% complete" when you haven't counted is dishonest and misleading.
```

### Step 3: Calculate Updates

```
- Determine feature completion changes (what moved from In Progress to Done?)
- Calculate new test counts (run test suite, count passing/failing/skipped)
- Identify resolved blockers (what was blocking before that's unblocked now?)
- Identify new blockers (what's blocking progress right now?)
- Note any new discoveries (bugs found, tech debt identified, risks surfaced)
```

### Step 4: Update Documents

Update in this order:
1. **Status document**: timestamp, changes summary, metrics, blockers
2. **Backlog document**: work item status changes, completed items, new items
3. **Other docs as needed** based on what changed (architecture, API, etc.)

### Step 5: Validate Consistency

```
Cross-check:
[ ] Status doc "Last Updated" is today's date
[ ] Status doc "Recent Changes" includes current session's work
[ ] Backlog work items match status doc progress metrics
[ ] Feature percentages are consistent across ALL documents
[ ] Test counts match actual test suite output
[ ] Blocker status is accurate and current
[ ] No duplicate information exists across documents
[ ] All cross-references are valid and current
```

---

### Phase Gate: STALENESS CHECK
**Cannot update any document without first:**
- [ ] Reading the current version (not from memory — actually read the file)
- [ ] Checking the "Last Updated" timestamp
- [ ] Identifying what changed since the last update

**If copying status from a previous session without checking**: STOP. The codebase may have changed. Read, verify, then write.

---

## Staleness Detection Protocol

Documentation goes stale silently. Actively check for staleness:

```
For EACH SSOT document:
1. Check "Last Updated" timestamp
   - >24 hours for STATUS.md? → STALE. Update immediately.
   - >1 week for ARCHITECTURE.md? → Check if any code changes affected architecture.
   - >1 sprint for BACKLOG.md? → STALE. Work items likely have wrong status.

2. Verify cross-references
   - Does STATUS.md reference a feature that was completed or cancelled?
   - Does BACKLOG.md list items that no longer exist?
   - Does ARCHITECTURE.md describe a pattern that was replaced?

3. Compare documented state to actual code
   - Run test suite — do test counts match documented counts?
   - Check git log — are recent changes reflected in CHANGELOG.md?
   - Review file structure — does ARCHITECTURE.md match actual directory layout?
```

---

## Automation Triggers

| Event | Target Documents | Update Type |
|-------|------------------|------------|
| Session End | Status, Backlog | Mandatory — run full protocol |
| Code Change | Status, Backlog | Update affected work items |
| Architecture Change | Architecture doc, Status | Document the change and rationale (ADR) |
| API Change | API doc, CHANGELOG | Update contracts, add changelog entry |
| Bug Fix | Status, Backlog, CHANGELOG | Mark resolved, add changelog entry |
| Test Addition | Status | Update test metrics |
| Dependency Update | CHANGELOG | Add entry with version change |
| New Blocker Discovered | Status, Backlog | Record immediately, don't wait for session end |

---

## Honesty Standard (NON-NEGOTIABLE)

- Never exaggerate completion percentages
- Report actual status, not desired status
- Include all blockers regardless of severity
- Use verified metrics (test counts, build status, actual measurements)
- Set realistic timelines based on actual progress velocity
- Document setbacks honestly — they're information, not failures
- If unsure about a metric, write "approximately" or "estimated"

**Why honesty matters**: Documentation drives decisions. Inflated metrics lead to premature launches. Hidden blockers lead to surprises. The owner needs reality to make good decisions.

---

## Freshness Requirements

| Document Type | Maximum Staleness | Action When Stale |
|--------------|-------------------|-------------------|
| Status document | 24 hours | Immediate update |
| Backlog document | Every session | Update work items |
| Technical docs | When content changes | Verify and update |
| CHANGELOG | Every release | Add version entry |
| Stale docs (no consumers) | N/A | Move to Archive/ |

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|---------------|-----------------|
| "I'll update docs later" | "Later" is next session. By then you've forgotten details, and the docs are stale for anyone who reads them between now and then. | Update immediately. 5 minutes now, or 30 minutes reconstructing later. |
| "The code is self-documenting" | Code explains WHAT and HOW. Documentation explains WHY, WHEN, and WHAT'S THE STATUS. They serve different audiences and purposes. | Update the docs. Code and docs are complementary, not alternatives. |
| "No one reads STATUS.md" | You read it at the start of every session to understand where things stand. Other agents read it. The owner reads it. | If no one read it, they wouldn't exist. Update them. |
| "Documentation slows me down" | Documentation that prevents one wrong decision saves more time than it costs. Debugging an issue that was already documented wastes everyone's time. | Budget 5-10 minutes per session for docs. It's part of the work, not separate from it. |
| "The PR description has enough detail" | PR descriptions are ephemeral — buried in a list of merged PRs. SSOT documents are persistent and findable. | PR descriptions supplement docs, not replace them. |
| "Nothing changed" | If nothing changed, that's worth documenting: "No progress on X due to Y blocker." Absence of updates creates ambiguity. | Document the lack of change and the reason. |

---

## Red Flags — HARD STOP

These are BLOCKING conditions, not suggestions. If ANY red flag is triggered, you CANNOT continue the current documentation task until the violation is remediated. Proceeding past a red flag invalidates all subsequent documentation updates.

- **Documenting percentages you haven't verified**: Run the actual measurement. "I think it's about 80%" is not documentation — it's a guess.
- **Copy-pasting status from a previous session without checking**: The status may have changed. Read the current state before writing.
- **Skipping documentation because "nothing changed"**: Document the lack of change. Silence is ambiguous.
- **Inflating completion metrics to look good**: Documentation serves truth, not morale. If progress is slow, say so — and explain why.
- **Creating a new document when one already exists for this concern**: You're creating a duplicate source of truth. Find the existing doc and update it.

---

## When NOT to Use This Skill

- **Writing or reviewing Swift code** → Use `forge-development` or `ios-review`
- **Git operations (commit, push, branch)** → Use `heimdall-git`
- **Coordinating multi-step workflows** → Use `maestro-orchestrator`
- **Documentation about Xcode project setup or CI/CD** → Use the relevant infrastructure skill

---

## Decision Framework: What to Document

```
START: Something happened in the codebase.

Q1: Did code change?
├── YES →
│   Q2: Does this change affect project status or metrics?
│   ├── YES → Update STATUS.md with new state.
│   └── NO → No status update needed.
│
│   Q3: Does this change affect a work item?
│   ├── YES → Update BACKLOG.md (mark completed, add new items, update progress).
│   └── NO → No backlog update needed.
│
│   Q4: Is this change user-facing?
│   ├── YES → Add CHANGELOG entry.
│   └── NO → No changelog update needed.
│
│   Q5: Does this change affect architecture?
│   ├── YES → Update ARCHITECTURE.md. Create ADR if this is a decision.
│   └── NO → No architecture update needed.
│
└── NO →
    Q6: Was a decision made?
    ├── YES → Create ADR. Update relevant docs.
    └── NO →
        Q7: Was a blocker discovered or resolved?
        ├── YES → Update STATUS.md immediately.
        └── NO → No documentation update needed this moment.
            BUT: Session-end protocol still runs.
```

---

## Quality Gates (Before Marking Complete)

- [ ] All SSOT documents checked for staleness
- [ ] Status document "Last Updated" is today's date
- [ ] All metrics verified (not estimated) — test counts, build status, completion %
- [ ] Recent changes section includes current session's work
- [ ] Backlog items reflect actual current state
- [ ] No duplicate information across documents
- [ ] Cross-references validated and current
- [ ] Blocker list is accurate and complete
- [ ] Honesty standard maintained — no inflated metrics

---

## Self-Audit (Before Declaring Documentation Complete)

Answer each question. If ANY answer is "no" or "unsure" — go back and fix it.

1. Did I read the current version of every document before updating it? (Not from memory — actually read the file)
2. Is every metric I wrote verified by an actual check? (Test suite run, file count, build output)
3. Did I check for staleness before writing updates?
4. Are there any aspirational statements disguised as facts? (e.g., "feature is 90% complete" without verification)
5. Did I cross-check consistency between STATUS, BACKLOG, and other SSOT documents?
6. Would the owner make correct decisions based on what I wrote? (If the documentation would mislead, fix it)

---

## Required Output Artifact

Every documentation update using this skill must produce:
1. **Updated documents** — with accurate "Last Updated" timestamps
2. **Verification evidence** — how each metric was verified (command run, file checked, etc.)
3. **Consistency check** — confirmation that cross-document references agree
4. **Change summary** — what was updated and why

A documentation update without verification evidence is incomplete. A metric without a verification method is a guess, not documentation.

---

## Cross-Skill References

| Skill | When to Use |
|-------|-------------|
| `forge-development` | When code changes trigger documentation updates |
| `heimdall-git` | When committing documentation changes |
| `ios-review` | When review findings need to be documented |
| `maestro-orchestrator` | When documentation updates are part of a larger workflow |

---

## Bypass Conditions (RARE)

Documentation updates may only be skipped if:
- Session involved pure research (no code/config changes)
- Emergency bug fix with immediate deploy requirement (document in NEXT session)
- Owner explicitly requests bypass with documented reason

In all bypass cases: add note to next session explaining the gap.

---

## References

- **Templates**: `references/templates.md` — STATUS update, BACKLOG update, ADR, CHANGELOG, API doc templates
- **Enforcement**: `references/enforcement.md` — mandatory enforcement protocol, validation commands, error handling, recovery procedures
- **Examples**: `references/examples.md` — side-by-side GOOD vs BAD documentation examples

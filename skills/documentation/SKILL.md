---
name: documentation
description: "This skill should be used when the user asks to 'update STATUS.md', 'update BACKLOG.md', 'session end documentation', 'SSOT maintenance', 'changelog entry', 'architecture decision record', or mentions documentation freshness. Provides automated SSOT document updates, cross-document consistency validation, and session-end enforcement."
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
user-invocable: false
---

# Documentation & SSOT Management

Automate session-end SSOT document updates, maintain cross-document consistency, generate changelog entries and ADRs, and enforce documentation freshness. Templates live in `references/templates.md`; enforcement protocol in `references/enforcement.md`.

> **Project Discovery**: Before executing, determine the project's SSOT documents, their locations, and update conventions from CLAUDE.md or project documentation. Common SSOT documents include STATUS.md, BACKLOG.md, CHANGELOG.md, ARCHITECTURE.md, and API.md — but each project may use different names or structures.

## Document Hierarchy

```
TIER 1 - Immutable Foundation (owner approval required):
    Project foundation document (e.g., PROJECT.md)

TIER 2 - Live State (updated every session):
    Status tracking, backlog, architecture docs, API docs

TIER 3 - AI Instructions:
    CLAUDE.md, .claude/skills/*/SKILL.md

TIER 4 - Historical:
    Archives, decision records
```

## Single Source of Truth Rules

> **Note**: Identify the project's specific SSOT document mapping from CLAUDE.md. Common pattern:

| Document | Is SSOT For |
|----------|-------------|
| STATUS.md (or equivalent) | Current project state |
| BACKLOG.md (or equivalent) | All work items |
| ARCHITECTURE.md | System design |
| API.md | Service contracts |
| CHANGELOG.md | Version history |

No duplicates. One document per concern. Cross-reference instead of copy.

## Session-End Protocol (MANDATORY)

### Step 1: Read Current State
- Parse status document current metrics
- Parse backlog document current work items
- Identify session work completed
- Collect any new issues discovered

### Step 2: Calculate Updates
- Determine feature completion changes
- Calculate new test counts
- Identify resolved/new blockers

### Step 3: Update Documents
- Status document: timestamp, changes, metrics, blockers
- Backlog document: work item status, completed items, stats
- Other docs as needed based on changes made

### Step 4: Validate Consistency
- Cross-check all updated values across documents
- Verify no duplicate information introduced
- Confirm SSOT principles maintained

## Automation Triggers

| Event | Target Documents |
|-------|------------------|
| Session End | Status, Backlog |
| Code Change | Status, Backlog |
| Architecture Change | Architecture doc, API doc |
| API Change | API doc |
| Bug Fix | Status, Backlog |
| Test Addition | Status |

## Honesty Standard (NON-NEGOTIABLE)

- Never exaggerate completion percentages
- Report actual status, not desired status
- Include all blockers regardless of severity
- Use accurate metrics (test counts, warnings)
- Set realistic timelines based on actual progress

## Freshness Requirements

- **Status document**: Never >24 hours stale
- **Backlog document**: Updated every session
- **Technical docs**: Updated when content changes
- **Stale docs**: Move to Archive/

## Cross-Document Validation Checklist

```
[ ] Status doc "Last Updated" is today's date
[ ] Status doc "Recent Changes" includes current session
[ ] Backlog work items match status doc progress
[ ] Feature percentages consistent across documents
[ ] Test counts match actual implementation
[ ] Blocker status accurate and up-to-date
[ ] No duplicate information exists
[ ] Cross-references valid and current
```

## Bypass Conditions (RARE)

Documentation updates may only be skipped if:
- Session involved pure research (no code/config changes)
- Emergency bug fix with immediate deploy requirement
- Owner explicitly requests bypass with documented reason

In all bypass cases: add note to next session explaining the gap.

## References

- **Templates**: `references/templates.md` -- STATUS update, BACKLOG update, ADR, CHANGELOG, API doc
- **Enforcement**: `references/enforcement.md` -- mandatory enforcement protocol, validation commands, error handling, recovery procedures

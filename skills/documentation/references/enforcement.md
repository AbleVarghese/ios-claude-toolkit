# Mandatory Enforcement Protocol

## Session-End Triggers (NON-OPTIONAL)

Before ending any session, execute this protocol. No exceptions.

### Pre-Response Validation Checklist

Before providing any "session complete" or similar final response:

```
MANDATORY_CHECKS = [
    "Check git status for uncommitted changes",
    "Detect if >10 files modified this session",
    "Validate status document 'Last Updated' is current",
    "Validate backlog document matches current session work",
    "Cross-check completion percentages are consistent",
    "Ensure all completed work items are documented",
    "Verify all new blockers are recorded"
]
```

### Enforcement Mechanism

**BLOCKING REQUIREMENT**: If ANY check fails:
1. Refuse to end session until documentation is updated
2. Auto-update required documents using templates (see `templates.md`)
3. Re-validate all checks before allowing session end
4. Generate conventional commit if documentation was updated

### Session-End Validation Commands

Execute these validations before session completion:

1. **Git Status Check**: `git status --porcelain | wc -l`
2. **Documentation Age Check**: Compare status document timestamp with current date
3. **Cross-Document Consistency**: Verify matching "Last Updated" dates
4. **Work Item Sync**: Ensure completed tasks are in backlog document
5. **Metrics Validation**: Check test counts, completion percentages align

## Error Handling and Recovery

### Stale Documentation Detection
```
Issue: Status or backlog document timestamp >24 hours old
Action: Immediate update with current session information
Validation: Verify all recent work is captured
Prevention: Enforce session end protocol
```

### Inconsistent Metrics
```
Issue: Different numbers reported across documents
Action: Identify canonical source and propagate correct values
Validation: Cross-check all dependent documents
Prevention: Single source validation rules
```

### Missing Work Items
```
Issue: Completed work not reflected in backlog document
Action: Add missing items with completion dates
Validation: Review session work against documented items
Prevention: Real-time work item tracking
```

### Duplicate Content
```
Issue: Same information in multiple documents
Action: Consolidate to appropriate SSOT document
Validation: Ensure all references point to canonical source
Prevention: SSOT violation monitoring
```

## Emergency Documentation Recovery

1. **Identify Scope**: Determine which documents are affected
2. **Establish Baseline**: Find last known good state via git history
3. **Reconstruct Changes**: Review git history and session logs
4. **Validate Accuracy**: Cross-check with implementation reality
5. **Update All Affected**: Ensure consistency across documents
6. **Prevent Recurrence**: Strengthen automation triggers

## Quality Gate Failures

1. **Stop Current Work**: Do not proceed with inconsistent docs
2. **Identify Root Cause**: Determine source of quality issue
3. **Fix Underlying Issue**: Address automation or process gap
4. **Validate Solution**: Test fix with current session
5. **Resume Work**: Continue only when quality gates pass

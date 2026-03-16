# Documentation Templates

## STATUS.md Update Template

```markdown
## Recent Changes

### YYYY-MM-DD Session - [SESSION_DESCRIPTION]
- **[COMPLETED_ITEM_1]**: [Description and impact]
- **[COMPLETED_ITEM_2]**: [Description and impact]
- **In Progress**: [CURRENT_WORK] (current session ongoing)
- **New Discovery**: [DISCOVERED_ISSUE] ([priority_level])
```

## BACKLOG.md Update Template

```markdown
## In Progress (Maximum 5 Items)

### Current Active Work Items
| Item | Assignee | Status | Expected Scope |
|------|----------|--------|----------------|
| **[CURRENT_WORK]** | Claude AI | [PERCENTAGE]% | [SCOPE_DESCRIPTION] |

### Work Item Details
1. **[CURRENT_WORK] ([CURRENT SESSION])**
   - [COMPLETED_SUBTASK_1]
   - [COMPLETED_SUBTASK_2]
   - [IN_PROGRESS_SUBTASK]
   - [REMAINING_SUBTASK]
```

## Architecture Decision Record Template

```markdown
# ADR-XXX: [Decision Title]

## Status
[Proposed | Accepted | Deprecated | Superseded]

## Context
[Why this decision is needed]

## Decision
[What was decided]

## Consequences
### Positive
- [Benefit 1]
- [Benefit 2]

### Negative
- [Drawback 1]
- [Drawback 2]

## Alternatives Considered
1. [Alternative 1] - Rejected because...
2. [Alternative 2] - Rejected because...
```

## CHANGELOG Entry Template

```markdown
## [Version] - YYYY-MM-DD

### Added
- New feature X that does Y

### Changed
- Modified behavior of Z

### Fixed
- Bug where A caused B

### Security
- Patched vulnerability in C

### Deprecated
- Feature D will be removed in version E
```

## API Documentation Template (DocC)

```swift
/// Brief one-line description
///
/// Detailed description of what this does,
/// why it exists, and any important context.
///
/// - Parameters:
///   - param1: Description of param1
///   - param2: Description of param2
/// - Returns: Description of return value
/// - Throws: `ErrorType` when condition occurs
///
/// - Example:
/// ```swift
/// let result = try await function(param1: value)
/// ```
///
/// - Note: Any important notes
/// - Warning: Any warnings about usage
func function(param1: Type, param2: Type) async throws -> ReturnType
```

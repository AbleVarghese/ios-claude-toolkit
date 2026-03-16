# Advanced Git Workflows

## Rebase Strategies

### When to Rebase vs Merge

| Scenario | Use | Why |
|----------|-----|-----|
| Feature branch behind main | Rebase | Linear history, clean log |
| Shared branch (multiple developers) | Merge | Preserves everyone's history |
| Long-lived feature branch | Merge | Too many conflicts to rebase cleanly |
| Single-developer branch, few commits | Rebase | Cleanest result |
| Release branch | Merge | Audit trail matters |

### Rebase Workflow

```bash
# Update feature branch with latest main
git checkout feature/scheduled-transfers
git fetch origin
git rebase origin/main

# If conflicts occur during rebase:
# 1. Resolve the conflict in the file
# 2. Stage the resolved file
git add Sources/Services/TransferService.swift
# 3. Continue rebase
git rebase --continue

# If rebase goes wrong:
git rebase --abort  # Returns to pre-rebase state
```

**Warning**: Never rebase commits that have been pushed to a shared branch. Rebase rewrites commit hashes, which diverges from the remote.

### Squash Before Merge (Conceptual)

When a feature branch has many small commits ("WIP", "fix typo", "actually fix it"):

```bash
# Squash last 5 commits into one
git reset --soft HEAD~5
git commit -m "feat(transfers): implement scheduled transfer support"

# Only do this for UNPUSHED commits on YOUR branch
```

---

## Cherry-Pick

### Single Commit

```bash
# Apply a specific commit from another branch to current branch
git cherry-pick abc1234

# Apply without committing (stage only)
git cherry-pick --no-commit abc1234
```

### Range of Commits

```bash
# Apply commits from abc1234 to def5678 (inclusive)
git cherry-pick abc1234^..def5678

# Note: abc1234^ means "start from the commit BEFORE abc1234"
# Without ^, abc1234 itself is excluded
```

### Cherry-Pick from Another Branch

```bash
# You need one specific bug fix from feature/payments on your branch
git log feature/payments --oneline
# Find the commit hash for the fix

git checkout feature/scheduled-transfers
git cherry-pick <fix-commit-hash>
```

### Handling Cherry-Pick Conflicts

```bash
# If cherry-pick conflicts:
# 1. Resolve the conflict
# 2. Stage resolved files
git add <resolved-file>
# 3. Continue
git cherry-pick --continue

# Or abort:
git cherry-pick --abort
```

**When to cherry-pick**: Hotfixes that need to go to multiple branches. Specific commits from abandoned branches. Backporting fixes to release branches.

---

## Bisect — Finding the Bug-Introducing Commit

`git bisect` performs a binary search through commit history to find which commit introduced a bug.

### Bisect Workflow

```bash
# 1. Start bisect
git bisect start

# 2. Mark current commit as bad (has the bug)
git bisect bad

# 3. Mark a known-good commit (before the bug existed)
git bisect good v1.2.0  # or a specific commit hash

# 4. Git checks out a middle commit. Test it.
#    - If the bug is present:
git bisect bad
#    - If the bug is NOT present:
git bisect good

# 5. Repeat step 4. Git narrows down automatically.
#    After log2(N) steps, it finds the exact commit.

# 6. Git reports: "abc1234 is the first bad commit"

# 7. End bisect
git bisect reset
```

### Automated Bisect with Test Script

```bash
# If you have a test that catches the bug:
git bisect start
git bisect bad HEAD
git bisect good v1.2.0
git bisect run swift test --filter "test_transferAmount_shouldNotRound"

# Git automatically runs the test at each step and finds the bad commit
```

**When to use bisect**: A bug appeared between two known points in time, and manual inspection of commits would take too long. Bisect reduces N commits to log2(N) checks.

---

## Stash Management

### Save Work in Progress

```bash
# Stash all uncommitted changes (tracked files only)
git stash push -m "WIP: transfer validation refactor"

# Stash including untracked files
git stash push -u -m "WIP: new transfer tests"

# Stash specific files only
git stash push -m "WIP: service changes" -- Sources/Services/TransferService.swift
```

### Retrieve Stashed Work

```bash
# List all stashes
git stash list
# stash@{0}: On feature/transfers: WIP: transfer validation refactor
# stash@{1}: On main: WIP: quick experiment

# Apply most recent stash (keeps it in stash list)
git stash apply

# Apply and remove from stash list
git stash pop

# Apply a specific stash
git stash apply stash@{1}
```

### Stash Inspection

```bash
# See what's in a stash without applying it
git stash show stash@{0}          # Summary
git stash show -p stash@{0}       # Full diff

# Create a branch from a stash (useful for complex stashes)
git stash branch recovery/validation-refactor stash@{0}
```

### Stash Cleanup

```bash
# Drop a specific stash
git stash drop stash@{1}

# Clear ALL stashes (destructive!)
git stash clear
```

**When to stash**: Quick context switch (interrupt), testing something on another branch, pulling when you have uncommitted changes.

**When NOT to stash**: Long-term storage of work. If work will be stashed for more than a few hours, commit it to a branch instead.

---

## Reflog — Your Safety Net

The reflog records every movement of HEAD. It's your last resort for recovering from mistakes.

### Common Reflog Uses

```bash
# View recent HEAD movements
git reflog
# or with timestamps:
git reflog --date=relative

# Find when a branch was at a specific state
git reflog show feature/transfers

# Find all commits that reference a specific message
git reflog | grep "transfer validation"
```

### Recovery Scenarios

```bash
# Scenario: Accidentally deleted a branch
git branch -D feature/important-work  # Oops!
git reflog | grep "important-work"
# Find the last commit hash
git checkout -b feature/important-work <commit-hash>

# Scenario: Lost commits after rebase
git reflog
# Find the commit hash from before the rebase
git reset --hard <pre-rebase-hash>

# Scenario: Need to see what I worked on yesterday
git reflog --date=relative | grep "yesterday"
```

### Reflog Expiration

```bash
# Default: unreachable commits kept for 30 days, reachable for 90 days
# Check current settings:
git config gc.reflogExpire        # default: 90 days
git config gc.reflogExpireUnreachable  # default: 30 days
```

---

## Tag Management

### Creating Tags

```bash
# Annotated tag (recommended — includes metadata)
git tag -a v1.2.0 -m "Release 1.2.0: scheduled transfers"

# Tag a specific commit (not HEAD)
git tag -a v1.1.1 -m "Hotfix: transfer rounding" abc1234
```

### Listing and Inspecting Tags

```bash
# List all tags
git tag

# List tags matching pattern
git tag -l "v1.2.*"

# Show tag details
git show v1.2.0
```

### Pushing Tags

```bash
# Push a specific tag
git push origin v1.2.0

# Push all tags
git push origin --tags
```

### Deleting Tags

```bash
# Delete local tag
git tag -d v1.2.0-beta

# Delete remote tag
git push origin --delete v1.2.0-beta
```

### Tagging Strategy for Releases

```
v1.0.0 — Major release (breaking changes)
v1.1.0 — Minor release (new features, backward compatible)
v1.1.1 — Patch release (bug fixes only)
v1.2.0-beta.1 — Pre-release (not for production)
```

Follow Semantic Versioning (semver.org):
- MAJOR: Breaking API changes
- MINOR: New features, backward compatible
- PATCH: Bug fixes, backward compatible

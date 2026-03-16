# Git Conflict Resolution Guide

## Conflict Types

### Content Conflict (Most Common)

Both branches modified the same lines in the same file.

```
<<<<<<< HEAD (your branch)
    func getBalance() async throws -> Decimal {
        try await repository.fetchBalance(userId: currentUserId)
    }
=======
    func getBalance(for userId: UUID) async throws -> Decimal {
        try await repository.fetchBalance(userId: userId)
    }
>>>>>>> feature/multi-user
```

**Resolution strategies**:

1. **Keep one side**: When one change supersedes the other
   ```swift
   // If feature/multi-user is the correct evolution:
   func getBalance(for userId: UUID) async throws -> Decimal {
       try await repository.fetchBalance(userId: userId)
   }
   ```

2. **Combine both**: When both changes are needed
   ```swift
   // If both the parameter change AND a new validation are needed:
   func getBalance(for userId: UUID) async throws -> Decimal {
       guard !userId.uuidString.isEmpty else {
           throw AppError.invalidInput(field: "userId", reason: "Empty")
       }
       try await repository.fetchBalance(userId: userId)
   }
   ```

### Rename Conflict

File was renamed on one branch and edited on another.

```bash
# Branch A renamed: UserService.swift → UserProfileService.swift
# Branch B edited: UserService.swift (added new method)

# Resolution: Apply Branch B's edits to the renamed file
git checkout feature-a -- Sources/Services/UserProfileService.swift
# Then manually add Branch B's new method to UserProfileService.swift
git add Sources/Services/UserProfileService.swift
git rm Sources/Services/UserService.swift  # Remove old name if still present
```

### Delete Conflict

File was deleted on one branch and edited on another.

```bash
# Branch A deleted: LegacyAuthService.swift
# Branch B edited: LegacyAuthService.swift (bug fix)

# Decision: Was the deletion correct?
# If YES (file is truly deprecated):
git rm Sources/Services/LegacyAuthService.swift
# Ensure the bug fix is applied to the replacement service

# If NO (file is still needed):
git checkout feature-b -- Sources/Services/LegacyAuthService.swift
git add Sources/Services/LegacyAuthService.swift
```

### Binary Conflict

Binary files (images, assets, compiled resources) cannot be merged.

```bash
# Choose one version explicitly:
git checkout --theirs -- Assets.xcassets/AppIcon.appiconset/icon.png
# OR
git checkout --ours -- Assets.xcassets/AppIcon.appiconset/icon.png

git add Assets.xcassets/AppIcon.appiconset/icon.png
```

---

## Resolution Workflow

### Step-by-Step Process

```bash
# 1. Understand the scope
git status
# Shows all files with conflicts

# 2. For EACH conflicting file:

# 2a. Read the conflict
cat Sources/Services/TransferService.swift | grep -n "<<<<<<\|======\|>>>>>>"
# Shows line numbers of all conflict markers

# 2b. Understand both sides
git log --oneline --left-right HEAD...MERGE_HEAD -- Sources/Services/TransferService.swift
# Shows commits on each side that touched this file

# 2c. Resolve (edit the file to remove markers and keep correct code)

# 2d. Verify no remaining markers
grep -r "<<<<<<\|======\|>>>>>>" Sources/Services/TransferService.swift
# Should return nothing

# 2e. Stage the resolved file
git add Sources/Services/TransferService.swift

# 3. After ALL conflicts resolved:
# 3a. Build and test
# 3b. Commit the merge
git commit -m "merge: resolve conflicts from feature/scheduled-transfers"
```

### Verification After Resolution

```bash
# Check for leftover conflict markers in entire codebase
grep -rn "<<<<<<\|======\|>>>>>>" Sources/ Tests/

# Build the project
xcodebuild build -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 16'

# Run tests
xcodebuild test -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 16'
```

---

## git rerere (Reuse Recorded Resolution)

`rerere` = "reuse recorded resolution." Git remembers how you resolved a conflict and automatically applies the same resolution if the same conflict occurs again.

### Setup

```bash
# Enable rerere globally
git config --global rerere.enabled true

# Enable auto-staging of rerere resolutions
git config --global rerere.autoupdate true
```

### How It Works

1. First time: You resolve a conflict manually. Git records the resolution.
2. Next time: Same conflict appears (e.g., during repeated rebases). Git auto-applies your previous resolution.
3. Verify: Always review auto-applied resolutions — they may not be correct if context changed.

### Managing rerere Cache

```bash
# View recorded resolutions
git rerere status

# Forget a specific resolution (if it was wrong)
git rerere forget Sources/Services/TransferService.swift

# Clear all recorded resolutions
git rerere gc
```

**When rerere is valuable**: Feature branches that are repeatedly rebased onto main. The same conflict resolution is needed each time you rebase.

---

## Emergency Recovery

### Recovering from a Bad Merge

```bash
# If you haven't committed the merge yet:
git merge --abort
# Returns to pre-merge state

# If you committed but haven't pushed:
git reset --hard HEAD~1
# WARNING: Destroys the merge commit. Only safe for unpushed commits.

# If you committed AND pushed:
git revert -m 1 HEAD
# Creates a new commit that undoes the merge. Safe for shared history.
```

### Recovering Lost Commits with Reflog

```bash
# View all recent HEAD movements (including "lost" commits)
git reflog

# Output looks like:
# abc1234 HEAD@{0}: commit: feat(auth): add biometric login
# def5678 HEAD@{1}: reset: moving to HEAD~1
# ghi9012 HEAD@{2}: commit: feat(auth): add token refresh  ← "lost" commit

# Recover the lost commit:
git cherry-pick ghi9012

# Or create a branch from it:
git checkout -b recovery/token-refresh ghi9012
```

**Reflog retention**: Git keeps reflog entries for 90 days by default. After that, unreachable commits may be garbage collected.

### Recovering from Accidental Hard Reset

```bash
# You did: git reset --hard HEAD~3 (accidentally lost 3 commits)
# 1. Find them in reflog
git reflog | head -10

# 2. Identify the commit BEFORE the reset
# Look for the entry just before "reset: moving to HEAD~3"

# 3. Reset back to that commit
git reset --hard <commit-hash>
```

### Recovering from Force Push

If someone force-pushed and overwrote your commits:

```bash
# 1. Find your commits in reflog
git reflog | grep "your commit message"

# 2. Create a branch from your last good commit
git checkout -b recovery/my-work <your-commit-hash>

# 3. Rebase onto the new main
git rebase origin/main

# 4. Push your recovered branch
git push -u origin recovery/my-work
```

**Prevention**: Enable branch protection rules on main. Disallow force push.

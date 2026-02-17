---
name: heimdall-git
description: "This skill should be used when the user asks to 'commit', 'push', 'create a branch', 'resolve merge conflict', or 'tag a release'. Provides Git operations with conventional commit enforcement, safe push, branch management, and conflict resolution."
allowed-tools:
  - Bash
  - Read
  - Grep
user-invocable: true
---

# HEIMDALL - Git Operations & Version Control

Git workflow skill covering conventional commits, safe push, branch management, conflict resolution, and release tagging.

## Standard Operating Procedures

### SOP-1: Smart Commit
```bash
# Analyze changes and generate commit
git add -A
git diff --cached --stat

# Generate conventional commit message
# Format: type(scope): description
git commit -m "feat(transfers): implement recipient selection flow"
```

### SOP-2: Safe Push
```bash
# Pre-push checks
git status
git log origin/main..HEAD --oneline

# Push to remote
git push origin main
```

### SOP-3: Branch Management
```bash
# Create feature branch
git checkout -b feature/recipient-selection

# Merge back to main
git checkout main
git merge --no-ff feature/recipient-selection
git branch -d feature/recipient-selection
```

### SOP-4: Conflict Resolution
```bash
# Identify conflicts
git status

# For each conflict:
# 1. Open file
# 2. Find <<<<<<< markers
# 3. Choose correct version
# 4. Remove markers
# 5. git add <file>

git commit -m "merge: resolve conflicts from feature branch"
```

## Conventional Commit Types

| Type | When to Use | Example |
|------|-------------|---------|
| `feat` | New feature | `feat(auth): add biometric login` |
| `fix` | Bug fix | `fix(transfer): correct amount calculation` |
| `docs` | Documentation | `docs(readme): update setup instructions` |
| `test` | Test changes | `test(payment): add edge case coverage` |
| `refactor` | Code restructure | `refactor(di): simplify container registration` |
| `perf` | Performance | `perf(list): implement lazy loading` |
| `chore` | Maintenance | `chore(deps): update Stripe SDK` |
| `ci` | CI/CD changes | `ci(fastlane): add test lane` |
| `security` | Security fix | `security(auth): patch token refresh` |

## Branch Strategy (Trunk-Based)

```
main (protected)
  |
  +-- Direct commits for small changes
  |
  +-- feature/* branches for larger work
      +-- Merge via PR with squash
```

## Git Safety Rules

- NEVER force push to main
- NEVER commit secrets
- ALWAYS verify before push
- ALWAYS use conventional commits

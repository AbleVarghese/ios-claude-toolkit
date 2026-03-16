---
name: heimdall-git
description: "Done for now, wrap up, end of session, save progress, save my work, upload, publish, commit, push, sync. Git operations with conventional commits, safe push, and branch management. Trigger on: committing, pushing, branching, merge conflicts, tagging releases, 'upload changes', 'ship it', 'let me save this', or any signal the user is finished working and wants to preserve their changes. Also handles branch creation, conflict resolution, and trunk-based workflow. Enforces branch verification before every operation."
allowed-tools:
  - Bash
  - Read
  - Grep
user-invocable: true
---

# HEIMDALL — Git Operations & Version Control

Git workflow skill covering conventional commits, safe push, branch management, conflict resolution, and release tagging. Every git operation begins with branch verification.

> **Iron Law**: "Verify branch before EVERY git operation. No exception."

---

## Phase Gate: PRE-OPERATION (Before EVERY Git Command)
**Cannot execute ANY git command until:**
- [ ] `git branch --show-current` — verified you're on the correct branch
- [ ] `git status` — reviewed for unexpected changes

This gate applies to EVERY git operation, EVERY time. Not just the first one. Branch switches happen silently (stash pop, rebase, checkout). Always verify.

**If on wrong branch**: `git checkout <correct-branch>` FIRST. Never rename branches as a workaround.

---

## Pre-Operation Safety Protocol

**Every git operation** — commit, push, merge, rebase, cherry-pick, tag — must start with this protocol. No shortcuts.

```bash
# Step 1: Verify branch
CURRENT_BRANCH=$(git branch --show-current)
echo "Current branch: $CURRENT_BRANCH"

# Step 2: Check for stashed changes that might be forgotten
STASH_COUNT=$(git stash list | wc -l)
if [ "$STASH_COUNT" -gt 0 ]; then
    echo "WARNING: $STASH_COUNT stashed changes exist"
    git stash list
fi

# Step 3: Check for uncommitted changes
if ! git diff --quiet || ! git diff --cached --quiet; then
    echo "Uncommitted changes detected:"
    git status --short
fi

# Step 4: Verify you're not on a detached HEAD
if [ -z "$CURRENT_BRANCH" ]; then
    echo "ERROR: Detached HEAD state. Checkout a branch before proceeding."
    exit 1
fi
```

**Why this protocol**: Agents, parallel sessions, stash operations, and cherry-picks can silently change the current branch. Committing to the wrong branch causes hours of cleanup. This 4-step check takes 2 seconds and prevents those hours.

---

## SOP-1: Smart Commit

### Full Commit Workflow

```bash
# 1. VERIFY BRANCH (mandatory first step)
git branch --show-current

# 2. Review what changed
git status
git diff --stat

# 3. Stage specific files (NEVER use git add -A blindly)
#    Review each file before staging
git add Sources/Services/TransferService.swift
git add Tests/TransferServiceTests.swift

# 4. Verify staged changes are correct
git diff --cached --stat
git diff --cached  # Read the actual diff

# 5. Generate conventional commit message
#    Analyze the diff to determine type and scope
git commit -m "feat(transfers): implement recipient validation with balance check"
```

### Commit Message Generation Algorithm

```
1. Analyze changed files:
   - What directories/modules were touched? → This is the SCOPE
   - What's the nature of the change? → This determines the TYPE

2. Determine TYPE from the diff:
   - New functionality added? → feat
   - Bug fixed? → fix
   - Code restructured without behavior change? → refactor
   - Tests added/changed? → test
   - Documentation updated? → docs
   - Performance improved? → perf
   - Dependencies/tooling changed? → chore
   - CI/CD changed? → ci
   - Build system changed? → build
   - Code style/formatting? → style
   - Security fix? → security

3. Determine SCOPE from changed file paths:
   - Sources/Services/Transfer* → transfers
   - Sources/ViewModels/Auth* → auth
   - Sources/Infrastructure/Supabase* → db
   - Tests/* → (use the module being tested)

4. Write description:
   - Lowercase
   - Imperative mood ("add", not "adds" or "added")
   - Under 50 characters
   - Describes WHAT was done, not HOW

5. Format: type(scope): description
```

### Commit Message Examples

| Diff Shows | Commit Message |
|-----------|---------------|
| New async method in TransferService + tests | `feat(transfers): add recipient validation` |
| Fixed nil crash in UserViewModel | `fix(profile): handle missing user gracefully` |
| Moved validation from VM to Service | `refactor(auth): extract validation to service layer` |
| Added test for edge case | `test(payments): cover zero-amount transfer rejection` |
| Updated dependency versions | `chore(deps): update Supabase SDK to 2.5.0` |
| Improved query performance | `perf(transfers): batch recipient lookups` |
| Fixed token refresh race condition | `security(auth): serialize token refresh requests` |

### Phase Gate: PRE-COMMIT
**Cannot `git commit` until:**
- [ ] `git diff --cached --stat` reviewed — you know what you're committing
- [ ] No `.env`, credential, or secret files staged
- [ ] Commit message follows conventional format: `type(scope): description`
- [ ] Staged files are specific (not `git add -A` or `git add .`)

**Proof**: List the files being committed and the commit message before executing.

### Multi-Line Commit Messages (When Needed)

Use HEREDOC syntax for commits that need a body:

```bash
git commit -m "$(cat <<'EOF'
feat(transfers): implement scheduled transfer support

- Add ScheduledTransfer entity with recurrence rules
- Create ScheduledTransferService with CRON-based scheduling
- Add database migration for scheduled_transfers table
- Wire up DI container registration

Closes #142
EOF
)"
```

---

## SOP-2: Safe Push

```bash
# 1. VERIFY BRANCH
CURRENT_BRANCH=$(git branch --show-current)
echo "Pushing from: $CURRENT_BRANCH"

# 2. Check what will be pushed
git log origin/$CURRENT_BRANCH..HEAD --oneline
# If this shows nothing, there's nothing to push

# 3. Verify no secrets in staged commits
git log origin/$CURRENT_BRANCH..HEAD -p | grep -iE "(api_key|secret|password|token|sk_live|sk_test)" && echo "WARNING: Possible secret detected!" || echo "No secrets detected"

# 4. Push
git push origin $CURRENT_BRANCH
```

### Never Force Push to Protected Branches

```
PROTECTED BRANCHES: main, master, develop, release/*

Force push to these branches:
- Destroys other developers' work
- Breaks CI/CD pipelines
- Corrupts shared history
- Is NEVER justified

If you think you need to force push to main:
1. Stop
2. Explain the situation to the owner
3. Use a different strategy (revert commit, fixup commit)
```

---

## SOP-3: Branch Management

### Creating Feature Branches

```bash
# 1. Start from up-to-date main
git checkout main
git pull origin main

# 2. Create feature branch with descriptive name
git checkout -b feature/scheduled-transfers

# 3. Push branch to remote to establish tracking
git push -u origin feature/scheduled-transfers
```

### Branch Naming Convention

| Prefix | Purpose | Example |
|--------|---------|---------|
| `feature/` | New feature | `feature/scheduled-transfers` |
| `fix/` | Bug fix | `fix/transfer-amount-rounding` |
| `refactor/` | Code restructure | `refactor/di-container-simplification` |
| `chore/` | Maintenance task | `chore/update-dependencies` |
| `release/` | Release preparation | `release/1.2.0` |
| `hotfix/` | Emergency production fix | `hotfix/auth-token-refresh` |

### Merging Feature Branches

```bash
# 1. Update main
git checkout main
git pull origin main

# 2. Merge with merge commit (preserves branch history)
git merge --no-ff feature/scheduled-transfers

# 3. Push
git push origin main

# 4. Delete merged branch
git branch -d feature/scheduled-transfers
git push origin --delete feature/scheduled-transfers
```

---

## SOP-4: Conflict Resolution

### Conflict Resolution Decision Tree

```
START: You have a merge conflict.

Q1: What type of conflict is it?
├── CONTENT conflict (same lines changed by both sides)
│   Q2: Do both changes need to be kept?
│   ├── YES → Manually combine both changes. Test the result.
│   └── NO → Keep the correct version. Verify with tests.
│
├── RENAME conflict (file moved/renamed on one side, edited on other)
│   → Accept the rename. Apply the edits to the new location.
│
├── DELETE conflict (file deleted on one side, edited on other)
│   Q3: Should the file exist?
│   ├── YES → Keep the file with edits.
│   └── NO → Accept the deletion. Verify no broken references.
│
└── BINARY conflict (non-mergeable files like images, assets)
    → Choose one version. Cannot merge binary files.
```

### Conflict Resolution Workflow

```bash
# 1. Identify conflicting files
git status  # Shows files with "both modified"

# 2. For each conflicting file:
#    a. Open the file
#    b. Find conflict markers: <<<<<<<, =======, >>>>>>>
#    c. Understand BOTH sides of the conflict
#    d. Choose the correct resolution (may combine both)
#    e. Remove ALL conflict markers
#    f. Test the resolved code

# 3. Stage resolved files
git add path/to/resolved/file.swift

# 4. Complete the merge
git commit -m "merge: resolve conflicts from feature/scheduled-transfers"

# 5. Verify
git log --oneline -5  # Confirm merge commit exists
```

### Post-Resolution Verification

After resolving conflicts:
1. Build the project — it must compile
2. Run tests — all must pass
3. Review the resolution diff — ensure no accidental deletions
4. Check for leftover conflict markers: `grep -r "<<<<<<" Sources/`

---

### SOP-5: Bulk Commit (20+ Changed Files)

When staging 20+ files, the standard "stage specific files" approach needs structure:

**Step 1: Categorize changes**
```bash
# Group by change type
git diff --name-only --diff-filter=A  # New files
git diff --name-only --diff-filter=M  # Modified files
git diff --name-only --diff-filter=D  # Deleted files
```

**Step 2: Consider splitting into logical commits**
If changes span multiple concerns, split into separate commits:
```bash
# Commit 1: Database changes
git add Backend/supabase/migrations/*.sql
git commit -m "feat(database): add recurring_transfers table with RLS"

# Commit 2: Domain layer
git add Sources/Domain/Models/*.swift Sources/Domain/Ports/*.swift
git commit -m "feat(domain): add RecurringTransfer model and port"

# Commit 3: Implementation
git add Sources/Infrastructure/*.swift Sources/Presentation/*.swift
git commit -m "feat(transfers): implement recurring transfer service and UI"

# Commit 4: Tests
git add Tests/**/*.swift
git commit -m "test(transfers): add recurring transfer test coverage"
```

**Step 3: If single commit is appropriate**
For bulk changes that are genuinely one concern (e.g., rename, formatting):
```bash
# Review ALL changes first
git diff --stat
# Stage by directory (more controlled than git add -A)
git add Sources/
git add Tests/
git diff --cached --stat  # Verify what's staged
```

**Decision tree: Split or single?**
- Different concerns (feature + tests + migration)? → Split into logical commits
- Same concern (bulk rename, formatting, dependency update)? → Single commit is fine
- Unsure? → Split. Atomic commits are easier to revert, bisect, and review.

---

## Conventional Commit Types

| Type | When to Use | Example |
|------|-------------|---------|
| `feat` | New feature or capability | `feat(auth): add biometric login` |
| `fix` | Bug fix | `fix(transfer): correct amount rounding` |
| `docs` | Documentation only | `docs(readme): update setup instructions` |
| `test` | Test additions or fixes | `test(payment): add edge case coverage` |
| `refactor` | Code restructure, no behavior change | `refactor(di): simplify container registration` |
| `perf` | Performance improvement | `perf(list): implement lazy loading` |
| `chore` | Maintenance, dependency updates | `chore(deps): update Stripe SDK to 3.0` |
| `ci` | CI/CD configuration | `ci(fastlane): add test lane` |
| `build` | Build system changes | `build(spm): add new package dependency` |
| `style` | Code style/formatting only | `style: apply swiftformat rules` |
| `security` | Security fix or improvement | `security(auth): patch token refresh vulnerability` |

---

## Branch Strategy (Trunk-Based Development)

```
main (protected, always deployable)
  │
  ├── Direct commits for small changes (<1 hour of work)
  │
  ├── feature/* branches for larger work (>1 hour)
  │   └── Merge via PR with squash or merge commit
  │
  ├── hotfix/* branches for emergency fixes
  │   └── Merge directly to main, then backport if needed
  │
  └── release/* branches for release preparation
      └── Tag with version, merge back to main
```

---

## User Intent Mapping

When the user expresses intent, map to the correct git operation:

| User Says | Git Operations |
|-----------|---------------|
| "commit" | Verify branch → stage changed files → commit with conventional message |
| "push" | Verify branch → push to remote |
| "commit and push" / "save" / "done for now" | Verify branch → stage → commit → push |
| "sync" | Verify branch → pull → commit → push |
| "upload" | Verify branch → push to remote |

**Critical**: Each session must commit its own work. Changes made in one session are NOT automatically included when a different session commits.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|---------------|-----------------|
| "I'll just force push this once" | Force push rewrites shared history. Other developers' branches diverge. CI artifacts become orphaned. | Use `git revert` to undo a bad commit. Never rewrite shared history. |
| "I know what branch I'm on" | You don't. Agents, stash pops, and cherry-picks silently switch branches. | Run `git branch --show-current` before every operation. Takes 0.1 seconds. |
| "This commit message is fine" | Vague commit messages make `git log`, `git bisect`, and changelogs useless. | Follow conventional commit format. 5 seconds more now saves 5 minutes later. |
| "Nobody reads commit messages" | You read them during bisect, blame, revert, and changelog generation. Future-you reads them. | Write for future-you who is debugging at 2am. |
| "I'll fix the history later" | Rewriting history after push is destructive. "Later" means "never" or "with collateral damage." | Get it right the first time. Amend only unpushed commits. |
| "git add -A is faster" | It stages everything including .env files, build artifacts, and debug files. | Stage specific files by name. Review with `git diff --cached`. |

---

## HARD STOP: Dangerous Operations

These commands require EXPLICIT human approval before execution:
- `git push --force` — Explain why. Get approval. NEVER force push to main/master.
- `git reset --hard` — What will be lost? Is it recoverable? Get approval.
- `git branch -D` — Is this branch merged? Could it have unmerged work?
- `git checkout -- .` / `git restore .` — This discards ALL uncommitted changes. Are you sure?

Proceeding without approval is a violation. These operations destroy work that cannot be recovered.

---

## Red Flags — HARD STOP

These are BLOCKING conditions, not suggestions. If ANY red flag is triggered, you CANNOT continue the current operation until the violation is remediated. Proceeding past a red flag invalidates all subsequent work.

- **Force pushing to main/master/develop**: This destroys shared history. Use `git revert` instead.
- **Committing .env, credentials, or secrets**: If already committed, the secret is compromised even after removal. Rotate the secret immediately.
- **Amending a commit that's already pushed**: This rewrites shared history. Create a new fixup commit instead.
- **Not verifying the current branch**: The most common source of git disasters. Always verify.
- **`git add -A` or `git add .` without reviewing**: You don't know what you're staging. Stage specific files.
- **Committing to detached HEAD**: Your commits will be garbage collected. Checkout a branch first.

---

## When NOT to Use This Skill

- **Writing or reviewing code** → Use `forge-development` or `ios-review`
- **Creating or updating documentation content** → Use `documentation`
- **Coordinating multi-step workflows** → Use `maestro-orchestrator`
- **Performance or security analysis** → Use `prometheus-performance` or `cipher-security`

---

## Decision Framework: Git Operation Selection

```
START: User wants to do something with git.

Q1: What is the intent?
├── COMMIT (save changes locally)
│   → Pre-op safety check → review changes → stage specific files → commit with conventional message
│
├── PUSH (send to remote)
│   → Pre-op safety check → verify commits to push → check for secrets → push
│
├── SAVE / DONE (commit + push)
│   → Pre-op safety check → stage → commit → push
│
├── BRANCH (create/switch/delete)
│   → Pre-op safety check → verify clean state → create/switch/delete branch
│
├── MERGE (combine branches)
│   → Pre-op safety check → verify target branch → merge → resolve conflicts if any → push
│
├── CONFLICT (resolve merge conflicts)
│   → Identify conflict type → resolve per decision tree → test → commit
│
├── RECOVER (undo mistake)
│   Q2: Was the commit pushed?
│   ├── YES → Use `git revert` (creates new commit that undoes the change)
│   └── NO → Use `git reset` or `git commit --amend` (safe for local-only)
│
└── INVESTIGATE (find information)
    → Use `git log`, `git blame`, `git bisect`, `git reflog`
```

---

## Quality Gates (Before Marking Complete)

- [ ] Branch verified before every git operation
- [ ] No secrets in staged changes (grep for API keys, tokens, passwords)
- [ ] Conventional commit message format: `type(scope): description`
- [ ] Commit message describes WHAT and WHY, not HOW
- [ ] Only specific files staged (not `git add -A`)
- [ ] Staged diff reviewed before committing
- [ ] No force push to protected branches
- [ ] Post-push verification (confirm remote received the commit)

---

## Self-Audit (Before Declaring Git Operation Complete)

Answer each question. If ANY answer is "no" or "unsure" — go back and fix it.

1. Did I verify the branch before EVERY git command? (Not just the first one)
2. Did I review `git diff --cached` before committing? (Did I actually read the diff?)
3. Are all staged files intentional? (No accidental `.env`, build artifacts, or debug files)
4. Does the commit message follow conventional format with accurate type and scope?
5. Did I check for secrets in the staged changes?
6. If I pushed, did I verify the remote received the commit?

---

## Required Output Artifact

Every git operation using this skill must produce:
1. **Branch verification** — output of `git branch --show-current` before the operation
2. **Change summary** — list of files staged/committed/pushed
3. **Commit message** — following `type(scope): description` format
4. **Post-operation status** — output of `git status` or `git log --oneline -1` confirming success

A commit without branch verification evidence is a violation. A push without post-push verification is incomplete.

---

## Cross-Skill References

| Skill | When to Use |
|-------|-------------|
| `forge-development` | When you need to write code before committing |
| `ios-review` | When reviewing changes before committing or merging |
| `documentation` | When documentation needs updating as part of the commit |
| `maestro-orchestrator` | When git operations are part of a larger coordinated workflow |

---

## References

- **Conflict Resolution**: `references/conflict-resolution.md` — detailed strategies by conflict type, git rerere, emergency recovery
- **Advanced Workflows**: `references/advanced-workflows.md` — rebase, cherry-pick, bisect, stash, reflog, tag management

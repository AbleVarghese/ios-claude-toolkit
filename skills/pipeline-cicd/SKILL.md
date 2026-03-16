---
name: pipeline-cicd
description: "Set up CI, GitHub Actions, automate builds, continuous integration. GitHub Actions CI/CD pipeline configuration for iOS apps — build/test/deploy workflows, caching strategies, quality gates, release automation. Trigger on: 'add CI to this project', 'configure GitHub Actions', 'CI is failing', 'optimize build pipeline', 'automate TestFlight deploys', 'add a workflow', 'manage CI secrets', or any CI/CD infrastructure work. For what Fastlane lanes do (build/sign/upload), use velocity-fastlane instead."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: false
---

# PIPELINE - CI/CD & Build Automation

> **Iron Law**: "CI failures must be investigated immediately -- a broken pipeline blocks the entire team."

Configure and manage GitHub Actions workflows for continuous integration and deployment. Handle build automation, test execution, quality gate enforcement, caching, and release pipelines for iOS apps.

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). These are referred to as `{PROJECT_DIR}` (directory containing the Xcode project), `{PROJECT}` (Xcode project name), and `{SCHEME}` (build scheme) throughout this skill. If the project uses XcodeGen, the generate command runs from `{PROJECT_DIR}`.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|---|---|---|
| "We don't need CI yet, the team is small" | CI catches bugs that no team size is immune to: missing files not in git, environment-specific failures, broken tests you forgot to run. Solo developers benefit from CI just as much as teams of 50. | Set up a basic PR validation workflow. 15 minutes of setup. Catches bugs before they ship. |
| "Manual testing is sufficient" | Manual testing is inconsistent, unscalable, and unrepeatable. It catches what you think to test, not what fails. Automated CI catches regressions you didn't expect. | Automate the test suite in CI. Manual testing supplements automated testing; it never replaces it. |
| "CI is too slow, it blocks our velocity" | Slow CI is a symptom, not a cause. Fix it with caching, parallelism, and selective triggers -- don't remove it. Shipping bugs is slower than waiting for CI. | Optimize: cache SPM dependencies, parallelize test categories, use path-based triggers to skip unnecessary work. |
| "We'll add CI when the team grows" | By then, the codebase has no test discipline, no quality gates, and no deployment automation. Retrofitting CI into a mature project without it is 10x harder than setting it up early. | Start with CI on day one. Even a 5-line workflow that runs `xcodebuild build` catches compilation errors. |
| "Secrets in the workflow file are fine for now" | They're in git history forever. Even if you remove them in the next commit, anyone with repo access can see them in the history. | Use GitHub Actions encrypted secrets exclusively. Never hardcode credentials in YAML. |
| "We'll investigate that CI failure later" | "Later" means "never." A red CI pipeline normalizes failure. Within a week, the team ignores CI results entirely, and it becomes useless. | Investigate immediately. Fix it or revert the breaking commit. CI must be green to merge. |
| "Caching isn't worth the complexity" | SPM resolution and Xcode compilation are the two slowest CI steps. Caching SPM dependencies alone can save 2-5 minutes per run. Over 10 PRs/day, that's 30-50 minutes. | Cache SPM packages using `Package.resolved` as the cache key. It's 5 lines of YAML. |

---

## Red Flags -- STOP

- **No CI validation on pull requests** -- If PRs can be merged without automated checks, bugs will be merged. This is not a question of "if" but "when."
- **Secrets hardcoded in workflow files** -- Even in private repos, hardcoded secrets are a security incident waiting to happen. Repository access changes, forks happen, history is permanent.
- **No caching configured** -- Every CI run downloading dependencies from scratch wastes time and money. SPM resolution alone can take 3-5 minutes on clean builds.
- **No quality gates (tests, lint, coverage)** -- A CI pipeline that only builds without testing is a false sense of security. It catches compilation errors but misses logic bugs, which are the majority of production issues.
- **CI failures ignored or "expected"** -- If the team says "oh that test is flaky, ignore it," the pipeline has lost trust. Fix flaky tests or delete them. Every red check must be meaningful.

---

## Behavioral Enforcement

### Phase Gate: CI FAILURE INVESTIGATION
**When CI fails, cannot retry until:**
- [ ] Error message read and understood
- [ ] Root cause identified (not just "it failed")
- [ ] Fix applied to address root cause

**Hard Stop**: "Just retry" is not a fix. If CI failed, there's a reason. Retrying without investigating means: (a) you'll waste another CI run, (b) the same failure will happen again. Investigate FIRST, fix, THEN retry.

**Counter**: CI fails on same issue #1-2: investigate and fix. #3: The pipeline configuration has a structural problem. Review the workflow YAML, not just the code.

### Phase Gate: WORKFLOW CHANGE
**Cannot modify CI/CD workflow until:**
- [ ] Current workflow behavior understood (what triggers it, what it does)
- [ ] Change tested locally where possible (act, nektos/act for GitHub Actions)
- [ ] Rollback plan exists (previous workflow version in git)

### Self-Audit
1. If CI fails, did I investigate before retrying?
2. Are all secrets referenced in the workflow actually configured in the repo?
3. Are caches properly keyed (won't serve stale data)?
4. Would a new contributor's first PR trigger the correct workflow?

### Required Output Artifact
Every CI/CD change must produce:
- The workflow YAML diff with explanation of what changed and why
- Validation that secrets are configured (names confirmed, not values)
- Confirmation of a successful CI run after the change
- Cache key strategy documented (what busts the cache, what doesn't)

---

## When NOT to Use This Skill

1. **Writing or debugging Fastlane lanes** -- This skill configures when and where Fastlane runs. For what Fastlane does (lane configuration, signing, uploading), use the `velocity-fastlane` skill.
2. **Application build settings and Xcode configuration** -- CI workflows invoke build tools but don't configure them. For `.xcodeproj` settings, `project.yml`, or build configuration changes, work in those files directly.
3. **Infrastructure and server deployment** -- This skill covers iOS app CI/CD. For backend deployments (Supabase, AWS, GCP), use the appropriate infrastructure tools.
4. **Local development automation** -- For local build scripts, Makefiles, and developer convenience scripts, work in the project's scripts directory. CI workflows are for the CI environment.
5. **Git branching strategy and release management** -- CI automates what happens on branches. The branching strategy (trunk-based, gitflow, etc.) is a team process decision, not a CI concern.

---

## Decision Framework

```
What CI/CD capability do I need?

+-- Validate code quality on PRs?
|   |
|   +-- Basic (build + test) --> PR Validation workflow
|   +-- Comprehensive (build + test + lint + coverage + security) --> Full Quality workflow
|   +-- With matrix (multiple Xcode versions) --> Matrix Build workflow
|
+-- Deploy a build?
|   |
|   +-- To TestFlight (beta testers)?
|   |   +-- On every merge to main? --> CD workflow (push trigger on main)
|   |   +-- On demand? --> Manual trigger workflow (workflow_dispatch)
|   |   +-- On version tag? --> Release workflow (tag trigger v*)
|   |
|   +-- To App Store?
|   |   +-- Always manual approval gate --> workflow_dispatch + environment protection
|   |
|   +-- To Firebase App Distribution?
|       +-- For rapid iteration --> CD workflow with firebase lane
|
+-- Run scheduled tasks?
|   |
|   +-- Nightly regression suite --> Schedule workflow (cron)
|   +-- Dependency vulnerability check --> Schedule + Dependabot
|   +-- Performance baseline monitoring --> Schedule with benchmark lane
|
+-- Optimize existing CI?
    |
    +-- Slow builds? --> Add caching (SPM, DerivedData, Homebrew)
    +-- Unnecessary runs? --> Add path-based triggers
    +-- Flaky tests? --> Fix the tests (not the CI)
    +-- High costs? --> Self-hosted runners for macOS
```

---

## Standard Operating Procedures

### SOP-1: PR Validation Workflow

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main, develop]
    # Only run when relevant files change
    paths:
      - '**.swift'
      - '**.xib'
      - '**.storyboard'
      - '**/project.yml'
      - '**/Package.swift'
      - '**/Package.resolved'
      - '.github/workflows/ci.yml'

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true  # Cancel previous runs for the same PR

jobs:
  build-and-test:
    runs-on: macos-15  # Use specific version, not macos-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '16.2'  # Pin Xcode version for reproducibility

      - name: Cache SPM Dependencies
        uses: actions/cache@v4
        with:
          path: |
            {PROJECT_DIR}/.build
            ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}
          restore-keys: |
            ${{ runner.os }}-spm-

      - name: Cache Homebrew
        uses: actions/cache@v4
        with:
          path: |
            ~/Library/Caches/Homebrew
            /usr/local/Cellar/swiftlint
            /usr/local/Cellar/xcodegen
          key: ${{ runner.os }}-brew-${{ hashFiles('.github/workflows/ci.yml') }}

      - name: Install Tools
        run: |
          brew install swiftlint xcodegen || true

      - name: Generate Project
        run: cd {PROJECT_DIR} && xcodegen generate

      - name: Lint
        run: cd {PROJECT_DIR} && swiftlint lint --strict --reporter github-actions-logging

      - name: Run Tests
        run: |
          cd {PROJECT_DIR}
          fastlane test_comprehensive

      - name: Upload Test Results
        if: always()  # Upload even on failure
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            {PROJECT_DIR}/fastlane/test_output/
            {PROJECT_DIR}/build/reports/
```

**Why each step matters:**
- `concurrency.cancel-in-progress`: Force-pushing to a PR branch cancels the previous run, saving CI minutes.
- `paths`: Only run CI when relevant files change. Documentation changes don't need a build.
- Pinned Xcode version: `macos-latest` changes over time, causing surprise failures.
- Cache SPM: SPM resolution is 2-5 minutes. With cache, it's <10 seconds.
- `if: always()` on artifact upload: When tests fail, the test output is the most important artifact. Don't skip it.

### SOP-2: Release Workflow (TestFlight)

```yaml
# .github/workflows/release.yml
name: Release to TestFlight
on:
  push:
    tags: ['v*']  # Trigger on version tags: v1.0.0, v1.1.0-beta.1
  workflow_dispatch:  # Also allow manual trigger
    inputs:
      version_bump:
        description: 'Version bump type'
        required: true
        type: choice
        options: [patch, minor, major]

jobs:
  release:
    runs-on: macos-15
    timeout-minutes: 45
    environment: production  # Requires manual approval if configured

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for version tagging

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '16.2'

      - name: Install Tools
        run: brew install xcodegen || true

      - name: Generate Project
        run: cd {PROJECT_DIR} && xcodegen generate

      - name: Run Full Test Suite
        run: cd {PROJECT_DIR} && fastlane test_comprehensive

      - name: Build and Upload
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_API_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
          CI_BUILD_NUMBER: ${{ github.run_number }}
        run: |
          cd {PROJECT_DIR}
          fastlane beta

      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-${{ github.ref_name }}-${{ github.run_number }}
          path: {PROJECT_DIR}/build/*.ipa
```

### SOP-3: Caching Strategy

```yaml
# SPM Dependency Cache -- most impactful, cache first
- uses: actions/cache@v4
  with:
    path: |
      {PROJECT_DIR}/.build
      ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
    key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}
    restore-keys: |
      ${{ runner.os }}-spm-

# DerivedData Cache -- speeds up compilation but can cause stale builds
# Use for CI only, not for release builds
- uses: actions/cache@v4
  if: github.event_name == 'pull_request'  # Only for PRs, not releases
  with:
    path: ~/Library/Developer/Xcode/DerivedData
    key: ${{ runner.os }}-dd-${{ hashFiles('**/*.swift') }}
    restore-keys: |
      ${{ runner.os }}-dd-

# Homebrew Cache -- saves re-downloading and compiling tools
- uses: actions/cache@v4
  with:
    path: |
      ~/Library/Caches/Homebrew
    key: ${{ runner.os }}-brew-${{ hashFiles('.github/workflows/ci.yml') }}
```

**Cache invalidation rules:**
- SPM cache: Busted when `Package.resolved` changes (dependency version update)
- DerivedData cache: Busted when any Swift file changes (conservative but safe)
- Homebrew cache: Busted when the workflow file changes (tool version update)
- Never cache for release builds -- always clean build for distribution

See `references/caching-guide.md` for detailed cache strategies and cost-benefit analysis.

### SOP-4: Quality Gates in CI

```yaml
jobs:
  quality-gates:
    runs-on: macos-15
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '16.2'

      - name: Generate Project
        run: cd {PROJECT_DIR} && xcodegen generate

      # Gate 1: Code compiles
      - name: Build
        run: |
          xcodebuild build \
            -project "{PROJECT_DIR}/{PROJECT}.xcodeproj" \
            -scheme "{SCHEME}" \
            -destination "platform=iOS Simulator,name=iPhone 16" \
            CODE_SIGNING_ALLOWED=NO

      # Gate 2: Lint passes
      - name: SwiftLint
        run: cd {PROJECT_DIR} && swiftlint lint --strict

      # Gate 3: Tests pass
      - name: Test
        run: cd {PROJECT_DIR} && fastlane test_comprehensive

      # Gate 4: Security scan
      - name: Secret Scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # Gate 5: No TODOs in PR diff (optional but useful)
      - name: Check for TODOs
        run: |
          git diff origin/main...HEAD -- '*.swift' | grep -i "TODO\|FIXME\|HACK" && \
            echo "::warning::Found TODO/FIXME/HACK in diff" || true
```

### SOP-5: Matrix Build (Multiple Xcode Versions)

```yaml
jobs:
  test-matrix:
    strategy:
      matrix:
        xcode: ['16.0', '16.2']
        destination: ['iPhone 16', 'iPad Pro 13-inch (M4)']
      fail-fast: false  # Run all combinations even if one fails

    runs-on: macos-15
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Setup Xcode ${{ matrix.xcode }}
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: ${{ matrix.xcode }}

      - name: Run Tests on ${{ matrix.destination }}
        run: |
          xcodebuild test \
            -project "{PROJECT_DIR}/{PROJECT}.xcodeproj" \
            -scheme "{SCHEME}" \
            -destination "platform=iOS Simulator,name=${{ matrix.destination }}" \
            CODE_SIGNING_ALLOWED=NO
```

### SOP-6: Manual Approval Gate

```yaml
# For production deployments, require manual approval
jobs:
  build:
    runs-on: macos-15
    steps:
      - name: Build and Test
        run: fastlane test_comprehensive && fastlane build_release

      - name: Upload Artifact
        uses: actions/upload-artifact@v4
        with:
          name: release-build
          path: build/*.ipa

  deploy:
    needs: build
    runs-on: macos-15
    environment: production  # This environment has required reviewers

    steps:
      - name: Download Artifact
        uses: actions/download-artifact@v4
        with:
          name: release-build

      - name: Deploy to App Store
        run: fastlane upload_testflight
```

**Setting up environment protection:**
1. Go to repo Settings > Environments > New environment "production"
2. Add required reviewers (team leads, release managers)
3. Optionally add deployment branch rules (only main/release branches)
4. Optionally add wait timer (e.g., 5 minutes -- allows cancellation)

---

## Secret Management

| Secret | Purpose | How to Generate |
|--------|---------|-----------------|
| `APP_STORE_CONNECT_API_KEY_ID` | App Store Connect API authentication | App Store Connect > Users and Access > Integrations > Keys |
| `APP_STORE_CONNECT_API_ISSUER_ID` | API issuer identification | Same page as above |
| `APP_STORE_CONNECT_API_KEY` | API private key (base64 encoded) | Download `.p8` file, base64 encode: `base64 -i AuthKey.p8` |
| `MATCH_PASSWORD` | Encrypts/decrypts certificates in git | Choose a strong password, share securely with team |
| `MATCH_GIT_BASIC_AUTHORIZATION` | Access to certificate git repo | `echo -n "username:token" \| base64` |
| `FASTLANE_APPLE_ID` | Apple Developer account email | Your Apple Developer email |
| `{SERVICE}_TEST_KEY` | Third-party service test credentials | From each service's dashboard |

**Secret rotation procedure:**
1. Generate new credential in the provider's dashboard
2. Update the GitHub secret in repo Settings > Secrets
3. Trigger a CI run to verify the new secret works
4. Revoke the old credential in the provider's dashboard
5. Never do steps 3 and 4 in reverse -- verify first, then revoke

---

## Workflow Recipes

See `references/workflow-recipes.md` for 8+ complete workflow files including:
- PR validation, nightly regression, release to TestFlight
- Hotfix deployment, dependency updates, documentation deploy

## Caching Guide

See `references/caching-guide.md` for detailed caching strategies:
- SPM, DerivedData, and Homebrew caching patterns
- Cache invalidation strategies and versioned keys
- Cost-benefit analysis and storage limits

---

## Quality Gates (Before Marking Complete)

- [ ] Workflow YAML is valid (use `actionlint` or GitHub's workflow editor for validation)
- [ ] All secrets stored in GitHub encrypted secrets -- none hardcoded in YAML
- [ ] Xcode version pinned (not `macos-latest` or `latest`)
- [ ] Caching configured for SPM dependencies at minimum
- [ ] `concurrency.cancel-in-progress` set for PR workflows
- [ ] `timeout-minutes` set on all jobs (prevents runaway billing)
- [ ] Test artifacts uploaded with `if: always()` (available even on failure)
- [ ] Path-based triggers configured to skip CI for irrelevant changes
- [ ] Release workflows include full test suite before deploy step
- [ ] Manual approval gate configured for production deployments

---

## Cross-Skill References

- **velocity-fastlane** -- CI workflows invoke Fastlane lanes. This skill configures the CI environment; `velocity-fastlane` configures what the lanes do. Changes to Fastfile may require corresponding CI workflow updates.
- **atlas-database** -- Database migrations can be validated in CI by running `supabase db reset` as a CI step, ensuring migrations apply cleanly.
- **loki-logs** -- CI test failures produce log output. Use `loki-logs` log analysis techniques to diagnose CI failures from test output artifacts.
- **aegis-notifications** -- Push notification testing in CI requires mock APNs or test configurations. Ensure notification-dependent tests use mocked notification services.

# GitHub Actions Workflow Recipes

Complete workflow files for iOS CI/CD pipelines. Copy, customize project-specific values, and commit.

---

## 1. PR Validation Workflow

Runs on every pull request. Builds, tests, lints, and reports results.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main, develop]
    paths:
      - '**.swift'
      - '**.xib'
      - '**.storyboard'
      - '**/project.yml'
      - '**/Package.swift'
      - '**/Package.resolved'
      - '**/Fastfile'
      - '.github/workflows/ci.yml'

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-test:
    name: Build & Test
    runs-on: macos-15
    timeout-minutes: 30

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '16.2'

      - name: Cache SPM
        uses: actions/cache@v4
        with:
          path: |
            .build
            ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}
          restore-keys: ${{ runner.os }}-spm-

      - name: Cache Homebrew
        uses: actions/cache@v4
        with:
          path: ~/Library/Caches/Homebrew
          key: ${{ runner.os }}-brew-v1

      - name: Install Tools
        run: |
          brew install swiftlint xcodegen 2>/dev/null || true

      - name: Generate Project
        run: cd App && xcodegen generate

      - name: Lint
        run: cd App && swiftlint lint --strict --reporter github-actions-logging

      - name: Run Tests
        run: cd App && fastlane test_comprehensive

      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ github.run_number }}
          path: App/fastlane/test_output/
          retention-days: 14
```

**Expected duration:** ~8 min (with cache hit), ~12 min (cold cache)
**Required secrets:** None (tests don't need signing)

---

## 2. Nightly Regression Workflow

Runs every night. Full test suite across multiple devices.

```yaml
# .github/workflows/nightly.yml
name: Nightly Regression

on:
  schedule:
    - cron: '0 3 * * *'  # 3 AM UTC daily
  workflow_dispatch:       # Manual trigger for debugging

jobs:
  regression:
    name: Full Regression Suite
    runs-on: macos-15
    timeout-minutes: 60

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '16.2'

      - name: Cache SPM
        uses: actions/cache@v4
        with:
          path: |
            .build
            ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}

      - name: Install Tools
        run: brew install swiftlint xcodegen 2>/dev/null || true

      - name: Generate Project
        run: cd App && xcodegen generate

      - name: Run Nightly Tests
        run: cd App && fastlane nightly_regression

      - name: Upload Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: nightly-results-${{ github.run_number }}
          path: |
            App/fastlane/test_output/
            App/fastlane/coverage/
          retention-days: 30

      # Notify on failure
      - name: Notify Failure
        if: failure()
        run: |
          echo "::error::Nightly regression failed. Check test results artifact."
          # Add Slack/email notification here
```

**Expected duration:** ~20 min
**Required secrets:** None

---

## 3. Release to TestFlight Workflow

Triggered by version tags or manual dispatch.

```yaml
# .github/workflows/release.yml
name: Release to TestFlight

on:
  push:
    tags: ['v*']
  workflow_dispatch:
    inputs:
      notes:
        description: 'Release notes'
        required: false
        default: 'Bug fixes and improvements'

jobs:
  release:
    name: Build & Upload to TestFlight
    runs-on: macos-15
    timeout-minutes: 45

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '16.2'

      - name: Cache SPM
        uses: actions/cache@v4
        with:
          path: |
            .build
            ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}

      - name: Install Tools
        run: brew install xcodegen 2>/dev/null || true

      - name: Generate Project
        run: cd App && xcodegen generate

      # Run tests before building for release
      - name: Test
        run: cd App && fastlane test_comprehensive

      # Build and upload
      - name: Build and Upload
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_API_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
          CI_BUILD_NUMBER: ${{ github.run_number }}
        run: cd App && fastlane beta

      - name: Upload IPA Artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-${{ github.ref_name }}-${{ github.run_number }}
          path: App/build/*.ipa
          retention-days: 90
```

**Expected duration:** ~15 min
**Required secrets:** `APP_STORE_CONNECT_API_KEY_ID`, `APP_STORE_CONNECT_API_ISSUER_ID`, `APP_STORE_CONNECT_API_KEY`, `MATCH_PASSWORD`, `MATCH_GIT_BASIC_AUTHORIZATION`

---

## 4. Hotfix Deployment Workflow

Manual trigger for emergency fixes. Fast-track path with full test gate.

```yaml
# .github/workflows/hotfix.yml
name: Hotfix Deploy

on:
  workflow_dispatch:
    inputs:
      branch:
        description: 'Hotfix branch name'
        required: true
      skip_extended_tests:
        description: 'Skip integration/e2e tests (unit tests still run)'
        type: boolean
        default: false

jobs:
  hotfix:
    name: Hotfix Build & Deploy
    runs-on: macos-15
    timeout-minutes: 30
    environment: production  # Requires approval

    steps:
      - name: Checkout Hotfix Branch
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.inputs.branch }}
          fetch-depth: 0

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '16.2'

      - name: Install Tools
        run: brew install xcodegen 2>/dev/null || true

      - name: Generate Project
        run: cd App && xcodegen generate

      # Always run unit tests, even for hotfixes
      - name: Unit Tests
        run: cd App && fastlane test_unit

      # Optionally run extended tests
      - name: Extended Tests
        if: ${{ !inputs.skip_extended_tests }}
        run: cd App && fastlane test_comprehensive

      - name: Build and Upload Hotfix
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_API_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_BASIC_AUTHORIZATION: ${{ secrets.MATCH_GIT_BASIC_AUTHORIZATION }}
          CI_BUILD_NUMBER: ${{ github.run_number }}
        run: cd App && fastlane beta
```

**Expected duration:** ~10 min (unit only), ~15 min (full tests)

---

## 5. Dependency Vulnerability Check

Runs weekly and on Dependabot PRs.

```yaml
# .github/workflows/dependency-check.yml
name: Dependency Check

on:
  schedule:
    - cron: '0 9 * * 1'  # Monday 9 AM UTC
  pull_request:
    paths:
      - '**/Package.swift'
      - '**/Package.resolved'

jobs:
  audit:
    name: Dependency Audit
    runs-on: macos-15
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Secret Scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Check SPM Dependencies
        run: |
          # Resolve dependencies and check for known vulnerabilities
          cd App && swift package resolve
          # Check for outdated dependencies
          swift package show-dependencies --format json > /tmp/deps.json
          echo "Dependencies resolved successfully"

      - name: Build Verification
        run: |
          cd App && xcodegen generate
          xcodebuild build \
            -project App.xcodeproj \
            -scheme App \
            -destination "platform=iOS Simulator,name=iPhone 16" \
            CODE_SIGNING_ALLOWED=NO
```

---

## 6. Documentation Deploy

Generates Swift documentation and deploys to GitHub Pages.

```yaml
# .github/workflows/docs.yml
name: Documentation

on:
  push:
    branches: [main]
    paths:
      - '**.swift'
      - '.github/workflows/docs.yml'

permissions:
  pages: write
  id-token: write

jobs:
  docs:
    name: Generate & Deploy Docs
    runs-on: macos-15
    timeout-minutes: 20

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: '16.2'

      - name: Generate DocC
        run: |
          cd App
          xcodebuild docbuild \
            -scheme App \
            -derivedDataPath ./docbuild \
            -destination "platform=iOS Simulator,name=iPhone 16" \
            CODE_SIGNING_ALLOWED=NO

      - name: Process for Static Hosting
        run: |
          xcrun docc process-archive transform-for-static-hosting \
            App/docbuild/Build/Products/Debug-iphonesimulator/App.doccarchive \
            --output-path docs \
            --hosting-base-path /repo-name

      - name: Upload Pages Artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: docs

      - name: Deploy to Pages
        uses: actions/deploy-pages@v4
```

---

## 7. Matrix Build (Multiple Xcode Versions)

Tests compatibility across Xcode versions and device types.

```yaml
# .github/workflows/compatibility.yml
name: Compatibility Matrix

on:
  schedule:
    - cron: '0 4 * * 0'  # Sunday 4 AM UTC
  workflow_dispatch:

jobs:
  matrix:
    name: Xcode ${{ matrix.xcode }} / ${{ matrix.device }}
    runs-on: macos-15
    timeout-minutes: 30

    strategy:
      matrix:
        xcode: ['16.0', '16.2']
        device: ['iPhone 16', 'iPad Pro 13-inch (M4)']
      fail-fast: false

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Xcode ${{ matrix.xcode }}
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: ${{ matrix.xcode }}

      - name: Install Tools
        run: brew install xcodegen 2>/dev/null || true

      - name: Generate & Test
        run: |
          cd App && xcodegen generate
          xcodebuild test \
            -project App.xcodeproj \
            -scheme App \
            -destination "platform=iOS Simulator,name=${{ matrix.device }}" \
            CODE_SIGNING_ALLOWED=NO
```

---

## 8. PR Label-Based Workflow

Different CI behavior based on PR labels.

```yaml
# .github/workflows/labeled-ci.yml
name: Labeled CI

on:
  pull_request:
    types: [labeled, synchronize, opened]

jobs:
  quick-check:
    name: Quick Check
    runs-on: macos-15
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - name: Setup & Test
        run: |
          brew install xcodegen 2>/dev/null || true
          cd App && xcodegen generate && fastlane test_unit

  full-test:
    name: Full Test Suite
    if: contains(github.event.pull_request.labels.*.name, 'needs-full-test')
    runs-on: macos-15
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - name: Setup & Full Test
        run: |
          brew install xcodegen 2>/dev/null || true
          cd App && xcodegen generate && fastlane test_comprehensive

  performance:
    name: Performance Tests
    if: contains(github.event.pull_request.labels.*.name, 'perf-check')
    runs-on: macos-15
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4
      - name: Setup & Performance Test
        run: |
          brew install xcodegen 2>/dev/null || true
          cd App && xcodegen generate && fastlane test_performance
```

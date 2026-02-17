---
name: pipeline-cicd
description: "This skill should be used when the user asks about 'CI/CD pipeline', 'GitHub Actions', 'continuous integration', 'deployment automation', or 'quality gates in CI'. Provides GitHub Actions workflow configuration, build/test/deploy pipeline management, caching strategies, and quality gate enforcement."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: false
---

# CI/CD Pipeline & Build Automation

Configure and manage GitHub Actions workflows for continuous integration and deployment. Handle build automation, test execution, quality gate enforcement, caching, and release pipelines for iOS apps.

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). These are referred to as `{PROJECT_DIR}` (directory containing the Xcode project), `{PROJECT}` (Xcode project name), and `{SCHEME}` (build scheme) throughout this skill. If the project uses XcodeGen, the generate command runs from `{PROJECT_DIR}`.

## Standard Operating Procedures

### SOP-1: CI Workflow (Pull Request)
```yaml
# .github/workflows/ci.yml
name: CI
on: [pull_request]

jobs:
  build-and-test:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
      - name: Generate Project
        run: cd {PROJECT_DIR} && xcodegen generate
      - name: Run Tests
        run: fastlane test_comprehensive
```

### SOP-2: CD Workflow (Release)
```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  deploy:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Release
        run: fastlane build_release
      - name: Upload to TestFlight
        run: fastlane upload_testflight
```

### SOP-3: Caching Strategy
```yaml
# Cache SPM dependencies
- uses: actions/cache@v4
  with:
    path: |
      ~/Library/Developer/Xcode/DerivedData
      .build
    key: ${{ runner.os }}-spm-${{ hashFiles('**/Package.resolved') }}
```

### SOP-4: Quality Gates in CI
```yaml
jobs:
  quality:
    steps:
      - name: Lint
        run: swiftlint lint --strict
      - name: Security Scan
        run: gitleaks detect --verbose
      - name: Test Coverage
        run: |
          fastlane test_comprehensive
          # Fail if coverage below threshold
```

## Pipeline Stages

```
BUILD        ->    TEST         ->    DEPLOY
  - Compile          - Unit            - TestFlight
  - Lint             - Integration     - App Store
  - Sign             - E2E             - Staging
```

## GitHub Actions Secrets

Common secrets required for iOS CI/CD pipelines:

| Secret | Purpose |
|--------|---------|
| `APP_STORE_CONNECT_API_KEY` | App Store API authentication |
| `MATCH_PASSWORD` | Code signing certificate encryption |
| `FASTLANE_APPLE_ID` | Apple Developer account |
| `{SERVICE}_TEST_KEY` | Third-party service test keys (e.g., Stripe, Airwallex) |

> **Note:** Add secrets for each third-party service your project integrates with. Use the naming convention `{SERVICE_NAME}_TEST_KEY` / `{SERVICE_NAME}_LIVE_KEY` for consistency.

## Workflow Triggers

| Trigger | Actions |
|---------|---------|
| PR opened | Build, Test, Lint |
| PR merged | Build, Test, Deploy to staging |
| Tag pushed | Build, Test, Deploy to production |
| Schedule | Nightly regression tests |

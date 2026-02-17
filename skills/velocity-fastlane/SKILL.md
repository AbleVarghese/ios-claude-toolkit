---
name: velocity-fastlane
description: "This skill should be used when the user asks about 'fastlane', 'TestFlight upload', 'App Store submission', 'code signing', 'match certificates', or 'beta distribution'. Provides Fastlane lane execution, code signing with Match, TestFlight/App Store uploads, and build automation."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: true
---

# Fastlane & Mobile DevOps

Execute Fastlane lanes for testing, building, code signing, and distribution. Manage Match certificates, upload to TestFlight, and submit to the App Store.

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). These are referred to as `{PROJECT}` (Xcode project name), `{SCHEME}` (build scheme), and `{BUNDLE_ID}` (app identifier) throughout this skill.

## Standard Operating Procedures

### SOP-1: Run Test Suite
```bash
# Comprehensive test suite
fastlane test_comprehensive

# Individual categories
fastlane test_unit
fastlane test_integration
fastlane test_e2e
fastlane test_security
fastlane test_performance
```

### SOP-2: Build for Distribution
```bash
# Development build
fastlane build_dev

# Release build
fastlane build_release

# Ad-hoc build
fastlane build_adhoc
```

### SOP-3: Upload to TestFlight
```bash
# Build and upload
fastlane beta

# Upload only (if already built)
fastlane upload_testflight
```

### SOP-4: Code Signing with Match
```bash
# Sync development certificates
fastlane match development

# Sync distribution certificates
fastlane match appstore

# Nuke and recreate (emergency)
fastlane match nuke distribution
fastlane match appstore --force
```

## Available Lanes

| Lane | Purpose | Duration |
|------|---------|----------|
| `test_comprehensive` | Full test suite | ~5 min |
| `test_unit` | Unit tests only | ~1 min |
| `test_integration` | Integration tests | ~2 min |
| `test_e2e` | End-to-end tests | ~3 min |
| `build_dev` | Debug build | ~2 min |
| `build_release` | Release build | ~3 min |
| `beta` | TestFlight upload | ~5 min |
| `release` | App Store release | ~10 min |

## Fastfile Structure

```ruby
# {PROJECT}/fastlane/Fastfile

default_platform(:ios)

platform :ios do
  before_all do
    # Setup environment
    setup_ci if is_ci
  end

  lane :test_comprehensive do
    run_tests(
      project: "{PROJECT}.xcodeproj",
      scheme: "{SCHEME}",
      code_coverage: true,
      result_bundle: true
    )
  end

  lane :beta do
    increment_build_number
    build_app(scheme: "{SCHEME}")
    upload_to_testflight
  end
end
```

## Match Configuration

```ruby
# Matchfile
git_url("<certificates-repo-url>")
storage_mode("git")
type("development") # or "appstore"

app_identifier(["{BUNDLE_ID}"])
username("<apple-developer-email>")
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `No matching profiles` | Missing provisioning | `fastlane match` |
| `Code signing error` | Certificate issue | `fastlane match nuke` |
| `Build failed` | Compilation error | Fix code, retry |
| `Upload failed` | API issue | Check credentials |

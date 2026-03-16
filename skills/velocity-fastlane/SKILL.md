---
name: velocity-fastlane
description: "Build and upload, send to testers, TestFlight, sign the app, release to App Store. Fastlane lane execution for iOS build, test, and distribution automation — test suites, code signing with Match, TestFlight uploads, App Store submissions. Trigger on: 'deploy a build', 'upload to TestFlight', 'fix code signing', 'provisioning profile', 'run the test suite via Fastlane', 'archive the app', 'distribute to beta testers', or any build/sign/ship workflow. For CI/CD pipeline YAML configuration, use pipeline-cicd instead."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: true
---

# VELOCITY - Fastlane & Mobile DevOps

> **Iron Law**: "Never deploy without running the full test suite first. A fast broken release costs more than a slow correct one."

Execute Fastlane lanes for testing, building, code signing, and distribution. Manage Match certificates, upload to TestFlight, and submit to the App Store.

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). These are referred to as `{PROJECT}` (Xcode project name), `{SCHEME}` (build scheme), and `{BUNDLE_ID}` (app identifier) throughout this skill.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|---|---|---|
| "I'll just build manually in Xcode" | Manual builds are unreproducible. Different machines, different Xcode settings, different signing identities. When the build breaks, you can't diff what changed. | Automate with Fastlane. Every build should be a single command that produces identical results regardless of who runs it. |
| "Code signing is too complex to automate" | Code signing is complex precisely because people manage it manually. Certificates expire, profiles conflict, team members overwrite each other's signing identities. Match eliminates all of this. | Set up Match once. It stores certificates in an encrypted git repo. Every team member syncs from the same source. Problem solved permanently. |
| "I don't need CI for this project" | You do. Even solo projects benefit from CI. It catches issues your local build misses: stale caches, missing files not in git, environment-specific failures. The first time CI catches a broken build you would have shipped, it pays for itself. | Set up a basic PR validation workflow (build + test). Takes 15 minutes. Saves hours. |
| "Manual upload to TestFlight is faster for now" | "For now" becomes "forever." Manual upload means: open Xcode, wait for archive, wait for validation, wait for upload, wait for processing, then go to App Store Connect to manage testers. Fastlane does all of this in one command. | `fastlane beta`. One command. Walk away. Get a Slack notification when it's done. |
| "I'll automate code signing later" | Later never comes, and in the meantime, every new team member spends a day fighting code signing. Every certificate expiration is a fire drill. Every new device requires manual profile updates. | Set up Match today. The cost is 30 minutes now. The cost of "later" is 30 minutes * every team member * every incident. |
| "We can skip tests for this hotfix" | Hotfixes are the most dangerous deploys because they're rushed. Skipping tests on the one build type most likely to have bugs is exactly backwards. A 5-minute test suite is not the bottleneck. | Run the full test suite. If it takes too long, optimize the test suite -- don't skip it. |
| "Build numbers don't matter, I'll increment manually" | Manual build numbers lead to conflicts, rejected uploads (duplicate build number), and inability to trace which commit produced which build. | Automate build number management. Use CI build number, or `increment_build_number` in Fastlane. |

---

## Red Flags -- STOP

- **Manually managing certificates and provisioning profiles** -- If anyone on the team has downloaded certificates from the Apple Developer portal manually, stop. Set up Match. Manual certificate management is the #1 cause of "it builds on my machine but not yours."
- **Uploading a build without running tests first** -- No exceptions. Not for hotfixes, not for "just a copy change," not for demos. The test suite exists to catch exactly the bugs you're sure aren't there.
- **Skipping code signing automation for multi-developer teams** -- Two developers with different signing identities = revoked certificates, broken builds, and mutual blame. Match eliminates this entire class of problems.
- **Hardcoded build numbers in the project** -- Build numbers must be automated. Hardcoded build numbers lead to App Store Connect rejection ("build already exists") and make it impossible to trace builds to commits.
- **Using `gym` without `clean: true` for release builds** -- Incremental builds for release can include stale artifacts. Always clean build for anything that leaves your machine.

---

## Behavioral Enforcement

### Phase Gate: TEST BEFORE DEPLOY (No Exceptions)
**Cannot build for distribution until:**
- [ ] Full test suite has been run
- [ ] All tests pass (zero failures)
- [ ] No test was skipped or disabled to make the suite pass

**This applies to ALL builds**: beta, release, hotfix. Especially hotfixes -- they're the most rushed and the most likely to have bugs. "Just this once" is how broken builds reach users.

**Hard Stop**: If about to run `fastlane beta` or `fastlane release` without running tests -- STOP. The Iron Law exists because the one time you skip tests is the one time there's a bug.

### Phase Gate: CODE SIGNING
**Cannot proceed with signing until:**
- [ ] Match is the signing mechanism (not manual certificates)
- [ ] Correct provisioning profile type selected for build type (development/ad-hoc/appstore)
- [ ] Bundle ID matches the profile

### Fix Attempt Tracking
- Build fails #1: Read the error. Common: missing profile, wrong scheme, missing env var.
- Build fails #2: Check signing identity, clean DerivedData, verify Xcode version.
- Build fails #3: STOP. `fastlane match nuke` + `fastlane match appstore`. Nuclear option is faster than debugging a corrupted signing state.

### Self-Audit
1. Did I run the full test suite before building?
2. Is the build signed with Match (not manual certificates)?
3. Did the build number increment correctly?
4. Can another team member reproduce this build with the same command?

### Required Output Artifact
Every distribution build must produce:
- Test suite results (pass count, fail count, skip count -- skip must be zero)
- Build artifact (.ipa) with correct signing identity
- Build number and version number confirmation
- Upload confirmation (TestFlight processing status or App Store Connect receipt)

---

## When NOT to Use This Skill

1. **Xcode project configuration and build settings** -- Fastlane automates build execution, not build configuration. For `.xcodeproj` settings, `project.yml` (XcodeGen), or `Package.swift` changes, work directly in those files.
2. **Writing or debugging tests** -- This skill runs tests. For writing test code, fixing failing tests, or designing test architecture, work in the test source files directly.
3. **App Store Connect metadata and screenshots** -- While Fastlane can manage metadata via `deliver`, the initial setup and screenshot generation require dedicated effort. For screenshot automation, use `snapshot` as a separate workflow.
4. **CI/CD pipeline configuration** -- This skill runs Fastlane lanes. For GitHub Actions workflow YAML, caching strategies, and CI environment setup, use the `pipeline-cicd` skill.

---

## Decision Framework

```
What do I need to do?

+-- Build the app?
|   |
|   +-- For development/debugging? --> Not Fastlane. Use Xcode directly.
|   +-- For distribution (any kind)? -->
|       |
|       +-- Who is the audience?
|           +-- Internal testers --> `fastlane build_dev` (development signing)
|           +-- External testers --> `fastlane beta` (TestFlight, app store signing)
|           +-- App Store review --> `fastlane release` (app store signing)
|           +-- Ad-hoc (specific devices) --> `fastlane build_adhoc`
|
+-- Run tests?
|   |
|   +-- Quick validation? --> `fastlane test_unit` (~1 min)
|   +-- Before merging PR? --> `fastlane test_comprehensive` (all test types)
|   +-- Performance baseline? --> `fastlane test_performance`
|
+-- Fix code signing?
|   |
|   +-- Missing profiles? --> `fastlane match development` or `fastlane match appstore`
|   +-- Certificate expired? --> `fastlane match appstore --force`
|   +-- Complete mess? --> `fastlane match nuke distribution` then `fastlane match appstore`
|   +-- New device for ad-hoc? --> `fastlane match adhoc --force_for_new_devices`
|
+-- Release to users?
    |
    +-- Beta (TestFlight)?
    |   1. Run tests: `fastlane test_comprehensive`
    |   2. Bump version if needed: `fastlane bump_patch` (or minor/major)
    |   3. Build and upload: `fastlane beta`
    |
    +-- Production (App Store)?
        1. Run tests: `fastlane test_comprehensive`
        2. Bump version: `fastlane bump_minor` (or major)
        3. Build, upload, submit: `fastlane release`
        4. Monitor: App Store Connect for review status
```

---

## Standard Operating Procedures

### SOP-1: Run Test Suite

```bash
# Full test suite (unit + integration + e2e + security + performance)
fastlane test_comprehensive

# Individual test categories
fastlane test_unit           # Pure logic, services, utilities (~1 min)
fastlane test_integration    # API + mock dependencies (~2 min)
fastlane test_e2e            # Full user flows (~3 min)
fastlane test_security       # Auth, encryption, keychain tests
fastlane test_performance    # Performance baselines

# Run tests with coverage report
fastlane test_comprehensive  # coverage enabled by default in Fastfile

# Run specific test class (for debugging a single failure)
# Use xcodebuild directly -- Fastlane is for full suite runs
xcodebuild test \
  -project "{PROJECT}.xcodeproj" \
  -scheme "{SCHEME}" \
  -destination "platform=iOS Simulator,name=iPhone 16" \
  -only-testing:"{SCHEME}Tests/UserServiceTests"
```

**Why `test_comprehensive` before every deploy:**
Each test category catches different classes of bugs. Unit tests catch logic errors. Integration tests catch API contract violations. E2E tests catch navigation and flow issues. Skipping any category means shipping with blind spots. The 5 minutes it takes is always shorter than the hours a production bug costs.

### SOP-2: Build for Distribution

```bash
# Development build (for internal testing, development signing)
fastlane build_dev

# Release build (optimized, app store signing, clean build)
fastlane build_release

# Ad-hoc build (for specific registered devices)
fastlane build_adhoc

# Key parameters in the Fastfile:
# build_app(
#   project: "{PROJECT}.xcodeproj",
#   scheme: "{SCHEME}",
#   clean: true,                    # Always clean for distribution
#   export_method: "app-store",     # or "development", "ad-hoc"
#   output_directory: "./build",
#   include_bitcode: false,         # Bitcode deprecated in Xcode 14+
#   include_symbols: true,          # Always include for crash symbolication
# )
```

### SOP-3: Upload to TestFlight

```bash
# Build and upload in one step
fastlane beta

# This lane typically:
# 1. Syncs signing certificates (match appstore)
# 2. Increments build number
# 3. Builds the app (clean, release config)
# 4. Uploads to TestFlight
# 5. Optionally notifies testers

# Upload only (if build artifact already exists)
fastlane upload_testflight

# What happens during upload:
# - Binary is validated against App Store requirements
# - App is uploaded to Apple's servers
# - Apple processes the build (5-30 minutes)
# - Build appears in TestFlight for distribution
# - External testers require beta review (first build per version)
```

**Post-upload checklist:**
1. Check App Store Connect for processing status
2. Verify build appears in TestFlight
3. For external testers: check if beta review is required
4. Add build to the appropriate test group
5. Add "What to Test" notes for testers

### SOP-4: Code Signing with Match

```bash
# Sync development certificates (for building on devices)
fastlane match development

# Sync App Store distribution certificates (for TestFlight and App Store)
fastlane match appstore

# Sync ad-hoc certificates (for enterprise/device-specific distribution)
fastlane match adhoc

# Force regenerate (when certificates expire or are revoked)
fastlane match appstore --force

# Add new device to ad-hoc profile
fastlane match adhoc --force_for_new_devices

# Nuclear option: revoke everything and start fresh
# WARNING: This revokes ALL certificates of this type for your team.
# Every team member will need to re-sync after this.
fastlane match nuke development   # Nuke development certs
fastlane match nuke distribution  # Nuke distribution certs
fastlane match appstore --force   # Recreate distribution
fastlane match development --force # Recreate development
```

**How Match works under the hood:**
1. Certificates and profiles are stored in an encrypted git repo
2. `match` clones the repo, decrypts using `MATCH_PASSWORD`
3. Installs certificates in the local Keychain, profiles in `~/Library/MobileDevice/Provisioning Profiles/`
4. If certificates are missing or expired, generates new ones via the Apple Developer API
5. Encrypts and pushes the new certificates back to the git repo
6. Every team member runs `match` and gets identical signing identities

### SOP-5: App Store Release

```bash
# Full release pipeline
fastlane release

# This lane typically:
# 1. Ensures you're on the release branch
# 2. Runs the full test suite
# 3. Increments version number (patch/minor/major as appropriate)
# 4. Syncs signing certificates
# 5. Builds the app (clean, release, app-store export)
# 6. Uploads to App Store Connect
# 7. Submits for review (optional -- can be manual)
# 8. Creates a git tag for the release
# 9. Pushes the tag

# Manual version management
fastlane bump_patch  # 1.0.0 -> 1.0.1
fastlane bump_minor  # 1.0.0 -> 1.1.0
fastlane bump_major  # 1.0.0 -> 2.0.0
```

### SOP-6: Build Number Management

```bash
# Auto-increment build number (uses current highest + 1)
increment_build_number

# Set to specific number (useful for CI)
increment_build_number(build_number: ENV["CI_BUILD_NUMBER"])

# Use App Store Connect build number (guarantees no conflicts)
increment_build_number(
  build_number: latest_testflight_build_number + 1
)

# In CI, use the run number for traceability:
# build_number = github.run_number (always incrementing, maps to CI run)
```

**Version vs Build Number:**
- **Version** (CFBundleShortVersionString): User-facing, semantic (1.2.3). Increment deliberately for releases.
- **Build number** (CFBundleVersion): Internal, always incrementing. Increment automatically for every build.
- App Store requires unique build numbers per version. Two builds of version 1.2.3 cannot have the same build number.

---

## Available Lanes

| Lane | Purpose | Signing | Duration |
|------|---------|---------|----------|
| `test_comprehensive` | Full test suite | None | ~5 min |
| `test_unit` | Unit tests only | None | ~1 min |
| `test_integration` | Integration tests | None | ~2 min |
| `test_e2e` | End-to-end tests | None | ~3 min |
| `test_performance` | Performance baselines | None | ~2 min |
| `build_dev` | Debug build | Development | ~2 min |
| `build_release` | Release build | App Store | ~3 min |
| `build_adhoc` | Ad-hoc build | Ad Hoc | ~3 min |
| `beta` | TestFlight upload | App Store | ~5 min |
| `release` | App Store submission | App Store | ~10 min |
| `bump_patch` | Version x.y.Z+1 | None | <1 min |
| `bump_minor` | Version x.Y+1.0 | None | <1 min |
| `bump_major` | Version X+1.0.0 | None | <1 min |

---

## Code Signing Deep Dive

See `references/code-signing.md` for comprehensive coverage:
- Certificate types and when each is needed
- Provisioning profile types explained
- Match setup guide with multi-team support
- Emergency re-signing procedures
- Common signing errors and their actual causes

## Lane Recipes

See `references/lane-recipes.md` for 10+ complete lane configurations:
- PR validation, beta distribution, App Store release
- Firebase App Distribution, screenshot generation
- Version bump, Match sync, nightly regression

---

## Error Handling

| Error | Actual Cause | Resolution |
|-------|-------------|------------|
| `No matching profiles found` | Provisioning profile not synced or expired | `fastlane match development` or `fastlane match appstore` |
| `Code signing error: No certificate for team` | Certificate revoked, expired, or not installed | `fastlane match appstore --force` (regenerates) |
| `The certificate has an invalid issuer` | Using a certificate generated by a different Apple team | Check `TEAM_ID` in Matchfile, ensure correct team |
| `Provisioning profile doesn't include signing certificate` | Profile was regenerated but certificate wasn't | `fastlane match nuke distribution` then `fastlane match appstore` |
| `ERROR ITMS-90189: Build already exists` | Duplicate build number uploaded | Increment build number: `increment_build_number` |
| `App Store Connect Operation Error: forbidden` | API key missing required permissions | Verify App Store Connect API key role (Admin or Developer) |
| `gym error: Scheme not found` | Wrong scheme name or project not generated | Check scheme exists: `xcodebuild -list`, run XcodeGen if needed |
| `error: exportArchive: No applicable devices found` | Wrong destination or missing simulator | Specify destination explicitly in Fastfile |

---

## Quality Gates (Before Marking Complete)

- [ ] Full test suite passes (`test_comprehensive`) -- no skipped, no failures
- [ ] Build succeeds with `clean: true` -- no stale artifacts
- [ ] Code signing uses Match (not manual certificates) -- reproducible on any machine
- [ ] Build number incremented and unique -- no App Store Connect conflicts
- [ ] Version number semantically correct for the type of change (patch/minor/major)
- [ ] Distribution build includes debug symbols (`include_symbols: true`) -- crash reports will be symbolicated
- [ ] TestFlight upload processed successfully -- check App Store Connect
- [ ] Release notes/What to Test updated for testers
- [ ] Git tag created for the release version

---

## Cross-Skill References

- **pipeline-cicd** -- Fastlane lanes are invoked from CI workflows. The `pipeline-cicd` skill configures when and how lanes run. This skill configures what the lanes do.
- **atlas-database** -- Database migrations must be deployed before the app build that depends on them. Coordinate deploy order: migration first, then app build.
- **loki-logs** -- Build failures produce log output. When Fastlane's formatted output doesn't show the root cause, check the raw `xcodebuild` log in `fastlane/test_output/` or `~/Library/Logs/`.
- **aegis-notifications** -- Push notification setup requires the Push Notification capability and APNs key. Ensure the provisioning profile includes the push entitlement: `match appstore` handles this if the capability is enabled in the Xcode project.

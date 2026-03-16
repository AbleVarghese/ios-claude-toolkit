# Lane Recipes

Complete Fastlane lane configurations for common iOS workflows.

---

## 1. PR Validation Lane

Runs on every pull request to catch issues before merge.

```ruby
lane :validate_pr do
  # Sync code signing (readonly -- don't regenerate in CI)
  match(type: "development", readonly: true)

  # Generate project (if using XcodeGen)
  sh("cd ../#{PROJECT_DIR} && xcodegen generate")

  # Lint
  swiftlint(
    mode: :lint,
    strict: true,
    reporter: "github-actions-logging"
  )

  # Build and test
  run_tests(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: SCHEME,
    devices: ["iPhone 16"],
    code_coverage: true,
    result_bundle: true,
    output_directory: "./fastlane/test_output"
  )
end
```

**Required env vars:** `MATCH_PASSWORD`, `MATCH_GIT_BASIC_AUTHORIZATION`
**Duration:** ~5 min

---

## 2. Beta Distribution Lane (TestFlight)

Builds, signs, and uploads to TestFlight.

```ruby
lane :beta do
  # Ensure clean working directory
  ensure_git_status_clean

  # Sync App Store signing
  match(type: "appstore", readonly: is_ci)

  # Generate project
  sh("cd ../#{PROJECT_DIR} && xcodegen generate")

  # Run full test suite
  run_tests(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: SCHEME,
    devices: ["iPhone 16"],
    code_coverage: true
  )

  # Increment build number
  increment_build_number(
    build_number: is_ci ? ENV["CI_BUILD_NUMBER"] : (latest_testflight_build_number + 1),
    xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj"
  )

  # Build
  build_app(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: SCHEME,
    clean: true,
    export_method: "app-store",
    output_directory: "./build",
    include_symbols: true
  )

  # Upload to TestFlight
  upload_to_testflight(
    skip_waiting_for_build_processing: true,
    api_key: app_store_connect_api_key
  )

  # Tag the release
  add_git_tag(tag: "beta/#{get_version_number}-#{get_build_number}")

  # Notify (optional)
  # slack(message: "New beta uploaded: #{get_version_number} (#{get_build_number})")
end
```

**Required env vars:** `APP_STORE_CONNECT_API_KEY_ID`, `APP_STORE_CONNECT_API_ISSUER_ID`, `APP_STORE_CONNECT_API_KEY`, `MATCH_PASSWORD`, `MATCH_GIT_BASIC_AUTHORIZATION`, `CI_BUILD_NUMBER` (in CI)
**Duration:** ~8 min

---

## 3. App Store Release Lane

Full release pipeline: test, build, upload, and optionally submit for review.

```ruby
lane :release do
  # Ensure on main/release branch
  ensure_git_branch(branch: '^(main|release/.*)$')
  ensure_git_status_clean

  # Sync signing
  match(type: "appstore", readonly: is_ci)

  # Generate project
  sh("cd ../#{PROJECT_DIR} && xcodegen generate")

  # Full test suite
  run_tests(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: SCHEME,
    devices: ["iPhone 16", "iPad Pro 13-inch (M4)"],
    code_coverage: true
  )

  # Build
  build_app(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: SCHEME,
    clean: true,
    export_method: "app-store",
    output_directory: "./build",
    include_symbols: true
  )

  # Upload to App Store Connect
  upload_to_app_store(
    api_key: app_store_connect_api_key,
    submit_for_review: false,  # Set true to auto-submit
    automatic_release: false,
    force: true  # Skip HTML preview verification
  )

  # Tag release
  version = get_version_number(xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj")
  build = get_build_number(xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj")
  add_git_tag(tag: "v#{version}-#{build}")
  push_git_tags
end
```

**Duration:** ~12 min

---

## 4. Firebase App Distribution Lane

For rapid iteration with internal testers (faster than TestFlight processing).

```ruby
lane :distribute_firebase do
  match(type: "adhoc", readonly: is_ci)

  sh("cd ../#{PROJECT_DIR} && xcodegen generate")

  increment_build_number(
    build_number: is_ci ? ENV["CI_BUILD_NUMBER"] : "#{Time.now.to_i}",
    xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj"
  )

  build_app(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: SCHEME,
    clean: true,
    export_method: "ad-hoc",
    output_directory: "./build"
  )

  firebase_app_distribution(
    app: ENV["FIREBASE_APP_ID"],
    groups: "internal-testers",
    release_notes: changelog_from_git_commits(
      commits_count: 10,
      pretty: "- %s"
    )
  )
end
```

**Required env vars:** `FIREBASE_APP_ID`, `FIREBASE_TOKEN` (or Google service account)
**Required plugin:** `fastlane-plugin-firebase_app_distribution`
**Duration:** ~5 min

---

## 5. Nightly Regression Lane

Scheduled full test suite with extended test categories.

```ruby
lane :nightly_regression do
  sh("cd ../#{PROJECT_DIR} && xcodegen generate")

  # Run ALL test plans including performance and stress tests
  run_tests(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: SCHEME,
    devices: ["iPhone 16", "iPhone SE (3rd generation)", "iPad Pro 13-inch (M4)"],
    code_coverage: true,
    result_bundle: true,
    output_directory: "./fastlane/test_output",
    testplan: "NightlyRegression"  # Separate test plan with all tests
  )

  # Generate coverage report
  xcov(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: SCHEME,
    output_directory: "./fastlane/coverage",
    minimum_coverage_percentage: 80.0
  )
end
```

**Duration:** ~15 min

---

## 6. Match Sync Lane

Sync all certificate types in one command.

```ruby
lane :sync_certificates do
  match(type: "development")
  match(type: "appstore")
  # match(type: "adhoc")  # Uncomment if using ad-hoc distribution
end

lane :sync_certificates_readonly do
  match(type: "development", readonly: true)
  match(type: "appstore", readonly: true)
end
```

**Duration:** <1 min

---

## 7. Version Bump Lanes

Semantic versioning management.

```ruby
lane :bump_patch do
  increment_version_number(
    bump_type: "patch",
    xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj"
  )
  commit_version_bump(xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj")
end

lane :bump_minor do
  increment_version_number(
    bump_type: "minor",
    xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj"
  )
  commit_version_bump(xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj")
end

lane :bump_major do
  increment_version_number(
    bump_type: "major",
    xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj"
  )
  commit_version_bump(xcodeproj: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj")
end
```

---

## 8. Screenshot Generation Lane

Automated screenshot capture for App Store listing.

```ruby
lane :screenshots do
  capture_screenshots(
    project: "#{PROJECT_DIR}/#{PROJECT}.xcodeproj",
    scheme: "#{SCHEME}UITests",
    devices: [
      "iPhone 16 Pro Max",
      "iPhone SE (3rd generation)",
      "iPad Pro 13-inch (M4)"
    ],
    languages: ["en-US"],
    output_directory: "./fastlane/screenshots",
    clear_previous_screenshots: true
  )

  # Optional: frame screenshots with device bezels
  frame_screenshots(
    path: "./fastlane/screenshots",
    white: true
  )
end
```

**Required:** UITest target with snapshot test methods using `snapshot("ScreenName")`
**Duration:** ~10 min

---

## Helper: App Store Connect API Key

Used by multiple lanes. Define once.

```ruby
# In Fastfile, define a helper method:
def app_store_connect_api_key
  app_store_connect_api_key(
    key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
    issuer_id: ENV["APP_STORE_CONNECT_API_ISSUER_ID"],
    key_content: ENV["APP_STORE_CONNECT_API_KEY"],
    is_key_content_base64: true,
    in_house: false
  )
end
```

---

## Helper: Project Constants

Define project-specific constants at the top of the Fastfile.

```ruby
default_platform(:ios)

# Project constants -- update these for your project
PROJECT_DIR = "App"           # Directory containing .xcodeproj
PROJECT = "MyApp"             # Xcode project name
SCHEME = "MyApp"              # Build scheme
BUNDLE_ID = "com.myapp.ios"   # Bundle identifier

platform :ios do
  before_all do
    setup_ci if is_ci
  end

  error do |lane, exception|
    # Notification on failure (Slack, email, etc.)
    # slack(
    #   message: "Lane #{lane} failed: #{exception.message}",
    #   success: false
    # )
  end

  # ... lanes ...
end
```

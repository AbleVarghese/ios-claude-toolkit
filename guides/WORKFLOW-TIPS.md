# Lessons from 75 Claude Code Sessions Building a Production iOS App

Practical tips learned the hard way while building a production iOS app with Claude Code across 75+ sessions. Each tip is something that cost real debugging time to discover.

---

## Session Management

**Name your sessions for easy retrieval.**
Use `/rename` after starting a meaningful session. When you have dozens of sessions, descriptive names like "fix-payment-flow" or "kyc-integration" save you from scrolling through timestamps.

**Know the resume shortcuts.**
Use `claude --continue` to pick up exactly where your last session left off. Use `claude --resume` for a searchable picker when you need a specific older session.

**Prevent session auto-deletion.**
Set `cleanupPeriodDays` to a high value so sessions never auto-delete. You will want to reference previous sessions for context, decisions made, and debugging history. Losing that history means re-explaining your entire architecture.

**Use `--dangerously-skip-permissions` for trusted solo projects.**
Create a shell alias (e.g., `dm`) to launch Claude Code without permission prompts. This dramatically speeds up sessions where Claude needs to read, write, and execute dozens of commands. Only use this on projects where you trust the operations being performed.

---

## Build and Test Efficiency

**Never run the full test suite unless explicitly needed.**
Use targeted `-only-testing:YourTests/ClassName` flags scoped to the files you changed. A large test suite can take 15+ minutes. Targeted runs take 5-30 seconds. Map changed source files to their test files and run only those.

**Always prefer an already-booted simulator.**
Check `xcrun simctl list devices | grep Booted` before launching a new one. Cold-booting a simulator adds 10-30 seconds every time. Prefer lightweight devices (iPhone SE) over Pro Max variants.

**Use `SWIFT_USE_INTEGRATED_DRIVER=NO` for test runs.**
Without this flag, XCBBuildService crashes intermittently during test execution. Add it to every `xcodebuild test` invocation. This is a known Xcode issue that has persisted across multiple versions.

**Never run two xcodebuild test commands in parallel targeting the same DerivedData.**
This causes a "database is locked" SQLite error that fails both runs. If you need parallel test execution, use separate DerivedData paths or run sequentially.

**Clean DerivedData when builds behave strangely.**
Stale build artifacts cause phantom errors. When you see failures that do not match the code, nuke DerivedData with `rm -rf ~/Library/Developer/Xcode/DerivedData/YourProject-*` and rebuild. Document this command in CLAUDE.md so Claude does it automatically.

**Use XcodeGen and never edit `.xcodeproj` directly.**
XcodeGen regenerates the project file from a declarative YAML config. This eliminates merge conflicts in `.pbxproj`, makes file additions automatic, and gives Claude a single file to modify. Run `xcodegen generate` after every `project.yml` change.

---

## Architecture

**Domain layer must have zero external imports.**
If you see `import Supabase` or `import Stripe` in your Domain directory, that is a boundary violation. Domain defines ports (protocols); infrastructure implements them. Claude Code can help enforce this if you document the rule in CLAUDE.md.

**DI container should use `NSRecursiveLock`, not `NSLock`.**
Factories registered in your DI container often call `resolve()` internally to get their own dependencies. With `NSLock`, this deadlocks silently (black screen on launch, no crash log). `NSRecursiveLock` allows re-entrant locking from the same thread.

**Use `registerInstance` for singletons shared with SwiftUI.**
If your auth service is both a `@StateObject` in your app root and resolved via DI elsewhere, `registerSingleton` creates a second instance. The SwiftUI view sees one instance, your services see another, and login state is never shared. `registerInstance` registers the exact object you pass in.

**Watch for DI registration order overwriting.**
If `configureCore()` runs after `configureInfrastructure()` and both register the same protocol, the later registration silently wins. You end up with the wrong implementation at runtime with no compiler warning. Map your registration order explicitly.

**All ViewModels should be `@MainActor`.**
SwiftUI views run on the main thread. If your ViewModel publishes state changes from a background thread, you get purple runtime warnings or subtle UI glitches. Mark every ViewModel with `@MainActor` and use `async/await` for background work.

---

## Edge Functions and Backend

**Unwrap the middleware envelope in Swift.**
If your edge functions use middleware, responses are wrapped in `{ success, data, correlationId }`. Your Swift decode logic must unwrap this envelope first, then decode the inner `data` field. Decoding the raw response body directly gives cryptic "data couldn't be read" errors.

**Deploy edge functions via CLI, not MCP tools.**
MCP deployment tools can break relative import paths like `../_shared/` by placing the entrypoint inside a subdirectory. The CLI operates on your actual file system and resolves relative paths correctly. Save yourself the debugging.

**Custom error classes must extend your base error class.**
If your middleware catches errors with `instanceof BaseError`, any custom error class that does not extend it becomes a generic 500 with no useful message. This is invisible until you check edge function logs.

**Group edge function deployments by auth setting.**
If some functions require JWT verification and others do not, never deploy them in the same batch with `--no-verify-jwt`. That flag applies to the entire batch and will disable auth on functions that need it.

---

## Git Workflow

**Pre-commit hooks are your first line of defense.**
Configure hooks to catch leaked secrets, lint violations, and merge conflict markers before they reach the repository. The cost of a leaked API key far exceeds the cost of a 2-second hook run.

**The `+` character is not allowed in conventional commit scope.**
A commit message like `fix(P0+P1): resolve bugs` will be rejected by strict conventional commit validators. Use hyphens or underscores instead.

**Update git hooks when you delete referenced files.**
If your pre-push hook checks for the existence of `PROJECT.md` and you delete or move that file, the hook fails on every push. Check your hooks whenever you restructure the project.

**Use feature branches for parallel Claude Code Web sessions.**
If you work on `main` in your IDE while Claude Code Web works in parallel, have Claude create a `claude/<feature-name>` branch. Merge when done. This prevents conflicts and lets both sessions make progress simultaneously.

---

## Testing

**`tearDown()` runs even when `setUp()` throws `XCTSkip`.**
If your `setUp()` determines the test should be skipped (no network, missing credentials), `tearDown()` still executes. Any implicitly unwrapped optional accessed in `tearDown()` will crash. Guard with `if service != nil { ... }` or optional chaining.

**`XCTFail()` plus `return` does NOT skip the test.**
The test body continues executing after `XCTFail()`. If you need to actually stop execution, use `throw XCTSkip("reason")`. This is a common source of confusing test results where a "failed" test still runs assertions below the failure point.

**Archived test files with `#if true` will still compile and run.**
If you move tests to a disabled/archive directory but leave `#if true` guards, XcodeGen (or any build system that includes all Swift files) will compile and execute them. Always use `#if false` for truly disabled code.

**Use `throw XCTSkip` for backend-dependent tests without auth.**
Checking for a Supabase URL is not enough to determine if backend tests can run -- the URL may exist in config even without valid auth. Check `client.auth.session` instead, which correctly throws when no user is logged in.

**Test one thing at a time when debugging failures.**
When you have 126 test failures, do not try to fix them all at once. Group by root cause, fix one category, re-run that subset, then move to the next. Waves of 20-30 fixes are manageable; 126 simultaneous changes are not.

---

## Concurrency (Swift 6)

**Adding `Sendable` to port protocols cascades to all implementors.**
When you mark a protocol as `Sendable`, every concrete type conforming to it must also be `Sendable`. Production adapters using `os.Logger` or other non-Sendable system types need `@unchecked Sendable`. Plan for this cascade before starting.

**Capture `let` bindings before `@Sendable` closures.**
A `var` captured in a `@Sendable` closure triggers a compiler error in strict concurrency mode. Create a `let` copy of the variable before the closure: `let headers = headerBuilder` then use `headers` inside the closure.

**`[String: Any]` fields need `@unchecked Sendable`.**
Dictionary types with `Any` values cannot automatically conform to `Sendable`. If your response structs contain `[String: Any]` metadata fields, the enclosing type needs `@unchecked Sendable` conformance.

**Swift nested structs inside generic functions are illegal.**
If you need a helper type like `EdgeFunctionEnvelope<T>` to decode a generic response, define it at file scope. Declaring it inside a generic function produces a compiler error that does not clearly explain the restriction.

---

## General

**Let Claude discover project values from CLAUDE.md.**
Do not hardcode simulator IDs, project paths, or API endpoints in skills or agent definitions. Put them in CLAUDE.md where they can be updated in one place. Skills should teach methodology; CLAUDE.md should teach project specifics.

**Use plan mode for complex tasks.**
Toggle with `Shift+Tab` before diving into multi-file changes. Exploring the problem space first prevents wasted implementation effort. This is especially valuable for refactoring and architectural changes.

**Skills teach methodology, CLAUDE.md teaches project specifics.**
A skill like "run tests" should describe the general approach (identify changed files, map to test classes, run targeted). CLAUDE.md should contain the actual test command, simulator ID, and project-specific flags. This separation means skills are reusable across projects.

**Track what Claude learns in a memory file.**
Maintain a lessons-learned section in your project memory. Every time Claude hits an obscure issue and solves it, record the solution. This prevents re-discovering the same fix in future sessions.

**Report progress as percentages during long operations.**
Tell Claude to report progress at every meaningful step during multi-file changes, test runs, or deployments. Silence during a 17-minute test run erodes trust. A simple "Wave 3/5 complete -- 60%" keeps you informed and engaged.

**Document your secrets configuration chain.**
Spell out exactly how secrets flow: `Secrets.xcconfig (gitignored) -> Debug.xcconfig -> Info.plist -> Bundle.main`. When Claude knows this chain, it never accidentally hardcodes a key or creates a duplicate config path.

**Run a pre-existence check before creating anything.**
Before Claude creates a new file, utility, or service, have it search the codebase for existing implementations. Duplicated code is the most common form of technical debt introduced during AI-assisted development. A 30-second search prevents hours of future cleanup.

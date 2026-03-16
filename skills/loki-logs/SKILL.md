---
name: loki-logs
description: "Check the logs, what happened, debug this, app crashed, nothing happens when I tap. iOS device and simulator log streaming, searching, filtering, and pattern analysis. Trigger on: 'check the console', 'what's in the logs', 'why isn't this working', 'app did something weird', 'trace the execution', 'diagnose this', 'read the crash log', or when debugging runtime behavior on device or simulator. Also use when correlating log output across components. For Sentry production crashes, use sentry-find-bugs instead."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: false
---

# LOKI - Log Intelligence & Analytics

> **Iron Law**: "Read the actual error message and surrounding context before hypothesizing. Logs don't lie -- your assumptions do."

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). Replace `{BUNDLE_ID}` with the app's bundle identifier and `{PROCESS_NAME}` with the app's process/executable name.

Stream, search, filter, and analyze iOS device and simulator logs. Detect error patterns, correlate traces across components, and extract actionable information from log output.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|---|---|---|
| "I'll just add a print() statement" | `print()` has no log levels, no subsystem filtering, no privacy controls, and is stripped in release builds (or worse, leaks data if not stripped). It's invisible to Console.app and Instruments. | Use `os.Logger` with proper subsystem/category. It persists, it filters, it's production-safe. |
| "The error message is obvious, I don't need more context" | Error messages describe symptoms, not causes. "Connection refused" could be DNS, firewall, wrong port, server down, or VPN. Context around the error tells you which. | Read 10-20 lines before and after the error. Look for the sequence of events that led to the failure. |
| "I don't need structured logging for a small app" | Small apps grow. When they do, you'll wish you had subsystem/category from day one. Retrofitting structured logging into an existing codebase is painful and error-prone. | Set up `os.Logger` with subsystem = bundle ID, categories = functional areas (networking, auth, storage, UI). Do it once, benefit forever. |
| "Console.app is good enough" | Console.app is a firehose. Without predicate filters, you're reading thousands of irrelevant system messages. It also doesn't help with crash analysis or historical log search. | Use `log show` with predicates for targeted search. Use Console.app only for real-time streaming with filters configured. |
| "I'll figure out the crash from the stack trace alone" | Stack traces show where the crash happened, not why. The log entries leading up to the crash reveal the state that caused it. A crash in `String.init` tells you nothing; the preceding "Received nil response from API" tells you everything. | Always read logs before and around the crash timestamp. Correlate with any request/response logging. |
| "Debug logging in production is fine, I'll remove it later" | You won't. Debug logs in production consume disk I/O, battery, and may leak sensitive data. Even if `os.Logger` debug level doesn't persist by default, excessive logging at info+ level has real performance cost. | Use `.debug` level for development diagnostics. Use `.info` for normal operations. Audit log levels before release. |

---

## Red Flags -- STOP

- **Using `print()` instead of `os.Logger`** -- If you see `print()` for anything other than a throwaway playground experiment, replace it with `os.Logger`. No exceptions.
- **PII in log output** -- Names, emails, phone numbers, tokens, passwords, financial data. Use `os.Logger`'s `.private` privacy level. If you see PII in logs, it's a security incident.
- **Missing log level classification** -- Everything logged at the same level (typically `.info` or `.error`) is useless for filtering. Classify properly: debug for dev, info for normal, error for failures, fault for system failures.
- **Debugging without checking logs first** -- If someone says "the app crashed" and your first action is to add breakpoints or print statements, stop. Read the crash log first. The answer is usually already there.
- **Swallowing errors in catch blocks** -- `catch { }` or `catch { return nil }` without logging is a debugging black hole. Every catch must log what failed and why.

---

## Behavioral Enforcement

### Phase Gate: EVIDENCE FIRST
**Cannot form ANY hypothesis until:**
- [ ] Actual error message read (not paraphrased from memory)
- [ ] Surrounding log context checked (5 minutes before and after the error)
- [ ] Error pattern matched against known patterns (see references/error-patterns.md)

**Hard Stop**: If about to say "I think the problem is..." without having read the actual logs -- STOP. Read the logs. Hypothesizing without evidence wastes time on wrong theories.

### Phase Gate: LOG ANALYSIS
**Cannot declare root cause until:**
- [ ] Error reproduced or pattern confirmed across multiple log entries
- [ ] Correlation IDs traced across components (if applicable)
- [ ] Timeline of events reconstructed from logs

### Self-Audit
1. Did I read the actual error message, or am I working from a description?
2. Did I check the surrounding context, or just the error line?
3. Can I reconstruct the sequence of events that led to the error?
4. Have I checked for similar errors in the recent log history?

### Required Output Artifact
Every log investigation must produce:
- The actual error message(s) with timestamps
- Timeline of events leading to the error (reconstructed from log context)
- Root cause determination with supporting log evidence (not assumption)

---

## When NOT to Use This Skill

1. **Performance profiling and optimization** -- Logs can identify slow operations, but Instruments is the right tool for CPU, memory, and energy profiling. Use Time Profiler, Allocations, and Leaks instruments instead.
2. **UI layout debugging** -- For Auto Layout constraint issues, use Xcode's Debug View Hierarchy or Reveal. Log entries about constraint conflicts are symptoms; visual debugging shows the cause.
3. **Network request/response inspection** -- For detailed HTTP traffic analysis, use Charles Proxy, Proxyman, or Instruments Network template. Logs show errors but not full request/response bodies.

---

## Decision Framework

```
Something is wrong with the app. What do I do?

+-- App crashed?
|   |
|   +-- Recent crash? --> `log show --last 5m --predicate 'process == "{PROCESS_NAME}"'`
|   |   Look for: EXC_BAD_ACCESS, SIGABRT, fatalError, preconditionFailure
|   |
|   +-- Historical crash? --> Check ~/Library/Logs/DiagnosticReports/
|   |   Look for: .ips files matching the app name
|   |   Symbolicate with: `atos -arch arm64 -o App.app/App -l <load_addr> <crash_addr>`
|   |
|   +-- Crash on device? --> `idevicesyslog -u <udid> | grep -i "{PROCESS_NAME}"`
|       Also check: Settings > Privacy > Analytics > Analytics Data
|
+-- App misbehaving (not crashing)?
|   |
|   +-- Know which subsystem? --> Filter by category
|   |   `log show --last 10m --predicate 'subsystem == "{BUNDLE_ID}" AND category == "networking"'`
|   |
|   +-- Don't know where? --> Search for errors across all categories
|   |   `log show --last 10m --predicate 'subsystem == "{BUNDLE_ID}" AND messageType >= 16'`
|   |   (messageType >= 16 = error and fault levels)
|   |
|   +-- Intermittent issue? --> Stream logs and reproduce
|       `xcrun simctl spawn booted log stream --predicate 'subsystem == "{BUNDLE_ID}"' --level debug`
|
+-- Need to understand behavior (not a bug)?
    |
    +-- Execution flow? --> Stream with debug level and trace the sequence
    +-- State changes? --> Search for specific state transition log messages
    +-- Performance? --> Search for timing/duration log entries, then switch to Instruments
```

---

## Standard Operating Procedures

### SOP-1: Real-time Device Log Streaming

```bash
# Stream logs from physical device (use actual device UDID)
# Find UDID: xcrun xctrace list devices
idevicesyslog -u <device-udid>

# Filter for app only
idevicesyslog -u <device-udid> | grep -i "{PROCESS_NAME}"

# Stream with timestamp and filter for errors
idevicesyslog -u <device-udid> -d | grep -iE "(error|fault|crash|exception)"

# Alternative: use log stream over device (requires developer disk image mounted)
# This gives you predicate filtering, which grep cannot do as precisely
xcrun devicectl device log stream --device <device-udid> \
  --predicate 'subsystem == "{BUNDLE_ID}"'
```

**Why device logs matter:**
Simulator behavior differs from device behavior. Keychain access, push notifications, background execution, network conditions, and memory pressure all behave differently on real hardware. If a bug only reproduces on device, device logs are your only source of truth.

### SOP-2: Simulator Log Streaming

```bash
# Stream from booted simulator -- filtered by your app's subsystem
xcrun simctl spawn booted log stream --level debug \
  --predicate 'subsystem == "{BUNDLE_ID}"'

# Filter by specific category (e.g., networking, auth, storage)
xcrun simctl spawn booted log stream --level debug \
  --predicate 'subsystem == "{BUNDLE_ID}" AND category == "networking"'

# Filter by process name (catches all logging, not just os.Logger)
xcrun simctl spawn booted log stream --process "{PROCESS_NAME}"

# Errors and faults only (quiet mode for monitoring)
xcrun simctl spawn booted log stream \
  --predicate 'subsystem == "{BUNDLE_ID}" AND messageType >= 16'

# Stream to file for later analysis
xcrun simctl spawn booted log stream --level debug \
  --predicate 'subsystem == "{BUNDLE_ID}"' > /tmp/app-logs.txt 2>&1 &
```

**Predicate syntax reference:**
- `subsystem == "com.app.bundle"` -- exact match
- `subsystem BEGINSWITH "com.app"` -- prefix match (catches extensions too)
- `category == "networking"` -- specific functional area
- `messageType >= 16` -- error level and above (16=error, 17=fault)
- `eventMessage CONTAINS "failed"` -- message content search
- Combine with `AND`, `OR`, `NOT`

### SOP-3: Historical Log Search

```bash
# Search recent logs (last 5 minutes)
log show --last 5m --predicate 'process == "{PROCESS_NAME}"'

# Search with specific level (error and above)
log show --last 1h --predicate 'process == "{PROCESS_NAME}" AND messageType >= 16'

# Search by time range
log show --start "2026-03-16 10:00:00" --end "2026-03-16 10:05:00" \
  --predicate 'subsystem == "{BUNDLE_ID}"'

# Search crash logs specifically
log show --predicate 'eventMessage CONTAINS "crash" OR eventMessage CONTAINS "SIGABRT"' \
  --last 1h

# Search with output formatting (for parsing)
log show --last 30m --predicate 'subsystem == "{BUNDLE_ID}"' \
  --style json > /tmp/logs.json

# Count errors by category (diagnostic overview)
log show --last 1h --predicate 'subsystem == "{BUNDLE_ID}" AND messageType >= 16' \
  --style compact | sort | uniq -c | sort -rn
```

### SOP-4: Crash Log Analysis

```bash
# 1. Find crash logs
# Simulator crash logs:
ls ~/Library/Logs/DiagnosticReports/*.ips | grep -i "{PROCESS_NAME}"

# Device crash logs (after sync):
ls ~/Library/Logs/CrashReporter/MobileDevice/*/

# 2. Read the crash report
# Key sections to examine:
#   - Exception Type: EXC_BAD_ACCESS / EXC_CRASH / EXC_BREAKPOINT
#   - Exception Subtype: KERN_INVALID_ADDRESS / KERN_PROTECTION_FAILURE
#   - Termination Reason: tells you WHY the OS killed the app
#   - Crashed Thread: the thread that caused the crash
#   - Thread N Crashed: the stack trace to read

# 3. Symbolicate if needed
# Find the dSYM:
mdfind "com_apple_xcode_dsym_uuids == <UUID-from-crash-report>"

# Symbolicate an address:
atos -arch arm64 -o /path/to/App.app.dSYM/Contents/Resources/DWARF/App \
  -l 0x100000000 0x100001234

# 4. Common crash types and what they mean:
# EXC_BAD_ACCESS (SIGSEGV)  -> Accessing deallocated memory or null pointer
# EXC_BAD_ACCESS (SIGBUS)   -> Misaligned memory access
# EXC_CRASH (SIGABRT)       -> Assertion failure, force unwrap nil, array out of bounds
# EXC_BREAKPOINT (SIGTRAP)  -> Swift runtime error (precondition, fatalError)
# EXC_RESOURCE               -> Exceeded resource limit (memory, CPU, disk)
```

### SOP-5: Error Correlation Across Components

```bash
# When an error spans multiple subsystems (e.g., network call -> parsing -> storage):

# 1. Find the initial error
log show --last 10m --predicate 'subsystem == "{BUNDLE_ID}" AND messageType >= 16' \
  --style compact

# 2. Get the timestamp of the first error
# Example output: 2026-03-16 10:03:45.123 error: NetworkService: Request failed

# 3. Search all categories around that timestamp (+/- 2 seconds)
log show --start "2026-03-16 10:03:43" --end "2026-03-16 10:03:47" \
  --predicate 'subsystem == "{BUNDLE_ID}"' --level debug

# 4. This reveals the full sequence:
#   10:03:44.900 debug [networking] Starting request to /api/items
#   10:03:45.100 debug [networking] Received response: 200, 1.2KB
#   10:03:45.110 error [parsing]   DecodingError: key "price" not found
#   10:03:45.120 error [storage]   Failed to save items: parsing error
#   10:03:45.123 error [networking] Request failed (surfaced to UI)

# Now you know: the server returned 200 but the response schema changed.
# The fix is in the model, not the network layer.
```

**Why correlation matters:**
Errors propagate. The error you see in the UI is usually 2-3 layers removed from the actual cause. Reading logs at a single point in time shows the symptom. Reading the timeline shows the cause. Always widen your search window.

---

## Log Level Matrix

| Level | os.Logger Method | When to Use | Persists to Disk | Visible in Console.app |
|---|---|---|---|---|
| **debug** | `.debug()` | Development diagnostics, variable values, flow tracing | No (memory only) | Only if streaming |
| **info** | `.info()` | Normal operations worth noting: user actions, state changes | Briefly | Yes |
| **notice** | `.notice()` | Noteworthy events: feature first-use, config changes | Yes | Yes |
| **error** | `.error()` | Failures that the app can recover from: network timeout, parse error | Yes | Yes |
| **fault** | `.fault()` | System-level failures: out of memory, database corruption, unrecoverable state | Yes (persists across reboot) | Yes |

**Choosing the right level:**
- If you only need it during development: `.debug()`
- If you want to see it happened in production but it's normal: `.info()`
- If it's unusual and you want it to persist for investigation: `.notice()`
- If something failed but the app can continue: `.error()`
- If something is fundamentally broken: `.fault()`

---

## Structured Logging Implementation

See `references/structured-logging.md` for comprehensive setup:
- `os.Logger` initialization with subsystem/category design
- Privacy levels (`.public`, `.private`, `.sensitive`) and GDPR implications
- Performance characteristics and why `os.Logger` is safe in production
- Integration with Console.app and Instruments

## Error Pattern Reference

### Common iOS Error Patterns

```
# Common iOS error patterns to watch
- [ERROR]          -> Application error
- EXC_BAD_ACCESS   -> Memory access violation (dangling pointer, use-after-free)
- SIGABRT          -> Assertion failure / forced abort
- URLError         -> Network error (timeout, DNS, connectivity)
- DecodingError    -> JSON/Codable parsing failure
- NSCocoaError     -> Foundation framework error (file I/O, Core Data)
- CancellationError -> Async task cancelled

# App-specific error patterns:
# Identify your app's custom error types by searching for `Error` enums/structs
# in the codebase and add them to this list. Examples:
# - APIError       -> Backend API failure
# - AuthError      -> Authentication/authorization failure
# - PaymentError   -> Payment processing failure
# - DatabaseError  -> Local/remote database error
```

See `references/error-patterns.md` for 30+ iOS error patterns:
- Networking errors (URLError codes, SSL, DNS)
- CoreData errors (merge conflicts, migration failures)
- Concurrency errors (CancellationError, actor isolation)
- Security errors (Keychain OSStatus, LAError biometric codes)
- UI errors (constraint conflicts, navigation stack corruption)

---

## Quality Gates (Before Marking Complete)

- [ ] All `print()` statements replaced with `os.Logger` calls (or confirmed as intentional for CLI tools)
- [ ] Log levels correctly classified (debug for dev, info for normal, error for failures, fault for system)
- [ ] No PII logged at `.public` privacy level -- names, emails, tokens use `.private` or `.sensitive`
- [ ] Every `catch` block logs the error with context (what operation failed, what input caused it)
- [ ] Subsystem matches bundle ID, categories match functional areas (networking, auth, storage, UI)
- [ ] Error messages are actionable: "Failed to decode User from /api/profile: missing key 'email'" not "Decode error"
- [ ] Crash log analyzed before proposing code changes (if investigating a crash)
- [ ] Log search covered sufficient time window (not just the exact crash moment, but 10-30s before)

---

## Cross-Skill References

- **atlas-database** -- After deploying migrations, monitor logs for RLS permission denials or query errors that indicate schema/policy misconfiguration.
- **pipeline-cicd** -- CI test failures produce log output. Use the same log analysis techniques to diagnose CI failures as you would device issues.
- **aegis-notifications** -- Notification delivery failures appear in device logs. Search for `apsd` (APNs daemon) and `UserNotifications` subsystem entries.
- **velocity-fastlane** -- Build failures produce Xcode log output. Fastlane's `xcpretty` output can mask the actual error; check the raw log when `xcpretty` output is insufficient.

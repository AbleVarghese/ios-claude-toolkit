---
name: loki-logs
description: "This skill should be used when the user asks to 'stream device logs', 'search logs', 'log analysis', 'console output', or 'debug trace'. Provides real-time log streaming, pattern analysis, trace correlation, and log filtering for iOS devices and simulators."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: false
---

# Log Intelligence & Analytics

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). Replace `{BUNDLE_ID}` with the app's bundle identifier and `{PROCESS_NAME}` with the app's process/executable name.

Stream, search, filter, and analyze iOS device and simulator logs. Detect error patterns, correlate traces across components, and extract performance metrics from log output.

## Standard Operating Procedures

### SOP-1: Real-time Device Log Streaming
```bash
# Stream logs from physical device (use actual device UDID)
idevicesyslog -u <device-udid>

# Filter for app only (replace with your app's process name)
idevicesyslog -u <device-udid> | grep -i "{PROCESS_NAME}"

# Stream with timestamp
idevicesyslog -u <device-udid> -d
```

### SOP-2: Simulator Log Streaming
```bash
# Stream from booted simulator
xcrun simctl spawn booted log stream --level debug \
  --predicate 'subsystem == "{BUNDLE_ID}"'

# Filter by process (replace with your app's process name)
xcrun simctl spawn booted log stream --process {PROCESS_NAME}
```

### SOP-3: Log Search
```bash
# Search recent logs on device (replace with your app's process name)
log show --last 5m --predicate 'processImagePath contains "{PROCESS_NAME}"'

# Search with specific level
log show --last 1h --level error

# Search crash logs
log show --predicate 'eventMessage contains "crash"' --last 1h
```

### SOP-4: Log Analysis Patterns
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

## Log Level Matrix

| Level | When to Use | Retention |
|-------|-------------|-----------|
| **debug** | Development only, verbose | Session |
| **info** | Normal operations | 24h |
| **warning** | Unexpected but recoverable | 7d |
| **error** | Failures requiring attention | 30d |
| **critical** | System failures | Permanent |

# Structured Logging with os.Logger

Reference guide for implementing structured logging in iOS applications using Apple's unified logging system.

---

## Setup

```swift
import os

// Define loggers with subsystem (app-level) and category (functional area)
// Subsystem should be your bundle identifier
// Categories should match functional areas of your app

enum Log {
    /// Networking: API calls, responses, connectivity
    static let networking = Logger(subsystem: Bundle.main.bundleIdentifier ?? "com.app", category: "networking")

    /// Authentication: login, logout, token refresh, biometrics
    static let auth = Logger(subsystem: Bundle.main.bundleIdentifier ?? "com.app", category: "auth")

    /// Storage: CoreData, file I/O, Keychain, UserDefaults
    static let storage = Logger(subsystem: Bundle.main.bundleIdentifier ?? "com.app", category: "storage")

    /// UI: navigation, view lifecycle, user interactions
    static let ui = Logger(subsystem: Bundle.main.bundleIdentifier ?? "com.app", category: "ui")

    /// Notifications: push, local, permission handling
    static let notifications = Logger(subsystem: Bundle.main.bundleIdentifier ?? "com.app", category: "notifications")

    /// Sync: background sync, data reconciliation
    static let sync = Logger(subsystem: Bundle.main.bundleIdentifier ?? "com.app", category: "sync")

    /// Payments: transactions, receipts, subscription management
    static let payments = Logger(subsystem: Bundle.main.bundleIdentifier ?? "com.app", category: "payments")

    /// General: anything that doesn't fit a specific category
    static let general = Logger(subsystem: Bundle.main.bundleIdentifier ?? "com.app", category: "general")
}
```

**Why subsystem + category:**
- Subsystem identifies your app among all system processes. In Console.app, filter by subsystem to see only your app's logs.
- Category identifies the functional area. Filter by category to focus on networking issues without seeing UI logs.
- Together, they create a structured namespace: `com.myapp.networking`, `com.myapp.auth`, etc.

---

## Log Levels

```swift
// DEBUG: Development-only diagnostics. Variable values, flow tracing.
// Not persisted to disk. Only visible when actively streaming.
Log.networking.debug("Request headers: \(headers, privacy: .private)")

// INFO: Normal operations worth noting. User actions, state changes.
// Persisted briefly in memory. Visible in Console.app.
Log.auth.info("User signed in successfully")

// NOTICE: Noteworthy but expected events. Configuration changes, feature first-use.
// Persisted to disk. Available in historical log searches.
Log.general.notice("App launched for the first time after update to \(version, privacy: .public)")

// ERROR: Failures the app can recover from. Network timeouts, parse errors.
// Persisted to disk with extended retention.
Log.networking.error("Request to /api/items failed: \(error.localizedDescription, privacy: .public)")

// FAULT: System-level failures. Database corruption, unrecoverable state.
// Persisted to disk, survives reboot. Highest severity.
Log.storage.fault("CoreData persistent store failed to load: \(error.localizedDescription, privacy: .public)")
```

**Level selection cheat sheet:**
- "I need this while developing" --> `.debug()`
- "I want to know this happened in production" --> `.info()`
- "This is unusual and I want it in the logs" --> `.notice()`
- "Something failed but the app keeps working" --> `.error()`
- "Something is fundamentally broken" --> `.fault()`

---

## Privacy Levels

os.Logger has built-in privacy controls that redact sensitive data in production logs. This is critical for GDPR and user privacy compliance.

```swift
// PUBLIC: Visible in all log outputs. Use for non-sensitive operational data.
Log.networking.info("Request to \(endpoint, privacy: .public) returned \(statusCode, privacy: .public)")

// PRIVATE (default): Redacted in log output unless debugging with Xcode attached.
// Appears as "<private>" in Console.app and log show.
Log.auth.info("User signed in: \(userId, privacy: .private)")
// Output in Console.app: "User signed in: <private>"
// Output in Xcode debugger: "User signed in: usr_abc123"

// SENSITIVE: Like private, but also redacted in Instruments and other tools.
// Use for PII, financial data, credentials.
Log.payments.info("Processing payment for \(email, privacy: .sensitive)")

// PRIVATE with hash: Redacted but consistent hash allows correlation.
// Same input always produces the same hash in the same session.
Log.networking.info("Request from user \(userId, privacy: .private(mask: .hash))")
// Output: "Request from user <hash: 7A3B2C>"
// Same userId always shows "<hash: 7A3B2C>" in this session, so you can correlate.
```

**Privacy decision guide:**
- Technical identifiers (request IDs, error codes, endpoint paths): `.public`
- User identifiers (user ID, session ID): `.private`
- PII (email, name, phone, address): `.sensitive`
- Financial data (amounts, card info, account numbers): `.sensitive`
- Tokens, passwords, API keys: NEVER LOG THESE. Not even at `.sensitive`.

---

## String Interpolation in os.Logger

os.Logger uses compile-time string interpolation, not runtime `String` construction. This is important for both performance and privacy.

```swift
// CORRECT: String interpolation in os.Logger is handled at compile time
// The log message is only formatted if the log level is enabled
Log.networking.debug("Response body: \(body, privacy: .private)")

// WRONG: Don't construct strings before logging
// This allocates the string even if debug logging is disabled
let message = "Response body: \(body)"  // Allocated regardless of log level
Log.networking.debug("\(message)")       // Privacy controls don't work on pre-built strings

// CORRECT: Use formatters inline
Log.networking.info("Request took \(duration, format: .fixed(precision: 2), privacy: .public)ms")
Log.storage.info("File size: \(bytes, format: .byteCount(countStyle: .file), privacy: .public)")
```

**Performance implications:**
- `os.Logger` is lazy: the message string is only constructed if the log level is active
- Leaving `debug` log calls in production code is safe -- they have near-zero cost when not streaming
- No need to wrap log calls in `#if DEBUG` -- the system handles this efficiently
- This makes `os.Logger` fundamentally different from `print()`, which always evaluates its argument

---

## Logging Patterns for Common Operations

### Network Requests
```swift
func fetchItems() async throws -> [Item] {
    let endpoint = "/api/items"
    Log.networking.debug("Starting request to \(endpoint, privacy: .public)")

    do {
        let (data, response) = try await urlSession.data(from: url)
        let httpResponse = response as? HTTPURLResponse
        let statusCode = httpResponse?.statusCode ?? -1

        Log.networking.info("Response from \(endpoint, privacy: .public): \(statusCode, privacy: .public), \(data.count, privacy: .public) bytes")

        let items = try decoder.decode([Item].self, from: data)
        Log.networking.debug("Decoded \(items.count, privacy: .public) items from \(endpoint, privacy: .public)")
        return items
    } catch let urlError as URLError {
        Log.networking.error("Network error on \(endpoint, privacy: .public): \(urlError.code.rawValue, privacy: .public) - \(urlError.localizedDescription, privacy: .public)")
        throw urlError
    } catch let decodingError as DecodingError {
        Log.networking.error("Decoding error on \(endpoint, privacy: .public): \(String(describing: decodingError), privacy: .public)")
        throw decodingError
    }
}
```

### Error Handling
```swift
func processPayment(_ payment: Payment) async throws -> Receipt {
    Log.payments.info("Processing payment \(payment.id, privacy: .private) for \(payment.amount, privacy: .sensitive) \(payment.currency, privacy: .public)")

    do {
        let receipt = try await paymentService.charge(payment)
        Log.payments.info("Payment \(payment.id, privacy: .private) succeeded: receipt \(receipt.id, privacy: .private)")
        return receipt
    } catch {
        // ALWAYS log errors with context
        Log.payments.error("Payment \(payment.id, privacy: .private) failed: \(error.localizedDescription, privacy: .public)")
        // Never: catch { } -- swallowing errors silently is a debugging black hole
        throw error
    }
}
```

### State Changes
```swift
func updateUserStatus(to newStatus: UserStatus) {
    let oldStatus = currentStatus
    currentStatus = newStatus
    Log.auth.notice("User status changed: \(String(describing: oldStatus), privacy: .public) -> \(String(describing: newStatus), privacy: .public)")
}
```

---

## Integration with Console.app

1. Open Console.app
2. Select your device or simulator from the sidebar
3. Filter by:
   - Subsystem: your bundle identifier (e.g., `com.myapp`)
   - Category: specific area (e.g., `networking`)
   - Type: `Error` to see only errors and faults
4. Use the search bar for message content filtering
5. Action menu > Include Info Messages / Include Debug Messages to see lower levels

**Tips:**
- Start streaming BEFORE reproducing the issue
- Use "Start" button to begin streaming, reproduce, then "Pause" to freeze output
- Right-click a message > "Show in Finder" to see the source binary

---

## Integration with Instruments

1. Open Instruments
2. Choose the "Logging" template
3. Record while using the app
4. Filter by subsystem and category in the detail view
5. Correlate log messages with other instrument tracks (CPU, memory, network)

**Why Instruments logging > Console.app for performance issues:**
- Timeline view shows log messages alongside performance data
- You can see exactly what log message corresponds to a CPU spike
- Filtering is more precise and the UI handles large volumes better

---

## Testing with Logs

```swift
// In tests, verify logging behavior using OSLogStore (iOS 15+)
import OSLog

func testErrorLogging() async throws {
    let store = try OSLogStore(scope: .currentProcessIdentifier)
    let position = store.position(timeIntervalSinceLatestBoot: 0)

    // Perform the operation that should log
    try? await sut.fetchItems()  // This should log an error

    // Query the log store
    let entries = try store.getEntries(at: position)
        .compactMap { $0 as? OSLogEntryLog }
        .filter { $0.subsystem == "com.myapp" && $0.category == "networking" }

    // Assert
    XCTAssertTrue(entries.contains { $0.level == .error })
}
```

Note: `OSLogStore` only captures logs from the current process. It works for unit tests but not for UI tests running in a separate process.

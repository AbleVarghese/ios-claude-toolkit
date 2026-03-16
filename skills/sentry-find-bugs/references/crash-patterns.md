# iOS Crash Patterns Catalog

Top 30+ iOS crash patterns organized by type. For each pattern: stack trace signature, typical root cause, fix approach, and prevention strategy.

---

## Memory Crashes

### 1. EXC_BAD_ACCESS -- Nil Dereference (0x0)

**Stack trace signature:**
```
EXC_BAD_ACCESS (SIGSEGV) KERN_INVALID_ADDRESS at 0x0000000000000000
```

**Typical root cause:** Accessing a property or calling a method on a nil Objective-C pointer, or force-unwrapping a nil Swift optional that bridges to Obj-C.

**Fix approach:**
- Find the nil value. Check optional bindings, delegate references, IBOutlet connections.
- Use `guard let` or `if let` instead of force unwrap.
- For Obj-C interop, ensure bridged optionals are properly checked.

**Prevention:** Enable `-warn-implicit-retain-self` compiler flag. Audit all `!` operators.

---

### 2. EXC_BAD_ACCESS -- Use-After-Free (non-zero address)

**Stack trace signature:**
```
EXC_BAD_ACCESS (SIGSEGV) KERN_INVALID_ADDRESS at 0x1a2b3c4d5e6f
```

**Typical root cause:** An `unowned` reference to an object that has been deallocated, or an unsafe pointer used after the underlying memory was freed.

**Fix approach:**
- Replace `unowned` with `weak` if the referenced object can be deallocated during the reference's lifetime.
- For unsafe pointers, ensure the owning scope outlives all uses.
- Enable Address Sanitizer (ASan) to catch use-after-free during development.

**Prevention:** Prefer `weak` over `unowned` unless the ownership guarantees are provably correct.

---

### 3. EXC_BAD_ACCESS -- Buffer Overflow

**Stack trace signature:**
```
EXC_BAD_ACCESS (SIGSEGV) at address adjacent to valid allocation
```

**Typical root cause:** Writing past the end of a buffer, often in C interop code (UnsafeMutableBufferPointer, Data.withUnsafeMutableBytes).

**Fix approach:**
- Add bounds checking before all buffer operations.
- Use Swift's safe buffer access methods instead of raw pointer arithmetic.

**Prevention:** Enable Address Sanitizer in test schemes. Minimize unsafe pointer usage.

---

### 4. Out of Memory (OOM / Jetsam)

**Stack trace signature:**
```
Terminated due to memory pressure
Exception Type: EXC_RESOURCE (RESOURCE_TYPE_MEMORY)
```
Or no crash report at all (Jetsam kills leave no crash log).

**Typical root cause:** Memory growing unboundedly from:
- Loading full-resolution images without downsampling
- Retain cycles preventing deallocation
- Unbounded caches
- Processing large datasets in memory

**Fix approach:**
- Profile with Allocations instrument. Look for persistent allocations growing over time.
- Downsample images to display size (see prometheus-performance references).
- Add memory warnings handling: `UIApplication.didReceiveMemoryWarningNotification`.
- Bound cache sizes with `NSCache` or custom LRU.

**Prevention:** Set memory budgets per feature. Monitor with `os_proc_available_memory()`.

---

## Threading Crashes

### 5. EXC_BAD_INSTRUCTION -- Actor Isolation Violation

**Stack trace signature:**
```
EXC_BAD_INSTRUCTION (SIGILL)
swift_task_enqueue_global
swift_task_switch
```

**Typical root cause:** Accessing an actor-isolated property from a non-isolated context without `await`. In Swift 6 strict concurrency mode, this is a compile error. In Swift 5 with complete checking, it's a runtime crash.

**Fix approach:**
- Add `await` before accessing actor-isolated properties.
- Add `@MainActor` to the calling function if it should be on the main actor.
- Use `nonisolated` for properties that don't need isolation.

**Prevention:** Enable Swift 6 mode or `SWIFT_STRICT_CONCURRENCY=complete`.

---

### 6. Main Thread Deadlock

**Stack trace signature:**
```
Thread 0 (main): blocked on semaphore/mutex/dispatch_sync
```

**Typical root cause:** Synchronous dispatch to main queue from main queue, or waiting on a semaphore that's signaled on main queue.

**Fix approach:**
- Never use `DispatchQueue.main.sync` from the main thread.
- Replace `DispatchSemaphore.wait()` on main thread with async/await patterns.
- Use `MainActor.run {}` instead of `DispatchQueue.main.async {}` in async contexts.

**Prevention:** Enable Main Thread Checker in Xcode diagnostics.

---

### 7. Data Race Crash

**Stack trace signature:**
```
Thread Sanitizer: data race
Read/Write at address 0x... by thread X
Previous write at address 0x... by thread Y
```

**Typical root cause:** Multiple threads reading and writing the same memory without synchronization.

**Fix approach:**
- Use actors for shared mutable state.
- Make types conform to `Sendable` or use `@unchecked Sendable` with internal synchronization.
- Use `os_unfair_lock` or `NSLock` for fine-grained synchronization (rare -- prefer actors).

**Prevention:** Enable Thread Sanitizer (TSan) in test schemes. Enable strict concurrency checking.

---

### 8. Background Thread UI Update

**Stack trace signature:**
```
[UIView setNeedsLayout] called from background thread
Main Thread Checker: UI API called on a background thread
```

**Typical root cause:** Updating `@Published` properties, UIKit views, or SwiftUI state from a background thread/actor.

**Fix approach:**
- Add `@MainActor` annotation to the class or method.
- Use `await MainActor.run { }` for specific updates.
- In UIKit, wrap in `DispatchQueue.main.async { }`.

**Prevention:** Add `@MainActor` to all ViewModels and UI-related classes.

---

## Logic Crashes

### 9. fatalError / preconditionFailure

**Stack trace signature:**
```
Fatal error: [message]
swift_fatalError
```

**Typical root cause:** Developer-placed fatal error for "impossible" states that turned out to be possible in production.

**Fix approach:**
- Replace `fatalError` with proper error handling.
- If the state truly should be impossible, add a crash report with context (but still handle gracefully).
- Investigate how the "impossible" state was reached.

**Prevention:** Never use `fatalError` in production paths. Use `assertionFailure` (debug only) + graceful handling.

---

### 10. Index Out of Range

**Stack trace signature:**
```
Fatal error: Index out of range
swift_checkIndex
```

**Typical root cause:** Accessing an array element at an index that doesn't exist. Common in:
- Collection views with stale data source counts
- Off-by-one errors in loop bounds
- Concurrent modification of arrays

**Fix approach:**
- Add bounds checking: `guard index < array.count else { return }`
- Use safe subscript extensions
- For collection views, ensure data source is updated before calling `reloadData()`

**Prevention:** Use `first`, `last`, and `indices.contains()` instead of direct subscript access.

---

### 11. Force Unwrap Nil Optional

**Stack trace signature:**
```
Fatal error: Unexpectedly found nil while unwrapping an Optional value
```

**Typical root cause:** Using `!` on a nil Optional. The most common Swift crash.

**Fix approach:**
- Replace `!` with `guard let` / `if let` / `??` (nil coalescing).
- Trace backward to find WHY the value is nil.
- Fix the data flow that failed to provide the value.

**Prevention:** Ban `!` in production code via SwiftLint rules. Use `guard let` everywhere.

---

### 12. Forced Downcast Failure

**Stack trace signature:**
```
Could not cast value of type 'X' to 'Y'
```

**Typical root cause:** Using `as!` for type casting when the types don't match. Common with:
- API response parsing
- UserInfo dictionary access
- Storyboard/XIB connections

**Fix approach:**
- Replace `as!` with `as?` and handle the nil case.
- For Codable types, use proper decoding with error handling.

**Prevention:** Ban `as!` in production code via SwiftLint rules.

---

### 13. Decoding Error

**Stack trace signature:**
```
DecodingError.keyNotFound("fieldName")
DecodingError.typeMismatch(expected: Int, found: String)
DecodingError.dataCorrupted
```

**Typical root cause:** API response structure doesn't match the Codable model. Common causes:
- API version change
- Nullable field not marked as Optional in model
- Type mismatch (number as string, etc.)

**Fix approach:**
- Make uncertain fields Optional.
- Use custom CodingKeys with default values.
- Add try-catch at the decode boundary with specific error handling.

**Prevention:** Validate API contracts with tests. Make all API-sourced fields Optional unless guaranteed by API contract.

---

## System Crashes

### 14. Watchdog Timeout

**Stack trace signature:**
```
Exception Type: EXC_CRASH (SIGKILL)
Exception Codes: 0x8badf00d ("ate bad food")
Termination Reason: FRONTBOARD_SERVICE_TIMEOUT
```

**Typical root cause:** Main thread blocked for > 10 seconds during:
- App launch (`applicationDidFinishLaunching`)
- Scene transition (`applicationWillEnterForeground`)
- System event handling

**Fix approach:**
- Move heavy work off the main thread: network calls, database operations, file I/O, image processing.
- Use async/await for all I/O operations.
- Show a loading indicator and process in the background.

**Prevention:** Profile with Time Profiler focused on main thread. Set up alerts for main thread work > 1s.

---

### 15. Background Task Expiry

**Stack trace signature:**
```
Exception Type: EXC_CRASH (SIGKILL)
Termination Reason: IDLE_EXIT or BGTA (Background Task Assertion expired)
```

**Typical root cause:** Background task not completed within the allowed time (typically 30 seconds for `UIApplication.beginBackgroundTask`).

**Fix approach:**
- Ensure `endBackgroundTask()` is always called (use defer).
- For long-running work, use `BGTaskScheduler` instead.
- Reduce background work to fit within the time limit.

```swift
// CORRECT: Always end the background task
func performBackgroundWork() {
    var backgroundTask: UIBackgroundTaskIdentifier = .invalid
    backgroundTask = UIApplication.shared.beginBackgroundTask {
        // Expiration handler -- clean up
        UIApplication.shared.endBackgroundTask(backgroundTask)
        backgroundTask = .invalid
    }

    Task {
        defer {
            UIApplication.shared.endBackgroundTask(backgroundTask)
            backgroundTask = .invalid
        }

        // Do work (must complete within ~30 seconds)
        await uploadPendingData()
    }
}
```

**Prevention:** Always use `defer` for `endBackgroundTask()`. Monitor background time remaining.

---

### 16. Memory Pressure Kill (Jetsam)

**Stack trace signature:**
No crash report generated. Device logs show:
```
kernel: memorystatus_kill_specific_process: pid X [AppName] (per-process-limit)
```

**Typical root cause:** App exceeded its memory limit (varies by device, typically 800MB-2GB).

**Fix approach:**
- See OOM pattern (#4) above.
- Implement `applicationDidReceiveMemoryWarning` to drop caches.
- Use `autoreleasepool` for batch processing.
- Monitor with `os_proc_available_memory()`.

**Prevention:** Set memory budgets. Profile with Allocations instrument regularly.

---

## Network Crashes

### 17. SSL Certificate Error

**Stack trace signature:**
```
NSURLErrorDomain Code=-1202 "The certificate for this server is invalid"
kCFStreamErrorDomainSSL
```

**Typical root cause:** Certificate pinning mismatch, expired certificate, or self-signed certificate in production.

**Fix approach:**
- Check if the server certificate was rotated without updating pins.
- Verify ATS configuration allows the connection.
- For pinning issues, update the pinned certificate hash.

**Prevention:** Pin backup certificates. Monitor certificate expiration dates. Use public key pinning (survives cert rotation) instead of certificate pinning.

---

### 18. Network Timeout

**Stack trace signature:**
```
NSURLErrorDomain Code=-1001 "The request timed out"
```

**Typical root cause:** Server not responding, network congestion, or request timeout too short.

**Fix approach:**
- Implement retry with exponential backoff.
- Show appropriate error UI.
- Consider offline-first architecture.

**Prevention:** Set appropriate timeouts (30s for API, 60s for uploads). Implement retry logic. Cache responses.

---

### 19. DNS Failure

**Stack trace signature:**
```
NSURLErrorDomain Code=-1003 "A server with the specified hostname could not be found"
```

**Typical root cause:** No network connectivity, DNS server issues, or incorrect hostname.

**Fix approach:**
- Check network reachability before making requests.
- Show offline mode when no connectivity.
- Verify hostname configuration.

**Prevention:** Use `NWPathMonitor` for reachability. Implement offline-first patterns.

---

## Concurrency Crashes (Swift 6)

### 20. Sendable Violation

**Stack trace signature:**
```
Sending 'value' risks causing data races
```

**Typical root cause:** Passing a non-Sendable type across actor/concurrency boundaries.

**Fix approach:**
- Make the type conform to `Sendable` (if all stored properties are Sendable).
- Use `@unchecked Sendable` with internal synchronization if needed.
- Copy values before sending across boundaries.

**Prevention:** Enable strict concurrency checking. Design types as Sendable from the start.

---

### 21. Task Cancellation

**Stack trace signature:**
```
CancellationError
Task was cancelled
```

**Typical root cause:** A SwiftUI `.task` modifier was cancelled when the view was dismissed, but the task didn't handle cancellation.

**Fix approach:**
- Check `Task.isCancelled` at iteration points.
- Catch `CancellationError` at the top level.
- Use `withTaskCancellationHandler` for cleanup.

**Prevention:** Always handle cancellation in long-running tasks. Design tasks to be cancellation-safe.

---

### 22. AsyncStream Lifetime

**Stack trace signature:**
```
EXC_BAD_ACCESS in AsyncStream continuation
```

**Typical root cause:** The AsyncStream's continuation was used after the stream was cancelled or finished.

**Fix approach:**
- Store continuation weakly or check `onTermination`.
- Set up `onTermination` handler to clean up resources.

**Prevention:** Always set `onTermination` on AsyncStream continuations.

---

## UIKit / SwiftUI Crashes

### 23. Collection View Inconsistency

**Stack trace signature:**
```
NSInternalInconsistencyException
Invalid update: invalid number of items in section 0
```

**Typical root cause:** Data source count changed without corresponding insert/delete calls.

**Fix approach:**
- Use `DiffableDataSource` instead of manual section/row management.
- If using manual updates, ensure batch updates match data source changes exactly.

**Prevention:** Use DiffableDataSource or SwiftUI Lists (which handle diffs automatically).

---

### 24. Auto Layout Conflict

**Stack trace signature:**
```
Unable to simultaneously satisfy constraints
Will attempt to recover by breaking constraint
```

**Typical root cause:** Conflicting constraints, usually from mixing storyboard and programmatic constraints.

**Fix approach:**
- Set `translatesAutoresizingMaskIntoConstraints = false` for programmatic views.
- Audit constraint priorities.
- Remove conflicting constraints.

**Prevention:** Use SwiftUI (no Auto Layout issues) or establish consistent constraint patterns.

---

### 25. Unbalanced Appearance Transitions

**Stack trace signature:**
```
Unbalanced calls to begin/end appearance transitions for ViewController
```

**Typical root cause:** Presenting a view controller from a view controller that hasn't finished its own transition.

**Fix approach:**
- Present from `viewDidAppear`, not `viewWillAppear` or `viewDidLoad`.
- Check if a presentation is already in progress before presenting.

**Prevention:** Centralize navigation logic. Avoid chained presentations.

---

## Core Data Crashes

### 26. Core Data Concurrency Violation

**Stack trace signature:**
```
CoreData: error: Serious application error.
Exception was caught during Core Data change processing.
```

**Typical root cause:** Accessing a managed object from the wrong queue/context.

**Fix approach:**
- Use `perform { }` or `performAndWait { }` for all context operations.
- Pass object IDs between contexts, not managed objects.
- Use `@MainActor` for view context operations.

**Prevention:** Enable `-com.apple.CoreData.ConcurrencyDebug 1` launch argument.

---

### 27. Core Data Migration Failure

**Stack trace signature:**
```
NSPersistentStoreIncompatibleVersionHashError
The model used to open the store is incompatible with the one used to create it
```

**Typical root cause:** Data model changed without creating a migration mapping.

**Fix approach:**
- Create a mapping model for the migration.
- Use lightweight migration if the changes are simple (add column, rename).
- For complex migrations, create a custom migration policy.

**Prevention:** Test migrations with production data copies. Never modify existing model versions -- always create new ones.

---

## Keychain / Security Crashes

### 28. Keychain Access Denied

**Stack trace signature:**
```
OSStatus error -25291 (errSecNotAvailable)
OSStatus error -25293 (errSecAuthFailed)
```

**Typical root cause:** Keychain item has biometric protection but device has no biometrics, or app was reinstalled without clearing old keychain items.

**Fix approach:**
- Handle all Keychain error codes explicitly.
- Provide fallback for missing biometric hardware.
- Clear keychain items on first launch after reinstall.

**Prevention:** Check biometric availability before setting biometric protection. Handle all error codes.

---

### 29. Entitlement Missing

**Stack trace signature:**
```
This application's application-identifier entitlement does not match that of the installed application
```

**Typical root cause:** Provisioning profile mismatch, or missing capability in entitlements.

**Fix approach:**
- Verify entitlements match between development and distribution profiles.
- Ensure all required capabilities are enabled in the provisioning profile.

**Prevention:** Automate entitlement validation in CI/CD pipeline.

---

## Notification / Background Crashes

### 30. Notification Content Extension Crash

**Stack trace signature:**
```
Termination Reason: EXTENSION_POINT_VIOLATION
Exception: exceeded memory limit for extension
```

**Typical root cause:** Notification content extension using too much memory (typically limited to 30-50MB).

**Fix approach:**
- Minimize resources loaded in the extension.
- Downsample images for notification display.
- Share data via app group, not by embedding in the extension.

**Prevention:** Profile extension memory usage separately. Keep extensions minimal.

---

### 31. Push Notification Registration Failure

**Stack trace signature:**
```
application:didFailToRegisterForRemoteNotificationsWithError:
Error Domain=NSCocoaErrorDomain Code=3000
```

**Typical root cause:** Missing push notification entitlement, or attempting registration on simulator without proper configuration.

**Fix approach:**
- Verify push notification capability is enabled.
- Handle registration failure gracefully (app should work without push).
- Check APN certificate validity.

**Prevention:** Test push registration on physical devices. Handle failure path.

---

### 32. Widget Timeline Crash

**Stack trace signature:**
```
Widget archival/unarchival crash
NSKeyedUnarchiver decoding failure
```

**Typical root cause:** Widget timeline entry contains types that can't be archived/unarchived, or shared data between app and widget is corrupted.

**Fix approach:**
- Ensure all timeline entry types are Codable.
- Validate shared data before use.
- Handle decode failures gracefully.

**Prevention:** Use Codable for all shared data. Test widget in isolation.

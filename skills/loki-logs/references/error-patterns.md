# iOS Error Patterns Reference

Common error patterns organized by subsystem, with log signatures, typical causes, severity, and fix approach.

---

## Networking Errors (URLError)

| Error Code | Name | Log Signature | Typical Cause | Severity | Fix Approach |
|---|---|---|---|---|---|
| -1001 | `.timedOut` | `NSURLErrorTimedOut` | Server slow or unresponsive, request timeout too short | Medium | Increase timeout, add retry with backoff |
| -1009 | `.notConnectedToInternet` | `NSURLErrorNotConnectedToInternet` | Device has no network connectivity | Low | Show offline state, queue request for retry |
| -1003 | `.cannotFindHost` | `NSURLErrorCannotFindHost` | DNS resolution failed, wrong hostname | High | Verify hostname, check DNS configuration |
| -1004 | `.cannotConnectToHost` | `NSURLErrorCannotConnectToHost` | Server unreachable, wrong port, firewall | High | Verify server is running, check port/firewall |
| -1005 | `.networkConnectionLost` | `NSURLErrorNetworkConnectionLost` | Connection dropped mid-request (WiFi handoff, tunnel) | Medium | Retry the request automatically |
| -1200 | `.secureConnectionFailed` | `NSURLErrorSecureConnectionFailed` | SSL/TLS handshake failed, certificate issue | Critical | Check certificate chain, ATS configuration |
| -1202 | `.serverCertificateUntrusted` | `NSURLErrorServerCertificateUntrusted` | Self-signed cert, expired cert, wrong domain | Critical | Fix server certificate, don't disable ATS |
| -999 | `.cancelled` | `NSURLErrorCancelled` | Request cancelled by code or task cancellation | Low | Usually intentional, verify cancellation logic |

### SSL/TLS Errors
```
Log: "An SSL error has occurred and a secure connection to the server cannot be made"
Cause: ATS violation, expired certificate, TLS version mismatch
Fix: Check Info.plist ATS settings, verify server supports TLS 1.2+
Never: Disable ATS entirely to "fix" SSL errors
```

### HTTP Status Errors (not URLError, but common)
```
401 Unauthorized -> Token expired, not authenticated. Refresh token and retry.
403 Forbidden    -> Authenticated but not authorized. Check user permissions.
404 Not Found    -> Resource doesn't exist. Check URL construction.
422 Unprocessable -> Validation error. Check request body.
429 Too Many Requests -> Rate limited. Respect Retry-After header.
500 Internal Server Error -> Server bug. Log full request for reproduction.
502/503/504 -> Server infrastructure issue. Retry with backoff.
```

---

## Decoding Errors (DecodingError)

| Error | Log Signature | Typical Cause | Fix Approach |
|---|---|---|---|
| `.keyNotFound` | `keyNotFound("fieldName", ...)` | API response missing expected key | Make the field optional, or fix the API |
| `.typeMismatch` | `typeMismatch(Int.self, ...)` | API returns string "123" but model expects Int | Use custom decoder or make field type flexible |
| `.valueNotFound` | `valueNotFound(String.self, ...)` | JSON has `null` for a non-optional field | Make the field optional in your model |
| `.dataCorrupted` | `dataCorrupted(...)` | Invalid JSON, wrong content type, date format | Check Content-Type header, validate JSON format |

```swift
// Debugging DecodingError with context
catch let error as DecodingError {
    switch error {
    case .keyNotFound(let key, let context):
        Log.networking.error("Missing key '\(key.stringValue, privacy: .public)' at path: \(context.codingPath.map(\.stringValue).joined(separator: "."), privacy: .public)")
    case .typeMismatch(let type, let context):
        Log.networking.error("Type mismatch: expected \(String(describing: type), privacy: .public) at path: \(context.codingPath.map(\.stringValue).joined(separator: "."), privacy: .public)")
    case .valueNotFound(let type, let context):
        Log.networking.error("Null value for \(String(describing: type), privacy: .public) at path: \(context.codingPath.map(\.stringValue).joined(separator: "."), privacy: .public)")
    case .dataCorrupted(let context):
        Log.networking.error("Data corrupted: \(context.debugDescription, privacy: .public)")
    @unknown default:
        Log.networking.error("Unknown decoding error: \(error.localizedDescription, privacy: .public)")
    }
}
```

---

## CoreData Errors (NSCocoaError Domain)

| Error Code | Name | Log Signature | Typical Cause | Fix Approach |
|---|---|---|---|---|
| 133000 | `NSPersistentStoreIncompatibleVersionHashError` | `"The model used to open the store is incompatible"` | Schema changed without migration | Add Core Data migration (lightweight or custom) |
| 133020 | `NSMigrationMissingSourceModelError` | `"Can't find model for source store"` | Old model version deleted from bundle | Keep all model versions in the project |
| 132001 | `NSManagedObjectValidationError` | `"The operation couldn't be completed"` | Validation rule violated (non-optional field is nil) | Set required fields before save |
| 133021 | `NSMigrationMissingMappingModelError` | `"No mapping model found"` | Complex migration without mapping model | Create explicit mapping model |

### Merge Conflicts
```
Log: "Could not merge changes"
Cause: Two contexts modified the same object simultaneously
Fix: Set merge policy on the context:
  context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
  // or handle conflicts manually in the merge notification
```

### Save Errors
```
Log: "The operation couldn't be completed. (Cocoa error 1570.)"
Cause: Non-optional attribute is nil at save time
Fix: Check all required attributes are set before calling save()
Debug: print(error.userInfo) shows which attribute failed validation
```

---

## Concurrency Errors (Swift 6)

| Error | Log Signature | Typical Cause | Fix Approach |
|---|---|---|---|
| `CancellationError` | `CancellationError()` | Task.cancel() called, or parent task cancelled | Check `Task.isCancelled` before expensive work, use `try Task.checkCancellation()` |
| Actor isolation | `"cannot access property from non-isolated context"` | Accessing @MainActor property from background | Use `await MainActor.run { }` or make the calling function @MainActor |
| Data race | `"Simultaneous access to..."` / `TSAN` warning | Multiple threads accessing shared mutable state | Use actors, Sendable conformance, or synchronization primitives |
| `TaskGroup` error | `"Cannot add child task to cancelled group"` | Adding tasks to a group that's already been cancelled | Check group cancellation before adding tasks |

### Swift 6 Strict Concurrency Common Issues
```
"Sending 'self.property' risks causing data races"
Cause: Passing non-Sendable value across isolation boundary
Fix: Make the type Sendable, use @unchecked Sendable (with justification),
     or restructure to avoid crossing the boundary

"Main actor-isolated property cannot be mutated from a nonisolated context"
Cause: Trying to update UI from a background context
Fix: Wrap in await MainActor.run { } or mark the calling function @MainActor

"Task-isolated value of type 'X' passed as a strongly transferred parameter"
Cause: Transferring ownership of a value that can't safely cross isolation
Fix: Copy the value or make it Sendable
```

---

## Security Errors

### Keychain (OSStatus Codes)

| Status Code | Name | Typical Cause | Fix Approach |
|---|---|---|---|
| -25299 | `errSecDuplicateItem` | Item already exists in Keychain | Use `SecItemUpdate` instead of `SecItemAdd`, or delete first |
| -25300 | `errSecItemNotFound` | No matching Keychain item | Item was never stored, or query attributes don't match |
| -25291 | `errSecNotAvailable` | Keychain not available | Device locked, data protection class too restrictive |
| -25293 | `errSecAuthFailed` | Authentication failed | Wrong access control, biometric changed, passcode changed |
| -34018 | `errSecMissingEntitlement` | Missing Keychain sharing entitlement | Add Keychain Sharing capability in Xcode |

### Biometric Auth (LAError)

| Error Code | Name | Typical Cause | Fix Approach |
|---|---|---|---|
| -1 | `.authenticationFailed` | Biometric didn't match | Offer passcode fallback |
| -2 | `.userCancel` | User tapped Cancel | Respect the cancellation, don't re-prompt immediately |
| -3 | `.userFallback` | User tapped "Enter Password" | Show password entry UI |
| -5 | `.passcodeNotSet` | No device passcode configured | Guide user to set up device passcode |
| -6 | `.touchIDNotAvailable` | No biometric hardware | Hide biometric option, use password only |
| -7 | `.touchIDNotEnrolled` | No fingerprints/face enrolled | Guide user to Settings > Face ID / Touch ID |
| -8 | `.touchIDLockout` | Too many failed attempts | Biometric locked, require passcode to unlock |

---

## UI Errors

### Auto Layout Constraint Conflicts
```
Log: "Unable to simultaneously satisfy constraints"
Cause: Two or more constraints contradict each other
Debug: Read the constraint list in the log, identify the conflicting pair
Fix: Set constraint priorities (<1000 for breakable), remove contradicting constraint
Tool: Xcode Debug View Hierarchy, or https://wtfautolayout.com to parse the log

Log: "Will attempt to recover by breaking constraint"
Cause: Same as above -- iOS broke a constraint to resolve the conflict
Impact: Layout may appear wrong but won't crash
Fix: Fix the constraint conflict so iOS doesn't have to guess
```

### View Lifecycle
```
Log: "Presenting view controller <X> on <Y> whose view is not in the window hierarchy"
Cause: Presenting a modal from a VC that isn't visible
Fix: Ensure the presenting VC is in the window hierarchy, use topmost VC

Log: "Unbalanced calls to begin/end appearance transitions for <X>"
Cause: Multiple presentations/dismissals overlapping
Fix: Wait for animations to complete before presenting again, use completion handlers

Log: "Warning: Attempt to present <X> on <Y> which is already presenting <Z>"
Cause: Trying to present when another modal is already presented
Fix: Dismiss the existing presentation first, or present from the already-presented VC
```

### Navigation Stack Corruption
```
Log: "Finishing up a navigation transition in an unexpected state"
Cause: Pushing/popping during an existing animation
Fix: Use completion handlers, avoid rapid push/pop sequences

Log: "Popped to a view controller that wasn't in the navigation stack"
Cause: popToViewController with a VC not in the stack
Fix: Check the navigation stack before popping: navigationController?.viewControllers.contains(targetVC)
```

---

## App Lifecycle Errors

```
Log: "BKSApplicationStateMonitor: ... application has been killed"
Cause: System terminated the app (memory pressure, background time expired)
Severity: Expected behavior, not a bug (unless happening frequently)

Log: "Received memory warning"
Cause: System is low on memory, app should release caches
Fix: Implement didReceiveMemoryWarning, clear image caches and non-essential data

Log: "Background task expired"
Cause: Background execution time (30s for push, ~3min for beginBackgroundTask) elapsed
Fix: Complete work faster, or use BGProcessingTask for long-running work
```

# How to Read Swift Stack Traces

A practical guide to reading and interpreting iOS/Swift stack traces from Sentry and crash reports.

---

## Stack Trace Anatomy

A typical Swift crash stack trace:

```
Thread 0 Crashed:
0   libswiftCore.dylib          0x00000001a2b3c4d5 swift_fatalError + 56
1   libswiftCore.dylib          0x00000001a2b3c4a0 swift_preconditionFailure + 24
2   libswiftCore.dylib          0x00000001a2b1a230 Array._checkSubscript + 88
3   MyApp                       0x0000000104abc123 TransactionService.processTransactions(_:) + 456
4   MyApp                       0x0000000104abd789 TransactionViewModel.loadData() + 234
5   MyApp                       0x0000000104abe000 closure #1 in TransactionView.body.getter + 128
6   SwiftUI                     0x00000001b5678900 ViewBody.update + 64
7   SwiftUI                     0x00000001b5679abc AttributeGraph.updateValue + 128
...
15  UIKitCore                   0x00000001a9876543 UIApplicationMain + 132
```

### Reading Each Frame

| Component | Meaning |
|-----------|---------|
| Frame number (0, 1, 2...) | Order of the call stack. Frame 0 = where the crash occurred |
| Module name | Binary that contains the code. Your code = your app name |
| Address (0x...) | Memory address of the instruction |
| Function name | Demangled function name (Swift or Obj-C) |
| Offset (+ 456) | Byte offset from function start. Higher = deeper into the function |

### Your Code vs System Code

| Module | Source |
|--------|--------|
| `MyApp` (your app name) | **YOUR CODE** -- focus here |
| `libswiftCore.dylib` | Swift standard library (Array, String, Optional) |
| `SwiftUI` | SwiftUI framework |
| `UIKitCore` | UIKit framework |
| `libdispatch.dylib` | GCD / dispatch queues |
| `libswift_Concurrency.dylib` | Swift concurrency runtime |
| `CoreData` | Core Data framework |
| `Foundation` | Foundation framework |

**Strategy:** Start at frame 0 and scan down until you find YOUR code. That's where the bug is.

---

## Symbolication

### What Is Symbolication?

Raw crash reports contain only memory addresses. Symbolication replaces those addresses with function names, file names, and line numbers.

```
// Unsymbolicated (useless):
3   MyApp   0x0000000104abc123 0x104a00000 + 769315

// Symbolicated (useful):
3   MyApp   0x0000000104abc123 TransactionService.processTransactions(_:) at TransactionService.swift:42
```

### Why Crashes May Be Unsymbolicated

- dSYM file not uploaded to Sentry
- dSYM UUID mismatch (built with different settings)
- Bitcode recompilation changed addresses (legacy, pre-Xcode 14)
- Release build without debug symbols

### How to Re-Symbolicate

```bash
# 1. Find the dSYM for the crash's build
# The dSYM UUID must match the binary UUID in the crash report
dwarfdump --uuid MyApp.app.dSYM

# 2. Upload to Sentry
sentry-cli upload-dif MyApp.app.dSYM

# 3. Or symbolicate locally with atos
atos -arch arm64 -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp -l 0x104a00000 0x0000000104abc123
```

---

## Swift Name Demangling

Swift uses name mangling to encode type information in symbol names. Sentry usually demangles automatically, but raw symbols look like this:

```
$s5MyApp18TransactionServiceC19processTransactionsySay0D0VGF
```

### Decoding Mangled Names

```bash
# Use swift-demangle
echo '$s5MyApp18TransactionServiceC19processTransactionsySay0D0VGF' | swift demangle
# Output: MyApp.TransactionService.processTransactions([MyApp.Transaction]) -> ()
```

### Common Mangled Prefixes

| Prefix | Meaning |
|--------|---------|
| `$s` | Swift symbol |
| `$S` | Swift symbol (older convention) |
| `_$s` | Swift symbol with underscore prefix |
| `$sSS` | Swift.String |
| `$sSa` | Swift.Array |
| `$sSi` | Swift.Int |
| `$sSb` | Swift.Bool |
| `$sSd` | Swift.Double |

### Understanding Generic Parameters in Mangled Names

```
$s5MyApp10RepositoryC5fetchyx_tSay0A0VGRQ lF
                                    ^^^^
                                    Generic parameter constraint
```

Use `swift demangle` to decode -- don't try to read these manually.

---

## Identifying Your Code

In a long stack trace, quickly find your frames:

1. **Search for your module name** (your app name or framework names)
2. **Ignore system frames** above and below your code
3. **The topmost frame of YOUR code** is usually where the bug manifests
4. **Lower frames of YOUR code** show how you got there

```
0   libswiftCore.dylib   swift_fatalError                    <- System: the crash mechanism
1   libswiftCore.dylib   Array._checkSubscript               <- System: bounds check failed
2   MyApp                TransactionService.process(:)        <- YOUR CODE: where the bug is
3   MyApp                TransactionViewModel.loadData()      <- YOUR CODE: what called it
4   SwiftUI              ViewBody.update                      <- System: SwiftUI lifecycle
```

Focus your investigation on frames 2 and 3.

---

## Reading Async Stack Traces

Swift concurrency makes stack traces more complex. Look for these patterns:

### Continuation Frames

```
Thread 1:
0   libswiftCore.dylib           swift_fatalError
1   MyApp                        TransactionService.fetchBalance() async  <- async function
2   libswift_Concurrency.dylib   swift_task_future_wait                    <- await point
3   libswift_Concurrency.dylib   swift_task_switch                         <- task switch
4   libswift_Concurrency.dylib   swift_task_enqueue_global                 <- task scheduling
```

The actual call site may not appear in the same thread's stack. Check the async backtrace if Sentry provides it.

### Actor-Isolated Crashes

```
Thread 3:
0   libswiftCore.dylib           swift_fatalError
1   libswift_Concurrency.dylib   swift_task_assertIsOnActorExecutor  <- Actor violation
2   MyApp                        MainActor.run                        <- Expected main actor
3   MyApp                        ViewModel.updateUI()                 <- Your code
```

This tells you `updateUI()` expected to be on MainActor but was called from a different context.

### Task Group and Structured Concurrency

```
Thread 5:
0   MyApp                        DataLoader.load() async
1   libswift_Concurrency.dylib   swift_taskGroup_addPending
2   MyApp                        DashboardService.loadAll() async
```

For task groups, each child task gets its own stack. Look at the parent task's stack to understand context.

---

## Common Crash Signatures Quick Reference

| Signature | Likely Cause |
|-----------|-------------|
| `swift_fatalError` + `Array._checkSubscript` | Index out of range |
| `swift_fatalError` + `Optional.unwrap` | Force unwrap nil |
| `swift_dynamicCast` | Forced downcast (`as!`) failure |
| `swift_task_assertIsOnActorExecutor` | Actor isolation violation |
| `objc_release` + `swift_release` | Use-after-free / over-release |
| `malloc_error_break` | Heap corruption |
| `_dispatch_barrier_sync_f_slow` | Deadlock on dispatch queue |
| `JSONDecoder.decode` + `DecodingError` | JSON parsing failure |
| `NSKeyedUnarchiver` + `decodeObject` | Archive corruption |
| `0x8badf00d` in termination reason | Watchdog timeout |

---

## Tips for Sentry-Specific Stack Traces

1. **Check "Raw" vs "Formatted" view** -- Sentry sometimes groups frames. The raw view shows everything.
2. **Look at the "Exception" section** -- it contains the error message, which is often more useful than the stack trace alone.
3. **Check "In-App" filter** -- Sentry can filter to show only your frames. Useful for long traces.
4. **Check "Breadcrumbs" tab** -- shows what happened before the crash (user actions, network calls, console logs).
5. **Check "Tags"** -- OS version, device model, app version. Crashes may be device/OS specific.
6. **Check "Additional Data"** -- custom context your app set via `Sentry.configureScope`.

---
name: prometheus-performance
description: "App is slow, takes too long to load, freezing, laggy, battery drain, memory leak. iOS performance profiling and optimization — launch time, memory analysis, retain cycles, CPU profiling, frame rate. Trigger on: 'why is this screen slow', 'app uses too much memory', 'scrolling is janky', 'UI is unresponsive', 'profile with Instruments', 'energy usage is high', 'app got killed in background', or any complaint about speed, responsiveness, or resource usage. For performance crashes (OOM, watchdog) appearing in Sentry, start with sentry-find-bugs."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: false
---

# Performance Optimization & Profiling

> **Iron Law**: "Measure before optimizing. Premature optimization without profiling data is guessing."

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

Profile app launch time, analyze memory usage, detect retain cycles, optimize CPU and main thread performance, and enforce performance thresholds. Target: sub-second response times, < 3s cold launch, < 100MB memory footprint.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|-----------------|----------------|-----------------|
| "The app feels fast enough" | Human perception adapts. Users notice degradation relative to other apps, not absolute speed. A 200ms delay feels fine until a competitor does it in 50ms. | Measure with Instruments. Compare against thresholds, not feelings. |
| "Users won't notice" | Users notice 100ms+ delays subconsciously. Apple reports that every 100ms of launch time improvement increases user retention. | Profile the actual user-facing latency. Even "unnoticeable" delays compound. |
| "We'll optimize later" | Performance debt compounds faster than tech debt. A slow screen with 3 callers becomes a slow screen with 30 callers, and now the fix requires rearchitecting. | Set performance budgets at feature inception. Measure during development, not after. |
| "It's only slow on old devices" | Old devices are real users' devices. iPhone SE, iPad Air 2, and 3-year-old phones are a significant portion of the install base. | Test on the oldest supported device. Optimize for the floor, not the ceiling. |
| "The profiler is too complex to learn" | Instruments has steep learning curve but is the only way to identify real bottlenecks. Guessing which code is slow is wrong more often than right. | See `references/instruments-guide.md` for step-by-step workflows. Start with Time Profiler. |
| "It's a debug build, of course it's slow" | Debug builds have different characteristics but can still reveal algorithmic problems, N+1 queries, and main thread violations. | Profile release builds for final measurements. Use debug builds to find structural issues. |
| "Adding a cache will fix it" | Caches add complexity, stale data bugs, and memory pressure. The real question is: why is the operation slow in the first place? | Profile first. Fix the root cause. Add caching only if the operation is inherently expensive AND frequently repeated. |

---

## Red Flags -- HARD STOP

These are not warnings. Each is a BLOCKING condition that prevents further progress until addressed.

- **BLOCKED: Optimizing without profiling data** -- if you haven't run Instruments or measured with concrete numbers, you're guessing. STOP. The bottleneck is rarely where you think it is. Run Instruments first (see references/instruments-guide.md).
- **BLOCKED: Optimizing the wrong layer** -- CPU optimization when the bottleneck is network latency. Memory optimization when the bottleneck is main thread blocking. STOP. Match the solution to the MEASURED problem, not the assumed one.
- **BLOCKED: Ignoring memory growth over time** -- a 10MB/minute memory leak is invisible in a 2-minute test session but crashes the app after 20 minutes of real usage. STOP. Profile memory over an extended session.
- **BLOCKED: Main thread blocking for any I/O, network, or crypto operation** -- this causes visible UI freezes and watchdog kills. There are zero acceptable exceptions. STOP. Move to background immediately.
- **BLOCKED: Adding `DispatchQueue.main.async` without understanding why** -- this is often a band-aid for a deeper architectural problem. STOP. Understand the threading model before dispatching. Document why the dispatch is needed.
- **BLOCKED: Cargo-culting `.lazy` or `LazyVStack` without measuring** -- lazy views add complexity and can cause their own performance issues (cell reuse bugs, state management). STOP. Measure BEFORE and AFTER to confirm they help.

---

## When NOT to Use This Skill

- **The performance issue is actually a crash** -- if the app crashes from OOM or watchdog timeout and appears in Sentry, start with `sentry-find-bugs` to analyze the crash, then return here for optimization.
- **Security review of performance-related code** -- if the concern is whether a caching layer introduces security issues, use `cipher-security` or `trailofbits-differential-review`.
- **The code hasn't been written yet** -- performance optimization applies to existing code. For new features, design with performance budgets in mind but don't optimize prematurely.
- **The bottleneck is server-side** -- if profiling shows the client is waiting on API responses, the optimization needs to happen on the backend, not in the iOS app.

---

## Behavioral Enforcement

### Phase Gate: MEASUREMENT (Before ANY Optimization)
**Cannot propose ANY optimization until:**
- [ ] Current performance MEASURED (not assumed)
- [ ] Bottleneck IDENTIFIED with profiling data (not guessed)
- [ ] Baseline metrics recorded (so improvement can be verified)

**Hard Stop**: If about to optimize without profiling data -- STOP. You are guessing. Premature optimization based on intuition wastes time optimizing the wrong thing. Run Instruments (see references/instruments-guide.md) first.

**Counter**: Optimization #1 didn't improve the metric? Understand why before #2. At #3 -- question whether you identified the right bottleneck. Re-profile.

### Phase Gate: VERIFICATION (After Optimization)
**Cannot declare optimization complete until:**
- [ ] Performance re-measured with same methodology as baseline
- [ ] Improvement quantified: "Before: X, After: Y, Delta: Z"
- [ ] No regressions in other metrics (faster but uses 10x memory = not an optimization)

**Required evidence**: Before/after numbers. "It feels faster" is not evidence.

### Counter-Based Escalation
- **Optimization #1**: Normal. Implement, measure, verify.
- **Optimization #2**: Pause. Why didn't #1 achieve the target? Write down what you learned. Is the bottleneck where you think it is?
- **Optimization #3**: STOP. Re-profile from scratch. You may be optimizing the wrong thing. Consider: is this an architectural issue that can't be solved with local optimization?
- **Still not meeting target after #3**: ESCALATE to human partner with: baseline measurements, optimizations tried, measurements after each attempt, current profile data.

---

## Decision Framework

```
User reports performance issue
|
+-- What's the symptom?
|
+-- APP LAUNCH IS SLOW (> 3s cold)
|   +-- Profile with App Launch template (see references/instruments-guide.md)
|   +-- Pre-main time high (> 1s)?
|   |   +-- Too many dylibs? Reduce dynamic frameworks, use static linking
|   |   +-- Too many ObjC classes? Reduce +load methods, lazy initialization
|   +-- Post-main time high?
|       +-- Heavy work in didFinishLaunching? Defer non-critical initialization
|       +-- Synchronous network calls? Make async, show skeleton UI
|       +-- Core Data migration? Run in background with progress indicator
|
+-- SCREEN LOADS SLOWLY (> 500ms)
|   +-- Profile with Time Profiler
|   +-- Main thread blocked?
|   |   +-- Move computation to background actor/queue
|   |   +-- Use async/await for data loading
|   +-- Too many views?
|   |   +-- Use LazyVStack/LazyHStack (measure first)
|   |   +-- Reduce view hierarchy depth
|   +-- Heavy image loading?
|       +-- Downsample images to display size (see references/optimization-cookbook.md)
|       +-- Use AsyncImage with placeholder
|
+-- SCROLLING IS JANKY (< 60 FPS)
|   +-- Profile with Time Profiler, check main thread
|   +-- Cell configuration expensive?
|   |   +-- Precompute values in ViewModel
|   |   +-- Cache formatted strings, attributed text
|   +-- Images not sized correctly?
|   |   +-- Downsample to cell size
|   |   +-- Use thumbnail cache
|   +-- Shadows, corner radius, transparency?
|       +-- Rasterize layers that don't change
|       +-- Use pre-rendered shadow images
|
+-- MEMORY KEEPS GROWING
|   +-- Profile with Allocations or Leaks template
|   +-- Retain cycle?
|   |   +-- Check closures for [weak self]
|   |   +-- Check delegate references
|   |   +-- Use Debug Memory Graph in Xcode
|   +-- Large allocations?
|   |   +-- Image buffers not released?
|   |   +-- Data buffers growing unbounded?
|   +-- Persistent growth without leaks?
|       +-- Cache not bounded?
|       +-- Notification observers not removed?
|       +-- Timer not invalidated?
|
+-- APP USES TOO MUCH ENERGY
|   +-- Profile with Energy Log template
|   +-- Excessive location updates?
|   +-- Background networking too frequent?
|   +-- Timer-based polling? (Use push notifications instead)
|   +-- CPU work during idle? (Reduce timer frequency, use reactive patterns)
|
+-- NETWORK IS SLOW
    +-- Profile with Network template
    +-- Too many sequential requests?
    |   +-- Use TaskGroup for parallel requests
    +-- Large payloads?
    |   +-- Paginate, compress, reduce response size
    +-- Missing caching?
        +-- Configure URLCache
        +-- Implement conditional requests (ETags, If-Modified-Since)
```

---

## Performance Thresholds

| Metric | Target | Acceptable | Critical | How to Measure |
|--------|--------|------------|----------|----------------|
| Cold Launch | < 2s | < 3s | > 5s | Instruments App Launch template |
| Warm Launch | < 500ms | < 1s | > 2s | Instruments App Launch template |
| Screen Load | < 300ms | < 500ms | > 1s | Time Profiler + signpost markers |
| API Response | < 200ms | < 500ms | > 1s | Network template + URLSession metrics |
| Memory Usage | < 80MB | < 100MB | > 200MB | Allocations template |
| Frame Rate | 60 FPS | 45 FPS | < 30 FPS | Core Animation template |
| Time to Interactive | < 1s | < 2s | > 3s | Custom signpost markers |
| Image Load | < 100ms | < 200ms | > 500ms | Time Profiler |
| Energy Impact | Low | Medium | High | Energy Log template |

### Setting Performance Budgets

For each screen, define a budget BEFORE development:

```
Screen: Transaction History
- Load time budget: 300ms (fetch + decode + render)
- Memory budget: 20MB (including image cache)
- Frame rate budget: 60 FPS during scroll
- Network budget: 1 request for initial load, pagination for scroll

Breakdown:
- API fetch: 150ms
- JSON decode: 30ms
- View model mapping: 20ms
- View rendering: 100ms
```

---

## Standard Operating Procedures

### SOP-1: Profile App Launch

See `references/instruments-guide.md` for detailed step-by-step Instruments workflows.

```bash
# Use Xcode Instruments (substitute actual device UDID)
xcrun xctrace record --device <device-udid> \
  --template 'App Launch' \
  --launch {BUNDLE_ID} \
  --output launch_profile.trace

# Analyze results
xcrun xctrace export --input launch_profile.trace --output launch_data
```

### SOP-2: Memory Analysis
```swift
// In-app memory tracking
#if DEBUG
func reportMemoryUsage() {
    var info = mach_task_basic_info()
    var count = mach_msg_type_number_t(MemoryLayout<mach_task_basic_info>.size)/4
    let result = withUnsafeMutablePointer(to: &info) {
        $0.withMemoryRebound(to: integer_t.self, capacity: 1) {
            task_info(mach_task_self_, task_flavor_t(MACH_TASK_BASIC_INFO), $0, &count)
        }
    }
    if result == KERN_SUCCESS {
        let memoryMB = info.resident_size / 1024 / 1024
        logger.debug("memory_usage", ["resident_mb": memoryMB])
    }
}
#endif
```

### SOP-3: Detect Retain Cycles
```swift
// Common retain cycle patterns to watch for:

// PATTERN 1: Closures capturing self strongly
// BAD -- strong reference cycle if closure is stored
class ViewModel: ObservableObject {
    var onUpdate: (() -> Void)?

    func setup() {
        onUpdate = { self.refresh() } // Retains self
    }
}

// GOOD -- use [weak self] for stored closures
class ViewModel: ObservableObject {
    var onUpdate: (() -> Void)?

    func setup() {
        onUpdate = { [weak self] in self?.refresh() }
    }
}

// PATTERN 2: Prefer async/await over closures to avoid cycles entirely
@MainActor
final class ViewModel: ObservableObject {
    func loadData() async {
        let result = try? await service.fetch()
        handleResult(result) // No closure, no capture, no cycle risk
    }
}

// PATTERN 3: Delegate retain cycles
// BAD -- strong delegate reference
protocol DataDelegate: AnyObject {
    func didUpdate(_ data: [Item])
}

class DataManager {
    var delegate: DataDelegate? // STRONG reference -- will retain the delegate
}

// GOOD -- weak delegate
class DataManager {
    weak var delegate: DataDelegate?
}

// PATTERN 4: Timer retain cycles
// BAD -- Timer retains target strongly
class PollingService {
    var timer: Timer?

    func startPolling() {
        timer = Timer.scheduledTimer(
            timeInterval: 30,
            target: self, // Timer retains self strongly
            selector: #selector(poll),
            userInfo: nil,
            repeats: true
        )
    }
}

// GOOD -- use closure-based timer with [weak self]
class PollingService {
    var timer: Timer?

    func startPolling() {
        timer = Timer.scheduledTimer(withTimeInterval: 30, repeats: true) { [weak self] _ in
            self?.poll()
        }
    }

    deinit {
        timer?.invalidate()
    }
}

// PATTERN 5: Check for cycles with Instruments
// Profile -> Leaks instrument -> Look for retained objects
// Or use Debug Memory Graph in Xcode (Debug -> Debug Memory Graph)
```

### SOP-4: Main Thread Checker
```bash
# Enable in scheme
# Edit Scheme -> Run -> Diagnostics -> Main Thread Checker

# Or via environment variable
XCODE_THREAD_SANITIZER_ENABLED=YES
```

### SOP-5: SwiftUI View Performance
```swift
// Identifying unnecessary redraws
// Add this to any view to track re-renders:
#if DEBUG
let _ = Self._printChanges() // Prints which property triggered the redraw
#endif

// Use .equatable() for views with stable content
struct ItemRow: View, Equatable {
    let item: Item

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.item.id == rhs.item.id && lhs.item.updatedAt == rhs.item.updatedAt
    }

    var body: some View {
        HStack {
            Text(item.name)
            Spacer()
            Text(item.amount.formatted(.currency(code: "USD")))
        }
    }
}

// In parent:
ForEach(items) { item in
    ItemRow(item: item)
        .equatable() // Only redraws when Equatable comparison changes
}
```

---

## Instruments Workflow

See `references/instruments-guide.md` for the complete guide. Quick reference:

| Problem | Template | What to Look For |
|---------|----------|-----------------|
| App launch slow | App Launch | Pre-main vs post-main time split |
| Screen load slow | Time Profiler | Hot spots in call tree, main thread work |
| Scrolling janky | Time Profiler | Main thread work during scroll |
| Memory growing | Allocations | Persistent vs transient allocations |
| Memory leaking | Leaks | Leaked objects, retain cycle chains |
| Battery drain | Energy Log | CPU, network, location, display energy |
| Network slow | Network | Request waterfall, payload sizes |
| Animations dropping | Core Animation | Offscreen rendering, blending |

---

## Optimization Strategies

See `references/optimization-cookbook.md` for 20+ specific optimization patterns with before/after metrics. Key strategies:

### Image Optimization
```swift
// PROBLEM: Loading a 12MB image for a 44x44pt thumbnail
// Before: 12MB memory per thumbnail, 200ms decode time

// SOLUTION: Downsample to display size
func downsample(imageAt url: URL, to pointSize: CGSize, scale: CGFloat) -> UIImage? {
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    guard let imageSource = CGImageSourceCreateWithURL(url as CFURL, imageSourceOptions) else {
        return nil
    }

    let maxDimensionInPixels = max(pointSize.width, pointSize.height) * scale
    let downsampleOptions = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceShouldCacheImmediately: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxDimensionInPixels
    ] as CFDictionary

    guard let downsampledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions) else {
        return nil
    }

    return UIImage(cgImage: downsampledImage)
}

// After: 0.1MB memory per thumbnail, 10ms decode time (90%+ reduction)
```

### List Performance
```swift
// PROBLEM: VStack with 1000 items renders all at once
// Before: 16s initial render, 800MB memory

// SOLUTION: LazyVStack only renders visible items
LazyVStack {
    ForEach(items) { item in
        ItemRow(item: item)
    }
}
// After: 0.3s initial render, 30MB memory
```

### Network Optimization
```swift
// PROBLEM: 10 sequential API calls
// Before: 5s total (500ms each, sequential)

// SOLUTION: Parallel with TaskGroup
func loadDashboard() async throws -> Dashboard {
    async let user = api.fetchUser()
    async let balance = api.fetchBalance()
    async let transactions = api.fetchRecentTransactions()
    async let notifications = api.fetchNotifications()

    let (u, b, t, n) = try await (user, balance, transactions, notifications)
    return Dashboard(user: u, balance: b, transactions: t, notifications: n)
}
// After: 0.6s total (parallel, bounded by slowest call)

// Cache responses
let cache = URLCache(
    memoryCapacity: 50_000_000,  // 50 MB memory
    diskCapacity: 100_000_000    // 100 MB disk
)
let config = URLSessionConfiguration.default
config.urlCache = cache
config.requestCachePolicy = .returnCacheDataElseLoad
```

### Precomputation
```swift
// PROBLEM: Expensive computation in SwiftUI body (runs on every redraw)
// Before: formatCurrency() called 60x/sec during scroll

// SOLUTION: Precompute in ViewModel
@MainActor
final class TransactionViewModel: ObservableObject {
    @Published private(set) var formattedAmount: String = ""
    @Published private(set) var formattedDate: String = ""

    func configure(with transaction: Transaction) {
        // Compute once, not on every frame
        formattedAmount = transaction.amount.formatted(.currency(code: transaction.currencyCode))
        formattedDate = transaction.date.formatted(.dateTime.month().day().year())
    }
}
// After: Computed once per data update, not once per frame
```

---

## Energy Profiling

### High Energy Impact Sources

| Source | Energy Cost | Optimization |
|--------|------------|--------------|
| Continuous location (GPS) | Very High | Use significant-change service, reduce accuracy when possible |
| Frequent network polling | High | Use push notifications, WebSocket, or server-sent events |
| Background processing | High | Use BGTaskScheduler with appropriate intervals |
| Continuous animations | Medium | Pause animations when off-screen, reduce frame rate when possible |
| Bluetooth scanning | Medium | Scan only when needed, stop when found |
| CPU-intensive computation | Medium | Offload to background, batch operations |

```swift
// BAD: Polling every 5 seconds
Timer.scheduledTimer(withTimeInterval: 5, repeats: true) { _ in
    Task { await self.refreshData() }
}

// GOOD: Use push notifications for real-time updates
// Or WebSocket for bidirectional communication
// Or increase polling interval with exponential backoff
func scheduleRefresh() {
    let request = BGAppRefreshTaskRequest(identifier: "com.app.refresh")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60) // 15 min minimum
    try? BGTaskScheduler.shared.submit(request)
}
```

---

## App Size Optimization

| Technique | Typical Savings | How |
|-----------|----------------|-----|
| Asset catalog optimization | 10-30% | Use asset catalogs (not loose files), enable app thinning |
| Dead code elimination | 5-15% | Enable `-dead_strip` in release builds (default in Xcode) |
| Framework thinning | 10-40% | Use static linking instead of dynamic frameworks |
| Image format optimization | 20-50% per image | Use HEIF/HEIC for photos, SVG/PDF for icons, WebP for web content |
| On-demand resources | Variable | Move non-essential assets to ODR |
| Bitcode (legacy) | N/A | Deprecated as of Xcode 14 |

```bash
# Analyze app size
xcrun ipa-analysis --input MyApp.ipa

# Check for unused resources
# Look for image assets not referenced in code or storyboards
find . -name "*.imageset" | while read dir; do
  name=$(basename "$dir" .imageset)
  count=$(grep -rl "$name" --include="*.swift" --include="*.storyboard" --include="*.xib" . | wc -l)
  if [ "$count" -eq 0 ]; then
    echo "Potentially unused: $name"
  fi
done
```

---

## Quality Gates (Before Marking Complete)

- [ ] Performance issue measured with concrete numbers BEFORE optimization (baseline established)
- [ ] Root cause identified with profiling data (not guessed)
- [ ] Optimization targets the actual bottleneck (not the assumed one)
- [ ] Performance measured AFTER optimization with same methodology (improvement quantified)
- [ ] Measurements taken on release build (not debug)
- [ ] Tested on oldest supported device (not just latest hardware)
- [ ] Memory profile shows no new leaks or unbounded growth
- [ ] Main thread remains unblocked (Time Profiler verification)
- [ ] No regression in other metrics (fixing CPU shouldn't increase memory)
- [ ] Performance budget documented for the optimized screen/flow

---

## Required Output Artifact

Every execution of this skill MUST produce a structured performance report:

```
## Performance Optimization Report
**Date**: [date]
**Target**: [screen/flow/operation optimized]
**Device**: [device model and OS version used for profiling]
**Build**: [debug/release]

### Baseline Measurements
| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| [metric] | [measured value] | [target threshold] | [PASS/FAIL] |

### Bottleneck Identified
- **What**: [specific operation/function]
- **Evidence**: [Instruments template used, what the profile showed]
- **Why it's the bottleneck**: [explanation with data]

### Optimization Applied
- **Change**: [what was changed]
- **File(s)**: [paths]
- **Approach**: [strategy used]

### Post-Optimization Measurements
| Metric | Before | After | Delta | Target | Status |
|--------|--------|-------|-------|--------|--------|
| [metric] | [before] | [after] | [improvement] | [target] | [PASS/FAIL] |

### Regression Check
- [ ] Memory: no increase
- [ ] CPU: no increase in other paths
- [ ] Frame rate: no drops
- [ ] Launch time: no regression

### Optimization Attempt #
[1/2/3 -- if 2+, document what was learned from previous attempts]
```

**Incomplete reports are not accepted.** Before/after numbers are mandatory.

---

## Self-Audit Protocol

Before declaring this skill's execution complete, verify ALL of the following:

- [ ] **Phase Gate: Measurement** -- Did you measure BEFORE optimizing? Can you quote the baseline number?
- [ ] **Phase Gate: Verification** -- Did you measure AFTER optimizing with the same methodology? Can you quote the improvement?
- [ ] **Hard Stops checked** -- Did you profile before optimizing? Are you optimizing the measured bottleneck (not the assumed one)?
- [ ] **Release build** -- Were final measurements taken on a release build (not debug)?
- [ ] **Oldest device** -- Was performance verified on the oldest supported device?
- [ ] **No regressions** -- Did you check that other metrics didn't regress? (Faster CPU but 10x memory = not an optimization.)
- [ ] **Output artifact produced** -- Is the performance report complete with before/after numbers?
- [ ] **Counter tracking** -- If this is optimization attempt #2+, did you document what you learned from previous attempts?

**If any checkbox is unchecked, you are not done.** Go back and complete it.

---

## Cross-Skill References

- `sentry-find-bugs` -- When performance issues cause crashes (OOM, watchdog kills) that appear in Sentry
- `cipher-security` -- When security operations (encryption, hashing, certificate validation) are the performance bottleneck
- `trailofbits-differential-review` -- When performance optimization changes touch security-sensitive code
- `forge-development` -- For implementing performance optimizations following project patterns
- `ios-review` -- For reviewing performance-related code changes

---

## References

- Instruments step-by-step guide: `references/instruments-guide.md`
- Optimization patterns with metrics: `references/optimization-cookbook.md`
- [WWDC: Analyze and Improve App Performance](https://developer.apple.com/videos/play/wwdc2023/10160/)
- [Apple Performance Best Practices](https://developer.apple.com/documentation/xcode/improving-your-app-s-performance)
- [Instruments Help](https://help.apple.com/instruments/)

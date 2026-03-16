# Instruments Step-by-Step Guide

Which Instruments template to use for which problem, and how to read the results.

---

## Template Selection Guide

| Problem | Template | When to Use |
|---------|----------|-------------|
| App launches slowly | **App Launch** | Cold/warm launch takes > 2-3 seconds |
| Screen loads slowly | **Time Profiler** | Specific screen transition is slow |
| Scrolling is janky | **Time Profiler** | Frame drops during scroll (< 60 FPS) |
| Memory keeps growing | **Allocations** | Memory usage increases over time |
| Memory leaks | **Leaks** | Objects never deallocated, retain cycles |
| Battery drains fast | **Energy Log** | Users report excessive battery usage |
| Network is slow | **Network** | API calls take too long or too many requests |
| Animations stutter | **Core Animation** | Visual glitches, dropped frames, off-screen rendering |
| Disk I/O issues | **File Activity** | Excessive file reads/writes, slow storage |
| Hang/freeze | **Time Profiler** (or **System Trace**) | App becomes unresponsive |

---

## Time Profiler

### When to Use
CPU-bound performance issues: slow screen loads, janky scrolling, expensive computations, main thread blocking.

### Step-by-Step Workflow

1. **Open Instruments** (Xcode -> Open Developer Tool -> Instruments, or Cmd+I from Xcode)
2. **Select Time Profiler** template
3. **Choose your target** (app + device/simulator)
4. **Configure recording settings:**
   - Recording Mode: Deferred (less overhead)
   - Sample rate: 1ms for detailed, 10ms for overview
5. **Start recording** (red record button)
6. **Reproduce the slow operation** in the app
7. **Stop recording**

### Reading the Call Tree

The call tree shows where CPU time was spent. Key columns:

| Column | Meaning |
|--------|---------|
| **Weight** | Total time spent in this function AND its callees |
| **Self Weight** | Time spent in this function ONLY (excluding callees) |
| **Symbol Name** | Function name |

**Strategy:**
- Sort by **Self Weight** to find the functions that are individually expensive
- Sort by **Weight** to find the call chains that are expensive overall
- Focus on YOUR code (uncheck "System Libraries" to hide framework code)

### Finding Hot Spots

1. **Expand the heaviest call stack** by clicking the disclosure triangles
2. **Look for your code** in the heaviest path
3. **Option-click** a disclosure triangle to expand the entire subtree
4. **Right-click -> Reveal in Xcode** to jump to the code

### Main Thread Analysis

1. In the thread list, select **Thread 1 (Main Thread)**
2. Look for heavy work on the main thread -- this causes UI freezes
3. Anything > 16ms on the main thread can cause a dropped frame at 60 FPS
4. Common culprits: JSON parsing, image decoding, Core Data fetches, string formatting

### Example Findings

```
PROBLEM: Main thread blocked for 200ms during cell configuration
Weight    Self    Symbol
200ms     0ms     TransactionListView.body.getter
  180ms   0ms     ForEach.init
    170ms 0ms     TransactionRow.body.getter
      160ms 150ms   formatCurrency(_:)  <-- HOT SPOT: 150ms of self time

FIX: Precompute formatted strings in ViewModel, not in view body
```

---

## Allocations

### When to Use
Memory growth over time, high memory usage, finding what's using memory.

### Step-by-Step Workflow

1. **Select Allocations** template in Instruments
2. **Start recording**
3. **Use the app normally** for 2-5 minutes, focusing on the suspect flow
4. **Stop recording**

### Reading the Results

#### Statistics View (default)
Shows all allocation categories. Sort by:
- **Persistent Bytes**: Memory currently in use (not freed)
- **# Persistent**: Count of live objects
- **# Transient**: Objects that were allocated and freed (normal)

#### Generation Analysis (Key Technique)
This is the most powerful technique for finding memory growth:

1. Click **"Mark Generation"** button (flag icon) at a baseline point
2. Perform an action in the app (e.g., navigate to a screen and back)
3. Click **"Mark Generation"** again
4. The "Growth" column shows memory allocated between marks that was NOT freed
5. If growth > 0 after returning to the same state, you have a leak or unbounded cache

```
Generation A -> Generation B
Growth: +2.3 MB  <-- This memory was allocated during the action but not freed

Drill into the growth to see WHAT was allocated:
- 1.8 MB: UIImage (not downsampled for display size)
- 0.3 MB: Data (API response not released)
- 0.2 MB: String (formatted strings cached but never evicted)
```

### Finding the Allocation Source

1. Click on a large allocation category
2. Switch to **"Call Tree"** view
3. Expand to find YOUR code that triggered the allocation
4. Right-click -> **"Reveal in Xcode"** to jump to the source

---

## Leaks

### When to Use
Suspected retain cycles, objects that are never deallocated.

### Step-by-Step Workflow

1. **Select Leaks** template in Instruments
2. **Start recording**
3. **Perform the suspect operation** (e.g., present and dismiss a view controller)
4. **Wait 10 seconds** (Leaks instrument checks periodically)
5. Look for leak indicators (red dots in the timeline)
6. **Stop recording**

### Interpreting Leak Reports

Each leak shows:
- **Leaked Object**: The type of object that leaked
- **Size**: Memory consumed by the leak
- **Responsible Frame**: The code that allocated the leaked object
- **Reference Count**: Usually shows a retain cycle path

### Common Leak Patterns in Swift

| Pattern | Detection | Fix |
|---------|-----------|-----|
| Closure retaining self | Leak shows your class and a closure block | Add `[weak self]` to closure capture list |
| Delegate retain cycle | Leak shows two objects retaining each other | Make delegate property `weak` |
| Timer retaining target | Leak shows Timer + your class | Use closure-based timer with `[weak self]`, invalidate in deinit |
| NotificationCenter observer | Leak shows your class held by NotificationCenter | Remove observer in deinit, or use Combine-based observation |
| KVO observation | Leak shows observed object retaining observer | Use `observe(_:options:changeHandler:)` which returns auto-invalidating token |

### Debug Memory Graph (Alternative)

In Xcode (not Instruments):
1. Run the app in debug mode
2. Reproduce the suspected leak
3. Click the **memory graph** button in the debug bar (between view hierarchy and location buttons)
4. Look for objects with **purple exclamation marks** (runtime-detected leaks)
5. Click an object to see its reference graph -- follow the arrows to find the retain cycle

---

## App Launch

### When to Use
App takes > 2 seconds to become interactive after tap.

### Step-by-Step Workflow

1. **Select App Launch** template
2. **Force-quit the app** (for cold launch measurement)
3. **Start recording** -- Instruments will launch the app
4. **Wait for the app to become interactive** (first frame rendered)
5. **Stop recording**

### Reading the Results

The timeline shows two phases:

#### Pre-Main Time
Time before `main()` is called. Includes:
- Dynamic library loading (dyld)
- Objective-C class registration
- `+load` methods
- Static initializers

**Optimization:**
- Reduce number of dynamic frameworks (use static linking)
- Remove `+load` methods (use `+initialize` or lazy initialization)
- Reduce number of Objective-C classes loaded at startup

#### Post-Main Time
Time from `main()` to first frame. Includes:
- `application(_:didFinishLaunchingWithOptions:)` work
- Initial view controller setup
- First SwiftUI body evaluation
- Network calls made during launch

**Optimization:**
- Defer non-critical initialization (analytics, feature flags, crash reporting can wait)
- Use lazy loading for services not needed at launch
- Show skeleton UI while loading data
- Move database migrations to background

### Target Breakdown

```
COLD LAUNCH TARGET: < 2 seconds total

Pre-main:     < 500ms (aim for < 200ms)
  dyld:         < 200ms
  +load:        < 50ms
  static init:  < 50ms

Post-main:    < 1500ms (aim for < 800ms)
  didFinish:    < 300ms
  first screen: < 500ms
  data load:    async (skeleton UI shown)
```

---

## Energy Log

### When to Use
Users report excessive battery drain. App appears in Settings -> Battery as a high consumer.

### Step-by-Step Workflow

1. **Select Energy Log** template
2. **Must use a physical device** (simulator doesn't have power management)
3. **Start recording**
4. **Use the app normally** for 5-10 minutes
5. **Also test background behavior** (lock screen, switch to other apps)
6. **Stop recording**

### Reading the Results

The Energy Log shows energy impact over time, broken down by:

| Component | High Impact Causes |
|-----------|-------------------|
| **CPU** | Continuous computation, tight loops, frequent timer fires |
| **Network** | Frequent polling, large uploads/downloads, connection keepalive |
| **Location** | Continuous GPS, high-accuracy when not needed |
| **Display** | Preventing screen dimming, continuous animations |
| **Bluetooth** | Continuous scanning, frequent connection changes |
| **GPU** | Complex rendering, continuous animation |

### Common Energy Hogs and Fixes

```
PROBLEM: Timer polling every 5 seconds in background
IMPACT: CPU + Network wake every 5 seconds, prevents device sleep
FIX: Use push notifications or BGTaskScheduler for background updates

PROBLEM: Continuous high-accuracy location tracking
IMPACT: GPS radio continuously active
FIX: Use significantLocationChange service, reduce accuracy when high precision not needed

PROBLEM: Animation running while off-screen
IMPACT: GPU + CPU active for invisible animation
FIX: Pause animations in viewDidDisappear/onDisappear, resume in viewDidAppear/onAppear
```

---

## Network

### When to Use
Slow API responses, too many network requests, large payloads.

### Step-by-Step Workflow

1. **Select Network** template
2. **Start recording**
3. **Perform the flow with network activity**
4. **Stop recording**

### Reading the Request Waterfall

The waterfall view shows:
- **Request timeline**: When each request started and finished
- **Sequential vs parallel**: Are requests that could be parallel running sequentially?
- **Payload sizes**: Are responses larger than necessary?
- **Redundant requests**: Is the same endpoint called multiple times?

### Optimization Opportunities

| Pattern | Problem | Solution |
|---------|---------|----------|
| Sequential waterfall | 5 requests in series = 5x latency | Use `async let` or `TaskGroup` for parallel |
| Large payloads | 5MB JSON response | Paginate, use GraphQL, compress |
| Redundant requests | Same endpoint called 3x on screen load | Cache responses, deduplicate in-flight requests |
| Missing caching | Every screen load fetches fresh data | Configure `URLCache`, implement conditional requests |
| No compression | Uncompressed JSON responses | Enable gzip/br with `Accept-Encoding` header |

---

## Quick Reference: Instruments Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| Cmd + R | Start/stop recording |
| Cmd + Shift + R | Stop recording |
| Option + drag | Zoom into time range |
| Cmd + F | Filter/search symbols |
| Cmd + E | Export trace data |
| Space | Pause/resume (live recording) |

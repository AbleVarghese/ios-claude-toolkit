---
name: prometheus-performance
description: "This skill should be used when the user asks to 'profile performance', 'memory leak', 'app launch time', 'CPU optimization', or 'Instruments profiling'. Provides performance profiling, memory analysis, retain cycle detection, and optimization strategies for iOS apps."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
user-invocable: false
---

# Performance Optimization & Profiling

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

Profile app launch time, analyze memory usage, detect retain cycles, optimize CPU and main thread performance, and enforce performance thresholds. Target: sub-second response times, < 3s cold launch, < 100MB memory footprint.

## Performance Thresholds

| Metric | Target | Critical |
|--------|--------|----------|
| Cold Launch | < 3s | > 5s |
| Warm Launch | < 1s | > 2s |
| Screen Load | < 500ms | > 1s |
| API Response | < 200ms | > 500ms |
| Memory Usage | < 100MB | > 200MB |
| Frame Rate | 60 FPS | < 30 FPS |

## Standard Operating Procedures

### SOP-1: Profile App Launch
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
        print("Memory: \(info.resident_size / 1024 / 1024) MB")
    }
}
#endif
```

### SOP-3: Detect Retain Cycles
```swift
// Weak self in closures
class ViewModel: ObservableObject {
    func loadData() {
        service.fetch { [weak self] result in
            guard let self = self else { return }
            self.handleResult(result)
        }
    }
}
```

### SOP-4: Main Thread Checker
```bash
# Enable in scheme
# Edit Scheme -> Run -> Diagnostics -> Main Thread Checker

# Or via environment variable
XCODE_THREAD_SANITIZER_ENABLED=YES
```

## Optimization Strategies

### Image Optimization
```swift
// Use appropriate image scale
Image("icon")
    .resizable()
    .aspectRatio(contentMode: .fit)
    .frame(width: 44, height: 44) // Avoid loading full-res for thumbnails

// Lazy loading for lists
LazyVStack {
    ForEach(items) { item in
        ItemRow(item: item)
    }
}
```

### Network Optimization
```swift
// Batch requests using structured concurrency
async let user = fetchUser()
async let orders = fetchOrders()
async let settings = fetchSettings()

let (u, o, s) = await (user, orders, settings)

// Cache responses
URLCache.shared.memoryCapacity = 50_000_000 // 50 MB
URLCache.shared.diskCapacity = 100_000_000 // 100 MB
```

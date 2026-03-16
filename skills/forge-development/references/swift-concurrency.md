# Swift 6 Strict Concurrency Patterns

## Sendable Conformance Decision Tree

```
Is this type shared across concurrency domains (passed between actors, tasks, or async contexts)?
├── NO → No Sendable needed. Stop.
└── YES →
    Is it a value type (struct/enum)?
    ├── YES →
    │   Are ALL stored properties Sendable?
    │   ├── YES → Swift auto-conforms. Verify with explicit `: Sendable`. Done.
    │   └── NO → Make non-Sendable properties Sendable, or redesign.
    └── NO (class) →
        Is it final with ONLY immutable (let) stored properties, all Sendable?
        ├── YES → Add `: Sendable`. Done.
        └── NO →
            Can you refactor to make it immutable?
            ├── YES → Make all properties `let`, make class `final`, add `: Sendable`.
            └── NO →
                Is it a system type you don't control (os.Logger, NSCache)?
                ├── YES → Use `@unchecked Sendable` with justification comment.
                └── NO →
                    Convert to an `actor` to protect mutable state.
                    Or isolate to @MainActor if it's UI-related.
```

### Quick Reference: What's Automatically Sendable

| Type | Sendable? | Notes |
|------|-----------|-------|
| `Int`, `String`, `Bool`, `UUID`, `Date`, `Decimal` | Yes | Value types with no mutable state |
| `struct` with all Sendable properties | Yes | Auto-conformed by compiler |
| `enum` with all Sendable associated values | Yes | Auto-conformed by compiler |
| `final class` with only `let` Sendable properties | Yes | Must explicitly declare `: Sendable` |
| `class` with `var` properties | No | Use `actor` instead |
| `actor` | Yes | Always Sendable by design |
| `@Sendable` closures | Yes | Cannot capture mutable state |
| `Array<Sendable>`, `Dictionary<Sendable, Sendable>` | Yes | Conditional conformance |

---

## @MainActor Rules

### When to Use @MainActor

- **All ViewModels** — They mutate `@Published` properties that drive UI.
- **Any code that updates UI state** — Setting text, colors, visibility.
- **SwiftUI view body computations** — Already implicitly @MainActor in SwiftUI.
- **Delegate callbacks that update UI** — Ensure they run on main thread.

### When NOT to Use @MainActor

- **Network calls** — These should run on background threads.
- **Data processing** — Heavy computation blocks the UI if on main thread.
- **Repository/Infrastructure layer** — Keep off the main thread.
- **Domain logic** — Business rules don't need the main thread.

### @MainActor Patterns

```swift
// ✅ Entire ViewModel is @MainActor
@MainActor
final class TransferListViewModel: ObservableObject {
    @Published private(set) var transfers: [Transfer] = []
    @Published private(set) var isLoading = false

    private let service: TransferServiceProtocol

    init(service: TransferServiceProtocol) {
        self.service = service
    }

    func loadTransfers() async {
        isLoading = true
        defer { isLoading = false }

        do {
            // The service call runs wherever the service is isolated.
            // The result assignment happens on @MainActor because
            // this method is @MainActor-isolated.
            transfers = try await service.getTransfers()
        } catch {
            // handle error
        }
    }
}
```

```swift
// ✅ When you need a specific method on main actor (rare — prefer class-level)
final class DataProcessor: Sendable {
    func processData(_ raw: Data) async -> ProcessedResult {
        // Heavy processing on background
        let result = // ... process ...
        return result
    }

    @MainActor
    func updateUI(with result: ProcessedResult) {
        // Only this method needs main actor
    }
}
```

---

## Actor Isolation Patterns

### When to Create a Custom Actor

Create a custom actor when you have mutable state that's accessed from multiple concurrency domains but doesn't need the main thread.

```swift
// ✅ Custom actor for in-memory cache
actor TransferCache {
    private var cache: [UUID: Transfer] = [:]

    func get(_ id: UUID) -> Transfer? {
        cache[id]
    }

    func set(_ transfer: Transfer) {
        cache[transfer.id] = transfer
    }

    func invalidate(_ id: UUID) {
        cache.removeValue(forKey: id)
    }

    func clear() {
        cache.removeAll()
    }
}
```

### When to Use @MainActor Instead of Custom Actor

Use `@MainActor` when the mutable state drives UI updates (ViewModels, UI state holders). Use a custom actor when the state is purely back-end (caches, rate limiters, connection pools).

### Actor Reentrancy

Actors suspend at `await` points, and other callers can interleave. This is a common source of bugs.

```swift
actor BankAccount {
    var balance: Decimal = 1000

    // ⚠️ BUG: Actor reentrancy
    func withdraw(_ amount: Decimal) async throws {
        guard balance >= amount else {
            throw AppError.insufficientFunds(available: balance, requested: amount)
        }

        // DANGER: Another caller can withdraw between the guard and this line
        // because `recordTransaction` has an await point.
        try await recordTransaction(amount)

        balance -= amount  // Balance may now be negative!
    }

    // ✅ FIX: Modify state before awaiting
    func withdrawSafe(_ amount: Decimal) async throws {
        guard balance >= amount else {
            throw AppError.insufficientFunds(available: balance, requested: amount)
        }

        // Modify state BEFORE the await point
        balance -= amount

        do {
            try await recordTransaction(amount)
        } catch {
            // Rollback on failure
            balance += amount
            throw error
        }
    }

    private func recordTransaction(_ amount: Decimal) async throws {
        // Network call or database write
    }
}
```

**Rule**: Always modify actor state before `await` points, not after. If the async operation fails, rollback the state change.

---

## Task and TaskGroup Patterns

### Structured Concurrency with TaskGroup

```swift
// ✅ Parallel data loading with TaskGroup
func loadDashboard() async throws -> Dashboard {
    async let profile = userService.getProfile()
    async let transfers = transferService.getRecent(limit: 10)
    async let notifications = notificationService.getUnread()

    // All three requests run in parallel. We await all results.
    return try await Dashboard(
        profile: profile,
        recentTransfers: transfers,
        unreadNotifications: notifications
    )
}
```

### TaskGroup for Dynamic Parallelism

```swift
// ✅ When you don't know the number of parallel tasks at compile time
func validateRecipients(_ ids: [UUID]) async throws -> [RecipientValidation] {
    try await withThrowingTaskGroup(of: RecipientValidation.self) { group in
        for id in ids {
            group.addTask {
                try await self.recipientService.validate(id: id)
            }
        }

        var results: [RecipientValidation] = []
        for try await result in group {
            results.append(result)
        }
        return results
    }
}
```

### Task Cancellation

```swift
// ✅ Check for cancellation in long-running operations
func processLargeDataset(_ items: [Item]) async throws -> [ProcessedItem] {
    var results: [ProcessedItem] = []

    for item in items {
        // Check if the task was cancelled before processing each item
        try Task.checkCancellation()

        let processed = try await process(item)
        results.append(processed)
    }

    return results
}
```

---

## Common Concurrency Bugs

### Bug 1: Data Race — Modifying State from Wrong Isolation

```swift
// ❌ BUG: @Published modified from background
final class BadViewModel: ObservableObject {
    @Published var items: [Item] = []

    func load() {
        Task {
            let result = try await service.getItems()
            items = result  // Data race! Task inherits no actor isolation.
        }
    }
}
```

```swift
// ✅ FIX: @MainActor ensures all mutations are on main thread
@MainActor
final class GoodViewModel: ObservableObject {
    @Published var items: [Item] = []

    func load() async {
        let result = try? await service.getItems()
        items = result ?? []  // Safe — @MainActor isolated
    }
}
```

### Bug 2: Priority Inversion

```swift
// ❌ BUG: Low-priority task blocks high-priority UI update
func fetchData() async {
    // This creates a detached low-priority task
    Task(priority: .background) {
        let data = await heavyComputation()
        // UI update depends on this completing, but it's low priority
        await MainActor.run { self.data = data }
    }
}
```

```swift
// ✅ FIX: Use structured concurrency, let the system manage priority
func fetchData() async {
    let data = await heavyComputation()  // Inherits caller's priority
    self.data = data
}
```

### Bug 3: Capturing Mutable State in @Sendable Closure

```swift
// ❌ BUG: Capturing mutable variable in @Sendable closure
var count = 0
await withTaskGroup(of: Void.self) { group in
    for _ in 0..<100 {
        group.addTask {
            count += 1  // Data race — multiple tasks mutate `count`
        }
    }
}
```

```swift
// ✅ FIX: Use actor or collect results
let results = await withTaskGroup(of: Int.self) { group in
    for _ in 0..<100 {
        group.addTask { 1 }
    }
    var count = 0
    for await value in group {
        count += value  // Sequential — no race
    }
    return count
}
```

---

## @unchecked Sendable Escape Hatches

### When Justified

Only use `@unchecked Sendable` for types that are genuinely thread-safe but lack compiler-verifiable conformance:

```swift
// ✅ os.Logger is thread-safe by Apple's documentation
// but doesn't declare Sendable conformance.
extension Logger: @unchecked Sendable {}

// ✅ NSLock-protected wrapper when actor is too heavy
final class AtomicValue<T>: @unchecked Sendable {
    private var _value: T
    private let lock = NSLock()

    init(_ value: T) { _value = value }

    var value: T {
        lock.lock()
        defer { lock.unlock() }
        return _value
    }

    func update(_ newValue: T) {
        lock.lock()
        defer { lock.unlock() }
        _value = newValue
    }
}
```

### When NOT Justified

```swift
// ❌ NEVER — This class has mutable state without synchronization
final class UserSession: @unchecked Sendable {
    var token: String?        // Mutable, unsynchronized
    var user: User?           // Mutable, unsynchronized
    var isAuthenticated: Bool // Mutable, unsynchronized
}
// FIX: Make this an actor, or use @MainActor if UI-related
```

**Rule of thumb**: If you can't explain why it's thread-safe in one comment line, it probably isn't, and you should use an `actor` instead.

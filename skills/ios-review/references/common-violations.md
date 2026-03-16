# Top 30 iOS Code Quality Violations

Each violation includes: bad example, good example, severity, explanation, and fix.

---

## Architecture Violations

### 1. Domain Imports SwiftUI
**Severity**: BLOCKER

```swift
// ❌ BAD
// File: Domain/Entities/Theme.swift
import SwiftUI

struct Theme {
    let primaryColor: Color
}
```

```swift
// ✅ GOOD
// File: Domain/Entities/Theme.swift
import Foundation

struct Theme: Sendable {
    let primaryColorHex: String
}

// File: Presentation/Extensions/Theme+UI.swift
import SwiftUI
extension Theme {
    var primaryColor: Color { Color(hex: primaryColorHex) }
}
```

**Why**: Domain must be framework-agnostic to enable pure unit testing and portability.

### 2. Singleton Dependency
**Severity**: CRITICAL

```swift
// ❌ BAD
class TransferViewModel: ObservableObject {
    func load() async {
        let transfers = try? await TransferService.shared.getAll()
    }
}
```

```swift
// ✅ GOOD
@MainActor
final class TransferViewModel: ObservableObject {
    private let service: TransferServiceProtocol
    init(service: TransferServiceProtocol) {
        self.service = service
    }
    func load() async {
        let transfers = try? await service.getAll()
    }
}
```

**Why**: Singletons hide dependencies, prevent testing with mocks, and create tight coupling.

### 3. Presentation Imports Infrastructure
**Severity**: CRITICAL

```swift
// ❌ BAD
import SupabaseSDK  // Infrastructure concern in Presentation

@MainActor
final class ProfileViewModel: ObservableObject {
    private let client: SupabaseClient
}
```

```swift
// ✅ GOOD
@MainActor
final class ProfileViewModel: ObservableObject {
    private let userService: UserServiceProtocol  // Domain protocol
}
```

**Why**: Presentation should depend on Domain abstractions, not Infrastructure implementations.

---

## Concurrency Violations

### 4. Missing @MainActor on ViewModel
**Severity**: BLOCKER

```swift
// ❌ BAD
final class UserViewModel: ObservableObject {
    @Published var user: User?

    func load() async {
        user = try? await service.getUser()  // Data race!
    }
}
```

```swift
// ✅ GOOD
@MainActor
final class UserViewModel: ObservableObject {
    @Published private(set) var user: User?

    func load() async {
        user = try? await service.getUser()  // Safe — MainActor isolated
    }
}
```

**Why**: @Published mutations from non-main-thread contexts are undefined behavior under Swift 6.

### 5. @Published Modified From Task Without Isolation
**Severity**: BLOCKER

```swift
// ❌ BAD
@MainActor
final class ListViewModel: ObservableObject {
    @Published var items: [Item] = []

    func refresh() {
        Task.detached {
            let result = try await self.service.getItems()
            self.items = result  // Escapes @MainActor isolation!
        }
    }
}
```

```swift
// ✅ GOOD
@MainActor
final class ListViewModel: ObservableObject {
    @Published private(set) var items: [Item] = []

    func refresh() async {
        items = (try? await service.getItems()) ?? []
    }
}
```

**Why**: `Task.detached` does not inherit actor isolation. The mutation is a data race.

### 6. Missing Sendable on Cross-Actor Type
**Severity**: CRITICAL

```swift
// ❌ BAD
struct TransferResult {
    var id: UUID
    var status: String
    var metadata: [String: Any]  // [String: Any] is not Sendable
}
```

```swift
// ✅ GOOD
struct TransferResult: Sendable {
    let id: UUID
    let status: TransferStatus
    let metadata: [String: String]  // All types are Sendable
}
```

**Why**: Non-Sendable types crossing concurrency boundaries are compile errors in Swift 6.

### 7. Completion Handler Instead of Async/Await
**Severity**: CRITICAL

```swift
// ❌ BAD
func fetchUser(id: String, completion: @escaping (Result<User, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { data, _, error in
        if let error { completion(.failure(error)); return }
        // ...
    }.resume()
}
```

```swift
// ✅ GOOD
func fetchUser(id: String) async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}
```

**Why**: Completion handlers are error-prone (forgetting to call, calling twice), hard to compose, and incompatible with structured concurrency.

### 8. @unchecked Sendable Without Justification
**Severity**: CRITICAL

```swift
// ❌ BAD
final class AppState: @unchecked Sendable {
    var currentUser: User?
    var authToken: String?
}
```

```swift
// ✅ GOOD — Use actor for mutable shared state
actor AppState {
    private(set) var currentUser: User?
    private(set) var authToken: String?

    func setUser(_ user: User) { currentUser = user }
    func setToken(_ token: String) { authToken = token }
}
```

**Why**: @unchecked Sendable disables compiler safety checks. Mutable state without synchronization is a data race.

---

## Security Violations

### 9. Force Unwrap
**Severity**: CRITICAL

```swift
// ❌ BAD
let user = users.first!
let name = json["name"] as! String
```

```swift
// ✅ GOOD
guard let user = users.first else {
    throw AppError.notFound(entity: "User", id: "first")
}
guard let name = json["name"] as? String else {
    throw AppError.invalidInput(field: "name", reason: "Missing or wrong type")
}
```

**Why**: Force unwraps crash the app. Optional handling is free and prevents crashes.

### 10. Sensitive Data in UserDefaults
**Severity**: BLOCKER

```swift
// ❌ BAD
UserDefaults.standard.set(authToken, forKey: "auth_token")
let token = UserDefaults.standard.string(forKey: "auth_token")
```

```swift
// ✅ GOOD
try await keychainService.store(authToken, for: .authToken)
let token = try await keychainService.retrieve(for: .authToken)
```

**Why**: UserDefaults is stored in plaintext plist files. Any process with file access can read them.

### 11. PII in Logs
**Severity**: BLOCKER

```swift
// ❌ BAD
logger.info("User logged in: \(user.email), phone: \(user.phone)")
```

```swift
// ✅ GOOD
logger.info("User logged in", metadata: ["userId": user.id.uuidString])
```

**Why**: Logs are aggregated, stored, and often accessible to operations teams. PII in logs violates privacy regulations.

### 12. Hardcoded API Key
**Severity**: BLOCKER

```swift
// ❌ BAD
let apiKey = "sk_live_abc123def456"
```

```swift
// ✅ GOOD
let apiKey = Config.stripePublishableKey  // Loaded from environment
```

**Why**: Hardcoded secrets in source are in version control permanently. Even after removal, they're in git history.

---

## SwiftUI Violations

### 13. @ObservedObject When @StateObject Needed
**Severity**: CRITICAL

```swift
// ❌ BAD — View creates VM but uses @ObservedObject
struct ProfileView: View {
    @ObservedObject var viewModel = ProfileViewModel()  // Recreated every redraw!

    var body: some View { ... }
}
```

```swift
// ✅ GOOD — View creates VM, uses @StateObject
struct ProfileView: View {
    @StateObject private var viewModel = ProfileViewModel()

    var body: some View { ... }
}
```

**Why**: @ObservedObject doesn't own the object. SwiftUI may recreate it on any view update, losing all state.

### 14. Complex View Body
**Severity**: WARNING

```swift
// ❌ BAD — 50+ lines, deep nesting
var body: some View {
    ScrollView {
        VStack {
            HStack {
                VStack {
                    Text(user.name)
                    Text(user.email)
                    // ... 40 more lines
                }
            }
        }
    }
}
```

```swift
// ✅ GOOD — Extracted subviews
var body: some View {
    ScrollView {
        VStack(spacing: 16) {
            headerSection
            contactSection
            actionButtons
        }
        .padding()
    }
}

@ViewBuilder
private var headerSection: some View { ... }
```

**Why**: Complex view bodies are hard to read, hard to modify, and cause unnecessary recomputation.

### 15. Logic in View Body
**Severity**: WARNING

```swift
// ❌ BAD — Business logic in view
var body: some View {
    let filteredItems = items.filter { $0.isActive && $0.amount > 100 }
    let totalAmount = filteredItems.reduce(0) { $0 + $1.amount }
    // ...
}
```

```swift
// ✅ GOOD — Logic in ViewModel
// ViewModel:
var filteredItems: [Item] { items.filter { $0.isActive && $0.amount > 100 } }
var totalAmount: Decimal { filteredItems.reduce(0) { $0 + $1.amount } }

// View:
var body: some View {
    Text(viewModel.totalAmount, format: .currency(code: "USD"))
}
```

**Why**: Views should only map state to UI. Business logic in views is untestable and violates separation of concerns.

---

## Error Handling Violations

### 16. Empty Catch Block
**Severity**: CRITICAL

```swift
// ❌ BAD
do {
    try await service.processPayment(amount: 100)
} catch {
    // silently ignored
}
```

```swift
// ✅ GOOD
do {
    try await service.processPayment(amount: 100)
} catch {
    logger.error("Payment failed", metadata: ["error": "\(error)"])
    self.error = AppError.from(error)
}
```

**Why**: Silent failures are invisible bugs. The payment failed but the user sees no feedback.

### 17. Bare try? Without Documented Reason
**Severity**: WARNING

```swift
// ❌ BAD
let data = try? await fetchData()
```

```swift
// ✅ GOOD — Either handle the error...
do {
    let data = try await fetchData()
    process(data)
} catch {
    logger.warning("Fetch failed, using cached data", metadata: ["error": "\(error)"])
    let data = cache.get()
    process(data)
}

// ...or document why it's safe to ignore
let data = try? await fetchOptionalEnrichment()  // Non-critical: UI still works without enrichment
```

**Why**: `try?` silently discards the error. If this is intentional, document why. If not, handle it.

### 18. Retain Cycle in Closure
**Severity**: WARNING

```swift
// ❌ BAD
service.onComplete = { result in
    self.processResult(result)  // Strong reference cycle if service holds closure strongly
}
```

```swift
// ✅ GOOD
service.onComplete = { [weak self] result in
    self?.processResult(result)
}
```

**Why**: If `self` holds `service` and `service` holds a closure that captures `self`, neither is deallocated.

---

## Quality Violations

### 19. Hardcoded User-Facing String
**Severity**: WARNING

```swift
// ❌ BAD
Text("Transfer Successful")
```

```swift
// ✅ GOOD
Text(String(localized: "transfer.success.title"))
```

**Why**: Hardcoded strings can't be localized. Even for English-only apps, centralizing strings enables easier updates.

### 20. File Over 300 Lines
**Severity**: WARNING

Split the file by concern. A ViewModel doesn't need to contain its own mock, factory, and extension.

### 21. No Tests for New Logic
**Severity**: CRITICAL

Every new public method needs a test. TDD means the test already exists because you wrote it first.

### 22. Test Without Assertions
**Severity**: CRITICAL

```swift
// ❌ BAD — Test always passes, proves nothing
func test_loadUser() async {
    await viewModel.loadUser(id: UUID())
    // No assertions!
}
```

```swift
// ✅ GOOD
func test_loadUser_whenExists_shouldPopulateUserState() async {
    mockService.getUserResult = UserFactory.createUser(name: "Jane")
    await viewModel.loadUser(id: UUID())
    let user = await viewModel.user
    XCTAssertEqual(user?.name, "Jane")
}
```

### 23-30. Additional Common Violations

| # | Violation | Severity | Quick Fix |
|---|-----------|----------|-----------|
| 23 | Implicitly unwrapped optional (`Type!`) | CRITICAL | Use `Type?` with proper unwrapping |
| 24 | Force cast (`as!`) | CRITICAL | Use `as?` with guard |
| 25 | Missing `private(set)` on @Published | WARNING | Prevent external mutation |
| 26 | Task.detached without justification | WARNING | Use structured `Task {}` |
| 27 | Mutable global/static state | BLOCKER | Use actor or DI |
| 28 | Missing defer for cleanup | WARNING | Add `defer { isLoading = false }` |
| 29 | String concatenation for URLs | WARNING | Use URLComponents |
| 30 | Raw string comparison for status | WARNING | Use typed enum |

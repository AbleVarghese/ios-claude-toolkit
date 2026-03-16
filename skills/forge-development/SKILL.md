---
name: forge-development
description: "Use ANY TIME Swift code is written, modified, or created — even for 'small' changes. TDD-first Swift/SwiftUI development with hexagonal architecture, MVVM, and protocol-based DI. Triggers: implementing features, writing Swift code, creating views or viewmodels, refactoring, fixing bugs, adding a new screen, building a service, wiring dependencies, setting up DI. Covers Swift 6 strict concurrency, Sendable compliance, and error handling. If unsure whether to use this skill, use it — it is the default for all Swift implementation work. For code review without changes, use ios-review instead."
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Edit
user-invocable: false
---

# FORGE — Development & Code Engineering

Swift/SwiftUI development skill for iOS apps. Enforces hexagonal architecture + MVVM + protocol-based DI with TDD-first methodology.

> **Iron Law**: "No implementation without a failing test first. No exception."

> **Project Discovery**: Before executing, determine project-specific values (project name, scheme, bundle ID, target, architecture layers) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

---

## Phased TDD Workflow: RED → GREEN → REFACTOR

TDD is not optional. Every implementation begins with a failing test. This section shows the complete cycle with real Swift/XCTest examples.

### Phase 1: RED — Write the Failing Test

Write the test first. It must fail. If it passes before you write implementation code, the test is wrong — it's not testing anything new.

```swift
// File: Tests/ServicesTests/TransferServiceTests.swift

final class TransferServiceTests: XCTestCase {

    // MARK: - test_methodName_whenCondition_shouldExpectedBehavior

    func test_executeTransfer_whenSufficientBalance_shouldDebitSenderAndCreditRecipient() async throws {
        // Arrange
        let mockRepo = MockTransferRepository()
        let mockBalanceService = MockBalanceService(balance: Decimal(1000))
        let sut = TransferService(
            repository: mockRepo,
            balanceService: mockBalanceService
        )
        let request = TransferRequest(
            senderId: UUID(),
            recipientId: UUID(),
            amount: Decimal(250),
            currency: .usd
        )

        // Act
        let result = try await sut.execute(request)

        // Assert
        XCTAssertEqual(result.status, .completed)
        XCTAssertEqual(mockRepo.savedTransfer?.amount, Decimal(250))
        XCTAssertEqual(mockBalanceService.debitedAmount, Decimal(250))
        XCTAssertEqual(mockBalanceService.creditedAmount, Decimal(250))
    }

    func test_executeTransfer_whenInsufficientBalance_shouldThrowInsufficientFunds() async {
        // Arrange
        let mockBalanceService = MockBalanceService(balance: Decimal(100))
        let sut = TransferService(
            repository: MockTransferRepository(),
            balanceService: mockBalanceService
        )
        let request = TransferRequest(
            senderId: UUID(),
            recipientId: UUID(),
            amount: Decimal(500),
            currency: .usd
        )

        // Act & Assert
        do {
            _ = try await sut.execute(request)
            XCTFail("Expected TransferError.insufficientFunds to be thrown")
        } catch let error as TransferError {
            XCTAssertEqual(error, .insufficientFunds(available: Decimal(100), requested: Decimal(500)))
        } catch {
            XCTFail("Unexpected error type: \(error)")
        }
    }
}
```

Run the test. It must fail:
```
❌ test_executeTransfer_whenSufficientBalance_shouldDebitSenderAndCreditRecipient — Error: Cannot find 'TransferService' in scope
```

This failure confirms the test is real. Now proceed to GREEN.

### Phase Gate: RED
**Cannot proceed to GREEN until ALL conditions are met:**
- [ ] Test file exists and compiles
- [ ] Test has been RUN (not just written)
- [ ] Output shows FAILURE (not compilation error — a genuine test assertion failure)
- [ ] You can explain in one sentence WHY it fails

**Proof required**: Show the test run command and its failure output.

**If test passes immediately**: The test is wrong — it's testing existing behavior, not new behavior. Delete the test. Write one that actually fails. This is the most common TDD mistake.

**If you wrote implementation code before reaching this gate**: STOP. Delete the implementation. You violated the Iron Law. Return to RED.

### Phase 2: GREEN — Minimum Implementation to Pass

Write only enough code to make the test pass. Resist the urge to over-engineer.

```swift
// File: Sources/Services/TransferService.swift

final class TransferService: TransferServiceProtocol {
    private let repository: TransferRepositoryProtocol
    private let balanceService: BalanceServiceProtocol

    init(repository: TransferRepositoryProtocol, balanceService: BalanceServiceProtocol) {
        self.repository = repository
        self.balanceService = balanceService
    }

    func execute(_ request: TransferRequest) async throws -> Transfer {
        let balance = try await balanceService.getBalance(for: request.senderId)

        guard balance >= request.amount else {
            throw TransferError.insufficientFunds(
                available: balance,
                requested: request.amount
            )
        }

        try await balanceService.debit(userId: request.senderId, amount: request.amount)
        try await balanceService.credit(userId: request.recipientId, amount: request.amount)

        let transfer = Transfer(
            id: UUID(),
            senderId: request.senderId,
            recipientId: request.recipientId,
            amount: request.amount,
            currency: request.currency,
            status: .completed,
            createdAt: Date()
        )

        try await repository.save(transfer)
        return transfer
    }
}
```

Run the test. It must pass:
```
✅ test_executeTransfer_whenSufficientBalance_shouldDebitSenderAndCreditRecipient — Passed
✅ test_executeTransfer_whenInsufficientBalance_shouldThrowInsufficientFunds — Passed
```

### Phase Gate: GREEN
**Cannot proceed to REFACTOR until ALL conditions are met:**
- [ ] The previously-failing test now PASSES
- [ ] You wrote the MINIMUM code to pass (not the "right" code — the minimum)
- [ ] All OTHER existing tests still pass (no regressions)

**Proof required**: Show test suite output. Previously-red test is green. No other tests broken.

**Counter**: If you wrote more than ~20 lines to pass one test, you're over-engineering the GREEN phase. The point is minimum viable — elegance comes in REFACTOR.

### Phase 3: REFACTOR — Improve Without Changing Behavior

Now clean up. Extract, rename, simplify — but the tests must stay green after every change.

Common refactoring targets:
- Extract validation into a private method
- Add `Sendable` conformance if the type crosses actor boundaries
- Ensure `@MainActor` is applied if needed
- Improve naming for clarity
- Remove duplication

### Phase Gate: REFACTOR
**Cannot commit until ALL conditions are met:**
- [ ] All tests pass (zero failures)
- [ ] No new compiler warnings introduced
- [ ] Import boundaries verified (Domain imports only Foundation)
- [ ] No force unwraps added
- [ ] No `.shared` singletons introduced

**Proof required**: Full test suite output — zero failures, zero warnings.

---

## Architecture Gate (Before Writing ANY File)

Before creating or modifying any Swift file, verify:
1. Which layer does this file belong to? (Presentation / Domain / Infrastructure)
2. What does it import? Check: Does any import violate the dependency rule?
   - Presentation: may import Domain + SwiftUI/UIKit
   - Domain: may import ONLY Foundation (no frameworks, no infrastructure)
   - Infrastructure: may import Domain + external frameworks (Supabase, Stripe, etc.)
3. If an import violates the rule, extract a protocol in the Domain layer first.

**Cannot write the file until layer and imports are verified.** Writing code in the wrong layer cascades architectural violations that are expensive to fix later.

---

## Architecture: Hexagonal (Ports & Adapters)

### Layer Diagram

```
┌─────────────────────────────────────────┐
│          Presentation Layer             │
│   Views, ViewModels, UI Components      │
│   Imports: SwiftUI, Domain              │
│   NEVER imports: Infrastructure         │
├─────────────────────────────────────────┤
│          Domain Layer                   │
│   Entities, Ports (Protocols),          │
│   Use Cases, Domain Errors              │
│   Imports: Foundation ONLY              │
│   NEVER imports: SwiftUI, UIKit,        │
│   third-party frameworks                │
├─────────────────────────────────────────┤
│        Infrastructure Layer             │
│   Adapters, Repositories,              │
│   API Clients, Database Access          │
│   Imports: Domain, third-party SDKs     │
│   NEVER imports: Presentation           │
└─────────────────────────────────────────┘

Dependency Direction: INWARD ONLY
  Presentation → Domain ← Infrastructure
```

### Architecture Violation Detection Decision Tree

```
START: You're about to add an import statement.

Q1: Does this import cross a layer boundary?
├── NO → Proceed. Same-layer imports are fine.
└── YES →
    Q2: Does the dependency flow inward (toward Domain)?
    ├── YES → Proceed. This is correct.
    └── NO → VIOLATION DETECTED.
        Q3: What kind of violation?
        ├── Domain imports a framework (SwiftUI, UIKit, SDK)
        │   → Extract a protocol (port) in Domain.
        │     Implement it as an adapter in Infrastructure.
        ├── Presentation imports Infrastructure directly
        │   → Inject the dependency via protocol from Domain.
        │     The DI container wires the concrete type.
        ├── Infrastructure imports Presentation
        │   → This is always wrong. Reverse the dependency.
        │     Use a delegate protocol or closure callback.
        └── Any layer imports a concrete type from another layer
            → Replace with protocol import.
              Register concrete type in DI container.
```

### Dependency Injection Pattern

```swift
// 1. Define port (protocol) in Domain layer
protocol UserRepositoryProtocol: Sendable {
    func findById(_ id: UUID) async throws -> User?
    func save(_ user: User) async throws
}

// 2. Implement adapter in Infrastructure layer
final class SupabaseUserRepository: UserRepositoryProtocol {
    private let client: SupabaseClient

    init(client: SupabaseClient) {
        self.client = client
    }

    func findById(_ id: UUID) async throws -> User? {
        // Supabase-specific implementation
    }

    func save(_ user: User) async throws {
        // Supabase-specific implementation
    }
}

// 3. Register in DI container (App layer)
container.register(UserRepositoryProtocol.self) { resolver in
    SupabaseUserRepository(client: resolver.resolve(SupabaseClient.self))
}

// 4. Consume via protocol (Presentation or Use Case layer)
@MainActor
final class UserProfileViewModel: ObservableObject {
    private let repository: UserRepositoryProtocol

    init(repository: UserRepositoryProtocol) {
        self.repository = repository
    }
}
```

---

## SwiftUI View Composition

### When to Extract Subviews

Extract a subview when:
- The `body` property exceeds ~30 lines
- A visual section is reused in multiple views
- A section has its own state or logic
- The view has more than 3 levels of nesting

### @ViewBuilder Usage

Use `@ViewBuilder` for computed properties and helper methods that return views:

```swift
struct TransferDetailView: View {
    let transfer: Transfer

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                headerSection
                amountSection
                statusSection
            }
            .padding()
        }
    }

    @ViewBuilder
    private var headerSection: some View {
        HStack {
            AvatarView(userId: transfer.senderId)
            Image(systemName: "arrow.right")
            AvatarView(userId: transfer.recipientId)
        }
    }

    @ViewBuilder
    private var amountSection: some View {
        VStack {
            Text(transfer.amount, format: .currency(code: transfer.currency.rawValue))
                .font(.largeTitle.bold())
            Text(transfer.createdAt, style: .date)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
    }

    @ViewBuilder
    private var statusSection: some View {
        Label(transfer.status.displayName, systemImage: transfer.status.iconName)
            .foregroundStyle(transfer.status.color)
    }
}
```

### View Body Complexity Limits

- Maximum 30 lines in `body`
- Maximum 3 levels of nesting before extracting
- No logic in `body` — move to ViewModel or computed property
- No inline closures longer than 5 lines — extract to a method

### @StateObject vs @ObservedObject

```swift
// @StateObject: View OWNS the ViewModel (creates it)
// Use when this view is the source of truth for the ViewModel.
struct ProfileView: View {
    @StateObject private var viewModel: ProfileViewModel

    init(userId: UUID) {
        _viewModel = StateObject(wrappedValue: ProfileViewModel(userId: userId))
    }
}

// @ObservedObject: View BORROWS the ViewModel (receives it)
// Use when a parent view passes the ViewModel down.
struct ProfileHeaderView: View {
    @ObservedObject var viewModel: ProfileViewModel
}
```

**Critical rule**: If a view creates the ViewModel, use `@StateObject`. If a parent passes it, use `@ObservedObject`. Getting this wrong causes the ViewModel to be recreated on every view redraw, losing state.

---

## ViewModel Pattern

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    // MARK: - Published State
    @Published private(set) var user: User?
    @Published private(set) var isLoading = false
    @Published private(set) var error: AppError?

    // MARK: - Dependencies (injected via protocol)
    private let userService: UserServiceProtocol
    private let logger: LoggerProtocol

    // MARK: - Init
    init(userService: UserServiceProtocol, logger: LoggerProtocol) {
        self.userService = userService
        self.logger = logger
    }

    // MARK: - Actions
    func loadUser(id: UUID) async {
        isLoading = true
        error = nil
        defer { isLoading = false }

        do {
            user = try await userService.getUser(id: id)
        } catch let appError as AppError {
            error = appError
            logger.error("Failed to load user", metadata: ["userId": id.uuidString, "error": appError.code])
        } catch {
            self.error = .unexpected(underlying: error)
            logger.error("Unexpected error loading user", metadata: ["userId": id.uuidString])
        }
    }
}
```

**Why `@MainActor` on every ViewModel**: ViewModels mutate `@Published` properties that drive UI updates. UI updates must happen on the main thread. Without `@MainActor`, setting `@Published` properties from an async context is a data race under Swift 6 strict concurrency.

---

## Swift 6 Strict Concurrency Decision Tree

```
START: You have a type that's used in your codebase.

Q1: Is this type shared across actors or concurrency domains?
├── NO → No Sendable conformance needed. Stop.
└── YES →
    Q2: Is this type a value type (struct/enum) with all Sendable stored properties?
    ├── YES → Swift auto-conforms it to Sendable. Verify, then stop.
    └── NO →
        Q3: Is this a class?
        ├── YES →
        │   Q4: Can you make it final with only let properties (all Sendable)?
        │   ├── YES → Make it final, add `: Sendable`. Done.
        │   └── NO →
        │       Q5: Can you isolate it to an actor?
        │       ├── YES → Use @MainActor or a custom actor.
        │       └── NO →
        │           Q6: Does it wrap a system type you don't control (os.Logger, etc.)?
        │           ├── YES → Use @unchecked Sendable with documented justification.
        │           └── NO → Refactor to make it Sendable. Extract mutable state into an actor.
        └── NO (protocol) →
            Q7: Should all conforming types be Sendable?
            ├── YES → Add Sendable constraint: protocol Foo: Sendable {}
            │         WARNING: This cascades to ALL implementors.
            └── NO → Don't constrain the protocol. Constrain at the usage site.
```

---

## Error Handling Taxonomy

Errors are not all equal. Classify them to determine the correct response.

```swift
// Domain-level error classification
enum AppError: LocalizedError, Equatable {
    // RETRIABLE — Temporary failures, retry with backoff
    case networkTimeout
    case rateLimited(retryAfter: TimeInterval)
    case serviceUnavailable

    // PERMANENT — Bad input or state, no point retrying
    case notFound(entity: String, id: String)
    case invalidInput(field: String, reason: String)
    case unauthorized
    case forbidden
    case insufficientFunds(available: Decimal, requested: Decimal)

    // FATAL — Corrupt state, crash loudly
    case corruptData(description: String)
    case missingConfiguration(key: String)
    case unexpected(underlying: Error)

    var isRetriable: Bool {
        switch self {
        case .networkTimeout, .rateLimited, .serviceUnavailable:
            return true
        default:
            return false
        }
    }

    var errorDescription: String? {
        switch self {
        case .networkTimeout:
            return "Request timed out. Please try again."
        case .rateLimited(let retryAfter):
            return "Too many requests. Please wait \(Int(retryAfter)) seconds."
        case .serviceUnavailable:
            return "Service temporarily unavailable."
        case .notFound(let entity, _):
            return "\(entity) not found."
        case .invalidInput(let field, let reason):
            return "Invalid \(field): \(reason)"
        case .unauthorized:
            return "Please sign in to continue."
        case .forbidden:
            return "You don't have permission for this action."
        case .insufficientFunds(let available, let requested):
            return "Insufficient balance. Available: \(available), requested: \(requested)."
        case .corruptData(let description):
            return "Data integrity error: \(description)"
        case .missingConfiguration(let key):
            return "Missing configuration: \(key)"
        case .unexpected:
            return "An unexpected error occurred."
        }
    }
}
```

### Error Handling Strategy by Classification

| Classification | Action | Retry? | User Feedback | Log Level |
|---------------|--------|--------|---------------|-----------|
| Retriable | Retry up to 3x with exponential backoff | Yes | Show loading/retry button | `.warning` |
| Permanent | Return error immediately | No | Show specific error message | `.info` |
| Fatal | Crash or escalate | No | Show generic error + support link | `.critical` |

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|---------------|-----------------|
| "It's faster to skip TDD" | You write the test anyway. Skipping TDD means debugging without a safety net, which takes longer. | Write the failing test first. Always. |
| "This is just a small change" | Small changes cause the most insidious bugs because they skip review rigor. | Same process regardless of size. RED → GREEN → REFACTOR. |
| "The architecture is overkill for this" | Consistency is the architecture's value. One shortcut creates precedent for ten more. | Follow the layer rules. Extract protocol if crossing boundary. |
| "I'll add tests later" | "Later" never comes. Untested code is legacy code from the moment it's written. | Write the test now. It takes 5 minutes. |
| "This view is too simple to test" | Simple views grow. The test ensures the ViewModel binding works correctly. | Test the ViewModel. Test the view's binding to it. |
| "Force unwrap is safe here — it's never nil" | You don't know the future. Optional chaining costs nothing; crashes cost everything. | Use `guard let` or optional chaining. Zero force unwraps. |
| "Completion handlers are fine for this callback" | Mixing async/await and callbacks creates concurrency bugs that are hard to trace. | Wrap in `withCheckedThrowingContinuation` or rewrite as async. |
| "@unchecked Sendable is easier" | It silences the compiler but doesn't fix the data race. It moves the crash from compile-time to runtime. | Use proper Sendable conformance. Reserve @unchecked for system types only. |

---

## Red Flags — HARD STOP

These are BLOCKING conditions, not suggestions. If ANY red flag is triggered, you CANNOT continue the current task until the violation is remediated. Proceeding past a red flag invalidates all subsequent work.

- **Writing implementation before test**: You have violated the Iron Law. Delete the implementation, write the test, watch it fail, then reimplement.
- **Importing UIKit/SwiftUI in the domain layer**: Architecture violation. The domain layer must have zero framework imports beyond Foundation. Extract a protocol.
- **Using `.shared` singletons for dependencies**: This makes testing impossible and hides coupling. Inject via initializer through a protocol.
- **Force unwraps (`!`) anywhere**: Replace with `guard let`, `if let`, or optional chaining. The only exception is `@IBOutlet` (which you shouldn't be using in SwiftUI).
- **Completion handlers instead of async/await**: All new code must use structured concurrency. Wrap legacy APIs with continuations.
- **`@Published` properties modified from a non-`@MainActor` context**: This is a data race. The ViewModel must be `@MainActor`.

---

## When NOT to Use This Skill

- **Reviewing existing code without implementing changes** → Use `ios-review`
- **Performance profiling or optimization** → Use `prometheus-performance`
- **Accessibility audit or VoiceOver compliance** → Use `athena-accessibility`
- **Git operations, commits, or branch management** → Use `heimdall-git`
- **Coordinating multi-step workflows across agents** → Use `maestro-orchestrator`

---

## Decision Framework: What to Do Next

```
START: You have a task to implement.

Q1: Does a failing test exist for this behavior?
├── NO → Write the failing test (RED). Go to Q1 again.
└── YES →
    Q2: Does the test pass?
    ├── NO → Write minimum implementation (GREEN). Go to Q2.
    └── YES →
        Q3: Is the code clean? (no duplication, clear naming, proper structure)
        ├── NO → Refactor while keeping tests green. Go to Q3.
        └── YES →
            Q4: Does this code cross a layer boundary?
            ├── YES → Verify dependency direction is inward.
            │         If not, extract protocol and fix.
            └── NO → Done. Move to next task.
```

---

## Quality Gates (Before Marking Complete)

- [ ] All tests pass (including the new ones you wrote)
- [ ] No force unwraps (`!`) in production code
- [ ] All ViewModels annotated with `@MainActor`
- [ ] All dependencies injected via protocol — no `.shared` singletons
- [ ] async/await used exclusively — no completion handlers in new code
- [ ] Domain layer has zero SwiftUI/UIKit imports
- [ ] Dependency direction flows inward only (Presentation → Domain ← Infrastructure)
- [ ] `Sendable` conformance applied where types cross concurrency domains
- [ ] Error handling uses the taxonomy (retriable/permanent/fatal) — no bare `try?`
- [ ] View body under 30 lines, max 3 nesting levels

---

## Self-Audit (Before Declaring Implementation Complete)

Answer each question. If ANY answer is "no" or "unsure" — go back and fix it.

1. Did I write a failing test BEFORE every piece of implementation code? (Cite the test name and what it tested)
2. Did I run the test suite after each RED-GREEN-REFACTOR cycle? (How many cycles did I complete?)
3. Does every new type/protocol live in the correct architecture layer? (List each new file and its layer)
4. Are all dependencies injected via protocols? (No `.shared`, no concrete types in initializers)
5. Would the test suite catch a regression if someone changed the implementation? (Are tests testing behavior, not implementation details?)

---

## Required Output Artifact

Every implementation task using this skill must produce:
1. **Test file(s)** — written FIRST, showing failing assertions
2. **Implementation file(s)** — minimum code to pass tests, properly layered
3. **Protocol(s)** — for any new dependencies (ports in hexagonal architecture)
4. **Test run evidence** — showing RED to GREEN progression

An implementation without test files is incomplete. An implementation where tests were written AFTER the code is a violation — the tests might be testing the implementation rather than the behavior.

---

## Cross-Skill References

| Skill | When to Use |
|-------|-------------|
| `ios-review` | After implementing, review code quality and catch violations you may have missed |
| `prometheus-performance` | When you suspect performance issues — view redraws, memory, network efficiency |
| `athena-accessibility` | When building UI — ensure VoiceOver labels, Dynamic Type, and contrast compliance |
| `heimdall-git` | When committing implementation work — conventional commits, branch management |
| `documentation` | After completing a feature — update STATUS.md, BACKLOG.md, and changelog |
| `maestro-orchestrator` | When the implementation requires coordinating multiple parallel tasks |

---

## References

- **Architecture Patterns**: `references/architecture-patterns.md` — hexagonal layer rules, DI container patterns, dependency direction violations
- **Swift Concurrency**: `references/swift-concurrency.md` — Sendable, @MainActor, actors, TaskGroup, concurrency bugs
- **TDD Patterns**: `references/tdd-patterns.md` — test factories, mock patterns, async testing, naming conventions

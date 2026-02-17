---
name: forge-development
description: "This skill should be used when the user asks to 'implement a feature', 'refactor code', 'add a view or viewmodel', 'write Swift code', or 'fix architecture violation'. Provides TDD-first Swift/SwiftUI development following hexagonal architecture, MVVM, and protocol-based dependency injection."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Edit
user-invocable: false
---

# FORGE - Development & Code Engineering

Swift/SwiftUI development skill for iOS apps. Enforces hexagonal architecture + MVVM + protocol-based DI with TDD-first methodology.

> **Project Discovery**: Before executing, determine project-specific values (project name, scheme, bundle ID, target, architecture layers) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

## Standard Operating Procedures

### SOP-1: TDD Feature Development
```swift
// 1. RED - Write failing test
func test_createItem_whenInputValid_shouldSucceed() async throws {
    // Given
    let sut = ItemService(repository: mockRepo)

    // When
    let result = try await sut.createItem(name: "Test")

    // Then
    XCTAssertEqual(result.status, .active)
}

// 2. GREEN - Implement minimum code to pass
// 3. REFACTOR - Improve code while keeping tests green
```

### SOP-2: Protocol-Based DI
```swift
// Define port (interface)
protocol UserServiceProtocol {
    func getUser(id: String) async throws -> User
}

// Implement adapter
final class UserService: UserServiceProtocol {
    private let repository: UserRepositoryProtocol

    init(repository: UserRepositoryProtocol) {
        self.repository = repository
    }

    func getUser(id: String) async throws -> User {
        // Implementation
    }
}

// Register in DI container
container.register(UserServiceProtocol.self) { resolver in
    UserService(repository: resolver.resolve(UserRepositoryProtocol.self))
}
```

### SOP-3: SwiftUI View Pattern
```swift
struct ItemDetailView: View {
    @StateObject private var viewModel: ItemDetailViewModel

    init(viewModel: ItemDetailViewModel) {
        _viewModel = StateObject(wrappedValue: viewModel)
    }

    var body: some View {
        // View implementation
    }
}
```

### SOP-4: ViewModel Pattern
```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published var isLoading = false
    @Published var errorMessage: String?

    private let userService: UserServiceProtocol

    init(userService: UserServiceProtocol) {
        self.userService = userService
    }

    func loadData() async {
        isLoading = true
        defer { isLoading = false }
        do {
            // Load data
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

### SOP-5: Error Handling
```swift
// Domain errors
enum ItemError: LocalizedError {
    case notFound
    case invalidInput(String)
    case networkFailure(underlying: Error)

    var errorDescription: String? {
        switch self {
        case .notFound: return "Item not found"
        case .invalidInput(let detail): return "Invalid input: \(detail)"
        case .networkFailure(let error): return "Network error: \(error.localizedDescription)"
        }
    }
}
```

## Code Quality Checklist

- [ ] No force unwraps (`!`)
- [ ] async/await only (no callbacks)
- [ ] Protocol-based DI (inject via initializers, not `.shared` singletons)
- [ ] Proper error handling with `LocalizedError` conformance
- [ ] `@Sendable` compliance for closures crossing actor boundaries
- [ ] `Sendable` conformance for types shared across concurrency domains
- [ ] DocC documentation on all public APIs
- [ ] Unit tests written first (RED -> GREEN -> REFACTOR)
- [ ] `@MainActor` on all ViewModels

## Swift 6 Strict Concurrency Notes

- Enable `SWIFT_STRICT_CONCURRENCY: complete` for full enforcement
- All closures crossing actor boundaries must be `@Sendable`
- Types shared across concurrency domains must conform to `Sendable`
- Use `@unchecked Sendable` sparingly (e.g., types wrapping `os.Logger` or other non-Sendable system types)
- Adding `Sendable` to port protocols cascades to ALL implementors — plan for this

## Architecture Layers

```
Presentation (Views/ViewModels)
        |
        v
    Use Cases
        |
        v
    Domain (Entities/Ports)
        |
        v
Infrastructure (Adapters/Repositories)
```

**Dependency rule**: Dependencies flow inward only. Domain layer must have zero external framework imports. Infrastructure adapters implement domain ports.

**Verify**: Check the project's CLAUDE.md or ARCHITECTURE.md for the specific layer structure and naming conventions used.

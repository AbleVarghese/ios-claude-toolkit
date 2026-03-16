# Architecture Patterns — Hexagonal Architecture for iOS

## Layer Rules & Dependency Direction

```
┌──────────────────────────────────────────────────────────────┐
│                    APP / COMPOSITION ROOT                     │
│  - DI container setup                                        │
│  - Wires concrete types to protocols                         │
│  - @main entry point                                         │
│  Imports: Everything (this is the only layer that can)        │
├──────────────────────────────────────────────────────────────┤
│                    PRESENTATION LAYER                         │
│  - Views (SwiftUI)                                           │
│  - ViewModels (@MainActor, ObservableObject)                 │
│  - View-specific formatters and helpers                      │
│  Imports: SwiftUI, Domain protocols                          │
│  NEVER imports: Infrastructure, third-party SDKs             │
├──────────────────────────────────────────────────────────────┤
│                    DOMAIN LAYER (innermost)                   │
│  - Entities (value types, business objects)                  │
│  - Ports (protocols for services, repositories)              │
│  - Use Cases (orchestrate business logic)                    │
│  - Domain Errors                                             │
│  Imports: Foundation ONLY                                    │
│  NEVER imports: SwiftUI, UIKit, Combine, third-party         │
├──────────────────────────────────────────────────────────────┤
│                    INFRASTRUCTURE LAYER                       │
│  - Adapters (implement Domain ports)                         │
│  - API Clients (network layer)                               │
│  - Database Access (CoreData, Supabase, etc.)                │
│  - External SDK wrappers                                     │
│  Imports: Domain protocols, third-party SDKs                 │
│  NEVER imports: Presentation (Views, ViewModels)             │
└──────────────────────────────────────────────────────────────┘

DEPENDENCY DIRECTION: Always inward.
  Presentation → Domain ← Infrastructure
  Domain depends on NOTHING outside itself.
```

## What Each Layer Can and Cannot Import

| Layer | Can Import | Cannot Import |
|-------|-----------|---------------|
| App/Composition Root | Everything | — |
| Presentation | SwiftUI, Domain | Infrastructure, SDKs |
| Domain | Foundation | SwiftUI, UIKit, Infrastructure, SDKs |
| Infrastructure | Domain, Foundation, SDKs | Presentation |

**Why**: Domain isolation means you can swap Infrastructure without touching business logic. You can test Domain without a database, network, or UI. This is the entire point of hexagonal architecture.

---

## DI Container Setup Patterns

### Protocol Registration

```swift
// File: App/DependencyContainer.swift

final class DependencyContainer {
    static let shared = DependencyContainer()

    // MARK: - Registrations

    lazy var userRepository: UserRepositoryProtocol = {
        SupabaseUserRepository(client: supabaseClient)
    }()

    lazy var transferRepository: TransferRepositoryProtocol = {
        SupabaseTransferRepository(client: supabaseClient)
    }()

    lazy var userService: UserServiceProtocol = {
        UserService(repository: userRepository)
    }()

    lazy var transferService: TransferServiceProtocol = {
        TransferService(
            repository: transferRepository,
            balanceService: balanceService
        )
    }()

    // MARK: - Infrastructure

    private lazy var supabaseClient: SupabaseClient = {
        SupabaseClient(
            supabaseURL: Config.supabaseURL,
            supabaseKey: Config.supabaseAnonKey
        )
    }()
}
```

### Resolution in Views

```swift
// The DI container is accessed ONLY at the composition root (App entry point).
// Views receive their ViewModels already wired with dependencies.

@main
struct MyApp: App {
    private let container = DependencyContainer.shared

    var body: some Scene {
        WindowGroup {
            NavigationStack {
                ProfileView(
                    viewModel: ProfileViewModel(
                        userService: container.userService
                    )
                )
            }
        }
    }
}
```

### Factory Methods for Complex Construction

When a ViewModel needs runtime parameters (like a user ID) plus injected services:

```swift
// File: App/ViewModelFactory.swift

@MainActor
final class ViewModelFactory {
    private let container: DependencyContainer

    init(container: DependencyContainer) {
        self.container = container
    }

    func makeTransferDetailViewModel(transferId: UUID) -> TransferDetailViewModel {
        TransferDetailViewModel(
            transferId: transferId,
            transferService: container.transferService,
            userService: container.userService
        )
    }
}
```

---

## Common Dependency Direction Violations

### Violation 1: Domain imports a framework

```swift
// ❌ WRONG — Domain layer importing SwiftUI
// File: Domain/Entities/User.swift
import SwiftUI  // VIOLATION

struct User {
    let id: UUID
    let name: String
    var avatarColor: Color  // SwiftUI type in Domain
}
```

```swift
// ✅ CORRECT — Domain uses Foundation types, Presentation maps to SwiftUI
// File: Domain/Entities/User.swift
import Foundation

struct User: Sendable {
    let id: UUID
    let name: String
    let avatarColorHex: String  // Platform-agnostic
}

// File: Presentation/Extensions/User+UI.swift
import SwiftUI

extension User {
    var avatarColor: Color {
        Color(hex: avatarColorHex)
    }
}
```

**Why**: If Domain depends on SwiftUI, you can't test Domain logic in a pure unit test target without importing the entire UI framework.

### Violation 2: Presentation imports Infrastructure directly

```swift
// ❌ WRONG — ViewModel directly depends on concrete repository
// File: Presentation/ViewModels/ProfileViewModel.swift
import Foundation

@MainActor
final class ProfileViewModel: ObservableObject {
    private let repo = SupabaseUserRepository()  // VIOLATION: concrete Infrastructure type
}
```

```swift
// ✅ CORRECT — ViewModel depends on protocol, injected via init
// File: Presentation/ViewModels/ProfileViewModel.swift
import Foundation

@MainActor
final class ProfileViewModel: ObservableObject {
    private let userService: UserServiceProtocol

    init(userService: UserServiceProtocol) {
        self.userService = userService
    }
}
```

**Why**: Direct Infrastructure import means you can't test the ViewModel without a real database connection.

### Violation 3: Infrastructure imports Presentation

```swift
// ❌ WRONG — Repository trying to update UI directly
// File: Infrastructure/Repositories/SyncRepository.swift
import SwiftUI  // VIOLATION

final class SyncRepository {
    func sync() async {
        // ... sync logic ...
        await MainActor.run {
            NotificationCenter.default.post(name: .syncComplete, object: nil)
        }
    }
}
```

```swift
// ✅ CORRECT — Repository returns result, caller handles UI update
// File: Infrastructure/Repositories/SyncRepository.swift
import Foundation

final class SyncRepository: SyncRepositoryProtocol {
    func sync() async throws -> SyncResult {
        // ... sync logic ...
        return SyncResult(updatedCount: count, timestamp: Date())
    }
}
```

**Why**: Infrastructure should never know about the UI. It returns data; the Presentation layer decides how to display it.

---

## When to Use @unchecked Sendable

`@unchecked Sendable` tells the compiler "trust me, this is thread-safe" without verification. This is dangerous because it moves data race detection from compile-time to runtime.

**Justified uses** (document with a comment explaining why):

```swift
// os.Logger is thread-safe but doesn't conform to Sendable.
// Apple hasn't added the conformance yet.
// Safe because os.Logger is internally synchronized.
extension Logger: @unchecked Sendable {}
```

```swift
// NSCache is thread-safe (Apple docs confirm this).
// Wrapping for Sendable conformance.
final class SendableCache<Key: Hashable, Value>: @unchecked Sendable {
    private let cache = NSCache<WrappedKey<Key>, WrappedValue<Value>>()
    // ...
}
```

**Never justified**:

```swift
// ❌ NEVER — This hides a real data race
final class UserManager: @unchecked Sendable {
    var currentUser: User?  // Mutable state without synchronization!
}
```

**Rule**: If the type has mutable state, use an `actor` instead of `@unchecked Sendable`.

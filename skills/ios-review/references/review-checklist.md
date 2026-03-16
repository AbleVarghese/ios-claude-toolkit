# iOS Code Review Checklist

Organized by category. Every item has a severity level and what to look for.

---

## Architecture (Layer Compliance)

| # | Check | Severity | What to Look For |
|---|-------|----------|-----------------|
| A1 | Domain layer has zero framework imports | BLOCKER | `import SwiftUI`, `import UIKit`, `import Combine` in Domain/ |
| A2 | Presentation never imports Infrastructure | CRITICAL | `import InfrastructureModule` in Views/ or ViewModels/ |
| A3 | Infrastructure never imports Presentation | BLOCKER | `import PresentationModule` in Adapters/ or Repositories/ |
| A4 | Dependencies flow inward only | BLOCKER | Outward arrows in dependency graph |
| A5 | All dependencies injected via protocol | CRITICAL | `.shared`, `static let`, concrete types in init |
| A6 | DI via initializer, not property injection | WARNING | `var service: ServiceProtocol!` (should be `let` in init) |
| A7 | No concrete type references across layers | CRITICAL | `SupabaseRepository` used in ViewModel (should be protocol) |
| A8 | Naming conventions match project standards | WARNING | Check CLAUDE.md for expected naming patterns |

---

## Concurrency (Swift 6 Strict)

| # | Check | Severity | What to Look For |
|---|-------|----------|-----------------|
| C1 | @MainActor on all ViewModels | BLOCKER | `class FooViewModel: ObservableObject` without `@MainActor` |
| C2 | @Published only mutated from @MainActor | BLOCKER | `Task { self.items = result }` without MainActor isolation |
| C3 | Sendable conformance for cross-actor types | CRITICAL | Entities passed between tasks without Sendable |
| C4 | @Sendable on closures crossing actor boundaries | CRITICAL | Non-sendable closures passed to Task {} |
| C5 | Actor reentrancy safety | CRITICAL | State modified after `await` without rollback strategy |
| C6 | @unchecked Sendable justified | CRITICAL | Missing comment explaining thread safety |
| C7 | No mutable state on non-isolated classes | BLOCKER | `var` properties on classes without actor isolation |
| C8 | No completion handlers in new code | CRITICAL | `completionHandler:`, `callback:`, `completion:` parameters |
| C9 | Structured concurrency preferred | WARNING | `Task.detached` without justification |

---

## Security

| # | Check | Severity | What to Look For |
|---|-------|----------|-----------------|
| S1 | No hardcoded secrets | BLOCKER | API keys, tokens, passwords as string literals |
| S2 | Keychain for sensitive storage | BLOCKER | `UserDefaults` storing tokens, credentials, PII |
| S3 | No PII in logs | BLOCKER | Email, phone, name, SSN in print/logger calls |
| S4 | No force unwraps | CRITICAL | `!` operator (not `!=`) |
| S5 | No force casts | CRITICAL | `as!` type casting |
| S6 | Input validation at boundaries | CRITICAL | API handlers without input validation |
| S7 | Error messages don't leak internals | WARNING | Stack traces, file paths, SQL in error responses |
| S8 | Auth check before business logic | CRITICAL | Business logic executing before auth verification |
| S9 | Certificate pinning for sensitive APIs | WARNING | Network calls without SSL pinning |

---

## Performance

| # | Check | Severity | What to Look For |
|---|-------|----------|-----------------|
| P1 | Main thread not blocked | CRITICAL | Synchronous network/DB calls on main thread |
| P2 | View body under 30 lines | WARNING | Complex body with deep nesting |
| P3 | LazyVStack/LazyHStack for lists | WARNING | `VStack` or `HStack` with `ForEach` for large datasets |
| P4 | No duplicate network calls | WARNING | Same endpoint called multiple times without caching |
| P5 | Image caching | WARNING | Direct URL image loading without cache |
| P6 | No N+1 query patterns | WARNING | Loop with async call per item instead of batch |
| P7 | Unnecessary view redraws | WARNING | @StateObject vs @ObservedObject misuse |
| P8 | Memory leaks from closures | WARNING | Missing `[weak self]` in escaping closures |

---

## SwiftUI Patterns

| # | Check | Severity | What to Look For |
|---|-------|----------|-----------------|
| U1 | @StateObject for owned VMs | CRITICAL | @ObservedObject used when view creates the VM |
| U2 | @ObservedObject for borrowed VMs | CRITICAL | @StateObject used when parent passes the VM |
| U3 | View body max 3 nesting levels | WARNING | Deeply nested view hierarchies |
| U4 | Extracted @ViewBuilder subviews | WARNING | Large inline view blocks |
| U5 | No logic in view body | WARNING | Computation, filtering, sorting in body |
| U6 | Proper navigation patterns | WARNING | Manual navigation state instead of NavigationPath |
| U7 | Environment values for theming | NOTE | Hardcoded colors/fonts instead of environment |

---

## Accessibility

| # | Check | Severity | What to Look For |
|---|-------|----------|-----------------|
| X1 | VoiceOver labels on interactive elements | WARNING | Buttons, links, controls without `.accessibilityLabel()` |
| X2 | Dynamic Type support | WARNING | Fixed font sizes instead of `.font(.body)` |
| X3 | Sufficient color contrast | WARNING | Light text on light backgrounds |
| X4 | Accessibility traits | WARNING | Missing `.accessibilityAddTraits(.isButton)` on tap targets |
| X5 | Meaningful reading order | WARNING | VoiceOver reads elements in wrong sequence |
| X6 | Accessibility identifiers for tests | NOTE | Missing `.accessibilityIdentifier()` on testable elements |

---

## Error Handling

| # | Check | Severity | What to Look For |
|---|-------|----------|-----------------|
| E1 | No empty catch blocks | CRITICAL | `catch { }` or `catch { _ in }` |
| E2 | No bare try? | WARNING | `try?` without documented reason for ignoring error |
| E3 | LocalizedError conformance | WARNING | Custom errors without `errorDescription` |
| E4 | Error classification (retriable/permanent/fatal) | WARNING | All errors handled identically |
| E5 | User-facing error messages | WARNING | Technical errors shown to users |
| E6 | Error logging with context | WARNING | Errors caught without logging |

---

## Testing

| # | Check | Severity | What to Look For |
|---|-------|----------|-----------------|
| T1 | Tests exist for new logic | CRITICAL | New service/VM methods without corresponding tests |
| T2 | Arrange-Act-Assert structure | WARNING | Tests without clear structure |
| T3 | Test naming convention | WARNING | Not following `test_method_when_should` pattern |
| T4 | No test interdependence | CRITICAL | Tests that fail when run individually |
| T5 | Assertions present | CRITICAL | Tests that don't assert anything |
| T6 | Mock reset between tests | WARNING | setUp/tearDown missing mock reinitialization |
| T7 | No sleep in tests | WARNING | `Thread.sleep` or `Task.sleep` in tests |
| T8 | Factories for test data | NOTE | Inline object construction instead of factories |

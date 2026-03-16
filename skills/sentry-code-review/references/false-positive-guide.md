# Sentry False Positive Identification Guide

How to distinguish true positives from false positives in Sentry's static analysis findings for Swift/iOS code.

---

## General Methodology

Before marking any finding as a false positive, verify ALL of the following:

1. **Read the actual code** at the flagged file and line number (not Sentry's snippet, which may be stale)
2. **Check if the flagged path is reachable** -- trace callers backward
3. **Check if Swift's type system already prevents the issue** -- optionals, generics, access control
4. **Check if the code has changed since Sentry analyzed it** -- the finding may be stale
5. **Document your reasoning** -- "I checked and it's fine" is not documentation

---

## Common False Positives in Swift

### 1. Optional Chaining Flagged as Missing Nil Check

**What Sentry says:** "Potential nil dereference at line X"

**Why it's often a false positive:** Sentry may not recognize that Swift's optional chaining (`?.`) already handles the nil case safely.

```swift
// Sentry flags this:
let name = user?.profile?.displayName
// But optional chaining returns nil safely -- no crash possible

// TRUE positive would be:
let name = user!.profile!.displayName
// Force unwrap WILL crash if nil
```

**How to verify:** Check if the flagged access uses `?.` (safe) or `!` (unsafe). If `?.`, it's likely a false positive.

---

### 2. SwiftUI body Property Flagged as Unused Variable

**What Sentry says:** "Variable 'X' is declared but never used"

**Why it's a false positive:** SwiftUI's `body` computed property is called by the framework, not by user code. Variables declared in `body` and used in the returned view hierarchy are effectively used.

```swift
struct TransactionRow: View {
    let transaction: Transaction

    var body: some View {
        // Sentry may flag 'formattedAmount' as unused
        let formattedAmount = transaction.amount.formatted(.currency(code: "USD"))
        HStack {
            Text(transaction.description)
            Spacer()
            Text(formattedAmount) // It IS used here
        }
    }
}
```

**How to verify:** Check if the "unused" variable is referenced in the view hierarchy returned by `body`. If yes, false positive.

---

### 3. @Published Property Access Patterns

**What Sentry says:** "Property 'X' written but never read" or "Potential data race on property 'X'"

**Why it's often a false positive:** `@Published` properties are observed by SwiftUI views. The "read" happens in the view layer via Combine's observation mechanism, which Sentry may not model.

```swift
@MainActor
final class ViewModel: ObservableObject {
    @Published var items: [Item] = [] // Sentry: "written but never read"

    func load() async {
        items = try? await api.fetchItems() ?? []
        // items IS read -- by the SwiftUI view observing this ViewModel
    }
}
```

**How to verify:** Check if a SwiftUI view uses `@ObservedObject`, `@StateObject`, or `@EnvironmentObject` with this ViewModel. If yes, the `@Published` property IS read.

---

### 4. Protocol Conformance Methods Flagged as Dead Code

**What Sentry says:** "Function 'X' is never called"

**Why it's a false positive:** Protocol methods may be called by the framework (UITableViewDelegate, URLSessionDelegate, etc.), not by your code directly.

```swift
// Sentry may flag this as "never called"
extension MyVC: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        // Called by UIKit, not by user code
    }
}
```

**How to verify:** Check if the method is part of a protocol conformance. If the protocol is a framework delegate/data source, false positive.

---

### 5. Enum Case Exhaustiveness

**What Sentry says:** "Missing handling for case 'X'"

**Why it's sometimes a false positive:** Swift's compiler enforces exhaustive switch statements. If the code compiles, all cases are handled (including via `default`).

```swift
// Sentry may flag this:
switch state {
case .loading: showSpinner()
case .loaded(let data): showData(data)
case .error(let error): showError(error)
// No default needed -- Swift compiler verifies exhaustiveness
}
```

**How to verify:** If the code compiles without warnings on the switch statement, Swift has verified exhaustiveness. False positive.

**Exception:** If using `@unknown default`, Sentry may correctly flag that new enum cases aren't explicitly handled.

---

### 6. Computed Property Side Effects

**What Sentry says:** "Property getter has side effects"

**Why it's sometimes a false positive in SwiftUI:** SwiftUI's `body` property IS a computed property with "side effects" (building the view hierarchy). This is the intended usage pattern.

**When it IS a true positive:** Computed properties that modify state, trigger network calls, or write to disk. These are genuine code quality issues.

---

### 7. Guard-Let Return Flagged as Unreachable Code

**What Sentry says:** "Code after guard statement is unreachable" or "Return value is unused"

**Why it's sometimes a false positive:** Sentry may not model all paths correctly. A guard-let with early return is intentional.

```swift
func processTransaction(_ tx: Transaction?) {
    guard let transaction = tx else {
        logger.warning("nil_transaction")
        return // This is intentional, not dead code
    }
    // Sentry may flag code below as potentially unreachable
    process(transaction)
}
```

---

### 8. Sendable Conformance Warnings

**What Sentry says:** "Type 'X' does not conform to Sendable"

**Why it's sometimes a false positive:** In Swift 5 compatibility mode, many Sendable warnings are advisory. The code works correctly but may need updates for Swift 6.

**When it IS a true positive:** If the type is actually passed across concurrency boundaries AND contains mutable state, this is a real data race risk.

**How to verify:** Check if the type is actually used across actor/task boundaries. If it's only used within a single actor context, the warning is low risk.

---

## When to Fix Anyway (Defensive Programming)

Even if a finding is technically a false positive, fix it if:

1. **The fix is trivial** (< 2 minutes) and makes the code clearer
2. **The code is in a security-critical path** -- defense in depth is worth the effort
3. **The pattern could become a true positive** if someone modifies the code later (e.g., adding a new enum case)
4. **The finding will recur** on every PR touching this file, creating noise

---

## When to Mark as False Positive

Mark as false positive and document when:

1. **The fix would make the code worse** -- adding unnecessary nil checks on non-optional values
2. **The finding contradicts Swift's type system** -- the language already guarantees safety
3. **The finding is about framework patterns** -- SwiftUI body, UIKit delegates, protocol conformance
4. **The finding is stale** -- code was already changed in the same PR

---

## Documentation Template for False Positive

When dismissing a finding, use this format in the PR summary:

```
### False Positive: [Sentry Finding Title]
- **File:** path/to/file.swift:42
- **Sentry claim:** [what Sentry said]
- **Why it's false positive:** [specific reasoning]
- **Verified by:** [what you checked -- "read current code", "traced callers", "type system guarantees"]
```

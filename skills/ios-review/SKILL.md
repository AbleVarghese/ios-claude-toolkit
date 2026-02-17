---
name: ios-review
description: "This skill should be used when the user asks to 'review iOS code', 'check architecture compliance', 'audit SwiftUI patterns', 'run a code quality audit', or 'review performance'. Provides comprehensive iOS-specific code review with security, architecture, and quality focus."
user-invocable: true
allowed-tools:
  - Read
  - Grep
  - Glob
---

# iOS Code Review

Perform comprehensive iOS-specific code review covering architecture compliance, SwiftUI patterns, performance, security, and accessibility.

> **Project Discovery**: Before executing, determine project-specific values (project name, scheme, bundle ID, target, architecture layers, ViewModel base class, dependency flow) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

## Review Areas

### Security Analysis
- Check for force unwraps (!) in production code
- Verify proper error handling patterns
- Validate sensitive data protection (Keychain, not UserDefaults)
- Review authentication and session management
- Ensure no PII in application logs

### Architecture Compliance
- Verify dependency flow matches the project's documented architecture
- Domain layer has zero external framework imports
- Protocol-based dependency injection (no `.shared` singletons)
- Proper service layer separation between presentation, business logic, and infrastructure
- Check for architectural drift (e.g., views calling repositories directly)

### iOS Best Practices
- async/await usage (no completion handlers)
- SwiftUI performance patterns (avoid unnecessary view redraws)
- Memory management and lifecycle (no retain cycles, proper weak references)
- Threading and @MainActor compliance on all ViewModels
- Strict concurrency compliance (SWIFT_STRICT_CONCURRENCY: complete)
- Sendable conformance for types crossing concurrency domains

### SwiftUI Patterns
- Proper use of @State, @Binding, @StateObject, @ObservedObject, @EnvironmentObject
- View composition and reusability
- Navigation patterns and deep linking
- Accessibility labels and traits
- Dynamic Type support

### Performance
- View body complexity (keep views simple, extract subviews)
- Lazy loading for collections (LazyVStack, LazyHStack)
- Image caching and optimization
- Network request efficiency (no duplicate calls)
- Memory footprint monitoring

### Domain-Specific Requirements
- Check CLAUDE.md for any domain-specific compliance requirements (e.g., financial compliance, healthcare HIPAA, etc.)
- Audit trail completeness for regulated operations
- Data integrity and transaction safety where applicable

## Review Process

1. **File Analysis** -- Review Swift files for patterns, violations, and anti-patterns
2. **Architecture Validation** -- Check layer separation and dependency direction (inward only)
3. **Security Scan** -- Identify potential vulnerabilities and data exposure risks
4. **Performance Review** -- Flag efficiency concerns, unnecessary allocations, main thread blocking
5. **Test Coverage** -- Verify TDD compliance and meaningful test assertions
6. **Accessibility Audit** -- Check VoiceOver support, Dynamic Type, and contrast ratios

## Common Issues to Flag

- Force unwraps in production code
- Missing @MainActor on ViewModels
- Direct concrete type usage instead of protocol injection
- Domain layer importing external frameworks
- Completion handlers instead of async/await
- Missing error handling (try without catch)
- Retain cycles in closures (missing [weak self])
- Hardcoded strings instead of localization keys
- Mutable shared state without proper synchronization
- `@Published` properties modified off the main actor

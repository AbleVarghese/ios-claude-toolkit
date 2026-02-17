---
name: cupertino-apple
description: "This skill should be used when the user asks about 'App Store submission', 'Apple guidelines compliance', 'Human Interface Guidelines', 'privacy requirements', or 'App Store rejection prevention'. Provides pre-submission checklists, HIG validation, and financial app compliance for Apple platforms."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: false
---

# Apple Ecosystem & App Store Compliance

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

Ensure the iOS app achieves and maintains Apple App Store approval through strict adherence to Apple guidelines, HIG principles, and iOS platform best practices.

## Standard Operating Procedures

### SOP-1: Pre-Submission Checklist
```
- App Store Review Guidelines checked (see references/app-store-guidelines.md)
- Human Interface Guidelines followed (see references/hig-checklist.md)
- Privacy Policy URL configured
- Privacy Labels accurate
- Contact information complete
- Age rating accurate
- Screenshots meet requirements (6.5", 5.5")
- App Preview video (optional)
- Keywords optimized
- Category correct (e.g., Finance, Productivity, Social)
- Support URL working
- Marketing URL working
```

### SOP-2: Privacy Requirements
```xml
<!-- Info.plist Privacy Descriptions -->
<!-- Include only the entries relevant to your app's functionality -->

<key>NSCameraUsageDescription</key>
<string>Camera access is needed for identity verification and document capture.</string>

<key>NSFaceIDUsageDescription</key>
<string>Face ID is used to securely authenticate you and protect your account.</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>Photo library access is needed to upload profile pictures and documents.</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>Location is used to provide relevant services based on your region.</string>
```

### SOP-3: Financial App Requirements
```
Apple Requirements for Financial/Money Apps:
1. Licensed financial entity (or partnered with one)
2. Clear fee disclosure before transaction
3. Transaction receipts/confirmation
4. Customer support contact
5. Regulatory compliance disclosure
6. KYC/AML procedures (if applicable)
7. Fraud prevention measures
8. Secure data handling
```

### SOP-4: App Store Connect Upload
```bash
# Upload via Fastlane
fastlane deliver --skip_screenshots --skip_metadata
```

## Swift & SwiftUI Patterns

### Swift 6 Concurrency
```swift
// Correct actor usage
@MainActor
final class ItemViewModel: ObservableObject {
    @Published var items: [Item] = []

    func loadItems() async {
        items = await repository.fetchAll()
    }
}
```

### SwiftUI Environment
```swift
// Correct environment usage
struct DetailView: View {
    @Environment(\.dismiss) private var dismiss
    @Environment(\.colorScheme) private var colorScheme

    var body: some View {
        // View code
    }
}
```

## Demo Account for Review
```
Email: {DEMO_ACCOUNT_EMAIL}
Password: [Configure secure password before submission]
Notes: "Use test card 4242 4242 4242 4242 for payments (if applicable)"
```

Set up a dedicated demo account before App Store submission. Provide credentials in the App Review Information section of App Store Connect.

## References

- Detailed App Store Review Guidelines tables: `references/app-store-guidelines.md`
- HIG checklist and rejection prevention: `references/hig-checklist.md`
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [Swift Documentation](https://docs.swift.org/swift-book/)
- [SwiftUI Documentation](https://developer.apple.com/documentation/swiftui)

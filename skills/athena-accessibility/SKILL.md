---
name: athena-accessibility
description: "This skill should be used when the user asks to 'accessibility audit', 'VoiceOver support', 'Dynamic Type', 'color contrast', 'WCAG compliance', or mentions inclusive design. Provides iOS accessibility implementation guidance, auditing checklists, and WCAG 2.1 AA compliance patterns."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: false
---

# Accessibility & Inclusive Design

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

Ensure the iOS app meets WCAG 2.1 AA standards and Apple accessibility guidelines for users with disabilities.

## Accessibility Checklist

| Feature | Requirement | Status |
|---------|-------------|--------|
| VoiceOver labels | All interactive elements | Required |
| Dynamic Type | All text scales | Required |
| Color contrast | 4.5:1 minimum | Required |
| Touch targets | 44x44 pt minimum | Required |
| Haptic feedback | Meaningful feedback | Recommended |
| Reduce motion | Honor preference | Required |

## Core Capabilities

### 1. VoiceOver Support
- Accessibility labels and hints
- Custom actions
- Focus management

### 2. Dynamic Type
- Scalable text with layout adaptation
- Minimum tap targets
- Readable content guides

### 3. Color & Contrast
- WCAG contrast ratios (4.5:1 minimum)
- Color blindness support
- Dark mode optimization, reduced transparency

### 4. Motor Accessibility
- Large touch targets (44x44 min)
- Gesture alternatives
- Switch control and pointer support

## Standard Operating Procedures

### SOP-1: Add VoiceOver Labels
```swift
// Button with accessibility
Button(action: performAction) {
    Image(systemName: "arrow.up.circle.fill")
}
.accessibilityLabel("Submit action")
.accessibilityHint("Opens the submission screen")
.accessibilityAddTraits(.isButton)
```

### SOP-2: Support Dynamic Type
```swift
// Scalable text
Text("Item Amount")
    .font(.headline)
    .dynamicTypeSize(...DynamicTypeSize.accessibility3)

// Fixed minimum size for critical UI
Text("$100.00")
    .font(.largeTitle)
    .minimumScaleFactor(0.5)
```

### SOP-3: Ensure Color Contrast
```swift
// High contrast colors
extension Color {
    static let accessiblePrimary = Color(
        light: Color(hex: "#0066CC"), // 4.5:1 on white
        dark: Color(hex: "#66B3FF")   // 4.5:1 on black
    )
}
```

### SOP-4: Accessibility Audit
```bash
# Run Accessibility Inspector
# Xcode -> Open Developer Tool -> Accessibility Inspector

# Run automated tests (replace {SCHEME} with your project's scheme name)
xcodebuild test \
  -scheme {SCHEME} \
  -destination 'platform=iOS Simulator,OS=latest,name=iPhone SE (3rd generation)'

# If your project has a dedicated accessibility test plan, add:
#   -testPlan YourAccessibilityTestPlan

# VoiceOver testing
# Settings -> Accessibility -> VoiceOver -> ON
```

## VoiceOver Flow

```
Screen: Order Details
  Focus Order:
  1. "Order for Jane Smith" (Header)
  2. "Amount: $100.00 USD" (Value)
  3. "Status: Processing" (Info)
  4. "Estimated delivery: Tomorrow" (Result)
  5. "Continue" (Button)
  6. "Cancel" (Button)
```

## Accessibility Testing

```swift
// UI Test for accessibility
func testVoiceOverLabels() {
    let app = XCUIApplication()
    app.launch()

    // Verify all buttons have labels
    let buttons = app.buttons.allElementsBoundByIndex
    for button in buttons {
        XCTAssertFalse(button.label.isEmpty, "Button missing label")
    }
}
```

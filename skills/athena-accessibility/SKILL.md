---
name: athena-accessibility
description: "VoiceOver, screen reader, accessible, disability, WCAG. iOS accessibility implementation and WCAG 2.1 AA compliance — VoiceOver labels, Dynamic Type, color contrast, touch targets, accessibility auditing. Trigger on: 'make this accessible', 'support assistive technologies', 'screen reader support', 'Dynamic Type', 'color contrast check', 'touch target too small', 'inclusive design', 'ADA compliance', 'can a blind user use this', or when any UI work should be checked for accessibility. For localizing accessibility labels, also see babel-localization."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: false
---

# Accessibility & Inclusive Design

> **Iron Law:** "Accessibility is not optional. 1 in 4 adults has a disability -- if your app excludes them, it's broken."

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

Ensure the iOS app meets WCAG 2.1 AA standards and Apple accessibility guidelines. This skill covers VoiceOver, Dynamic Type, color contrast, motor accessibility, custom controls, focus management, and accessibility testing.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|----------------|-----------------|
| "Most of our users don't have disabilities" | 26% of US adults have a disability (CDC). Even "temporary" disabilities count -- a broken arm, bright sunlight, loud environment. Accessibility benefits everyone. | Implement accessibility from the start. It's cheaper to build in than retrofit. |
| "VoiceOver is too complex to support" | VoiceOver support is mostly adding labels and traits -- work you should already be doing. SwiftUI provides reasonable defaults. The complex part is fixing the bad code that breaks those defaults. | Add `.accessibilityLabel()` to every interactive element. Test with VoiceOver for 5 minutes per screen. |
| "We'll add accessibility later" | "Later" never comes. Every sprint adds more inaccessible UI. The backlog grows exponentially. Teams that defer accessibility ship apps that 25% of the population cannot use. | Make accessibility part of your definition of done for every feature. No PR merges without accessibility labels on interactive elements. |
| "The system handles accessibility automatically" | SwiftUI and UIKit provide baseline support, but custom views, complex gestures, decorative images, and grouped elements all need manual intervention. "Automatic" accessibility is often wrong -- unlabeled buttons, nonsensical reading order, missing context. | Audit every screen with VoiceOver. Automatic support covers ~60% of cases. You must handle the other 40%. |
| "Accessibility testing is too time-consuming" | A basic VoiceOver walkthrough takes 5-10 minutes per screen. Automated accessibility audits run in seconds. This is less time than debugging a production crash. | Add accessibility checks to your PR checklist. Run Accessibility Inspector on every new screen. |
| "Only blind users need VoiceOver" | VoiceOver users include people with low vision, motor disabilities (using Switch Control), cognitive disabilities, and situational disabilities. Voice Control uses the same accessibility tree. If VoiceOver doesn't work, Voice Control doesn't work. | Test with VoiceOver AND Voice Control. Both depend on the same accessibility labels. |
| "Our design doesn't allow for 44x44 touch targets" | Then your design is wrong. Apple's HIG mandates 44x44pt minimum. Small targets cause frustration for all users and are impossible for users with motor impairments. | Redesign. Use `.contentShape(Rectangle())` to expand hit areas beyond visual bounds if needed. |
| "Dark mode is enough for low vision" | Dark mode helps some users but does nothing for contrast ratios, text scaling, color blindness, or screen magnification. Low vision support requires Dynamic Type, sufficient contrast, and scalable layouts. | Support Dynamic Type up to accessibility sizes. Verify 4.5:1 contrast ratios. Test with Zoom enabled. |

---

## Red Flags -- STOP

- **Images without accessibility labels**: Every `Image` that conveys information needs `.accessibilityLabel()`. Decorative images need `.accessibilityHidden(true)`.
- **Interactive elements smaller than 44x44pt**: Buttons, links, toggles, and any tappable element must meet minimum touch target size. No exceptions.
- **Color as the only differentiator**: "Red means error, green means success" fails for 8% of men with color vision deficiency. Always use icons, text, or patterns in addition to color.
- **Missing Dynamic Type support**: If text doesn't scale with the system text size setting, users with low vision cannot use your app. Every `Text` must use dynamic type styles.
- **Form fields without labels**: Text fields, toggles, pickers, and other form elements must have associated labels that VoiceOver reads. Placeholder text is NOT a label -- it disappears on input.
- **Custom gestures without alternatives**: If a feature requires a complex gesture (pinch, rotate, multi-finger swipe), provide an accessible alternative via `.accessibilityAction()` or a button.

---

## When NOT to Use This Skill

- **Designing the visual UI**: This skill covers accessibility implementation, not visual design. For HIG-compliant visual design, see **cupertino-apple**.
- **Server-side accessibility**: WCAG for web APIs (screen reader-compatible error messages, semantic HTML) belongs in web development, not iOS.
- **Automated test infrastructure setup**: For setting up XCTest and test infrastructure, see **forge-development**. This skill covers what to test for accessibility, not how to configure test runners.
- **Localization of accessibility labels**: If you need to localize VoiceOver labels into multiple languages, see **babel-localization** for string externalization.

---

## Decision Framework

```
What aspect of accessibility am I addressing?

VISUAL
  |-- Can the user see the content?
  |     NO  --> Add VoiceOver labels (.accessibilityLabel)
  |     YES but low vision --> Support Dynamic Type + verify contrast ratios
  |
  |-- Is color the only indicator?
        YES --> Add icon, text, or pattern as secondary indicator
        NO  --> Verify 4.5:1 contrast ratio for text, 3:1 for large text

MOTOR
  |-- Is the touch target >= 44x44pt?
  |     NO  --> Expand with .contentShape() or redesign
  |
  |-- Does it require complex gestures?
        YES --> Add .accessibilityAction() alternatives
        NO  --> Verify works with Switch Control

COGNITIVE
  |-- Is the navigation predictable?
  |     NO  --> Use standard navigation patterns (NavigationStack, TabView)
  |
  |-- Are instructions clear?
        NO  --> Add .accessibilityHint() to explain non-obvious interactions

DYNAMIC CONTENT
  |-- Does content update without user action?
        YES --> Post accessibility notification:
               UIAccessibility.post(notification: .announcement, argument: message)
        NO  --> Standard labels are sufficient
```

---

## Accessibility Requirements Summary

| Feature | Requirement | WCAG Criterion | Priority |
|---------|-------------|---------------|----------|
| VoiceOver labels | All interactive elements labeled | 1.1.1 Non-text Content | Required |
| Dynamic Type | All text scales to accessibility sizes | 1.4.4 Resize Text | Required |
| Color contrast | 4.5:1 text, 3:1 large text/UI | 1.4.3 Contrast | Required |
| Touch targets | 44x44 pt minimum | 2.5.8 Target Size | Required |
| Focus order | Logical reading/navigation order | 2.4.3 Focus Order | Required |
| Reduce Motion | Honor `UIAccessibility.isReduceMotionEnabled` | 2.3.1 Three Flashes | Required |
| Error identification | Errors identified by more than color | 3.3.1 Error Identification | Required |
| Haptic feedback | Meaningful, not decorative | Best Practice | Recommended |
| Reduce Transparency | Honor transparency preferences | Best Practice | Recommended |

---

## VoiceOver Implementation

### Labels, Hints, and Values

```swift
// Label: WHAT the element is (read first)
Button(action: sendMoney) {
    Image(systemName: "arrow.up.circle.fill")
}
.accessibilityLabel("Send money")

// Hint: WHAT HAPPENS when activated (read after a pause)
.accessibilityHint("Opens the send money screen")

// Value: CURRENT STATE for stateful elements
Slider(value: $amount, in: 0...1000)
    .accessibilityValue("\(Int(amount)) dollars")
```

### Traits

Traits tell VoiceOver what kind of element this is, which changes how it announces and interacts:

```swift
// Add traits for semantic meaning
Text("Recent Transfers")
    .accessibilityAddTraits(.isHeader)
    // VoiceOver announces: "Recent Transfers, heading"

Image("promo_banner")
    .accessibilityAddTraits(.isImage)
    .accessibilityLabel("Spring promotion: 50% off transfer fees")

Link("Terms of Service", destination: termsURL)
    .accessibilityAddTraits(.isLink)
    // VoiceOver announces: "Terms of Service, link"

// Remove incorrect default traits
Button(action: {}) {
    Text("Status: Active")
}
.accessibilityRemoveTraits(.isButton)
// If this "button" is actually just styled text, remove the button trait
```

Available traits: `.isButton`, `.isLink`, `.isHeader`, `.isImage`, `.isSearchField`, `.isKeyboardKey`, `.isStaticText`, `.playsSound`, `.startsMediaSession`, `.isModal`, `.isSelected`, `.isSummaryElement`, `.updatesFrequently`, `.allowsDirectInteraction`, `.causesPageTurn`, `.isToggle`, `.tabBar`

### Custom Actions

Custom actions appear when the user swipes up/down on an element, providing additional operations:

```swift
// A transfer row with multiple actions
TransferRow(transfer: transfer)
    .accessibilityElement(children: .combine)
    .accessibilityLabel("\(transfer.recipientName), \(transfer.amount.formatted(currency: .usd)), \(transfer.status)")
    .accessibilityAction(named: "View details") {
        showDetail(transfer)
    }
    .accessibilityAction(named: "Repeat transfer") {
        repeatTransfer(transfer)
    }
    .accessibilityAction(named: "Cancel transfer") {
        cancelTransfer(transfer)
    }
```

### Magic Tap and Escape

```swift
// Magic Tap: two-finger double-tap, for the most important action
ContentView()
    .accessibilityAction(.magicTap) {
        togglePlayPause()
    }

// Escape: two-finger scrub (Z gesture), for dismissing
ModalView()
    .accessibilityAction(.escape) {
        dismiss()
    }
```

---

## Element Grouping and Children

### Combining Children

When a card has multiple text elements that should be read as one unit:

```swift
// WRONG: VoiceOver focuses each element separately
VStack {
    Text("Jane Smith")        // Focus 1: "Jane Smith"
    Text("$500.00")           // Focus 2: "$500.00"
    Text("Completed")         // Focus 3: "Completed"
}

// CORRECT: Combined into single focus point
VStack {
    Text("Jane Smith")
    Text("$500.00")
    Text("Completed")
}
.accessibilityElement(children: .combine)
// VoiceOver reads: "Jane Smith, $500.00, Completed"
```

### Containing Children

When you want VoiceOver to recognize a group but still focus individual children:

```swift
VStack {
    Text("Transfer Details").accessibilityAddTraits(.isHeader)
    Text("Amount: $500")
    Text("Fee: $5")
}
.accessibilityElement(children: .contain)
// VoiceOver can focus each child, but knows they're grouped
```

### Ignoring Children

When the children's default accessibility is wrong and you want full manual control:

```swift
// Custom chart where default accessibility is meaningless
ChartView(data: chartData)
    .accessibilityElement(children: .ignore)
    .accessibilityLabel("Transfer volume chart")
    .accessibilityValue("Showing 30 days. Peak: March 10 with 45 transfers. Average: 28 transfers per day.")
```

### Custom Representation

For complex controls, provide a simpler accessible representation:

```swift
// A custom circular slider
CircularSlider(value: $percentage)
    .accessibilityRepresentation {
        Slider(value: $percentage, in: 0...100) {
            Text("Transfer progress")
        }
    }
```

---

## Focus Management

### Controlling VoiceOver Focus

```swift
struct TransferConfirmation: View {
    @AccessibilityFocusState private var focusedElement: FocusableElement?

    enum FocusableElement: Hashable {
        case confirmationMessage
        case doneButton
    }

    var body: some View {
        VStack {
            Text("Transfer sent successfully!")
                .accessibilityFocused($focusedElement, equals: .confirmationMessage)

            Button("Done") { dismiss() }
                .accessibilityFocused($focusedElement, equals: .doneButton)
        }
        .onAppear {
            // Move VoiceOver focus to the confirmation message
            focusedElement = .confirmationMessage
        }
    }
}
```

### Announcements for Dynamic Content

When content changes without a screen transition, announce the change:

```swift
func transferCompleted() {
    // Update UI
    status = .completed

    // Announce to VoiceOver users
    UIAccessibility.post(
        notification: .announcement,
        argument: "Transfer completed successfully. $500 sent to Jane Smith."
    )
}

func showError(_ message: String) {
    errorMessage = message

    // Screen changed notification moves focus to the error
    UIAccessibility.post(
        notification: .screenChanged,
        argument: errorLabel  // UIView to focus, or String to announce
    )
}

// Layout changed: use when content layout changes but screen doesn't
func expandSection() {
    isExpanded.toggle()
    UIAccessibility.post(notification: .layoutChanged, argument: nil)
}
```

### Sort Priority

Control the reading order when the visual layout doesn't match the logical order:

```swift
HStack {
    StatusIcon()
        .accessibilitySortPriority(1)  // Read second

    TransferInfo()
        .accessibilitySortPriority(2)  // Read first (higher = earlier)

    ActionButton()
        .accessibilitySortPriority(0)  // Read last
}
```

---

## Dynamic Type Support

### Scalable Text

```swift
// CORRECT: Uses dynamic type style (scales automatically)
Text("Transfer Amount")
    .font(.headline)

// CORRECT: Custom font with dynamic scaling
Text("$500.00")
    .font(.custom("Avenir-Heavy", size: 24, relativeTo: .title))

// Limit maximum scale for critical UI (if needed, use sparingly)
Text("PIN Entry")
    .font(.body)
    .dynamicTypeSize(...DynamicTypeSize.accessibility1)

// WRONG: Fixed font size (never scales)
Text("Amount").font(.system(size: 16))
```

### Layout Adaptation for Large Text

```swift
// Adapt layout when text size is large
@Environment(\.dynamicTypeSize) var typeSize

var body: some View {
    if typeSize.isAccessibilitySize {
        // Stack vertically for very large text
        VStack(alignment: .leading) {
            label
            value
        }
    } else {
        // Side by side for standard sizes
        HStack {
            label
            Spacer()
            value
        }
    }
}
```

### Minimum Tap Targets with Dynamic Type

```swift
// Ensure tap targets remain large even when text is small
Button("Send") { }
    .frame(minWidth: 44, minHeight: 44)
    // Or expand the tappable area without changing visual size:
    .contentShape(Rectangle().size(width: 44, height: 44))
```

---

## Color and Contrast

### WCAG Contrast Requirements

| Element Type | Minimum Ratio | Example |
|-------------|--------------|---------|
| Normal text (< 18pt) | 4.5:1 | Body text, labels, captions |
| Large text (>= 18pt or >= 14pt bold) | 3:1 | Headlines, titles |
| UI components (icons, borders) | 3:1 | Buttons, form fields, icons |
| Decorative elements | No requirement | Backgrounds, dividers |

### High Contrast Color Definitions

```swift
extension Color {
    // Verify contrast ratios in Accessibility Inspector
    static let accessiblePrimary = Color(
        light: Color(red: 0, green: 0.4, blue: 0.8),   // #0066CC -- 4.5:1 on white
        dark: Color(red: 0.4, green: 0.7, blue: 1.0)    // #66B3FF -- 4.5:1 on black
    )

    static let accessibleError = Color(
        light: Color(red: 0.8, green: 0, blue: 0),      // #CC0000 -- 4.6:1 on white
        dark: Color(red: 1.0, green: 0.4, blue: 0.4)    // #FF6666 -- 5.1:1 on black
    )
}
```

### Beyond Color

```swift
// WRONG: Color is the only differentiator
Text("Status")
    .foregroundColor(isError ? .red : .green)

// CORRECT: Color + icon + text
HStack {
    Image(systemName: isError ? "xmark.circle.fill" : "checkmark.circle.fill")
        .foregroundColor(isError ? .accessibleError : .accessibleSuccess)
    Text(isError ? "Transfer failed" : "Transfer complete")
        .foregroundColor(isError ? .accessibleError : .accessibleSuccess)
}
```

### Respecting User Preferences

```swift
@Environment(\.colorSchemeContrast) var contrast

var body: some View {
    Text("Amount")
        .foregroundColor(contrast == .increased ? .primary : .secondary)
}

// Reduce Motion
@Environment(\.accessibilityReduceMotion) var reduceMotion

var body: some View {
    if reduceMotion {
        content.transition(.opacity)
    } else {
        content.transition(.slide.combined(with: .opacity))
    }
}

// Reduce Transparency
@Environment(\.accessibilityReduceTransparency) var reduceTransparency

var body: some View {
    background
        .opacity(reduceTransparency ? 1.0 : 0.8)
}
```

---

## Accessible Custom Controls

When building non-standard UI elements, you must make them accessible manually:

### Custom Toggle

```swift
struct CustomToggle: View {
    @Binding var isOn: Bool

    var body: some View {
        Button {
            isOn.toggle()
        } label: {
            HStack {
                Text("Notifications")
                Spacer()
                Circle()
                    .fill(isOn ? Color.green : Color.gray)
                    .frame(width: 24, height: 24)
            }
        }
        .accessibilityElement(children: .ignore)
        .accessibilityLabel("Notifications")
        .accessibilityValue(isOn ? "On" : "Off")
        .accessibilityAddTraits(.isToggle)
        .accessibilityAction {
            isOn.toggle()
        }
    }
}
```

### Custom Stepper

```swift
struct AmountStepper: View {
    @Binding var amount: Int

    var body: some View {
        HStack {
            Button("-") { amount -= 10 }
            Text("$\(amount)")
            Button("+") { amount += 10 }
        }
        .accessibilityElement(children: .ignore)
        .accessibilityLabel("Transfer amount")
        .accessibilityValue("\(amount) dollars")
        .accessibilityAdjustableAction { direction in
            switch direction {
            case .increment: amount += 10
            case .decrement: amount = max(0, amount - 10)
            @unknown default: break
            }
        }
    }
}
```

---

## Voice Control Support

Voice Control uses accessibility labels as voice commands. Provide alternative labels for Voice Control users:

```swift
Button("Send $500 to Jane") { }
    .accessibilityInputLabels([
        "Send 500 to Jane",     // Most specific
        "Send to Jane",         // Less specific
        "Send"                  // Least specific
    ])
// User can say any of these to activate the button
```

---

## Phase Gate: ACCESSIBILITY AUDIT ✓

**Cannot mark accessibility work complete until:**
- [ ] Every interactive element has an accessibility label
- [ ] Every image has either a label or is marked decorative (.accessibilityHidden(true))
- [ ] Touch targets are minimum 44x44 points
- [ ] Dynamic Type tested at largest accessibility size
- [ ] Color contrast verified at 4.5:1 minimum ratio

**Hard Stop**: An interactive element without an accessibility label is invisible to VoiceOver users. This is not a polish item — it's a broken feature for 25% of users. Fix before proceeding.

### Self-Audit

1. Did I test with VoiceOver enabled? (Not just check labels exist — actually navigate)
2. Does every screen work at the largest Dynamic Type size without truncation or overlap?
3. Is color NEVER the only way to convey information?
4. Can every action be performed without precise motor control?

---

## Quality Gates (Before Marking Complete)

1. Every interactive element has an `.accessibilityLabel()` that describes its purpose
2. Every image has either `.accessibilityLabel()` (informative) or `.accessibilityHidden(true)` (decorative)
3. All touch targets meet 44x44pt minimum
4. All text uses Dynamic Type styles and scales to accessibility sizes
5. Color contrast meets WCAG AA ratios (4.5:1 normal text, 3:1 large text)
6. Color is never the sole means of conveying information
7. Animations respect `accessibilityReduceMotion`
8. Form fields have labels (not just placeholders)
9. Dynamic content updates are announced via `UIAccessibility.post(notification:)`
10. VoiceOver walkthrough completed for every new or modified screen

---

## Cross-Skill References

- **babel-localization**: All accessibility labels must be localized. VoiceOver reads in the user's selected language.
- **cupertino-apple**: Apple's App Store review includes accessibility checks. Apps that don't support Dynamic Type or VoiceOver may be rejected under Guideline 4.0 (Design).
- **forge-development**: Accessibility tests are part of the test suite. XCUITest provides accessibility queries for automated verification.
- **sentry-code-review**: Code reviews should check for accessibility labels on all interactive elements.

---

## References

- Testing methodology: `references/testing-accessibility.md`
- SwiftUI modifiers reference: `references/swiftui-a11y.md`
- [Apple: Accessibility](https://developer.apple.com/accessibility/)
- [WCAG 2.1 Quick Reference](https://www.w3.org/WAI/WCAG21/quickref/)
- [Apple: Supporting VoiceOver in Your App](https://developer.apple.com/documentation/accessibility/supporting-voiceover-in-your-app)

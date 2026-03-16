# Accessibility Testing Reference

Comprehensive testing methodology for iOS accessibility compliance.

---

## Accessibility Inspector Workflow

Accessibility Inspector is Xcode's built-in tool for auditing accessibility. It runs on Simulator or real devices.

### Opening Accessibility Inspector

1. Xcode > Open Developer Tool > Accessibility Inspector
2. Select target: Simulator or connected device from the dropdown
3. Use the crosshair tool to inspect individual elements

### What to Inspect Per Element

For every interactive element on screen, verify:

| Property | What to Check | Failure Example |
|----------|--------------|-----------------|
| Label | Descriptive, concise, not empty | "button" or "" |
| Value | Current state for stateful elements | Slider with no value |
| Traits | Correct element type | Button without `.isButton` trait |
| Frame | Meets 44x44pt minimum | 24x24 icon button |
| Contrast | Passes ratio check | Light gray text on white |

### Running an Audit

1. Navigate to the screen you want to audit
2. In Accessibility Inspector: click the Audit icon (checkmark in circle)
3. Click "Run Audit"
4. Inspector reports issues: missing labels, small touch targets, contrast failures
5. Click each issue to see the affected element highlighted on screen

### Common Audit Findings

| Finding | Severity | Fix |
|---------|----------|-----|
| "Element has no description" | Critical | Add `.accessibilityLabel()` |
| "Touch target too small" | Critical | Expand to 44x44pt minimum |
| "Contrast ratio insufficient" | High | Adjust colors to meet 4.5:1 |
| "Element not accessible" | Critical | Add `.accessibilityElement()` or remove `.accessibilityHidden()` |
| "Redundant traits" | Low | Remove duplicate traits |

---

## VoiceOver Manual Testing Script

For every screen, follow this script. Budget 5-10 minutes per screen.

### Setup

1. Enable VoiceOver: Settings > Accessibility > VoiceOver > On
2. Or use Accessibility Shortcut: Settings > Accessibility > Accessibility Shortcut > VoiceOver (then triple-click side button to toggle)

### VoiceOver Gestures Reference

| Gesture | Action |
|---------|--------|
| Single tap | Select and read element |
| Swipe right | Move to next element |
| Swipe left | Move to previous element |
| Double tap | Activate selected element |
| Three-finger swipe up | Scroll down |
| Two-finger tap | Pause/resume reading |
| Two-finger double tap | Magic Tap (primary action) |
| Two-finger scrub (Z) | Escape/back |
| Swipe up/down | Adjust value or cycle rotor |

### Testing Flow: Login Screen

```
Step 1: Swipe right through all elements. Verify:
  [ ] Email field: reads "Email, text field" (not "TextField")
  [ ] Password field: reads "Password, secure text field"
  [ ] Login button: reads "Log in, button"
  [ ] Forgot password link: reads "Forgot password, link"
  [ ] No unlabeled elements encountered

Step 2: Interact with form fields:
  [ ] Double-tap email field: keyboard appears, VoiceOver announces "editing"
  [ ] Type email: characters announced correctly
  [ ] Double-tap password field: characters NOT announced (secure field)

Step 3: Trigger validation error:
  [ ] Submit with empty fields
  [ ] Error message announced automatically (via .announcement notification)
  [ ] Error message associates with correct field

Step 4: Navigate with rotor:
  [ ] Rotate rotor to "Headings": can jump between sections
  [ ] Rotate rotor to "Form Controls": can jump between inputs
```

### Testing Flow: List Screen

```
Step 1: Swipe through list items:
  [ ] Each item reads complete, meaningful description
  [ ] Items don't read individual sub-elements separately (should be combined)
  [ ] List position announced ("1 of 15")

Step 2: Test custom actions:
  [ ] Swipe up/down on list item reveals actions ("Delete", "Edit", etc.)
  [ ] Each action is clearly labeled
  [ ] Actions perform correctly when activated

Step 3: Test pull-to-refresh:
  [ ] Three-finger swipe down triggers refresh
  [ ] VoiceOver announces "Refreshing" / loading state
  [ ] Completion announced when refresh finishes
```

### Testing Flow: Detail Screen

```
Step 1: Verify reading order:
  [ ] Elements read in logical order (top-to-bottom, grouped logically)
  [ ] Headers marked as headers (announces "heading")
  [ ] Decorative elements hidden from VoiceOver

Step 2: Verify data presentation:
  [ ] Currency amounts read correctly ("500 dollars" not "dollar sign 5 0 0")
  [ ] Dates read naturally ("March 16, 2026" not "0 3 slash 1 6 slash 2 0 2 6")
  [ ] Status reads as text ("Completed" not just a green dot)

Step 3: Verify actions:
  [ ] All buttons labeled with action ("Send money" not "Button 3")
  [ ] Destructive actions have hints ("Double tap to cancel this transfer")
  [ ] Confirmation dialogs are accessible
```

### Testing Flow: Form Screen

```
Step 1: Tab through all form elements:
  [ ] Every field has a label (not just placeholder text)
  [ ] Required fields indicated ("Amount, required, text field")
  [ ] Field types correct (number pad for amounts, email keyboard for email)

Step 2: Test validation:
  [ ] Inline errors announced when they appear
  [ ] Error text associated with correct field
  [ ] Focus moves to first error on form submission failure

Step 3: Test pickers and custom controls:
  [ ] Pickers announce current value and options
  [ ] Custom steppers adjustable with swipe up/down
  [ ] Segmented controls announce selected state
```

---

## XCTest Accessibility Assertions

### Basic Label Verification

```swift
func testAllButtonsHaveLabels() throws {
    let app = XCUIApplication()
    app.launch()

    let buttons = app.buttons.allElementsBoundByIndex
    for button in buttons {
        XCTAssertFalse(
            button.label.isEmpty,
            "Button at index \(button.frame) has no accessibility label"
        )
    }
}
```

### Verifying Specific Elements

```swift
func testTransferRowAccessibility() throws {
    let app = XCUIApplication()
    app.launch()

    // Navigate to transfers list
    app.tabBars.buttons["Transfers"].tap()

    // Verify first transfer row
    let firstRow = app.cells.firstMatch
    XCTAssertTrue(firstRow.exists)

    // Label should contain recipient and amount
    let label = firstRow.label
    XCTAssertTrue(label.contains("Jane Smith"), "Row label should contain recipient name")
    XCTAssertTrue(label.contains("500"), "Row label should contain amount")
}
```

### Touch Target Size Verification

```swift
func testTouchTargetSizes() throws {
    let app = XCUIApplication()
    app.launch()

    let buttons = app.buttons.allElementsBoundByIndex
    for button in buttons where button.isHittable {
        let frame = button.frame
        XCTAssertGreaterThanOrEqual(
            frame.width, 44,
            "Button '\(button.label)' width \(frame.width) is below 44pt minimum"
        )
        XCTAssertGreaterThanOrEqual(
            frame.height, 44,
            "Button '\(button.label)' height \(frame.height) is below 44pt minimum"
        )
    }
}
```

### Verifying Traits

```swift
func testHeaderTraits() throws {
    let app = XCUIApplication()
    app.launch()

    // Verify section headers have header trait
    let headers = app.staticTexts.matching(
        NSPredicate(format: "label CONTAINS 'Section'")
    )

    for i in 0..<headers.count {
        let header = headers.element(boundBy: i)
        // XCUIElement doesn't expose traits directly, but we can check
        // the element exists in the headers query using identifier
        XCTAssertTrue(header.exists, "Header should exist and be accessible")
    }
}
```

### Dynamic Type Testing

```swift
func testDynamicTypeScaling() throws {
    let app = XCUIApplication()

    // Test with largest accessibility size
    app.launchArguments += ["-UIPreferredContentSizeCategoryName", "UICTContentSizeCategoryAccessibilityXXXL"]
    app.launch()

    // Verify text is still visible and not truncated
    let amountLabel = app.staticTexts["amount_label"]
    XCTAssertTrue(amountLabel.exists, "Amount label should be visible at largest text size")
    XCTAssertTrue(amountLabel.isHittable, "Amount label should not be hidden by overflow")

    // Verify layout adapts (e.g., horizontal becomes vertical)
    let sendButton = app.buttons["send_button"]
    XCTAssertTrue(sendButton.exists && sendButton.isHittable,
                  "Send button must remain accessible at largest text size")
}
```

---

## Automated Accessibility Snapshot Testing

Use accessibility snapshot tests to catch regressions in accessibility labels, traits, and values:

```swift
// Using SnapshotTesting or similar library
func testTransferDetailAccessibility() throws {
    let view = TransferDetailView(transfer: .mock)
    let vc = UIHostingController(rootView: view)

    // Assert accessibility tree matches expectation
    assertSnapshot(matching: vc, as: .accessibilityTree)
}
```

---

## WCAG 2.1 AA Checklist Mapped to iOS

### Perceivable

| Criterion | iOS Implementation | How to Verify |
|-----------|-------------------|---------------|
| 1.1.1 Non-text Content | `.accessibilityLabel()` on images and icons | VoiceOver reads meaningful description |
| 1.3.1 Info and Relationships | `.accessibilityAddTraits(.isHeader)`, form labels | Rotor navigation works for headings |
| 1.3.4 Orientation | Support both portrait and landscape | Rotate device, verify layout |
| 1.4.1 Use of Color | Icons + text alongside color | View in grayscale mode |
| 1.4.3 Contrast (Minimum) | 4.5:1 ratio | Accessibility Inspector contrast check |
| 1.4.4 Resize Text | Dynamic Type support | Set to largest accessibility size |
| 1.4.10 Reflow | Content reflows without horizontal scroll | Test at largest text size |
| 1.4.11 Non-text Contrast | 3:1 for UI components | Accessibility Inspector |
| 1.4.12 Text Spacing | Text handles increased spacing | Not typically an iOS issue (system handles) |

### Operable

| Criterion | iOS Implementation | How to Verify |
|-----------|-------------------|---------------|
| 2.1.1 Keyboard | All functions via VoiceOver/Switch Control | Navigate entire app with VoiceOver only |
| 2.4.3 Focus Order | `.accessibilitySortPriority()` | Swipe through elements, verify logical order |
| 2.4.6 Headings and Labels | `.accessibilityAddTraits(.isHeader)` | Rotor "Headings" finds all sections |
| 2.4.7 Focus Visible | VoiceOver cursor visible | VoiceOver highlights focused element |
| 2.5.1 Pointer Gestures | Single-tap alternatives for multi-touch | Test with Switch Control |
| 2.5.5 Target Size | 44x44pt minimum | Measure in Accessibility Inspector |

### Understandable

| Criterion | iOS Implementation | How to Verify |
|-----------|-------------------|---------------|
| 3.1.1 Language of Page | Set `accessibilityLanguage` if multilingual | VoiceOver uses correct pronunciation |
| 3.2.1 On Focus | No unexpected changes on focus | VoiceOver focus doesn't trigger actions |
| 3.3.1 Error Identification | Error announcements + visual indicators | Submit invalid form, verify error handling |
| 3.3.2 Labels or Instructions | `.accessibilityLabel()` on all form fields | VoiceOver reads field purpose |

### Robust

| Criterion | iOS Implementation | How to Verify |
|-----------|-------------------|---------------|
| 4.1.2 Name, Role, Value | Labels + traits + values on all elements | Accessibility Inspector shows correct info |
| 4.1.3 Status Messages | `UIAccessibility.post(notification:)` | Dynamic content changes announced |

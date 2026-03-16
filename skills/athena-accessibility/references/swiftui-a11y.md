# SwiftUI Accessibility Modifiers Reference

Complete reference for every SwiftUI accessibility modifier with usage examples.

---

## Labels and Descriptions

### .accessibilityLabel(_:)

What VoiceOver reads to identify the element. Should be concise and describe WHAT the element is, not what it does.

```swift
// Icon-only button needs a label
Button(action: sendMoney) {
    Image(systemName: "paperplane.fill")
}
.accessibilityLabel("Send money")

// Override default label when it's not meaningful
Image("company_logo")
    .accessibilityLabel("Acme Financial Services")

// Dynamic label based on state
Toggle(isOn: $notifications) {
    Image(systemName: notifications ? "bell.fill" : "bell.slash")
}
.accessibilityLabel(notifications ? "Notifications enabled" : "Notifications disabled")
```

### .accessibilityHint(_:)

Describes what happens when the element is activated. Read after a pause. Don't repeat the label.

```swift
Button("Transfer") { }
    .accessibilityLabel("Transfer funds")
    .accessibilityHint("Opens the transfer form to send money to a recipient")
    // VoiceOver: "Transfer funds, button. Opens the transfer form to send money to a recipient."

// Don't use hints for obvious actions
Button("Cancel") { }
    .accessibilityLabel("Cancel")
    // No hint needed -- "Cancel" is self-explanatory
```

### .accessibilityValue(_:)

The current value of a stateful element. Used for sliders, steppers, progress indicators, and custom controls.

```swift
// Slider with meaningful value
Slider(value: $progress, in: 0...100)
    .accessibilityValue("\(Int(progress)) percent complete")

// Custom star rating
StarRating(rating: rating)
    .accessibilityLabel("Rating")
    .accessibilityValue("\(rating) out of 5 stars")

// Toggle with value
Toggle("Auto-save", isOn: $autoSave)
    .accessibilityValue(autoSave ? "Enabled" : "Disabled")
```

---

## Traits

### .accessibilityAddTraits(_:)

Tells VoiceOver what kind of element this is. Affects announcement and interaction.

```swift
// Header -- VoiceOver announces "heading", rotor can jump to it
Text("Account Settings")
    .font(.title)
    .accessibilityAddTraits(.isHeader)

// Image -- VoiceOver announces "image"
CustomIconView(name: "transfer")
    .accessibilityAddTraits(.isImage)
    .accessibilityLabel("Transfer icon")

// Link -- VoiceOver announces "link", different activation behavior
Text("Privacy Policy")
    .onTapGesture { openURL(privacyURL) }
    .accessibilityAddTraits(.isLink)

// Selected state -- VoiceOver announces "selected"
TabButton(title: "Home", isSelected: selectedTab == .home)
    .accessibilityAddTraits(selectedTab == .home ? .isSelected : [])

// Summary element -- read first when screen appears
Text("Account balance: $5,432.10")
    .accessibilityAddTraits(.isSummaryElement)

// Updates frequently -- VoiceOver can poll for changes
StockTickerView(symbol: "AAPL")
    .accessibilityAddTraits(.updatesFrequently)

// Starts media session -- VoiceOver won't interfere with audio
Button("Play message") { playAudio() }
    .accessibilityAddTraits(.startsMediaSession)
```

### .accessibilityRemoveTraits(_:)

Remove incorrect default traits.

```swift
// SwiftUI Button that's actually used as a non-interactive display element
Button(action: {}) {
    StatusBadge(status: .active)
}
.accessibilityRemoveTraits(.isButton)
.disabled(true)

// NavigationLink used for display, not navigation
NavigationLink(destination: EmptyView()) {
    InfoRow(title: "Version", value: "2.1.0")
}
.disabled(true)
.accessibilityRemoveTraits(.isButton)
```

---

## Actions

### .accessibilityAction(_:)

Default action (what double-tap does) or named custom actions (swipe up/down to cycle).

```swift
// Override default action
CustomButton(label: "Confirm")
    .accessibilityAction {
        hapticFeedback()
        confirmTransfer()
    }

// Named actions (appear when user swipes up/down on element)
TransferRow(transfer: transfer)
    .accessibilityAction(named: "View receipt") {
        showReceipt(transfer)
    }
    .accessibilityAction(named: "Request refund") {
        requestRefund(transfer)
    }
    .accessibilityAction(named: "Share details") {
        shareTransfer(transfer)
    }
```

### .accessibilityAction(.magicTap)

Two-finger double-tap. Reserved for the most important action on screen.

```swift
// Video call screen -- magic tap toggles mute
VideoCallView()
    .accessibilityAction(.magicTap) {
        toggleMute()
    }
```

### .accessibilityAction(.escape)

Two-finger Z-scrub gesture. For dismissing modals, going back, or canceling.

```swift
ModalSheet()
    .accessibilityAction(.escape) {
        dismissSheet()
    }
```

### .accessibilityAdjustableAction

For elements that can be incremented/decremented (swipe up = increment, swipe down = decrement).

```swift
QuantityPicker(value: $quantity)
    .accessibilityLabel("Quantity")
    .accessibilityValue("\(quantity)")
    .accessibilityAdjustableAction { direction in
        switch direction {
        case .increment:
            quantity = min(quantity + 1, 99)
        case .decrement:
            quantity = max(quantity - 1, 1)
        @unknown default:
            break
        }
    }
```

---

## Element Organization

### .accessibilityElement(children:)

Controls how child elements are exposed to VoiceOver.

```swift
// .combine -- Merge children into single focus point
// Use for: cards, rows, grouped information
HStack {
    Text("Jane Smith")
    Spacer()
    Text("$500.00")
}
.accessibilityElement(children: .combine)
// VoiceOver: "Jane Smith, $500.00"

// .contain -- Keep children focusable but mark as group
// Use for: sections, form groups
VStack {
    Text("Personal Info").accessibilityAddTraits(.isHeader)
    TextField("Name", text: $name)
    TextField("Email", text: $email)
}
.accessibilityElement(children: .contain)

// .ignore -- Hide all children, provide custom accessibility
// Use for: charts, custom drawings, complex visuals
PieChartView(data: expenses)
    .accessibilityElement(children: .ignore)
    .accessibilityLabel("Expense breakdown")
    .accessibilityValue("Housing 40%, Food 25%, Transport 15%, Other 20%")
```

### .accessibilityHidden(_:)

Hide elements from VoiceOver entirely. Use for decorative elements only.

```swift
// Decorative divider
Divider()
    .accessibilityHidden(true)

// Background pattern
PatternOverlay()
    .accessibilityHidden(true)

// Redundant icon (text already describes the element)
HStack {
    Image(systemName: "envelope")
        .accessibilityHidden(true)  // "Email" text is sufficient
    Text("Email")
}

// NEVER hide informative content
Image("warning_icon")
    .accessibilityHidden(true)  // WRONG if this icon conveys meaning
```

### .accessibilityRepresentation

Provide a simpler accessible representation for complex custom controls.

```swift
// Complex circular progress control
CircularProgress(value: progress, total: 100)
    .accessibilityRepresentation {
        ProgressView(value: progress, total: 100) {
            Text("Upload progress")
        }
    }

// Custom date picker wheel
CustomDateWheel(selection: $date)
    .accessibilityRepresentation {
        DatePicker("Select date", selection: $date)
    }
```

---

## Input and Voice Control

### .accessibilityInputLabels(_:)

Alternative labels for Voice Control. Users say these to activate the element.

```swift
Button("Send $500.00 to Jane Smith") { }
    .accessibilityInputLabels([
        "Send 500 to Jane",        // Most specific, tried first
        "Send to Jane Smith",      // Medium specificity
        "Send money",              // Least specific
        "Send"                     // Shortest
    ])

// Menu items
Button("Account Settings & Preferences") { }
    .accessibilityInputLabels([
        "Account Settings",
        "Settings",
        "Preferences"
    ])
```

---

## Focus Management

### .accessibilityFocused(_:equals:)

Programmatically move VoiceOver focus.

```swift
@AccessibilityFocusState var focus: Field?

enum Field: Hashable {
    case error, firstInput, confirmation
}

var body: some View {
    VStack {
        if let error = errorMessage {
            Text(error)
                .foregroundColor(.red)
                .accessibilityFocused($focus, equals: .error)
        }

        TextField("Amount", text: $amount)
            .accessibilityFocused($focus, equals: .firstInput)

        Text("Transfer confirmed!")
            .accessibilityFocused($focus, equals: .confirmation)
    }
    .onChange(of: errorMessage) { _, newValue in
        if newValue != nil {
            focus = .error  // Move VoiceOver to error on appearance
        }
    }
}
```

### .accessibilitySortPriority(_:)

Control reading order. Higher numbers are read first.

```swift
// Visual layout: [Icon] [Info] [Action]
// Desired reading order: Info -> Action -> Icon
HStack {
    StatusIcon()
        .accessibilitySortPriority(0)  // Read last

    VStack {
        Text("Jane Smith")
        Text("$500.00")
    }
    .accessibilitySortPriority(2)  // Read first

    Button("Details") { }
        .accessibilitySortPriority(1)  // Read second
}
```

---

## Common Patterns

### Accessible Card

```swift
struct TransferCard: View {
    let transfer: Transfer

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            HStack {
                AsyncImage(url: transfer.recipientAvatarURL)
                    .accessibilityHidden(true)  // Name text is sufficient

                VStack(alignment: .leading) {
                    Text(transfer.recipientName).font(.headline)
                    Text(transfer.date.formatted(date: .abbreviated, time: .omitted))
                        .font(.caption).foregroundColor(.secondary)
                }
            }

            HStack {
                Text(transfer.amount.formatted(currency: .usd))
                    .font(.title2)
                Spacer()
                StatusBadge(status: transfer.status)
            }
        }
        .accessibilityElement(children: .combine)
        .accessibilityAction(named: "View details") { showDetail(transfer) }
        .accessibilityAction(named: "Repeat transfer") { repeatTransfer(transfer) }
    }
}
```

### Accessible Form

```swift
struct TransferForm: View {
    @State private var amount = ""
    @State private var recipient = ""
    @State private var amountError: String?
    @AccessibilityFocusState private var focusedField: FormField?

    enum FormField: Hashable {
        case amount, recipient, amountError
    }

    var body: some View {
        Form {
            Section {
                TextField("Amount", text: $amount)
                    .keyboardType(.decimalPad)
                    .accessibilityLabel("Transfer amount")
                    .accessibilityFocused($focusedField, equals: .amount)

                if let error = amountError {
                    Text(error)
                        .foregroundColor(.red)
                        .font(.caption)
                        .accessibilityFocused($focusedField, equals: .amountError)
                }

                TextField("Recipient email", text: $recipient)
                    .keyboardType(.emailAddress)
                    .textContentType(.emailAddress)
                    .accessibilityLabel("Recipient email address")
                    .accessibilityFocused($focusedField, equals: .recipient)
            } header: {
                Text("Transfer Details")
                    .accessibilityAddTraits(.isHeader)
            }
        }
        .onChange(of: amountError) { _, newError in
            if newError != nil {
                focusedField = .amountError
            }
        }
    }
}
```

### Accessible Navigation

```swift
struct MainTabView: View {
    @State private var selectedTab = Tab.home

    var body: some View {
        TabView(selection: $selectedTab) {
            HomeView()
                .tabItem {
                    Label("Home", systemImage: "house")
                }
                .tag(Tab.home)
                // VoiceOver: "Home, tab, 1 of 4"

            TransfersView()
                .tabItem {
                    Label {
                        Text("Transfers")
                    } icon: {
                        Image(systemName: "arrow.left.arrow.right")
                    }
                }
                .tag(Tab.transfers)
                .badge(pendingCount)
                // VoiceOver: "Transfers, 3 items, tab, 2 of 4"
        }
    }
}
```

### Accessible Chart

```swift
struct SpendingChart: View {
    let data: [SpendingCategory]

    var body: some View {
        Chart(data) { category in
            SectorMark(angle: .value("Amount", category.amount))
                .foregroundStyle(by: .value("Category", category.name))
        }
        .accessibilityElement(children: .ignore)
        .accessibilityLabel("Spending breakdown chart")
        .accessibilityValue(chartSummary)
    }

    private var chartSummary: String {
        let sorted = data.sorted { $0.amount > $1.amount }
        let descriptions = sorted.map { "\($0.name): \($0.amount.formatted(.currency(code: "USD")))" }
        return descriptions.joined(separator: ". ")
        // "Housing: $2,000. Food: $800. Transport: $400."
    }
}
```

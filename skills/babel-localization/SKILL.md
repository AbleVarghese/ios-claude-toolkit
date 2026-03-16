---
name: babel-localization
description: "Translate, Spanish, German, multiple languages, format currency for different countries. iOS string localization, currency formatting, date formatting, and pluralization rules. Trigger on: 'add French support', 'localize this screen', 'format money for Europe', 'date looks wrong in Japan', 'RTL layout', 'Arabic support', i18n, l10n, String Catalog, Localizable.strings, stringsdict, or any work making the app usable in multiple languages or regions. For localizing accessibility labels specifically, also see athena-accessibility."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Edit
user-invocable: false
---

# Localization & Internationalization

> **Iron Law:** "Every user-visible string must come from a localization file. Hardcoded strings are bugs."

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). Identify supported locales and currencies from the project's Localizable.strings files, .xcstrings files, lproj directories, and domain configuration.

Handle all localization for iOS apps: manage String Catalogs (.xcstrings) or legacy Localizable.strings, format currencies for multiple regions, format dates with timezone awareness, implement pluralization rules, support RTL languages, and enable dynamic locale switching.

## Supported Locales

Structure your locale support based on your app's target markets. Example layout:

| Locale | Currency | Status |
|--------|----------|--------|
| en-US | USD | Primary |
| en-GB | GBP | Example |
| de-DE | EUR | Example |
| fr-FR | EUR | Example |
| es-MX | MXN | Example |

Adapt this table to your project's actual supported locales and currencies.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|----------------|-----------------|
| "We only support English" | 37% of iOS users use non-English locales. Even English-only apps need locale-aware formatting for dates, numbers, and currencies. A user in the UK sees `$1,234.56` and doesn't know if it's USD or their local currency. | Use `Locale.current` for all formatting. Externalize strings even for English-only so the infrastructure exists when you expand. |
| "I'll localize later" | Retrofitting localization is 5-10x more expensive than building it in. String concatenation, hardcoded formats, and layout assumptions compound into weeks of rework. | Use String Catalogs from day one. Every string goes through localization infrastructure. No exceptions. |
| "NSLocalizedString is enough" | NSLocalizedString requires manual key management, has no compile-time verification, and doesn't support String Catalog features like automatic extraction, state tracking, or plural/device variations. | Use String Catalogs (.xcstrings) for new projects. Use `String(localized:)` in Swift code. |
| "RTL is an edge case" | Arabic and Hebrew represent 400M+ native speakers. RTL isn't rare, it's one of the two fundamental layout directions. And your layout will break in ways you never tested. | Use `.leading`/`.trailing` instead of `.left`/`.right`. Test with RTL pseudo-localization. Never assume text flows left-to-right. |
| "Format strings are fine for building sentences" | Grammar differs across languages. English says "5 items in 3 folders" but German rearranges word order. Concatenating localized fragments produces gibberish. | Use complete sentences with format specifiers: `"items_in_folders" = "%1$d items in %2$d folders"`. Let translators reorder arguments. |
| "Date formatting with fixed format strings works" | `"MM/dd/yyyy"` is incomprehensible in most of the world. Even `DateFormatter` with a custom format ignores the user's locale preferences. | Use `Date.FormatStyle` or `DateFormatter` with `.dateStyle`/`.timeStyle`. Never hardcode date format patterns for user-visible dates. |
| "I'll use Google Translate for other languages" | Machine translation misses context, tone, legal implications, and cultural norms. "Wire $500" machine-translated could imply physical wire, not money transfer. | Use professional translators. Budget for it. Machine translation is acceptable only for initial pseudo-localization testing. |

---

## Red Flags -- STOP

- **Hardcoded strings in SwiftUI views**: `Text("Submit")` instead of `Text("submit_button", tableName: "Actions")` or `Text(String(localized: "submit_button"))` -- this is a localization bug, not a shortcut.
- **String concatenation for localized text**: `greeting + " " + userName` -- languages have different word orders. Use format strings: `String(localized: "greeting_format \(userName)")`.
- **Missing pluralization rules**: Using `"\(count) items"` instead of stringsdict/String Catalog plural rules. Languages like Arabic have 6 plural forms (zero, one, two, few, many, other).
- **Hardcoded date/number formats**: `"MM/dd/yyyy"` or manual decimal separators. Always use `FormatStyle` or locale-aware formatters.
- **`.frame` with fixed widths on text**: German text is 30-40% longer than English. Fixed-width frames cause truncation in other locales.
- **Using `.left`/`.right` instead of `.leading`/`.trailing`**: Breaks RTL layouts entirely.

---

## When NOT to Use This Skill

- **Internal developer tools or debug screens**: Strings only developers see don't need localization infrastructure. But if end users might ever see it, localize it.
- **Log messages and analytics event names**: These are machine-consumed, not user-visible. Use English constants. (But user-visible error messages derived from these DO need localization.)
- **Unit test assertions**: Test strings are developer-facing. Use raw strings in tests.
- **Server-side string formatting**: If the server returns localized strings, localization belongs in the server codebase, not the iOS app. The app should pass the user's locale to the server.

---

## Decision Framework

```
Is this string visible to the user?
  NO  --> Raw string is fine (logs, analytics, tests)
  YES --> Does the string contain dynamic values?
           NO  --> Add to String Catalog with static key
           YES --> Does it involve counts/quantities?
                    YES --> Use plural rules (stringsdict or String Catalog variations)
                    NO  --> Use String Interpolation in String Catalog
                            or format specifiers (%@, %d, %1$@)

Is this a date, number, or currency?
  YES --> Use FormatStyle / locale-aware Formatter
          Never hardcode format patterns for display

Does the app support RTL languages?
  YES --> Audit all .left/.right usage --> replace with .leading/.trailing
          Test with RTL pseudo-localization
  NOT YET --> Still use .leading/.trailing now to be future-proof
```

---

## String Catalogs (.xcstrings) -- Modern Approach

String Catalogs are the modern replacement for Localizable.strings, introduced in Xcode 15. They provide automatic string extraction, state tracking, plural/device variations, and a visual editor.

### Setting Up String Catalogs

1. In Xcode: File > New > File > String Catalog
2. Name it `Localizable.xcstrings` (default table) or a custom name
3. Xcode automatically extracts strings from `Text()`, `String(localized:)`, and `LocalizedStringResource`
4. Manage translations, plurals, and device variations in the visual editor

### Using String Catalogs in Code

```swift
// Automatic extraction -- Xcode finds these and adds to the catalog
Text("Welcome to the app")  // Key: "Welcome to the app"

// Explicit key with default value
Text("welcome_title")  // Key: "welcome_title", provide translation in catalog

// String interpolation (extracted with placeholders)
Text("Hello, \(userName)")  // Key: "Hello, %@"

// Using String(localized:) for non-View contexts
let message = String(localized: "transfer_complete_message")

// With explicit table
let label = String(localized: "amount_label", table: "Transfers")
```

### Plural Variations in String Catalogs

In the String Catalog editor, select a key and add a "Plural" variation. Xcode provides fields for: zero, one, two, few, many, other. Each language only needs the categories that apply to its grammar.

```swift
// In code -- just use interpolation with an integer
Text("You have \(itemCount) items")
// The String Catalog handles the plural rules per locale
```

---

## Legacy Localizable.strings (Existing Projects)

### Adding Localized Strings

```swift
// In Localizable.strings (en.lproj)
"item.amount.label" = "Amount";
"item.recipient.label" = "Recipient";
"item.fee.format" = "Fee: %@";
"item.rate.format" = "Rate: 1 %1$@ = %2$@ %3$@";

// Usage in code
Text(NSLocalizedString("item.amount.label", comment: "Label for amount field"))
Text(String(format: NSLocalizedString("item.fee.format", comment: "Fee display"), formattedFee))
```

### Pluralization with stringsdict

```xml
<!-- Localizable.stringsdict -->
<key>item.count</key>
<dict>
    <key>NSStringLocalizedFormatKey</key>
    <string>%#@items@</string>
    <key>items</key>
    <dict>
        <key>NSStringFormatSpecTypeKey</key>
        <string>NSStringPluralRuleType</string>
        <key>NSStringFormatValueTypeKey</key>
        <string>d</string>
        <key>zero</key>
        <string>No items</string>
        <key>one</key>
        <string>%d item</string>
        <key>other</key>
        <string>%d items</string>
    </dict>
</dict>
```

```swift
// Usage
Text(String.localizedStringWithFormat(
    NSLocalizedString("item.count", comment: "Number of items"),
    itemCount
))
```

---

## RTL Language Support

RTL support is not optional if you target Arabic, Hebrew, Persian, or Urdu markets.

### Layout Mirroring

```swift
// WRONG: Hardcoded directional values
HStack {
    icon.padding(.left, 16)
    text.padding(.right, 8)
}

// CORRECT: Semantic directional values (auto-mirror in RTL)
HStack {
    icon.padding(.leading, 16)
    text.padding(.trailing, 8)
}
```

### Text Alignment

```swift
// WRONG: Forces left alignment regardless of locale
Text("Amount").multilineTextAlignment(.leading)
// This is actually correct -- .leading auto-adapts to RTL

// WRONG: Hardcoded left alignment
Text("Amount").frame(maxWidth: .infinity, alignment: .leading)
// This is correct -- .leading adapts

// ACTUALLY WRONG: Using explicit .left
someView.frame(alignment: .init(horizontal: .leading, vertical: .center))
// Use .leading, never construct with explicit left/right
```

### Image and Icon Mirroring

```swift
// Icons that should mirror in RTL (arrows, disclosure indicators)
Image(systemName: "chevron.right")
    .flipsForRightToLeftLayoutDirection(true)

// Icons that should NOT mirror (play button, checkmark)
Image(systemName: "checkmark")
    // No flipping needed
```

### Testing RTL

In the scheme editor: Edit Scheme > Run > Options > Application Language > "Right-to-Left Pseudolanguage". This forces RTL layout without needing an actual Arabic translation.

---

## Currency Formatting

```swift
// Modern approach: FormatStyle (iOS 15+)
let amount: Decimal = 1234.56

// Format with explicit currency code
amount.formatted(.currency(code: "USD"))          // "$1,234.56" (in en_US)
amount.formatted(.currency(code: "EUR"))          // "EUR1,234.56" (in en_US)
                                                   // "1.234,56 EUR" (in de_DE)

// Format respecting user's locale
amount.formatted(.currency(code: "USD").locale(Locale(identifier: "de_DE")))
// "1.234,56 $"

// Reusable currency formatter
extension Decimal {
    func formatted(currencyCode: String, locale: Locale = .current) -> String {
        self.formatted(
            .currency(code: currencyCode)
            .locale(locale)
        )
    }
}
```

### Money Formatting Guidelines

```
USD (en-US):
  Amount:   $1,234.56
  Negative: -$1,234.56
  Cents:    $0.99

EUR (de-DE):
  Amount:   1.234,56 EUR
  Negative: -1.234,56 EUR
  French:   1 234,56 EUR

GBP (en-GB):
  Amount:   GBP1,234.56
  Negative: -GBP1,234.56
```

Adapt the formatting examples above to your project's supported currencies and locales.

### Currency Enum Pattern

```swift
enum Currency: String, Codable {
    case usd, eur, gbp, cad, mxn

    var code: String { rawValue.uppercased() }

    var defaultLocale: Locale {
        switch self {
        case .usd: return Locale(identifier: "en_US")
        case .eur: return Locale(identifier: "de_DE")
        case .gbp: return Locale(identifier: "en_GB")
        case .cad: return Locale(identifier: "en_CA")
        case .mxn: return Locale(identifier: "es_MX")
        }
    }
}

extension Decimal {
    func formatted(currency: Currency) -> String {
        formatted(currencyCode: currency.code, locale: currency.defaultLocale)
    }
}
```

---

## Date Formatting

```swift
// Modern approach: FormatStyle (iOS 15+)
let date = Date()

// Relative formatting
date.formatted(.relative(presentation: .named))     // "2 hours ago"

// Standard date styles
date.formatted(date: .abbreviated, time: .shortened) // "Mar 16, 2026, 3:45 PM"
date.formatted(date: .long, time: .omitted)          // "March 16, 2026"

// Custom components (still locale-aware)
date.formatted(.dateTime.day().month(.wide).year())  // "March 16, 2026" (en)
                                                      // "16. Maerz 2026" (de)

// WRONG: Fixed format string (locale-ignorant)
let formatter = DateFormatter()
formatter.dateFormat = "MM/dd/yyyy"  // Never do this for user-visible dates
```

---

## Dynamic Locale Switching

Changing the app language at runtime (without restarting):

```swift
final class LocaleManager: ObservableObject {
    static let shared = LocaleManager()

    @AppStorage("appLanguage") var currentLanguage: String = "en" {
        didSet { updateLocale() }
    }

    @Published var locale: Locale = .current

    private func updateLocale() {
        locale = Locale(identifier: currentLanguage)
        // Force bundle to load correct lproj
        UserDefaults.standard.set([currentLanguage], forKey: "AppleLanguages")
    }
}

// In your App struct
@main
struct MyApp: App {
    @StateObject private var localeManager = LocaleManager.shared

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.locale, localeManager.locale)
        }
    }
}
```

Note: Full dynamic switching requires either SwiftUI's `.environment(\.locale)` (which affects `Text` views using String Catalogs) or a custom localization bundle loader. Some UIKit components may still require an app restart.

---

## Localization QA Checklist

Before shipping any localized build:

- [ ] All user-visible strings externalized (no hardcoded text in views)
- [ ] Pseudo-localization tested (catches truncation, layout issues, hardcoded strings) -- see `references/testing-locales.md`
- [ ] Number formatting respects locale (decimal separators, grouping)
- [ ] Date formatting respects locale (no hardcoded MM/dd/yyyy)
- [ ] Currency formatting includes correct symbol placement per locale
- [ ] Pluralization rules defined for all quantity strings (zero/one/two/few/many/other)
- [ ] RTL layout tested if supporting Arabic/Hebrew
- [ ] Text truncation checked with longest supported language (German is ~30% longer than English)
- [ ] Character encoding verified (emoji, CJK characters, diacritics)
- [ ] Screenshots captured for all supported locales (App Store requirement)
- [ ] String Catalog shows 100% translation state for all shipped locales
- [ ] No string concatenation used to build sentences

---

## Phase Gate: COMPLETENESS ✓

**Cannot mark localization work complete until:**
- [ ] Every user-visible string comes from a localization file (no hardcoded strings in views)
- [ ] Pluralization rules tested for quantities: 0, 1, 2, many
- [ ] Currency formatting tested with at least 2 different locales
- [ ] Date formatting verified across timezones

**Hard Stop**: Hardcoded strings in SwiftUI views are localization bugs. If you see `Text("Submit")` instead of `Text("button.submit".localized)` — fix it before proceeding.

### Self-Audit

1. Are ALL user-visible strings localized (not just the new ones)?
2. Did I test with a non-English locale to catch formatting issues?
3. Are format strings used correctly (no string concatenation for localized text)?
4. Are pluralization rules complete for the target locales?

---

## Quality Gates (Before Marking Complete)

1. All user-visible strings are externalized in String Catalog or Localizable.strings
2. No hardcoded date, number, or currency format patterns in user-facing code
3. All layout constraints use `.leading`/`.trailing` (not `.left`/`.right`)
4. Plural strings use stringsdict or String Catalog plural variations
5. Pseudo-localization pass shows no truncation or layout breaks
6. Format strings use positional specifiers (`%1$@`, `%2$@`) so translators can reorder
7. String Catalog translation state is 100% for all shipped locales
8. Dynamic Type still works with localized text (longer strings don't break layout at larger sizes)
9. Currency amounts display correctly for all supported currencies and locales
10. No string concatenation used to construct localized sentences

---

## Cross-Skill References

- **athena-accessibility**: Localized strings need accessibility labels too. VoiceOver reads localized text, so labels must be localized.
- **cupertino-apple**: App Store metadata (name, subtitle, keywords, description) must be localized per market. Screenshots must show localized UI.
- **forge-development**: Build configurations may need per-locale scheme settings for testing.
- **documentation**: Localization strategy decisions should be documented in ADRs.

---

## References

- Locale testing methodology: `references/testing-locales.md`
- [Apple: Localizing and varying text with a string catalog](https://developer.apple.com/documentation/xcode/localizing-and-varying-text-with-a-string-catalog)
- [Apple: Formatting data for display](https://developer.apple.com/documentation/foundation/data_formatting)
- [CLDR Plural Rules](https://cldr.unicode.org/index/cldr-spec/plural-rules)

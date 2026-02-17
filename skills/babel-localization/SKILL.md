---
name: babel-localization
description: "This skill should be used when the user asks to 'localize strings', 'add a translation', 'format currency', 'format dates', 'internationalization', or mentions locale support. Provides string localization, currency/date/number formatting, and pluralization for iOS apps with multi-locale support."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Edit
user-invocable: false
---

# Localization & Internationalization

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift). Identify supported locales and currencies from the project's Localizable.strings files, lproj directories, and domain configuration.

Handle all localization for iOS apps: manage Localizable.strings, format currencies for multiple regions, format dates with timezone awareness, and implement pluralization rules across supported locales.

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

## Standard Operating Procedures

### SOP-1: Add Localized String
```swift
// In Localizable.strings (en)
"item.amount.label" = "Amount";
"item.recipient.label" = "Recipient";
"item.fee.format" = "Fee: %@";
"item.rate.format" = "Rate: 1 %@ = %@ %@";

// Usage in code
Text("item.amount.label".localized)
Text(String(format: "item.fee.format".localized, formattedFee))
```

### SOP-2: Format Currency
```swift
extension Decimal {
    func formatted(currencyCode: String, localeIdentifier: String) -> String {
        let formatter = NumberFormatter()
        formatter.numberStyle = .currency
        formatter.currencyCode = currencyCode
        formatter.locale = Locale(identifier: localeIdentifier)
        return formatter.string(from: self as NSDecimalNumber) ?? "$0.00"
    }
}

// Example with a Currency enum pattern
enum Currency: String {
    case usd, eur, gbp, cad, mxn

    var code: String { rawValue.uppercased() }

    var defaultLocale: String {
        switch self {
        case .usd: return "en_US"
        case .eur: return "de_DE"
        case .gbp: return "en_GB"
        case .cad: return "en_CA"
        case .mxn: return "es_MX"
        }
    }
}

extension Decimal {
    func formatted(currency: Currency) -> String {
        formatted(currencyCode: currency.code, localeIdentifier: currency.defaultLocale)
    }
}

// Usage
let amount: Decimal = 100.50
Text(amount.formatted(currency: .usd)) // "$100.50"
Text(amount.formatted(currency: .eur)) // "100,50 EUR"
```

### SOP-3: Format Dates
```swift
extension Date {
    var relativeFormatted: String {
        let formatter = RelativeDateTimeFormatter()
        formatter.unitsStyle = .full
        return formatter.localizedString(for: self, relativeTo: Date())
    }

    var transactionFormatted: String {
        let formatter = DateFormatter()
        formatter.dateStyle = .medium
        formatter.timeStyle = .short
        formatter.doesRelativeDateFormatting = true
        return formatter.string(from: self)
    }
}

// Usage
Text(item.createdAt.relativeFormatted)      // "2 hours ago"
Text(item.createdAt.transactionFormatted)    // "Today at 3:45 PM"
```

### SOP-4: Pluralization
```swift
// In Localizable.stringsdict
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
        <key>one</key>
        <string>%d item</string>
        <key>other</key>
        <string>%d items</string>
    </dict>
</dict>

// Usage
Text(String.localizedStringWithFormat(
    NSLocalizedString("item.count", comment: ""),
    itemCount
)) // "1 item" or "5 items"
```

## Money Formatting Guidelines

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

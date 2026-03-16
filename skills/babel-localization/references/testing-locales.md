# Locale Testing Reference

Comprehensive testing methodology for localized iOS apps.

---

## Pseudo-Localization

Pseudo-localization replaces English strings with accented versions that are longer and use non-ASCII characters, without needing actual translations. It catches three categories of bugs simultaneously:

### What It Catches

| Bug Category | Example | How Pseudo-loc Reveals It |
|-------------|---------|--------------------------|
| Hardcoded strings | `Text("Submit")` not in localization file | String appears normal while all localized strings appear accented |
| Truncation | Label too narrow for longer text | Pseudo-localized text is 30-50% longer, exposing overflow |
| Character encoding | Non-ASCII characters rendered as `?` or boxes | Accented characters (e, a, o) reveal encoding issues |
| Layout assumptions | Fixed-width frames, hardcoded padding | Longer text breaks rigid layouts |
| Concatenation bugs | `"Hello " + name` bypasses localization | The concatenated fragment shows in plain English |

### Enabling Pseudo-Localization in Xcode

1. Edit Scheme > Run > Options
2. Application Language: select one of:
   - **"Accented Pseudolanguage"** -- replaces ASCII with accented equivalents, adds length
   - **"Bounded String Pseudolanguage"** -- wraps strings in brackets `[Submit]` to show boundaries
   - **"Right-to-Left Pseudolanguage"** -- forces RTL layout without needing Arabic/Hebrew translations
3. Run the app and navigate every screen

### What to Look For

- **Normal English text** = hardcoded string not going through localization
- **Truncated accented text** = container too small, needs flexible layout
- **Overlapping text** = constraints don't accommodate text growth
- **Broken layout** = fixed widths or heights that assume English string length

---

## Testing Methodology Per Locale

For each supported locale, verify these categories:

### 1. String Completeness

```
For every screen in the app:
  - Switch to target locale
  - Verify no English strings appear (unless intentional, like brand names)
  - Verify no empty labels or missing text
  - Check that plurals work: test with 0, 1, 2, 5, 21 items
  - Verify format strings render correctly with real data
```

### 2. Date and Time Formatting

| Test | en_US | en_GB | de_DE | ja_JP |
|------|-------|-------|-------|-------|
| Short date | 3/16/26 | 16/03/2026 | 16.03.26 | 2026/03/16 |
| Medium date | Mar 16, 2026 | 16 Mar 2026 | 16.03.2026 | 2026/03/16 |
| Time | 3:45 PM | 15:45 | 15:45 | 15:45 |
| Relative | "2 hours ago" | "2 hours ago" | "vor 2 Stunden" | "2 jikan mae" |

Verify:
- No hardcoded date separators (/, -, .)
- 12h vs 24h time matches user's locale preference
- Relative dates use correct language
- First day of week is correct (Sunday in US, Monday in Europe)

### 3. Number and Currency Formatting

| Test | en_US | de_DE | fr_FR | ja_JP |
|------|-------|-------|-------|-------|
| Number | 1,234.56 | 1.234,56 | 1 234,56 | 1,234.56 |
| USD | $1,234.56 | 1.234,56 $ | 1 234,56 $US | $1,234.56 |
| EUR | EUR1,234.56 | 1.234,56 EUR | 1 234,56 EUR | EUR1,234.56 |
| Percentage | 75% | 75 % | 75 % | 75% |

Verify:
- Decimal separator matches locale (. vs ,)
- Thousands grouping matches locale (, vs . vs space)
- Currency symbol position matches locale (prefix vs suffix)
- Negative number formatting matches locale (-$5 vs -5 $ vs (5))

### 4. Layout and RTL

For RTL locales (Arabic, Hebrew):
- Navigation bar: back button on right, title centered/right
- Lists: content aligned to right edge
- Icons: directional icons (arrows, chevrons) mirrored
- Text input: cursor starts from right
- Progress bars: fill from right to left
- Swipe gestures: directions reversed

For all locales:
- Long strings don't truncate critical information
- Multi-line text doesn't overflow containers
- Dynamic Type still works (large accessibility sizes)

---

## Screenshot Automation for App Store

Each locale you support in the App Store needs its own set of screenshots. Automate this:

### Using XCTest + Fastlane Snapshot

```swift
// ScreenshotTests.swift
import XCTest

final class ScreenshotTests: XCTestCase {
    let app = XCUIApplication()

    override func setUp() {
        super.setUp()
        continueAfterFailure = false
        setupSnapshot(app)
        app.launch()
    }

    func testHomeScreen() {
        snapshot("01_HomeScreen")
    }

    func testTransferScreen() {
        app.buttons["new_transfer_button"].tap()
        snapshot("02_TransferScreen")
    }

    func testHistoryScreen() {
        app.tabBars.buttons["History"].tap()
        snapshot("03_HistoryScreen")
    }
}
```

### Fastlane Snapfile

```ruby
# Snapfile
devices([
  "iPhone 16 Pro Max",    # 6.7" (required)
  "iPhone SE (3rd generation)",  # 4.7" (if supporting)
  "iPad Pro 13-inch (M4)"       # iPad (if universal)
])

languages([
  "en-US",
  "de-DE",
  "fr-FR",
  "es-MX",
  "ar"       # RTL
])

scheme("YourAppScheme")
output_directory("./screenshots")
clear_previous_screenshots(true)
```

Run: `fastlane snapshot`

This generates screenshots for every device x language combination automatically.

---

## Common Locale Bugs

### 1. Date Format Assumptions

```swift
// BUG: Assumes MM/DD/YYYY ordering
let components = dateString.split(separator: "/")
let month = components[0]  // Wrong in DD/MM/YYYY locales

// FIX: Use DateFormatter for parsing, never manual string splitting
let formatter = DateFormatter()
formatter.dateStyle = .short
formatter.locale = Locale.current
let date = formatter.date(from: dateString)
```

### 2. Currency Symbol Position

```swift
// BUG: Assumes currency symbol is a prefix
let display = "$" + amount  // Wrong for locales where symbol is suffix

// FIX: Use locale-aware formatter
let display = amount.formatted(.currency(code: "USD"))
```

### 3. Decimal Separator in Input Fields

```swift
// BUG: Only accepts "." as decimal separator
guard let value = Double(textField.text ?? "") else { return }
// Fails for "1.234,56" (German input)

// FIX: Use locale-aware number parsing
let formatter = NumberFormatter()
formatter.numberStyle = .decimal
formatter.locale = Locale.current
guard let value = formatter.number(from: textField.text ?? "")?.doubleValue else { return }
```

### 4. String Length Assumptions

```
English:  "Submit"        (6 chars)
German:   "Einreichen"    (10 chars, +67%)
French:   "Soumettre"     (9 chars, +50%)
Arabic:   "ارسال"          (5 chars, but RTL)
Japanese: "送信"           (2 chars, but wider glyphs)
```

Design layouts with flexible widths. Test with German (longest common Western language) and Japanese (widest glyphs).

### 5. Pluralization Beyond One/Other

English has 2 plural forms: one, other.
Arabic has 6: zero, one, two, few (3-10), many (11-99), other (100+).
Polish has 4: one, few, many, other.

If you support these languages, your stringsdict/String Catalog must define all required plural categories. Missing categories cause the "other" form to appear incorrectly.

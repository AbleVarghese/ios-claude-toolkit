---
name: cupertino-apple
description: "Submit to App Store, App Store rejection, Apple guidelines, ready to ship. App Store submission compliance and Human Interface Guidelines. Trigger on: 'prepare for App Store review', 'will Apple reject this', 'privacy labels', 'App Store Connect', 'privacy policy', 'review guidelines', 'in-app purchase rules', 'got rejected', 'submission checklist', 'ready to launch', or any question about Apple's requirements for publishing. For Fastlane upload mechanics, use velocity-fastlane instead."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: false
---

# Apple Ecosystem & App Store Compliance

> **Iron Law:** "Review Apple's guidelines BEFORE implementation, not after rejection. Every rejection costs 1-2 weeks."

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

Ensure the iOS app achieves and maintains Apple App Store approval through strict adherence to Apple guidelines, HIG principles, privacy requirements, and iOS platform best practices. This skill covers rejection prevention, privacy compliance, in-app purchase rules, and App Store metadata optimization.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|----------------|-----------------|
| "Apple won't notice" | Apple's review team uses automated scanning, manual testing, and AI-assisted review. They notice. They have caught private API usage, undisclosed data collection, and hidden features consistently for 15+ years. | Assume Apple will find every violation. Build compliance into your development process, not your submission prayers. |
| "Other apps do this" | Survivorship bias. You see apps that got through; you don't see the thousands rejected for the same practice. Apple also retroactively enforces guidelines -- an app that slipped through can be removed later. | Follow the published guidelines. "But they did it" is not an appeal strategy. |
| "We can appeal the rejection" | Appeals take 1-2 weeks, have low success rates for legitimate violations, and burn goodwill with the review team. Multiple rejections flag your account for stricter review. | Prevent rejections, don't plan to appeal them. Read the guidelines before writing code. |
| "The guidelines are unclear so we're probably fine" | Ambiguity in guidelines is not permission. When Apple is unclear, they tend to reject first and discuss later. The App Review Board interprets ambiguity conservatively. | When guidelines are ambiguous, contact Apple Developer Support BEFORE building the feature. Get written guidance. |
| "We'll add the privacy stuff before submission" | Privacy requirements affect architecture (data collection scope, consent flows, ATT implementation). Bolting them on at the end creates bugs, missed disclosures, and a rushed implementation that reviewers can spot. | Design privacy into your architecture from day one. Privacy labels, consent flows, and data handling are features, not checkboxes. |
| "TestFlight is good enough for testing" | TestFlight and App Store review are different environments with different rules. TestFlight doesn't enforce payment rules, privacy labels, or content policies. Your TestFlight app can work perfectly and still get rejected. | Test against the actual review guidelines, not just functionality. Use the pre-submission checklist below. |

---

## Red Flags -- STOP

- **Mentioning competing platforms in the app**: Referencing Android, Windows, or web versions in the app UI or metadata. Apple rejects for "irrelevant platform references." Put cross-platform info on your website, not in the app.
- **Using private APIs**: Any API not in the public SDK documentation. Apple's static analysis catches these even when obfuscated. No exceptions.
- **Missing privacy policy**: Required for ALL apps. Must be hosted at a working URL, accessible from within the app and from App Store Connect.
- **Incomplete demo account**: Reviewers must be able to test every feature. If your app requires login, provide a demo account with prepopulated data. If it requires physical verification (KYC), provide a bypass for review.
- **Digital goods without IAP**: If users purchase digital content, subscriptions, or premium features, Apple requires In-App Purchase via StoreKit. Physical goods and services (Uber rides, Amazon products) are exempt.
- **Undisclosed data collection**: If your app collects data not reflected in your privacy nutrition labels, automatic rejection. This includes third-party SDKs that collect data.

---

## When NOT to Use This Skill

- **Enterprise distribution**: Apps distributed via Apple Business Manager / MDM have different rules. This skill covers App Store public distribution only.
- **macOS-only apps**: While many guidelines overlap, macOS has distinct HIG and review considerations. This skill focuses on iOS/iPadOS.
- **Web apps / PWAs**: Web content inside WKWebView still needs to follow guidelines, but pure web apps are not reviewed by Apple.
- **Prototype or TestFlight-only builds**: If you're not submitting to the App Store, guidelines are advisory. But build compliance in early -- retrofitting costs 5x more.

---

## Decision Framework

```
What type of feature am I implementing?

PAYMENT / MONETIZATION
  |-- Is it for digital goods/content/features?
  |     YES --> Must use StoreKit (In-App Purchase)
  |             No linking to external payment pages
  |     NO  --> Is it for physical goods or services?
  |             YES --> Can use Stripe/custom payment (no IAP required)
  |                     Must still follow Guideline 3.1.5
  |
  |-- Is it a subscription?
        YES --> Must use auto-renewable subscriptions via StoreKit
                Must support grace periods and billing retry
                Must link to subscription management

CONTENT
  |-- Does the app display user-generated content?
  |     YES --> Must implement reporting/blocking (Guideline 1.2)
  |             Must have moderation system
  |             Must filter objectionable content
  |
  |-- Does the app use web views for core functionality?
        YES --> Risk of Guideline 4.2 (minimum functionality) rejection
                App must provide value beyond a website wrapper

DATA / PRIVACY
  |-- Does the app collect ANY user data?
  |     YES --> Privacy policy required (hosted URL)
  |             Privacy nutrition labels must be accurate
  |             ATT prompt required if tracking across apps/websites
  |
  |-- Does the app use third-party SDKs?
        YES --> Audit each SDK's data collection
                Include SDK data in privacy labels
                Ensure SDK privacy manifests are included

SOCIAL / COMMUNICATION
  |-- Does the app allow users to communicate?
        YES --> Must implement blocking and reporting
                Must have Terms of Service
                Must comply with Guideline 1.2 (User Generated Content)
```

---

## Pre-Submission Checklist

### App Completeness (Guideline 2.1)

```
[ ] App launches without crashes on all supported devices
[ ] All features are fully functional (no placeholder content, no "coming soon")
[ ] All links work (deep links, universal links, web URLs)
[ ] Error states handled gracefully (no raw error messages, no blank screens)
[ ] Demo account provided with pre-populated data for reviewers
[ ] App works offline or shows appropriate offline message
[ ] No beta, test, trial, or demo labels in production build
[ ] All placeholder images and lorem ipsum text removed
```

### Privacy Compliance (Guideline 5.1)

```
[ ] Privacy Policy URL configured in App Store Connect
[ ] Privacy Policy accessible from within the app
[ ] Privacy nutrition labels accurately reflect ALL data collection
[ ] All Info.plist usage descriptions present and specific
[ ] ATT prompt implemented if tracking users across apps/websites
[ ] Third-party SDK privacy manifests included (required since Spring 2024)
[ ] Data collection purposes match actual usage
[ ] Data linked to user identity properly disclosed
```

### Metadata (Guideline 2.3)

```
[ ] App name doesn't include price, rankings, or "free"
[ ] Screenshots show actual app UI (not mockups or marketing material)
[ ] App description accurately describes functionality
[ ] Keywords relevant and not duplicating app name/category
[ ] Category selection is accurate
[ ] Age rating reflects actual content
[ ] Support URL working and responsive
[ ] No references to other platforms (Android, Windows)
```

### In-App Purchase (Guideline 3.1)

```
[ ] Digital goods use StoreKit IAP (not third-party payment)
[ ] Subscription terms clearly displayed before purchase
[ ] Restore purchases button present and functional
[ ] Subscription management link provided
[ ] Free trial terms clearly communicated
[ ] No references to external purchase mechanisms for digital goods
```

---

## Privacy Requirements

### Info.plist Privacy Descriptions

Every permission your app requests must include a purpose string. Be specific about WHY you need the permission, not just WHAT it is.

```xml
<!-- GOOD: Specific, explains user benefit -->
<key>NSCameraUsageDescription</key>
<string>Camera access is needed to scan documents for identity verification.</string>

<key>NSFaceIDUsageDescription</key>
<string>Face ID is used to securely authenticate and protect your financial data.</string>

<key>NSPhotoLibraryUsageDescription</key>
<string>Photo library access lets you upload identification documents for account verification.</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>Your location helps us show nearby ATMs and comply with regional financial regulations.</string>

<!-- BAD: Vague, doesn't explain why -->
<key>NSCameraUsageDescription</key>
<string>This app needs camera access.</string>
<!-- This WILL be rejected -->
```

### Privacy Nutrition Labels

Map every piece of data your app collects:

| Data Type | Collected | Linked to Identity | Used for Tracking | Purpose |
|-----------|-----------|-------------------|-------------------|---------|
| Email | Yes | Yes | No | Account creation |
| Name | Yes | Yes | No | Account identification |
| Phone | Yes | Yes | No | 2FA, account recovery |
| Payment Info | Yes | Yes | No | Transaction processing |
| Location | Yes | No | No | Regulatory compliance |
| Usage Data | Yes | No | No | App analytics |
| Crash Data | Yes | No | No | Bug fixing |

Audit third-party SDKs too. Firebase, Stripe, analytics tools, and crash reporters all collect data that must be disclosed.

### App Tracking Transparency (ATT)

Required when tracking users across apps or websites owned by other companies.

```swift
import AppTrackingTransparency

func requestTrackingPermission() {
    ATTrackingManager.requestTrackingAuthorization { status in
        switch status {
        case .authorized:
            // Enable tracking / ad personalization
            break
        case .denied, .restricted, .notDetermined:
            // Disable tracking, use non-personalized ads
            break
        @unknown default:
            break
        }
    }
}
```

ATT is NOT required for: first-party analytics, fraud detection, or data that stays on-device.

---

## In-App Purchase Compliance

### What Requires IAP

| Content Type | Requires IAP? | Notes |
|-------------|--------------|-------|
| Premium app features | Yes | Subscriptions or one-time unlock |
| Digital content (books, music, videos) | Yes | Must use StoreKit |
| Virtual currency / tokens | Yes | Must use consumable IAP |
| Subscriptions to digital services | Yes | Auto-renewable subscriptions |
| Physical goods | No | Use any payment processor |
| Real-world services (rides, food) | No | Use any payment processor |
| Person-to-person payments | No | Guideline 3.1.5(a) exemption |
| Enterprise/B2B apps | Varies | Can use web-based purchase for B2B |

### Subscription Requirements

```swift
// Required: Subscription management link
// Show this in settings or account screen
if let url = URL(string: "https://apps.apple.com/account/subscriptions") {
    Link("Manage Subscription", destination: url)
}

// Required: Restore purchases
Button("Restore Purchases") {
    Task {
        try await AppStore.sync()
    }
}
```

---

## App Store Metadata Optimization

### App Name (30 characters max)

- Lead with your brand name
- Don't include generic terms Apple may reject ("best", "free", "#1")
- Don't include price or promotional text

### Subtitle (30 characters max)

- Describe the app's value proposition
- Don't repeat the app name

### Keywords (100 characters max)

- Comma-separated, no spaces after commas
- Don't duplicate words from app name or subtitle (they're indexed automatically)
- Include competitor names at your own risk (Apple may reject)
- Use singular forms (Apple indexes both singular and plural)

### Screenshots

| Device | Size | Required |
|--------|------|----------|
| iPhone 6.7" | 1290 x 2796 | Yes (if supporting iPhone) |
| iPhone 6.5" | 1284 x 2778 | Yes (legacy requirement) |
| iPad 12.9" | 2048 x 2732 | Yes (if universal app) |

- First screenshot is most important (shown in search results)
- Show actual app UI, not marketing renders
- Localize screenshots for each supported locale

---

## Demo Account for Review

```
Email: {DEMO_ACCOUNT_EMAIL}
Password: [Configure secure password before submission]
Notes to reviewer:
- "Use test card 4242 4242 4242 4242 for payments (if applicable)"
- "All KYC verification is pre-approved for this account"
- "Account is pre-funded with $10,000 test balance"
```

Provide this in App Store Connect > App Review Information > Sign-In Required.

The demo account MUST:
- Log in successfully
- Have data populated (not an empty state)
- Access all features without additional verification
- Not expire during the review period (keep it active for 30+ days after submission)

---

## Phase Gate: PRE-SUBMISSION ✓

**Cannot submit to App Store until:**
- [ ] All items in pre-submission checklist verified (not assumed)
- [ ] Privacy Policy URL tested (returns 200, not 404)
- [ ] Demo account credentials set up and tested
- [ ] All privacy labels accurate (matches actual data collection)
- [ ] App tested on oldest supported device/OS

**Hard Stop**: Missing privacy policy or inaccurate privacy labels = automatic rejection. Verify these FIRST — they're the easiest to check and the most common rejection reason.

### Self-Audit

1. Have I checked the app against the top 5 rejection reasons? (references/rejection-prevention.md)
2. Is the demo account actually functional for the reviewer?
3. Are privacy labels accurate (not just what we think we collect — what we actually collect)?
4. Does the app work on the oldest supported iOS version?

---

## Quality Gates (Before Marking Complete)

1. App launches and runs without crashes on all target devices
2. Privacy policy URL is live, accessible, and accurate
3. Privacy nutrition labels match actual data collection (including third-party SDKs)
4. All Info.plist usage descriptions are present, specific, and explain user benefit
5. In-app purchases use StoreKit where required
6. Restore Purchases is present and functional
7. Demo account is prepared with pre-populated data
8. Screenshots reflect current app UI for all required device sizes
9. No references to competing platforms in app UI or metadata
10. All links (support URL, marketing URL, privacy policy) return 200 status

---

## Cross-Skill References

- **athena-accessibility**: Apple reviews for accessibility. Apps failing Dynamic Type or VoiceOver may be rejected under Guideline 4.0 (Design).
- **babel-localization**: App Store metadata must be localized per market. Screenshots must show the localized UI.
- **stripe-best-practices**: Physical goods and person-to-person payments can use Stripe. Digital goods must use StoreKit IAP.
- **cipher-security**: Security practices affect privacy compliance. Data encryption, secure storage, and network security are reviewed.
- **velocity-fastlane**: Fastlane automates App Store submission, screenshot generation, and metadata management.

---

## References

- Detailed compliance tables: `references/app-store-guidelines.md`
- HIG checklist and rejection prevention: `references/hig-checklist.md`
- Top rejection reasons and prevention: `references/rejection-prevention.md`
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [App Tracking Transparency](https://developer.apple.com/documentation/apptrackingtransparency)
- [StoreKit 2](https://developer.apple.com/documentation/storekit)

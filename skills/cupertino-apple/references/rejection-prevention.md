# App Store Rejection Prevention

Top 20 rejection reasons with prevention strategies and fix templates.

---

## 1. Guideline 2.1 -- App Completeness

**What it means**: App crashes, has broken links, placeholder content, or features that don't work.

**Prevention**:
- Test every feature on physical devices (not just Simulator)
- Remove all placeholder text, lorem ipsum, and "coming soon" sections
- Verify all deep links and universal links resolve correctly
- Test offline behavior and error states
- Run the app through Instruments for memory leaks and crashes

**Fix template if rejected**:
```
Thank you for the feedback. We have:
1. Fixed the crash in [specific screen/flow]
2. Removed placeholder content from [location]
3. Verified all links are functional
4. Tested on [list devices and OS versions]
Please re-review at your convenience.
```

---

## 2. Guideline 2.3 -- Accurate Metadata

**What it means**: Screenshots, description, or app name don't match actual app functionality.

**Prevention**:
- Capture screenshots from the current build (never use old/mocked screenshots)
- Ensure description describes actual features, not planned features
- Don't include "free", "best", "#1", or ranking claims in app name
- Don't use competitor names in metadata

**Fix template if rejected**:
```
We have updated our metadata to accurately reflect the app's current functionality:
1. New screenshots captured from build [version]
2. Description updated to reflect only available features
3. Removed [specific problematic claim] from [name/subtitle/description]
```

---

## 3. Guideline 3.1.1 -- In-App Purchase

**What it means**: App sells digital goods or subscriptions without using Apple's IAP system.

**Prevention**:
- Audit every purchase flow: digital goods MUST use StoreKit
- Don't link to external websites for digital purchases
- Don't mention alternative payment methods for digital content
- Physical goods and real-world services are exempt

**Fix template if rejected**:
```
We have migrated our [feature] purchase flow to use StoreKit In-App Purchase:
1. Created [consumable/non-consumable/subscription] IAP products in App Store Connect
2. Implemented StoreKit 2 purchase flow
3. Removed references to external payment for digital content
4. Added Restore Purchases functionality
```

---

## 4. Guideline 4.0 -- Design (HIG Compliance)

**What it means**: App uses non-native UI patterns, has poor quality design, or doesn't follow platform conventions.

**Prevention**:
- Use standard iOS navigation (NavigationStack, TabView)
- Follow Apple's HIG for layout, spacing, and interaction patterns
- Support Dynamic Type and Dark Mode
- Use SF Symbols for system-style icons
- Ensure touch targets are 44x44pt minimum

**Fix template if rejected**:
```
We have updated our design to follow Apple's Human Interface Guidelines:
1. Replaced custom navigation with NavigationStack
2. Added Dynamic Type support across all text
3. Implemented Dark Mode support
4. Increased touch targets to meet 44pt minimum
```

---

## 5. Guideline 5.1.1 -- Data Collection and Storage

**What it means**: App collects user data not disclosed in privacy labels, or lacks privacy policy.

**Prevention**:
- Audit every SDK for data collection (Firebase, analytics, crash reporters)
- Include privacy manifests for all third-party SDKs
- Map every data point to a privacy nutrition label category
- Host privacy policy at a stable, accessible URL
- Include privacy policy link in the app's settings

**Fix template if rejected**:
```
We have corrected our privacy disclosures:
1. Updated privacy nutrition labels to include [specific data types]
2. Added privacy manifests for [SDK names]
3. Updated privacy policy to disclose [specific data usage]
4. Privacy policy URL verified accessible: [URL]
```

---

## 6. Guideline 5.1.2 -- Data Use and Sharing

**What it means**: App shares user data with third parties without adequate disclosure or consent.

**Prevention**:
- Disclose all third-party data sharing in privacy policy
- Implement consent mechanisms for non-essential data sharing
- Don't share data with ad networks without ATT consent
- Document data flow from app to all third-party services

---

## 7. Guideline 2.1 -- Performance (Crashes)

**What it means**: App crashes during review.

**Prevention**:
- Run full test suite before submission
- Test on the oldest supported device and OS version
- Profile with Instruments (Memory Graph, Leaks, Time Profiler)
- Test with low memory warnings enabled
- Handle all force-unwrap optionals

---

## 8. Guideline 4.2 -- Minimum Functionality

**What it means**: App is too simple, a thin web wrapper, or doesn't provide enough native value.

**Prevention**:
- Don't wrap a website in WKWebView as the primary experience
- Provide native UI for core functionality
- Add platform-specific features (widgets, notifications, Shortcuts)
- Ensure the app provides value beyond what the website offers

---

## 9. Guideline 1.2 -- User Generated Content

**What it means**: App allows users to post content but lacks moderation, reporting, or blocking.

**Prevention**:
- Implement report/flag functionality for all user-generated content
- Implement block user functionality
- Add content moderation (automated or manual)
- Include Terms of Service
- Filter objectionable content before display

---

## 10. Guideline 2.5.1 -- Software Requirements

**What it means**: App uses deprecated APIs, private APIs, or undocumented features.

**Prevention**:
- Only use public APIs from Apple's official documentation
- Update deprecated API usage before submission
- Don't use `dlopen`, `dlsym`, or other dynamic loading for private frameworks
- Run the app with `-Weverything` to catch deprecated API usage

---

## 11. Guideline 3.1.2 -- Subscriptions

**What it means**: Subscription terms are unclear, missing management link, or deceptive pricing.

**Prevention**:
- Display price, duration, and renewal terms before purchase
- Show free trial length clearly with auto-renewal disclosure
- Include "Manage Subscriptions" link in settings
- Implement Restore Purchases
- Handle grace periods and billing retry states

---

## 12. Guideline 5.1.1(v) -- Account Deletion

**What it means**: App supports account creation but doesn't offer account deletion.

**Prevention**:
- If users can create accounts, provide in-app account deletion
- Account deletion must actually delete data (not just deactivate)
- Deletion flow must be easy to find (Settings or Profile)
- Provide clear confirmation before deletion
- Handle subscription cancellation during account deletion

---

## 13. Guideline 2.3.3 -- Screenshots

**What it means**: Screenshots include device frames, status bars from wrong devices, or misleading UI.

**Prevention**:
- Capture screenshots from actual app running on target device
- If using device frames, match the actual device shown in the store listing
- Don't add elements to screenshots that aren't in the app
- Status bar should match the device or be removed

---

## 14. Guideline 3.2.2 -- Unacceptable Business Model

**What it means**: App's business model violates Apple's policies (gambling without license, pyramid schemes, etc.).

**Prevention**:
- Verify business model compliance before development
- Financial apps need proper licensing documentation
- Gambling apps need jurisdiction licenses
- Don't collect payments for nothing (no "tip jar" without value exchange)

---

## 15. Guideline 2.4.5 -- Apple Watch

**What it means**: Watch app doesn't function independently or provides no value.

**Prevention**:
- If including a Watch app, ensure it works without the iPhone app nearby
- Provide meaningful Watch-specific functionality
- Don't include an empty or placeholder Watch app

---

## 16. Guideline 4.3 -- Spam / Copycats

**What it means**: App duplicates another app's functionality without differentiation.

**Prevention**:
- Provide unique value or a meaningfully different approach
- Don't clone popular apps with minor UI changes
- Don't submit multiple versions of the same app

---

## 17. Guideline 5.2.1 -- Intellectual Property (Infringement)

**What it means**: App uses copyrighted content, trademarks, or intellectual property without permission.

**Prevention**:
- Only use content you own or have licensed
- Don't use Apple trademarks (Apple logo, product names) beyond what's permitted
- Verify all fonts, images, and sounds are properly licensed

---

## 18. Guideline 1.4.1 -- Physical Harm

**What it means**: App could lead to physical harm through medical advice, dangerous instructions, or weapons.

**Prevention**:
- Don't provide medical diagnoses or treatment recommendations
- Include disclaimers for health/fitness features
- Don't facilitate the sale of weapons or controlled substances

---

## 19. Guideline 3.1.3(b) -- Multiplatform Services

**What it means**: App tries to bypass IAP for multiplatform digital subscriptions without proper entitlement.

**Prevention**:
- If offering subscriptions on web and iOS, the iOS version must use StoreKit
- Reader apps (Netflix, Spotify) can link to web for sign-up under specific conditions
- Consult Apple's guidelines on Reader apps if applicable

---

## 20. Guideline 2.3.1 -- Hidden Features

**What it means**: App contains hidden features, undocumented URLs, or easter eggs that violate guidelines.

**Prevention**:
- Don't hide features from reviewers
- Don't use remote feature flags to enable guideline-violating features post-review
- Document all features in review notes
- Don't include features that activate based on location/time to avoid review detection

---

## Appeal Strategy (When Legitimately Wronged)

If you believe the rejection is a misunderstanding:

1. **Read the rejection carefully** -- identify the specific guideline cited
2. **Respond with facts** -- explain why your implementation complies, citing the specific guideline section
3. **Provide screenshots/video** -- show the reviewer what they may have missed
4. **Be professional** -- don't argue, don't be condescending, present evidence
5. **Request App Review Board** -- if the resolution center doesn't help, escalate to the App Review Board
6. **Contact Apple Developer Support** -- for complex cases, especially regulatory/financial apps

```
Appeal template:
Dear App Review Team,

Thank you for reviewing [App Name]. Regarding the rejection under Guideline [X.X]:

Our implementation complies because:
1. [Specific technical/business explanation]
2. [Reference to guideline language that supports your case]
3. [Evidence: screenshots, documentation, or Apple's own guidance]

We have also included [additional context] in the review notes.

Please re-review at your convenience. We're happy to provide additional information.

Best regards,
[Name]
```

# App Store Review Guidelines -- Compliance Reference

Detailed compliance mapping for each App Store Review Guidelines section relevant to iOS apps.

## 1. Safety (Guidelines 1.x)

| Rule | Requirement | Compliance Notes | Common Violation |
|------|-------------|------------------|-----------------|
| 1.1 | Objectionable content | Ensure no objectionable UGC if applicable | Unmoderated user content |
| 1.1.6 | False information | Don't present false info as fact | Health misinformation |
| 1.2 | User generated content | Implement moderation, reporting, blocking if UGC present | Missing report button |
| 1.3 | Kids category | Mark if targeting children (COPPA), no ads/tracking | Third-party analytics in kids apps |
| 1.4 | Physical harm | No medical diagnoses, weapons facilitation | Health apps giving diagnoses |
| 1.5 | Developer info | Contact info provided in App Store Connect | Missing support email |
| 1.6 | Developer Code of Conduct | Don't manipulate reviews, inflate rankings | Prompting for 5-star reviews |

## 2. Performance (Guidelines 2.x)

| Rule | Requirement | Compliance Notes | Common Violation |
|------|-------------|------------------|-----------------|
| 2.1 | App completeness | Fully functional, no placeholder content, no crashes | "Coming soon" features, lorem ipsum |
| 2.2 | Beta/demo labels | Remove all beta, test, demo references in production | "Beta" in navigation bar |
| 2.3 | Accurate metadata | Screenshots and description match real functionality | Marketing screenshots vs actual UI |
| 2.3.1 | Hidden features | No features hidden from reviewers | Remote config enabling post-review |
| 2.3.3 | Screenshots | Must show actual app UI, proper device frames | Photoshopped screenshots |
| 2.4 | Hardware compatibility | Universal iOS support for declared targets | iPad layout not implemented |
| 2.4.5 | Apple Watch | Watch app must work independently if included | Empty Watch companion |
| 2.5 | Software requirements | Use latest stable, public APIs only | Private API usage, deprecated APIs |
| 2.5.1 | Only public APIs | No private frameworks, no dynamic loading tricks | Using IOKit directly |

## 3. Business (Guidelines 3.x)

| Rule | Requirement | Compliance Notes | Common Violation |
|------|-------------|------------------|-----------------|
| 3.1.1 | In-app purchase | Digital goods/subscriptions via StoreKit | Stripe for premium features |
| 3.1.2 | Subscriptions | Clear terms, management link, restore button | Missing renewal disclosure |
| 3.1.3 | Multiplatform | iOS subscriptions must use StoreKit even if also on web | Web payment link in app |
| 3.1.5(a) | P2P payments | Allowed for person-to-person money transfers | N/A for money transfer apps |
| 3.1.7 | Advertising | Ads appropriate for age rating, ad SDKs disclosed | Undisclosed ad network data |
| 3.2.1 | Acceptable business | Verify business model compliance | Pyramid scheme, illegal gambling |
| 3.2.2 | Unacceptable | No predatory practices, fake subscriptions | Dark patterns in IAP flow |

## 4. Design (Guidelines 4.x)

| Rule | Requirement | Compliance Notes | Common Violation |
|------|-------------|------------------|-----------------|
| 4.0 | HIG compliance | Native SwiftUI/UIKit patterns, platform conventions | Custom navigation breaking conventions |
| 4.1 | Copycats | Don't clone other apps' UI/branding | Similar icon to popular app |
| 4.2 | Minimum functionality | Full feature set, not a thin wrapper | WKWebView wrapping website |
| 4.3 | Spam | Don't submit duplicate/similar apps | Multiple white-label versions |
| 4.4 | Extensions | Extensions must be functional and valuable | Empty share extension |
| 4.5 | Apple sites | Don't mimic Apple interfaces or use Apple branding | Custom alert looking like system alert |
| 4.7 | HTML5 games | HTML5 games in apps must be native-quality | Laggy web game in WebView |

## 5. Legal (Guidelines 5.x)

| Rule | Requirement | Compliance Notes | Common Violation |
|------|-------------|------------------|-----------------|
| 5.1 | Privacy | Privacy policy URL required, ATT if tracking | Missing privacy policy |
| 5.1.1 | Data collection | Privacy labels accurate, all collection disclosed | SDK data not disclosed |
| 5.1.1(v) | Account deletion | Must offer account deletion if sign-up exists | No delete account option |
| 5.1.2 | Data use | All data use disclosed in privacy policy | Analytics not mentioned in policy |
| 5.1.3 | Health data | HealthKit data handling rules | Sharing HealthKit data with third parties |
| 5.1.4 | Kids | COPPA compliance, no ads/tracking | Facebook SDK in kids app |
| 5.2 | Intellectual property | All content original or licensed | Copyrighted images |
| 5.2.1 | Trademarks | Don't use others' trademarks without permission | Apple logo in app |
| 5.3 | Gaming/gambling | License required for real-money gambling | Unlicensed betting features |
| 5.4 | VPN apps | Must use NEVPNManager, clear purpose | VPN for content circumvention |
| 5.6 | Developer Code of Conduct | Comply with reviewer communications | Abusive appeal responses |

## Financial App Specific Requirements

| Requirement | Details | Evidence to Provide |
|-------------|---------|-------------------|
| Licensing | Money transmitter license or partnership | License number in review notes |
| Fee disclosure | Fees shown before transaction confirmation | Screenshots of fee display |
| Transaction receipts | Confirmation shown/emailed after transaction | Screenshot of receipt |
| Customer support | Accessible support contact within app | Settings > Help/Support |
| Regulatory compliance | Compliance with local financial regulations | Regulatory disclosures in app |
| KYC/AML | Identity verification where required by law | Demo account with completed KYC |
| Fraud prevention | Measures to prevent fraudulent transactions | Description of fraud prevention in notes |
| Data encryption | Financial data encrypted in transit and at rest | Technical description in notes |

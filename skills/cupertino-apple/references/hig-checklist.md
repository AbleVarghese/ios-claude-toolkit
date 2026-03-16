# Human Interface Guidelines Checklist

HIG compliance checklist and rejection prevention strategies for iOS apps.

## Navigation

- [ ] Standard navigation patterns (NavigationStack, not custom back buttons)
- [ ] Clear back button behavior (system back button, not custom)
- [ ] Tab bar for main sections (max 5 tabs)
- [ ] Modal presentation for tasks (not for navigation)
- [ ] Sheet presentation for supplementary content
- [ ] Consistent navigation throughout app (don't mix paradigms)
- [ ] Toolbar items follow platform conventions
- [ ] Search uses `.searchable()` modifier

## Visual Design

- [ ] SF Symbols for system-style icons (latest version)
- [ ] Dynamic Type support across ALL text
- [ ] Dark Mode support with semantic colors
- [ ] Safe area respect (no content under notch/home indicator)
- [ ] System colors when appropriate (`.primary`, `.secondary`, `.accent`)
- [ ] Consistent spacing using standard values (4, 8, 12, 16, 20, 24)
- [ ] Card-style or inset grouped style for forms and settings
- [ ] System backgrounds (`.background`, not hardcoded white/black)
- [ ] Blur effects for overlapping content (`.ultraThinMaterial`)
- [ ] Semantic content shapes (`.containerShape`, rounded rectangles)

## Layout

- [ ] Responsive to all screen sizes (SE to Pro Max)
- [ ] iPad layout adaptation (sidebar, split view where appropriate)
- [ ] Landscape support (or explicit opt-out with justification)
- [ ] Content within readable width on large screens
- [ ] No horizontal scrolling for primary content
- [ ] Scroll views for content that may exceed screen height
- [ ] Safe area insets respected on all edges

## Interaction

- [ ] Touch targets >= 44pt x 44pt
- [ ] Haptic feedback for meaningful actions (success, error, selection)
- [ ] Pull-to-refresh where data can be refreshed
- [ ] Swipe actions for list items (delete, archive, etc.)
- [ ] Long-press context menus where appropriate
- [ ] Standard gestures (no custom gestures that conflict with system)
- [ ] Undo support for destructive actions
- [ ] Confirmation alerts for irreversible actions

## Typography

- [ ] Dynamic Type styles used (`.body`, `.headline`, `.title`, etc.)
- [ ] No fixed font sizes (or with `.relativeTo` for custom fonts)
- [ ] Text truncation handled gracefully (with `.lineLimit` and `.truncationMode`)
- [ ] Text scales to all accessibility sizes
- [ ] Readable fonts (no decorative fonts for body text)

## Color

- [ ] System semantic colors used (`.primary`, `.secondary`, `.accent`)
- [ ] Custom colors defined with light/dark variants
- [ ] High contrast mode support
- [ ] Color contrast meets WCAG AA (4.5:1 for text, 3:1 for large text)
- [ ] Color is never the sole indicator of information

## Accessibility

- [ ] VoiceOver labels on all interactive elements
- [ ] VoiceOver hints on non-obvious interactions
- [ ] Dynamic Type across all text
- [ ] Reduce Motion support (no spring animations when enabled)
- [ ] Reduce Transparency support
- [ ] High contrast mode compatibility
- [ ] Voice Control labels on interactive elements
- [ ] Switch Control navigable

## Loading and Feedback

- [ ] Loading states shown (ProgressView, skeleton screens)
- [ ] Error states handled with user-friendly messages
- [ ] Empty states shown with helpful guidance
- [ ] Success confirmation after actions (visual + haptic)
- [ ] Activity indicators for network operations
- [ ] No blocking UI for background operations

## System Integration

- [ ] Widgets (if applicable, using WidgetKit)
- [ ] Shortcuts / Siri integration (if applicable)
- [ ] Spotlight indexing for searchable content
- [ ] Universal Links for web-to-app linking
- [ ] Handoff support (if applicable)
- [ ] Share sheet integration (if applicable)

---

## App Store Rejection Prevention

| Rejection Reason | Prevention | Guideline |
|------------------|------------|-----------|
| Incomplete info | All fields in App Store Connect filled | 2.1 |
| Broken links | Test all URLs before submission | 2.1 |
| Placeholder content | Remove all lorem ipsum, "coming soon" | 2.1 |
| Crasher | Test on physical devices, all supported OS versions | 2.1 |
| Privacy violation | Accurate privacy labels, privacy manifests | 5.1.1 |
| No demo account | Provide test credentials with review notes | 2.1 |
| Insufficient features | App provides native value beyond website | 4.2 |
| Non-native UI | Use standard navigation, system controls | 4.0 |
| Missing account deletion | Add delete account flow in settings | 5.1.1(v) |
| Wrong IAP type | Digital goods via StoreKit, not external payment | 3.1.1 |

## Common Rejection Causes for Financial Apps

| Cause | Mitigation | Priority |
|-------|------------|----------|
| Missing license info | Include money transmitter license or partnership details in review notes | Critical |
| Unclear fee disclosure | Show fees prominently before transaction confirmation | Critical |
| No demo account | Provide test credentials with pre-funded account | Critical |
| Privacy label mismatch | Audit all data collection against declared privacy labels | Critical |
| Missing purpose strings | Include all required NSUsageDescription entries in Info.plist | High |
| No KYC bypass for review | Provide pre-verified demo account | High |
| Missing customer support | Add Help/Support accessible from within the app | High |
| Transaction receipts missing | Show confirmation screen with receipt details | Medium |
| Regulatory disclosures | Include required legal disclaimers for financial services | Medium |
| Data encryption undisclosed | Describe encryption methods in review notes | Medium |

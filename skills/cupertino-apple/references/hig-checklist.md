# Human Interface Guidelines Checklist

HIG compliance checklist and rejection prevention strategies for iOS apps.

## Navigation
- Standard navigation patterns (NavigationStack)
- Clear back button behavior
- Tab bar for main sections
- Modal presentation for tasks

## Visual Design
- SF Symbols for icons
- Dynamic Type support
- Dark Mode support
- Safe area respect
- System colors when appropriate

## Interaction
- Touch targets >= 44pt
- Haptic feedback for actions
- Pull-to-refresh where appropriate
- Swipe actions for lists

## Accessibility
- VoiceOver labels on all interactive elements
- Dynamic Type across all text
- Reduce Motion support
- High contrast mode compatibility

## App Store Rejection Prevention

| Rejection Reason | Prevention |
|------------------|------------|
| Incomplete info | All fields in App Store Connect filled |
| Broken links | Test all URLs before submission |
| Placeholder content | Remove all lorem ipsum |
| Crasher | Test on multiple devices |
| Privacy violation | Accurate privacy labels |
| Guideline 2.1 | Fully functional demo account |
| Guideline 4.2 | Sufficient features |

## Common Rejection Causes for Financial Apps

| Cause | Mitigation |
|-------|------------|
| Missing license info | Include money transmitter license or partnership details in review notes |
| Unclear fee disclosure | Show fees prominently before transaction confirmation |
| No demo account | Provide test credentials with review notes |
| Privacy label mismatch | Audit all data collection against declared privacy labels |
| Missing purpose strings | Include all required NSUsageDescription entries in Info.plist |

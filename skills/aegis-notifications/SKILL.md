---
name: aegis-notifications
description: "Send a notification, push notification, alert the user, remind the user. iOS push and local notification implementation — APNs setup, scheduling, rich content, actions, permission handling. Trigger on: 'notify the user', 'send a push', 'schedule a reminder', 'show a badge', 'background update', 'silent push', 'notification categories', 'ask for notification permission', or any work involving alerting users of events. Even indirect phrases like 'tell the user when it's done' or 'ping them' should trigger this skill."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: false
---

# AEGIS - Push Notifications & Alerts

> **Iron Law**: "Never send a notification without the user's informed consent. Every notification must be timely, relevant, and actionable."

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

Implement timely, relevant, and actionable notifications -- critical for status updates, security alerts, and user engagement in any iOS application. This skill covers the full notification lifecycle: permission handling, push and local notification delivery, rich content, notification actions, deep linking, and interruption level management.

## Notification Types

| Type | Priority | Use Case |
|------|----------|----------|
| Order Confirmed | High | Confirmation of completed action |
| Item Received | High | Delivery/arrival notification |
| Action Failed | Critical | Failure alert requiring attention |
| Security Alert | Critical | Suspicious activity |
| Account Verified | Medium | Verification complete |
| Price Alert | Low | Price/rate change updates |

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|---|---|---|
| "Every feature needs a notification" | Notification fatigue is the #1 reason users disable notifications entirely. One irrelevant notification can cause the user to turn off all notifications for your app permanently. That is a one-way door. | Every notification must pass the test: Is this timely? Is it relevant to this specific user right now? Can the user take action on it? If any answer is no, don't send it. |
| "Users want to be notified about everything" | They don't. Users want to be notified about things that require their attention. "Your weekly summary is ready" is useful. "Someone you don't follow posted something" is not. Study your notification opt-out rate -- it tells you what users actually think. | Categorize notifications by importance. Let users control which categories they receive. Default to fewer notifications, not more. |
| "We'll add notification preferences later" | By "later," users have already disabled notifications entirely because you sent too many irrelevant ones. You cannot undo this -- the user must manually re-enable in Settings, which almost never happens. | Build notification preferences into the first version. At minimum: on/off per notification type. Better: frequency controls and quiet hours. |
| "Silent push is just like regular push" | Silent push has strict throttling (Apple limits frequency), is not guaranteed to wake the app, and has a 30-second execution window. It's a best-effort background update mechanism, not a reliable delivery channel. | Use silent push only for background data sync where missed updates are acceptable. Never use it for time-critical notifications. For reliable delivery, use visible push. |
| "We can just request permission on app launch" | Requesting permission with no context results in low opt-in rates (typically 40-50%). Users don't know what they're agreeing to or why they should. | Use provisional authorization for low-commitment opt-in, or show an in-app pre-permission screen explaining what notifications the user will receive and why. Then request system permission. Opt-in rates increase to 70-80%. |
| "Badge count doesn't matter" | An incorrect badge count (especially a stuck non-zero badge) trains users to ignore badges entirely, or worse, creates anxiety. A badge that says "3" when there are 0 unread items erodes trust. | Manage badge count server-side. Update it with every push payload. Clear it when the user opens the app. Never let it drift. |
| "We don't need notification categories" | Without categories, every notification tap opens the app to the home screen. The user then has to navigate to find what the notification was about. This is a poor experience that teaches users to ignore notifications. | Define categories with appropriate actions. Handle notification taps by deep-linking to the relevant content. |

---

## Red Flags -- STOP

- **Sending notifications without checking permission status** -- Always check `UNUserNotificationCenter.current().notificationSettings()` before attempting to send. Sending without permission wastes server resources and the notification is silently dropped.
- **Not handling notification tap navigation** -- If tapping a notification opens the app to the home screen instead of the relevant content, the notification is useless. Every notification type must have a deep link target.
- **Missing notification categories** -- Categories define what actions appear on a notification and how the app handles taps. Without categories, you lose actionable notifications and structured handling entirely.
- **No notification preferences UI** -- If users cannot control which notifications they receive within the app, their only option is the nuclear one: disabling all notifications in Settings.
- **Logging device tokens or push payloads at public privacy level** -- Device tokens are user-identifying. Push payloads may contain sensitive data. Use `.private` or `.sensitive` log levels.

---

## Behavioral Enforcement

### Phase Gate: PERMISSION CHECK (Before Any Notification)
**Cannot send or schedule ANY notification until:**
- [ ] Permission status checked (`UNUserNotificationCenter.current().getNotificationSettings()`)
- [ ] All authorization states handled: .authorized, .denied, .notDetermined, .provisional
- [ ] Denied state has graceful degradation (not a crash or silent failure)

**Hard Stop**: Sending a notification without checking permission status will fail silently on denied, crash on edge cases, and violate Apple's guidelines. Always check first.

### Phase Gate: NOTIFICATION CONTENT
**Before sending any notification:**
- [ ] Content is timely (user needs this NOW, not later)
- [ ] Content is relevant (user opted into this type)
- [ ] Content is actionable (user can DO something with this information)
- [ ] No sensitive data in notification body (visible on lock screen)

**Hard Stop**: If notification contains PII, financial data, or authentication tokens in the body -- STOP. This is a security violation. Notification content is visible on the lock screen to anyone with physical access.

### Self-Audit
1. Did I check notification permission before sending?
2. Does the notification pass the "timely, relevant, actionable" test?
3. Is sensitive data excluded from notification body?
4. Did I handle the notification tap (deep linking to correct screen)?

### Required Output Artifact
Every notification implementation must produce:
- Permission handling code that covers all authorization states
- Category registration with appropriate actions
- Deep link handler for notification taps (no home-screen-only navigation)
- Evidence that no PII appears in notification body content

---

## When NOT to Use This Skill

1. **In-app messaging and alerts** -- For showing alerts, banners, or toasts while the user is actively using the app, use SwiftUI `.alert()`, `.sheet()`, or a custom in-app notification system. Push/local notifications are for when the user is not looking at the app.
2. **Background data sync without user-visible notification** -- For background data fetching that doesn't need to notify the user, use `BGAppRefreshTask` or `BGProcessingTask` from the BackgroundTasks framework. Silent push is throttled and unreliable for periodic sync.
3. **Real-time communication (chat, calls)** -- For real-time messaging, use a persistent connection (WebSocket, XMPP) with push notifications as a fallback for when the app is suspended. Don't rely on push as the primary delivery mechanism for real-time data.
4. **Marketing and promotional content** -- Push notifications for marketing require explicit user consent separate from transactional notifications. In many jurisdictions (EU, California), promotional push notifications are regulated. Consult legal before implementing.

---

## Decision Framework

```
What kind of notification do I need?

+-- User needs to see this NOW?
|   |
|   +-- Critical (safety, security, health)?
|   |   --> Interruption level: .critical
|   |   Requires: entitlement from Apple (apply in developer portal)
|   |   Behavior: Plays sound even in DND/Focus, overrides ringer switch
|   |
|   +-- Time-sensitive (delivery, expiring offer, ride arriving)?
|   |   --> Interruption level: .timeSensitive
|   |   Requires: Time Sensitive Notifications capability
|   |   Behavior: Breaks through Focus mode, appears on lock screen prominently
|   |
|   +-- Normal (message, transaction complete, status update)?
|       --> Interruption level: .active (default)
|       Behavior: Standard notification, respects DND/Focus
|
+-- User should see this when convenient?
|   --> Interruption level: .passive
|   Behavior: Silently added to Notification Center, no sound/banner
|   Good for: recommendations, weekly summaries, non-urgent updates
|
+-- Need to trigger background work (no visible notification)?
|   --> Silent push notification
|   Set: content-available: 1, no alert/sound/badge
|   Limitations: throttled by iOS, 30s execution, not guaranteed
|   Good for: data sync, content prefetch, state invalidation
|
+-- Notification at a specific time (no server needed)?
    --> Local notification
    Triggers: UNTimeIntervalNotificationTrigger (delay)
              UNCalendarNotificationTrigger (specific date/time)
              UNLocationNotificationTrigger (geofence)
    Good for: reminders, alarms, location-based alerts

How should the notification be delivered?

+-- From your server --> Push notification (APNs)
|   Setup: APNs key (.p8), server integration, device token management
|
+-- From the device itself --> Local notification
    Setup: UNUserNotificationCenter, no server needed
```

---

## Standard Operating Procedures

### SOP-1: Request Notification Permission

```swift
import UserNotifications
import UIKit

/// Requests notification permission with informed consent.
///
/// Call this AFTER showing an in-app explanation of what notifications the user
/// will receive. Never call this on first launch with no context.
///
/// - Returns: Whether permission was granted
func requestNotificationPermission() async throws -> Bool {
    let center = UNUserNotificationCenter.current()

    // Check current status first -- don't re-prompt if already decided
    let settings = await center.notificationSettings()

    switch settings.authorizationStatus {
    case .authorized, .provisional:
        // Already authorized -- just ensure registered for remote
        await MainActor.run {
            UIApplication.shared.registerForRemoteNotifications()
        }
        return true

    case .denied:
        // User explicitly denied -- don't re-request (system won't show prompt)
        // Guide user to Settings if they want to re-enable
        return false

    case .notDetermined:
        // First time -- request authorization
        let options: UNAuthorizationOptions = [
            .alert,
            .sound,
            .badge,
            .providesAppNotificationSettings  // Shows in-app settings button in system Settings
        ]

        let granted = try await center.requestAuthorization(options: options)

        if granted {
            await MainActor.run {
                UIApplication.shared.registerForRemoteNotifications()
            }
        }

        return granted

    case .ephemeral:
        // App Clip -- limited notification support
        return true

    @unknown default:
        return false
    }
}

/// Request provisional authorization (try-before-you-buy).
///
/// Provisional notifications appear silently in Notification Center.
/// The user sees "Keep" or "Turn Off" after the first few, deciding
/// without an upfront prompt. Great for increasing opt-in rates.
func requestProvisionalPermission() async throws -> Bool {
    let center = UNUserNotificationCenter.current()

    let options: UNAuthorizationOptions = [
        .alert, .sound, .badge, .provisional
    ]

    let granted = try await center.requestAuthorization(options: options)

    if granted {
        await MainActor.run {
            UIApplication.shared.registerForRemoteNotifications()
        }
    }

    return granted
}
```

**Why provisional authorization matters:**
Standard permission request shows a system dialog: "Allow Notifications?" Users who don't understand the value say "Don't Allow," and that decision is very difficult to reverse. Provisional authorization skips the dialog, delivers notifications quietly, and lets the user decide based on actual notifications they've received. Opt-in rates are dramatically higher.

### SOP-2: Handle Push Notification Registration

```swift
// In AppDelegate -- these callbacks cannot be in SceneDelegate

func application(
    _ application: UIApplication,
    didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
) {
    // Convert token to hex string for your server
    let tokenString = deviceToken.map { String(format: "%02x", $0) }.joined()

    // Send to your server
    // IMPORTANT: Token can change between app launches. Always send the latest.
    Task {
        await sendDeviceTokenToServer(tokenString)
    }
}

func application(
    _ application: UIApplication,
    didFailToRegisterForRemoteNotificationsWithError error: Error
) {
    // This fails on Simulator (no APNs support) -- don't treat as fatal
    // On device, this indicates a provisioning or entitlement issue
    logger.error("Failed to register for remote notifications: \(error.localizedDescription, privacy: .public)")
}
```

### SOP-3: Handle Incoming Push Notifications

```swift
// In AppDelegate -- handles background push (including silent push)
func application(
    _ application: UIApplication,
    didReceiveRemoteNotification userInfo: [AnyHashable: Any]
) async -> UIBackgroundFetchResult {

    // Silent push: content-available = 1, no alert
    guard let type = userInfo["type"] as? String else {
        return .noData
    }

    switch type {
    case "data_sync":
        // Silent push: sync data in background
        // You have ~30 seconds to complete
        let success = await performBackgroundSync()
        return success ? .newData : .failed

    case "content_update":
        // Silent push: invalidate cache, prefetch new content
        await invalidateContentCache(userInfo)
        return .newData

    default:
        return .noData
    }
}

// UNUserNotificationCenterDelegate -- handles foreground + tap
// Set delegate in application(_:didFinishLaunchingWithOptions:)
// UNUserNotificationCenter.current().delegate = self

extension AppDelegate: UNUserNotificationCenterDelegate {

    /// Called when notification arrives while app is in foreground
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification
    ) async -> UNNotificationPresentationOptions {
        let userInfo = notification.request.content.userInfo

        // Decide whether to show the notification in foreground
        // If the user is already looking at the relevant screen, don't show it
        if isUserViewingRelatedContent(userInfo) {
            // Update the UI directly instead of showing a notification
            await updateUIWithNewData(userInfo)
            return []  // Don't show notification
        }

        // Show notification even though app is in foreground
        return [.banner, .sound, .badge]
    }

    /// Called when user taps a notification
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse
    ) async {
        let userInfo = response.notification.request.content.userInfo

        switch response.actionIdentifier {
        case UNNotificationDefaultActionIdentifier:
            // User tapped the notification body -- navigate to content
            await handleNotificationTap(userInfo)

        case UNNotificationDismissActionIdentifier:
            // User dismissed the notification
            break

        case "APPROVE_ACTION":
            // Custom action: user tapped "Approve"
            await handleApproveAction(userInfo)

        case "REJECT_ACTION":
            // Custom action: user tapped "Reject"
            await handleRejectAction(userInfo)

        default:
            break
        }
    }
}
```

### SOP-4: Schedule Local Notification

```swift
import UserNotifications

/// Schedules a local notification with proper configuration.
///
/// - Parameters:
///   - identifier: Unique ID for this notification (use for cancellation)
///   - title: Notification title (keep short -- truncated after ~50 chars)
///   - body: Notification body (keep under 150 chars for full visibility)
///   - trigger: When to fire (time interval, calendar, or location)
///   - categoryIdentifier: Category for actions (must be registered first)
///   - userInfo: Custom data for handling tap navigation
func scheduleLocalNotification(
    identifier: String,
    title: String,
    body: String,
    trigger: UNNotificationTrigger,
    categoryIdentifier: String? = nil,
    userInfo: [AnyHashable: Any] = [:]
) async throws {
    let content = UNMutableNotificationContent()
    content.title = title
    content.body = body
    content.sound = .default
    content.userInfo = userInfo

    if let categoryIdentifier {
        content.categoryIdentifier = categoryIdentifier
    }

    let request = UNNotificationRequest(
        identifier: identifier,
        content: content,
        trigger: trigger
    )

    try await UNUserNotificationCenter.current().add(request)
}

// USAGE EXAMPLES:

// Time-based: Fire in 1 hour
let timeTrigger = UNTimeIntervalNotificationTrigger(
    timeInterval: 3600,
    repeats: false
)

// Calendar-based: Fire at 9 AM tomorrow
var dateComponents = DateComponents()
dateComponents.hour = 9
dateComponents.minute = 0
let calendarTrigger = UNCalendarNotificationTrigger(
    dateMatching: dateComponents,
    repeats: false  // Set true for daily recurring
)

// Location-based: Fire when entering a region
let center = CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194)
let region = CLCircularRegion(center: center, radius: 100, identifier: "office")
region.notifyOnEntry = true
region.notifyOnExit = false
let locationTrigger = UNLocationNotificationTrigger(
    region: region,
    repeats: false
)

// Cancel a pending notification
UNUserNotificationCenter.current().removePendingNotificationRequests(
    withIdentifiers: ["reminder-\(itemId)"]
)

// Cancel a delivered (visible) notification
UNUserNotificationCenter.current().removeDeliveredNotifications(
    withIdentifiers: ["reminder-\(itemId)"]
)
```

### SOP-5: Notification Categories and Actions

```swift
/// Register notification categories at app launch.
///
/// Call this in application(_:didFinishLaunchingWithOptions:) BEFORE
/// any notifications arrive. Categories define what actions are available
/// and how the system handles notifications of each type.
func registerNotificationCategories() {
    // CATEGORY 1: Actionable item (approve/reject)
    let approveAction = UNNotificationAction(
        identifier: "APPROVE_ACTION",
        title: "Approve",
        options: [.authenticationRequired]  // Requires Face ID / passcode
    )

    let rejectAction = UNNotificationAction(
        identifier: "REJECT_ACTION",
        title: "Reject",
        options: [.destructive, .authenticationRequired]  // Red text + auth
    )

    let actionCategory = UNNotificationCategory(
        identifier: "ACTION_APPROVAL",
        actions: [approveAction, rejectAction],
        intentIdentifiers: [],
        options: [.customDismissAction]  // Notifies delegate on dismiss too
    )

    // CATEGORY 2: Message (reply inline)
    let replyAction = UNTextInputNotificationAction(
        identifier: "REPLY_ACTION",
        title: "Reply",
        options: [],
        textInputButtonTitle: "Send",
        textInputPlaceholder: "Type a message..."
    )

    let messageCategory = UNNotificationCategory(
        identifier: "MESSAGE",
        actions: [replyAction],
        intentIdentifiers: [],
        options: []
    )

    // CATEGORY 3: Delivery update (view details / contact support)
    let viewAction = UNNotificationAction(
        identifier: "VIEW_DETAILS",
        title: "View Details",
        options: [.foreground]  // Opens the app
    )

    let contactAction = UNNotificationAction(
        identifier: "CONTACT_SUPPORT",
        title: "Contact Support",
        options: [.foreground]
    )

    let deliveryCategory = UNNotificationCategory(
        identifier: "DELIVERY_UPDATE",
        actions: [viewAction, contactAction],
        intentIdentifiers: [],
        options: []
    )

    // Register all categories
    UNUserNotificationCenter.current().setNotificationCategories([
        actionCategory,
        messageCategory,
        deliveryCategory
    ])
}
```

### SOP-6: Notification Grouping and Threading

```swift
// Group notifications by conversation, order, or topic
let content = UNMutableNotificationContent()
content.title = "New message from Alice"
content.body = "Hey, are you free for lunch?"

// Thread identifier groups notifications in Notification Center
content.threadIdentifier = "conversation-\(conversationId)"

// Summary format for grouped notifications
// When 3+ notifications are grouped:
// "5 more messages from Alice"
content.summaryArgument = "Alice"
content.summaryArgumentCount = 1

// In the push payload, set thread-id:
// {
//   "aps": {
//     "alert": { "title": "...", "body": "..." },
//     "thread-id": "conversation-abc123"
//   }
// }
```

### SOP-7: Interruption Levels (iOS 15+)

```swift
// Set interruption level based on notification urgency
let content = UNMutableNotificationContent()
content.title = "Security Alert"
content.body = "New sign-in from unknown device"

// .passive -- silent, goes to Notification Center only
// .active -- default, sound + banner, respects Focus
// .timeSensitive -- breaks through Focus mode
// .critical -- breaks through DND + Focus, ignores ringer switch (requires entitlement)
content.interruptionLevel = .timeSensitive

// In push payload:
// {
//   "aps": {
//     "alert": { ... },
//     "interruption-level": "time-sensitive"
//   }
// }

// Relevance score (0.0 to 1.0) determines ordering in notification summary
content.relevanceScore = 0.9  // High relevance -- shown first in summary
```

---

## Push Notification Payload Reference

```json
{
  "aps": {
    "alert": {
      "title": "Item Received",
      "subtitle": "From Alice",
      "body": "You received a $100.00 USD payment"
    },
    "sound": "default",
    "badge": 3,
    "category": "ITEM_RECEIVED",
    "thread-id": "transactions-alice",
    "interruption-level": "active",
    "relevance-score": 0.8,
    "content-available": 0,
    "mutable-content": 1
  },
  "type": "item_received",
  "item_id": "uuid-here",
  "deep_link": "/transactions/uuid-here",
  "amount": 10000,
  "currency": "USD"
}
```

**Payload field reference:**
- `alert.title`: Bold text, first line (~50 chars visible)
- `alert.subtitle`: Second line, smaller text (optional)
- `alert.body`: Main content (~150 chars visible on lock screen)
- `sound`: `"default"` or custom sound filename (must be <30 seconds, in app bundle)
- `badge`: App icon badge number (set to 0 to clear)
- `category`: Matches registered `UNNotificationCategory` identifier
- `thread-id`: Groups notifications in Notification Center
- `interruption-level`: `"passive"`, `"active"`, `"time-sensitive"`, `"critical"`
- `content-available`: Set to `1` for silent push (background fetch)
- `mutable-content`: Set to `1` to trigger Notification Service Extension (modify content before display)

---

## Notification Architecture

See `references/notification-architecture.md` for comprehensive setup:
- APNs setup end-to-end (capability, key, server configuration)
- Certificate vs Key authentication (p8 vs p12)
- Token management (registering, storing, refreshing)
- Silent push limitations and reliability considerations
- Notification Service Extension (modifying content, downloading media)
- Multi-environment setup (sandbox vs production APNs)

## Deep Linking from Notifications

See `references/deep-linking.md` for navigation patterns:
- Parsing notification `userInfo` to determine destination
- Navigating to specific screens from notification taps
- Handling notifications in foreground vs background vs terminated states
- URL scheme and Universal Link handling

---

## Quality Gates (Before Marking Complete)

- [ ] Permission requested with context -- not on cold launch with no explanation
- [ ] `notificationSettings()` checked before attempting to schedule/send
- [ ] Every notification type has a registered category with appropriate actions
- [ ] Every notification tap navigates to the relevant content (deep link), not the home screen
- [ ] Foreground presentation handled (`willPresent`) -- notifications don't appear when user is already viewing the content
- [ ] Badge count managed correctly -- cleared on app open, updated with every push
- [ ] No PII in notification content visible on lock screen (consider `.hiddenPreviewsBodyPlaceholder`)
- [ ] Silent push used only for non-critical background updates (not time-sensitive delivery)
- [ ] Notification preferences UI exists -- users can control which types they receive
- [ ] Device token sent to server on every `didRegisterForRemoteNotificationsWithDeviceToken` (token can change)

---

## Cross-Skill References

- **velocity-fastlane** -- Push notification capability must be enabled in the Xcode project and included in the provisioning profile. When setting up Match, ensure the Push Notifications entitlement is present.
- **pipeline-cicd** -- Notification-dependent tests in CI must use mocked notification services. APNs is not available in CI environments. Test notification handling logic in isolation.
- **loki-logs** -- Notification delivery issues appear in device logs. Search for the `apsd` process (APNs daemon) and `UserNotifications` framework entries. Use `os.Logger` with a "notifications" category for structured debugging.
- **atlas-database** -- Notification preferences and device tokens are stored in the database. Ensure the device tokens table has: `user_id`, `token`, `platform`, `created_at`, `updated_at`. Soft-delete stale tokens; never hard-delete.

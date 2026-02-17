---
name: aegis-notifications
description: "This skill should be used when the user asks to 'push notifications', 'APNs setup', 'local notification', 'notification permission', 'rich notifications', or mentions in-app alerts. Provides iOS notification implementation patterns for APNs, local scheduling, rich content, and notification actions."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: false
---

# Push Notifications & Alerts

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, target) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift).

Implement timely, relevant, and actionable notifications -- critical for status updates and security alerts in any iOS application.

## Notification Types

| Type | Priority | Use Case |
|------|----------|----------|
| Order Confirmed | High | Confirmation of completed action |
| Item Received | High | Delivery/arrival notification |
| Action Failed | Critical | Failure alert requiring attention |
| Security Alert | Critical | Suspicious activity |
| Account Verified | Medium | Verification complete |
| Price Alert | Low | Price/rate change updates |

## Standard Operating Procedures

### SOP-1: Request Notification Permission
```swift
// Request authorization
func requestNotificationPermission() async throws -> Bool {
    let center = UNUserNotificationCenter.current()

    let options: UNAuthorizationOptions = [
        .alert, .sound, .badge, .providesAppNotificationSettings
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

### SOP-2: Handle Push Notification
```swift
// In AppDelegate or SceneDelegate
func application(
    _ application: UIApplication,
    didReceiveRemoteNotification userInfo: [AnyHashable: Any]
) async -> UIBackgroundFetchResult {

    guard let type = userInfo["type"] as? String else {
        return .noData
    }

    switch type {
    case "item_received":
        await handleItemReceived(userInfo)
        return .newData
    case "action_failed":
        await handleActionFailed(userInfo)
        return .newData
    default:
        return .noData
    }
}
```

### SOP-3: Schedule Local Notification
```swift
// Schedule notification
func scheduleReminder(itemId: String, itemDescription: String) async throws {
    let content = UNMutableNotificationContent()
    content.title = "Action Pending"
    content.body = "Your request for \(itemDescription) is still processing"
    content.sound = .default
    content.badge = 1

    let trigger = UNTimeIntervalNotificationTrigger(
        timeInterval: 3600, // 1 hour
        repeats: false
    )

    let request = UNNotificationRequest(
        identifier: "reminder-\(itemId)",
        content: content,
        trigger: trigger
    )

    try await UNUserNotificationCenter.current().add(request)
}
```

### SOP-4: Notification Actions
```swift
// Define notification category with actions
func setupNotificationCategories() {
    let approveAction = UNNotificationAction(
        identifier: "APPROVE_ACTION",
        title: "Approve",
        options: [.authenticationRequired]
    )

    let rejectAction = UNNotificationAction(
        identifier: "REJECT_ACTION",
        title: "Reject",
        options: [.destructive, .authenticationRequired]
    )

    let category = UNNotificationCategory(
        identifier: "ACTION_APPROVAL",
        actions: [approveAction, rejectAction],
        intentIdentifiers: [],
        options: []
    )

    UNUserNotificationCenter.current().setNotificationCategories([category])
}
```

## Notification Payload

```json
{
  "aps": {
    "alert": {
      "title": "Item Received!",
      "body": "Jane sent you a $100.00 USD payment"
    },
    "sound": "default",
    "badge": 1,
    "category": "ITEM_RECEIVED"
  },
  "type": "item_received",
  "item_id": "uuid-here",
  "amount": 10000,
  "currency": "USD"
}
```

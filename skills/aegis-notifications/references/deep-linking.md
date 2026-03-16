# Deep Linking from Notifications

How to navigate users to the right screen when they tap a notification.

---

## Core Pattern

Every notification should include routing information in `userInfo`. When the user taps, parse this data and navigate.

```
Notification Tap
  |
  v
UNUserNotificationCenterDelegate.didReceive(response:)
  |
  v
Parse response.notification.request.content.userInfo
  |
  v
Determine destination (screen + parameters)
  |
  v
Navigate (set navigation path, present modal, switch tab)
```

---

## Push Payload Design for Deep Linking

Include a `deep_link` field in your push payload. This is the routing key that tells the app where to navigate.

```json
{
    "aps": {
        "alert": { "title": "Payment Received", "body": "Alice sent you $100" },
        "category": "TRANSACTION"
    },
    "deep_link": "/transactions/abc123",
    "type": "transaction_received",
    "transaction_id": "abc123"
}
```

**Design principles:**
- `deep_link` uses URL-path style for consistency with Universal Links
- `type` identifies the notification type (for analytics and category-specific handling)
- Additional fields (`transaction_id`) provide parameters without URL parsing

---

## Handling Notification Taps

```swift
extension AppDelegate: UNUserNotificationCenterDelegate {

    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse
    ) async {
        let userInfo = response.notification.request.content.userInfo

        // Handle action buttons
        switch response.actionIdentifier {
        case UNNotificationDefaultActionIdentifier:
            // User tapped the notification body
            await navigateFromNotification(userInfo)

        case UNNotificationDismissActionIdentifier:
            // User dismissed (swiped away)
            break

        case "VIEW_DETAILS":
            // Custom action: navigate to detail view
            await navigateFromNotification(userInfo)

        case "REPLY_ACTION":
            // Text input action
            if let textResponse = response as? UNTextInputNotificationResponse {
                await handleReply(text: textResponse.userText, context: userInfo)
            }

        default:
            break
        }
    }
}
```

---

## Navigation Router

Centralize deep link resolution in a router. This keeps navigation logic out of the delegate and makes it testable.

```swift
/// Resolves notification data into a navigation destination.
enum NotificationDestination {
    case transaction(id: String)
    case conversation(id: String)
    case orderStatus(id: String)
    case settings
    case unknown
}

struct NotificationRouter {

    static func destination(from userInfo: [AnyHashable: Any]) -> NotificationDestination {
        // Primary: use deep_link path
        if let deepLink = userInfo["deep_link"] as? String {
            return resolve(path: deepLink, userInfo: userInfo)
        }

        // Fallback: use type field
        if let type = userInfo["type"] as? String {
            return resolve(type: type, userInfo: userInfo)
        }

        return .unknown
    }

    private static func resolve(
        path: String,
        userInfo: [AnyHashable: Any]
    ) -> NotificationDestination {
        let components = path.split(separator: "/").map(String.init)

        switch components.first {
        case "transactions":
            guard let id = components[safe: 1] else { return .unknown }
            return .transaction(id: id)

        case "conversations":
            guard let id = components[safe: 1] else { return .unknown }
            return .conversation(id: id)

        case "orders":
            guard let id = components[safe: 1] else { return .unknown }
            return .orderStatus(id: id)

        case "settings":
            return .settings

        default:
            return .unknown
        }
    }

    private static func resolve(
        type: String,
        userInfo: [AnyHashable: Any]
    ) -> NotificationDestination {
        switch type {
        case "transaction_received", "transaction_sent":
            guard let id = userInfo["transaction_id"] as? String else { return .unknown }
            return .transaction(id: id)

        case "new_message":
            guard let id = userInfo["conversation_id"] as? String else { return .unknown }
            return .conversation(id: id)

        case "order_update":
            guard let id = userInfo["order_id"] as? String else { return .unknown }
            return .orderStatus(id: id)

        default:
            return .unknown
        }
    }
}
```

---

## Performing Navigation

### SwiftUI (NavigationPath)

```swift
@MainActor
func navigateFromNotification(_ userInfo: [AnyHashable: Any]) {
    let destination = NotificationRouter.destination(from: userInfo)

    switch destination {
    case .transaction(let id):
        // Update the navigation state to show the transaction
        appState.selectedTab = .transactions
        appState.navigationPath.append(TransactionRoute.detail(id: id))

    case .conversation(let id):
        appState.selectedTab = .messages
        appState.navigationPath.append(ConversationRoute.detail(id: id))

    case .orderStatus(let id):
        appState.selectedTab = .orders
        appState.navigationPath.append(OrderRoute.status(id: id))

    case .settings:
        appState.selectedTab = .settings

    case .unknown:
        // Don't navigate -- stay on current screen
        break
    }
}
```

### UIKit (NavigationController)

```swift
@MainActor
func navigateFromNotification(_ userInfo: [AnyHashable: Any]) {
    let destination = NotificationRouter.destination(from: userInfo)

    // Dismiss any presented modals first
    window?.rootViewController?.dismiss(animated: false)

    guard let tabBarController = window?.rootViewController as? UITabBarController,
          let navigationController = tabBarController.selectedViewController as? UINavigationController
    else { return }

    switch destination {
    case .transaction(let id):
        tabBarController.selectedIndex = 0  // Transactions tab
        let vc = TransactionDetailViewController(transactionId: id)
        navigationController.pushViewController(vc, animated: true)

    case .conversation(let id):
        tabBarController.selectedIndex = 1  // Messages tab
        let vc = ConversationViewController(conversationId: id)
        navigationController.pushViewController(vc, animated: true)

    default:
        break
    }
}
```

---

## App State When Notification Arrives

The app can be in three states when a notification arrives. Each requires different handling.

### Foreground (app is visible)

```swift
func userNotificationCenter(
    _ center: UNUserNotificationCenter,
    willPresent notification: UNNotification
) async -> UNNotificationPresentationOptions {
    // App is in foreground. Decide whether to show the notification.
    let userInfo = notification.request.content.userInfo
    let destination = NotificationRouter.destination(from: userInfo)

    // If user is already viewing the relevant content, update inline
    if isCurrentlyViewing(destination) {
        await refreshCurrentView(with: userInfo)
        return []  // Don't show notification banner
    }

    // Otherwise, show the banner so the user knows about it
    return [.banner, .sound, .badge]
}
```

### Background (app is in memory but not visible)

The `didReceive(response:)` delegate method is called. The app is already running, so navigation works normally.

### Terminated (app was not running)

The `didReceive(response:)` delegate method is called after `application(_:didFinishLaunchingWithOptions:)`. The app needs to fully initialize before navigating.

```swift
func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
    // Set notification delegate BEFORE the app finishes launching
    // This ensures didReceive is called for cold-start taps
    UNUserNotificationCenter.current().delegate = self

    // Check if launched from notification
    if let notification = launchOptions?[.remoteNotification] as? [AnyHashable: Any] {
        // Store the notification data; navigate after UI is ready
        pendingNotificationUserInfo = notification
    }

    return true
}

// Navigate after the root view is ready
func sceneDidBecomeActive(_ scene: UIScene) {
    if let userInfo = pendingNotificationUserInfo {
        pendingNotificationUserInfo = nil
        Task {
            await navigateFromNotification(userInfo)
        }
    }
}
```

---

## Testing Deep Links

### Simulating a Push Notification in Development

Create a `.apns` file and drag it onto the simulator:

```json
{
    "Simulator Target Bundle": "com.yourapp.ios",
    "aps": {
        "alert": {
            "title": "Test Notification",
            "body": "Tap to navigate to transaction"
        },
        "category": "TRANSACTION",
        "sound": "default"
    },
    "deep_link": "/transactions/test-123",
    "type": "transaction_received",
    "transaction_id": "test-123"
}
```

Save as `test-push.apns` and drag onto the running simulator.

### Using xcrun to send test notifications

```bash
xcrun simctl push booted com.yourapp.ios test-push.apns
```

### Unit Testing the Router

```swift
func testTransactionDeepLink() {
    let userInfo: [AnyHashable: Any] = [
        "deep_link": "/transactions/abc123",
        "type": "transaction_received",
        "transaction_id": "abc123"
    ]

    let destination = NotificationRouter.destination(from: userInfo)

    guard case .transaction(let id) = destination else {
        XCTFail("Expected transaction destination")
        return
    }

    XCTAssertEqual(id, "abc123")
}

func testUnknownDeepLink() {
    let userInfo: [AnyHashable: Any] = [
        "deep_link": "/unknown/path"
    ]

    let destination = NotificationRouter.destination(from: userInfo)
    guard case .unknown = destination else {
        XCTFail("Expected unknown destination")
        return
    }
}
```

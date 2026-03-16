# Notification Architecture

End-to-end reference for iOS push notification infrastructure, from APNs setup to delivery.

---

## APNs Setup End-to-End

### Step 1: Enable Push Notifications Capability

1. Open Xcode project settings
2. Select your app target
3. Go to "Signing & Capabilities"
4. Click "+ Capability"
5. Add "Push Notifications"
6. This adds the `aps-environment` entitlement to your app

If using XcodeGen (`project.yml`):
```yaml
targets:
  MyApp:
    entitlements:
      path: MyApp/MyApp.entitlements
    settings:
      CODE_SIGN_ENTITLEMENTS: MyApp/MyApp.entitlements
```

Entitlements file (`MyApp.entitlements`):
```xml
<key>aps-environment</key>
<string>development</string>  <!-- or "production" for App Store builds -->
```

### Step 2: Generate APNs Authentication Key (.p8)

1. Go to [Apple Developer Portal](https://developer.apple.com/account/resources/authkeys/list)
2. Click "Keys" > "+" to create a new key
3. Name it (e.g., "APNs Key")
4. Check "Apple Push Notifications service (APNs)"
5. Click "Continue" > "Register"
6. Download the `.p8` file (you can only download it ONCE)
7. Note the Key ID and your Team ID

**Store securely:**
- The `.p8` file is your signing key for all APNs requests
- Never commit it to source control
- Store as a CI secret (base64 encoded)
- You only need one key per team (works for all apps)

### Step 3: Configure Your Server

Your server sends push notifications to APNs. It needs:
- The `.p8` key file (or its contents)
- The Key ID (10-character string from Apple)
- The Team ID (10-character string from Apple)
- The app's bundle identifier

```
Server -> APNs request:
  POST https://api.push.apple.com/3/device/{device_token}
  Headers:
    authorization: bearer {JWT signed with .p8 key}
    apns-topic: {bundle_id}
    apns-push-type: alert (or background, voip, etc.)
    apns-priority: 10 (immediate) or 5 (power-efficient)
  Body: {push payload JSON}
```

---

## Certificate vs Key Authentication

| Feature | Key Auth (.p8) | Certificate Auth (.p12) |
|---|---|---|
| Validity | Never expires (until revoked) | 1 year |
| Scope | All apps in the team | One app (or one app + extensions) |
| Renewal effort | None | Annual re-generation |
| Recommended | Yes | No (legacy) |
| Max per team | 2 keys | Unlimited certs |

**Use Key auth (.p8) for all new projects.** Certificate auth is legacy and requires annual renewal, which is easy to forget and causes notification outages.

---

## Token Management

### Registration Flow

```swift
// 1. App requests permission (see SKILL.md SOP-1)
// 2. App registers for remote notifications
UIApplication.shared.registerForRemoteNotifications()

// 3. System calls delegate with token (or error)
func application(
    _ application: UIApplication,
    didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
) {
    let token = deviceToken.map { String(format: "%02x", $0) }.joined()
    // 4. Send token to your server
    Task { await registerToken(token) }
}
```

### Token Lifecycle

```
App installed -> first registerForRemoteNotifications() -> NEW token
App updated   -> registerForRemoteNotifications()       -> SAME or NEW token
OS updated    -> registerForRemoteNotifications()       -> SAME or NEW token
Backup restore on new device -> NEW token
App reinstalled -> NEW token
```

**Critical rules:**
- Tokens CAN change between app launches. Always send the current token to your server on every app launch.
- Old tokens are not immediately invalidated. APNs will eventually return `410 Gone` for expired tokens.
- Store tokens server-side with: `user_id`, `token`, `platform` (ios/android), `created_at`, `updated_at`
- When APNs returns `410`, soft-delete the token record

### Multi-Device Handling

A user may have multiple devices. Each device has its own token.

```
User "Alice":
  - iPhone token: abc123...
  - iPad token: def456...

When sending a notification to Alice:
  -> Send to ALL of Alice's active tokens
  -> Each device receives the notification independently
```

---

## Silent Push

Silent push wakes the app in the background without showing a visible notification. The app gets ~30 seconds of execution time.

### Payload

```json
{
    "aps": {
        "content-available": 1
    },
    "type": "data_sync",
    "sync_id": "abc123"
}
```

**No `alert`, `sound`, or `badge`** -- any visible element makes it a regular push.

### Limitations

- **Throttled by iOS**: Apple limits silent push frequency per app. The exact limit is undocumented but is approximately 2-3 per hour.
- **Not guaranteed to wake the app**: If the device is in Low Power Mode, the app has been force-quit by the user, or the system is under resource pressure, silent push may be ignored.
- **30-second execution window**: Your background work must complete within ~30 seconds or iOS terminates the task.
- **No user interaction**: Cannot show alerts, play sounds, or update badges.

### When to Use

- Background data sync (new messages, updated content)
- Cache invalidation (tell the app to fetch fresh data)
- State synchronization (update local state to match server)

### When NOT to Use

- Time-critical delivery (use visible push instead)
- Large data downloads (use BGProcessingTask)
- Anything that requires guaranteed delivery

---

## Notification Service Extension

A Notification Service Extension lets you modify push content before it's displayed. You have ~30 seconds to process.

### Use Cases

- Download and attach images/media to the notification
- Decrypt end-to-end encrypted notification content
- Modify the notification text (e.g., add sender name from local database)
- Track notification delivery (analytics)

### Setup

1. File > New > Target > Notification Service Extension
2. Name it (e.g., `NotificationService`)
3. Add to your provisioning profile (Match handles this if the bundle ID is in the Matchfile)

```swift
// NotificationService/NotificationService.swift
import UserNotifications

final class NotificationService: UNNotificationServiceExtension {

    private var contentHandler: ((UNNotificationContent) -> Void)?
    private var bestAttemptContent: UNMutableNotificationContent?

    override func didReceive(
        _ request: UNNotificationRequest,
        withContentHandler contentHandler: @escaping (UNNotificationContent) -> Void
    ) {
        self.contentHandler = contentHandler
        bestAttemptContent = (request.content.mutableCopy() as? UNMutableNotificationContent)

        guard let bestAttemptContent else {
            contentHandler(request.content)
            return
        }

        // Example: Download and attach an image
        if let imageURLString = bestAttemptContent.userInfo["image_url"] as? String,
           let imageURL = URL(string: imageURLString) {
            downloadImage(from: imageURL) { attachment in
                if let attachment {
                    bestAttemptContent.attachments = [attachment]
                }
                contentHandler(bestAttemptContent)
            }
        } else {
            contentHandler(bestAttemptContent)
        }
    }

    override func serviceExtensionTimeWillExpire() {
        // Called just before the extension is terminated (30s limit)
        // Deliver the best attempt content
        if let contentHandler, let bestAttemptContent {
            contentHandler(bestAttemptContent)
        }
    }

    private func downloadImage(
        from url: URL,
        completion: @escaping (UNNotificationAttachment?) -> Void
    ) {
        URLSession.shared.downloadTask(with: url) { localURL, _, error in
            guard let localURL, error == nil else {
                completion(nil)
                return
            }

            // Move to a location with a file extension (required for attachments)
            let tempDir = FileManager.default.temporaryDirectory
            let fileURL = tempDir.appendingPathComponent(UUID().uuidString + ".jpg")

            try? FileManager.default.moveItem(at: localURL, to: fileURL)

            let attachment = try? UNNotificationAttachment(
                identifier: UUID().uuidString,
                url: fileURL,
                options: nil
            )

            completion(attachment)
        }.resume()
    }
}
```

### Push Payload for Service Extension

The push payload must include `mutable-content: 1` to trigger the extension:

```json
{
    "aps": {
        "alert": {
            "title": "New photo from Alice",
            "body": "Tap to view"
        },
        "mutable-content": 1,
        "sound": "default"
    },
    "image_url": "https://example.com/photo.jpg"
}
```

---

## Notification Content Extension

A Notification Content Extension provides a custom UI for expanded notifications (3D Touch or long-press).

### Use Cases

- Display a map for location-based notifications
- Show a custom transaction receipt layout
- Display an interactive chart or media player

This is less commonly needed than the Service Extension. Only implement if the default notification UI is insufficient for your content.

---

## Multi-Environment Setup

### Sandbox vs Production APNs

| Environment | Server | When Used |
|---|---|---|
| Sandbox | `api.sandbox.push.apple.com` | Development builds, TestFlight builds |
| Production | `api.push.apple.com` | App Store builds |

**How iOS determines which environment:**
- Built with Development provisioning profile -> Sandbox
- Built with Distribution provisioning profile (TestFlight) -> Sandbox
- Downloaded from App Store -> Production

**Your server must handle both:**
1. Store the environment alongside the device token
2. Send to the correct APNs endpoint based on the token's environment
3. When you get a `400 BadDeviceToken` error, the token may be for the wrong environment

### Common Multi-Environment Pitfall

TestFlight builds use the sandbox APNs environment, not production. If your server only sends to production, TestFlight users won't receive push notifications. Always test push notifications on TestFlight before releasing to the App Store.

```
Development build -> Sandbox APNs -> api.sandbox.push.apple.com
TestFlight build  -> Sandbox APNs -> api.sandbox.push.apple.com  (!)
App Store build   -> Production APNs -> api.push.apple.com
```

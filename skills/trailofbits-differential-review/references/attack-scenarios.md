# iOS Attack Scenarios

Step-by-step exploitation scenarios for iOS-specific vulnerabilities. Use these during adversarial analysis (Phase 5 of differential review) to model realistic attacks.

---

## Scenario 1: Stealing Auth Tokens via Pasteboard

### Context
App copies authentication tokens or session identifiers to the system pasteboard (e.g., "Copy Account ID" feature that includes the session token).

### Attacker Goal
Steal the user's authentication token to impersonate them.

### Prerequisites
- Target app puts sensitive data on `UIPasteboard.general`
- Attacker's app is installed on the same device
- iOS < 16 (no paste permission prompt) OR user grants paste permission

### Attack Steps
1. User copies data from the target app (e.g., sharing an account number that also copies a session token)
2. User switches to the attacker's app (or the attacker's app is a keyboard extension)
3. Attacker's app reads `UIPasteboard.general.string` in `applicationDidBecomeActive`
4. Attacker's app sends the token to the attacker's server
5. Attacker uses the token to make authenticated API requests as the user

### Impact
- Full account takeover
- Financial transactions on behalf of the user
- Access to personal data

### Detection Method
- Search for `UIPasteboard.general` write operations
- Check what data is placed on the pasteboard
- Check if pasteboard items include expiration or `localOnly` options

### Prevention
```swift
// Use local-only pasteboard with expiration
let pasteboard = UIPasteboard(name: UIPasteboard.Name("com.app.local"), create: true)
pasteboard?.setItems([
    [UIPasteboard.typeAutomatic: accountNumber]
], options: [
    .localOnly: true,
    .expirationDate: Date().addingTimeInterval(60) // Expires in 60 seconds
])

// Or better: don't put sensitive data on pasteboard at all
// Use share sheet with custom activities instead
```

---

## Scenario 2: Bypassing Biometric Authentication

### Context
App uses `LAContext.evaluatePolicy` for authentication but relies solely on the boolean result without tying it to a Keychain-stored secret.

### Attacker Goal
Bypass biometric authentication to access the app's protected features.

### Prerequisites
- Target app uses `LAContext` without Keychain-backed authentication
- Device is jailbroken (attacker has Frida installed)
- OR attacker has physical access to unlocked device with passcode known

### Attack Steps (Frida/Jailbreak Path)
1. Attacker installs Frida on the jailbroken device
2. Attacker writes a Frida script that hooks `LAContext.evaluatePolicy`:
   ```javascript
   // Frida script
   var LAContext = ObjC.classes.LAContext;
   Interceptor.attach(LAContext['- evaluatePolicy:localizedReason:reply:'].implementation, {
     onEnter: function(args) {
       // Replace the reply block to always return success
       var reply = new ObjC.Block(args[4]);
       reply.implementation = function(success, error) {
         reply(true, null); // Always succeed
       };
     }
   });
   ```
3. Biometric prompt appears but any input (or cancellation + re-hook) returns `true`
4. App grants access based on the boolean result
5. Attacker accesses all protected features

### Attack Steps (Passcode Fallback Path)
1. Attacker knows the device passcode (social engineering, shoulder surfing)
2. App uses `.deviceOwnerAuthentication` policy (allows passcode fallback)
3. Biometric prompt appears, attacker taps "Enter Passcode"
4. Attacker enters the known passcode
5. App grants access -- biometric "security" completely bypassed

### Impact
- Access to protected features (financial data, personal information)
- Ability to perform transactions
- No audit trail of the bypass

### Detection Method
- Check which `LAPolicy` is used (`.deviceOwnerAuthenticationWithBiometrics` vs `.deviceOwnerAuthentication`)
- Check if biometric auth result is tied to Keychain access
- Check for anti-tampering/jailbreak detection

### Prevention
```swift
// Tie biometric auth to Keychain-stored secret
// The secret itself is the proof of auth, not the boolean

// 1. Store a secret with biometric protection
func storeBiometricSecret(_ secret: Data) throws {
    let access = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        .biometryCurrentSet, // Invalidated if biometrics change
        nil
    )

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: "com.app.biometric-auth",
        kSecAttrAccount as String: "auth-secret",
        kSecValueData as String: secret,
        kSecAttrAccessControl as String: access as Any
    ]

    SecItemAdd(query as CFDictionary, nil)
}

// 2. Authenticate by retrieving the secret (requires biometric)
func authenticateWithBiometric() throws -> Data {
    let context = LAContext()
    context.localizedReason = "Authenticate to access your account"

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: "com.app.biometric-auth",
        kSecAttrAccount as String: "auth-secret",
        kSecReturnData as String: true,
        kSecUseAuthenticationContext as String: context
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    // Frida can't bypass this -- the Keychain hardware won't release
    // the secret without genuine biometric authentication
    guard status == errSecSuccess, let data = result as? Data else {
        throw AuthError.biometricFailed
    }
    return data
}
```

---

## Scenario 3: Hijacking Deep Links to Steal OAuth Tokens

### Context
App uses a custom URL scheme (`myapp://`) for OAuth callback redirect. OAuth flow redirects to `myapp://callback?token=abc123`.

### Attacker Goal
Intercept the OAuth callback to steal the authentication token.

### Prerequisites
- Target app uses custom URL scheme (not Universal Links) for OAuth redirect
- Attacker can install a malicious app on the user's device (via TestFlight, enterprise profile, or App Store)

### Attack Steps
1. Attacker creates a malicious app that registers the same URL scheme (`myapp://`)
2. User installs the malicious app (appears as a different app entirely)
3. User initiates OAuth login in the target app
4. Browser redirects to `myapp://callback?token=abc123`
5. iOS shows disambiguation dialog (or on older iOS, may route to attacker's app directly)
6. If user taps attacker's app (or if iOS routes there by default), attacker captures the token
7. Attacker uses the token to access the user's account

### Impact
- Full account takeover via stolen OAuth token
- Access to all user data and actions

### Detection Method
- Check Info.plist for `CFBundleURLTypes` (custom URL schemes)
- Check if OAuth callbacks use custom URL schemes or Universal Links
- Check if URL handler validates the source application

### Prevention
```swift
// Use Universal Links instead of custom URL schemes for OAuth
// Universal Links are verified via apple-app-site-association file
// and cannot be hijacked by other apps

// 1. Set up apple-app-site-association on your server:
// {
//   "applinks": {
//     "apps": [],
//     "details": [{
//       "appID": "TEAM_ID.com.company.app",
//       "paths": ["/oauth/callback"]
//     }]
//   }
// }

// 2. Handle Universal Link in app
func application(_ application: UIApplication,
                 continue userActivity: NSUserActivity,
                 restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL,
          url.host == "app.company.com",
          url.path == "/oauth/callback" else {
        return false
    }

    // Also verify state parameter to prevent CSRF
    guard let components = URLComponents(url: url, resolvingAgainstBaseURL: true),
          let state = components.queryItems?.first(where: { $0.name == "state" })?.value,
          state == storedOAuthState else {
        logger.warning("oauth_state_mismatch")
        return false
    }

    handleOAuthCallback(url)
    return true
}
```

---

## Scenario 4: Extracting Secrets from Insecure Storage

### Context
App stores authentication tokens in UserDefaults instead of Keychain.

### Attacker Goal
Extract authentication tokens to impersonate the user.

### Prerequisites
- Target app stores sensitive data in UserDefaults, plist files, or unencrypted databases
- Attacker has physical access to the device (or access to a device backup)

### Attack Steps (Backup Extraction)
1. Attacker creates an iTunes/Finder backup of the device (encrypted backups can be decrypted with passcode)
2. Attacker uses a backup extraction tool (iExplorer, libimobiledevice, iBackupBot)
3. Attacker navigates to `Library/Preferences/com.company.app.plist`
4. Attacker reads the plist file (plaintext XML) and extracts the auth token
5. Attacker uses the token to access the API from another device

### Attack Steps (Jailbroken Device)
1. Attacker has a jailbroken device (or jailbreaks the target device)
2. Attacker accesses the file system via SSH or file browser
3. Attacker reads UserDefaults plist at `/var/mobile/Containers/Data/Application/{UUID}/Library/Preferences/`
4. All UserDefaults data is plaintext readable

### Impact
- Account takeover
- Access to personal and financial data
- Ability to perform transactions

### Detection Method
- Search for `UserDefaults.standard.set` with sensitive key names (token, password, secret, key, session)
- Check if `NSUbiquitousKeyValueStore` is used for sensitive data (syncs to iCloud in plaintext)
- Check if files contain sensitive data without `isExcludedFromBackup`

### Prevention
- Store all sensitive data in Keychain with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`
- Set `isExcludedFromBackup = true` on any files containing sensitive data
- Use encrypted Core Data (SQLCipher) for sensitive structured data

---

## Scenario 5: Man-in-the-Middle on Unpinned Connection

### Context
App communicates with API over HTTPS but does not implement certificate pinning.

### Attacker Goal
Intercept and modify API traffic to steal data or manipulate app behavior.

### Prerequisites
- Target app does not implement certificate pinning
- Attacker controls the network (public WiFi, compromised router, ARP spoofing)

### Attack Steps
1. Attacker sets up a rogue WiFi access point (e.g., "Free Coffee Shop WiFi")
2. User connects to the rogue WiFi
3. Attacker runs mitmproxy or Charles Proxy on the access point
4. Attacker installs a custom CA certificate profile on the proxy
5. All HTTPS traffic from the app is intercepted and decrypted by the proxy
6. Attacker reads API responses (account balances, personal data, transaction history)
7. Attacker modifies API requests (changes transfer amounts, redirects payments)

### Impact
- Complete visibility into all API traffic
- Ability to modify requests and responses
- Data theft, financial manipulation

### Detection Method
- Check for `URLSessionDelegate` with `didReceive challenge` handler
- Check if certificate/public key pinning is implemented
- Check ATS configuration for exceptions

### Prevention
```swift
// Implement public key pinning (see cipher-security references for full implementation)
// Key points:
// 1. Pin public keys, not certificates (survives cert rotation)
// 2. Always have a backup pin
// 3. Implement pin failure reporting
// 4. Have an emergency disable mechanism
```

---

## Scenario 6: Data Theft via App Extension

### Context
App shares data with a widget or share extension via app group container. Sensitive data is stored in the shared container without encryption.

### Attacker Goal
Access sensitive data through a compromised or malicious extension.

### Prerequisites
- App uses app group shared container for data sharing
- Sensitive data is stored in shared container without encryption
- Attacker has compromised the extension (or built a malicious extension that shares the app group)

### Attack Steps
1. Attacker identifies the app group identifier (visible in entitlements)
2. Attacker creates a malicious app that declares the same app group
3. Note: Same app group requires same team ID, so this attack vector is primarily about extension compromise or developer account compromise
4. If extension is compromised (via supply chain attack on extension dependency), it reads all shared container data
5. Attacker exfiltrates the data via the extension's network access

### Impact
- Access to all data in shared container
- Potentially includes auth tokens, user data, cached content

### Detection Method
- Check app group container for sensitive data storage
- Check if shared data is encrypted
- Check extension dependencies for known vulnerabilities

### Prevention
- Encrypt all sensitive data in shared containers
- Minimize what's shared (only what the extension needs)
- Validate data integrity when reading from shared containers
- Use Keychain sharing (which has its own access control) instead of file-based sharing for credentials

---

## Using These Scenarios in Review

During Phase 5 (Adversarial Analysis) of a differential review:

1. **Identify which scenarios apply** to the changed code
2. **Walk through the attack steps** against the new code
3. **Check if the change introduces or removes protections** against each applicable scenario
4. **Rate exploitability**: EASY (automated tools), MEDIUM (requires specific setup), HARD (requires physical access + jailbreak)
5. **Document in the report** with specific line references to the diff

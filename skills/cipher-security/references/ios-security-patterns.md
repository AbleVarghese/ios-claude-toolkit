# iOS Security Patterns Reference

Implementation patterns for iOS-specific security features. Each pattern includes the correct implementation, common mistakes, and attacker bypass techniques.

---

## Keychain Wrapper Patterns

### Storing Authentication Tokens

```swift
import Security

enum KeychainError: Error {
    case duplicateEntry
    case itemNotFound
    case unexpectedData
    case unhandledError(status: OSStatus)
}

struct KeychainService {
    private let service: String
    private let accessGroup: String?

    init(service: String, accessGroup: String? = nil) {
        self.service = service
        self.accessGroup = accessGroup
    }

    // Store a token securely
    func store(token: Data, for account: String) throws {
        var query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecValueData as String: token,
            // CRITICAL: Use the most restrictive accessibility level
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        // Add access group for keychain sharing between apps
        if let accessGroup = accessGroup {
            query[kSecAttrAccessGroup as String] = accessGroup
        }

        let status = SecItemAdd(query as CFDictionary, nil)

        if status == errSecDuplicateItem {
            // Update existing item instead of failing
            let updateQuery: [String: Any] = [
                kSecClass as String: kSecClassGenericPassword,
                kSecAttrService as String: service,
                kSecAttrAccount as String: account
            ]
            let attributes: [String: Any] = [
                kSecValueData as String: token
            ]
            let updateStatus = SecItemUpdate(updateQuery as CFDictionary, attributes as CFDictionary)
            guard updateStatus == errSecSuccess else {
                throw KeychainError.unhandledError(status: updateStatus)
            }
        } else if status != errSecSuccess {
            throw KeychainError.unhandledError(status: status)
        }
    }

    // Retrieve a stored token
    func retrieve(for account: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess else {
            if status == errSecItemNotFound {
                throw KeychainError.itemNotFound
            }
            throw KeychainError.unhandledError(status: status)
        }

        guard let data = result as? Data else {
            throw KeychainError.unexpectedData
        }

        return data
    }

    // Delete a stored token
    func delete(for account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account
        ]

        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unhandledError(status: status)
        }
    }
}
```

### Keychain Accessibility Levels (Choose Carefully)

| Level | When Accessible | Survives Restore | Use Case |
|-------|----------------|------------------|----------|
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | When unlocked, this device only | NO | **Recommended default** for auth tokens, session data |
| `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | After first unlock, this device only | NO | Background operations needing credentials |
| `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` | When unlocked + passcode set | NO | Highest security, deleted if passcode removed |
| `kSecAttrAccessibleWhenUnlocked` | When unlocked, any device | YES | Data that should sync via backup/restore |
| `kSecAttrAccessibleAfterFirstUnlock` | After first unlock, any device | YES | Background + sync needs |
| `kSecAttrAccessibleAlways` | **ALWAYS** | YES | **NEVER USE** -- accessible even when locked |

### Biometric-Protected Keychain Items

```swift
// Store an item that requires biometric authentication to access
func storeBiometricProtected(token: Data, for account: String) throws {
    let access = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        .biometryCurrentSet, // Invalidated if biometrics change
        nil
    )

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: service,
        kSecAttrAccount as String: account,
        kSecValueData as String: token,
        kSecAttrAccessControl as String: access as Any,
        kSecUseAuthenticationContext as String: LAContext() // Reuse context for multiple operations
    ]

    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else {
        throw KeychainError.unhandledError(status: status)
    }
}
```

### Keychain Sharing Between Apps

To share keychain items between apps in the same developer account:
1. Enable Keychain Sharing capability in both apps
2. Use the same access group identifier
3. Both apps must be signed by the same team

```swift
// App 1: Store shared credential
let sharedKeychain = KeychainService(
    service: "com.company.shared-auth",
    accessGroup: "TEAM_ID.com.company.shared"
)
try sharedKeychain.store(token: tokenData, for: "user-token")

// App 2: Read shared credential
let sharedKeychain = KeychainService(
    service: "com.company.shared-auth",
    accessGroup: "TEAM_ID.com.company.shared"
)
let token = try sharedKeychain.retrieve(for: "user-token")
```

---

## Jailbreak Detection

### Multiple Detection Methods

No single method is sufficient. Use multiple checks and combine results.

```swift
struct JailbreakDetector {
    // Combine multiple signals for higher confidence
    func isDeviceCompromised() -> Bool {
        return checkFileSystemArtifacts()
            || checkSandboxIntegrity()
            || checkDynamicLibraries()
            || checkSymbolicLinks()
            || checkWritableSystemPaths()
    }

    // Method 1: Check for jailbreak-related files
    private func checkFileSystemArtifacts() -> Bool {
        let suspiciousPaths = [
            "/Applications/Cydia.app",
            "/Applications/Sileo.app",
            "/Library/MobileSubstrate/MobileSubstrate.dylib",
            "/usr/sbin/sshd",
            "/usr/bin/sshd",
            "/usr/libexec/sftp-server",
            "/etc/apt",
            "/private/var/lib/apt",
            "/private/var/lib/cydia",
            "/private/var/stash",
            "/var/lib/dpkg/info"
        ]

        for path in suspiciousPaths {
            if FileManager.default.fileExists(atPath: path) {
                return true
            }
        }
        return false
    }

    // Method 2: Check sandbox integrity
    private func checkSandboxIntegrity() -> Bool {
        // On non-jailbroken devices, fork() returns -1
        // This method may be flagged by App Review -- use with caution
        let canFork = canOpenRestrictedPath("/bin/sh")
        return canFork
    }

    private func canOpenRestrictedPath(_ path: String) -> Bool {
        let file = fopen(path, "r")
        if file != nil {
            fclose(file)
            return true // Should NOT be accessible on non-jailbroken devices
        }
        return false
    }

    // Method 3: Check for injected dynamic libraries
    private func checkDynamicLibraries() -> Bool {
        let suspiciousLibs = [
            "SubstrateLoader.dylib",
            "SSLKillSwitch2.dylib",
            "FridaGadget.dylib",
            "frida-agent.dylib",
            "libcycript.dylib"
        ]

        for i in 0..<_dyld_image_count() {
            guard let imageName = _dyld_get_image_name(i) else { continue }
            let name = String(cString: imageName)
            for lib in suspiciousLibs {
                if name.contains(lib) {
                    return true
                }
            }
        }
        return false
    }

    // Method 4: Check for symbolic links (jailbreaks often symlink system dirs)
    private func checkSymbolicLinks() -> Bool {
        let pathsToCheck = ["/Applications", "/var/stash", "/Library/Ringtones"]
        for path in pathsToCheck {
            var s = stat()
            if lstat(path, &s) == 0 {
                if (s.st_mode & S_IFLNK) == S_IFLNK {
                    return true
                }
            }
        }
        return false
    }

    // Method 5: Check if we can write outside sandbox
    private func checkWritableSystemPaths() -> Bool {
        let testPath = "/private/jailbreak-test-\(UUID().uuidString)"
        do {
            try "test".write(toFile: testPath, atomically: true, encoding: .utf8)
            try FileManager.default.removeItem(atPath: testPath)
            return true // Should NOT be writable on non-jailbroken devices
        } catch {
            return false
        }
    }
}
```

### How Attackers Bypass Each Method

| Method | Bypass Technique | Mitigation |
|--------|-----------------|------------|
| File checks | Rename/move jailbreak files | Check many paths, use stat() not just fileExists |
| Sandbox checks | Hook sandbox check functions | Combine with other methods, use inline checks |
| Dylib detection | Rename injected libraries | Check for known patterns, not just names |
| Symbolic links | Remove symlinks temporarily | Check during runtime, not just at launch |
| Write tests | Restrict write in jailbreak config | Use multiple detection methods |

### When to Use Jailbreak Detection

- **Financial apps**: Required by compliance (PCI DSS, banking regulations)
- **DRM-protected content**: Prevents content extraction
- **Enterprise apps**: Enforces device security policy
- **High-security apps**: Additional defense layer

### When NOT to Use Jailbreak Detection

- Apple may reject apps that detect jailbreaking in certain ways (fork, private APIs)
- Some enterprise users have legitimate reasons for jailbroken devices
- False positives frustrate users -- always provide an appeal path

---

## Certificate Pinning

### Implementation with URLSession

```swift
final class PinnedURLSessionDelegate: NSObject, URLSessionDelegate {
    // Store the SHA-256 hash of the server's public key
    // Generate with: openssl x509 -in cert.pem -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl enc -base64
    private let pinnedPublicKeyHashes: Set<String> = [
        "BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=", // Primary cert
        "CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC="  // Backup cert (for rotation)
    ]

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // Evaluate the server's certificate chain
        let policies = [SecPolicyCreateSSL(true, challenge.protectionSpace.host as CFString)]
        SecTrustSetPolicies(serverTrust, policies as CFTypeRef)

        var error: CFError?
        guard SecTrustEvaluateWithError(serverTrust, &error) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // Extract and verify the public key hash
        guard let serverCertificate = SecTrustGetCertificateAtIndex(serverTrust, 0),
              let serverPublicKey = SecCertificateCopyKey(serverCertificate),
              let serverPublicKeyData = SecKeyCopyExternalRepresentation(serverPublicKey, nil) as Data? else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let hash = SHA256.hash(data: serverPublicKeyData)
        let hashString = Data(hash).base64EncodedString()

        if pinnedPublicKeyHashes.contains(hashString) {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            // Pin mismatch -- possible MITM attack
            logger.error("certificate_pin_mismatch", [
                "host": challenge.protectionSpace.host,
                "received_hash": hashString
            ])
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

### Pin Rotation Strategy

1. Always pin at least 2 certificates (primary + backup)
2. Rotate pins via app update BEFORE certificate rotation
3. Include the new certificate's pin in the app update
4. Wait for adoption (e.g., 90%+ of users on new version)
5. Rotate the server certificate
6. Remove the old pin in the next app update

### Emergency Pin Disable

For cases where a pin mismatch would lock out all users:

```swift
// Feature flag to disable pinning in emergencies
// This should be controlled by a server-side feature flag service
// that itself uses a different pinned connection or is hardcoded
struct PinningConfiguration {
    let isPinningEnabled: Bool
    let pinnedHashes: Set<String>
    let emergencyBypassToken: String? // Time-limited, server-issued

    static func load() -> PinningConfiguration {
        // Load from secure, pinned configuration endpoint
        // Fallback to bundled configuration if endpoint unreachable
        // ...
    }
}
```

---

## Biometric Authentication

### LAContext Usage

```swift
import LocalAuthentication

actor BiometricAuthService {
    enum BiometricError: Error {
        case notAvailable
        case notEnrolled
        case authenticationFailed
        case userCancelled
        case systemCancelled
        case passcodeNotSet
    }

    func authenticate(reason: String) async throws -> Bool {
        let context = LAContext()
        var error: NSError?

        // Check availability BEFORE prompting
        guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
            if let error = error {
                switch error.code {
                case LAError.biometryNotAvailable.rawValue:
                    throw BiometricError.notAvailable
                case LAError.biometryNotEnrolled.rawValue:
                    throw BiometricError.notEnrolled
                case LAError.passcodeNotSet.rawValue:
                    throw BiometricError.passcodeNotSet
                default:
                    throw BiometricError.authenticationFailed
                }
            }
            throw BiometricError.notAvailable
        }

        // SECURITY CRITICAL: Use .deviceOwnerAuthenticationWithBiometrics
        // NOT .deviceOwnerAuthentication (which allows passcode fallback)
        // unless passcode fallback is an explicit product requirement
        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            )
            return success
        } catch let error as LAError {
            switch error.code {
            case .userCancel:
                throw BiometricError.userCancelled
            case .systemCancel:
                throw BiometricError.systemCancelled
            default:
                throw BiometricError.authenticationFailed
            }
        }
    }
}
```

### Fallback Handling

```swift
// When biometric auth fails, provide a secure fallback
func authenticateUser() async throws {
    do {
        let biometricSuccess = try await biometricAuth.authenticate(
            reason: "Authenticate to view your balance"
        )
        if biometricSuccess {
            return // Authenticated
        }
    } catch BiometricAuthService.BiometricError.notAvailable,
            BiometricAuthService.BiometricError.notEnrolled {
        // Biometrics not available -- fall back to PIN/password
        // Do NOT fall back to device passcode without explicit product decision
    } catch BiometricAuthService.BiometricError.userCancelled {
        // User cancelled -- do not fall back, respect their decision
        throw AuthError.cancelled
    } catch {
        // Authentication failed -- allow retry, not fallback
        throw AuthError.biometricFailed
    }

    // Secure fallback: app-specific PIN or password
    // NOT device passcode (which may be a simple 4-digit code)
    try await promptForAppPIN()
}
```

### Replay Attack Prevention

Biometric auth result (`true`/`false`) is just a boolean. An attacker with runtime access can hook `evaluatePolicy` and always return `true`. To prevent this:

1. **Tie biometric auth to Keychain access**: Store a secret in a biometric-protected Keychain item. Successful biometric auth unlocks the item -- the secret itself is the proof of auth, not the boolean.
2. **Server-side verification**: After biometric auth, use the Keychain secret to sign a challenge from the server.
3. **Time-bound tokens**: Biometric auth should produce a time-limited token, not a permanent "authenticated" state.

---

## App Transport Security

### When ATS Exceptions Are Justified

| Exception | Justification | Risk |
|-----------|--------------|------|
| `NSAllowsArbitraryLoads` | **NEVER** in production | All traffic unprotected |
| `NSExceptionAllowsInsecureHTTPLoads` for specific domain | Legacy API without TLS, migration planned | Only that domain unprotected |
| `NSExceptionMinimumTLSVersion` | Server only supports TLS 1.0/1.1 | Weaker encryption, migration planned |
| `NSAllowsLocalNetworking` | Communication with local hardware (printers, IoT) | Only local network affected |
| `NSAllowsArbitraryLoadsInWebContent` | WebView loading third-party content | Only WebView traffic affected |

### How to Audit ATS Configuration

```bash
# Extract ATS configuration from Info.plist
/usr/libexec/PlistBuddy -c "Print :NSAppTransportSecurity" Info.plist

# Check for overly permissive settings
# RED FLAGS:
# - NSAllowsArbitraryLoads = YES
# - Exception domains with NSExceptionAllowsInsecureHTTPLoads = YES
# - NSTemporaryExceptionMinimumTLSVersion < TLSv1.2
```

---

## Secure Data Erasure

### Zeroing Memory Buffers

```swift
// After using sensitive data, zero the memory
func processSecretKey(_ key: inout [UInt8]) {
    defer {
        // Zero the buffer after use
        key.withUnsafeMutableBufferPointer { buffer in
            memset_s(buffer.baseAddress, buffer.count, 0, buffer.count)
        }
    }

    // Use the key...
    performCryptoOperation(key)
}

// For Data objects
func processToken(_ token: inout Data) {
    defer {
        token.resetBytes(in: 0..<token.count)
    }

    // Use the token...
    authenticateWith(token)
}
```

### Secure Key Deletion

```swift
// When a user logs out, delete all sensitive keychain items
func secureLogout() throws {
    // Delete all items for this service
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: serviceName
    ]
    let status = SecItemDelete(query as CFDictionary)
    guard status == errSecSuccess || status == errSecItemNotFound else {
        throw KeychainError.unhandledError(status: status)
    }

    // Also clear any in-memory caches of sensitive data
    tokenCache = nil
    sessionData = nil

    // Clear pasteboard if it might contain sensitive data
    UIPasteboard.general.items = []

    // Clear URL cache
    URLCache.shared.removeAllCachedResponses()

    // Clear cookies for authenticated sessions
    if let cookies = HTTPCookieStorage.shared.cookies {
        for cookie in cookies {
            HTTPCookieStorage.shared.deleteCookie(cookie)
        }
    }
}
```

---

## Runtime Protection

### Anti-Debugging

```swift
// Detect if a debugger is attached
func isDebuggerAttached() -> Bool {
    var info = kinfo_proc()
    var size = MemoryLayout<kinfo_proc>.stride
    var mib: [Int32] = [CTL_KERN, KERN_PROC, KERN_PROC_PID, getpid()]

    let result = sysctl(&mib, UInt32(mib.count), &info, &size, nil, 0)
    guard result == 0 else { return false }

    return (info.kp_proc.p_flag & P_TRACED) != 0
}

// Usage: check at security-critical points
func performSensitiveOperation() {
    #if !DEBUG
    if isDebuggerAttached() {
        // Log the attempt and refuse to continue
        logger.error("debugger_detected_during_sensitive_op")
        fatalError("Security violation detected")
    }
    #endif
}
```

### Integrity Verification

```swift
// Verify the app binary hasn't been modified
func verifyAppIntegrity() -> Bool {
    // Check that the app is signed with the expected team ID
    guard let bundleInfo = Bundle.main.infoDictionary,
          let executablePath = Bundle.main.executablePath else {
        return false
    }

    // Verify code signature
    var staticCode: SecStaticCode?
    let url = URL(fileURLWithPath: executablePath)
    guard SecStaticCodeCreateWithPath(url as CFURL, [], &staticCode) == errSecSuccess,
          let code = staticCode else {
        return false
    }

    let status = SecStaticCodeCheckValidity(code, [], nil)
    return status == errSecSuccess
}
```

### Code Obfuscation Considerations

- Swift code is harder to reverse engineer than Objective-C (no runtime metadata exposure)
- String constants are still visible in the binary -- encrypt sensitive strings
- Class names and method names may be visible -- avoid naming methods `validatePassword` or `checkLicense`
- Consider commercial obfuscation tools for high-value targets (financial apps, DRM)
- Obfuscation is defense-in-depth, not a security boundary -- a determined attacker will reverse engineer regardless

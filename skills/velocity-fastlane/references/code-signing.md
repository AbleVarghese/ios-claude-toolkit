# Code Signing Deep Dive

Reference guide for iOS code signing: certificates, provisioning profiles, Match automation, and troubleshooting.

---

## Certificate Types

| Certificate | Purpose | Issued To | Max Per Team | Validity |
|---|---|---|---|---|
| **Apple Development** | Build and run on devices during development | Individual developer | Unlimited | 1 year |
| **Apple Distribution** | Distribute via App Store, TestFlight, Ad Hoc | Team (shared) | 3 | 1 year |
| **Apple Push Notification (APNs)** | Send push notifications to your app | Per App ID | 2 (sandbox + production) | 1 year |

**Key distinction:**
- Development certificates are per-developer. Each developer has their own.
- Distribution certificates are per-team. The team shares a limited number (max 3).
- This is why manual management breaks down: developers overwrite each other's certificates.

---

## Provisioning Profile Types

| Profile Type | Signing Certificate | Devices | Distribution | Use Case |
|---|---|---|---|---|
| **Development** | Apple Development | Registered devices only | Direct install | Running on test devices |
| **Ad Hoc** | Apple Distribution | Registered devices only (max 100) | Direct install | Beta testing without TestFlight |
| **App Store** | Apple Distribution | Any device (via App Store/TestFlight) | App Store Connect | TestFlight and App Store distribution |
| **Enterprise (In-House)** | Apple Distribution (Enterprise) | Any device | Direct install | Company-internal apps ($299/yr program) |

**Decision guide:**
- "I need to run on my device during development" --> Development profile
- "I need to send a build to specific testers" --> Ad Hoc profile (limited) or App Store/TestFlight (unlimited)
- "I need to send to TestFlight" --> App Store profile
- "I need to publish to the App Store" --> App Store profile
- "I need to distribute within my company" --> Enterprise profile

---

## Match Setup Guide

Match stores code signing identities in an encrypted git repository, ensuring every team member uses the same certificates and profiles.

### Initial Setup

```bash
# 1. Create a private git repo for certificates
# Example: github.com/your-org/ios-certificates (private)

# 2. Initialize Match
fastlane match init

# 3. Configure the Matchfile
```

```ruby
# fastlane/Matchfile
git_url("https://github.com/your-org/ios-certificates.git")
storage_mode("git")

type("development")  # Default type

app_identifier(["com.yourapp.ios"])  # Array: supports multiple app IDs
username("developer@yourcompany.com")

# For CI: use token-based auth instead of username
# git_basic_authorization(ENV["MATCH_GIT_BASIC_AUTHORIZATION"])

# Team ID (if you belong to multiple teams)
# team_id("ABCD1234EF")
```

```bash
# 4. Generate certificates for each type
fastlane match development
fastlane match appstore

# 5. Share the MATCH_PASSWORD with the team (securely!)
# Every team member needs this to decrypt the certificates
```

### Multi-Target Setup

```ruby
# Matchfile for apps with extensions, widgets, etc.
app_identifier([
    "com.yourapp.ios",           # Main app
    "com.yourapp.ios.widgets",   # Widget extension
    "com.yourapp.ios.share",     # Share extension
    "com.yourapp.ios.notification-service" # Notification service extension
])
```

### Multi-Team Signing

```ruby
# When you work on apps under different Apple Developer teams

# Option 1: Separate Matchfiles per team
# fastlane/Matchfile.teamA, fastlane/Matchfile.teamB

# Option 2: Use environment variables
git_url(ENV["MATCH_GIT_URL"])
team_id(ENV["TEAM_ID"])
app_identifier([ENV["BUNDLE_ID"]])

# Usage:
# MATCH_GIT_URL=... TEAM_ID=... BUNDLE_ID=... fastlane match development
```

### CI Configuration

```bash
# In GitHub Actions / CI environment:
# 1. Store these as encrypted secrets:
#    MATCH_PASSWORD - decryption password for the certificate repo
#    MATCH_GIT_BASIC_AUTHORIZATION - base64(username:personal_access_token)

# 2. In the Fastfile, add setup_ci:
# before_all do
#   setup_ci if is_ci
# end
# This creates a temporary Keychain so certificates don't conflict with the system Keychain
```

---

## Emergency Re-signing

When certificates expire mid-release cycle:

```bash
# 1. Check what's expired
fastlane match development --readonly
# Look for: "Certificate ... has expired"

# 2. Force regenerate the expired certificate type
fastlane match development --force
# or
fastlane match appstore --force

# 3. If profiles are also corrupted, nuke and recreate
# WARNING: This revokes ALL certificates of this type for your team
fastlane match nuke development   # or distribution
fastlane match development --force
fastlane match appstore --force

# 4. Every team member must re-sync
# Send team notification to run:
fastlane match development
fastlane match appstore
```

**Prevention:**
- Set calendar reminders 30 days before certificate expiration
- Automate expiration checking in CI (check cert expiry date in a scheduled workflow)
- Consider using Apple's automatic certificate management in Xcode for development (but keep Match for distribution)

---

## Xcode Automatic vs Manual Signing

| Feature | Automatic Signing | Manual Signing (Match) |
|---|---|---|
| Setup effort | Zero | One-time Match setup |
| Team consistency | No (each dev has own cert) | Yes (everyone uses same cert) |
| CI compatibility | Poor (needs Xcode to manage) | Excellent (Match handles everything) |
| Multi-target | Works but can be confusing | Explicit, predictable |
| Certificate conflicts | Common (revokes other devs' certs) | Impossible (single source of truth) |
| Debugging | Black box (Xcode decides) | Transparent (you control everything) |

**Recommendation:**
- Solo developer, no CI: Automatic signing is fine
- Team of 2+, or any CI: Manual signing with Match
- Never mix: if you use Match, disable automatic signing in Xcode for all targets

```ruby
# In Fastfile, disable automatic signing:
update_code_signing_settings(
    use_automatic_signing: false,
    team_id: "ABCD1234EF",
    profile_name: "match AppStore com.yourapp.ios",
    code_sign_identity: "Apple Distribution",
    targets: ["{SCHEME}"]
)
```

---

## Common Signing Errors and Their Actual Causes

### "No matching profiles found"
```
Actual cause: Provisioning profile not installed on this machine
NOT: "code signing is broken"
Fix: fastlane match development (or appstore)
Verify: ls ~/Library/MobileDevice/Provisioning\ Profiles/
```

### "Code signing error: No certificate for team 'XXXX'"
```
Actual cause: Certificate not in Keychain, or wrong certificate type
NOT: "Apple revoked my certificate"
Fix: fastlane match appstore (or development)
Verify: security find-identity -v -p codesigning
```

### "The certificate has an invalid issuer"
```
Actual cause: Using a certificate from a different Apple Developer team
NOT: "Apple's servers are down"
Fix: Check team_id in Matchfile matches the project's team
Verify: security find-certificate -c "Apple" -a | grep "issu"
```

### "Provisioning profile doesn't include signing certificate"
```
Actual cause: Profile was regenerated with a different certificate
NOT: Random corruption
Fix: fastlane match nuke distribution && fastlane match appstore
This recreates both cert and profile in sync
```

### "This provisioning profile has expired"
```
Actual cause: Profile validity is 1 year, and it's been over a year
Fix: fastlane match appstore --force (regenerates profile)
Prevention: Schedule annual cert/profile renewal in CI
```

### "A valid provisioning profile for this executable was not found"
```
Actual cause: Bundle ID in profile doesn't match the app's bundle ID
Fix: Verify BUNDLE_ID in Matchfile matches the Xcode project's bundle identifier
Check: grep -r "PRODUCT_BUNDLE_IDENTIFIER" *.xcodeproj/project.pbxproj
```

### "Ambiguous signing identity"
```
Actual cause: Multiple certificates of the same type in Keychain
Fix:
  security delete-certificate -c "Apple Distribution: Your Team" (remove duplicates)
  fastlane match appstore (reinstall the correct one)
Verify: security find-identity -v -p codesigning | grep "Apple Distribution"
  Should show exactly ONE matching certificate
```

---

## Certificate Lifecycle

```
Create (match --force)
  |
  v
Install (match)          <-- Every team member runs this
  |
  v
Use (xcodebuild / gym)   <-- Building and signing
  |
  v
Expire (1 year)
  |
  v
Renew (match --force)    <-- Creates new cert, updates profiles
  |
  v
Revoke (match nuke)      <-- Nuclear option, affects entire team
```

**Best practices:**
- Let Match manage the lifecycle. Don't touch certificates manually.
- Run `match --readonly` in CI to avoid accidentally regenerating certificates during builds.
- Use `match appstore --force` only when you need to renew, not as a troubleshooting first step.
- Keep the Match git repo private and access-controlled. It contains your signing identity.

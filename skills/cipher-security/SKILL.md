---
name: cipher-security
description: "Is our data safe, check for vulnerabilities, security posture, before we ship. Security scanning, vulnerability assessment, and compliance auditing for iOS apps — secrets management, OWASP Mobile Top 10, KYC/AML/PCI DSS, encryption audits. Trigger on: 'are we secure', 'audit security', 'scan for secrets', 'hardcoded API keys', 'check authentication', 'compliance review', 'data protection', or pre-launch security review. For security review of a specific PR or diff, use trailofbits-differential-review instead. For production crash investigation, use sentry-find-bugs."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: true
---

# Security & Compliance

> **Iron Law**: "Never ship code that handles sensitive data without a security review. Velocity is not an excuse."

Assess application security posture, enforce compliance standards, and prevent credential exposure across iOS codebases and infrastructure.

> **Project Discovery**: Before executing, determine project-specific values (project name, secrets configuration files, gitignore patterns, backend infrastructure) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, .gitignore).

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|-----------------|----------------|-----------------|
| "Security review slows us down" | A breach costs 100x more than a review. Equifax lost $1.4B from one missed patch. | Schedule security review as non-negotiable sprint work. Budget time for it. |
| "We're not a target" | Automated scanners don't care about your company size. Every app with user data is a target. | Apply baseline security controls regardless of perceived threat level. |
| "The framework handles security" | Frameworks provide tools, not guarantees. Misconfigured Keychain is as bad as no Keychain. | Verify framework security features are correctly configured for your use case. |
| "We'll add security later" | Security debt compounds. Retrofitting auth or encryption into a shipped product is 10x harder. | Build security in from day one. Every PR touching sensitive data gets reviewed. |
| "It's just test data" | Test environments with real-looking data train developers to be careless. Test API keys in production happen constantly. | Use obviously fake test data. Never share credentials between test and production. |
| "Apple's sandbox protects us" | Sandbox prevents some attack vectors, not all. Logic flaws, server-side attacks, and social engineering bypass sandboxing entirely. | Implement defense in depth. Do not rely on a single security boundary. |
| "Our users wouldn't do that" | Users don't attack you -- attackers who compromise user accounts do. | Model threats from the attacker's perspective, not the user's. |
| "It passed App Review" | App Review checks guidelines, not security. Apple does not audit your Keychain usage, API security, or business logic. | Treat App Review as a UX/policy gate, not a security audit. |

---

## Red Flags -- HARD STOP

These are not warnings. Each is a BLOCKING condition. If encountered, STOP all other work until resolved.

- **BLOCKED: Hardcoded API keys or secrets found anywhere in source code** -- even in comments, even in tests, even "temporarily". STOP and remediate before any other work. Remediation: Remove from code, rotate credential, add to .gitignore, verify git history with `git log -p --all -S 'SECRET_VALUE'` to confirm no exposure.
- **BLOCKED: PII appearing in application logs** (NSLog, os_log, print, Logger) -- user emails, phone numbers, tokens, or financial data in log output. STOP. Remove logging immediately. Audit log storage for existing PII exposure. Notify owner.
- **BLOCKED: Force unwraps (`!`) in security-critical code paths** -- authentication, encryption, token validation, biometric checks. A crash in security code is a denial-of-service vulnerability. STOP. Replace with guard/if-let and proper error handling before proceeding.
- **BLOCKED: Sensitive data stored in UserDefaults or plist files** -- these are plaintext XML files trivially readable on jailbroken devices or via device backups. STOP. Migrate to Keychain with appropriate accessibility level before proceeding.
- **BLOCKED: Missing SSL/TLS pinning in production builds** -- without pinning, any CA-signed certificate is trusted, enabling man-in-the-middle attacks on any network. STOP. Implement pinning before shipping.
- **BLOCKED: ATS exceptions allowing arbitrary HTTP loads** -- `NSAllowsArbitraryLoads = YES` disables all transport security. STOP. Each exception must be individually justified with documented business reason.

---

## When NOT to Use This Skill

- **Reviewing a specific PR or diff for security regressions** -- use `trailofbits-differential-review` instead, which is designed for differential analysis with blast radius calculation.
- **Investigating a production crash or error from Sentry** -- use `sentry-find-bugs` to analyze the crash, then return here if the root cause is a security issue.
- **Performance profiling or optimization** -- use `prometheus-performance`. Performance and security occasionally intersect (e.g., timing attacks), but the primary skill should match the primary concern.
- **General code review without security focus** -- use `ios-review` or `sentry-code-review` for non-security-focused review.
- **Infrastructure or CI/CD security** -- this skill focuses on application-level security. Infrastructure security (server hardening, network segmentation) is out of scope.

---

## Decision Framework

```
What kind of security concern is this?
|
+-- Secrets/Credentials
|   +-- Hardcoded in source? --> SOP-1: Pre-Commit Security Scan (P0 Critical)
|   +-- In wrong storage? --> Migrate to Keychain (see references/ios-security-patterns.md)
|   +-- Rotation needed? --> SOP-5 step 4: Encryption Audit + rotation plan
|
+-- Authentication/Authorization
|   +-- Auth bypass possible? --> SOP-5 step 3: Authentication Review (P0 Critical)
|   +-- Missing permission checks? --> Add server-side + client-side validation
|   +-- Biometric implementation? --> See references/ios-security-patterns.md biometric section
|
+-- Data Protection
|   +-- Data at rest? --> Verify Keychain usage, encryption, accessibility levels
|   +-- Data in transit? --> Verify TLS 1.3, certificate pinning, ATS config
|   +-- Data in memory? --> Verify secure erasure after use (zeroing buffers)
|   +-- Data in logs? --> Audit all logging for PII leakage
|
+-- Network Security
|   +-- Missing pinning? --> Implement certificate pinning (P1 High)
|   +-- ATS exceptions? --> Audit each exception, remove unnecessary ones
|   +-- Cleartext traffic? --> Block all HTTP, enforce HTTPS
|
+-- Compliance
|   +-- Financial (KYC/AML/PCI)? --> See references/fintech-compliance.md
|   +-- Privacy (GDPR/CCPA)? --> Data mapping + consent + deletion audit
|   +-- Healthcare (HIPAA)? --> Encryption + access controls + audit logging
|
+-- Platform Security
    +-- Jailbreak detection? --> See references/ios-security-patterns.md
    +-- Runtime protection? --> Anti-debug, integrity checks, obfuscation
    +-- App Transport Security? --> ATS configuration audit
```

---

## Behavioral Enforcement

### Phase Gate: SECRETS SCAN (MUST Be First)
**Cannot proceed to ANY other analysis until:**
- [ ] `gitleaks detect --source . --verbose` has been run (or equivalent)
- [ ] Results reviewed -- zero secrets found, OR found secrets documented as P0 Critical
- [ ] If secrets found: STOP all other work. Report immediately. Rotate compromised credentials.

**Why first**: A hardcoded secret is a P0 Critical that makes everything else irrelevant. Finding it after spending an hour on STRIDE analysis wastes time. Find it first.

### Phase Gate: STRIDE COMPLETION
**Cannot proceed to findings until ALL 6 categories addressed:**
- [ ] **S**poofing -- identity threats analyzed
- [ ] **T**ampering -- data integrity threats analyzed
- [ ] **R**epudiation -- audit trail threats analyzed
- [ ] **I**nformation Disclosure -- data exposure threats analyzed
- [ ] **D**enial of Service -- availability threats analyzed
- [ ] **E**levation of Privilege -- authorization threats analyzed

Missing category = incomplete threat model. Go back and address it.

### Phase Gate: FINDINGS FORMAT
**Every finding MUST follow this exact format:**
`[SEC-XXX] | [STRIDE Category] | [Critical/High/Medium/Low] | [Description] | [Evidence] | [Remediation]`

Findings without all 6 fields are INCOMPLETE. Generic findings like "improve security" are REJECTED.

**Hard Stop**: If a Critical finding is identified -- STOP all other analysis. Report it immediately. Critical findings block deployment.

### Counter-Based Escalation
- **Review pass #1**: Normal. Work through STRIDE, document findings.
- **Review pass #2**: If revisiting because pass #1 missed something, write down what was missed and why.
- **Review pass #3**: STOP. If you are on your third pass, the scope may be too large or the architecture too complex. ESCALATE to human partner with: what you have found so far, what you are uncertain about, what you need clarified.

---

## Threat Modeling: STRIDE for iOS

Before any security assessment, perform STRIDE analysis adapted for mobile:

### S -- Spoofing (Identity)
**Question**: Can an attacker impersonate a legitimate user or component?

| Attack Vector | iOS Context | Detection Method |
|--------------|-------------|------------------|
| Stolen session tokens | Tokens stored insecurely or transmitted without TLS | Grep for token storage locations, check Keychain accessibility |
| Fake biometric auth | LAContext result bypassed or replayed | Review LAPolicy usage, check for server-side verification |
| Deep link spoofing | Malicious app registers same URL scheme | Check for Universal Links validation, URL scheme handling |
| Push notification spoofing | Fake push payloads triggering actions | Verify push payload validation, server-side origin check |

### T -- Tampering
**Question**: Can an attacker modify data, code, or behavior?

| Attack Vector | iOS Context | Detection Method |
|--------------|-------------|------------------|
| Jailbreak + runtime manipulation | Method swizzling, debugger attachment | Check for jailbreak detection, anti-debug measures |
| Man-in-the-middle | Modified API responses | Check certificate pinning, TLS configuration |
| Local data modification | Modified Keychain, UserDefaults, Core Data | Verify data integrity checks, server-side validation |
| Binary patching | Modified IPA sideloaded | Check code signing verification, integrity hashes |

### R -- Repudiation
**Question**: Can a user deny performing an action?

| Attack Vector | iOS Context | Detection Method |
|--------------|-------------|------------------|
| Missing audit trail | Financial transactions without server-side logging | Check audit log completeness, server-side event recording |
| Client-side-only logging | Logs that can be deleted by the user | Verify critical events are logged server-side |
| Unsigned transactions | Actions without cryptographic proof of origin | Check for request signing, transaction receipts |

### I -- Information Disclosure
**Question**: Can sensitive data leak to unauthorized parties?

| Attack Vector | iOS Context | Detection Method |
|--------------|-------------|------------------|
| Log leakage | PII in os_log, NSLog, print statements | Grep all logging calls for sensitive data patterns |
| Backup exposure | Sensitive data included in iTunes/iCloud backups | Check `isExcludedFromBackup` on sensitive file URLs |
| Pasteboard leakage | Copied passwords/tokens accessible to other apps | Check `UIPasteboard` usage for sensitive operations |
| Screenshot exposure | Sensitive data visible in app switcher snapshot | Check for overlay/blur on `applicationWillResignActive` |
| Memory dump | Sensitive data in memory readable via debugger | Check for memory zeroing after use |

### D -- Denial of Service
**Question**: Can an attacker make the app unavailable?

| Attack Vector | iOS Context | Detection Method |
|--------------|-------------|------------------|
| Force unwrap crash | Nil value in security-critical path crashes app | Grep for `!` in auth, crypto, and validation code |
| Resource exhaustion | Unbounded data loading, infinite loops | Check for pagination, timeouts, cancellation |
| Watchdog kill | Main thread blocked during security operations | Verify crypto and network operations are off main thread |

### E -- Elevation of Privilege
**Question**: Can an attacker gain unauthorized access?

| Attack Vector | iOS Context | Detection Method |
|--------------|-------------|------------------|
| Client-side auth decisions | Role checks only on client | Verify all authorization is server-enforced |
| Jailbreak privilege escalation | App runs with elevated privileges on jailbroken device | Check jailbreak detection, behavior modification |
| Entitlement abuse | Over-requested entitlements | Audit entitlements file for minimum necessary |
| Keychain access group misconfiguration | Other apps reading your Keychain items | Verify access group configuration |

---

## Data Flow Diagram Methodology

Trace sensitive data through every stage. For each sensitive data type in your app, map:

### Step 1: Identify Sensitive Data
```
- Authentication tokens (access token, refresh token, session ID)
- User credentials (password, biometric template reference)
- Personal information (name, email, phone, address, SSN)
- Financial data (account numbers, transaction amounts, card data)
- Encryption keys (symmetric keys, private keys, certificates)
```

### Step 2: Map the Flow
For each data type, document:
```
INPUT (where does it enter the app?)
  --> User input (keyboard, camera, biometric sensor)
  --> API response (server, third-party service)
  --> Local storage (Keychain, Core Data, file system)
  --> System service (push notification, deep link, pasteboard)

PROCESSING (how is it used?)
  --> Validation (is it checked before use?)
  --> Transformation (is it encrypted, hashed, or encoded?)
  --> Business logic (what decisions depend on it?)

STORAGE (where does it persist?)
  --> Keychain (with what accessibility level?)
  --> Core Data / SQLite (encrypted?)
  --> File system (excluded from backup?)
  --> UserDefaults (NEVER for sensitive data)
  --> Memory only (zeroed after use?)

OUTPUT (where does it leave the app?)
  --> API request (over TLS? pinned?)
  --> Logs (PII stripped?)
  --> UI display (masked? redacted in screenshots?)
  --> Pasteboard (expiring? local-only?)
  --> Other apps (via share sheet, URL scheme, extensions?)
```

### Step 3: Identify Violations
At each transition point, verify:
- Is the data encrypted in transit?
- Is the data encrypted at rest?
- Is access properly authorized?
- Is the data minimized (only what's needed)?
- Can the data be intercepted or leaked?

---

## Penetration Testing Methodology

### What to Test

| Category | Tests | Tools |
|----------|-------|-------|
| **Network** | MITM, certificate pinning bypass, ATS compliance | Charles Proxy, mitmproxy, Burp Suite |
| **Storage** | Keychain dump, file system inspection, backup analysis | Keychain-Dumper, iExplorer, libimobiledevice |
| **Binary** | String extraction, symbol analysis, class dump | otool, nm, class-dump, Hopper |
| **Runtime** | Method swizzling, debugger attachment, Frida scripts | Frida, lldb, Cycript |
| **Auth** | Token replay, session hijacking, biometric bypass | Custom scripts, Burp Suite |
| **Input** | Deep link injection, pasteboard interception, IPC fuzzing | Custom test harnesses |

### How to Test (Workflow)

```
1. STATIC ANALYSIS
   - Extract IPA, inspect binary for strings, symbols, embedded credentials
   - Review Info.plist for ATS exceptions, URL schemes, entitlements
   - Analyze entitlements for over-provisioning
   - Check for debug symbols in release builds

2. DYNAMIC ANALYSIS
   - Set up proxy (Charles/Burp) to intercept network traffic
   - Attempt certificate pinning bypass
   - Monitor file system writes during app usage
   - Inspect Keychain entries after authentication
   - Check pasteboard contents after copy operations

3. RUNTIME MANIPULATION
   - Attach debugger to running process
   - Hook security-critical methods (Frida)
   - Bypass jailbreak detection
   - Modify return values of auth checks

4. BUSINESS LOGIC
   - Replay captured API requests with modified parameters
   - Test authorization boundaries (access other users' data)
   - Test rate limiting on sensitive operations
   - Verify transaction integrity
```

---

## Core Capabilities

### Secrets Management
- API key and credential protection
- Gitignore validation for sensitive files
- Environment variable security
- Secrets rotation guidance

### Vulnerability Assessment
- Code scanning for security issues
- Dependency vulnerability checking
- OWASP Mobile Top 10 compliance
- Penetration test guidance

### Financial Compliance
- KYC (Know Your Customer) implementation
- AML (Anti-Money Laundering) compliance
- BSA (Bank Secrecy Act) requirements
- PCI DSS compliance for payment data
- GDPR/CCPA data protection

### Secure Development
- Secure coding practices
- Authentication/authorization patterns
- Encryption implementation
- Input validation

### Transaction Security
- Transaction validation and integrity
- Fraud detection mechanisms
- Rate limiting and velocity checks
- Audit trail completeness

### iOS-Specific Security
- Jailbreak detection implementation
- Secure keychain usage
- App transport security
- Runtime application self-protection

---

## Standard Operating Procedures

### SOP-1: Pre-Commit Security Scan
```bash
# Scan for secrets before commit
gitleaks detect --source . --verbose

# Check that sensitive files are gitignored
# Adapt these to your project's secrets configuration:
git check-ignore *.xcconfig        # Xcode config files with API keys
git check-ignore .env              # Environment variable files
git check-ignore *.p12             # Certificate files
git check-ignore *-Info.plist      # Service configuration plists (if containing keys)
```

### SOP-2: Dependency Vulnerability Scan
```bash
# Review Package.resolved for pinned versions
cat Package.resolved | grep -E '"version"|"package"' | paste - - | sort

# Check each dependency against known CVE databases
# 1. Review GitHub Security Advisories for each dependency
# 2. Check National Vulnerability Database (NVD) for matching CVEs
# 3. Review dependency changelogs for security-related fixes

# Automated scanning options:
# - GitHub Dependabot (enable in repository settings)
# - Snyk: snyk test --all-projects
# - OWASP Dependency-Check (if configured)

# Manual Package.resolved audit checklist:
# [ ] All dependencies pinned to exact versions (not ranges)
# [ ] No dependencies with known unpatched CVEs
# [ ] All dependencies from trusted sources (verified publishers)
# [ ] No unnecessary dependencies (remove unused)
# [ ] Last audit date recorded
```

### SOP-3: Security Code Review Checklist
```
[ ] No hardcoded secrets or API keys
[ ] No force unwraps in security-critical code
[ ] Input validation on all user inputs
[ ] Proper error handling (no sensitive data in errors)
[ ] Secure storage for sensitive data (Keychain)
[ ] SSL pinning enabled for production
[ ] Biometric auth properly implemented
[ ] Session timeout configured
[ ] Jailbreak detection active
[ ] Proper encryption for data at rest (AES-256) and in transit (TLS 1.3)
[ ] No PII in application logs
[ ] Session tokens have appropriate expiration and rotation
[ ] Proper memory management for sensitive data (zeroing buffers)
[ ] Screenshot/app switcher protection for sensitive screens
[ ] Pasteboard cleared after sensitive operations
[ ] Deep link parameters validated and sanitized
[ ] Background snapshot does not expose sensitive data
```

### SOP-4: Audit Trail Verification
```sql
-- Verify audit logging is enabled
SELECT * FROM audit_logs ORDER BY created_at DESC LIMIT 10;

-- Check for gaps in audit trail
SELECT
  DATE_TRUNC('hour', created_at) as hour,
  COUNT(*) as events
FROM audit_logs
GROUP BY 1
ORDER BY 1 DESC;
```

### SOP-5: Security Review Process
1. **Secrets Scan** -- Search codebase for hardcoded keys, tokens, passwords, and credentials
2. **Data Flow Analysis** -- Trace sensitive data from input through storage to transmission (see Data Flow Diagram Methodology above)
3. **STRIDE Threat Model** -- Run STRIDE analysis for the feature or component under review
4. **Authentication Review** -- Verify auth implementation against security best practices
5. **Encryption Audit** -- Validate cryptographic implementations and key management
6. **Compliance Check** -- Verify applicable compliance requirements (see `references/fintech-compliance.md`)
7. **iOS Security Review** -- Check platform-specific security (see `references/ios-security-patterns.md`)
8. **Dependency Audit** -- Review third-party libraries for known vulnerabilities
9. **Penetration Test Guidance** -- Identify what should be tested and how (see Penetration Testing Methodology above)

---

## OWASP Mobile Top 10

| ID | Category | Key Checks |
|----|----------|------------|
| M1 | Improper Platform Usage | Correct iOS API usage, entitlements |
| M2 | Insecure Data Storage | Keychain for secrets, no PII in UserDefaults |
| M3 | Insecure Communication | TLS 1.3, certificate pinning |
| M4 | Insecure Authentication | MFA, biometric, session management |
| M5 | Insufficient Cryptography | AES-256 at rest, no custom crypto |
| M6 | Insecure Authorization | RLS policies, permission checks |
| M7 | Client Code Quality | No force unwraps, input validation |
| M8 | Code Tampering | Jailbreak detection, integrity checks |
| M9 | Reverse Engineering | Obfuscation, anti-debug |
| M10 | Extraneous Functionality | No debug endpoints in production |

---

## Sensitive Files Registry

> **Note**: Determine your project's specific sensitive files by checking `.gitignore` and CLAUDE.md. Common patterns below.

| File Pattern | Contains | Protection |
|--------------|----------|------------|
| `*.xcconfig` (secrets) | API keys, service credentials | .gitignore |
| `.env`, `.env.*` | Environment variables | .gitignore |
| `*.p12`, `*.cer`, `*.key` | Certificates and private keys | .gitignore |
| `*ServiceAccount*.json` | Cloud service credentials | .gitignore |
| `*-Info.plist` (with keys) | Service configuration (Firebase, etc.) | .gitignore |
| `*.keychain-db` | Keychain databases | .gitignore |
| `*.mobileprovision` | Provisioning profiles | .gitignore |
| `*.ipa` | App binaries (may contain embedded secrets) | .gitignore |

---

## Security Response

| Concern | Action |
|---------|--------|
| Vulnerability found | Document, assess severity (CVSS), fix immediately if Critical/High |
| Data breach suspected | Incident response, notify owner, preserve evidence |
| Compliance question | Research, document, escalate to owner |
| Hardcoded secret found | Rotate immediately, remove from code, add to .gitignore |
| PII in logs discovered | Remove logging, audit log storage for existing PII, notify owner |

---

## Quality Gates (Before Marking Complete)

- [ ] All hardcoded secrets identified and remediated (or confirmed absent)
- [ ] Data flow diagram completed for all sensitive data types
- [ ] STRIDE threat model completed for the component under review
- [ ] OWASP Mobile Top 10 checklist verified
- [ ] Keychain usage verified with correct accessibility levels
- [ ] ATS configuration audited (no unnecessary exceptions)
- [ ] Certificate pinning verified for production
- [ ] No PII in any logging statements
- [ ] Compliance requirements identified and verified (KYC/AML/PCI/GDPR as applicable)
- [ ] Security findings documented with severity, evidence, and remediation

---

## Required Output Artifact

Every execution of this skill MUST produce a structured security report containing:

```
## Security Assessment Report
**Date**: [date]
**Scope**: [component/feature/app reviewed]
**Reviewer**: [agent/human]

### Secrets Scan
- Tool used: [gitleaks / manual / other]
- Result: [PASS -- zero secrets / FAIL -- N secrets found]
- Actions taken: [if any]

### STRIDE Threat Model
| Category | Threats Identified | Severity | Status |
|----------|-------------------|----------|--------|
| Spoofing | [findings or "No threats identified"] | [sev] | [open/mitigated] |
| Tampering | ... | ... | ... |
| Repudiation | ... | ... | ... |
| Information Disclosure | ... | ... | ... |
| Denial of Service | ... | ... | ... |
| Elevation of Privilege | ... | ... | ... |

### Findings
[SEC-001] | [STRIDE Category] | [Severity] | [Description] | [Evidence] | [Remediation]
[SEC-002] | ...

### OWASP Mobile Top 10 Compliance
| ID | Category | Status | Notes |
|----|----------|--------|-------|
| M1-M10 | ... | PASS/FAIL/N/A | ... |

### Coverage Limitations
[What was NOT reviewed and why]
```

**Incomplete reports are not accepted.** If you cannot fill a section, state why explicitly.

---

## Self-Audit Protocol

Before declaring this skill's execution complete, verify ALL of the following:

- [ ] **Phase Gate: Secrets Scan** -- Was `gitleaks` (or equivalent) run FIRST? Can you quote the result?
- [ ] **Phase Gate: STRIDE** -- Are all 6 STRIDE categories addressed? (Not 5. All 6.)
- [ ] **Phase Gate: Findings** -- Does every finding have all 6 fields? (`[ID] | [Category] | [Severity] | [Description] | [Evidence] | [Remediation]`)
- [ ] **Hard Stops checked** -- Were all Red Flag conditions checked? If any were found, were they treated as BLOCKING?
- [ ] **No generic findings** -- Review each finding. Would a developer know EXACTLY what to do from reading it? If not, add specificity.
- [ ] **Output artifact produced** -- Is the structured report complete with all sections filled?
- [ ] **Coverage limitations stated** -- Did you explicitly document what was NOT reviewed?

**If any checkbox is unchecked, you are not done.** Go back and complete it.

---

## Cross-Skill References

- `trailofbits-differential-review` -- For security review of specific PRs/diffs with blast radius analysis
- `sentry-find-bugs` -- When a security issue manifests as a production crash
- `sentry-code-review` -- When Sentry bot identifies security-related findings on a PR
- `prometheus-performance` -- When security operations (encryption, hashing) impact performance
- `forge-development` -- For implementing security patterns during feature development

---

## References

- Detailed financial compliance checklists: `references/fintech-compliance.md`
- iOS-specific security implementation patterns: `references/ios-security-patterns.md`
- [OWASP Mobile Security Testing Guide](https://mas.owasp.org/MASTG/)
- [Apple Platform Security Guide](https://support.apple.com/guide/security/)

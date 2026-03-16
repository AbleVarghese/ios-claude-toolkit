---
name: trailofbits-differential-review
description: "Is this change safe, security implications of this PR, did we break any security. Security-focused differential code review for PRs, commits, and diffs — blast radius calculation, adversarial analysis, test coverage verification, vulnerability pattern detection. Trigger on: 'review this PR for security', 'could this introduce a vulnerability', 'check if the refactor broke security', 'what's the risk of merging this', or any security-focused review of specific changes. For broad security audits not tied to a diff, use cipher-security instead."
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
---

# Differential Security Review

> **Iron Law**: "Security review is not optional for changes touching auth, crypto, payments, or external calls. Size of the diff is irrelevant -- Heartbleed was 2 lines."

Security-focused code review for PRs, commits, and diffs.

Source: [github.com/trailofbits/skills](https://github.com/trailofbits/skills)

---

## Core Principles

1. **Risk-First**: Focus on auth, crypto, value transfer, external calls
2. **Evidence-Based**: Every finding backed by git history, line numbers, attack scenarios
3. **Adaptive**: Scale to codebase size (SMALL/MEDIUM/LARGE)
4. **Honest**: Explicitly state coverage limits and confidence level
5. **Output-Driven**: Always generate comprehensive markdown report file

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|-----------------|----------------|-----------------|
| "Small PR, quick review" | Heartbleed was 2 lines. The goto fail bug was one duplicated line. | Classify by RISK, not size |
| "I know this codebase" | Familiarity breeds blind spots. You stop seeing what you expect to be there. | Build explicit baseline context |
| "Git history takes too long" | History reveals regressions, reverted security fixes, and patterns of past vulnerabilities. | Never skip Phase 1 |
| "Blast radius is obvious" | Transitive callers get missed. A change to a utility function can affect 200+ call sites. | Calculate quantitatively |
| "No tests = not my problem" | Missing tests = elevated risk rating. You can't verify the change is safe without tests. | Flag in report, elevate severity |
| "Just a refactor, no security impact" | Refactors break invariants. Moving code between modules can cross trust boundaries. Access control changes hide in refactors. | Analyze as HIGH until proven LOW |
| "I'll explain verbally" | No artifact = findings lost. The next reviewer won't know what you checked. | Always write report |
| "It's internal code, not user-facing" | Internal code handles user data. Injection, access control, and data leaks happen in internal code. | Apply the same rigor to all code that touches sensitive data |
| "The tests pass, so it's safe" | Tests verify intended behavior. Security bugs are about unintended behavior. Tests don't test for attacks. | Test coverage is necessary but not sufficient. Adversarial analysis is separate. |

---

## Red Flags -- HARD STOP

These are not warnings. Each is a BLOCKING condition. If encountered, STOP all other work until the condition is resolved or explicitly accepted by the owner.

- **BLOCKED: Removed code from commits with "security", "CVE", or "fix vulnerability" in the message** -- this is a potential security regression. STOP. Investigate why the fix was removed. Cannot proceed without understanding and documenting the justification.
- **BLOCKED: Access control modifiers weakened** (e.g., `private` -> `public`, `internal` -> `open`) -- especially on methods that handle auth, crypto, or data access. STOP. Document why the change is necessary and what exposure it creates.
- **BLOCKED: Validation removed without replacement** -- if input validation, bounds checking, or sanitization was removed, STOP. The change is HIGH risk until proven otherwise. Require explicit justification.
- **BLOCKED: External calls added without error handling or input validation** -- new network requests, IPC calls, or URL scheme handlers without proper validation are injection vectors. STOP. Add validation before proceeding.
- **BLOCKED: High blast radius (50+ callers) combined with HIGH risk change** -- requires full impact analysis. STOP. A bug here affects the entire app. Cannot proceed without sampling callers.
- **BLOCKED: Certificate pinning disabled or weakened** -- any change to pinning configuration, SSL trust evaluation, or ATS exceptions. STOP. Requires immediate investigation and owner approval.
- **BLOCKED: Keychain accessibility level changed** -- changing from `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` to a less restrictive level exposes data. STOP. Requires security justification.
- **BLOCKED: Biometric authentication policy changed** -- switching from `.deviceOwnerAuthenticationWithBiometrics` to `.deviceOwnerAuthentication` adds passcode fallback. STOP. May not meet security requirements. Requires owner approval.

---

## When NOT to Use This Skill

- **Greenfield code** (no baseline to compare) -- use `cipher-security` for initial security review of new code
- **Documentation-only changes** (no security impact) -- standard review is sufficient
- **Formatting/linting** (cosmetic changes) -- unless the formatter moved code across trust boundaries
- **User explicitly requests quick summary only** (they accept risk) -- document that the user accepted reduced review
- **Investigating a production crash** -- use `sentry-find-bugs` for crash investigation, then return here if the fix needs security review

---

## Behavioral Enforcement

### Phase Gate: BLAST RADIUS (For Every HIGH-Risk Change)
**Cannot proceed to deep analysis until:**
- [ ] Blast radius calculated: `grep -rl "functionName" --include="*.swift" . | wc -l`
- [ ] All direct callers identified
- [ ] Transitive impact assessed (callers of callers for HIGH blast radius)

**Cannot skip for "simple" changes**: Heartbleed was 2 lines. Blast radius is calculated by IMPACT, not by diff size.

### Phase Gate: GIT HISTORY (For Removed/Changed Security Code)
**If code was removed or significantly changed, MUST check:**
- [ ] `git log --oneline -10 -- <file>` -- why was this code added?
- [ ] `git blame <file>` -- who added it and when?
- [ ] Was it from a commit mentioning "fix", "security", "CVE", or "vulnerability"?

**If yes -- CRITICAL ESCALATION**: Code that was added to fix a security issue is being removed. This MUST be justified. Cannot proceed without understanding why the original fix was needed and confirming the removal doesn't reintroduce the vulnerability.

### Counter-Based Escalation
- **Finding #1**: Normal. Document with evidence.
- **Finding #2**: Check for pattern. Are both findings related to the same root cause?
- **Finding #3**: STOP. Three security findings in one PR is a signal. Consider whether the PR should be restructured, whether the author needs security guidance, or whether a broader security review is needed.
- **Still finding issues**: ESCALATE to human partner with: all findings documented, pattern analysis, recommendation on whether to proceed or request rework.

---

## Quick Reference

### Codebase Size Strategy

| Codebase Size | Strategy | Approach |
|---------------|----------|----------|
| SMALL (<20 files) | DEEP | Read all deps, full git blame |
| MEDIUM (20-200) | FOCUSED | 1-hop deps, priority files |
| LARGE (200+) | SURGICAL | Critical paths only |

### Risk Level Triggers

| Risk Level | Triggers |
|------------|----------|
| HIGH | Auth, crypto, external calls, value transfer, validation removal, Keychain, biometric, certificate pinning |
| MEDIUM | Business logic, state changes, new public APIs, deep link handling, data model changes |
| LOW | Comments, tests, UI styling, logging, localization |

---

## Decision Framework

```
PR/diff received for security review
|
+-- Phase 0: Triage
|   +-- List all changed files
|   +-- Classify each by risk level (HIGH/MEDIUM/LOW)
|   +-- Any HIGH risk files?
|       +-- YES --> Full review (all phases)
|       +-- NO, but MEDIUM --> Focused review (Phases 0-3, skip Phase 4-5 unless concerns found)
|       +-- ALL LOW --> Quick review + document coverage limits
|
+-- Phase 1: Code Analysis
|   +-- Read diffs line by line for HIGH/MEDIUM files
|   +-- Check git blame on security-relevant removed code
|   +-- Trace data flow across trust boundaries
|   |
|   +-- iOS-specific checks:
|       +-- URL scheme / deep link handlers changed? --> Check for injection
|       +-- Keychain operations changed? --> Check accessibility levels
|       +-- Biometric auth changed? --> Check policy and fallback
|       +-- Pasteboard operations? --> Check for sensitive data exposure
|       +-- WebView configuration? --> Check for JavaScript injection
|
+-- Phase 2: Test Coverage
|   +-- Find tests for each changed file
|   +-- Are new code paths tested?
|   +-- Are security-specific scenarios tested?
|   +-- Missing tests? --> Elevate risk rating
|
+-- Phase 3: Blast Radius
|   +-- Count callers of changed functions
|   +-- Identify transitive dependencies
|   +-- HIGH blast radius? --> Phase 4 required
|
+-- Phase 4: Deep Context (HIGH risk only)
|   +-- Check git history for security-related commits
|   +-- Review architecture docs for security constraints
|   +-- Check dependencies for known vulnerabilities
|
+-- Phase 5: Adversarial Analysis (HIGH risk findings)
|   +-- Model attacker perspective
|   +-- Build concrete exploit scenarios
|   +-- See references/attack-scenarios.md for iOS-specific scenarios
|
+-- Phase 6: Report
    +-- Generate report using references/report-template.md
    +-- Summarize for user
```

---

## Workflow

```
Pre-Analysis -> Phase 0: Triage -> Phase 1: Code Analysis -> Phase 2: Test Coverage
    |              |                    |                        |
Phase 3: Blast Radius -> Phase 4: Deep Context -> Phase 5: Adversarial -> Report
```

---

## Phase 0: Triage

### Phase Gate: FILE-LEVEL TRIAGE ✓
**Cannot proceed to Phase 1 until EVERY changed file is individually classified.**

```bash
# Get changed files
git diff --name-only HEAD~1..HEAD
# Or for a PR
git diff --name-only main..HEAD
```

**Classify EACH file individually** — not by category, not by change type. Every. Single. File.

**Output format (mandatory):**
```
| # | File | Risk | Reason |
|---|------|------|--------|
| 1 | AuthService.swift | HIGH | Touches authentication flow |
| 2 | TokenStorage.swift | HIGH | Handles credential storage |
| 3 | README.md | LOW | Documentation only |
```

If the PR says "12 files changed", your triage table must have 12 rows. If you classify by category instead of by file, the triage is INCOMPLETE — go back and list each file individually. Category-level triage misses the file that doesn't fit neatly into a category but still has security implications.

**Risk classification per file:**
- **HIGH**: Files touching auth, crypto, payments, validation, access control, external calls, Keychain, biometric, certificate pinning, URL scheme handling, entitlements
- **MEDIUM**: Business logic, state management, API endpoints, data models, navigation, deep links
- **LOW**: Comments, documentation, UI styling, test files, logging, localization

---

## Phase 1: Code Analysis

For each HIGH and MEDIUM risk file:

1. **Read the diff** carefully, line by line
2. **Check git blame** on removed security-relevant code:
   ```bash
   git log --oneline -10 -- path/to/file.swift
   git blame path/to/file.swift
   ```
3. **Trace data flow** from inputs to outputs
4. **Identify trust boundaries** crossed by the change

### iOS-Specific Analysis Points

For each changed file, check for these iOS-specific vulnerability patterns. See `references/vulnerability-patterns.md` for the full catalog.

#### URL Scheme / Universal Link Changes
```swift
// RED FLAG: URL scheme handler without input validation
func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey: Any]) -> Bool {
    let action = url.host // UNVALIDATED -- attacker controls this value
    performAction(action!) // Force unwrap + unvalidated input = crash + injection
    return true
}

// SECURE: Validate and sanitize all URL parameters
func application(_ app: UIApplication, open url: URL, options: [UIApplication.OpenURLOptionsKey: Any]) -> Bool {
    guard let host = url.host,
          let action = Action(rawValue: host) else {
        logger.warning("invalid_url_scheme", ["url": url.scheme ?? "nil"])
        return false
    }
    // Validate source app if needed
    guard let sourceApp = options[.sourceApplication] as? String,
          allowedSourceApps.contains(sourceApp) else {
        return false
    }
    performAction(action)
    return true
}
```

#### Keychain Operation Changes
```swift
// RED FLAG: Accessibility level too permissive
let query: [String: Any] = [
    kSecClass: kSecClassGenericPassword,
    kSecAttrAccessible: kSecAttrAccessibleAlways, // Accessible even when locked!
    kSecValueData: tokenData
]

// SECURE: Restrict to unlocked, this device only
let query: [String: Any] = [
    kSecClass: kSecClassGenericPassword,
    kSecAttrAccessible: kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
    kSecAttrAccessGroup: "com.company.app.shared", // Explicit access group
    kSecValueData: tokenData
]
```

#### Pasteboard Data Exposure
```swift
// RED FLAG: Sensitive data on system pasteboard
UIPasteboard.general.string = accountNumber // Accessible to ALL apps

// SECURE: Use local-only pasteboard with expiration
let pasteboard = UIPasteboard(name: .init("com.company.app.local"), create: true)
pasteboard?.setItems([[UIPasteboard.typeAutomatic: accountNumber]],
                      options: [.localOnly: true,
                               .expirationDate: Date().addingTimeInterval(60)])
```

#### WebView JavaScript Injection
```swift
// RED FLAG: Loading untrusted content with JavaScript enabled
let webView = WKWebView()
let config = WKWebViewConfiguration()
config.preferences.javaScriptEnabled = true // Default is true
webView.load(URLRequest(url: untrustedURL)) // Attacker-controlled content can execute JS

// SECURE: Restrict JavaScript, validate URLs, use content rules
let config = WKWebViewConfiguration()
let contentController = WKUserContentController()
// Add content rules to block unwanted scripts
config.userContentController = contentController
config.defaultWebpagePreferences.allowsContentJavaScript = false // Disable unless needed
let webView = WKWebView(frame: .zero, configuration: config)

// Only load trusted URLs
guard trustedDomains.contains(url.host ?? "") else {
    logger.warning("blocked_untrusted_webview_url", ["url": url.absoluteString])
    return
}
```

#### Background Snapshot Exposure
```swift
// RED FLAG: Sensitive data visible in app switcher
// No protection on applicationWillResignActive

// SECURE: Blur or overlay sensitive content
func applicationWillResignActive(_ application: UIApplication) {
    let blurEffect = UIBlurEffect(style: .light)
    let blurView = UIVisualEffectView(effect: blurEffect)
    blurView.frame = window?.bounds ?? .zero
    blurView.tag = 999
    window?.addSubview(blurView)
}

func applicationDidBecomeActive(_ application: UIApplication) {
    window?.viewWithTag(999)?.removeFromSuperview()
}
```

---

## Phase 2: Test Coverage

For each changed file, verify:

```bash
# Find related test files
find . -name "*Tests.swift" | xargs grep -l "ClassName"

# Check if new code paths have tests
grep -n "func test" path/to/related/Tests.swift
```

Flag any HIGH risk changes without corresponding test coverage.

**Security-specific test coverage to look for:**
- [ ] Authentication bypass scenarios tested
- [ ] Authorization boundary tests (user A can't access user B's data)
- [ ] Input validation edge cases (empty, nil, malformed, oversized)
- [ ] Error handling paths tested (not just happy path)
- [ ] Concurrency scenarios tested (parallel access, race conditions)
- [ ] Deep link injection scenarios tested (malicious URL parameters)

---

## Phase 3: Blast Radius

Calculate how many callers are affected:

```bash
# Find all callers of changed function
grep -rn "functionName" --include="*.swift" .

# Count affected files
grep -rl "functionName" --include="*.swift" . | wc -l
```

| Blast Radius | Callers | Action |
|-------------|---------|--------|
| LOW | 0-5 | Standard review |
| MEDIUM | 6-20 | Check all callers |
| HIGH | 21-50 | Sample callers + elevate risk |
| CRITICAL | 50+ | Full impact analysis required |

---

## Phase 4: Deep Context

For HIGH risk changes, gather additional context:

- **Git history**: Was this code previously fixed for security? Check commit messages for "fix", "CVE", "security", "vulnerability"
  ```bash
  git log --all --oneline -- path/to/file.swift | grep -iE "fix|security|vuln|CVE|auth|crash"
  ```
- **Related issues**: Are there open issues related to this code?
- **Documentation**: Does architecture documentation mention security constraints?
- **Dependencies**: Do changed dependencies have known vulnerabilities?

---

## Phase 5: Adversarial Analysis

For HIGH risk findings, model attacker perspective. See `references/attack-scenarios.md` for detailed iOS-specific exploitation scenarios.

1. **Attacker Goal**: What could an attacker achieve by exploiting this?
2. **Attack Surface**: What inputs does the attacker control?
3. **Exploit Scenario**: Step-by-step exploitation
4. **Impact**: Data breach, financial loss, privilege escalation, DoS
5. **Exploitability**: How difficult is exploitation? (EASY/MEDIUM/HARD)

### iOS-Specific Attack Vectors to Consider

| Vector | Attacker Control | Common Exploit |
|--------|-----------------|----------------|
| Deep links / URL schemes | Full URL, parameters, source app | Parameter injection, action hijacking |
| Pasteboard | Read data from general pasteboard | Token/credential theft from copied values |
| Push notifications | Payload content (via compromised server) | UI spoofing, phishing, action triggering |
| Network (MITM) | API responses if pinning is bypassed | Data manipulation, credential interception |
| Local file system (jailbreak) | Full file system read/write | Keychain dump, data extraction, binary patching |
| Method swizzling (jailbreak) | Runtime method behavior | Auth bypass, biometric bypass, integrity check bypass |
| IPC / App extensions | Shared container data, extension requests | Data injection, privilege escalation |
| Background snapshots | Visual state of app captured by system | Sensitive data capture from app switcher |

---

## Expanded Vulnerability Taxonomy

See `references/vulnerability-patterns.md` for the complete catalog of 30+ vulnerability patterns. Key categories:

### Authentication & Authorization
- Auth bypass via modified client state
- Token leakage via logging, pasteboard, or insecure storage
- Biometric bypass via LAPolicy manipulation
- Session fixation via predictable tokens
- Missing re-authentication for sensitive operations

### Data Protection
- Keychain accessibility misconfiguration
- Pasteboard data exposure
- Background snapshot sensitive data
- Backup inclusion of sensitive files
- Screenshot capture of sensitive screens

### Network Security
- Certificate pinning bypass
- ATS exception abuse
- Custom URL handler SSRF
- WebView content injection
- Insecure WebSocket connections

### Code Integrity
- Runtime method swizzling
- Binary analysis exposure
- NSCoding deserialization attacks
- Dynamic library injection
- Debugger attachment

### Input Handling
- Deep link parameter injection
- Universal Link validation bypass
- URL scheme hijacking
- Notification content exploitation
- IPC/XPC authorization bypass

---

## Quality Gates (Before Marking Complete)

- [ ] All changed files analyzed and classified by risk level
- [ ] Git blame completed on removed security-relevant code
- [ ] Blast radius calculated for all HIGH risk changes
- [ ] Attack scenarios are concrete with specific exploit steps (not generic "could be bad")
- [ ] Findings reference specific line numbers, commits, and code snippets
- [ ] iOS-specific vulnerability patterns checked (URL schemes, Keychain, pasteboard, WebView, snapshots)
- [ ] Test coverage gaps identified and documented
- [ ] Report file generated using `references/report-template.md`
- [ ] Coverage limitations explicitly stated in report
- [ ] User notified with summary and severity breakdown

---

## Self-Audit Protocol

Before declaring this skill's execution complete, verify ALL of the following:

- [ ] **Phase Gate: Blast Radius** -- For every HIGH-risk change, did you calculate the blast radius with concrete numbers? (Not "a few callers" -- exact count.)
- [ ] **Phase Gate: Git History** -- For removed/changed security code, did you check `git log` and `git blame`? Was any removed code originally added for a security fix?
- [ ] **Hard Stops checked** -- Were all Red Flag conditions evaluated? If any were triggered, were they treated as BLOCKING?
- [ ] **Evidence-based findings** -- Does every finding reference specific line numbers, commits, and code snippets? (Not "this could be bad" -- concrete evidence.)
- [ ] **Attack scenarios concrete** -- For HIGH-risk findings, did you model specific exploit steps? (Not "an attacker could exploit this" -- step-by-step scenario.)
- [ ] **Test coverage gaps documented** -- Did you identify which HIGH-risk changes lack test coverage?
- [ ] **Coverage limitations stated** -- Did you explicitly document what was NOT reviewed and why?
- [ ] **Report file generated** -- Was the structured report written to a file (not just chat output)?

**If any checkbox is unchecked, you are not done.** Go back and complete it.

---

## Cross-Skill References

- `cipher-security` -- For broad security audits not tied to specific changes (app-wide posture)
- `sentry-find-bugs` -- When the change is a fix for a production crash (verify the fix is secure)
- `sentry-code-review` -- When Sentry bot has also commented on the same PR (combine findings)
- `prometheus-performance` -- When security changes have performance implications (crypto overhead, validation latency)
- `forge-development` -- For implementing security fixes following project development patterns

---

## Tips for Best Results

**Do:**
- Start with git blame for removed code
- Calculate blast radius early to prioritize
- Generate concrete attack scenarios with specific steps
- Reference specific line numbers and commits
- Be honest about coverage limitations
- Always generate the output file
- Check iOS-specific vulnerability patterns for every review

**Don't:**
- Skip git history analysis
- Make generic findings without evidence
- Claim full analysis when time-limited
- Forget to check test coverage
- Miss high blast radius changes
- Output report only to chat (file required)
- Assume "internal" code is safe

---

## References

- iOS-specific vulnerability patterns: `references/vulnerability-patterns.md`
- iOS attack scenarios: `references/attack-scenarios.md`
- Report output template: `references/report-template.md`
- [Trail of Bits Skills](https://github.com/trailofbits/skills)
- [Trail of Bits Blog](https://blog.trailofbits.com/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
- [OWASP Mobile Security Testing Guide](https://mas.owasp.org/MASTG/)

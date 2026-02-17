---
name: cipher-security
description: "This skill should be used when the user asks to 'run a security scan', 'check for vulnerabilities', 'audit secrets management', 'review compliance', 'check OWASP', 'PCI DSS audit', 'KYC compliance review', or 'AML check'. Provides comprehensive security assessment, fintech compliance validation, and secure development guidance."
allowed-tools:
  - Read
  - Grep
  - Glob
user-invocable: true
---

# Security & Compliance

Assess application security posture, enforce compliance standards, and prevent credential exposure across iOS codebases and infrastructure.

> **Project Discovery**: Before executing, determine project-specific values (project name, secrets configuration files, gitignore patterns, backend infrastructure) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, .gitignore).

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
# Check for vulnerable dependencies
# Swift Package Manager doesn't have built-in audit
# Use manual review of Package.resolved
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
2. **Data Flow Analysis** -- Trace sensitive data from input through storage to transmission
3. **Authentication Review** -- Verify auth implementation against security best practices
4. **Encryption Audit** -- Validate cryptographic implementations and key management
5. **Compliance Check** -- Verify applicable compliance requirements (see `references/fintech-compliance.md`)
6. **iOS Security Review** -- Check platform-specific security (keychain, jailbreak, ATS)
7. **Dependency Audit** -- Review third-party libraries for known vulnerabilities

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

## Security Response

| Concern | Action |
|---------|--------|
| Vulnerability found | Document, assess severity, fix immediately |
| Data breach suspected | Incident response, notify owner |
| Compliance question | Research, document, escalate to owner |

## References

Detailed financial compliance checklists: `references/fintech-compliance.md`

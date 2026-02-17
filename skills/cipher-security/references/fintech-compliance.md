# Fintech Compliance Checklists

Detailed compliance checklists for iOS fintech and financial applications.

> **Project Discovery**: Determine which compliance standards apply to your specific application (e.g., money transfer, banking, insurance, trading) and jurisdiction. Not all checklists may apply.

## Authentication

- [ ] Secure password hashing (bcrypt/argon2)
- [ ] Multi-factor authentication support
- [ ] Session management (secure tokens)
- [ ] Logout all devices capability
- [ ] Account lockout after failed attempts
- [ ] Biometric authentication follows Apple Security guidelines
- [ ] Session tokens have appropriate expiration and rotation

## Data Protection

- [ ] Encryption at rest (AES-256)
- [ ] Encryption in transit (TLS 1.3)
- [ ] Keychain for sensitive storage (not UserDefaults or files)
- [ ] No PII in logs
- [ ] Data minimization
- [ ] Proper memory management for sensitive data (zeroing buffers)
- [ ] Key management practices documented

## Payment Security

- [ ] PCI DSS compliance (via payment processor tokenization)
- [ ] Tokenization for card data
- [ ] Fraud detection active
- [ ] Transaction limits enforced
- [ ] Suspicious activity monitoring

## Compliance Standards

### KYC (Know Your Customer)
- [ ] Identity verification system implemented
- [ ] Document verification workflow
- [ ] Re-verification triggers defined
- [ ] Verification status tracking

### AML (Anti-Money Laundering)
- [ ] Transaction monitoring active
- [ ] Suspicious activity reporting configured
- [ ] Velocity checks implemented
- [ ] Pattern analysis operational

### BSA (Bank Secrecy Act)
- [ ] Currency transaction reports for thresholds
- [ ] Record-keeping requirements met
- [ ] SAR filing procedures documented

### PCI DSS
- [ ] Card data never touches app servers (processor tokenization)
- [ ] No card numbers stored locally
- [ ] Secure network communication verified
- [ ] Access control policies enforced

### GDPR/CCPA
- [ ] Data subject rights implemented
- [ ] Privacy policy current and accessible
- [ ] Terms of service current
- [ ] Data retention policies defined
- [ ] Right to deletion supported

## OWASP Mobile Top 10 Detailed Checks

### M1: Improper Platform Usage
- Correct iOS API usage for security features
- Proper entitlement configuration
- No misuse of platform security mechanisms

### M2: Insecure Data Storage
- Keychain used for all credentials
- No sensitive data in UserDefaults, plist files, or SQLite
- Core Data encryption for sensitive records
- No sensitive data in backup files

### M3: Insecure Communication
- TLS 1.3 for all connections
- Certificate pinning enabled in production
- No cleartext traffic allowed (ATS enforced)
- Secure WebSocket connections

### M4: Insecure Authentication
- Strong password policy enforced
- Biometric authentication properly implemented
- Session management with secure token rotation
- No client-side authentication decisions

### M5: Insufficient Cryptography
- Industry-standard algorithms only (AES-256, RSA-2048+)
- No custom cryptographic implementations
- Secure key storage in Keychain
- Proper initialization vectors and salts

### M6: Insecure Authorization
- Server-side authorization (e.g., RLS policies, middleware guards)
- Role-based access control
- No client-side authorization bypass possible
- API endpoint authorization verified

### M7: Client Code Quality
- No force unwraps in production code
- Input validation on all user inputs
- Error handling without information leakage
- No stack traces or internal paths exposed

### M8: Code Tampering
- Jailbreak detection active
- Integrity checking for critical code paths
- Anti-debugging measures in production
- Tamper detection for sensitive operations

### M9: Reverse Engineering
- Code obfuscation for sensitive logic
- Anti-debugging protections
- String encryption for sensitive constants
- No embedded credentials

### M10: Extraneous Functionality
- No debug endpoints in production builds
- No test accounts in release builds
- No hidden admin features
- No verbose logging in production

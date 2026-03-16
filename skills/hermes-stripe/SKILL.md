---
name: hermes-stripe
description: "Test payments, Stripe CLI, webhook testing, test cards. Stripe CLI operations and payment flow testing — webhook forwarding, test event triggering, PaymentIntent creation, Connected Account management, test card reference. Trigger on: 'test a payment', 'forward webhooks locally', 'which test card for decline', 'stripe listen', 'stripe trigger', 'verify webhook delivery', 'test 3D Secure', 'simulate a refund', or any hands-on Stripe testing work. For Stripe integration architecture and API selection, use stripe-best-practices instead."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
user-invocable: true
---

# HERMES - Stripe Payment Testing & CLI Operations

> **Iron Law:** "Test every payment path with test cards before touching live mode. Every. Single. Path."

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, Stripe key locations, webhook endpoints) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift, .env, or xcconfig files).

Stripe payment testing skill covering systematic test card usage, webhook debugging, CLI operations, PaymentSheet verification, Connect marketplace testing, subscription lifecycle testing, and payment flow validation.

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|----------------|-----------------|
| "I tested the happy path, that's enough" | Payment integrations have 20+ distinct scenarios: success, multiple decline codes, 3DS authentication, webhook race conditions, refunds, disputes, network timeouts, currency edge cases. Testing only success covers roughly 5% of real-world scenarios. The other 95% fail in production with real money. | Use the complete testing matrix in `references/testing-checklist.md`. Every card type, every decline code, every 3DS variant. No exceptions. |
| "Webhooks work the same in test and live" | Test mode webhooks arrive faster, have no latency variation, and never fail due to network issues. Live mode introduces: delivery delays (up to minutes), out-of-order delivery, duplicate delivery, signature timing issues from clock skew, and IP allowlist requirements. | Test webhook retry behavior. Verify idempotent processing. Test signature verification with intentionally wrong secrets. Simulate out-of-order delivery. |
| "I'll test edge cases later" | Edge cases in payments are not edge cases -- they are real scenarios that happen daily at scale. A declined card is not an edge case; it is 5-15% of all transactions. 3DS challenges are not optional; they are legally required in the EU. "Later" means "after a customer loses money." | Test edge cases first. Declines, 3DS, insufficient funds, expired cards, and processing errors are all part of the primary flow, not afterthoughts. |
| "Manual dashboard checks are sufficient" | The Stripe Dashboard shows you what happened. It does not tell you whether your code handled it correctly. You need to verify that your database updated, your user saw the right message, your webhook processed idempotently, and your error recovery worked. The Dashboard is a monitoring tool, not a test suite. | Automate verification. After each test card transaction, check your database state, verify webhook processing, confirm user-facing UI updates, and validate error messages programmatically. |
| "3DS testing isn't necessary for our market" | Strong Customer Authentication (SCA) is mandatory in the EU/EEA and increasingly adopted worldwide. Even in markets without SCA requirements, issuers may request 3DS at their discretion. If your integration cannot handle a 3DS challenge, payments will fail silently. | Test with all 3DS cards: `4000002760003184` (required), `4000002500003155` (optional), `4000008260003178` (always authenticate). Verify your app handles the authentication flow correctly. |
| "The CLI trigger command is good enough for webhook testing" | `stripe trigger` sends pre-built fixtures with synthetic data. Real webhooks contain actual object IDs, nested expansions, metadata, and timing that fixtures do not replicate. Triggers verify your endpoint is reachable; they do not verify your business logic handles real payloads correctly. | Use `stripe trigger` for connectivity testing. Then create real test transactions with test cards and verify the actual webhook payloads your code receives match your expectations. |
| "We only need to test in USD" | Currency handling bugs are among the most common payment integration issues. Zero-decimal currencies (JPY, KRW) use different amount semantics. Some currencies have three decimal places (KWD, BHD). Multi-currency display, rounding, and conversion all need verification. | Test with at least: USD (standard), EUR (SCA market), JPY (zero-decimal), GBP (different symbol). Verify amounts display correctly in each. |
| "Refund testing can wait until we need it" | You will need refunds. Every payment integration does. If you have not tested refunds, you will discover bugs when a real customer needs their money back -- the worst possible time to debug. Partial refunds, full refunds, and refund-after-dispute all behave differently. | Test full refund, partial refund, and multiple partial refunds. Verify your database state, user notification, and webhook handling for each. |

---

## Red Flags -- STOP

- **Testing only success scenarios**: If your test plan contains only `4242424242424242`, you have not tested payments. You have tested one payment. Stop and add decline cards, 3DS cards, and error scenarios before proceeding.
- **Not verifying webhook signatures**: If your webhook endpoint processes events without calling `stripe.webhooks.constructEvent()` with the endpoint signing secret, your endpoint accepts forged events from anyone. This is a security vulnerability, not a convenience shortcut.
- **Skipping 3DS authentication testing**: If you have not tested with `4000002760003184` (3DS required), your integration will fail for any customer whose issuer requests authentication. This is legally required in the EU and increasingly common globally.
- **Not testing decline scenarios**: If you have not verified what your user sees when their card is declined, you have not tested payments. Decline handling is the most user-visible part of your payment flow.
- **Going to live mode without completing the testing checklist**: Live mode uses real money. Every untested path is a path where real customers can lose real money, generate disputes, or encounter broken flows. Complete `references/testing-checklist.md` first.
- **Ignoring webhook delivery failures in the CLI output**: When `stripe listen` shows delivery failures or signature mismatches, those are real bugs. Do not dismiss them as "CLI issues."

---

## When NOT to Use This Skill

- **Designing Stripe integration architecture**: For choosing between CheckoutSessions vs PaymentIntents, designing Connect marketplace flows, or selecting charge types, see **stripe-best-practices**. This skill tests implementations; that skill designs them.
- **Apple In-App Purchase testing**: Digital goods and subscriptions sold within iOS apps must use StoreKit, not Stripe. For IAP testing, see **cupertino-apple**.
- **Security review of payment handling code**: For reviewing PCI compliance, secret management, or security vulnerabilities in payment code, see **cipher-security**.
- **Database schema for payment records**: For designing tables that store payment records, webhook events, or subscription state, see **supabase-postgres-best-practices** and **atlas-database**.
- **Production monitoring of live payments**: This skill is for pre-deployment testing. For monitoring live payment flows in production, use Stripe Dashboard alerts and your application monitoring stack.

---

## Decision Framework

```
What am I trying to test?

WEBHOOK CONNECTIVITY
  |-- "Is my endpoint reachable?"
  |     YES --> stripe listen --forward-to <URL>
  |             Then: stripe trigger payment_intent.succeeded
  |     Endpoint responds 200? --> Connectivity confirmed
  |     Endpoint responds 4xx/5xx? --> Check endpoint code, URL path, port
  |     No response? --> Check if server is running, check port, check firewall
  |
  |-- "Is my webhook processing correct?"
        --> Create REAL test transactions (not just triggers)
        --> Use test cards via PaymentSheet or CLI
        --> Verify database state after webhook processes
        --> Verify idempotent processing (send same event twice)

WEBHOOK NOT ARRIVING? (Debugging Tree)
  |-- Is `stripe listen` running?
  |     NO  --> Start it: stripe listen --forward-to <URL>
  |     YES --> Continue
  |
  |-- Does `stripe listen` show the event?
  |     NO  --> Event was not fired
  |     |      --> Did the triggering action complete? Check Stripe Dashboard > Events
  |     |      --> Is the event type registered? Check Dashboard > Webhooks > Event types
  |     |      --> Are you on the right Stripe account? Check `stripe config --list`
  |     YES --> Continue
  |
  |-- Does `stripe listen` show a delivery attempt?
  |     NO  --> Event type not in the listen filter
  |     |      --> Remove --events flag or add the missing event type
  |     YES --> Continue
  |
  |-- What status code does the delivery show?
        200 --> Webhook received. Check your handler code for processing bugs.
        400 --> Signature verification failed. Check STRIPE_WEBHOOK_SECRET matches
               the signing secret from `stripe listen` output (starts with whsec_)
        404 --> Wrong URL path. Verify --forward-to URL matches your route.
        500 --> Handler crashed. Check server logs for the exception.
        Timeout --> Handler too slow. Process async, return 200 immediately.

PAYMENT FLOW TESTING
  |-- What flow am I testing?
  |     ONE-TIME --> Use SOP-4, test cards from matrix
  |     SUBSCRIPTION --> Use SOP-6, test full lifecycle
  |     CONNECT/MARKETPLACE --> Use SOP-5, test payouts
  |
  |-- For each flow:
        1. Test success path (4242 4242 4242 4242)
        2. Test decline path (4000 0000 0000 0002)
        3. Test 3DS path (4000 0027 6000 3184)
        4. Test insufficient funds (4000 0000 0000 9995)
        5. Verify webhook received and processed
        6. Verify database state updated correctly
        7. Verify user-facing UI shows correct state
        8. Verify error messages are user-friendly

CUSTOM EVENT SIMULATION
  |-- Need an event not in `stripe trigger` built-in list?
  |     --> Create the actual Stripe object via CLI or API
  |     --> The creation/update will fire real webhook events
  |     --> Example: create a PaymentIntent, then cancel it
  |         to get payment_intent.canceled event
  |
  |-- Need to test event payloads with specific data?
        --> Use stripe fixtures (JSON fixture files)
        --> stripe trigger with --override flag
        --> Or create objects with specific metadata via CLI
```

---

## Standard Operating Procedures

### SOP-1: Start Webhook Forwarding

```bash
# Basic: Forward all Stripe webhooks to local server
stripe listen --forward-to localhost:3000/webhook

# With specific profile
stripe listen --project-name <CLI_PROFILE> --forward-to <WEBHOOK_URL>

# Forward only specific events (reduces noise)
stripe listen --forward-to localhost:3000/webhook \
  --events payment_intent.succeeded,payment_intent.payment_failed,checkout.session.completed,customer.subscription.created,customer.subscription.deleted,invoice.payment_succeeded,invoice.payment_failed

# IMPORTANT: Copy the webhook signing secret from the output (whsec_xxx)
# and set it as STRIPE_WEBHOOK_SECRET in your .env
```

### SOP-2: Trigger Built-In Test Events

```bash
# One-time payment events
stripe trigger payment_intent.succeeded
stripe trigger payment_intent.payment_failed
stripe trigger charge.succeeded
stripe trigger charge.failed
stripe trigger charge.refunded

# Checkout events
stripe trigger checkout.session.completed

# Subscription events
stripe trigger customer.subscription.created
stripe trigger customer.subscription.updated
stripe trigger customer.subscription.deleted
stripe trigger invoice.payment_succeeded
stripe trigger invoice.payment_failed

# Connect events
stripe trigger account.updated
stripe trigger payout.paid
stripe trigger payout.failed

# Dispute events
stripe trigger charge.dispute.created
stripe trigger charge.dispute.closed
```

### SOP-3: Custom Event Simulation (Beyond Built-In Triggers)

```bash
# For events not available via `stripe trigger`, create real objects:

# Simulate payment_intent.canceled
stripe payment_intents create --amount 1000 --currency usd
# Copy the pi_xxx ID
stripe payment_intents cancel pi_xxx
# --> Fires payment_intent.canceled webhook

# Simulate payment_intent.requires_action (3DS)
stripe payment_intents create \
  --amount 1000 \
  --currency usd \
  --payment-method pm_card_authenticationRequired \
  --confirm
# --> Fires payment_intent.requires_action webhook

# Simulate refund events
stripe payment_intents create --amount 5000 --currency usd \
  --payment-method pm_card_visa --confirm
# Copy the pi_xxx ID
stripe refunds create --payment-intent pi_xxx --amount 2500
# --> Fires charge.refunded with partial refund

# Simulate subscription trial ending
stripe subscriptions create \
  --customer cus_xxx \
  --items[0][price] price_xxx \
  --trial-end $(date -v+1S +%s)
# --> Trial ends in 1 second, fires customer.subscription.trial_will_end
```

### SOP-4: Test Payment Flows with Test Cards

```bash
# Success payment
stripe payment_intents create \
  --amount 1000 \
  --currency usd \
  --payment-method pm_card_visa \
  --confirm

# Declined payment
stripe payment_intents create \
  --amount 1000 \
  --currency usd \
  --payment-method pm_card_chargeDeclined \
  --confirm

# Insufficient funds
stripe payment_intents create \
  --amount 1000 \
  --currency usd \
  --payment-method pm_card_chargeDeclinedInsufficientFunds \
  --confirm

# 3DS required
stripe payment_intents create \
  --amount 1000 \
  --currency usd \
  --payment-method pm_card_authenticationRequired \
  --confirm

# Specific decline code testing (see references/testing-checklist.md for full matrix)
```

### SOP-5: Connected Account Operations

```bash
# List connected accounts
stripe accounts list

# Create test connected account
stripe accounts create \
  --type express \
  --country US \
  --email test@example.com

# Check account status
stripe accounts retrieve acct_xxx

# Create onboarding link
stripe account_links create \
  --account acct_xxx \
  --refresh-url https://example.com/reauth \
  --return-url https://example.com/return \
  --type account_onboarding

# Create test payout on connected account
stripe payouts create \
  --amount 1000 \
  --currency usd \
  --stripe-account acct_xxx

# Test destination charge
stripe payment_intents create \
  --amount 10000 \
  --currency usd \
  --payment-method pm_card_visa \
  --confirm \
  --transfer-data[destination] acct_xxx \
  --transfer-data[amount] 9000
```

### SOP-6: Subscription Lifecycle Testing

```bash
# 1. Create a test customer
stripe customers create --email test@example.com --name "Test User"
# Copy cus_xxx

# 2. Create a subscription with trial
stripe subscriptions create \
  --customer cus_xxx \
  --items[0][price] price_xxx \
  --default-payment-method pm_card_visa \
  --trial-period-days 7

# 3. Cancel at period end
stripe subscriptions update sub_xxx \
  --cancel-at-period-end true

# 4. Cancel immediately
stripe subscriptions cancel sub_xxx

# 5. Test payment failure on renewal
# Use a card that will fail on the next charge:
stripe customers update cus_xxx \
  --invoice-settings[default-payment-method] pm_card_chargeDeclined
# Then trigger an invoice:
stripe invoices create --customer cus_xxx --subscription sub_xxx
stripe invoices pay inv_xxx
```

### SOP-7: View and Debug Events

```bash
# List recent events
stripe events list --limit 10

# Get specific event details
stripe events retrieve evt_xxx

# Tail events in real-time (separate from listen)
stripe events tail

# Filter events by type
stripe events list --type payment_intent.succeeded --limit 5

# Check webhook delivery attempts in Dashboard
# Dashboard > Developers > Webhooks > Select endpoint > Recent deliveries
```

---

## Stripe Dashboard <-> CLI Correlation

| CLI Action | Dashboard Location | What to Verify |
|------------|-------------------|----------------|
| `stripe listen` output | Developers > Webhooks > Logs | Event delivery status matches CLI output |
| `stripe trigger <event>` | Developers > Events | Event appears with correct type and data |
| `stripe payment_intents create` | Payments > All payments | PaymentIntent appears with correct amount/status |
| `stripe customers create` | Customers | Customer created with correct email |
| `stripe subscriptions create` | Billing > Subscriptions | Subscription active with correct plan |
| `stripe refunds create` | Payments > select payment > Refund section | Refund appears with correct amount |
| `stripe accounts create` | Connect > Accounts | Connected account appears |
| `stripe payouts create` | Connect > select account > Payouts | Payout appears with correct amount |

After running CLI commands, always cross-check in Dashboard to verify:
1. The object was created with expected values
2. Associated events were generated
3. Webhook delivery succeeded (check Developers > Webhooks)
4. Object status matches expectations

---

## Test Card Quick Reference

| Card | Number | Use Case |
|------|--------|----------|
| Visa (success) | 4242 4242 4242 4242 | Successful payment |
| Visa (decline) | 4000 0000 0000 0002 | Generic decline |
| Visa (insufficient) | 4000 0000 0000 9995 | Insufficient funds |
| Visa (expired) | 4000 0000 0000 0069 | Expired card |
| Visa (CVC fail) | 4000 0000 0000 0127 | Incorrect CVC |
| Visa (processing error) | 4000 0000 0000 0119 | Processing error |
| Visa (3DS required) | 4000 0027 6000 3184 | 3D Secure required |
| Visa (3DS optional) | 4000 0025 0000 3155 | 3D Secure optional |
| Visa (always auth) | 4000 0082 6000 3178 | Always authenticate |
| Mastercard (success) | 5555 5555 5555 4444 | Successful MC payment |
| AMEX (success) | 3782 822463 10005 | Successful AMEX payment |

For the complete testing matrix with expected results and webhook events, see `references/testing-checklist.md`.

---

## Webhook Events to Handle

| Event | When | Action | Verify |
|-------|------|--------|--------|
| `payment_intent.succeeded` | Payment completed | Update order status to paid | DB record updated, user notified |
| `payment_intent.payment_failed` | Payment failed | Mark order failed, notify user | Error message shown, retry available |
| `payment_intent.canceled` | Payment canceled | Release held resources | Order status reverted |
| `checkout.session.completed` | Checkout done | Process order or subscription | Correct items fulfilled |
| `customer.subscription.created` | New subscription | Activate premium features | Feature flags enabled |
| `customer.subscription.updated` | Plan changed | Update feature access level | Correct tier applied |
| `customer.subscription.deleted` | Cancelled | Revoke premium features | Access removed at period end |
| `invoice.payment_succeeded` | Renewal paid | Extend access period | New period_end recorded |
| `invoice.payment_failed` | Renewal failed | Notify user, grace period | Banner shown, retry scheduled |
| `charge.dispute.created` | Dispute opened | Gather evidence, freeze funds | Evidence deadline tracked |
| `charge.refunded` | Refund processed | Update order, credit user | Refund amount correct |
| `account.updated` | Connect account change | Update onboarding status | Requirements checked |
| `payout.paid` | Payout sent | Mark transfer complete | Connected account notified |
| `payout.failed` | Payout failed | Investigate, retry or notify | Failure reason logged |

---

## Phase Gate: TEST COVERAGE ✓

**Cannot declare payment testing complete until:**
- [ ] Success path tested (4242 4242 4242 4242)
- [ ] Decline path tested (4000 0000 0000 0002)
- [ ] 3D Secure path tested (4000 0027 6000 3184)
- [ ] Webhook delivery verified for each test
- [ ] Insufficient funds tested (4000 0000 0000 9995)

**Hard Stop**: Testing only the success path is not testing. The most common production payment issues are declines and 3DS — test them.

### Self-Audit

1. Did I test decline scenarios (not just success)?
2. Did I verify webhooks are arriving and being processed correctly?
3. Are webhook signatures being verified (not just in production — in test too)?
4. Did I test the 3D Secure flow end-to-end?

---

## Quality Gates (Before Marking Complete)

1. All test cards from `references/testing-checklist.md` have been tested for the implemented flow
2. Webhook forwarding verified with `stripe listen` -- events received and processed
3. Webhook signature verification confirmed working (test with wrong secret to verify rejection)
4. All decline scenarios produce user-friendly error messages (not raw Stripe errors)
5. 3DS authentication flow tested and working end-to-end (required + optional cards)
6. Idempotent webhook processing verified (same event sent twice produces same result)
7. Database state verified after each test scenario (not just UI state)
8. Subscription lifecycle fully tested: create, renew, fail, cancel, upgrade/downgrade
9. Connect payouts tested if marketplace flow is implemented
10. Dashboard cross-checked against CLI output for every test scenario

---

## Cross-Skill References

- **stripe-best-practices**: Stripe integration design patterns, API selection (CheckoutSessions vs PaymentIntents), Connect charge type decisions, webhook security patterns, and deprecated API migration. Design the integration there, test it here.
- **cipher-security**: PCI compliance review, secret key management, webhook signature security, and payment data handling. Use for security audits of payment code.
- **atlas-database**: Database migrations for payment-related tables (orders, subscriptions, webhook_events, stripe_customers). Use when creating or modifying payment data schemas.
- **supabase-postgres-best-practices**: Query optimization for payment data, RLS policies on financial tables, and connection pooling for webhook handlers.
- **forge-development**: Data access layer patterns for payment repositories. Payment queries should go through the data access layer, not direct DB calls.
- **cupertino-apple**: StoreKit In-App Purchase for digital goods. Stripe cannot be used for digital goods sold within iOS apps per App Store rules.

---

## References

- Complete testing matrix: `references/testing-checklist.md`
- Payment flow implementations: `../stripe-best-practices/references/payment-flows.md`
- [Stripe Testing Documentation](https://docs.stripe.com/testing)
- [Stripe CLI Reference](https://docs.stripe.com/cli)
- [Stripe Webhook Best Practices](https://docs.stripe.com/webhooks/best-practices)
- [Test Card Numbers](https://docs.stripe.com/testing#cards)
- [Stripe Go Live Checklist](https://docs.stripe.com/get-started/checklist/go-live)

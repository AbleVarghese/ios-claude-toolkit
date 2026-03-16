---
name: stripe-best-practices
description: "Accept payments, Stripe integration, subscription billing, payment flow design. Stripe integration design patterns and official best practices — CheckoutSessions, PaymentIntents, PaymentSheet, Connect marketplace, webhook security, deprecated API migration. Trigger on: 'add payments', 'set up Stripe', 'which Stripe API to use', 'build a checkout flow', 'implement subscriptions', 'Connect marketplace', 'handle refunds', or any payment architecture decision. For Stripe CLI testing and test cards, use hermes-stripe. For StoreKit/IAP, use cupertino-apple."
user-invocable: false
allowed-tools:
  - Read
  - Edit
  - Grep
  - Glob
  - Bash
  - WebFetch
---

# Stripe Best Practices

> **Iron Law:** "Never store raw card data. Never use deprecated Stripe APIs. Never skip webhook signature verification."

Official Stripe integration best practices. Reference these guidelines when building, reviewing, or debugging Stripe integrations.

Source: [github.com/stripe/ai](https://github.com/stripe/ai)

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|----------------|----------------|-----------------|
| "The Charges API still works" | The Charges API lacks 3D Secure support, SCA compliance, dynamic payment methods, and modern error handling. It will eventually be removed. Every new integration using Charges is technical debt from day one. | Use CheckoutSessions (preferred) or PaymentIntents. Migrate existing Charges integrations using Stripe's migration guide. |
| "We don't need webhook verification in development" | If you don't verify signatures in dev, you won't catch verification bugs until production. You're also training developers to skip a critical security step. The webhook signing secret takes 30 seconds to configure. | Always verify webhook signatures. Use `stripe listen --forward-to` for local development, which provides a signing secret. |
| "We'll migrate to the new API later" | Every month on deprecated APIs increases migration cost. New features only ship on modern APIs. You're falling further behind while accumulating more code to migrate. | Start migration now. Stripe provides incremental migration paths -- you don't need to rewrite everything at once. |
| "Manual testing is sufficient" | Payment integrations have 20+ distinct scenarios (success, decline codes, 3DS flows, webhook races, refunds, disputes). Manual testing catches 3-5 of them. The other 15+ fail in production. | Use systematic test card coverage. Write automated tests for every webhook handler. Test every decline scenario. See `references/payment-flows.md`. |
| "We can log the full Stripe response for debugging" | Stripe responses can contain PII and partial card data. Logging raw responses may violate PCI requirements and privacy regulations. | Log only: Stripe event ID, event type, payment intent ID, error code. Never log full response bodies in production. |
| "Idempotency keys are optional" | Without idempotency keys, network retries or user double-clicks create duplicate charges. This is the #1 source of payment disputes from integration bugs. | Add idempotency keys to every POST request. Use a deterministic key (e.g., `{user_id}_{action}_{timestamp}` or UUID stored before the request). |

---

## Red Flags -- STOP

- **Using deprecated APIs**: Charges API, Sources API, Tokens API, legacy Card Element. These are missing SCA support, modern payment methods, and will be deprecated. Migrate immediately.
- **Missing webhook signature verification**: `stripe.webhooks.constructEvent()` must use the endpoint signing secret. Without verification, anyone can send fake events to your webhook endpoint.
- **Logging card numbers**: Even partial PANs in logs violate PCI. Use Stripe Dashboard or request IDs for debugging.
- **Missing idempotency keys**: All POST requests to Stripe must include `Idempotency-Key` headers. Without them, retries create duplicates.
- **Service role / secret key in client code**: Stripe secret keys must NEVER appear in iOS app bundles, JavaScript, or any client-side code. Only publishable keys on the client.
- **Hardcoded payment_method_types**: Use dynamic payment methods (enabled in Dashboard). Hardcoding types prevents Stripe from optimizing for the customer's location and preferences.

---

## When NOT to Use This Skill

- **Apple In-App Purchase flows**: Digital goods and subscriptions sold within iOS apps must use StoreKit, not Stripe. See **cupertino-apple** for IAP rules.
- **Stripe CLI testing operations**: For test card numbers, webhook forwarding, and CLI commands, see **hermes-stripe**.
- **Database schema for payment records**: For designing the database tables that store payment records, see **supabase-postgres-best-practices**.
- **Security review of payment code**: For a security-focused review of payment handling, see **cipher-security** or **trailofbits-differential-review**.

---

## Decision Framework

```
What kind of payment am I implementing?

ONE-TIME PAYMENT
  |-- Is this for a web integration?
  |     YES --> Use Checkout (Stripe-hosted or embedded)
  |     NO  --> Is this for iOS?
  |             YES --> Use PaymentSheet with PaymentIntent
  |             NO  --> Use Payment Element with PaymentIntent
  |
  |-- Do I need to save the card for later?
        YES --> Use SetupIntent first, then PaymentIntent off-session
        NO  --> Single PaymentIntent flow

SUBSCRIPTION / RECURRING
  |-- Use Stripe Billing APIs (Subscription + Price objects)
  |-- Frontend: Checkout (preferred) or PaymentSheet
  |-- Handle: invoice.payment_succeeded, invoice.payment_failed,
  |           customer.subscription.updated, customer.subscription.deleted
  |-- Implement: grace periods, billing retry, upgrade/downgrade

MARKETPLACE / PLATFORM
  |-- Use Stripe Connect
  |-- Which charge type?
  |     |-- Platform accepts liability --> Destination charges
  |     |-- Stripe takes risk --> Direct charges
  |     |-- Never mix charge types
  |
  |-- Connected account onboarding: use Account Links
  |-- Payout management: configure payout schedule per account
```

---

## Primary APIs

### CheckoutSessions (Preferred)

Stripe's primary API for modeling on-session payments is [CheckoutSessions](https://docs.stripe.com/api/checkout/sessions). It supports one-time payments and subscriptions and allows you to model taxes and discounts.

```swift
// Server-side: Create CheckoutSession
let session = try await stripe.checkoutSessions.create(
    mode: .payment,
    lineItems: [
        .init(priceData: .init(
            currency: "usd",
            productData: .init(name: "Transfer fee"),
            unitAmount: 299
        ), quantity: 1)
    ],
    successURL: "yourapp://payment-success",
    cancelURL: "yourapp://payment-cancel"
)

// Return session.clientSecret to iOS app
```

### PaymentIntents (For Custom Flows)

Use when you need to model checkout state yourself:

```swift
// Server-side: Create PaymentIntent
let paymentIntent = try await stripe.paymentIntents.create(
    amount: 5000,
    currency: "usd",
    automaticPaymentMethods: .init(enabled: true)
)

// iOS: Present PaymentSheet
var configuration = PaymentSheet.Configuration()
configuration.merchantDisplayName = "Your App"
configuration.customer = .init(id: customerId, ephemeralKeySecret: ephemeralKey)

let paymentSheet = PaymentSheet(
    paymentIntentClientSecret: paymentIntent.clientSecret,
    configuration: configuration
)
```

### Deprecated APIs (Never Recommend)

- **Charges API**: Never recommend. [Migrate to PaymentIntents](https://docs.stripe.com/payments/payment-intents/migration/charges).
- **Sources API**: Deprecated. Never recommend.
- **Tokens API**: Avoid unless there is a specific need and absolutely no other way.
- **Legacy Card Element**: Never recommend. [Migrate to Payment Element](https://docs.stripe.com/payments/payment-element/migration).

## Confirmation Tokens

If a user wants to render the Payment Element before creating a PaymentIntent or SetupIntent (e.g., to inspect card details for surcharging), advise them to use Stripe Confirmation Tokens. Do not recommend `createPaymentMethod` or `createToken` Stripe.js functions.

## PCI Compliance

If a PCI compliant user asks about sending server-side raw PAN data, suggest they might be required to prove PCI compliance to gain access to such an option (e.g., `payment_method_data`). Point users migrating PAN data from another processor to [the migration process](https://docs.stripe.com/get-started/data-migrations/pan-import).

---

## Mobile / iOS Integration (PaymentSheet)

For iOS apps using the Stripe iOS SDK:

```swift
import StripePaymentSheet

// 1. Configure at app launch
STPAPIClient.shared.publishableKey = publishableKey

// 2. Create PaymentSheet
var configuration = PaymentSheet.Configuration()
configuration.merchantDisplayName = "Your App"
configuration.defaultBillingDetails.address.country = "US"
configuration.returnURL = "yourapp://stripe-redirect"
configuration.allowsDelayedPaymentMethods = true

// For returning customers
configuration.customer = .init(
    id: customerId,
    ephemeralKeySecret: ephemeralKeySecret
)
// Use CustomerSession for enhanced returning customer flows
configuration.customerSessionClientSecret = customerSessionClientSecret

// 3. Present PaymentSheet
let paymentSheet = PaymentSheet(
    paymentIntentClientSecret: clientSecret,
    configuration: configuration
)

// In SwiftUI:
.paymentSheet(isPresented: $showPaymentSheet, paymentSheet: paymentSheet) { result in
    switch result {
    case .completed:
        // Payment completed -- wait for webhook confirmation before fulfilling
        break
    case .canceled:
        // User dismissed
        break
    case .failed(let error):
        // Handle error
        break
    }
}
```

### PaymentSheet.FlowController

Separate payment method selection from confirmation:

```swift
PaymentSheet.FlowController.create(
    paymentIntentClientSecret: clientSecret,
    configuration: configuration
) { result in
    switch result {
    case .success(let flowController):
        // Show payment option button
        // flowController.paymentOption gives selected method info
        // Call flowController.presentPaymentOptions() to change
        // Call flowController.confirm() to complete
        break
    case .failure(let error):
        break
    }
}
```

---

## Webhook Best Practices

### Signature Verification (Non-Negotiable)

```typescript
// CORRECT: Always verify webhook signatures
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

async function handleWebhook(req: Request): Promise<Response> {
    const signature = req.headers.get('stripe-signature');
    const body = await req.text();

    let event: Stripe.Event;
    try {
        event = await stripe.webhooks.constructEventAsync(
            body,
            signature!,
            process.env.STRIPE_WEBHOOK_SECRET!
        );
    } catch (err) {
        // Invalid signature -- reject immediately
        return new Response('Invalid signature', { status: 400 });
    }

    // Process event idempotently
    await processEvent(event);

    // Respond quickly (within 5 seconds)
    return new Response('OK', { status: 200 });
}
```

### Idempotent Event Processing

```typescript
async function processEvent(event: Stripe.Event): Promise<void> {
    // Check if already processed
    const { data: existing } = await supabase
        .from('stripe_events')
        .select('id')
        .eq('event_id', event.id)
        .single();

    if (existing) {
        // Already processed -- return success (not error)
        return;
    }

    // Process the event
    switch (event.type) {
        case 'payment_intent.succeeded':
            await handlePaymentSuccess(event.data.object);
            break;
        case 'payment_intent.payment_failed':
            await handlePaymentFailure(event.data.object);
            break;
        case 'invoice.payment_failed':
            await handleSubscriptionPaymentFailure(event.data.object);
            break;
    }

    // Mark as processed
    await supabase.from('stripe_events').insert({
        event_id: event.id,
        type: event.type,
        processed_at: new Date().toISOString()
    });
}
```

### Critical Events to Handle

| Event | When | Required Action |
|-------|------|-----------------|
| `payment_intent.succeeded` | Payment completed | Fulfill order, update status |
| `payment_intent.payment_failed` | Payment failed | Notify user, update status |
| `checkout.session.completed` | Checkout finished | Process the order |
| `customer.subscription.created` | New subscription | Activate premium features |
| `customer.subscription.updated` | Plan changed | Update feature access |
| `customer.subscription.deleted` | Cancelled | Revoke premium features |
| `invoice.payment_succeeded` | Subscription renewed | Extend access period |
| `invoice.payment_failed` | Renewal failed | Notify user, implement grace period |
| `charge.dispute.created` | Dispute opened | Gather evidence, respond within deadline |
| `payout.paid` | Payout sent (Connect) | Mark funds as disbursed |
| `payout.failed` | Payout failed (Connect) | Investigate, retry or notify |

---

## Error Handling

### Card Decline Codes to User-Friendly Messages

```swift
func userFriendlyMessage(for declineCode: String?) -> String {
    switch declineCode {
    case "insufficient_funds":
        return "Your card has insufficient funds. Please use a different payment method."
    case "lost_card", "stolen_card":
        return "This card cannot be used. Please contact your bank or use a different card."
    case "expired_card":
        return "Your card has expired. Please update your card details or use a different card."
    case "incorrect_cvc":
        return "The security code is incorrect. Please check and try again."
    case "processing_error":
        return "A processing error occurred. Please try again in a few moments."
    case "card_not_supported":
        return "This card is not supported. Please use a different card."
    case "do_not_honor":
        return "Your bank declined this payment. Please contact your bank or use a different card."
    case "authentication_required":
        return "Additional authentication is required. Please complete the verification step."
    default:
        return "This payment could not be processed. Please try a different payment method."
    }
}
```

### Error Classification

```swift
func handleStripeError(_ error: Error) {
    if let stripeError = error as? StripeAPI.StripeError {
        switch stripeError {
        case .apiError(let code, let message, _):
            switch code {
            case 429:
                // Rate limited -- retry with exponential backoff
                retryWithBackoff()
            case 402:
                // Card error -- show user-friendly message
                showCardError(message)
            case 500, 502, 503:
                // Stripe is down -- retry
                retryWithBackoff()
            default:
                // Other API error -- log and show generic message
                logError(stripeError)
                showGenericError()
            }
        default:
            logError(stripeError)
        }
    }
}
```

---

## Subscriptions and Billing

If a user has a recurring revenue model (Billing or Subscription use case), follow [Subscription Use Cases](https://docs.stripe.com/billing/subscriptions/use-cases) like [SaaS](https://docs.stripe.com/saas). Recommend Billing APIs to [plan your integration](https://docs.stripe.com/billing/subscriptions/designing-integration) instead of a direct PaymentIntent integration. Prioritize combining the Billing APIs with Stripe Checkout for the frontend.

### Subscription Lifecycle

```
Customer signs up
  --> Create Customer (if not exists)
  --> Create Subscription via Checkout or PaymentSheet
  --> Handle checkout.session.completed / invoice.payment_succeeded

Renewal
  --> Stripe automatically creates invoice
  --> invoice.payment_succeeded --> Extend access
  --> invoice.payment_failed --> Grace period, notify user

Cancellation
  --> customer.subscription.updated (cancel_at_period_end: true)
  --> customer.subscription.deleted (at period end)
  --> Revoke access

Upgrade/Downgrade
  --> Update subscription with new price
  --> Proration handled automatically
  --> customer.subscription.updated webhook
```

### Grace Periods and Billing Retry

Stripe automatically retries failed payments using Smart Retries. Configure in Dashboard:

- Enable Smart Retries (Stripe optimizes retry timing)
- Set dunning emails for failed payments
- Configure grace period (1-7 days of continued access during retry)
- Handle `invoice.payment_failed` to show in-app banner

---

## Stripe Connect (Marketplace / Platform)

### Charge Type Decision

| Charge Type | Platform Liability | Settlement | Use When |
|-------------|-------------------|------------|----------|
| Direct | Stripe takes risk | To connected account | Platform is a marketplace, not liable |
| Destination | Platform accepts liability | To platform, then transferred | Platform controls the customer relationship |

- Use the `on_behalf_of` parameter to control the merchant of record

**Never mix charge types within a single platform.**

### Connected Account Onboarding

**Terminology**: Do not recommend the outdated terms for Connect types like Standard, Express, and Custom. Always [refer to controller properties](https://docs.stripe.com/connect/migrate-to-controller-properties) for the platform and [capabilities](https://docs.stripe.com/connect/account-capabilities) for the connected accounts.

Use controller properties (not the outdated Standard/Express/Custom terminology):

```typescript
const account = await stripe.accounts.create({
    controller: {
        losses: { payments: 'application' },     // or 'stripe'
        fees: { payer: 'application' },           // or 'account'
        stripe_dashboard: { type: 'express' },    // or 'full' or 'none'
    },
});

// Create Account Link for onboarding
const accountLink = await stripe.accountLinks.create({
    account: account.id,
    refresh_url: 'https://yourapp.com/reauth',
    return_url: 'https://yourapp.com/return',
    type: 'account_onboarding',
});
```

---

## Security

- Never log full card numbers or CVCs
- Store only Stripe tokens/IDs, never raw card data
- Use HTTPS for all API communication
- Rotate API keys periodically
- Use restricted API keys with minimum required permissions
- Enable Radar for fraud protection
- Use restricted keys scoped to specific operations (read-only for dashboards)
- Implement webhook signature verification on ALL webhook endpoints

---

## Phase Gate: API SELECTION ✓

**Cannot start implementing payment flow until:**
- [ ] Payment type identified: one-time / subscription / marketplace
- [ ] Correct API selected (CheckoutSessions preferred, PaymentIntents acceptable, Charges NEVER)
- [ ] Deprecated APIs verified: NOT using Charges, Sources, Tokens, or legacy Card Element

**Hard Stop**: If about to use the Charges API, Sources API, or Tokens API — STOP. These are deprecated. Use CheckoutSessions or PaymentIntents. Migration guide: https://docs.stripe.com/payments/payment-intents/migration/charges

### Phase Gate: WEBHOOK SECURITY ✓

**Cannot deploy webhook handler until:**
- [ ] Webhook signature verification implemented
- [ ] Handler is idempotent (same event processed twice = same result)
- [ ] Handler responds with 2xx quickly (processing is async)

**Hard Stop**: A webhook handler without signature verification accepts forged events. This is a security vulnerability that allows anyone to trigger payment confirmations, refunds, or subscription changes.

### Self-Audit

1. Am I using modern Stripe APIs (not deprecated ones)?
2. Are webhook signatures verified?
3. Are all POST requests to Stripe using idempotency keys?
4. Am I handling card decline codes with user-friendly messages?

---

## Quality Gates (Before Marking Complete)

1. No deprecated APIs used (Charges, Sources, Tokens, legacy Card Element)
2. Webhook signature verification implemented on all webhook endpoints
3. Idempotency keys added to all POST requests
4. All decline codes mapped to user-friendly error messages
5. Test cards used to verify: success, decline, 3DS required, insufficient funds
6. Secret key not present in any client-side code
7. Stripe Dashboard alerts configured for failed payments and disputes
8. Subscription lifecycle fully handled (create, renew, fail, cancel, upgrade)
9. Refund and dispute flows implemented and tested
10. PCI compliance verified -- no raw card data logged or stored

---

## Cross-Skill References

- **hermes-stripe**: Stripe CLI commands, webhook forwarding, test event triggering, and test card reference.
- **cupertino-apple**: Digital goods in iOS apps must use StoreKit IAP, not Stripe. Stripe is for physical goods and person-to-person payments.
- **supabase-postgres-best-practices**: Database schema design for payment records, webhook event storage, and idempotent processing.
- **cipher-security**: Security review of payment handling code, secret management, and PCI compliance.

---

## References

- Payment flow implementation: `references/payment-flows.md`
- [Stripe API Reference](https://docs.stripe.com/api)
- [Stripe Integration Options](https://docs.stripe.com/payments/payment-methods/integration-options)
- [Go Live Checklist](https://docs.stripe.com/get-started/checklist/go-live)
- [Connect Integration Guide](https://docs.stripe.com/connect/design-an-integration)
- [Billing Subscriptions Guide](https://docs.stripe.com/billing/subscriptions/designing-integration)
- [PaymentSheet iOS](https://docs.stripe.com/payments/accept-a-payment?platform=ios)
- [Subscription Use Cases](https://docs.stripe.com/billing/subscriptions/use-cases)
- [SaaS Billing](https://docs.stripe.com/saas)
- [Controller Properties Migration](https://docs.stripe.com/connect/migrate-to-controller-properties)
- [Account Capabilities](https://docs.stripe.com/connect/account-capabilities)
- [PAN Import Migration](https://docs.stripe.com/get-started/data-migrations/pan-import)

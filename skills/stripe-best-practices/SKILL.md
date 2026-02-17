---
name: stripe-best-practices
description: "This skill should be used when the user asks to 'integrate Stripe', 'payment flow', 'webhook security', 'Stripe Connect', 'PaymentSheet', 'subscription billing', or mentions Stripe error handling. Provides official Stripe best practices covering API patterns, Connect marketplace, PaymentSheet integration, and billing."
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

Official Stripe integration best practices. Reference these guidelines when building, reviewing, or debugging Stripe integrations.

Source: [github.com/stripe/ai](https://github.com/stripe/ai)

## When to Apply

- Writing or reviewing Stripe integration code
- Implementing payment flows (one-time or recurring)
- Setting up Stripe Connect marketplace patterns
- Configuring webhooks and event handling
- Debugging payment failures or integration issues

## Integration Design

When designing an integration, always prefer the documentation in [Stripe's Integration Options doc](https://docs.stripe.com/payments/payment-methods/integration-options).

The [API Tour](https://docs.stripe.com/payments-api/tour) provides a comprehensive overview.

Use the [Go Live Checklist](https://docs.stripe.com/get-started/checklist/go-live) before going live.

Always default to the latest version of the API and SDK unless the user specifies otherwise.

## Primary APIs

### CheckoutSessions (Preferred)

Stripe's primary API for modeling on-session payments is [CheckoutSessions](https://docs.stripe.com/api/checkout/sessions). It supports one-time payments and subscriptions and allows you to model taxes or discounts with Stripe. Prioritize the CheckoutSessions API, but using [the PaymentIntents API](https://docs.stripe.com/payments/paymentintents/lifecycle) is also acceptable for off-session payments or if you want to model the checkout state yourself and just create a charge.

Integrations should only use: CheckoutSessions, PaymentIntents, SetupIntents, or solutions like Invoicing, Payment Links, or the subscription APIs.

### Deprecated APIs (Never Recommend)

- **Charges API**: Never recommend. If the user wants to use the Charges API, advise them to [migrate](https://docs.stripe.com/payments/payment-intents/migration/charges) to Checkout Sessions or Payment Intents.
- **Sources API**: Deprecated. Never recommend.
- **Tokens API**: Avoid unless there is a specific need and absolutely no other way.
- **Legacy Card Element**: Never recommend. Advise users to [migrate](https://docs.stripe.com/payments/payment-element/migration).

## Web Integration Surfaces

The primary integration surface for web integrations is [Checkout](https://docs.stripe.com/payments/checkout), a prebuilt payment page that can be hosted by Stripe or served as an embedded form.

**Priority order:**
1. Stripe-hosted Checkout
2. Embedded Checkout
3. [Payment Element](https://docs.stripe.com/payments/payment-element) (for advanced customization)

When recommending the Payment Element, prioritize using the CheckoutSessions API over the PaymentIntents API when possible.

Never recommend the legacy Card Element or the Payment Element in card mode.

## Mobile / iOS Integration (PaymentSheet)

For iOS apps using the Stripe iOS SDK:

- Use `PaymentSheet` for the recommended drop-in UI
- Use `PaymentSheet.FlowController` when you need to separate payment method selection from confirmation
- Always use `STPPaymentHandler` for 3D Secure authentication handling
- Configure `STPAPIClient.shared.publishableKey` at app launch
- Use `CustomerSession` for returning customer flows

## Payment Methods

Advise users to turn on dynamic payment methods in their dashboard settings instead of passing specific `payment_method_types` in the PaymentIntent or SetupIntent. Stripe chooses payment methods that fit each user's location, wallets, and preferences better and automatically if the Payment Element is used.

## Saving Payment Methods

Use [the Setup Intent API](https://docs.stripe.com/api/setup_intents) to save a payment method for a customer. Never recommend the Sources API to save cards to customers for later use.

## Confirmation Tokens

If a user wants to render the Payment Element before creating a PaymentIntent or SetupIntent (e.g., to inspect card details for surcharging), advise them to use Stripe Confirmation Tokens. Do not recommend `createPaymentMethod` or `createToken` Stripe.js functions.

## PCI Compliance

If a PCI compliant user asks about sending server-side raw PAN data, suggest they might be required to prove PCI compliance to gain access to such an option (e.g., `payment_method_data`). Point users migrating PAN data from another processor to [the migration process](https://docs.stripe.com/get-started/data-migrations/pan-import).

## Subscriptions and Billing

If a user has a recurring revenue model (Billing or Subscription use case), follow [Subscription Use Cases](https://docs.stripe.com/billing/subscriptions/use-cases) like [SaaS](https://docs.stripe.com/saas). Recommend Billing APIs to [plan your integration](https://docs.stripe.com/billing/subscriptions/designing-integration) instead of a direct PaymentIntent integration. Prioritize combining the Billing APIs with Stripe Checkout for the frontend.

## Stripe Connect (Marketplace / Platform)

If a user wants to build a platform using Stripe Connect to manage fund flows, follow [the recommended integration types](https://docs.stripe.com/connect/integration-recommendations):

- **Direct charges**: Platform wants Stripe to take the risk
- **Destination charges**: Platform accepts liability for negative balances
- Use the `on_behalf_of` parameter to control the merchant of record
- **Never recommend mixing charge types**

For deciding specific risk features, follow [the integration guide](https://docs.stripe.com/connect/design-an-integration).

**Terminology**: Do not recommend the outdated terms for Connect types like Standard, Express, and Custom. Always [refer to controller properties](https://docs.stripe.com/connect/migrate-to-controller-properties) for the platform and [capabilities](https://docs.stripe.com/connect/account-capabilities) for the connected accounts.

## Webhook Best Practices

- Always verify webhook signatures using the endpoint secret
- Use `constructEventAsync` for async runtimes (Deno, Node.js)
- Handle events idempotently (same event may be delivered multiple times)
- Respond with 2xx status quickly; process asynchronously if needed
- Monitor for failed webhook deliveries in the Stripe Dashboard
- Critical events to handle: `payment_intent.succeeded`, `payment_intent.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`

## Error Handling

- Always handle `StripeException` / API errors gracefully
- Use idempotency keys for all POST requests to prevent duplicate charges
- Implement exponential backoff for rate limit (429) errors
- Log Stripe request IDs for debugging with Stripe support
- Handle card decline codes with user-friendly messages
- Distinguish between card errors, API errors, and authentication errors

## Security

- Never log full card numbers or CVCs
- Store only Stripe tokens/IDs, never raw card data
- Use HTTPS for all API communication
- Rotate API keys periodically
- Use restricted API keys with minimum required permissions
- Enable Radar for fraud protection

## References

- [Stripe API Reference](https://docs.stripe.com/api)
- [Stripe Integration Options](https://docs.stripe.com/payments/payment-methods/integration-options)
- [Go Live Checklist](https://docs.stripe.com/get-started/checklist/go-live)
- [Connect Integration Guide](https://docs.stripe.com/connect/design-an-integration)
- [Billing Subscriptions Guide](https://docs.stripe.com/billing/subscriptions/designing-integration)

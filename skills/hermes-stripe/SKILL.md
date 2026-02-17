---
name: hermes-stripe
description: "This skill should be used when the user asks to 'test a stripe payment', 'forward webhooks', 'create a subscription', 'manage connected accounts', or 'debug payment intent'. Provides Stripe CLI operations, payment flow testing, webhook management, and Connect account orchestration."
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - WebFetch
user-invocable: true
---

# HERMES - Stripe Payment Integration

> **Project Discovery:** Before executing, determine project-specific values (project name, scheme, bundle ID, Stripe key locations, webhook endpoints) from project configuration files (CLAUDE.md, project.yml, .xcodeproj, Package.swift, .env, or xcconfig files).

Stripe payment integration skill covering PaymentSheet, Connect marketplace, subscriptions, payouts, webhook forwarding, and test event generation.

## Standard Operating Procedures

### SOP-1: Start Webhook Forwarding
```bash
# Forward Stripe webhooks to local server
stripe listen --forward-to localhost:3000/webhook

# Forward to a specific endpoint (e.g., Supabase Edge Function, Express, etc.)
stripe listen --project-name <CLI_PROFILE> --forward-to <WEBHOOK_URL>
```

### SOP-2: Trigger Test Events
```bash
# Trigger payment succeeded
stripe trigger payment_intent.succeeded

# Trigger checkout completed
stripe trigger checkout.session.completed

# Trigger subscription events
stripe trigger customer.subscription.created
stripe trigger invoice.payment_succeeded
```

### SOP-3: View Recent Events
```bash
# List recent events
stripe events list --limit 10

# Get specific event details
stripe events retrieve evt_xxx

# Tail events in real-time
stripe events tail
```

### SOP-4: Test Payment Flows
```bash
# Create test PaymentIntent
stripe payment_intents create \
  --amount 1000 \
  --currency usd \
  --payment-method pm_card_visa \
  --confirm

# Create test subscription
stripe subscriptions create \
  --customer cus_xxx \
  --items[0][price] price_xxx
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

# Create payout
stripe payouts create \
  --amount 1000 \
  --currency usd \
  --stripe-account acct_xxx
```

## Test Card Numbers

| Card | Number | Use Case |
|------|--------|----------|
| Visa (success) | 4242 4242 4242 4242 | Successful payment |
| Visa (decline) | 4000 0000 0000 0002 | Generic decline |
| Visa (insufficient) | 4000 0000 0000 9995 | Insufficient funds |
| Visa (3DS required) | 4000 0027 6000 3184 | 3D Secure required |
| Visa (3DS optional) | 4000 0025 0000 3155 | 3D Secure optional |

## Webhook Events to Handle

| Event | When | Action |
|-------|------|--------|
| `payment_intent.succeeded` | Payment completed | Update transfer status |
| `payment_intent.payment_failed` | Payment failed | Mark transfer failed |
| `checkout.session.completed` | Checkout done | Process subscription |
| `customer.subscription.created` | New subscription | Activate premium |
| `customer.subscription.deleted` | Cancelled | Revoke premium |
| `account.updated` | Connect account change | Update status |
| `payout.paid` | Payout sent | Complete transfer |
| `payout.failed` | Payout failed | Handle failure |

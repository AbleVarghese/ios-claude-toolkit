# Stripe Payment Testing Checklist

Complete testing matrix for systematic payment verification. Every flow must be tested with every relevant card before going to live mode.

---

## One-Time Payment Cards

| Card Number | PM Token | Expected Result | Verify in App | Expected Webhook Events |
|-------------|----------|-----------------|---------------|------------------------|
| 4242 4242 4242 4242 | pm_card_visa | Success | Order status = paid, confirmation shown | `payment_intent.succeeded`, `charge.succeeded` |
| 5555 5555 5555 4444 | pm_card_mastercard | Success | Order status = paid, confirmation shown | `payment_intent.succeeded`, `charge.succeeded` |
| 3782 822463 10005 | pm_card_amex | Success | Order status = paid, confirmation shown | `payment_intent.succeeded`, `charge.succeeded` |
| 4000 0000 0000 0002 | pm_card_chargeDeclined | Generic decline | User-friendly decline message, retry option | `payment_intent.payment_failed` |
| 4000 0000 0000 9995 | pm_card_chargeDeclinedInsufficientFunds | Insufficient funds | "Insufficient funds" message, suggest another card | `payment_intent.payment_failed` |
| 4000 0000 0000 9987 | pm_card_chargeDeclinedLostCard | Lost card | "Card cannot be used" message | `payment_intent.payment_failed` |
| 4000 0000 0000 9979 | pm_card_chargeDeclinedStolenCard | Stolen card | "Card cannot be used" message | `payment_intent.payment_failed` |
| 4000 0000 0000 0069 | pm_card_chargeDeclinedExpiredCard | Expired card | "Card expired" message, prompt update | `payment_intent.payment_failed` |
| 4000 0000 0000 0127 | pm_card_chargeDeclinedIncorrectCvc | Incorrect CVC | "Check security code" message | `payment_intent.payment_failed` |
| 4000 0000 0000 0119 | pm_card_chargeDeclinedProcessingError | Processing error | "Try again" message with retry | `payment_intent.payment_failed` |
| 4000 0000 0000 0101 | pm_card_cvcCheckFail | CVC check fails | Payment may succeed but CVC check flagged | `payment_intent.succeeded` (with CVC check fail) |

---

## 3D Secure (SCA) Cards

| Card Number | PM Token | Expected Result | Verify in App | Expected Webhook Events |
|-------------|----------|-----------------|---------------|------------------------|
| 4000 0027 6000 3184 | pm_card_authenticationRequired | 3DS required, succeeds on auth | 3DS challenge presented, payment succeeds after auth | `payment_intent.requires_action`, then `payment_intent.succeeded` |
| 4000 0025 0000 3155 | pm_card_authenticationRequiredOnSetup | 3DS optional, succeeds | May or may not show 3DS challenge | `payment_intent.succeeded` |
| 4000 0082 6000 3178 | pm_card_authenticationRequiredChargeDeclinedInsufficientFunds | 3DS required, fails after auth | 3DS challenge presented, then decline after auth | `payment_intent.requires_action`, then `payment_intent.payment_failed` |
| 4000 0000 0000 3220 | pm_card_threeDSecure2Required | 3DS2 required | 3DS2 modal flow presented | `payment_intent.requires_action`, then `payment_intent.succeeded` |
| 4000 0000 0000 3063 | pm_card_threeDSecureRequired | 3DS1 required | 3DS1 redirect flow | `payment_intent.requires_action`, then `payment_intent.succeeded` |

---

## Subscription Lifecycle Testing

### Phase 1: Creation

| Scenario | Card / Setup | Expected Result | Verify | Webhook Events |
|----------|-------------|-----------------|--------|----------------|
| New subscription (no trial) | 4242... + valid price_id | Subscription active immediately | Premium features enabled, billing date set | `customer.subscription.created`, `invoice.payment_succeeded` |
| New subscription (with trial) | 4242... + trial_period_days | Subscription trialing | Trial end date shown, features enabled | `customer.subscription.created` (status: trialing) |
| Failed initial payment | 4000000000000002 + price_id | Subscription incomplete | Error shown, retry prompted | `invoice.payment_failed`, `customer.subscription.created` (status: incomplete) |

### Phase 2: Renewal

| Scenario | Setup | Expected Result | Verify | Webhook Events |
|----------|-------|-----------------|--------|----------------|
| Successful renewal | Active sub with 4242... | Subscription renewed | Period end extended, access continues | `invoice.payment_succeeded`, `customer.subscription.updated` |
| Failed renewal (1st attempt) | Switch to 4000000000000341 | Payment fails, retries scheduled | Grace period starts, user notified | `invoice.payment_failed` |
| Failed renewal (final) | Keep failing card | Subscription canceled after retries | Access revoked, cancellation email | `customer.subscription.deleted` |

### Phase 3: Changes

| Scenario | Action | Expected Result | Verify | Webhook Events |
|----------|--------|-----------------|--------|----------------|
| Upgrade plan | Update sub with higher price_id | Proration applied, new plan active | New features available, prorated charge | `customer.subscription.updated`, `invoice.payment_succeeded` |
| Downgrade plan | Update sub with lower price_id | Credit applied, new plan at next period | Features adjusted at period end | `customer.subscription.updated` |
| Cancel at period end | Set cancel_at_period_end: true | Access continues until period end | Cancellation date shown, features still active | `customer.subscription.updated` |
| Cancel immediately | Cancel subscription | Access revoked immediately | Features disabled, refund if applicable | `customer.subscription.deleted` |
| Reactivate after cancel | Remove cancel_at_period_end | Cancellation reversed | Cancellation notice removed | `customer.subscription.updated` |

### Phase 4: Trial Management

| Scenario | Setup | Expected Result | Verify | Webhook Events |
|----------|-------|-----------------|--------|----------------|
| Trial ends, payment succeeds | Trial sub + 4242... | Converts to paid | Billing starts, access continues | `customer.subscription.trial_will_end`, `invoice.payment_succeeded`, `customer.subscription.updated` |
| Trial ends, payment fails | Trial sub + 4000000000000002 | Subscription past_due or canceled | Access revoked or grace period | `customer.subscription.trial_will_end`, `invoice.payment_failed` |

---

## Connect Payout Testing

| Scenario | Setup | Expected Result | Verify | Webhook Events |
|----------|-------|-----------------|--------|----------------|
| Successful payout | Connected account with balance | Payout sent to bank | Payout status = paid in Dashboard | `payout.paid` |
| Failed payout | Connected account, invalid bank | Payout fails | Failure reason logged, account notified | `payout.failed` |
| Destination charge | pi with transfer_data[destination] | Payment split between platform and connected account | Platform fee retained, connected account credited | `payment_intent.succeeded`, `transfer.created` |
| Direct charge | pi on connected account with app fee | Payment on connected account | Application fee collected by platform | `payment_intent.succeeded` (on connected account), `application_fee.created` |
| Account onboarding complete | New Express account, complete onboarding | Account charges_enabled = true | Account appears active in Connect dashboard | `account.updated` |
| Account requirements due | Incomplete onboarding | Account has requirements.currently_due | Prompt shown to complete onboarding | `account.updated` |

---

## Refund Testing

| Scenario | Setup | Expected Result | Verify | Webhook Events |
|----------|-------|-----------------|--------|----------------|
| Full refund | Successful payment, then full refund | Full amount returned | Order status = refunded, user notified | `charge.refunded` |
| Partial refund | Successful payment, then partial refund | Partial amount returned | Refund amount shown, order partially refunded | `charge.refunded` (amount < original) |
| Multiple partial refunds | Successful payment, then 2+ partial refunds | Each refund processed | Running refund total tracked | Multiple `charge.refunded` events |
| Refund on dispute | Payment with dispute, then refund | Dispute auto-closed | Dispute status updated | `charge.refunded`, `charge.dispute.closed` |

---

## Pre-Go-Live Checklist

Before switching from test mode to live mode, confirm ALL of the following:

- [ ] All cards in "One-Time Payment Cards" section tested
- [ ] All cards in "3D Secure Cards" section tested
- [ ] Subscription creation, renewal, failure, and cancellation tested
- [ ] Decline error messages are user-friendly (not raw API errors)
- [ ] Webhook signature verification working (tested with wrong secret)
- [ ] Idempotent webhook processing verified (duplicate events handled)
- [ ] Connect payouts tested (if applicable)
- [ ] Refund flow tested (full and partial)
- [ ] Currency handling verified for all supported currencies
- [ ] Stripe Dashboard alerts configured for failures and disputes
- [ ] Live mode API keys stored securely (not in client code or version control)
- [ ] Webhook endpoint registered in Stripe Dashboard for live mode
- [ ] Rate limiting configured on webhook endpoint
- [ ] Error logging captures Stripe event IDs for debugging (not full payloads)

# Payment Flow Implementation Reference

Complete implementation patterns for Stripe payment flows with Swift (iOS) and server-side code.

---

## One-Time Payment

### Server: Create PaymentIntent

```typescript
// POST /api/create-payment-intent
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

async function createPaymentIntent(req: Request): Promise<Response> {
    const { amount, currency, customerId } = await req.json();

    // Validate input
    if (!amount || amount <= 0) {
        return Response.json({ error: 'Invalid amount' }, { status: 400 });
    }

    const paymentIntent = await stripe.paymentIntents.create({
        amount: Math.round(amount), // Amount in cents
        currency: currency || 'usd',
        customer: customerId,
        automatic_payment_methods: { enabled: true },
        metadata: {
            order_id: generateOrderId(),
        },
    }, {
        idempotencyKey: `pi_${customerId}_${amount}_${Date.now()}`,
    });

    // Also create/refresh ephemeral key for PaymentSheet
    const ephemeralKey = await stripe.ephemeralKeys.create(
        { customer: customerId },
        { apiVersion: '2024-12-18.acacia' }
    );

    return Response.json({
        clientSecret: paymentIntent.client_secret,
        ephemeralKey: ephemeralKey.secret,
        customerId: customerId,
    });
}
```

### iOS: Present PaymentSheet

```swift
import StripePaymentSheet

final class PaymentManager: ObservableObject {
    @Published var paymentSheet: PaymentSheet?
    @Published var paymentResult: PaymentSheetResult?

    func preparePayment(amount: Int, currency: String = "usd") async throws {
        // 1. Fetch PaymentIntent from your server
        let response = try await APIClient.shared.createPaymentIntent(
            amount: amount,
            currency: currency
        )

        // 2. Configure PaymentSheet
        var configuration = PaymentSheet.Configuration()
        configuration.merchantDisplayName = "Your App"
        configuration.customer = .init(
            id: response.customerId,
            ephemeralKeySecret: response.ephemeralKey
        )
        configuration.returnURL = "yourapp://stripe-redirect"
        configuration.defaultBillingDetails.address.country = "US"
        configuration.applePay = .init(
            merchantId: "merchant.com.yourapp",
            merchantCountryCode: "US"
        )

        // 3. Create PaymentSheet
        await MainActor.run {
            self.paymentSheet = PaymentSheet(
                paymentIntentClientSecret: response.clientSecret,
                configuration: configuration
            )
        }
    }
}

// SwiftUI View
struct CheckoutView: View {
    @StateObject private var paymentManager = PaymentManager()

    var body: some View {
        VStack {
            // Payment details...

            if let paymentSheet = paymentManager.paymentSheet {
                PaymentSheet.PaymentButton(
                    paymentSheet: paymentSheet,
                    onCompletion: handlePaymentResult
                ) {
                    Text("Pay $50.00")
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(Color.blue)
                        .foregroundColor(.white)
                        .cornerRadius(12)
                }
            }
        }
        .task {
            try? await paymentManager.preparePayment(amount: 5000)
        }
    }

    private func handlePaymentResult(_ result: PaymentSheetResult) {
        switch result {
        case .completed:
            // Payment completed on client side
            // Wait for webhook to confirm server-side before fulfilling
            showSuccessPending()
        case .canceled:
            // User dismissed PaymentSheet
            break
        case .failed(let error):
            showError(error.localizedDescription)
        }
    }
}
```

### Webhook: Handle payment_intent.succeeded

```typescript
// Webhook handler
case 'payment_intent.succeeded': {
    const paymentIntent = event.data.object as Stripe.PaymentIntent;

    // Fulfill the order
    await supabase.from('orders').update({
        status: 'paid',
        stripe_payment_intent_id: paymentIntent.id,
        paid_at: new Date().toISOString(),
    }).eq('id', paymentIntent.metadata.order_id);

    // Send confirmation to user
    await sendPaymentConfirmation(paymentIntent);
    break;
}

case 'payment_intent.payment_failed': {
    const paymentIntent = event.data.object as Stripe.PaymentIntent;
    const error = paymentIntent.last_payment_error;

    await supabase.from('orders').update({
        status: 'payment_failed',
        failure_reason: error?.message || 'Unknown error',
    }).eq('id', paymentIntent.metadata.order_id);

    // Notify user
    await sendPaymentFailureNotification(paymentIntent, error?.decline_code);
    break;
}
```

---

## Subscription Implementation

### Server: Create Subscription via Checkout

```typescript
// POST /api/create-subscription-checkout
async function createSubscriptionCheckout(req: Request): Promise<Response> {
    const { customerId, priceId, trialDays } = await req.json();

    const session = await stripe.checkout.sessions.create({
        mode: 'subscription',
        customer: customerId,
        line_items: [{ price: priceId, quantity: 1 }],
        subscription_data: trialDays ? {
            trial_period_days: trialDays,
        } : undefined,
        success_url: 'yourapp://subscription-success?session_id={CHECKOUT_SESSION_ID}',
        cancel_url: 'yourapp://subscription-cancel',
    });

    return Response.json({ url: session.url, clientSecret: session.client_secret });
}
```

### Webhook: Handle Subscription Lifecycle

```typescript
// Subscription created
case 'customer.subscription.created': {
    const subscription = event.data.object as Stripe.Subscription;
    await activatePremium(subscription.customer as string, {
        subscriptionId: subscription.id,
        status: subscription.status,
        currentPeriodEnd: new Date(subscription.current_period_end * 1000).toISOString(),
        cancelAtPeriodEnd: subscription.cancel_at_period_end,
    });
    break;
}

// Invoice paid (renewal)
case 'invoice.payment_succeeded': {
    const invoice = event.data.object as Stripe.Invoice;
    if (invoice.subscription) {
        await extendPremiumAccess(
            invoice.customer as string,
            invoice.subscription as string,
            new Date(invoice.period_end * 1000).toISOString()
        );
    }
    break;
}

// Invoice payment failed (renewal failure)
case 'invoice.payment_failed': {
    const invoice = event.data.object as Stripe.Invoice;
    const attempt = invoice.attempt_count;

    if (attempt === 1) {
        // First failure: notify user
        await notifyPaymentFailed(invoice.customer as string, 'soft');
    } else if (attempt >= 3) {
        // Multiple failures: warn about cancellation
        await notifyPaymentFailed(invoice.customer as string, 'hard');
    }
    break;
}

// Subscription cancelled
case 'customer.subscription.deleted': {
    const subscription = event.data.object as Stripe.Subscription;
    await revokePremiumAccess(
        subscription.customer as string,
        subscription.id
    );
    break;
}

// Subscription updated (upgrade/downgrade)
case 'customer.subscription.updated': {
    const subscription = event.data.object as Stripe.Subscription;
    const previousAttributes = event.data.previous_attributes;

    if (previousAttributes?.items) {
        // Plan changed
        await updateSubscriptionPlan(
            subscription.customer as string,
            subscription.id,
            subscription.items.data[0].price.id
        );
    }

    if (subscription.cancel_at_period_end) {
        // User scheduled cancellation
        await markCancellationScheduled(
            subscription.customer as string,
            new Date(subscription.current_period_end * 1000).toISOString()
        );
    }
    break;
}
```

### Handling Upgrade/Downgrade

```typescript
// POST /api/change-subscription-plan
async function changeSubscriptionPlan(req: Request): Promise<Response> {
    const { subscriptionId, newPriceId } = await req.json();

    const subscription = await stripe.subscriptions.retrieve(subscriptionId);

    const updatedSubscription = await stripe.subscriptions.update(subscriptionId, {
        items: [{
            id: subscription.items.data[0].id,
            price: newPriceId,
        }],
        proration_behavior: 'create_prorations', // or 'none' or 'always_invoice'
    });

    return Response.json({
        status: updatedSubscription.status,
        currentPeriodEnd: updatedSubscription.current_period_end,
    });
}
```

---

## Connect Marketplace

### Destination Charges (Platform Accepts Liability)

```typescript
// Platform creates the payment, funds go to connected account
const paymentIntent = await stripe.paymentIntents.create({
    amount: 10000,  // $100
    currency: 'usd',
    customer: platformCustomerId,
    transfer_data: {
        destination: connectedAccountId,
        amount: 9000,  // $90 goes to connected account
                        // $10 stays with platform as fee
    },
    automatic_payment_methods: { enabled: true },
});
```

### Direct Charges (Stripe Takes Risk)

```typescript
// Charge is created directly on connected account
const paymentIntent = await stripe.paymentIntents.create({
    amount: 10000,
    currency: 'usd',
    application_fee_amount: 1000,  // $10 platform fee
    automatic_payment_methods: { enabled: true },
}, {
    stripeAccount: connectedAccountId,
});
```

### Connected Account Onboarding

```typescript
// 1. Create connected account
const account = await stripe.accounts.create({
    controller: {
        losses: { payments: 'application' },
        fees: { payer: 'application' },
        stripe_dashboard: { type: 'express' },
    },
    country: 'US',
    email: 'seller@example.com',
});

// 2. Create onboarding link
const accountLink = await stripe.accountLinks.create({
    account: account.id,
    refresh_url: 'https://yourapp.com/connect/reauth',
    return_url: 'https://yourapp.com/connect/complete',
    type: 'account_onboarding',
});
// Redirect user to accountLink.url

// 3. Handle account.updated webhook
case 'account.updated': {
    const account = event.data.object as Stripe.Account;
    if (account.charges_enabled && account.payouts_enabled) {
        await markAccountActive(account.id);
    } else if (account.requirements?.currently_due?.length > 0) {
        await promptForAdditionalInfo(account.id, account.requirements.currently_due);
    }
    break;
}
```

### Payout Management

```typescript
// Manual payout (if not using automatic schedule)
const payout = await stripe.payouts.create({
    amount: 5000,
    currency: 'usd',
}, {
    stripeAccount: connectedAccountId,
});

// Handle payout events
case 'payout.paid': {
    const payout = event.data.object as Stripe.Payout;
    await markPayoutComplete(payout.id, event.account);
    break;
}

case 'payout.failed': {
    const payout = event.data.object as Stripe.Payout;
    await handlePayoutFailure(payout.id, event.account, payout.failure_message);
    break;
}
```

---

## Migration from Deprecated APIs

### Charges API to PaymentIntents

```typescript
// OLD (Charges API) -- DO NOT USE
const charge = await stripe.charges.create({
    amount: 5000,
    currency: 'usd',
    source: 'tok_visa',
});

// NEW (PaymentIntents API)
const paymentIntent = await stripe.paymentIntents.create({
    amount: 5000,
    currency: 'usd',
    automatic_payment_methods: { enabled: true },
    confirm: true,
    payment_method: 'pm_card_visa',
});
```

### Legacy Card Element to Payment Element

The Payment Element automatically handles:
- Card input
- 3D Secure authentication
- Dynamic payment method display
- Address collection
- Saved payment methods

Migration path:
1. Replace `CardElement` with `PaymentElement` in frontend
2. Replace `stripe.confirmCardPayment()` with `stripe.confirmPayment()`
3. Update server to use PaymentIntents if still on Charges
4. Enable dynamic payment methods in Dashboard

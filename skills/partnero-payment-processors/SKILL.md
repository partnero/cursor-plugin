---
name: partnero-payment-processors
description: Get referral attribution through Stripe, Paddle or Chargebee so Partnero records transactions automatically — Payment Links, Pricing Tables, Buy Buttons, Checkout Sessions, Paddle checkouts and Chargebee custom fields. Use when payments are handled by a processor connected to Partnero but commissions aren't being attributed.
---

# Attribution through a payment processor

When Stripe, Paddle or Chargebee is connected to Partnero, transactions and
refunds arrive on their own and you should not also post them to
`/v1/transactions` — that double counts.

What the connection cannot do by itself is know *who referred the buyer*. The
attribution key lives in a cookie on your site, and the checkout happens on the
processor's. Everything below is about carrying that key across.

The cookie is `partnero_partner` in an affiliate program and `partnero_referral`
in a refer-a-friend program. Confirm which with the MCP `get_program` tool —
`program.type` reads `Affiliate` or `Referral`.

Connections are made in the Partnero dashboard under Program → Integration.
Webhooks are created automatically; there is no manual webhook setup for Stripe
or Paddle.

## What Partnero looks for, in order

This is the part worth getting right. When the processor's webhook arrives,
Partnero tries a sequence of fields to work out who the buyer is. Knowing the
sequence tells you exactly what your backend has to attach.

**Stripe**, on `checkout.session.completed`:

1. `metadata.customer_key` on the session or charge — matched against an
   existing Partnero customer by key.
2. `metadata.customer_key` on the Stripe **Customer** object.
3. The billing email, matched against an existing customer.
4. `client_reference_id` on the session — treated as a **partner key**, not a
   customer key. Partnero creates the customer itself.
5. Billing email plus `client_reference_id`, when there's no Stripe customer.
6. `metadata.partner_key`, only when the program uses the transaction
   attribution model.

**Paddle**, on `transaction.completed`, the same shape with `custom_data` in
place of `metadata`:

1. `custom_data.customer_key` on the transaction.
2. `custom_data.customer_key` on the Paddle customer.
3. The billing email.
4. `custom_data.client_reference_id`, falling back to `custom_data.customer_key`
   — at this point read as a **partner key**.
5. `custom_data.partner_key`, in transaction attribution mode.

Note the trap in step 4: `customer_key` means your Partnero customer key if it
matches one, and is otherwise reinterpreted as a partner key. The frontend
snippets further down rely on that fallback. Don't put both meanings in play at
once.

Both processors also support a **custom field name** configured in the program's
integration settings, used in place of `customer_key`. If a project already
stores its user ID in Stripe metadata under some other name, map it there rather
than duplicating the field.

### The rule this gives you

If your backend already creates the customer in Partnero at sign-up, attach
`metadata.customer_key` (Stripe) or `custom_data.customer_key` (Paddle) with
**that same key** whenever you create a session, subscription or charge. That's
step 1, it's exact, and it needs no cookie at checkout time.

If you don't create customers yourself and rely entirely on the processor, use
`client_reference_id` with the partner key from the cookie. Partnero then makes
the customer for you — keyed `stripe_cus_XXXX` from the Stripe customer ID, not
by anything your app knows. That's why a Partnero program can end up full of
`stripe_`-prefixed customers: it means attribution is arriving through step 4
rather than step 1.

## Stripe

### Payment Links, Pricing Tables, Buy Buttons

One line, placed after the Partnero tracking script:

```html
<script>po('integration', 'stripe', null);</script>
```

That appends `client_reference_id` to every `buy.stripe.com` link, and sets the
`client-reference-id` attribute on every `<stripe-pricing-table>` and
`<stripe-buy-button>` element on the page. It also supports coupon
auto-inclusion through a `prefilled_promo_code` parameter.

If the Payment Links use a custom domain:

```html
<script>po('integration', 'stripe', { stripe_payment_links_domain: 'pay.yourdomain.com' });</script>
```

### Checkout Sessions

A session created from your backend has no access to the browser's cookie, so
pass it through in two steps.

Frontend — read the cookie and send it with the checkout request:

```javascript
const partnerKey = document.cookie.match(/(^| )partnero_partner=([^;]+)/)?.[2];

fetch('/api/create-checkout-session', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    priceId: 'price_xxx',
    partneroPartner: partnerKey ? decodeURIComponent(partnerKey) : null,
  }),
});
```

Backend — put it in `client_reference_id`:

```javascript
const session = await stripe.checkout.sessions.create({
  mode: 'subscription',
  line_items: [{ price: req.body.priceId, quantity: 1 }],
  success_url: 'https://yoursite.com/success',
  cancel_url: 'https://yoursite.com/cancel',
  client_reference_id: req.body.partneroPartner || undefined,
});
```

```python
session = stripe.checkout.Session.create(
    mode="subscription",
    line_items=[{"price": price_id, "quantity": 1}],
    success_url="https://yoursite.com/success",
    cancel_url="https://yoursite.com/cancel",
    client_reference_id=partnero_partner or None,
)
```

```php
$session = \Stripe\Checkout\Session::create([
    'mode' => 'subscription',
    'line_items' => [['price' => $priceId, 'quantity' => 1]],
    'success_url' => 'https://yoursite.com/success',
    'cancel_url' => 'https://yoursite.com/cancel',
    'client_reference_id' => $request->input('partneroPartner'),
]);
```

Partnero reads `client_reference_id` off the `checkout.session.completed` event.
A Checkout Session integration with no Payment Links or Buy Buttons doesn't need
`po('integration', 'stripe', null)` at all — just the `client_reference_id`.

### Subscriptions, invoices and anything created purely server-side

A session created by an API call from a cron job, an admin action or a
subscription renewal has no cookie behind it. `client_reference_id` isn't
available, so use metadata keyed to the customer you already have:

```javascript
await stripe.customers.update(stripeCustomerId, {
  metadata: { customer_key: yourInternalUserId },
});
```

Set it once on the Stripe Customer and every future charge for that customer
resolves through step 2 without any per-transaction work. This is the most
robust Stripe setup there is, and the one to recommend when a project creates
Stripe customers in its own code.

Per-session metadata does the same for one-off payments:

```javascript
await stripe.checkout.sessions.create({
  mode: 'payment',
  line_items: [{ price: priceId, quantity: 1 }],
  success_url: 'https://yoursite.com/success',
  cancel_url: 'https://yoursite.com/cancel',
  metadata: { customer_key: yourInternalUserId },
});
```

## Paddle

Paddle Classic, through the checkout link's data attributes:

```html
<script>
document.querySelectorAll('a.paddle_button').forEach((link) => {
  const cookieValue = (document.cookie.match(/(^| )partnero_partner=([^;]+)/) || [])[2];
  if (cookieValue) {
    link.setAttribute('data-custom-data', JSON.stringify({
      customer_key: decodeURIComponent(cookieValue),
    }));
  }
});
</script>
```

Paddle Billing, by wrapping the checkout call:

```html
<script>
const partnerKey = (document.cookie.match(/(^| )partnero_partner=([^;]+)/) || [])[2];
if (partnerKey) {
  const originalOpen = Paddle.Checkout.open;
  Paddle.Checkout.open = function (options) {
    options.customData = options.customData || {};
    options.customData.customer_key = decodeURIComponent(partnerKey);
    return originalOpen.call(this, options);
  };
}
</script>
```

Run this after the Paddle script has loaded, or `Paddle.Checkout` won't exist
yet.

Server-side, when you create a transaction or subscription through Paddle's API,
attach the customer key you already use in Partnero:

```javascript
await paddle.transactions.create({
  items: [{ priceId, quantity: 1 }],
  customData: { customer_key: yourInternalUserId },
});
```

Setting the same `custom_data` on the Paddle customer covers every future
renewal, the same way Stripe customer metadata does.

## Chargebee

Chargebee carries the key in a customer custom field, which has to exist before
the integration is connected. In Chargebee → Settings → Configure Chargebee →
Custom Fields, create a **Single Line Text** field on the **Customer** entity:

| Program type | API name | Label |
| --- | --- | --- |
| Affiliate | `cf_partnero_partner` | Partner Code |
| Refer-a-friend | `cf_partnero_referral` | Referral Code |

Then set it when creating the customer from your backend:

```javascript
chargebee.customer.create({
  email: customerEmail,
  cf_partnero_partner: partnerKeyFromCookie,
});
```

Partnero registers its own Chargebee webhook when you connect, subscribed to
`payment_succeeded`, `payment_refunded`, `transaction_created` and
`transaction_deleted`. On sites where Chargebee's API won't accept webhook
creation the connection still succeeds and the webhook has to be added by hand,
pointing at the URL in the integration settings. So if transactions never
arrive from a connected Chargebee, check that the webhook actually exists in
Chargebee before looking at custom fields.

## Verify

Complete a real checkout through a referral link, then call `list_transactions`
on the MCP server and confirm the sale is there with a `partner` on it.

The failure modes read differently:

- **No transaction at all** — the connection or its webhook isn't working, or
  the event type isn't one Partnero handles.
- **Transaction present, `partner` empty** — the key never reached the
  processor. That's your checkout code, not the connection.
- **Transaction present, attributed, but the customer is `stripe_cus_…`** —
  attribution arrived through `client_reference_id` rather than a customer key.
  It works, but that customer is not joined up with the one your app created.
  Switch to `metadata.customer_key` if you want one record per user.

Don't debug this by posting to `/v1/transactions` as well. A connected processor
already reports the sale, and a manual call on top double counts it.

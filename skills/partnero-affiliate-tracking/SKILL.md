---
name: partnero-affiliate-tracking
description: Add Partnero affiliate tracking to a website or app — the tracking script, customer creation on sign-up, transaction recording on payment, and commission reversal on refund. Use when someone wants partners or affiliates to earn commission on referred sign-ups and sales, or when existing Partnero tracking isn't attributing conversions.
---

# Partnero affiliate tracking

An affiliate program pays partners for customers they send you. Tracking has
three parts and all three must be present for a partner to get paid: the script
that captures the referral, the sign-up call that ties a customer to the partner,
and the transaction call that records the sale.

## Before you start

If the Partnero MCP server is connected, call `get_program` first. It gives you
everything except the API key:

- `program.type` — must be `Affiliate`. `Referral` means a refer-a-friend
  program, so use the `partnero-refer-a-friend` skill instead; `Newsletter`
  means none of this applies.
- `program.public_id` — the ID for the tracking snippet below.
- `program_settings.ref_link_param` and `ref_link_format` — what a referral URL
  looks like, which you'll need to test with.
- `program_settings.cookie_lifetime` — how many days attribution survives.

Without MCP, ask for the program ID from the snippet in the dashboard.

The API key is separate and never comes from MCP. It's in the dashboard under
**Integration → API**. Ask for it and put it in an environment variable — it
must not reach the repo or the client bundle.

Then find, in the user's codebase, the sign-up handler and the payment success
handler. Everything below hangs off those two places.

## 1. Install the tracking script

Before `</head>` on every page of the marketing site and the app, including the
sign-up page:

```html
<script>
(function(p,t,n,e,r,o){ p['__partnerObject']=r;function f(){
var c={ a:arguments,q:[]};var r=this.push(c);return "number"!=typeof r?r:f.bind(c.q);}
f.q=f.q||[];p[r]=p[r]||f.bind(f.q);p[r].q=p[r].q||f.q;o=t.createElement(n);
var _=t.getElementsByTagName(n)[0];o.async=1;o.src=e+'?v'+(~~(new Date().getTime()/1e6));
_.parentNode.insertBefore(o,_);})(window, document, 'script', 'https://app.partnero.com/js/universal.js', 'po');
po('settings', 'assets_host', 'https://assets.partnero.com');
po('program', 'PROGRAM_ID', 'load');
</script>
```

The script reads the referral key out of the incoming URL and stores it in a
first-party `partnero_partner` cookie. Without it on the landing page there is
nothing to attribute.

In a single-page app, run the same bootstrap once at the root — `next/script` with
`strategy="afterInteractive"` in Next.js, a `useEffect` in a plain React root —
and prefix the calls with `window.`.

## 2. Create the customer on sign-up

In the server-side sign-up handler, read the `partnero_partner` cookie off the
request and send it with the new customer:

```http
POST https://api.partnero.com/v1/customers
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json

{
  "partner": { "key": "value of the partnero_partner cookie" },
  "key": "your internal user id",
  "email": "user@example.com",
  "name": "First",
  "surname": "Last"
}
```

Only call this when the cookie is present. In an affiliate program a customer
exists in the context of a partner, so the API requires one of `partner.key`,
`partner.email` or `partner.id` and rejects the request without it. A partner
key that doesn't resolve to a real partner is rejected too. Both come back as
errors, not as a silently ignored no-op.

That makes the cookie read the load-bearing line. If it's missing on a visit
that really came through a referral link, the commission is gone and can't be
backfilled.

Social logins and OAuth flows must use this API call. The client-side helpers
can't see the redirect, so a JavaScript-only integration loses exactly the
sign-ups that go through Google or GitHub.

## 3. Record the transaction on payment

In the payment success handler — the webhook from the processor, not the browser
redirect:

```http
POST https://api.partnero.com/v1/transactions
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json

{
  "customer": { "key": "the same internal user id used at sign-up" },
  "key": "order or invoice id",
  "amount": 99.99,
  "action": "sale"
}
```

Optional fields: `amount_units` (a currency code — the program's currency is
assumed without it), `product_id` and `product_type` for per-product commission
rules, and `discount_amount`, `gross_amount` and `discount_code` where discounts
matter.

Because affiliate customers only exist when they were referred, a transaction
for an organic customer fails with "customer not found". That's expected, and
harmless as long as the call is non-blocking — don't add a lookup to avoid it.

**Send each payment once.** Transaction keys are not unique, so posting the
same order twice creates two transactions and two commissions; Partnero will
not collapse them for you. That matters most here, because payment processors
retry their webhooks on any non-2xx response and will happily redeliver an
event your handler already processed. Guard on your side — record that the
order was reported before calling Partnero, and make the guard survive a
retry.

If Stripe, Paddle or Chargebee is already connected to Partnero, skip this step —
the processor reports transactions on its own. Confirm with the user before
adding a call that would double-count. Sign-up tracking is still required.

## 4. Reverse commission on refund

After the refund goes through with the payment provider:

```http
DELETE https://api.partnero.com/v1/transactions/{order or invoice id}
Authorization: Bearer YOUR_API_KEY
```

This is the reason step 3 uses the order ID as the key.

## Verify

With the MCP server connected you can drive most of this yourself rather than
handing the user a checklist:

1. Call `list_partners` and take a real partner key. Build the referral URL
   from `ref_link_format` and `ref_link_param` — with `query` and `aff`, that's
   `https://their-site.com/?aff=THE_KEY`.
2. Ask the user to open that URL and confirm the `partnero_partner` cookie
   exists, then complete a sign-up.
3. Call `list_customers` and check the new customer is there, attached to that
   partner.
4. After a test purchase, call `list_transactions` and confirm the amount and
   the resulting reward.

If the cookie is set but step 3 finds nothing, the sign-up handler isn't reading
the cookie — the most common failure by a wide margin.

## Common mistakes

- Wrapping the API call so that a Partnero outage breaks sign-up. Always try/catch
  and continue.
- Calling `api.partnero.com` from the browser, which exposes the API key.
- Using a different customer key at sign-up and at payment, so sales attach to
  nobody.
- Treating a rejected customer create as a bug when the visitor simply wasn't
  referred. Log it and move on.

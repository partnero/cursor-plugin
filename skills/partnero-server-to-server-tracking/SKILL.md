---
name: partnero-server-to-server-tracking
description: Track Partnero referrals without a browser cookie, using server-registered click IDs and postback URLs — for affiliate networks, ad platforms, API-only products, and conversions that happen off-site. Use when someone mentions postbacks, S2S, click IDs, cookieless tracking, or an affiliate network that needs a conversion URL.
---

# Server-to-server tracking

Cookie tracking assumes a browser that lands on your site, keeps a cookie, and
comes back to sign up. When that assumption doesn't hold — an affiliate network
that redirects through its own domain, an ad platform, a product with no web
front end, a conversion that happens days later in another system — Partnero
can track the same relationship with a click ID instead.

You register the click yourself and get an ID back. You hold that ID. When the
conversion happens, you post it back.

## Before you start

This is **affiliate programs only**. Refer-a-friend programs have no postback
handler; a request for one is accepted and ignored.

Postback tracking must be switched on for the program, under **Integration →
Postbacks** in the dashboard. While it's off, click registration returns an
empty response and postbacks are discarded.

You need the program's public ID — `program.public_id` from the MCP
`get_program` tool. Both endpoints below are keyless: they're scoped by that ID
and carry no API key, so don't send one.

## 1. Register the click

When a visitor arrives through a partner's link — or when an affiliate network
hands your server a partner key — register it:

```http
POST https://api.partnero.com/pub/v1/s2s/clicks/{program_public_id}
Content-Type: application/json

{
  "partner": "the partner's key",
  "url": "https://yoursite.com/landing?aff=KEY&utm_source=..."
}
```

```json
{ "status": 1, "click_id": "generated identifier" }
```

`url` is optional. When present, Partnero pulls the program's trackable query
parameters out of it and stores them as sub IDs against the click, which is how
you get per-campaign breakdowns later.

**An empty response body means the click was not registered** — the program
isn't an affiliate program, postback tracking is off, or the partner key doesn't
match a partner. The endpoint returns 200 either way, so check for `click_id`
rather than the status code.

PartneroJS registers clicks through this same endpoint when it runs in a
browser, and keeps the result in a `partnero_click_id` cookie. So on a page that
loads the script you may already have a click ID without registering one: read
that cookie, and only register server-side when it's absent. Where the form is
one the script can see, adding `data-partnero` to it makes the script post the
click ID along with the referral key as hidden fields.

## 2. Hold the click ID

Store it against the visitor: a session, a short-lived record keyed by whatever
identifier you have, or a field on the pending signup. It plays exactly the role
the cookie plays in browser tracking, so it has to survive as long as your
attribution window — `program_settings.cookie_lifetime` days.

## 3. Post back the conversion

One endpoint, two events, `POST` or `GET`:

```http
POST https://api.partnero.com/pub/v1/s2s/postback/{program_public_id}
```

**A sign-up:**

```json
{
  "click_id": "the stored click ID",
  "event": "customer_create",
  "key": "your internal user id",
  "email": "user@example.com",
  "name": "Customer Name"
}
```

The partner comes from the click ID, so there's no partner field. If the
customer already exists, the existing one is returned rather than erroring.

**A sale:**

```json
{
  "click_id": "the stored click ID",
  "event": "transaction_create",
  "key": "order or invoice id",
  "amount": 99.99,
  "action": "sale"
}
```

`customer.key` is optional — without it, the transaction attaches to the
customer created against that same click ID. Send `options.create_customer` as
true if the sale can arrive before any sign-up was recorded.

Any other `event` value is ignored.

## 4. Expect silence

The postback endpoint returns `200 ok` no matter what happens — including when
the click ID is unknown, the event is unsupported, or validation fails. This is
deliberate, because the caller is often an ad network that would retry forever
on an error.

It means **you cannot tell success from failure at the call site.** Every
outcome is recorded instead, under **Integration → Postbacks** in the dashboard,
with a status of success, ignored or failed and the reason. Send the user there
when a postback appears to do nothing.

## Using the authenticated API instead

If you already call the REST API with a key, you don't need the keyless
endpoint. `POST /v1/customers` and `POST /v1/transactions` both accept
`options.postback_tracking_id`, which takes the click ID in place of a partner
or customer reference:

```json
{
  "key": "your internal user id",
  "email": "user@example.com",
  "options": { "postback_tracking_id": "the stored click ID" }
}
```

This is the better route when the conversion happens inside your own backend,
because you get real validation errors instead of a silent 200.

## Verify

Register a click with a real partner key and confirm you get a `click_id` back
rather than an empty object. Post back a `customer_create` with it, then check
the customer through `list_customers` on the MCP server — it should be attached
to that partner. If it isn't, the postback log in the dashboard will say why.

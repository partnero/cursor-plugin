---
name: partnero-refer-a-friend
description: Add Partnero refer-a-friend tracking to an app, including the in-app referral dashboard where users see their referral link, referred friends and reward balance. Use when existing customers should be able to refer new ones, or when building a referral page that reads a customer's link, stats, balance or referrals.
---

# Partnero refer-a-friend

In a refer-a-friend program every customer is also a potential referrer. That
single fact drives the whole integration: you create a Partnero customer for
everyone who signs up, not only for people who arrived through a link, because
each one needs their own referral link.

## Before you start

With the Partnero MCP server connected, call `get_program` first:

- `program.type` — must be `Referral`. `Affiliate` means partners rather than
  customers do the referring, so use `partnero-affiliate-tracking` instead; the
  payload field differs and the wrong one drops attribution.
- `program.public_id` — the ID for the tracking snippet.
- `program_settings.ref_link_param` and `ref_link_format` — the shape of a
  referral URL, for testing.
- `program_settings.cookie_lifetime` — the attribution window in days.

The API key isn't available through MCP. It's in the dashboard under
**Integration → API**, and belongs in an environment variable.

## 1. Install the tracking script

Before `</head>` on every page:

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

It stores the referrer's key in a first-party `partnero_referral` cookie.

## 2. Create every customer on sign-up

In the sign-up handler, read the `partnero_referral` cookie from the request.

Referred sign-up:

```json
{
  "referring_customer": { "key": "value of the partnero_referral cookie" },
  "key": "your internal user id",
  "email": "user@example.com",
  "name": "Customer Name"
}
```

Organic sign-up — same call, minus the referrer:

```json
{
  "key": "your internal user id",
  "email": "user@example.com",
  "name": "Customer Name"
}
```

Both go to `POST https://api.partnero.com/v1/customers` with
`Authorization: Bearer YOUR_API_KEY`.

The response contains `referral_link`, a `referral_links` array of any custom
links, and `referrals_count`. Store the link on the user record so the dashboard
doesn't have to fetch it on every page load. The `id` in the response is the key
you sent.

Unlike an affiliate program, the referrer here is genuinely optional, so both
shapes are accepted. Send every sign-up.

## 3. Record purchases

Rewards usually depend on the referred friend actually buying something, so the
payment handler still reports:

```http
POST https://api.partnero.com/v1/transactions
Authorization: Bearer YOUR_API_KEY

{
  "customer": { "key": "your internal user id" },
  "key": "order or invoice id",
  "amount": 99.99,
  "action": "sale"
}
```

Use the order ID as the key so a refund can reverse it with
`DELETE /v1/transactions/{key}`. The key is not a uniqueness constraint, so
posting the same order twice creates two transactions and two rewards — if this
runs in a processor webhook handler, guard against redelivery on your side.

## 4. Build the referral dashboard

Four read endpoints, all `GET` with `Authorization: Bearer YOUR_API_KEY`, all
keyed by the customer key you chose in step 2:

| Endpoint | Gives you |
| --- | --- |
| `/v1/customers/{key}` | `referral_link`, `referrals_count`, name, email |
| `/v1/customers/{key}/stats` | Click and referral counts, social sharing links |
| `/v1/customers/{key}/balance` | Reward balance per currency |
| `/v1/customers/{key}/referrals` | The referred customers and their status |

The last three exist only in refer-a-friend programs. Calling them against an
affiliate program returns a program-type error, not data.

Fetch them in parallel from the backend and render:

- The referral link with a copy button
- Share buttons built from the links in the stats response
- Total referrals and current reward balance
- A table of referred friends with name and status

Never call these from the browser. The API key is a server-side secret, and
`/v1/customers/{key}` will happily return any customer to whoever holds it.

## Verify

Sign up a test user, take the `referral_link` from the create response, open it
in a clean browser profile, confirm the `partnero_referral` cookie appears, then
sign up a second user. The second should show up under the first in
`GET /v1/customers/{first}/referrals`.

With the MCP server connected, `list_customers` confirms both arrived and
`get_customer` on the first one shows the referral relationship, so you can
check this without the user opening the dashboard.

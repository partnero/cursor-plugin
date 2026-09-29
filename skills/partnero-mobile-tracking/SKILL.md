---
name: partnero-mobile-tracking
description: Add Partnero referral tracking to an iOS, Android, React Native or Flutter app, where there are no cookies and attribution has to come from a deep link. Use when referrals need to survive install and sign-up in a native or hybrid mobile app.
---

# Partnero tracking in mobile apps

On the web, the Partnero script writes the referral key to a first-party cookie
and the backend reads it at sign-up. A mobile app has no such cookie, so the app
has to carry the key itself from the deep link to the sign-up request.

## 1. Capture the key from the deep link

Referral links look like `https://yoursite.com/?ref=SOME_KEY`. When the app opens
from one — universal link, app link, or custom scheme — read the `ref` query
parameter and persist it:

| Platform | Storage |
| --- | --- |
| iOS / Swift | `UserDefaults` |
| Android / Kotlin | `SharedPreferences` |
| React Native | `AsyncStorage` |
| Flutter | `shared_preferences` |

It must survive app restarts, because the gap between clicking the link and
finishing sign-up is often days. Keep the first key you see rather than
overwriting it on every launch, unless the program's attribution model says
otherwise.

If the link can also be opened before install, pair this with whatever deferred
deep link mechanism the project already uses; Partnero only needs the key to
arrive eventually.

## 2. Send it with the sign-up

Pass the stored key to your own backend as part of the sign-up request. The
backend — never the app — calls Partnero, using the API key from the dashboard
under **Integration → API**, held in a server-side environment variable:

```http
POST https://api.partnero.com/v1/customers
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json
```

For an affiliate program:

```json
{
  "partner": { "key": "STORED_REFERRAL_KEY" },
  "key": "your internal user id",
  "email": "user@example.com",
  "name": "User Name"
}
```

For a refer-a-friend program, the field is `referring_customer` instead of
`partner`. Check `program.type` with the MCP `get_program` tool before picking
one — it reads `Affiliate` or `Referral`.

The two program types also differ in what happens without a key. Refer-a-friend
accepts the call and creates an unreferred customer, so send every sign-up.
Affiliate rejects it, so only send sign-ups that actually carry a stored key.

Clear the stored key once the customer has been created successfully.

## 3. Record purchases from the backend

App store receipts and in-app purchases should be validated server-side anyway.
From that same server-side confirmation:

```http
POST https://api.partnero.com/v1/transactions
Authorization: Bearer YOUR_API_KEY

{
  "customer": { "key": "your internal user id" },
  "key": "order or receipt id",
  "amount": 49.99,
  "action": "sale"
}
```

Use the same customer key as at sign-up, and the receipt or order ID as the
transaction key so a refund can reverse it with `DELETE /v1/transactions/{key}`.

## Never call Partnero from the app

The API key would ship inside the binary, where anyone can extract it and write
to your program. Every Partnero call belongs on your backend. This is the one
constraint in this skill with no acceptable workaround.

## Verify

Open a referral link on a device, confirm the key is stored, kill and relaunch
the app, then complete a sign-up. The new customer should appear attributed in
Partnero, and the stored key should be gone afterwards.

---
name: verify-partnero-integration
description: Audit this codebase's Partnero integration and report what would stop a partner from being paid
---

# Verify Partnero integration

Audit the current codebase for a correct Partnero integration. Read the code —
do not run anything that writes to Partnero. Report findings grouped as
**Broken**, **Risky** and **Fine**, and quote the file and line for each.

## 0. Establish what correct means here

If the Partnero MCP server is connected, call `get_program` before reading any
code. `program.type` (`Affiliate`, `Referral` or `Newsletter`) decides which
cookie and payload shape the code should be using, and `program.public_id` is
the ID that must appear in the tracking snippet. An integration wired for the
wrong program type is the finding, not the symptom.

Without MCP, ask the user which type the program is rather than inferring it
from the code you're supposed to be auditing.

## 1. The tracking script

- Find where `universal.js` is loaded. It must come from
  `https://app.partnero.com/js/universal.js`. Any other host — `cdn.partnero.com`,
  `js.partnero.com`, `static.partnero.com` — is broken, not a DNS problem.
- The init call must be `po('program', '<id>', 'load')`. Flag `po('init', ...)`
  or a program ID passed as a query string on the script URL.
- Check the ID in the snippet against `program.public_id`. A snippet pointing at
  a different program looks completely healthy and tracks nothing.
- Confirm it loads on the landing pages and the sign-up page, not only inside
  the authenticated app. A script that only runs after login never sees the
  referral URL.

## 2. Sign-up

- Find the sign-up handler and check it calls `POST /v1/customers`.
- Confirm it reads the attribution cookie from the request — `partnero_partner`
  for affiliate programs, `partnero_referral` for refer-a-friend.
- Check the condition around the call against the program type. Refer-a-friend
  programs should send every sign-up, referred or not. Affiliate programs should
  send only the ones with a cookie, because the API rejects a customer with no
  partner attached.
- Check social login and OAuth callbacks separately. They often bypass the main
  handler, and a client-side-only integration misses them entirely.
- Note which value is used as `key`.

## 3. Transactions

- Find the payment success path and check it calls `POST /v1/transactions`.
- Confirm `customer.key` is the same value identified in step 2. A mismatch here
  is the single most common reason commissions never appear.
- Confirm the transaction `key` is the order or invoice ID, not a random or
  generated value — refunds need it to be lookup-able.
- If the project also has Stripe, Paddle or Chargebee connected to Partnero,
  flag the manual call as a possible double count.
- Check the call can't run twice for one payment. Transaction keys are not
  unique, so a redelivered processor webhook or a reloaded confirmation page
  posts a second sale and pays a second commission. Look for an idempotency
  guard that survives a retry, and flag its absence in any handler invoked by
  an external webhook.
- Flag any `po('transactions', 'create', ...)` in client-side code where a
  server-side path exists. The amount is editable by whoever is on the page.
- Check the refund path calls `DELETE /v1/transactions/{key}`.

## 4. Safety

- Every Partnero call must be non-blocking: wrapped in try/catch, failures
  logged and swallowed, never propagated into sign-up or checkout.
- The API key must not appear in client bundles, mobile code, or anything
  committed to the repo. Flag any `Bearer` token in frontend source or a
  `NEXT_PUBLIC_`-style variable holding it.
- Any webhook receiver must verify the `Signature` header against the raw
  request body before trusting the payload.

## 5. Report

End with the shortest list of changes that would make the integration correct,
ordered by how much money each one is losing.

If the Partnero MCP server is connected, you may call read-only tools such as
`get_program`, `list_customers` and `list_transactions` to confirm that data is
actually arriving. Do not create or modify anything.

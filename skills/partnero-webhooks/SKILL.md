---
name: partnero-webhooks
description: Build an endpoint that receives Partnero webhooks — partner, customer, transaction, lead and reward events — with signature verification, idempotency and a fast acknowledgement. Use when syncing Partnero activity into another system, or when an existing Partnero webhook handler is failing, retrying or double-processing.
---

# Partnero webhooks

Partnero POSTs JSON to a URL you register when something happens in a program.
This skill covers the receiving end: the handler, how to prove the request really
came from Partnero, and how to survive retries.

## 1. Find out which events this program sends

Event availability depends on the program type, so don't work from a remembered
list. If the Partnero MCP server is connected, call `list_webhooks` — the response
includes `available_events`, which is the exact set of values the program accepts.
Otherwise the same list is in the dashboard under the webhook form.

The full vocabulary, of which each program exposes a subset:

- Partners: `partner.created`, `partner.updated`, `partner.deleted`,
  `partner.approved`, `partner.rejected`, `partner.archived`, `partner.rewarded`
- Customers: `customer.created`, `customer.updated`, `customer.deleted`,
  `customer.rewarded`, `customer.credited`
- Transactions: `transaction.created`, `transaction.deleted`
- Leads: `lead.submitted`, `lead.converted`, `lead.rejected`
- Subscribers: `subscriber.created`, `subscriber.updated`,
  `subscriber.deleted`, `subscriber.rewarded`

The `.rewarded` and `.credited` events are the ones worth wiring up if you're
mirroring earnings into another system — they fire when commission is actually
granted, which `transaction.created` doesn't tell you.

## 2. Write the endpoint

The request body looks like this:

```json
{
  "event": "transaction.created",
  "url": "https://yourapp.com/webhooks/partnero",
  "webhook_key": "the key of the webhook that fired",
  "created_at": "2026-05-07T10:30:00.000000Z",
  "data": { "...": "the event subject" }
}
```

Requirements for the handler:

- Accept `POST`, parse JSON, branch on the `event` field.
- Respond `200` within a few seconds. Do the real work in a queue or background
  job — a slow handler is treated as a failure and retried.
- Be idempotent. Retries mean the same event can arrive more than once. Key your
  processing on something stable from `data` (the transaction or customer key),
  not on receipt order.
- Log the raw body before processing. Webhook bugs are impossible to diagnose
  after the fact without it.
- Ignore unknown event names instead of erroring, so new events don't break you.

## 3. Verify the signature

Every delivery carries a `Signature` header: the HMAC-SHA256 of the raw request
body, hex encoded, keyed with that webhook's signing secret. The secret lives in
the dashboard under **Integration → Webhooks** — it is deliberately never
returned by the API or the MCP tools, so ask the user to copy it and store it as
an environment variable.

Compute the HMAC over the **raw body bytes**, before any JSON parsing or
re-serialisation, and compare with a constant-time comparison:

```js
const expected = crypto
  .createHmac('sha256', process.env.PARTNERO_WEBHOOK_SECRET)
  .update(rawBody)
  .digest('hex');

if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature))) {
  return res.status(401).end();
}
```

Frameworks that eagerly parse JSON will break this. In Express, mount
`express.raw({ type: 'application/json' })` on the webhook route. In Laravel,
use `$request->getContent()`. In Next.js route handlers, read `await req.text()`.

## 4. Register the URL

With the MCP server connected, `create_webhook` takes a `name`, an HTTPS `url`
and an `events` array chosen from `available_events`. It will reject any event
name outside that set. The signing secret it generates has to be copied from the
dashboard afterwards.

Otherwise register it in the Partnero dashboard under the program's webhook
settings.

## Verify

Trigger a real event — create a test customer or transaction — and confirm the
handler receives it, the signature check passes, and replaying the same payload
twice leaves your system in the same state.

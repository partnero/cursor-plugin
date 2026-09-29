---
name: partnero-troubleshoot-attribution
description: Work out why a specific sign-up or sale in Partnero didn't earn a commission, by walking the chain from customer to partner to transaction to reward with the read-only MCP tools. Use when someone says a partner should have been paid and wasn't, a referral is missing, a sale shows no commission, or the numbers in Partnero don't match their own system.
---

# Why didn't this earn a commission?

A commission is the end of a chain: a customer attributed to a partner, a
transaction attached to that customer, and a reward generated from that
transaction. Exactly one link is usually missing, and which one tells you
whether the bug is in the tracking code, the program configuration, or nothing
at all.

Work the chain in order. Don't skip ahead — a missing reward usually means a
missing transaction, and a missing transaction usually means a missing customer.

**Read only.** Every step here uses a read tool. Never "fix" a gap by calling
`create_transaction` or `create_customer`: that fabricates revenue and pays a
commission on a sale Partnero has no record of. The fix is in the integration,
not in the data.

## 0. Establish the shape of the program

Call `get_program`:

- `program.type` — `Affiliate` means partners refer; `Referral` means customers
  refer. It determines which field you're looking for in step 2.
- `program_settings.cookie_lifetime` — the attribution window in days. A sign-up
  outside it is correctly unattributed.
- `program_settings.commission_description`, `commission_value` and
  `has_multiple_commissions` — what should have been earned, and whether more
  than one rule could apply.

## 1. Does the customer exist?

`get_customer` accepts `key`, `id` or `email`. If you only have a partial name or
address, `search_customers` with a `query` fragment first.

**Not found** is the most common answer, and it means the sign-up never reached
Partnero. In an affiliate program that's usually correct behaviour rather than a
bug: customers are only created when the attribution cookie was present, so an
organic sign-up legitimately has no record. In a refer-a-friend program every
sign-up should be there, so a missing one is a broken sign-up handler.

**Found with `deleted` true** means the customer was archived. Archived
customers stop earning.

## 2. Is the customer attributed?

In the customer payload, an affiliate program shows `partner` — the attributed
partner's ID. A refer-a-friend program shows `referring_customer`.

**Empty** means the customer was created without attribution. The sign-up call
fired but the cookie value never made it in. That's a code bug: run
`/verify-partnero-integration` and look at how the handler reads the cookie.

**Present** — carry the partner ID to step 4, and compare the customer's
`created_at` against `cookie_lifetime`. A click far outside the window is
supposed to fall off.

## 3. Does the transaction exist?

Three ways in, depending on what you have:

- `get_transaction` with `key` — the order or invoice ID.
- `get_customer_transactions` with `customer_id` — everything for one customer.
- `list_transactions` with `partner_key` — everything credited to one partner.

**Not found** means the payment handler never called Partnero, or called it with
a different key than you're searching for. Check whether the processor is
connected instead: if Stripe, Paddle or Chargebee reports transactions, they
arrive on their own schedule and a very recent sale may not have landed yet.

**Found with `deleted_at` set** means it was reversed — almost always a refund
calling `DELETE /transactions/{key}`. That's the system working.

**Found more than once for the same order** is the opposite complaint — a
partner paid twice for one sale. Transaction keys aren't unique, so this is a
double-send: a webhook handler that ran twice on a processor redelivery, a
confirmation page that was reloaded, or both a processor connection and a
manual call reporting the same payment. Check whether a processor is connected
before assuming the handler is at fault.

**Found but `customer` is empty or a different key** is the classic mismatch: the
sign-up used one customer key and the payment used another, so the sale is
recorded against nobody. Compare the two values literally, including case.

## 4. Was a reward generated?

The transaction payload carries a `rewards` array. If it has entries, the
commission exists — use `get_reward` for the detail, or `list_rewards` for the
program-wide picture.

**Empty `rewards`** with everything else intact means the transaction was
recorded but earned nothing. Check, in this order:

1. `get_partner` on the attributed partner. An unapproved or rejected partner
   doesn't earn. Look at `approved` and `status`.
2. The commission rules from step 0. If `has_multiple_commissions` is true, a
   rule may exist that excludes this product, amount or currency.
3. `is_currency` on the transaction. When false, the amount counts units rather
   than money, and a percentage commission on it behaves differently than
   expected.
4. Whether the transaction was created with `rewardable` false, which suppresses
   the reward deliberately.

## 5. Report

Say which link broke, in plain terms, and whether it's a code bug, a
configuration choice or correct behaviour. Those three have very different
fixes, and "correct behaviour" is a common and useful answer — an organic
sign-up in an affiliate program has no partner to pay, and that is not a defect.

If it is a code bug, name the handler that needs changing rather than describing
the symptom.

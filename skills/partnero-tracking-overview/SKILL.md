---
name: partnero-tracking-overview
description: How Partnero tracking works end to end — the referral link, the cookie, the sign-up that creates the relationship, the transaction that attaches revenue, and the commission that results. Use when someone is new to Partnero, is deciding how to integrate, isn't sure which program type or method applies, or asks how the tracking actually works.
---

# How Partnero tracking works

Read this before wiring anything up. Most broken integrations are broken because
one link in a four-link chain is missing, and you can't tell which one unless
you know what the chain is.

## The chain

**1. A referral link carries a key.** Every partner (or, in refer-a-friend, every
customer) has a unique key. Their link embeds it — as a query parameter like
`?aff=KEY`, or as a path segment like `/ref/KEY`, depending on the program's
`ref_link_format` and `ref_link_param`. A program can also have direct tracking
switched on, in which case any single-segment URL — `yoursite.com/anna` — is
read as a key when nothing else supplies one.

**2. PartneroJS reads the key and stores it.** The tracking script on your site
takes the key out of the incoming URL and writes it to a first-party cookie on
your domain: `partnero_partner` for an affiliate program, `partnero_referral`
for refer-a-friend. It expires after the program's `cookie_lifetime`.

If a visitor arrives through two partners' links, which one wins is the
program's `referral_click_attribution_model`: `last_click` (the default)
overwrites the cookie, `first_click` keeps the one already there. Worth knowing
before you test several links in one browser and conclude the script is broken.

Nothing is attributed yet. At this point Partnero doesn't know the visitor
exists — the key is sitting in a cookie in one browser.

**3. Your backend creates the relationship.** When the visitor signs up, your
server reads that cookie and tells Partnero: this new customer came from that
key. That call is what creates the durable link between a customer and the
partner who referred them. It is the only step that can't be backfilled later,
because the cookie is gone once the browser moves on.

If reading the cookie server-side is awkward — a form that posts to another
domain, a framework that doesn't hand you request cookies where you need them —
put `data-partnero` on the `<form>` instead. The script injects the key into it
as a hidden input named `partnero_partner` or `partnero_referral` (plus
`partnero_click_id` when one exists), so it arrives in the POST body like any
other field. Forms rendered after page load are picked up too.

**4. Your backend attaches the revenue.** When the customer pays, you send a
transaction naming the same customer key. Partnero looks up which partner that
customer belongs to, applies the program's commission rules, and creates the
reward.

Sign-up creates the *relationship*; the transaction attaches *money* to it. Both
name the customer by the same key, and that key is yours to choose — your user
ID or their email. A mismatch between the two calls is the single most common
reason commissions never appear.

## Four ways the key travels

The cookie is the usual path, not the only one. Which one applies changes the
whole integration:

| Route | When | Skill |
| --- | --- | --- |
| Cookie → your backend → Partnero API | You control the sign-up handler | `partnero-affiliate-tracking`, `partnero-refer-a-friend` |
| Cookie → payment processor metadata → Partnero | Stripe, Paddle or Chargebee is connected and reports transactions for you | `partnero-payment-processors` |
| Server-registered click ID → postback URL | No browser at all: affiliate networks, ad platforms, API-only products | `partnero-server-to-server-tracking` |
| Deep link → app storage → your backend | Mobile apps, where there are no cookies | `partnero-mobile-tracking` |

They combine. A typical SaaS uses the cookie for sign-up and the processor for
transactions, and never writes a transaction call at all.

## Program type changes the vocabulary

Check it with `get_program` before writing anything — `program.type` reads
`Affiliate`, `Referral` or `Newsletter`.

In an **affiliate** program, partners are separate people who refer customers. A
customer exists only in the context of a partner, so you create one only when
the cookie is present; the API rejects a customer with no partner attached.

In a **refer-a-friend** program (`Referral`), customers refer each other. Every
customer is also a potential referrer, so you create one for every sign-up and
the referrer is optional. Each gets a `referral_link` of their own.

Sending the wrong field — `partner` where `referring_customer` belongs — loses
attribution.

## What breaks, and where to look

| Symptom | Missing link |
| --- | --- |
| Cookie never appears | Script missing from the landing page, or loaded from a hostname that doesn't exist |
| Cookie appears, customer unattributed | Sign-up handler isn't reading the cookie |
| Customer attributed, no revenue | Transaction never sent, or sent with a different customer key |
| Revenue recorded, no commission | Partner not approved, or commission rules don't match |

`/verify-partnero-integration` audits the code for the first three.
`partnero-troubleshoot-attribution` traces a specific sale through the live data.

## Two constants

The tracking script is served from exactly one host,
`https://app.partnero.com/js/universal.js`, and the REST API key is server-side
only. Everything else varies by program; those two don't.

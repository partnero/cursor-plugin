---
name: partnero-no-code-platforms
description: Install Partnero tracking on a hosted site builder or funnel platform — Webflow, Squarespace, Framer, Kajabi, WordPress, Wix, GoHighLevel, Simvoly, Thinkific, BigCommerce, Calendly, Typeform, Shopify, WooCommerce. Use when there is no backend to edit and tracking has to go in through a code injection field, or when a form on one of these platforms isn't producing customers in Partnero.
---

# Partnero on hosted platforms

On a platform you don't control the server of, all three tracking layers have to
be installed through whatever code injection field the platform offers. The
script part is easy and identical everywhere. Capturing the sign-up is where
platforms differ, because each one submits forms its own way.

## First: is there a native app?

| Platform | Do this instead |
| --- | --- |
| Shopify | Install the Partnero app from the Shopify App Store, connect the account, pick the program |
| WooCommerce | Install the "Partnero Affiliate Management" plugin, activate it, enter the program API key |

Both handle script injection and transactions themselves. Don't hand-build an
integration on either one.

## 1. The script

Paste this into the platform's head/header injection field. It is the same
snippet everywhere — only the field changes.

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

Get `PROGRAM_ID` from the MCP `get_program` tool (`program.public_id`) if the
server is connected, otherwise from the Partnero dashboard. Where a step below
needs an API key instead, that comes from **Integration → API** in the
dashboard and must stay server-side.

Where it goes, per platform:

| Platform | Field |
| --- | --- |
| Webflow | Project Settings → Custom Code → Head Code |
| Squarespace | Settings → Advanced → Code Injection → Header |
| Framer | Project Settings → Custom Code → Head, "Start of `<head>` tag" |
| Kajabi | Settings → Site → Code Injections → Header Code |
| WordPress | A snippet plugin such as WPCode Lite, set to header |
| Wix | Website → Settings → Custom code → Head |
| GoHighLevel | Funnel settings → Head tracking code |
| Simvoly | Builder → Website Settings → Custom Code → Custom Head Markup |
| Thinkific | Settings → Code & analytics → Site footer code |
| BigCommerce | Script Manager → Header, all pages, Analytics |
| Calendly | Your own site; Calendly itself hosts no custom code |

It must be on the landing pages, not only the checkout. That's where the
referral URL arrives.

## 2. The sign-up

### First check whether JavaScript can see the sign-up at all

Everything in this section works by watching a form submit in the page. If the
sign-up doesn't happen that way, none of it applies and no amount of selector
tuning will help:

- **Social sign-up** — Google, Apple, Facebook, "Continue with…" — hands the
  user off to the provider and back. There is no form submit carrying an email.
- **Magic links and email OTP** — the account is created when the link is
  clicked, on a different page load.
- **Hosted auth** — Auth0, Clerk, Memberstack, Webflow Memberships and the
  like — puts the sign-up on a domain or iframe you can't observe.

In all of these, the customer has to be created from the server, after the
provider tells you who signed up. See `partnero-affiliate-tracking` or
`partnero-refer-a-friend`. A site with both a password form and a Google button
needs the server-side call regardless, or it silently loses every referral who
chose Google — so if there is any social option at all, go server-side and skip
the rest of this section.

### Custom JavaScript — the usual answer

Prefer this. Sign-up forms are effectively unique per site: hosted builders
rename fields, wrap inputs in their own components, submit over AJAX, and
change markup between template versions. Reading the real selectors off the
live page and wiring them yourself is more work once and far more reliable than
hoping a generic matcher fits.

Open the actual published page, find the actual submit control and the actual
inputs, and use those:

```html
<script>
document.addEventListener('click', function (e) {
  if (e.target.matches('ACTUAL_BUTTON_SELECTOR')) {
    const email = document.querySelector('ACTUAL_EMAIL_SELECTOR').value;
    const name = document.querySelector('ACTUAL_NAME_SELECTOR')?.value;
    if (email) {
      po('customers', 'signup', { data: { key: email, email: email, name: name || null } }, null);
    }
  }
});
</script>
```

Never ship placeholder selectors. If you can't fetch the page, ask for the
markup rather than guessing.

### Universal — for a plain main form

Where the page has one ordinary sign-up form built from real inputs, the
universal helper is enough and saves the selector work:

```html
<script>po('integration', 'universal', null);</script>
```

For the email it accepts `input[name="email"]`, any `input[type="email"]`,
`input[name="data[email]"]` or `input[data-type="email"]`. For the name it takes
`name`, `full_name` or `fullName`, and otherwise assembles one from
`first_name` / `firstName` / `fname` / `nombre` plus `last_name` / `lastName`.

It binds to every form on the page, so on a page that also has a newsletter box
or a contact form, name the one you want:

```html
<script>po('integration', 'universal', { form: 'signup-form-id' });</script>
```

Check that it actually fires before shipping it. If it doesn't, don't tune it —
write the custom version.

### Platform helpers — narrow, and only if they match

A few platforms have a purpose-built helper: `gohighlevel`, `simvoly`,
`calendly`, `oncehub`, `boomerangme`, `brilliantdirectories` and `bui`, each
called as `po('integration', 'NAME', null)`. There is also a `wix` helper, but
it does nothing — on Wix, write the custom version.

These are hard-coded to one markup pattern each, listed in `platforms.md`. That
pattern is the platform's default template at the time the helper was written,
so a customised funnel or a newer template will simply not match, with no error.
Treat a helper as a shortcut worth testing, not a solution to rely on: confirm
the customer actually arrives in Partnero, and fall back to custom JavaScript
the moment it doesn't.

### Two things that make all of these look broken

They only fire when an attribution key is present — no referral cookie and no
key in the URL means no call at all, by design. A test submission in a clean
browser produces nothing, and that is correct. Always test through a real
referral link.

They also depend on **customer creation via JavaScript** being enabled under
Integration → Security in the program settings. It's on by default, but if
someone turned it off, every call is rejected and the page looks fine. Check
that before blaming the selectors.

## 3. The transaction

Transactions are money, so where the amount comes from matters more than
convenience. In order:

**Connect the payment processor.** On most of these platforms this is the whole
answer — Stripe, Paddle or Chargebee under Program → Integration. The amount
then comes from the processor's own webhook, which nobody can edit, and refunds
reverse commissions automatically. See `partnero-payment-processors` for how
attribution reaches the processor.

**Call the API from a server.** Where the platform has any backend hook —
Thinkific and Wix both do — send the transaction from there with the program
API key. Same trust properties, and you get real validation errors back.

**JavaScript, only when there is no other option.** Some platforms expose order
variables on a confirmation page and nothing else: BigCommerce's conversion
tracking field, PayPal return pages. It works, but understand what you're
accepting — the call runs in the browser, so the amount, the order ID and the
customer key are all visible and editable by whoever is on the page. Someone
can open the console and post a larger sale, and Partnero will record it,
because a public JavaScript call carries no proof of what was actually paid.

```html
<script>
po('transactions', 'create', {
  data: { key: 'ORDER_ID', amount: 99.99, customer: { key: 'ORDER_EMAIL' } }
}, null);
</script>
```

Send the real order ID as `key` so the transaction can be traced back and
reversed on refund. Be aware that this does **not** deduplicate: transaction
keys are not unique, so a reloaded confirmation page posts the sale a second
time and earns a second commission. Nothing on the Partnero side prevents that,
which is another reason to prefer the processor connection wherever the
platform allows one. And never let the amount come from a URL parameter or
anything else the visitor controls.

Because of all that, **transaction creation via JavaScript is off by default**
under Integration → Security — the opposite of the customer setting. If it's
off, calls are silently rejected while the page looks fine. Turning it on is a
deliberate decision to trust the browser, so before you tell someone to enable
it, make sure the processor connection and the server-side route are genuinely
unavailable.

## Per-platform detail

Read `platforms.md` in this skill directory for the specific quirks: which
platforms have AJAX forms that defeat universal tracking, which ones block
scripts on checkout pages, and the backend hooks available on Wix and Thinkific.

## When a settings change doesn't take effect

`po('program', ID, 'load')` reads the program's settings from a cached file, so
a change you just made in the dashboard — the two Integration → Security
toggles especially — may not reach a browser that already holds the old copy.

Add this above the `load` call while you work, and it bypasses the cache on
every page load:

```html
<script>po('settings', 'debug', true);</script>
```

Take it out when you're done; it defeats the cache for real visitors too.

## Verify

Open the site through a real referral link, confirm the attribution cookie
appears in the browser, submit the form, and check the customer lands in
Partnero. If the cookie is there and the customer isn't, the sign-up capture is
the broken layer, not the script.

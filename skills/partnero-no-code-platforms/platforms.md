# Platform notes

Per-platform detail for `partnero-no-code-platforms`. The script placement table
and the four sign-up approaches are in `SKILL.md`; this file only covers what
differs.

The CSS selectors quoted here are what Partnero's helpers actually match, so
they are exact. The merge tags and order variables — Thinkific's `{{email}}`,
BigCommerce's `%%ORDER_ID%%` — belong to those platforms and can be renamed by
them, so check the field reference in the platform's own docs if a value comes
through empty.

## Webflow

Forms are standard HTML submitted over AJAX, which the universal helper handles.

- Start with `po('integration', 'universal', null)`; Webflow's default field
  names are among the ones it recognises.
- If the field names are genuinely outside that list, read the real ones off
  the published page and use custom JavaScript.
- Webflow Memberships and any external auth need the server-side API instead —
  the sign-up doesn't happen in a page you can observe.
- Transactions: connect Stripe, Paddle or Chargebee to Partnero. Webflow itself
  doesn't process payments.

## Squarespace

Forms submit over AJAX in a way the universal helper often can't see. Expect to
write custom JavaScript that intercepts the submission and calls
`po('customers', 'signup', { data: { key, email, name } }, null)`.

For Squarespace Commerce, transactions arrive automatically once Stripe is
connected to Partnero. External auth or sign-up flows need the API.

## Framer

Framer pages are React components, not HTML forms, so there is usually nothing
for the universal helper to hook into.

- Checkout through Stripe Payment Links, Pricing Tables or Buy Buttons: add
  `po('integration', 'stripe', null)` and you're done — see
  `partnero-payment-processors`.
- Sign-up on an external page: install the script there too and track it there.
- Framer's native forms: custom JavaScript on the submission event.
- Transactions: connect the payment processor.

## Kajabi

Kajabi's forms are custom components with non-standard submission, so
server-side API tracking is the recommended path for sign-ups. If a page happens
to contain a plain HTML form, `po('integration', 'universal', null)` or custom
JavaScript will work.

Payments go through Stripe or PayPal — connect Stripe to Partnero for automatic
transactions.

Not every Kajabi plan allows custom scripts on checkout and offer pages. When
script injection is restricted there, the API is the only option.

## WordPress

Install a code snippet plugin — WPCode Lite is the usual choice — then put the
tracking script in the header and `po('integration', 'universal', null)` in the
footer.

For sales, connect Stripe, Paddle or Chargebee to Partnero, or call the API from
a hook in the theme or a plugin. If the store is WooCommerce, stop and use the
Partnero WooCommerce plugin instead of any of this.

## Wix

Script goes in Website → Settings → Custom code → Head. There is no Wix-specific
sign-up helper, so use universal tracking or custom JavaScript.

Wix Stores can report transactions from the backend. Create `events.js` in the
backend section:

```javascript
import { fetch } from 'wix-fetch';

export async function wixStores_onOrderPaid(event) {
  await fetch('https://api.partnero.com/v1/transactions', {
    method: 'POST',
    headers: {
      'Content-type': 'application/json',
      Authorization: 'Bearer YOUR_API_TOKEN',
    },
    body: JSON.stringify({
      action: 'sale',
      customer: { key: event.billingInfo.email },
      key: event._id,
      amount: event.totals.total,
      amount_units: event.currency,
    }),
  });
}
```

## GoHighLevel

Script goes in the funnel's head tracking code.

`po('integration', 'gohighlevel', null)` only works if the page has a form
container with class `.container-order-form-two-step`, a submit button with
class `.form-btn`, and inputs named `email` and `name`. Anything else needs
custom JavaScript with the real selectors.

## Simvoly

Script goes in Builder → Website Settings → Custom Code → Custom Head Markup.

`po('integration', 'simvoly', null)` expects checkout buttons matching
`.checkout-info-inner a.btn[data-role="next-step"]` or
`.checkout-info-inner a.btn[data-role="continue"]`, with inputs named `email`
and `name`. Otherwise, custom JavaScript.

## Thinkific

Script goes in Settings → Code & analytics → Site footer code.

Thinkific exposes template variables, so sign-up tracking goes in the dedicated
"Signup tracking code" field:

```html
<script>
window.onload = function () {
  po('customers', 'signup', {
    data: { key: '{{email}}', name: '{{first_name}}', email: '{{email}}' }
  }, null);
};
</script>
```

Transactions can go in "Order tracking code", but this is the last-resort route
described in `SKILL.md` §3 — skip it entirely if Stripe is connected to
Partnero, and prefer Thinkific's webhooks to a server of your own if you have
one. The amounts below are client-side and therefore editable by the buyer:

```html
<script>
window.onload = function () {
  po('transactions', 'create', {
    data: { key: '{{product_id}}', amount: {{total_price}}, customer: { key: '{{billing_email}}' } }
  }, null);
};
</script>
```

Transaction creation via JavaScript must be enabled in the program settings.

## BigCommerce

Script goes through Script Manager, on the header, all pages, category
Analytics.

BigCommerce has no Partnero-side payment connection, so transactions come from
the storefront's Affiliate Conversion Tracking field. Read `SKILL.md` §3 first
— this is a client-side amount, and BigCommerce's own webhooks to a small
endpoint of your own are the sturdier option if the program is worth much:

```html
<script>
po('transactions', 'create', {
  data: {
    key: '%%ORDER_ID%%',
    amount: %%ORDER_AMOUNT%%,
    customer: { key: '%%ORDER_EMAIL%%', email: '%%ORDER_EMAIL%%' }
  },
  options: { create_customer: true }
}, null);
</script>
```

`create_customer: true` makes the customer on the fly, so no separate sign-up
call is needed. Transaction creation via JavaScript must be enabled in the
program settings.

## Calendly

Calendly hosts no custom code, so the script lives on your own site and the
tracking happens on the confirmation page you redirect to. Enable "Pass event
details" on the event.

`po('integration', 'calendly', null)` expects the confirmation URL to carry
`invitee_email`, and optionally `invitee_full_name`. If the parameter names
differ, read them yourself:

```html
<script>
const params = new URLSearchParams(window.location.search);
const email = params.get('ACTUAL_EMAIL_PARAM');
const name = params.get('ACTUAL_NAME_PARAM');
if (email) {
  po('customers', 'signup', { data: { key: email, email: email, name: name || null } }, null);
}
</script>
```

## OnceHub

Same shape as Calendly — the booking happens off your site, so tracking runs on
the confirmation page you redirect to. `po('integration', 'oncehub', null)`
reads `Email` and `Name` from the confirmation URL. Note the capitals: these
are different parameter names from Calendly's, and the helper is case
sensitive.

## Typeform

The problem with Typeform is that the form is on Typeform's domain, so the
attribution cookie doesn't travel with the visitor. Carry the key in the URL
instead.

1. In Typeform, create a hidden field named `aff` under Logic → Hidden Fields.
2. On your own site, rewrite links to the form so they carry the cookie value:

```html
<script>
(function () {
  const baseUrlPattern = 'https://link-to-form.com';
  const affParameter = 'aff';

  const updateLinkHref = (link) => {
    const cookieValue = (document.cookie.match(/(^| )partnero_partner=([^;]+)/) || [])[2];
    if (!cookieValue) return;
    try {
      const url = new URL(link.href);
      url.hash = `${affParameter}=${decodeURIComponent(cookieValue)}`;
      link.href = url.href;
    } catch (error) {
      console.error('Error:', error);
    }
  };

  document.querySelectorAll(`a[href^="${baseUrlPattern}"]`).forEach((link) => {
    updateLinkHref(link);
    link.addEventListener('click', () => updateLinkHref(link));
  });
})();
</script>
```

Use `partnero_referral` instead of `partnero_partner` for a refer-a-friend
program.

3. Connect the submission: a webhook from Typeform (Connect → Webhooks) to the
   Partnero webhook URL under Integration → Webhooks.

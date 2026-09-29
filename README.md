# Partnero for Cursor

Two things in one plugin:

- **Your program, in the agent.** A connection to Partnero's MCP server, so you
  can ask about partners, customers, transactions, leads and rewards — and act on
  them — without leaving the editor.
- **Integration help that is actually correct.** Skills, a rule and a command
  that know the real endpoints, the real script host and the handful of mistakes
  that quietly stop partners from getting paid.

## Install

From the Cursor marketplace: open **Customize** in the sidebar, search for
*Partnero*, and select **Install**.

To connect only the MCP server, without the skills, paste this into your
browser's address bar:

```text
cursor://anysphere.cursor-deeplink/mcp/install?name=partnero&config=eyJ1cmwiOiAiaHR0cHM6Ly9hcGkucGFydG5lcm8uY29tL21jcCJ9
```

## Connecting

The first Partnero request opens an authorization page in your browser. Sign in,
choose which program the agent may access, and approve. There is nothing to
configure and no API key to paste — the server registers Cursor as a client on
its own.

Access is scoped to the program you pick. Re-authorize to switch programs or to
revoke it.

Because the agent can read the program, it doesn't have to interrogate you about
it. It knows the program's ID and type, what a referral link looks like, how long
attribution survives and which currency commissions are in, so the integration it
writes matches the program you actually have.

The one thing it can't read is your REST API key, since the MCP server
deliberately doesn't expose it. Integration code needs one — it's in the Partnero
dashboard under **Integration → API**.

## What's inside

| Component | What it does |
| --- | --- |
| `mcp.json` | Connects to `https://api.partnero.com/mcp` |
| `partnero-tracking-overview` | How tracking works end to end, and which method applies |
| `partnero-affiliate-tracking` | Wire up an affiliate program: script, sign-up, sales, refunds |
| `partnero-refer-a-friend` | Refer-a-friend tracking plus the in-app referral dashboard |
| `partnero-webhooks` | Receive Partnero events, with signature verification and idempotency |
| `partnero-mobile-tracking` | Attribution from deep links in iOS, Android, React Native and Flutter |
| `partnero-no-code-platforms` | Webflow, Squarespace, Framer, Kajabi, WordPress, Wix, GoHighLevel, Simvoly, Thinkific, BigCommerce, Calendly, Typeform |
| `partnero-payment-processors` | Attribution through Stripe, Paddle and Chargebee, front end and back |
| `partnero-server-to-server-tracking` | Cookieless tracking with click IDs and postbacks |
| `partnero-troubleshoot-attribution` | Why a given sale earned no commission, traced through the live data |
| `partnero-api` rule | Canonical endpoints and script host, so the agent stops guessing |
| `/verify-partnero-integration` | Audits an existing integration and reports what's losing money |

The skills load themselves when they're relevant. You can also call one directly
with `/partnero-affiliate-tracking`.

## Things to ask it

```
Add Partnero affiliate tracking to my Next.js app
Which partners drove the most revenue last month?
Why didn't order 10482 pay a commission?
Build a referral dashboard page for logged-in users
/verify-partnero-integration
Create a transaction for customer 4821 for $99
```

Write operations — creating partners, approving leads, recording transactions —
go through the same confirmation Cursor applies to any tool that changes
something. Several of them send email or fire webhooks to your endpoints, and
they are annotated as such so Cursor asks before running them.

## Requirements

A Partnero account with at least one program. Some tools only apply to certain
program types: leads are specific to affiliate programs, and a refer-a-friend
program has no partners to list.

## Developing this plugin

Clone it into `~/.cursor/plugins/local/partnero` and run **Developer: Reload
Window**. The plugin appears under Customize with its skills, rule and command.

## Links

- [Partnero](https://www.partnero.com)
- [Documentation](https://docs.partnero.com)
- [MCP server guide](https://docs.partnero.com/guides/mcp-server)
- [API reference](https://docs.partnero.com/api-reference/introduction)

Questions or problems: [hello@partnero.com](mailto:hello@partnero.com), or open
an issue on this repository.

## License

MIT. See [LICENSE](LICENSE).

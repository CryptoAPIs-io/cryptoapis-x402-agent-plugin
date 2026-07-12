# x402 — a Claude Code plugin

**[x402](https://x402.org) payments for AI agents — both sides of the transaction.**

One plugin, two roles:

- 💸 **BUY** — give your agent the **`x402_pay`** tool: fetch a URL, and if it responds `402 Payment
  Required`, pay it automatically (signing **locally**, non-custodially) and return the paid response.
- 🏷️ **SELL** — a skill that helps you **integrate the CryptoAPIs x402 merchant SDK** into your own
  Express/Hono/Next app, so your API can charge callers per request.

The plugin picks the right side from what you ask — "pay for this endpoint" (buyer) vs. "add payment to my
API" (merchant).

> Built on standard MCP (`@cryptoapis-io/mcp-x402-pay`) + the merchant SDK
> (`@cryptoapis-io/x402-merchant-sdk`), so the capability works in any MCP-capable agent (Cursor, Codex, …)
> and any Node app too.

---

## Install

```
/plugin marketplace add CryptoAPIs-io/x402-agent-plugin
/plugin install x402
```

---

## Buying — pay for x402 endpoints

Set your buyer credentials once (they stay out of chat + tool logs):

```bash
export CRYPTOAPIS_API_KEY="…"     # CryptoAPIs key with the X402_BUYER feature
export X402_WALLET_ID="…"         # the agent wallet to pay from
export X402_PRIVATE_KEY="0x…"     # that wallet's EVM key — signs locally, never sent
```

Then just ask:

> **You:** Fetch `https://api.example.com/premium` for me.
> **Agent:** That endpoint costs $0.01 USDC on Base — pay it? **You:** Yes. → *(pays, returns the data + tx)*

**Guardrails** (the `x402-pay` skill enforces): confirm before non-trivial payments, a `maxAmount` hard
cap, `allowedNetworks` restriction, never-loop, and a settlement receipt on every paid call. **Non-custodial** — your key signs locally; the CryptoAPIs facilitator settles.

---

## Selling — charge for your API

Ask the agent to add payment to your API and it writes the integration using
`@cryptoapis-io/x402-merchant-sdk` (Express / Hono / Next.js):

> **You:** Add a $0.01 USDC paywall to my `/premium` Express route.
> **Agent:** *(installs the SDK, wires `paymentMiddleware`, sets the price, guides env setup)*

The merchant SDK is **non-custodial** (never holds keys) and **zero-dependency**. You'll need a CryptoAPIs
key with the **X402_FACILITATOR** feature and a receiving address.

---

## What's inside

| Component | Side | What it does |
|---|---|---|
| `x402_pay` tool (MCP) | buyer | fetch → on 402, authorize + sign locally + retry → paid response |
| `x402-pay` skill | buyer | when to pay, spending guardrails, wallet setup, safety model |
| `x402-accept` skill | merchant | integrate the merchant SDK into your app to charge for endpoints |

## Related

- [`@cryptoapis-io/mcp-x402-pay`](https://www.npmjs.com/package/@cryptoapis-io/mcp-x402-pay) — the MCP server (buyer tool).
- [`@cryptoapis-io/x402-buyer-sdk`](https://www.npmjs.com/package/@cryptoapis-io/x402-buyer-sdk) — buyer capability as an npm SDK (apps + other agent frameworks).
- [`@cryptoapis-io/x402-merchant-sdk`](https://www.npmjs.com/package/@cryptoapis-io/x402-merchant-sdk) — the merchant SDK the sell skill integrates.
- [CryptoAPIs docs](https://developers.cryptoapis.io) · [x402 protocol](https://x402.org)

## License

MIT © Crypto APIs, Inc.

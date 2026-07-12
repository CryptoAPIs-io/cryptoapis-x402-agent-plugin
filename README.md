# x402 Payments — a Claude Code plugin

**Give your coding agent the ability to pay for [x402](https://x402.org)-gated HTTP endpoints.**

Install this plugin and your agent gains one tool — **`x402_pay`** — that fetches a URL and, if the server
responds `402 Payment Required`, pays it automatically (signing **locally** with your key) and returns the
paid response. Plus a skill that teaches the agent to spend safely (confirmation, caps, receipts).

- 🔒 **Non-custodial** — your private key signs locally and never leaves your machine.
- 🛡️ **Guardrails built in** — confirm-before-paying, a hard `maxAmount` cap, network allowlists, no retry loops.
- 🌐 Pays EVM stablecoin endpoints (e.g. Base USDC) via the CryptoAPIs x402 facilitator.

> Works in Claude Code. The underlying tool is a standard MCP server
> (`@cryptoapis-io/mcp-x402-pay`), so any MCP-capable agent (Cursor, Codex, …) can use it too.

---

## Install

```
/plugin marketplace add CryptoAPIs-io/x402-agent-plugin
/plugin install x402-payments
```

Then set your credentials as environment variables (so the key stays out of chat + tool logs):

```bash
export CRYPTOAPIS_API_KEY="…"     # a CryptoAPIs key with the X402_BUYER feature
export X402_WALLET_ID="…"         # the agent wallet to pay from
export X402_PRIVATE_KEY="0x…"     # that wallet's EVM key — signs locally, never sent
```

That's it. Ask your agent to fetch a paywalled URL and it will pay for it.

---

## Example

> **You:** Get me the premium forecast from `https://api.example.com/premium/weather`.
>
> **Agent:** That endpoint requires a $0.01 USDC payment on Base. Shall I pay it?
>
> **You:** Yes.
>
> **Agent:** *(calls `x402_pay`)* Paid $0.01 (tx `0x…`). Here's the forecast: …

---

## What's inside

| Component | What it does |
|---|---|
| **`x402_pay` tool** (MCP) | fetch → on 402, authorize + sign locally + retry → return the paid response |
| **`x402-payments` skill** | tells the agent when to pay, spending guardrails, wallet setup, the safety model |

The tool takes `url` (+ optional `method`, `body`, `allowedNetworks`, `maxAmount`). Credentials come from
the env vars above (or can be passed per-call). It returns `{ status, paid, body, settlement? }`.

## Safety model

- **Non-custodial:** the plugin never holds funds. Your key signs the payment authorization locally; the
  CryptoAPIs facilitator settles it on-chain.
- **Bounded:** set `X402_PRIVATE_KEY` to a wallet you fund for agent spending only. Use `maxAmount` +
  `allowedNetworks` to cap exposure. The skill instructs the agent to confirm before non-trivial payments
  and never to loop.
- **Auditable:** every paid call returns the on-chain settlement receipt.

## Not for accepting payments

This is the **buyer** side (spending). To **charge** for your own API, use the merchant SDK —
[`@cryptoapis-io/x402-merchant-sdk`](https://www.npmjs.com/package/@cryptoapis-io/x402-merchant-sdk).

## Related

- [`@cryptoapis-io/mcp-x402-pay`](https://www.npmjs.com/package/@cryptoapis-io/mcp-x402-pay) — the MCP server this bundles.
- [`@cryptoapis-io/x402-buyer-sdk`](https://www.npmjs.com/package/@cryptoapis-io/x402-buyer-sdk) — the same capability as an npm SDK (apps + other agent frameworks).
- [CryptoAPIs docs](https://developers.cryptoapis.io) · [x402 protocol](https://x402.org)

## License

MIT © Crypto APIs, Inc.

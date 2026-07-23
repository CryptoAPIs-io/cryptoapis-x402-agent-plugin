---
name: x402-pay
description: Use this skill WHENEVER the user asks the agent to access, fetch, buy, or call a paid / paywalled / metered / x402-gated HTTP endpoint or API, or when a fetch returns HTTP 402 Payment Required. Treat "pay for this API", "this endpoint needs payment", "buy access to X", "it returned 402", "purchase this data", "call the paid API", "settle the micropayment", "pay with USDC/x402" as matches. Covers the `x402_pay` MCP tool (fetch → auto-pay a 402 → retry), spending guardrails, wallet setup, and the non-custodial safety model. NOT for accepting payments on YOUR api (that is the merchant SDK, not this plugin).
disable-model-invocation: false
---

# Paying for x402 endpoints

This plugin gives you the **`x402_pay`** tool: fetch a URL and, if it responds `402 Payment Required`,
pay it with the [x402](https://x402.org) protocol and return the paid response — automatically. Payments
are **non-custodial**: the private key signs locally and never leaves the machine.

## When to use `x402_pay`

Use it instead of a plain fetch **whenever the target may require payment**:
- the user asks to access a paid / metered / premium / paywalled API or dataset,
- a normal request came back **HTTP 402**,
- the user explicitly says to pay / buy / purchase access with x402 / USDC.

Just call `x402_pay` with the `url` (and `method`/`body` if needed). It returns
`{ status, paid, body, settlement? }` — `paid: true` means a payment was made and `settlement` has the
on-chain receipt.

## Setup (once)

The tool needs three credentials, read from env by default (so you never type a key into a tool call):

| Env var | What |
|---|---|
| `CRYPTOAPIS_API_KEY` | a CryptoAPIs API key with the **X402_BUYER** feature |
| `X402_WALLET_ID` | the CryptoAPIs agent wallet to pay from |
| `X402_PRIVATE_KEY` | that wallet's EVM private key (signs locally, never sent) |

If they aren't set, `x402_pay` returns a clear `missing credentials` message — tell the user which env
vars to set (in the plugin's MCP config or their shell). **Do not ask the user to paste a private key into
the chat** — env vars keep it out of the transcript and tool logs.

## First-time setup — create the agent wallet (`X402_WALLET_ID`)

`X402_WALLET_ID` refers to a CryptoAPIs **agent wallet** that must be created ONCE per blockchain+network
before any payment. If the user doesn't have one, create it with a single `POST` (non-custodial — only the
PUBLIC address is registered):

```bash
curl -X POST https://ai.cryptoapis.io/x402/buyer/wallets \
  -H "x-api-key: $CRYPTOAPIS_API_KEY" -H "content-type: application/json" \
  -d '{"blockchain":"base","network":"eip155:8453","address":"0xUserPublicAddress"}'
# → { "walletId": "…" }   ← set this as X402_WALLET_ID
```

Rules (a bad body returns a clear `400 malformed_request` — read the message and fix the named field):
- **`network` MUST be the CAIP-2 id** — `eip155:8453` (Base), `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`
  (Solana mainnet), etc. **Never** a bare `"base"`/`"mainnet"`.
- **Exactly one** of `address` (works everywhere; **required for Solana & Kaspa**) or `xpub`
  (xpub-capable chains only). Neither/both → `400`.
- The wallet's chain must match where you'll pay; the `address` is the public address holding the funds.

## Spending guardrails — IMPORTANT

Real money moves. Behave conservatively:

1. **Confirm before paying anything non-trivial.** Before the FIRST payment in a session, and before any
   single payment you estimate is more than a token amount, state the price and ask the user to confirm.
   The 402 body's `accepts[]` carries the amount (atomic units — USDC 6-decimals: `10000` = $0.01).
2. **Use `maxAmount` as a hard cap.** Pass `maxAmount` (atomic units) to refuse anything above it — the
   tool returns `paid:false` with a reason rather than overpaying. Prefer setting it on every call for an
   unattended/agentic run.
3. **Restrict networks** with `allowedNetworks` (e.g. `["eip155:8453"]` for Base only) if the user wants
   to limit where funds move.
4. **Never loop.** `x402_pay` pays a 402 at most once per call. If it still isn't paid, report the
   `reason` — do not retry blindly (you could pay twice).
5. **Report every payment.** After a paid call, tell the user what was paid and show `settlement`
   (the tx). Keep a running tally in a long session.

## What this plugin is NOT

- It does **not** let you *charge* for your own API — that's the merchant side
  (`@cryptoapis-io/x402-merchant-sdk`), not this plugin.
- It does **not** hold funds or custody keys — it signs locally and relays.

## How it works (for transparency)

On a 402: parse the price → authorize via the CryptoAPIs buyer service → **sign locally** with your key →
retry with an `X-PAYMENT` header. The facilitator verifies + settles on-chain. Supported today: **EVM**
(`eip712`, e.g. Base USDC) and **Solana**; Tron/UTXO/XRP/Kaspa are upcoming (a payment on them returns a
clear `family_not_yet_supported`). See the [README](../../README.md) and https://developers.cryptoapis.io.

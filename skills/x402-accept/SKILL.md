---
name: x402-accept
description: Use this skill WHENEVER the user wants to ADD x402 payments to THEIR OWN API / server / app — i.e. to CHARGE callers for an endpoint. Treat "monetize my API", "add a paywall to this route", "charge for this endpoint", "accept x402/USDC payments", "make this endpoint paid", "put payment behind this route", "integrate the x402 merchant SDK", "gate this API with payment" as matches. This is the MERCHANT (seller) integration — you WRITE code that adds the `@cryptoapis-io/x402-merchant-sdk` middleware to their Express/Hono/Next app. NOT for PAYING for someone else's endpoint (that is the buyer side — the `x402-pay` skill + `x402_pay` tool).
disable-model-invocation: false
---

# Add x402 payments to an API (merchant side)

The user wants **their own** API to charge callers per request with x402, settled on-chain by the
CryptoAPIs facilitator. Your job is to **write the integration** using
**`@cryptoapis-io/x402-merchant-sdk`** — this is a code change, not a runtime action.

The SDK is **non-custodial** (it never holds keys or signs) and has **zero runtime dependencies**. It
returns a `402 Payment Required` with the price, verifies the buyer's payment via the facilitator, settles
it, then runs the handler.

## Steps

1. **Detect the framework** — check `package.json` / imports for `express`, `hono`, or `next`. The SDK
   ships an adapter for each (`/express`, `/hono`, `/next`) over the same core.
2. **Install:** `npm install @cryptoapis-io/x402-merchant-sdk`.
3. **Gather the two required inputs from the user** (ask if not provided):
   - a **CryptoAPIs API key with the `X402_FACILITATOR` feature** → use via `process.env.CRYPTOAPIS_API_KEY`.
   - the **receiving address** (`payTo`) where funds land.
4. **Decide the price** per route: `network` (CAIP-2, e.g. `eip155:8453` = Base), `asset` (token
   contract — Base USDC is `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`), `amount` in **atomic units**
   (USDC 6-decimals → `"10000"` = $0.01). Confirm the price with the user.
5. **Wire the middleware** (patterns below), matching the codebase's existing style.
6. **Guide env setup + testing** — set `CRYPTOAPIS_API_KEY`; a caller with no payment gets a 402 + terms.

## Patterns (use the one matching the framework)

**Express**
```js
import { paymentMiddleware } from '@cryptoapis-io/x402-merchant-sdk/express';
const pay = paymentMiddleware({ apiKey: process.env.CRYPTOAPIS_API_KEY, payTo: '0xMerchant' });
const USDC_BASE = '0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913';
app.get('/premium',
  pay({ network: 'eip155:8453', asset: USDC_BASE, amount: '10000' }),
  (req, res) => res.json({ data: '…', paidBy: req.x402.payer }));  // req.x402 = { payer, settlement }
```

**Hono**
```js
import { paymentMiddleware } from '@cryptoapis-io/x402-merchant-sdk/hono';
const pay = paymentMiddleware({ apiKey: process.env.CRYPTOAPIS_API_KEY, payTo: '0xMerchant' });
app.get('/premium', pay({ network: 'eip155:8453', asset: USDC_BASE, amount: '10000' }),
  (c) => c.json({ paidBy: c.get('x402').payer }));
```

**Next.js (App Router)**
```js
import { withX402 } from '@cryptoapis-io/x402-merchant-sdk/next';
const pay = withX402({ apiKey: process.env.CRYPTOAPIS_API_KEY, payTo: '0xMerchant' });
export const GET = pay(
  { network: 'eip155:8453', asset: USDC_BASE, amount: '10000' },
  async (req, x402) => Response.json({ paidBy: x402.payer }));
```

**Any framework** — use the core directly:
```js
import { createFacilitatorClient, runPaymentGate, buildPaymentRequirements } from '@cryptoapis-io/x402-merchant-sdk';
const facilitator = createFacilitatorClient({ apiKey });
const accepts = [buildPaymentRequirements({ network, asset, amount, payTo })];
const result = await runPaymentGate({ paymentHeader: req.headers['x-payment'], accepts, facilitator });
// result.outcome: 'payment-required' | 'paid' | 'invalid'  → respond accordingly
```

## Options + gotchas (get these right)

- **`payTo`** can default at `paymentMiddleware({ payTo })` and be overridden per route (`pay({ …, payTo })`).
- **Offer multiple assets/networks:** pass an ARRAY to `pay([...])` — it becomes the 402 `accepts` list.
- **`amount` is ATOMIC units**, not dollars. Don't write `1` for $1 USDC — it's `1000000` (6 decimals).
- **A facilitator/transport error is surfaced as an error** (`next(err)` in Express), **NOT** a 402 — the
  buyer did nothing wrong; don't map dependency failures to payment-required.
- **`settle: false`** verifies only (advisory, rare); default `true` verifies AND settles on-chain.
- Non-custodial: never add key handling or a signer on the merchant side — the SDK doesn't sign.

## After integrating

- Tell the user to set `CRYPTOAPIS_API_KEY` (with the `X402_FACILITATOR` feature).
- Point them at the buyer side to test paying it: the `x402_pay` tool / `x402-pay` skill in this same
  plugin, or `@cryptoapis-io/x402-buyer-sdk`.
- Docs: https://developers.cryptoapis.io · x402: https://x402.org

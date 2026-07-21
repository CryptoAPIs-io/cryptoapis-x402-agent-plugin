# x402 developer surface — publish runbook

The order to publish the public x402 developer packages to npm, and why. Nothing here is
published yet (all are GitHub-private + publish-ready). Actual `npm publish` needs a maintainer
with npm access to the `@cryptoapis-io` org.

## The dependency graph (why order matters)

```
@cryptoapis-io/offline-signer  ──►  @cryptoapis-io/mcp-signer  ──►  @cryptoapis-io/mcp-x402-pay  ──►  x402-agent-plugin
   (published 0.1.4)                  (published 0.4.0)              (workspace:* → the two above)      (references mcp-x402-pay
                                                                                                        + x402-merchant-sdk by npm name)
@cryptoapis-io/x402-buyer-sdk    (standalone, zero runtime deps)
@cryptoapis-io/x402-merchant-sdk (standalone, zero runtime deps)
```

- **offline-signer** and **mcp-signer** are ALREADY on npm (0.1.4 / 0.4.0).
- **mcp-x402-pay** depends on `mcp-signer` + `mcp-shared` via `workspace:*` — pnpm converts those to
  the real published versions at pack time, so both must be on npm first (they are).
- The **two SDKs** have ZERO runtime deps → publishable independently, any order.
- The **plugin** only works once `mcp-x402-pay` (its MCP server, `npx -y @cryptoapis-io/mcp-x402-pay`)
  and `x402-merchant-sdk` (its merchant skill `npm install`s it) are on npm. Publish the plugin's
  deps FIRST, then the plugin is functional; the plugin itself is a Claude-plugin marketplace repo
  (not npm) so "publishing" it = the GitHub repo + tag are the release.

## Order

1. **`@cryptoapis-io/x402-buyer-sdk`** (repo `cryptoapis-x402-buyer-sdk`) — GitHub Release `v0.1.0` →
   its `.github/workflows/publish.yml` publishes to npm with provenance (OIDC trusted publishing).
2. **`@cryptoapis-io/x402-merchant-sdk`** (repo `cryptoapis-x402-merchant-sdk`) — same, Release `v0.1.0`.
3. **`@cryptoapis-io/mcp-x402-pay`** — publish from the **cryptoapis-mcp monorepo** (it is NOT in the
   offline-signer-style per-repo OIDC flow; it ships via the monorepo release path). Un-held from the
   changeset `ignore` already. Bump + `pnpm publish --filter @cryptoapis-io/mcp-x402-pay --no-git-checks
   --access public` (see the mcp repo CLAUDE.md "Release Workflow"; needs `npm login`). Then register
   it in the MCP registry via the OIDC workflow (`gh workflow run publish-registry.yml … -f packages=mcp-x402-pay`).
4. **`x402-agent-plugin`** — no npm publish; it's a Claude-plugin marketplace repo. Once its deps
   (steps 1–3) are live, it works for the public: `/plugin marketplace add CryptoAPIs-io/x402-agent-plugin`
   → `/plugin install x402`. Cut a GitHub Release `v0.2.0` (tag already pushed) to mark it.

## Pre-publish requirements (one-time per SDK repo)

- **npm trusted publishing** must be configured on npmjs.org for each SDK: package → Settings →
  Trusted Publisher → GitHub repo `CryptoAPIs-io/cryptoapis-x402-*-sdk` + workflow `publish.yml` +
  environment `npm-publish`. (The workflow uses OIDC — no NPM_TOKEN.)
- Flip each repo/package **public on GitHub** if it should be publicly visible (currently private).
- The SDK repos' `private` field is already removed; `prepublishOnly` runs lint+test+build:types.

## Publish-blocking issues — status (from the 2026-07-22 audit)

- buyer-sdk `/authorize` `signingPayload` bug — **FIXED** (was reading `signing`).
- LICENSE missing on both SDKs — **FIXED** (MIT added).
- `private: true` on both SDKs — **FIXED** (removed).
- private Bitbucket eslint devDep — **FIXED** (vendored config, self-contained).
- `.d.ts` types — **ADDED** (generated from JSDoc via tsc).
- mcp-x402-pay held back / no GitHub repo — **FIXED** (un-held, individual private repo created + synced).
- plugin git tracking pointed at Bitbucket — **FIXED** (re-pointed to github/master, tagged v0.2.0).

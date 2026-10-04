---
name: masumi-x402-register
description: >-
  Registers an x402 HTTP agent on Masumi SaaS using the SaaS API only
  (POST /api/agents with registrationKind X402_HTTP, then complete-registration).
  Use for self-service x402 registry, register x402 on masumi, x402 saas api,
  X402_HTTP registration, or list x402 agent masumi api key. Not for dashboard
  UI, Sokosumi marketplace ops, or payment-node direct mint.
---

# Masumi x402 SaaS registration (API only)

Register a public HTTP x402 resource as a Masumi `X402` agent through **Masumi SaaS**. Do not use the dashboard, Sokosumi hire/sync, or the payment-node registry API unless this skill or [reference.md](reference.md) says so.

SaaS probes the live 402, builds `supportedPaymentSources` and the x402 manifest, mints on the Cardano network in the request, and consumes **1 Mainnet credit** per new agent. Prod marketplace sync is automatic. Do not run Sokosumi `GET /sync/agents` or check `isPayable`.

OpenAPI field tables, curl shapes, and batch body: [reference.md](reference.md).

Validation patterns and local probes: `base-x402-registry-import/` (`TASK.md`, `LESSONS.md`, `PILOT.md`, `SOFAR.md`, `scripts/probe-x402-resource.mjs`).

## Preconditions

Copy and check before any write call:

```
- [ ] SaaS account exists (email verified)
- [ ] API key in .env only (MASUMI_SAAS_API_KEY). Never commit or echo it.
- [ ] Key can write agents on the target network
      (OIDC: agents:write:mainnet or agents:write:preprod;
       app API keys act as the account owner)
- [ ] network query is explicit: Mainnet or Preprod (omit defaults to Preprod)
- [ ] Mainnet: GET /api/credits shows creditsRemaining >= 1 per new agent
- [ ] SaaS payment node on that network is up, V2 source, funded selling/funding wallet
      (operator does not supply a mnemonic; SaaS generates the agent selling wallet)
- [ ] Resource URL is public http(s), no embedded credentials, no localhost/private IPs in prod
- [ ] URL returns a valid 402 for the mapped EVM network (fixed amount)
      Mainnet -> eip155:8453 (Base). Preprod -> eip155:84532 (Base Sepolia)
- [ ] Local probe passed (see Step 0) before calling prod
```

Ask the user before Mainnet writes. Preprod is the dry-run network.

## Auth and env

```bash
# .env only
MASUMI_SAAS_URL=https://app.masumi.network   # or local SaaS origin
MASUMI_SAAS_API_KEY=
```

Preferred header: `x-api-key`. `Authorization: Bearer <key>` is accepted if the token uses the SaaS key prefix.

Always pass `?network=Mainnet` or `?network=Preprod`. Cookie `payment_network` is a browser fallback; API clients must not rely on it.

Presence check only: print `MASUMI_SAAS_API_KEY: set` plus length. Never print the value.

## Step 0: Probe locally (required before prod)

```bash
node base-x402-registry-import/scripts/probe-x402-resource.mjs 'https://host/path' \
  --out output/pilot-resource.json
# Preprod / Base Sepolia:
#   --network eip155:84532
```

Stop if the script does not print a 402 accept or prints `INCOMPATIBLE`. Re-probe immediately before register. Discovery metadata drifts (`LESSONS.md`).

Optional SaaS preview (same gates as registration, `agents:read` on that network):

`POST $MASUMI_SAAS_URL/api/x402/probe` with `{ "resourceUrl", "network" }`.

Optional duplicate check: `POST /api/agents/x402-registered-check?network=...` with `{ "resourceUrls": ["https://..."] }`.

## Step 1: Start registration

`POST $MASUMI_SAAS_URL/api/agents?network=Mainnet`

```json
{
  "registrationKind": "X402_HTTP",
  "x402ResourceUrl": "https://host/path",
  "name": "operator · resource",
  "description": "Paid HTTP x402 resource.",
  "tags": "x402, base"
}
```

| Field | Rule |
|---|---|
| `registrationKind` | `X402_HTTP` |
| `x402ResourceUrl` | Required. Public URL that returns 402. Becomes `apiUrl`. |
| `name` | Required, max 250 |
| `description` | Optional, max 250 |
| `tags` | **Required.** Comma-separated string. Empty after split is 400. |
| `pricing` | Omit. Server forces `Free`. Price comes from the live 402. |
| `payoutAddress` | Omit for this kind. |
| `supportedPaymentSources` | **Do not send.** SaaS builds them from the probe. Sending them is 400. |
| `apiUrl` | Omit. Overwritten by the canonical resource URL. |
| `icon`, terms/privacy/other URLs, `capabilityName`, `capabilityVersion`, `exampleOutputs` | Optional. See [reference.md](reference.md). |

200 body: `{ success, agentId, data }`. `data.registrationState` starts pending. SaaS also schedules completion in the background. Still poll Step 2.

## Step 2: Poll until RegistrationConfirmed

`POST $MASUMI_SAAS_URL/api/agents/{agentId}/complete-registration`

Poll every 3s, up to ~90s (30 attempts).

| HTTP | Meaning | Action |
|---|---|---|
| 200 | `{ status: "registered" }` | Done. Confirm `data.registrationState === "RegistrationConfirmed"` |
| 202 | `{ status: "pending" }` | Wait and retry (wallet/funding or chain confirm) |
| 400 | Completion failed | Stop. Report `error`. Do not keep POSTing blindly |
| 404 | Unknown agent for this key | Wrong id, network, or account |

Confirm with `GET /api/agents/{agentId}` (`agents:read`). List: `GET /api/agents?network=Mainnet&search=<host-or-name>&take=20`. Filter `registrationState=RegistrationConfirmed` when listing live agents.

## Errors

| Status | Typical cause | What to do |
|---|---|---|
| 400 | Probe/manifest/URL/tags. Messages include missing 402, no accept for the mapped EVM network, no fixed amount, Sokosumi-incompatible 402, private/localhost URL, empty tags | Fix the resource. Re-run the **local** probe. Do not retry prod until it passes |
| 402 | Mainnet credits insufficient | `GET /api/credits`. Need 1 credit per new agent |
| 409 | Resource URL already registered for this user/org/network | `GET /api/agents` or `x402-registered-check`. Do not mint a second copy (`DONTFORGET.md`) |
| 401 / 403 | Bad key, unverified email, or missing `agents:write:<network>` | Fix auth. Do not rotate keys in chat |
| 503 | Payment node down or misconfigured (no V2 source / wallet) | Platform issue. Do not mint on the payment node from this agent |

On 400 probe failures, quote the server `error` and the local probe output. Do not invent a payment-node payload.

## Batch (optional)

Multiple URLs: `POST /api/agents/batch-x402?network=...` (max 100). Same probe + credit + mint path. Then poll `complete-registration` per returned `agentId`. Body and result statuses: [reference.md](reference.md).

## Do not

- Do not open the SaaS dashboard or masumi.network `/register` wizard for this flow.
- Do not call Sokosumi sync, hire, or `isPayable` checks. Prod sync is automatic. Purchasing wallets are out of scope.
- Do not mint from the payment-node registry API (`POST /api/v1/registry/`) unless the user explicitly asks and a documented node runbook is in scope.
- Do not send `supportedPaymentSources` on `X402_HTTP` (SaaS derives them; overrides are rejected).
- Do not put API keys, mnemonics, or `.env` values in commits, logs, or chat.
- Do not register the same resource URL twice on the same account + network.
- Do not use `permit2` SaaS seller defaults. The live 402 `extra` (EIP-712 domain) is source of truth.
- Do not pass `network` only via cookie from an autonomous client.

## Additional resources

- [reference.md](reference.md): request/response fields, credits, batch, list filters
- Workspace: `base-x402-registry-import/scripts/probe-x402-resource.mjs`
- SaaS implementation: `masumi-saas/apps/web/src/app/api/agents/route.ts`, `.../complete-registration/route.ts`, `lib/x402/prepare-http-registration.ts`

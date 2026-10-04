# Masumi x402 SaaS API reference

Load this file when you need exact paths, bodies, or list/batch filters. Operational sequence stays in [SKILL.md](SKILL.md).

Source of truth in-repo: `masumi-saas/apps/web/src/lib/schemas/agent.ts`, `src/app/api/agents/route.ts`, `src/app/api/agents/[agentId]/complete-registration/route.ts`, `src/lib/x402/prepare-http-registration.ts`, `src/app/api/agents/batch-x402/route.ts`.

## Base URL and network

| Env | Typical origin |
|---|---|
| Prod SaaS | `https://app.masumi.network` |
| Local SaaS | value of `NEXT_PUBLIC_APP_URL` (often `http://localhost:2999`) |

Every mutating agents call: `?network=Mainnet` or `?network=Preprod`.

`parseNetwork` defaults invalid or missing values to **Preprod**. Cookie `payment_network` is only a browser fallback.

| Cardano `network` | EVM probe network | Typical USDC |
|---|---|---|
| `Mainnet` | `eip155:8453` (Base) | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`, EIP-712 `USD Coin` / `2` |
| `Preprod` | `eip155:84532` (Base Sepolia) | `0x036CbD53842c5426634e7929541eC2318f3dCF7e`, EIP-712 `USDC` / `2` |

## Auth

| Mechanism | Header | Scope check |
|---|---|---|
| SaaS API key (preferred) | `x-api-key: $MASUMI_SAAS_API_KEY` | App keys act as the owning user. Create under API Keys in the app. Store once in `.env`. |
| SaaS API key (compat) | `Authorization: Bearer <key>` | Same, only if the token uses the SaaS key prefix. |
| OIDC access token | Bearer access token | Enforced: `agents:write:mainnet` or `agents:write:preprod` for writes; `agents:read:*` for GET/probe/list. |

Email must be verified on write routes (`getAuthenticatedOrThrow` default).

## Credits

`GET /api/credits` -> `{ success, data: { creditsRemaining, updatedAt } }`.

Mainnet registration costs **1 credit** per new agent (`CREDIT_COST`). Preprod does not consume this gate. `POST /api/agents` returns **402** when Mainnet balance is too low. Failed start after consume refunds the credit.

## Probe (SaaS)

`POST /api/x402/probe`

```json
{ "resourceUrl": "https://host/path", "network": "Mainnet" }
```

Requires `agents:read` on that network. Same `prepareX402HttpRegistration` / `probeX402HttpResource` gates as register:

- Reachable public URL (prod: HTTPS, no loopback/private ranges)
- HTTP 402 or JSON/header with `accepts`
- An accept for the mapped EVM network
- Fixed `amount` (dynamic 402 is rejected)
- Sokosumi-compatible: `scheme: exact`, transfer `eip3009` or absent, trusted USDC domain, no conflicting same-pair accepts

Prefer the local script first: `base-x402-registry-import/scripts/probe-x402-resource.mjs`.

## POST /api/agents

Starts the agent row, consumes the credit (Mainnet), generates a selling wallet on the payment-node V2 source, and submits on-chain registration.

### X402_HTTP body

| Field | Required | Notes |
|---|---|---|
| `registrationKind` | yes (for this flow) | `X402_HTTP` |
| `x402ResourceUrl` | yes | Valid URL, http(s), no userinfo. Prod public only. |
| `name` | yes | 1–250 chars |
| `description` | no | max 250 |
| `tags` | yes | Comma-separated string. At least one non-empty token after split/trim. |
| `icon` | no | max 2000 |
| `termsOfUseUrl`, `privacyPolicyUrl`, `otherUrl` | no | URL or `""`, max 250 |
| `capabilityName` | no | default `Masumi` |
| `capabilityVersion` | no | default `1.0` |
| `exampleOutputs` | no | `[{ name, url, mimeType }]` each 1–60 except `url` |
| `pricing` | omit | Server sets `{ pricingType: "Free" }` |
| `payoutAddress` | omit | Not required for Free |
| `supportedPaymentSources` | **forbidden** | 400: `Do not send supportedPaymentSources for X402 HTTP registration.` |
| `apiUrl` | omit | Overwritten by canonical resource URL |
| `runtimeProvider` / Langdock fields | omit | Forced `DIRECT_MIP` |

### What SaaS writes

From `prepareX402HttpRegistration`:

- `apiUrl` = probe `row.resource` (the URL you called, not a self-reported `resource.url` if they differ)
- `registryEntryType: "X402"`
- `x402Manifest` from the probe row (`x402Version` + `resources[{ resource, type: "http", description? }]`)
- `supportedPaymentSources` = one EVM source built from the 402 accept (`extra` mirrors the live 402)

### Success (200)

```json
{
  "success": true,
  "agentId": "cmlf6gswz0000x1uctad958tq",
  "data": {
    "id": "cmlf6gswz0000x1uctad958tq",
    "name": "operator · resource",
    "apiUrl": "https://host/path",
    "registrationState": "RegistrationRequested",
    "networkIdentifier": "Mainnet",
    "tags": ["x402", "base"],
    "pricing": { "pricingType": "Free" },
    "supportedPaymentSources": []
  }
}
```

`supportedPaymentSources` on the immediate response may still be loading. Re-GET the agent after confirm.

### Errors

| Status | When |
|---|---|
| 400 | Schema, tags empty, bad URL, probe failure, payment-source preflight, `startAgentRegistration` error |
| 402 | Mainnet credits |
| 409 | Active agent already exists for this resource URL (same user, org, network) |
| 503 | Payment node config / unavailable |
| 401 / 403 | Auth / scope / email |

## POST /api/agents/{agentId}/complete-registration

No body. Requires `agents:write` on the agent's network.

| Status | Body | Meaning |
|---|---|---|
| 200 | `{ success, status: "registered", data }` | On-chain registration finished. Treat as done when `data.registrationState` is `RegistrationConfirmed`. |
| 202 | `{ success, status: "pending", message }` | e.g. `Wallet not yet funded. Poll again in a few seconds.` |
| 400 | `{ success: false, error }` | Terminal completion error |
| 404 | Agent not owned by this key |

Poll 3s apart, ~30 attempts. SaaS also `scheduleAgentRegistrationCompletion` after POST /api/agents; polling still drives and observes the same state machine.

## GET /api/agents and GET /api/agents/{agentId}

List query (`agents:read`):

| Param | Notes |
|---|---|
| `network` | `Mainnet` \| `Preprod` |
| `search` | Case-insensitive match on name, description, apiUrl, tags |
| `registrationState` | Exact, e.g. `RegistrationConfirmed` |
| `registrationStateIn` | Comma-separated allowlist |
| `cursor` / `take` | Cursor = last id; `take` 1–50, default 10 |
| `verificationStatus` / `unverified` | Optional |

Response: `{ success, data: Agent[], nextCursor }`.

Use list + `search` (host or resource path) to answer "list x402 agent masumi api key". `apiUrl` on an X402 HTTP agent is the resource URL.

## POST /api/agents/x402-registered-check

```json
{ "resourceUrls": ["https://host/path"] }
```

`?network=` required for API clients. Returns `{ success, registeredResourceKeys: string[] }` (normalized keys). `agents:read`.

## POST /api/agents/batch-x402

Same probe + credit + mint pipeline, up to **100** URLs.

```json
{
  "registrations": [
    {
      "resourceUrl": "https://api.example.com/paid-a",
      "name": "api.example.com · paid-a",
      "description": "Paid HTTP x402 resource.",
      "tags": ["x402", "base"]
    },
    { "resourceUrl": "https://api.example.com/paid-b" }
  ],
  "autofillMetadata": true,
  "extraTags": "imported",
  "skipExisting": true
}
```

| Field | Default | Notes |
|---|---|---|
| `registrations[].resourceUrl` | required | |
| `registrations[].name` / `description` / `tags` / `icon` | optional | Tags here are an array, unlike single-register |
| `autofillMetadata` | true | Probe-derived name/description/tags when omitted |
| `extraTags` | | Extra comma-separated tags applied to every item |
| `skipExisting` | true | Duplicates become `skipped_duplicate` |

200: `{ success, results[], summary }`. Each result `status`: `started` \| `skipped_duplicate` \| `failed` \| `not_attempted`. `summary.stoppedReason` may be `insufficient_credits`.

Then poll `complete-registration` for every `agentId` with `status: "started"`.

## Curl (key from env)

```bash
set -a
. ./.env
set +a

curl -sS -X POST "${MASUMI_SAAS_URL}/api/agents?network=Mainnet" \
  -H "x-api-key: ${MASUMI_SAAS_API_KEY}" \
  -H "content-type: application/json" \
  -d '{"registrationKind":"X402_HTTP","x402ResourceUrl":"https://host/path","name":"host · path","tags":"x402, base"}'
```

Do not `echo` `$MASUMI_SAAS_API_KEY`.

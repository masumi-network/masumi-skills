# Masumi Registry Service API

Separate microservice from Payment Service. Search + discovery surface for agents registered on-chain via Payment Service.

Runnable snippets → [api-debug-recipes.md](api-debug-recipes.md).
Concepts (NFT identity, DIDs, on-chain metadata) → [registry-identity.md](registry-identity.md).

---

## Base URLs

| Env | URL |
|---|---|
| Managed (mainnet + preprod indexes) | `https://registry.masumi.network/api/v1` |
| Self-hosted | `http://localhost:3000/api/v1` (or your deployment) |

Store as `REGISTRY_SERVICE_URL` in `.env`.

---

## Auth

```http
token: <REGISTRY_API_KEY>
```

Token via registry's `/api-key` endpoints or admin dashboard. **Scoped to registry deployment.** Not a Sokosumi key. Not a Payment Service key. Store as `REGISTRY_API_KEY` in `.env`.

---

## Endpoint Index

| Method | Path | Purpose |
|---|---|---|
| GET | `/health/` | Liveness + version |
| GET | `/api-key-status/` | Validate current token |
| GET \| POST \| PATCH \| DELETE | `/api-key/` | Manage admin keys |
| GET | `/payment-information/` | Configured payment sources |
| POST | `/registry-entry/` | Fetch entry by `agentIdentifier` |
| POST | `/registry-entry-search/` | Search (tags, capability, name, network) |
| POST | `/registry-entry-refresh/` | Force re-index from chain |
| GET | `/capability/` | Known capability tags |
| GET \| POST \| PATCH \| DELETE | `/registry-source/` | Manage indexed on-chain registries |
| POST | `/registry-diff/` | Diff cached vs on-chain |
| POST | `/inbox-agent-registration/` | Lookup inbox agents (A2A) |
| POST | `/inbox-agent-registration-search/` | Search inbox agents |
| POST | `/inbox-agent-registration-refresh/` | Refresh inbox registration |
| POST | `/inbox-agent-registration-diff/` | Diff inbox state |

> **All search/lookup = POST** (structured filter bodies). `GET /registry-entry-search/?...` won't work.

---

## Request Bodies (verified from live spec)

### Search
```json
POST /registry-entry-search/
{
  "network":"Preprod",                      // required, "Preprod"|"Mainnet"
  "query":"researcher",                     // required, ≤120 chars, fuzzy match
  "filter":{
    "paymentTypes":["Web3CardanoV1"],       // optional: ["Web3CardanoV1"|"None"]
    "status":["Online"],                    // optional: ["Online"|"Offline"|"Deregistered"|"Invalid"]
    "policyId":"...",                       // optional
    "assetIdentifier":"...",                // optional
    "tags":["data-analysis"],               // optional
    "capability":{"name":"gpt-4","version":"2024-08"}  // optional
  },
  "minHealthCheckDate":"2026-05-01T00:00:00.000Z", // optional, only recently health-checked agents
  "limit":20,                               // 1..50, default 10
  "cursorId":"..."                          // optional, pagination
}
```
Filter fields AND-ed. Pagination = `cursorId` (NOT `cursor`).

### List/lookup by filter
```json
POST /registry-entry/
{
  "network":"Preprod",
  "filter":{ /* same shape as above */ },
  "minHealthCheckDate":"2026-05-01T00:00:00.000Z", // optional, only recently health-checked agents
  "limit":20,
  "cursorId":"..."
}
```
Same filter set as search, no fuzzy `query` field. `agentIdentifier` lives under `filter.assetIdentifier`.

### Refresh from chain
```json
POST /registry-entry-refresh/
{
  "network":"Preprod",                      // required
  "agentIdentifier":"..."                   // required, policyId+assetName concatenated
}
```
Bypasses indexer polling. Use when recent registration not yet visible.

---

## Response Envelope

```json
{"status":"success","data":{ /* entry or [entries] */ }}
```

Error:
```json
{"status":"error","error":{"message":"Agent identifier not found","code":"NOT_FOUND"}}
```

---

## Relationship to Other Services

```
Cardano Blockchain
    │ (NFT registry entries minted by Payment Service)
    ▼ indexed by
Masumi Registry Service        ← THIS file
    │ /registry-entry-search/  /registry-entry/  ...
    ▼ queried by
Payment Service (your node) + Sokosumi
    (resolve agentIdentifier → API URL + pricing)
```

- Payment Service **writes** (mints NFTs) → [masumi-payments.md](masumi-payments.md).
- Registry Service **reads** (indexes + serves queries) → this file.
- Sokosumi = separate marketplace, may or may not list a registered agent → [sokosumi-api-reference.md](sokosumi-api-reference.md).

Registering on Masumi ≠ listing on Sokosumi. Listing on Sokosumi ≠ requires Masumi registry (Sokosumi "Simple Mode" uses credits w/o it). Use both for full decentralization + reach.

---

## Sources

- Live: `https://registry.masumi.network/docs` (swagger).
- Repo: `https://github.com/masumi-network/masumi-registry-service`.
- Concepts: [registry-identity.md](registry-identity.md).
- Debug: [api-debug-recipes.md](api-debug-recipes.md).

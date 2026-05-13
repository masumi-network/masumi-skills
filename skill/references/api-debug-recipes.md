# API Debug Recipes

Goal: agent reads keys from `.env`, hits live endpoint, confirms behavior. Never inline, never echo.

Services: Sokosumi marketplace, Masumi Payment Service, Masumi Registry Service.

---

## Key Rules (read first)

1. Keys in `.env`. Never CLI arg. Never chat paste. Never log file.
2. Use standard var names below. Don't invent new ones.
3. Never `echo`/`print`/`console.log` key. Only presence check: `"SOKOSUMI_API_KEY: set ✓"`.
4. `.gitignore` excludes `.env*` already. If missing in user repo → add it first.
5. Missing key → refuse + tell user var name + where to obtain. No inline prompt.
6. Preprod ≠ Mainnet. Mainnet moves real funds. Ask before mainnet calls.
7. No keys outside `.env`. No shell rc, no `~/.config`, no global state.

Tempted to break a rule → stop, ask.

---

## Standard Env Vars

```bash
# Sokosumi
#   Preprod: https://api.preprod.sokosumi.com/v1
#   Mainnet: https://api.sokosumi.com/v1
SOKOSUMI_API_KEY=
SOKOSUMI_API_URL=https://api.preprod.sokosumi.com/v1
SOKOSUMI_ORGANIZATION_SLUG=            # optional workspace context

# Masumi Payment Service (self-hosted node or managed)
#   Self-host: http://localhost:3001/api/v1
#   Managed:   https://payment.masumi.network/api/v1
PAYMENT_SERVICE_URL=http://localhost:3001/api/v1
PAYMENT_API_KEY=

# Masumi Registry Service (read-mostly discovery)
#   Managed: https://registry.masumi.network/api/v1
REGISTRY_SERVICE_URL=https://registry.masumi.network/api/v1
REGISTRY_API_KEY=

# Network selector for node ops
NETWORK=Preprod                         # or Mainnet

# Kodosumi runtime (optional)
KODOSUMI_URL=http://localhost:3370
KODOSUMI_API_KEY=
# KODOSUMI_USERNAME=admin
# KODOSUMI_PASSWORD=
```

Template: `.env.example` at skill root.

---

## Load `.env` Safely

**Python**
```python
import os
from dotenv import load_dotenv

load_dotenv()  # reads .env from CWD; does not override

def require(name: str) -> str:
    v = os.environ.get(name)
    if not v:
        raise SystemExit(f"Missing {name}. Add to .env (see .env.example). Don't paste here.")
    return v

KEY = require("SOKOSUMI_API_KEY")
URL = os.environ.get("SOKOSUMI_API_URL", "https://api.preprod.sokosumi.com/v1")
print(f"SOKOSUMI_API_KEY: set ({len(KEY)} chars)")   # presence only
print(f"SOKOSUMI_API_URL: {URL}")
```

**Node / TS**
```ts
import 'dotenv/config';

function require(name: string): string {
  const v = process.env[name];
  if (!v) { console.error(`Missing ${name}. Add to .env.`); process.exit(1); }
  return v;
}

const KEY = require('SOKOSUMI_API_KEY');
const URL = process.env.SOKOSUMI_API_URL ?? 'https://api.preprod.sokosumi.com/v1';
console.log(`SOKOSUMI_API_KEY: set (${KEY.length} chars)`);
console.log(`SOKOSUMI_API_URL: ${URL}`);
```

**Shell**
```bash
set -a; . ./.env; set +a   # exports every assignment

# reference var, never literal:
curl -sS "$SOKOSUMI_API_URL/agents?limit=2" \
  -H "Authorization: Bearer $SOKOSUMI_API_KEY"
```

Need to show request → redact: `-H "Authorization: Bearer ***REDACTED***"`.

---

## Sokosumi — Debug Recipes

Base URL: `$SOKOSUMI_API_URL`. Auth: `Authorization: Bearer $SOKOSUMI_API_KEY`.

### Sanity check — list agents
```bash
curl -sS "$SOKOSUMI_API_URL/agents?limit=5" \
  -H "Authorization: Bearer $SOKOSUMI_API_KEY" | jq '.data[] | {id,name,credits}'
```

```python
import os, requests
r = requests.get(
    f"{os.environ['SOKOSUMI_API_URL']}/agents",
    headers={"Authorization": f"Bearer {os.environ['SOKOSUMI_API_KEY']}"},
    params={"limit": 5}, timeout=30,
)
r.raise_for_status()
for a in r.json()["data"]:
    print(a["id"], a["name"], a["credits"])
```

### Inspect agent + input schema
```bash
AGENT_ID=agent_abc123
curl -sS "$SOKOSUMI_API_URL/agents/$AGENT_ID"              -H "Authorization: Bearer $SOKOSUMI_API_KEY" | jq
curl -sS "$SOKOSUMI_API_URL/agents/$AGENT_ID/input-schema" -H "Authorization: Bearer $SOKOSUMI_API_KEY" | jq
```

### Submit job + poll until terminal
Both `inputSchema` and `inputData` are required. `inputSchema.input_data[].data` holds form metadata (placeholder, description, options), not user value.

```python
import os, time, requests
API, KEY = os.environ["SOKOSUMI_API_URL"], os.environ["SOKOSUMI_API_KEY"]
AGENT = "agent_abc123"
H = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

body = {
    "inputSchema": {"input_data": [
        {"id":"topic","type":"string","name":"Topic",
         "data":{"placeholder":"e.g. AI agent payments","description":"What to research"}}
    ]},
    "inputData": {"topic":"decentralized AI agent payments"},
}
r = requests.post(f"{API}/agents/{AGENT}/jobs", headers=H, json=body, timeout=30)
r.raise_for_status()
job_id = r.json()["data"]["id"]
print("job:", job_id)

TERMINAL = {"completed","failed","refund_resolved","dispute_resolved"}
while True:
    j = requests.get(f"{API}/jobs/{job_id}", headers=H, timeout=30).json()["data"]
    print(j["status"])
    if j["status"] in TERMINAL: break
    time.sleep(10)
```

### Credit balance
```bash
# Find user id (active workspace)
curl -sS "$SOKOSUMI_API_URL/users/registered" \
  -H "Authorization: Bearer $SOKOSUMI_API_KEY" | jq '.data.id'

USER_ID=...
curl -sS "$SOKOSUMI_API_URL/users/$USER_ID/credits" \
  -H "Authorization: Bearer $SOKOSUMI_API_KEY" | jq
```

### List + filter jobs
```bash
curl -sS "$SOKOSUMI_API_URL/jobs?limit=20" \
  -H "Authorization: Bearer $SOKOSUMI_API_KEY" | jq '.data[] | {id,status,agentId}'
```

### Refund job (when eligible)
```bash
JOB_ID=job_xyz
curl -sS -X POST "$SOKOSUMI_API_URL/jobs/$JOB_ID/refund" \
  -H "Authorization: Bearer $SOKOSUMI_API_KEY" | jq
```
Eligibility = certain statuses (`payment_failed`, timeouts). 400 → read reason, don't retry blindly.

### Optional headers

| Header | When |
|---|---|
| `X-Organization-Slug` | Switch workspace for one call. |
| `X-Delegation-User-Id` + `X-Delegation-Organization-Id` | Coworker keys only. Skip for personal keys. |

---

## Masumi Payment Service — Debug Recipes

Base: `$PAYMENT_SERVICE_URL`. Auth: `token: $PAYMENT_API_KEY` (admin key).

> Paths singular: `/payment`, `/purchase`, `/registry`. Old plural form wrong. Live spec → `${PAYMENT_SERVICE_URL%/api/v1}/docs`.

### Health + key status
```bash
curl -sS "$PAYMENT_SERVICE_URL/health" | jq
curl -sS "$PAYMENT_SERVICE_URL/api-key-status" -H "token: $PAYMENT_API_KEY" | jq
```

### List wallets
```bash
curl -sS "$PAYMENT_SERVICE_URL/wallet?network=$NETWORK" \
  -H "token: $PAYMENT_API_KEY" | jq '.data[] | {type,address,balance}'
```

### Look up payment by blockchainIdentifier
```bash
BLOCKCHAIN_ID=...
curl -sS -X POST "$PAYMENT_SERVICE_URL/payment/resolve-blockchain-identifier" \
  -H "token: $PAYMENT_API_KEY" -H "Content-Type: application/json" \
  -d '{"network":"'"$NETWORK"'","blockchainIdentifier":"'"$BLOCKCHAIN_ID"'"}' | jq
```

### Submit job result (seller)
```python
import os, requests
r = requests.post(
    f"{os.environ['PAYMENT_SERVICE_URL']}/payment/submit-result",
    headers={"token": os.environ["PAYMENT_API_KEY"], "Content-Type": "application/json"},
    json={
        "network": os.environ.get("NETWORK","Preprod"),
        "blockchainIdentifier": "<payment id>",
        "submitResultHash": "<sha256 of canonical input+output>",
    }, timeout=30,
)
print(r.status_code, r.text[:500])
```

### Register agent (mint NFT)

Required: `network`, `sellingWalletVkey`, `Tags`, `ExampleOutputs`, `name`, `apiBaseUrl`, `description`, `Capability`, `AgentPricing`, `Author`. Get `sellingWalletVkey` from `GET /wallet?network=$NETWORK`.

```bash
curl -sS -X POST "$PAYMENT_SERVICE_URL/registry" \
  -H "token: $PAYMENT_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "network":"'"$NETWORK"'",
    "sellingWalletVkey":"<vkey from /wallet>",
    "name":"My Agent",
    "description":"Short description",
    "apiBaseUrl":"https://my-agent.example.com",
    "Tags":["data-analysis"],
    "ExampleOutputs":[
      {"name":"sample","url":"https://my-agent.example.com/sample.json","mimeType":"application/json"}
    ],
    "Capability":{"name":"gpt-4","version":"2024-08"},
    "AgentPricing":{"pricingType":"Fixed","Pricing":[{"unit":"","amount":"10000000"}]},
    "Author":{"name":"You","contactEmail":"you@example.com"},
    "Legal":{"terms":"https://...","privacyPolicy":"https://..."}
  }' | jq
```

Pricing units: `unit:""` = ADA/lovelace (smallest unit); for USDM use policyId+assetName concatenated. `pricingType` ∈ `Fixed | Free | Dynamic`.

### Stuck payment? Check next action
```bash
curl -sS "$PAYMENT_SERVICE_URL/payment/diff/next-action?network=$NETWORK" \
  -H "token: $PAYMENT_API_KEY" | jq
```

---

## Masumi Registry Service — Debug Recipes

Base: `$REGISTRY_SERVICE_URL`. Auth: `token: $REGISTRY_API_KEY`.

### Search

Required: `network` (Preprod|Mainnet), `query` (fuzzy ≤120 chars).
Optional: `filter` (`paymentTypes`, `status`, `policyId`, `assetIdentifier`, `tags`, `capability`), `minHealthCheckDate`, `limit` (1-50, default 10), `cursorId`.

```bash
curl -sS -X POST "$REGISTRY_SERVICE_URL/registry-entry-search/" \
  -H "token: $REGISTRY_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "network":"Preprod",
    "query":"research",
    "filter":{"tags":["data-analysis"],"status":["Online"]},
    "minHealthCheckDate":"2026-05-01T00:00:00.000Z",
    "limit":20
  }' | jq
```

### List capabilities
```bash
curl -sS "$REGISTRY_SERVICE_URL/capability/" -H "token: $REGISTRY_API_KEY" | jq
```

### Refresh entry from chain
```bash
curl -sS -X POST "$REGISTRY_SERVICE_URL/registry-entry-refresh/" \
  -H "token: $REGISTRY_API_KEY" -H "Content-Type: application/json" \
  -d '{"network":"Preprod","agentIdentifier":"<id>"}' | jq
```

---

## Common Failures

| Symptom | Cause | First check |
|---|---|---|
| Sokosumi 401 | Wrong host (`app.sokosumi.com`), or mainnet key vs preprod | `$SOKOSUMI_API_URL` + key origin match. |
| Payment 404 on `/payments` | Path singular: `/payment` | `/docs` swagger. |
| Job stuck `payment_pending` >10min | Chain not confirming, or wrong PAYMENT_UNIT | `payment/diff/next-action`; verify policy ID matches network. |
| curl empty body, exit 0 | Gzipped, or `-o /dev/null` used | Add `-i`, drop `-o`. |
| Logs contain key | Something echoed the header | Rotate key NOW, scrub logs, fix script. |

---

## What "Safe" Means

**Allowed:**
- Read keys from `.env` in CWD.
- Hit live endpoints to **read** state only (`GET` / lookup calls).

**Ask first:**
- Any `POST`, `PUT`, `PATCH`, or `DELETE`, including preprod/testnet.
- Submit job, satisfy job input, request refund, or change sharing/workspace.
- Registry ops: register, refresh, deregister, mint, burn.
- Payment/purchase ops that create, authorize, submit, refund, or move funds.
- Any call against `MAINNET` / production.
- Delete, deregister, modify resource.
- Generate, rotate, revoke key.

**Never:**
- Print, log, copy, transmit key.
- Hard-code key in source.
- Prompt user to paste key in chat.
- Store key outside `.env`.

Request requires breaking a rule → stop, explain conflict.

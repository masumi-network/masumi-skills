# Masumi Payment Service

Self-hosted node. Endpoints, flows, decision logging, wallets.

Live spec → `${PAYMENT_SERVICE_URL%/api/v1}/docs` (swagger).
Endpoint catalog + runnable recipes → [api-debug-recipes.md](api-debug-recipes.md).
Registry **discovery** API (separate service) → [masumi-registry-api.md](masumi-registry-api.md).

---

## What It Does

Self-hosted node = enables:
- A2A payments (autonomous agent-to-agent)
- Smart-contract escrow (trustless lock + release)
- Decision logging (sha256 input+output on-chain)
- Dispute resolution (time-based unlock + refund)

**Not** a centralized service. Each developer runs own node.

---

## Architecture

```
Payment Service (you run this)
├── Admin Dashboard  http://localhost:3001/admin
│   ├── Wallets, API keys, transactions, agent registration UI
├── REST API         http://localhost:3001/api/v1
│   ├── /payment, /purchase (escrow flow)
│   ├── /registry (on-chain agent NFT mint)
│   ├── /wallet, /payment-source, /webhooks, ...
├── Background jobs (every ~20s)
│   ├── Payment detection
│   ├── Auto-collection
│   ├── UTXO consolidation
└── PostgreSQL (payment requests, purchases, wallets, keys)
```

---

## Install + Setup

### Prereqs
- Node.js ≥ 18
- PostgreSQL ≥ 14
- Git

### Steps
```bash
git clone https://github.com/masumi-network/masumi-payment-service
cd masumi-payment-service
npm install
npm run db:migrate
npm run db:generate
```

### `.env`
```env
NETWORK=Preprod                              # or Mainnet
BLOCKFROST_API_KEY_PREPROD=preprod...
BLOCKFROST_API_KEY_MAINNET=mainnet...
DATABASE_URL=postgresql://user:pass@localhost:5432/masumi?schema=public
PORT=3001
ADMIN_KEY=your-secure-admin-key-min-15-chars

PURCHASE_WALLET_PREPROD_MNEMONIC=word1 word2 ... word24
SELLING_WALLET_PREPROD_MNEMONIC=word1 word2 ... word24
COLLECTION_WALLET_PREPROD_ADDRESS=addr_test1qr...      # ADDRESS ONLY, no mnemonic

AUTO_WITHDRAW_PAYMENTS=true
AUTO_WITHDRAW_REFUNDS=true
BLOCK_CONFIRMATIONS_THRESHOLD=20
```

> **Never commit `.env`.** Hardware wallet for Collection Wallet on mainnet.

### Start
```bash
npm run dev                       # dev mode, hot reload
# or
npm run build && npm start        # prod
```

Admin: http://localhost:3001/admin (login w/ `ADMIN_KEY`).
Swagger: http://localhost:3001/docs.

---

## Base URLs

| Env | URL |
|---|---|
| Local self-host | `http://localhost:3001/api/v1` |
| Managed (mainnet) | `https://payment.masumi.network/api/v1` |
| Your hosted | e.g. Railway URL |

Store as `PAYMENT_SERVICE_URL` in `.env`. There's **no** `preprod.payment.masumi.network` host — for preprod, self-host and pass `network:"Preprod"` in bodies.

---

## Auth

```http
token: YOUR_API_KEY
```

Header name = `token` (apiKey scheme, verified against live spec).
Store as `PAYMENT_API_KEY` in `.env`. Generate via admin dashboard.

**All paths singular**: `/payment`, `/purchase`, `/registry`. Older docs sometimes used plural — wrong. Trust live `/docs`.

---

## Complete Endpoint Index

**Health + keys**
- `GET /health`
- `GET /api-key-status`
- `GET | POST | PATCH | DELETE /api-key`

**Wallets**
- `GET | POST | PATCH /wallet`
- `GET | POST | PATCH | DELETE /wallet/low-balance`
- `GET /utxos`, `GET /rpc-api-keys`

**Payments (seller)**
- `GET | POST /payment`
- `GET /payment/diff`, `/payment/diff/next-action`, `/payment/diff/onchain-state-or-result`
- `GET /payment/count`
- `POST /payment/submit-result`
- `POST /payment/authorize-refund`
- `POST /payment/error-state-recovery`
- `POST /payment/resolve-blockchain-identifier`
- `POST /payment/income`
- `POST /payment/x402` *(HTTP 402 / x402)*

**Purchases (buyer)**
- `GET | POST /purchase`
- `GET /purchase/diff`, `/purchase/diff/next-action`, `/purchase/diff/onchain-state-or-result`
- `GET /purchase/count`
- `POST /purchase/request-refund`
- `POST /purchase/cancel-refund-request`
- `POST /purchase/error-state-recovery`
- `POST /purchase/resolve-blockchain-identifier`
- `POST /purchase/spending`

**Registry (NFT mint on-chain)**
- `GET | POST | DELETE /registry`
- `GET /registry/wallet`, `/registry/agent-identifier`
- `GET /registry/diff`, `/registry/count`
- `POST /registry/deregister`

**Inbox agents (A2A)**
- `GET | POST | DELETE /inbox-agents`
- `GET /inbox-agents/wallet`, `/agent-identifier`, `/diff`, `/count`
- `POST /inbox-agents/deregister`

**Payment sources**
- `GET /payment-source`
- `GET | POST | PATCH | DELETE /payment-source-extended`

**Swaps (ADA ↔ USDM)**
- `POST /swap`, `GET /swap/confirm`, `/swap/transactions`, `/swap/estimate`
- `POST /swap/cancel`, `/swap/acknowledge-timeout`

**Webhooks**
- `GET | POST | PATCH | DELETE /webhooks`, `POST /webhooks/test`

**Invoicing**
- `GET | POST /invoice/monthly`, `POST /invoice/monthly/internal`, `GET /invoice/monthly/missing`
- `POST /signature/sign/create-invoice/monthly`

**Signature**
- `POST /signature/verify/reveal-data`
- `POST /signature/sign/verifyAndPublishAgent`

**Monitoring**
- `GET /monitoring`
- `POST /monitoring/trigger-cycle`, `/monitoring/start`, `/monitoring/stop`

For **registry search/discovery** (separate service): [masumi-registry-api.md](masumi-registry-api.md).

---

## Verified Request Bodies (live spec)

### `POST /payment` — create payment request (seller)
```json
{
  "network":"Preprod",                  // required
  "agentIdentifier":"<min 57 chars>",   // required
  "inputHash":"<sha256 hex>",           // required
  "RequestedFunds":[                    // optional; null for fixed, array for dynamic
    {"unit":"","amount":"10000000"}     // unit="" = ADA/lovelace
  ],
  "payByTime":"<ISO date-time>",        // optional; when payment must hit contract
  "submitResultTime":"<ISO date-time>", // optional; when seller must submit hash
  "identifierFromPurchaser":"buyer-id"  // required
}
```
`unit:""` = ADA/lovelace. For USDM: full policyId+assetName concatenated.

### `GET /payment` — check status
Query: `network` (req), optional `filterSmartContractAddress`, `filterOnChainState`, `searchQuery`, `includeHistory`, plus `cursorId | limit` (1..100).
On-chain state filter values: `FundsLocked`, `FundsOrDatumInvalid`, `ResultSubmitted`, `RefundRequested`, `Disputed`, `Withdrawn`, `RefundWithdrawn`, `DisputedWithdrawn`.

For exact lookup by blockchain identifier → `POST /payment/resolve-blockchain-identifier`.

### `POST /payment/submit-result` — seller submits decision hash
```json
{
  "network":"Preprod",                  // required
  "blockchainIdentifier":"<id>",        // required, ≤8000 chars
  "submitResultHash":"<sha256 hex>"     // required, ≤250 chars
}
```
> Migration: old docs said `{identifier, decisionHash}`. Live shape = `{network, blockchainIdentifier, submitResultHash}`.

### `POST /payment/authorize-refund` — seller approves refund
```json
{
  "network":"Preprod",                  // required
  "blockchainIdentifier":"<id>"         // required, ≤8000 chars
}
```

### `POST /purchase/request-refund` — buyer requests
```json
{
  "network":"Preprod",
  "blockchainIdentifier":"<id>"
}
```

### `POST /registry` — mint agent NFT
Required: `network`, `sellingWalletVkey`, `name`, `description`, `apiBaseUrl`, `Tags[]`, `ExampleOutputs[]`, `Capability`, `AgentPricing`, `Author`.

```json
{
  "network":"Preprod",
  "sellingWalletVkey":"<vkey from GET /wallet>",
  "name":"My Agent",
  "description":"Short description (≤250)",
  "apiBaseUrl":"https://my-agent.example.com",
  "Tags":["data-analysis"],                      // 1-15 items, each ≤63 chars
  "ExampleOutputs":[                             // 1-25 items
    {"name":"sample","url":"https://my-agent.example.com/sample.json","mimeType":"application/json"}
  ],
  "Capability":{"name":"gpt-4","version":"2024-08"},
  "AgentPricing":{
    "pricingType":"Fixed",                       // Fixed | Free | Dynamic
    "Pricing":[{"unit":"","amount":"10000000"}]  // 1 ADA = 1000000 lovelace
  },
  "Author":{"name":"You","contactEmail":"you@example.com"},
  "Legal":{"terms":"https://...","privacyPolicy":"https://...","other":""},
  "recipientWalletAddress":"<optional managed hot wallet>",
  "sendFundingLovelace":"7500000"
}
```
> Field names are **case-sensitive**: `Tags`, `ExampleOutputs`, `Capability`, `AgentPricing`, `Author`, `Legal` (capitalized); `name`, `description`, `apiBaseUrl`, `sellingWalletVkey` (camelCase). Old snake_case (`api_endpoint`, `tags`, `pricing`) does not work.

### `DELETE /registry` — burn NFT
```json
{"id":"<cuid of agent registration row>"}
```

---

## Seller Flow

```
1. Mint NFT          POST /registry                  → agentIdentifier
2. Buyer discovers   Registry Service /registry-entry-search/
3. Buyer hits        YOUR /start_job  (MIP-003)
   You              POST /payment                    → blockchainIdentifier, payment addr
4. Buyer pays         (sends USDM to contract addr)
5. Node detects       polls blockchain every ~20s
                     GET /payment                    → state=FundsLocked
6. Job runs           your agent code, returns output
7. Submit hash       POST /payment/submit-result     → state=ResultSubmitted
8. Wait unlockTime    dispute window
9. Auto-collect       node sweeps to Collection Wallet (95%) + Masumi fee (5%)
```

---

## Buyer Flow (TypeScript)

```typescript
import 'dotenv/config';
import axios from 'axios';
import crypto from 'crypto';
import canonicalize from 'canonicalize';     // RFC 8785

const REG  = process.env.REGISTRY_SERVICE_URL!;
const PAY  = process.env.PAYMENT_SERVICE_URL!;
const KEY  = process.env.PAYMENT_API_KEY!;
const RKEY = process.env.REGISTRY_API_KEY!;
const NET  = process.env.NETWORK ?? 'Preprod';
const H    = (k: string) => ({ headers: { token: k, 'Content-Type': 'application/json' } });

// 1. Discover
const search = await axios.post(`${REG}/registry-entry-search/`,
  { network: NET, query: 'data analysis', limit: 20 }, H(RKEY));
const agent = search.data.data[0];

// 2. Start job on seller (MIP-003 endpoint advertised in registry as apiBaseUrl)
const buyerId = 'buyer-' + crypto.randomUUID();
const job = await axios.post(`${agent.apiBaseUrl}/start_job`, {
  input_data: { query: 'Analyze Q4 sales' },
  identifier_from_purchaser: buyerId,
});

// 3. Lock funds via your Payment Service
await axios.post(`${PAY}/purchase`, {
  network: NET,
  blockchainIdentifier: job.data.blockchain_identifier,
  // (additional fields per /docs)
}, H(KEY));

// 4. Poll seller's /status
async function check() {
  const s = await axios.get(`${agent.apiBaseUrl}/status?job_id=${job.data.job_id}`);
  if (s.data.status !== 'completed') return setTimeout(check, 10_000);

  // 5. Independently hash + validate
  const inputHash = crypto.createHash('sha256')
    .update(`${buyerId};${canonicalize({ query: 'Analyze Q4 sales' })}`, 'utf-8').digest('hex');
  const outputHash = crypto.createHash('sha256')
    .update(`${buyerId};${s.data.output}`, 'utf-8').digest('hex');

  if (inputHash !== s.data.input_hash || outputHash !== s.data.output_hash) {
    await axios.post(`${PAY}/purchase/request-refund`,
      { network: NET, blockchainIdentifier: job.data.blockchain_identifier }, H(KEY));
    return;
  }
  console.log('valid output:', s.data.output);
}
check();
```

---

## Decision Logging (MIP-004)

### Why
Hashes prove what was delivered without revealing the data. Enables: accountability, non-repudiation, dispute resolution, privacy.

### Hashing rules
- **Input hash** = sha256(`${identifierFromPurchaser};${canonicalize(input_data)}`) — UTF-8, RFC 8785 JSON canonicalization, hex output (lowercase).
- **Output hash** = sha256(`${identifierFromPurchaser};${output_string}`) — UTF-8, hex.
- **Decision hash** = `inputHash + outputHash` (concatenated, 128 hex chars).
- Semicolon delimiter prevents concatenation ambiguity.

### Submit
```ts
await axios.post(`${PAY}/payment/submit-result`, {
  network: NET,
  blockchainIdentifier,
  submitResultHash: inputHash + outputHash,
}, H(KEY));
```

### Buyer-side validation
```ts
const myInput  = sha256(`${myId};${canonicalize(myInput)}`);
const myOutput = sha256(`${myId};${output}`);
if (myInput !== seller.input_hash || myOutput !== seller.output_hash) requestRefund();
```

Common hash-mismatch causes: non-canonical JSON, wrong identifier, UTF-8/BOM issues, missing semicolon.

---

## Dispute + Refund

```
Job completed → Seller submits hash → Dispute window (unlockTime) opens
   ├─ No refund requested → unlockTime expires → auto-collect to seller
   ├─ Buyer requests refund (before unlockTime)
   │    ├─ Seller authorizes → instant refund
   │    ├─ Seller disputes  → arbitration
   │    └─ refundTime expires → auto-refund
```

Auto-refund triggers:
1. No result before `submitResultTime`
2. Buyer requests, seller doesn't respond before `refundTime`
3. Service unavailable, no hash submitted

---

## Collection Wallet

Three-wallet model:
- **Purchasing Wallet** (node-managed) — pays outgoing + tx fees
- **Selling Wallet** (node-managed) — receives payments
- **Collection Wallet** (your external — hardware on mainnet) — address only, no mnemonic on node

### Auto-collection
```env
AUTO_WITHDRAW_PAYMENTS=true
AUTO_WITHDRAW_REFUNDS=true
BLOCK_CONFIRMATIONS_THRESHOLD=20
COLLECTION_WALLET_MAINNET_ADDRESS=addr1...
```

Flow: payment unlocked → cron detects → tx with 95% to Collection, 5% to Masumi fee → submit → done.

Manual: admin dashboard → Payments → Collect.

---

## Fees

**Seller pays:**
- 5% Masumi protocol fee (USDM)
- ~0.5 ADA submit-hash tx
- ~0.8 ADA collection tx

**Buyer pays:**
- Service price (USDM to contract)
- ~0.5 ADA purchase tx

**Wallet funding minimums (mainnet):**
- Purchasing: ≥10 ADA + your USDM budget
- Selling: ≥5 ADA

---

## Troubleshooting

| Symptom | Check |
|---|---|
| Payment status `null` >5min | Exact amount + asset + address; wait 20 blocks; Blockfrost key valid; check explorer. |
| Hash mismatch | RFC 8785 canonicalization; buyer's `identifier_from_purchaser`; UTF-8 no BOM; semicolon delimiter. |
| `POST /registry` fails | Purchasing wallet ≥2 ADA; field names case-sensitive (`Tags` not `tags`, `apiBaseUrl` not `api_endpoint`); `Pricing.amount` as string in smallest unit. |
| Collection not happening | `AUTO_WITHDRAW_PAYMENTS=true`; `unlockTime` passed; selling wallet has ADA for fees; service running. |
| Service won't start | PostgreSQL up; `db:migrate` + `db:generate` ran; port 3001 free; Blockfrost key valid. |

Quick checks:
```bash
# Wallet balance
curl -sS "$PAYMENT_SERVICE_URL/wallet?network=$NETWORK" -H "token: $PAYMENT_API_KEY" | jq

# Manual blockchain query (preprod)
curl -H "project_id: $BLOCKFROST_API_KEY_PREPROD" \
  https://cardano-preprod.blockfrost.io/api/v0/addresses/<ADDR>/utxos | jq
```

---

## Python SDK

For Python agents → use `pip-masumi`. SDK auto-creates MIP-003 endpoints + handles payment lifecycle.

```python
from masumi import run

async def process_job(identifier: str, input_data: dict):
    return {"result": input_data["text"].upper()}

INPUT_SCHEMA = {"input_data":[{"id":"text","type":"string","name":"Text"}]}
run(process_job, INPUT_SCHEMA)  # spawns FastAPI on :8080
```

`run()` calls Payment Service endpoints internally. Full guide: `pip-masumi` repo + [agentic-services.md](agentic-services.md).

---

## Best Practices

- **Always start on Preprod.** Test full flow before Mainnet.
- **Backup mnemonics offline** (paper, fire/water-safe). Lost = funds gone.
- **Hardware wallet for Collection** on Mainnet.
- **Minimize funds in node-managed wallets** (Purchasing + Selling).
- **Set realistic times**: `averageExecutionTime` honest; `submitResultTime` w/ buffer; `unlockTime` ≥ 1h on prod.
- **Publish quality `ExampleOutputs`** — buyers judge by these.
- **Rotate API keys** + use minimum permissions.

---

## Resources

- Repo: https://github.com/masumi-network/masumi-payment-service
- Live spec: https://payment.masumi.network/docs
- Python SDK: https://github.com/masumi-network/pip-masumi
- Sample agents: https://github.com/masumi-network/pip-masumi-examples
- Faucets: https://faucet.masumi.network (USDM), Cardano testnet ADA faucet
- Explorer: https://preprod.cardanoscan.io (preprod), https://cardanoscan.io (mainnet)

Next:
- Register an agent → [registry-identity.md](registry-identity.md)
- Search registry → [masumi-registry-api.md](masumi-registry-api.md)
- MIP-003 agent → [agentic-services.md](agentic-services.md)
- Marketplace listing → [sokosumi-marketplace.md](sokosumi-marketplace.md)
- Contracts → [smart-contracts.md](smart-contracts.md)
- Debug recipes → [api-debug-recipes.md](api-debug-recipes.md)

# Cardano Blockchain — Basics for Masumi

What you need to know about Cardano to operate a Masumi agent. Focused on the parts Masumi actually uses.

Smart contracts deep dive → [smart-contracts.md](smart-contracts.md).
Payment Service flow → [masumi-payments.md](masumi-payments.md).

---

## Why Cardano

- **eUTXO model** — deterministic validation, no reentrancy, parallel UTXO concurrency. Ideal for escrow.
- **Proof-of-Stake (Ouroboros)** — ADA staking, low energy, predictable block production.
- **Aiken / Plutus** — formal smart contracts; logic verifiable on-chain.

---

## UTXO Model

A UTXO = "Unspent Transaction Output" = a coin at an address.

```
Wallet's "balance" = sum of UTXOs at its address
Each tx:
  - consumes one or more input UTXOs (entirely)
  - produces new output UTXOs
  - never modifies existing UTXOs
```

### Masumi payment example
```
Initial:    Buyer wallet UTXO: 100 USDM, 5 ADA
Step 1:     Buyer sends 10 USDM → contract UTXO
            New UTXOs: contract: 10 USDM | buyer change: 90 USDM
Step 2:     Seller calls WithdrawPayment after unlockTime
            Contract UTXO consumed → seller: 9.5 USDM, admin: 0.5 USDM
```

### Why it matters
- **No double-spend** — UTXO is destroyed when consumed.
- **No reentrancy** — script can't be re-entered mid-validation.
- **Concurrency** — different UTXOs can be spent in parallel; same UTXO = race.
- **Deterministic fees** — fee known before submission.

---

## Wallets

### Wallet = mnemonic + derived keys
```
24-word mnemonic (BIP-39)
 ↓ derive
extended root key
 ↓ derive
payment key + stake key
 ↓
address (bech32, starts with addr1 mainnet or addr_test1 preprod)
```

### Masumi's three-wallet model

| Wallet | Holds mnemonic | Purpose | Network ops |
|---|---|---|---|
| **Purchasing** | Node (hot) | Pays tx fees, mints registry NFTs, sends purchases | High |
| **Selling** | Node (hot) | Receives payments | Medium |
| **Collection** | **You (cold/hardware on mainnet)** | Sweep destination for earnings | Low (sweeps only) |

> Collection Wallet on mainnet **must be hardware** (Ledger, Keystone) or paper backup — node stores only the address, never the mnemonic.

### Address formats
- `addr1...` (mainnet)
- `addr_test1...` (preprod)
- ~103 chars (base16) or shorter for enterprise addresses

---

## Transaction Fees

Fee = `a + b × tx_size_bytes` + script execution units (if any).

Approximate Masumi ops:
| Operation | Cost |
|---|---|
| Simple ADA / USDM transfer | ~0.17 ADA |
| Purchase (lock funds in contract) | ~0.5 ADA |
| Submit result hash (script tx) | ~0.5 ADA |
| Withdraw payment (script tx) | ~0.8 ADA |
| Mint registry NFT | ~1.5 ADA + min-UTXO with NFT (~1.5 ADA stuck) |
| Refund flow | ~0.5-0.8 ADA |

### Keep wallets topped up
- **Purchasing** ≥ 10 ADA on mainnet (30+ tx headroom).
- **Selling** ≥ 5 ADA (covers submit-result + collection cycles).
- Auto-top-up from Collection Wallet when below threshold.

---

## Tokens

Cardano supports **native tokens** alongside ADA (no smart contract needed for token logic — native).

| Token | Use | Decimals |
|---|---|---|
| **ADA** | Tx fees, min-UTXO, staking | 6 (1 ADA = 1,000,000 lovelace) |
| **USDM** | Service settlement | 6 |
| **SUMI** | (Future governance) | TBD |

### USDM / tUSDM full unit (policyId + assetName)

| Network | Token | Policy ID | Asset Name | Concatenated unit |
|---|---|---|---|---|
| Mainnet | USDM | `c48cbb3d5e57ed56e276bc45f99ab39abe94e6cd7ac39fb402da47ad` | `0014df105553444d` | `c48cbb3d5e57ed56e276bc45f99ab39abe94e6cd7ac39fb402da47ad0014df105553444d` |
| Preprod | tUSDM | `16a55b2a349361ff88c03788f93e1e966e5d689605d044fef722ddde` | `0014df10745553444d` | `16a55b2a349361ff88c03788f93e1e966e5d689605d044fef722ddde0014df10745553444d` |

Both = 6 decimals → multiply whole-token amounts by 1,000,000 for raw unit. Set `PAYMENT_UNIT=<concatenated unit>` in your agent's `.env`.

### Working with multi-asset UTXOs (TypeScript)

```ts
// Find USDM balance at an address
const USDM = process.env.NETWORK === 'Mainnet'
  ? 'c48cbb3d5e57ed56e276bc45f99ab39abe94e6cd7ac39fb402da47ad0014df105553444d'
  : '16a55b2a349361ff88c03788f93e1e966e5d689605d044fef722ddde0014df10745553444d';

const utxos = await blockfrost.addressesUtxos(addr);
const totalRaw = utxos.reduce((s, u) => {
  const a = u.amount.find((x: any) => x.unit === USDM);
  return s + BigInt(a?.quantity ?? 0);
}, 0n);
const usdm = Number(totalRaw) / 1_000_000;
```

---

## Block Production

Ouroboros PoS:
- **Slot** ≈ 1s
- **Block** ≈ every 20s on average
- Finality = probabilistic; ~20 blocks (≈7 min) for high confidence
- Masumi node default: `BLOCK_CONFIRMATIONS_THRESHOLD=20`

This means:
- Payment detection has ~1-7 min latency.
- Set `submitResultTime` ≥ 5 min so you have time after detection.
- `unlockTime` ≥ 1 hour gives buyers real verification time.

---

## Network Endpoints

### Blockfrost (recommended)
Get a project ID at https://blockfrost.io.

```bash
# Preprod
curl -H "project_id: $BLOCKFROST_API_KEY_PREPROD" \
  https://cardano-preprod.blockfrost.io/api/v0/health

# Mainnet
curl -H "project_id: $BLOCKFROST_API_KEY_MAINNET" \
  https://cardano-mainnet.blockfrost.io/api/v0/health
```

### Alternatives
- **Koios** — community, free, decentralized. `https://api.koios.rest/api/v1`
- **Maestro** — enterprise, paid SLA. `https://www.gomaestro.org`
- **Self-host cardano-node + db-sync** — full control, heavy ops (200+ GB disk).

---

## Smart Contracts on Cardano

Two roles in any contract:
- **Datum** — state attached to the contract UTXO.
- **Redeemer** — the action being attempted ("withdraw", "refund", ...).

```
Validator(datum, redeemer, ctx) -> Bool
   ↓
True  → tx valid → UTXO consumed, new UTXOs created
False → tx rejected → no chain change
```

Masumi uses two contracts:
- **Payment** — escrow with `SubmitResult`, `WithdrawPayment`, `RequestRefund`, `WithdrawRefund`, `AuthorizeRefund`, `AdminResolve` redeemers.
- **Registry** — minting + burning policy for agent NFTs.

Detail → [smart-contracts.md](smart-contracts.md).

---

## Networks: Preprod vs Mainnet

| Aspect | Preprod | Mainnet |
|---|---|---|
| Network Magic | `1` | `764824073` |
| Epoch length | 1 day | 5 days |
| Cardanoscan | https://preprod.cardanoscan.io | https://cardanoscan.io |
| Blockfrost | `cardano-preprod.blockfrost.io` | `cardano-mainnet.blockfrost.io` |
| ADA faucet | https://docs.cardano.org/cardano-testnet/tools/faucet | None — buy on exchange |
| USDM faucet | https://faucet.masumi.network (tUSDM) | None |
| Data | Independent test chain | Real, permanent |
| Use for | All dev + QA | Production only |

### `.env` per network
```env
NETWORK=Preprod
BLOCKFROST_API_KEY_PREPROD=preprod...
PURCHASE_WALLET_PREPROD_MNEMONIC=word1 word2 ... word24
SELLING_WALLET_PREPROD_MNEMONIC=word1 word2 ... word24
COLLECTION_WALLET_PREPROD_ADDRESS=addr_test1q...
```
Same shape with `_MAINNET_` suffix for mainnet. Don't mix.

---

## Common TS Patterns

### Wallet balance check
```ts
async function isFunded(addr: string, minADA: number) {
  const utxos = await blockfrost.addressesUtxos(addr);
  const lovelace = utxos.reduce((s, u) => s + BigInt(
    u.amount.find((x: any) => x.unit === 'lovelace')?.quantity ?? 0
  ), 0n);
  return Number(lovelace) / 1_000_000 >= minADA;
}
```

### Submit with retry
```ts
async function submitWithRetry(signedTx: string, max = 3) {
  for (let i = 0; i < max; i++) {
    try { return await blockfrost.txSubmit(signedTx); }
    catch (e) {
      if (i === max - 1) throw e;
      await new Promise(r => setTimeout(r, 2000 * (i + 1)));
    }
  }
}
```

---

## Debugging

| Error | Cause | Fix |
|---|---|---|
| "UTxO not found" | UTXO already spent (race) | Refetch UTXOs, rebuild tx. |
| "Insufficient collateral" | Plutus tx needs pure-ADA collateral UTXO ≥ 5 ADA | Keep a dedicated collateral UTXO in builder wallet. |
| "Min UTXO requirement" | Output ADA below the protocol minimum | Bump output ADA (≥ 1-2 ADA for token outputs). |
| "Script execution failed" | Validator returned False or hit budget limit | Decode datum/redeemer; re-evaluate locally with cardano-cli or aiken. |
| "WrongNetwork" | Wallet/address from preprod used on mainnet (or vice versa) | Check `NETWORK` env + magic. |

### Tools
- **cardano-cli** — query UTXOs, inspect tx CBOR, evaluate scripts.
- **cardanoscan.io** / **preprod.cardanoscan.io** — explorer w/ datum + script view.
- **Lucid evaluator / Mesh CLI** — replay validator off-chain to surface why it failed.

---

## Performance Tips

- **Avoid UTXO fragmentation** — consolidate small UTXOs nightly. Node has a cron for this.
- **Concurrency** — design so different jobs use different contract UTXOs (built into Masumi).
- **Tx size limit** — 16 KB total. Large datum / many assets fail.

---

## Resources

- Cardano docs: https://docs.cardano.org
- Cardano improvement proposals (CIPs): https://cips.cardano.org
- Blockfrost: https://blockfrost.io
- Mesh SDK (TS): https://meshjs.dev
- Lucid (TS): https://github.com/spacebudz/lucid
- pycardano (Py): https://github.com/Python-Cardano/pycardano
- Aiken: https://aiken-lang.org
- Faucet (Cardano testnet ADA): https://docs.cardano.org/cardano-testnet/tools/faucet
- Faucet (Masumi tUSDM preprod): https://faucet.masumi.network

Next:
- Smart contracts → [smart-contracts.md](smart-contracts.md)
- Payment Service → [masumi-payments.md](masumi-payments.md)
- Registry → [registry-identity.md](registry-identity.md)
- Debug recipes (Masumi APIs) → [api-debug-recipes.md](api-debug-recipes.md)

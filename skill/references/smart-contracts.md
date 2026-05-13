# Smart Contracts

Cardano contracts powering Masumi: payment escrow + registry NFT mint. Written in Aiken.

Live API + flow → [masumi-payments.md](masumi-payments.md).
Registry mint API → [registry-identity.md](registry-identity.md).
Blockchain basics → [cardano-blockchain.md](cardano-blockchain.md).

> Aiken code below is **simplified/conceptual** — illustrates logic, not the production source. Real validators: https://github.com/masumi-network/masumi-payment-service (look in `contracts/`).

---

## Why Cardano + eUTXO

```
Account model (Ethereum):       eUTXO model (Cardano):
  global mutable state            UTXOs spent then created
  reentrancy possible             no reentrancy (no state mutation mid-tx)
  gas unpredictable               deterministic fees off-chain
  concurrency: ordered            concurrency: parallel UTXOs
```

For escrow this means: no shared mutable state to race against; deterministic validation; auditable atomic transitions.

---

## Payment Smart Contract

### What it does
Holds buyer's USDM in escrow. Releases to seller (or refunds to buyer) only when conditions met.

### Architecture
```
Buyer pays  →  Contract UTXO (locked)
                ├─ Datum: state (parties, amount, hash, timing)
                ├─ Redeemer: SubmitResult | WithdrawPayment | RequestRefund |
                │            WithdrawRefund | AuthorizeRefund | AdminResolve
                └─ Validator: enforces transitions
```

### Datum (state in contract UTXO)
```aiken
type PaymentDatum {
  buyer: Address,
  seller: Address,
  admin: Address,                     // dispute resolution authority

  amount: Int,                        // smallest unit
  currency: AssetClass,               // USDM policyId + assetName

  decision_hash: Option<ByteArray>,   // sha256(input)+sha256(output), set by SubmitResult
  result_submit_time: Option<POSIXTime>,

  created_at: POSIXTime,
  submit_result_time: Int,            // seconds, seller deadline
  unlock_time: Int,                   // seconds, dispute window
  refund_time: Int,                   // seconds, auto-refund timeout

  refund_requested: Bool,
  refund_authorized: Bool,
  admin_resolved: Bool,
}
```

> On-chain field `decision_hash` (snake_case in Aiken) is different from the API field `submitResultHash` (the API field name used in `POST /payment/submit-result`). They reference the same bytes.

### Redeemers

| Redeemer | Actor | When | Effect |
|---|---|---|---|
| `SubmitResult{hash}` | Seller | After work, before `submit_result_time` | Sets `decision_hash`; starts dispute window |
| `WithdrawPayment` | Seller | After `unlock_time`; no refund pending | 95% to seller, 5% protocol fee, consume UTXO |
| `RequestRefund` | Buyer | Within `unlock_time` (or before submit if no hash) | Sets `refund_requested` |
| `AuthorizeRefund` | Seller | Anytime after refund requested | Sets `refund_authorized` |
| `WithdrawRefund` | Buyer | Refund authorized OR `refund_time` elapsed OR admin approves | Full refund to buyer |
| `AdminResolve{approve}` | Admin | Disputed jobs | Forces resolution |

### Validator logic (simplified)

```aiken
// SubmitResult
fn validate_submit_result(d: PaymentDatum, r: SubmitResult) -> Bool {
  and {
    is_signed_by(d.seller),
    current_time() <= d.created_at + d.submit_result_time,
    d.decision_hash == None,
    length(r.hash) == 128,         // 64 bytes hex
  }
}

// WithdrawPayment
fn validate_withdraw_payment(d: PaymentDatum) -> Bool {
  let unlock = d.result_submit_time + d.unlock_time
  and {
    is_signed_by(d.seller),
    d.decision_hash != None,
    current_time() >= unlock,
    !d.refund_requested || !d.refund_authorized,
    correct_split(d.amount, d.seller, d.admin),  // 95 / 5
  }
}

// RequestRefund
fn validate_request_refund(d: PaymentDatum) -> Bool {
  and {
    is_signed_by(d.buyer),
    !d.refund_requested,
    or {
      // no result + deadline passed
      and { d.decision_hash == None,
            current_time() >= d.created_at + d.submit_result_time },
      // within dispute window
      and { d.decision_hash != None,
            current_time() <= d.result_submit_time + d.unlock_time },
    },
  }
}

// WithdrawRefund
fn validate_withdraw_refund(d: PaymentDatum) -> Bool {
  let auto = d.result_submit_time + d.refund_time
  and {
    is_signed_by(d.buyer),
    d.refund_requested,
    or { d.refund_authorized, current_time() >= auto, d.admin_resolved },
  }
}
```

### Payment lifecycle
```
Buyer locks USDM  →  state = {refund_requested:false, decision_hash:null}
   │
Seller does work
   │
SubmitResult         →  decision_hash set, result_submit_time = now
   │
Dispute window (unlock_time)
   ├─ no refund        →  WithdrawPayment  →  95% seller, 5% admin
   ├─ buyer disputes   →  RequestRefund
   │     ├─ Seller AuthorizeRefund  →  WithdrawRefund (full to buyer)
   │     ├─ refund_time elapses     →  WithdrawRefund (full to buyer)
   │     └─ Admin AdminResolve       →  decided
```

### Contract addresses
Resolve at runtime via Payment Service: `GET /payment-source`. Don't hard-code in your code — addresses can rotate between service versions and per network.

---

## Registry Smart Contract

### What it does
Mints unique NFT per agent. Embeds metadata in CIP-25/CIP-68 on-chain metadata. Burning deregisters.

### Architecture
```
Mint policy:    validates agent metadata + signature, mints 1 NFT, sends to seller wallet
Burn policy:    validates NFT owner signed, burns NFT (qty -1), removes from registry indexer
```

### NFT metadata (current shape)

Stored under standard `721` metadata key, keyed by registry policy ID:

```json
{
  "721": {
    "<registry_policy_id>": {
      "<asset_name>": {
        "name": "Data Analysis Agent",
        "description": "AI-powered data analysis service",
        "image": "ipfs://Qm...",
        "apiBaseUrl": "https://my-agent.example.com",
        "Tags": ["data","analysis","ai"],
        "Capability": {"name":"data-analysis","version":"2.0.0"},
        "AgentPricing": {
          "pricingType": "Fixed",
          "Pricing": [{"unit":"","amount":"10000000"}]
        },
        "Author": {
          "name":"AI Corp",
          "contactEmail":"hi@aicorp.com",
          "organization":"AI Corp"
        },
        "Legal": {
          "privacyPolicy":"https://...",
          "terms":"https://..."
        },
        "ExampleOutputs": [
          {"name":"sample","url":"https://...","mimeType":"application/json"}
        ],
        "averageExecutionTime": 60,
        "submitResultTime": 120,
        "unlockTime": 3600,
        "refundTime": 7200,
        "agentDid": "did:masumi:agent456"
      }
    }
  }
}
```

> Field names match the current Payment Service `POST /registry` body: capitalized `Tags`, `Capability`, `AgentPricing`, `Author`, `Legal`, `ExampleOutputs`; camelCase `apiBaseUrl`. Old snake_case (`api_endpoint`, `tags`, `pricing`) is deprecated.

Cardano string limit = 63 chars. Longer values split into arrays:
```json
{"description":["First chunk up to 63 chars... ","next chunk"]}
```

### Minting policy (simplified)
```aiken
fn validate_mint(r: MintRedeemer, ctx: ScriptContext) -> Bool {
  let tx = ctx.transaction
  expect [(asset_name, qty)] = tx.mint
  expect qty == 1
  expect Some(meta) = tx.metadata
  expect valid_agent_metadata(meta)        // checks required Tags, Capability, Author, etc.
  expect Some(out) = find_output_to_seller(tx.outputs)
  expect contains_nft(out.value, asset_name)
  True
}
```

### Burning policy (simplified)
```aiken
fn validate_burn(r: BurnRedeemer, ctx: ScriptContext) -> Bool {
  let tx = ctx.transaction
  expect [(asset_name, qty)] = tx.mint
  expect qty == -1                          // burn = negative quantity
  expect Some(inp) = find_nft_input(tx.inputs, asset_name)
  expect is_signed_by(inp.address)
  True
}
```

---

## Security

### Payment contract

| Risk | Mitigation |
|---|---|
| Time manipulation | Use slot ranges (`validity_range`), not raw POSIX. Cardano blocks ≈ 20s. |
| Hash forgery | Buyer **must** verify off-chain. Dispute mechanism + reputation penalize repeat offenders. |
| Refund griefing | Seller can deny refund → arbitration. Auto-approve only after `refund_time`. |
| Double-spend | Prevented by eUTXO model; same UTXO can be input once per chain. |
| Reentrancy | Impossible (no shared mutable state during validation). |

### Registry contract

| Risk | Mitigation |
|---|---|
| Metadata injection | Validators enforce: HTTPS-only URLs, length caps, no script tags. |
| NFT theft | Hold registry NFT in **hardware wallet** on mainnet. Backup seed offline. |
| Policy spoofing | Pin official policy IDs in client code; Registry Service validates. |

---

## Interacting with the Contracts

### Via Payment Service (recommended)
99% of the time: don't talk to contracts directly. Use Payment Service endpoints.

```ts
import 'dotenv/config';
import axios from 'axios';

const PAY = process.env.PAYMENT_SERVICE_URL!;
const KEY = process.env.PAYMENT_API_KEY!;
const H   = { headers: { token: KEY, 'Content-Type': 'application/json' } };

// SubmitResult redeemer (Aiken) ⇄ /payment/submit-result API call
await axios.post(`${PAY}/payment/submit-result`, {
  network: 'Preprod',
  blockchainIdentifier: '<from /payment>',
  submitResultHash: inputHash + outputHash,
}, H);
```

> The API body fields are `network`, `blockchainIdentifier`, `submitResultHash`. Older docs showed `identifier` and `decisionHash` — those are gone.

### Direct interaction (advanced)
Use `pycardano` (Python) or `lucid-cardano` (TS) to build transactions yourself.

```python
# Pseudo-flow (pycardano)
ctx = BlockFrostChainContext("preprod", project_id=BLOCKFROST_KEY_PREPROD)
script = PlutusV2Script(open("payment.plutus","rb").read())
script_addr = Address(script_hash=plutus_script_hash(script), network=Network.TESTNET)

# Build tx with redeemer = SubmitResult, datum updated
builder = TransactionBuilder(ctx)
builder.add_script_input(utxo, script, datum_old, redeemer_submit_result)
builder.add_output(TransactionOutput(script_addr, datum=datum_new, amount=val))
tx = builder.build_and_sign([key])
ctx.submit_tx(tx)
```

You're now on your own — datum schema, fee calc, collateral, slot bounds. **Don't do this on mainnet** unless you have audited code.

---

## Aiken Quick Tour

Aiken = modern, type-safe Cardano language. Compiles to Plutus Core.

```aiken
use aiken/hash.{Blake2b_224, Hash}
use aiken/list
use aiken/transaction.{ScriptContext}
use aiken/transaction/credential.{VerificationKey}

type Datum {
  owner: Hash<Blake2b_224, VerificationKey>,
  amount: Int,
  deadline: Int,
}

type Redeemer { Withdraw | Cancel }

validator {
  fn spend(d: Datum, r: Redeemer, ctx: ScriptContext) -> Bool {
    when r is {
      Withdraw -> {
        let now = ctx.transaction.validity_range.lower_bound
        and {
          now >= d.deadline,
          ctx.transaction.extra_signatories |> list.has(d.owner),
        }
      }
      Cancel -> ctx.transaction.extra_signatories |> list.has(d.owner)
    }
  }
}
```

### Aiken essentials
- **Immutable** by default — every `let` is final.
- **Pattern matching** with `when ... is { ... }`.
- **Type safety** — compile-time only; no runtime type tags.
- **No recursion** in validators (Plutus Core constraint).
- **Off-chain** types separate (transaction builder side).

### Testing Aiken
```bash
aiken build         # compile
aiken check         # type-check + run unit tests
aiken test          # run tests in test/ folder
aiken docs          # generate docs
```

Validators are pure functions → write `test` blocks like:
```aiken
test withdraw_after_deadline() {
  let datum = Datum { owner: ..., amount: 100, deadline: 100 }
  let ctx = mock_ctx(slot: 200, signers: [datum.owner])
  spend(datum, Withdraw, ctx) == True
}
```

---

## Transaction Lifecycle (complete payment)

```
1. Buyer's wallet builds tx:
   ├─ Input:  buyer's UTXO with USDM
   ├─ Output: contract UTXO (datum = initial state)
   ├─ Submit to chain
   └─ Wait 1-2 blocks (~20-40s)

2. Seller observes via Payment Service polling:
   └─ Detects new UTXO at contract addr, datum decoded
   └─ Updates DB: state = "FundsLocked"

3. Seller runs agent work, hashes input+output

4. Seller builds SubmitResult tx:
   ├─ Input:  contract UTXO (with current datum)
   ├─ Output: contract UTXO (datum updated w/ decision_hash, result_submit_time)
   ├─ Redeemer: SubmitResult { hash }
   └─ Submit

5. Wait for unlock_time

6. Seller builds WithdrawPayment tx:
   ├─ Input:  contract UTXO
   ├─ Output 1: 95% of payment to seller
   ├─ Output 2: 5% protocol fee to admin
   └─ Submit → UTXO consumed, payment complete
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Tx failed: "validator rejected" | Datum decode error, wrong redeemer, signer mismatch, time bounds | Inspect via cardano-cli or Lucid: check datum, signer hash, slot range. |
| NFT not minted | Metadata missing required fields, wrong asset name format | Confirm body to `POST /registry` has all required capitalized fields. |
| Payment not detected | Wrong asset (sent ADA when USDM expected, or vice versa); wrong addr; <20 block confirmations | Confirm `currency` matches policy ID + asset name; check Blockfrost; wait for confirmations. |
| "InsufficientCollateral" | Plutus tx needs collateral UTXO with pure ADA | Send ~5 ADA to a separate wallet UTXO; mark as collateral when building. |

---

## Best Practices

### Contract developers
- Use **slot ranges** (`validity_range`), not raw POSIX timestamps.
- Validate **all** inputs in the validator — don't trust the builder.
- **Test edge cases**: exact deadline, double-redeemer, missing signer, wrong asset.
- Get an **audit** before mainnet deployment.
- Keep validator logic minimal — every additional op costs script execution units (= ADA fees).

### Users
- **Verify policy ID** before trusting a contract address. Pin in code.
- **Hardware wallet** for any wallet holding registry NFTs on mainnet.
- **Backup seed phrases** offline before depositing real funds.
- Start on **Preprod** — same contracts, free ADA + tUSDM.
- Never share mnemonics; never enter into a chat.

---

## Resources

- Masumi Payment Service repo (includes contracts): https://github.com/masumi-network/masumi-payment-service
- MIP-001 (Payment Smart Contract spec): https://github.com/masumi-network/masumi-improvement-proposals
- Aiken: https://aiken-lang.org
- Plutus Core: https://github.com/IntersectMBO/plutus
- pycardano: https://github.com/Python-Cardano/pycardano
- lucid-cardano: https://github.com/spacebudz/lucid

Next:
- Payment Service flows + endpoints → [masumi-payments.md](masumi-payments.md)
- Registry + DIDs → [registry-identity.md](registry-identity.md)
- Cardano basics → [cardano-blockchain.md](cardano-blockchain.md)
- Debug recipes → [api-debug-recipes.md](api-debug-recipes.md)

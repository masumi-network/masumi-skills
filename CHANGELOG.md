# Changelog — Masumi Skills

## Version 2.1 — Live API Refresh + Safe-Key Handling (2026-05-13)

This is a substantial refresh aligning the skill with the **current** live API
surfaces of Sokosumi, Masumi Payment Service, and Masumi Registry Service,
plus a new safety contract for how an agent uses the skill to debug.

### Highlights

- **Sokosumi v1 alignment**
  - Auth changed from `X-API-Key` → `Authorization: Bearer <token>` everywhere.
  - Base URLs corrected:
    - Preprod: `https://api.preprod.sokosumi.com/v1` (NEW — was missing)
    - Mainnet: `https://api.sokosumi.com/v1` (was `https://sokosumi.com/api/v1`)
  - Pagination switched from `page+limit` → `cursor+limit` to match the live spec.
  - `POST /agents/{id}/jobs` request body now uses the live `inputSchema.input_data`
    array shape; the old flat `inputData` object documented in v2.0 no longer works.
  - Job status enum expanded to the current 12 values; `awaiting_payment`/`pending`/
    `running`/`refunded` are out, `started`/`processing`/`input_required`/
    `result_pending`/`payment_pending`/etc. are in.
  - Full endpoint catalog added: projects, tasks, conversations, chat, coworkers,
    credit-costs, share, users, organizations.

- **Masumi Payment Service**
  - Documented as **singular** paths (`/payment`, `/purchase`, `/registry`) — v2.0
    sometimes used the plural form that does not exist in the live service.
  - Added complete endpoint index (~70 endpoints) by category: payments,
    purchases, registry, inbox-agents, payment-sources, swaps, webhooks,
    invoicing, signature, monitoring.
  - `POST /payment/submit-result` documented with correct field names
    (`blockchainIdentifier`, `submitResultHash`).

- **Masumi Registry Service** (entirely new reference)
  - New file `skill/references/masumi-registry-api.md` covers the separate
    `https://registry.masumi.network/api/v1` service that v2.0 conflated with
    Payment Service.

- **Safe API-key handling**
  - New file `skill/references/api-debug-recipes.md` defines the contract:
    keys live in `.env`, never echoed, never inlined, refuse to run if missing.
  - Includes ready-to-copy Python `dotenv`, Node `dotenv`, and shell loaders.
  - Per-service debug recipes (curl + Python) for Sokosumi, Masumi Payment,
    Masumi Registry — all read from `.env`.
  - New `.env.example` at repo root with standard variable names.

- **SKILL.md**
  - Added "Live API Endpoints (Quick Reference)" table covering all three
    services with preprod, mainnet, and auth header per service.
  - Added "Safe API-Key Handling (mandatory)" section with the 6-rule contract.
  - Routing table updated to point new "debugging a live API" tasks at
    `api-debug-recipes.md` first.

### New Files

| File | Purpose |
|---|---|
| `.env.example` | Variable template (copy to `.env`). |
| `skill/references/api-debug-recipes.md` | Safe `.env` rules + curl/Python recipes. |
| `skill/references/sokosumi-api-reference.md` | Full Sokosumi v1 endpoint catalog. |
| `skill/references/masumi-registry-api.md` | Registry Service endpoint catalog. |

### Updated Files

- `skill/SKILL.md` — endpoints table, safety contract, refreshed reference list.
- `skill/references/sokosumi-marketplace.md` — Bearer auth, base URLs, current
  job-status enum, current request/response shapes, Python example reads from
  `.env`.
- `skill/references/masumi-payments.md` — endpoint index, singular paths, new
  field names for submit-result, base-URL clarification.
- `README.md` — reflects new structure + safety section.

### Sources of Truth Used

- `https://api.sokosumi.com/v1/openapi.json` (live, fetched during v2.1 work)
- `https://api.preprod.sokosumi.com/v1/openapi.json` (live)
- `https://payment.masumi.network/docs` swagger UI spec (live)
- `https://registry.masumi.network/docs` swagger UI spec (live)
- Local `sokosumi-docs/content/docs` (Coworkers, Projects, Auth)
- Local `pip-docs/HOW_ENDPOINTS_WORK.md` for MIP-003 endpoint flow

### Pass 2 — Full Audit + Caveman Rewrite (2026-05-13, same day)

Closed remaining gaps from Pass 1:

**Verified schemas from live spec** (was: inferred):
- `POST /payment/submit-result` body: `{network, blockchainIdentifier, submitResultHash}`.
- `POST /payment/authorize-refund` body: `{network, blockchainIdentifier}`.
- `POST /registry` body: 10 required fields with case-sensitive names (`Tags`, `ExampleOutputs`,
  `Capability`, `AgentPricing`, `Author`, `Legal`; camelCase `sellingWalletVkey`,
  `apiBaseUrl`, `name`, `description`).
- `POST /registry-entry-search/` body: `network`, optional `query` (fuzzy ≤120), `filter`
  (`paymentTypes`, `status`, `policyId`, `assetIdentifier`, `tags`, `capability`), `limit`
  (1-50, default 10), `cursorId` (NOT `cursor`).
- Confirmed Payment + Registry Service auth scheme = `apiKey` header `token`.

**Audited + caveman-rewrote v2.0 reference files** (was: untouched in pass 1):
- `masumi-payments.md` — 1360 → 497 lines (-63%). Fixed `POST /registry` body, submit-result
  field names, full ~70-endpoint index, removed misleading "Masumi L2 Q3 2025" claim.
- `sokosumi-marketplace.md` — 1057 → 377 lines (-64%). Removed stale endpoint examples,
  retained concepts + MCP + Kodosumi integration.
- `registry-identity.md` — 995 → 381 lines (-62%). Fixed `paymentType` (singular) →
  `paymentTypes` enum; `page` → `cursorId`; snake_case → capitalized field names;
  added migration table.
- `agentic-services.md` — 1114 → 429 lines (-61%). Fixed submit-result body shape; added
  `/provide_input` + `/demo` as optional endpoints (was: 4 only).
- `smart-contracts.md` — 1120 → 439 lines (-61%). Fixed NFT metadata field casing; fixed
  Payment Service API field names in interaction examples; clarified Aiken snippets as
  illustrative.
- `cardano-blockchain.md` — 594 → 287 lines (-52%). Full USDM/tUSDM unit + asset name added;
  endpoint check + faucet URLs verified.
- `kodosumi-runtime.md` — 510 → 325 lines (-36%). Flagged Kodosumi APIs as not
  live-verified in this skill pass; otherwise routine compression.

**Total skill size**: 8669 → 4074 lines across 11 files (~53% reduction). SKILL.md
unchanged at 607 lines (already compressed in pass 1).

**Audit punch-lists** were produced by parallel Explore subagents (registry-identity,
agentic-services, smart-contracts+cardano-blockchain) — findings folded into the
rewrites. One agent claim ("`/payment` plural") was wrong and disregarded; the live
spec uses singular `/payment`.

---

## Version 2.0 - Ecosystem Integration Update (2025-03-09)

### Major Changes

#### 🎯 Complete Ecosystem Coverage
- **Expanded scope** from Masumi-only to full ecosystem (Masumi + Sokosumi + Kodosumi)
- **Integrated documentation** for all three platforms in a unified skill
- **Token-efficient architecture** with progressive disclosure pattern

#### 📁 New Files

1. **kodosumi-runtime.md** (NEW)
   - Comprehensive guide to Kodosumi runtime platform
   - Ray-based distributed execution
   - Flow deployment and lifecycle management
   - Integration with Masumi payments
   - 600+ lines of technical documentation

#### 📝 Updated Files

1. **skill.md** (Entry Point - sokosumi-landing/apps/masumi/public/)
   - **Reduced from 360 to ~254 lines** (30% reduction)
   - Added ecosystem overview (3 platforms)
   - Added use-case based navigation
   - Added platform-specific quick links
   - Now mentions accessibility via https://masumi.network/skill.md
   - Better organized with focus on progressive disclosure

2. **SKILL.md** (Main Guide)
   - Updated description to include all 3 platforms
   - Enhanced "What is the Masumi Ecosystem?" section
   - Added detailed platform breakdowns:
     - Masumi (Payments & Identity)
     - Sokosumi (Marketplace)
     - Kodosumi (Runtime)
   - Redesigned "Progressive Disclosure Strategy" section
   - Added reference file loading guide with table
   - Token-efficient architecture clearly explained

3. **sokosumi-marketplace.md**
   - Added ecosystem integration section
   - Added MCP Integration section for Claude Code
   - Added Kodosumi deployment workflow
   - Added integration examples showing full stack
   - Expanded documentation links

4. **README.md**
   - Updated title to "Masumi Ecosystem Developer Skill"
   - Added "Three Integrated Platforms" section
   - Listed 8 reference guides (was 7, now includes Kodosumi)
   - Added skill structure diagram
   - Added usage guide with common workflows
   - Reorganized resources by category (Platforms, Docs, Repos, Community)
   - Enhanced contributing guidelines

### Token Efficiency Improvements

#### Before (v1.x)
- Entry point: 360 lines
- Loaded all content upfront
- No clear guidance on when to load references

#### After (v2.0)
- Entry point: 254 lines (30% reduction)
- Progressive loading strategy
- Clear table showing which reference to load for each task
- Modular architecture

### Architecture

```
Entry Point (skill.md)
  ↓ (254 lines, concise overview)
Main Guide (SKILL.md)
  ↓ (Progressive disclosure strategy)
References (*.md)
  ├── masumi-payments.md (Load when: Setting up payments)
  ├── registry-identity.md (Load when: Registering agent)
  ├── smart-contracts.md (Load when: Understanding contracts)
  ├── cardano-blockchain.md (Load when: Blockchain concepts)
  ├── agentic-services.md (Load when: Building MIP-003 agent)
  ├── sokosumi-marketplace.md (Load when: Listing on marketplace)
  └── kodosumi-runtime.md (Load when: Deploying at scale) [NEW]
```

### Documentation Updates

#### Entry Point (skill.md)
**Old structure:**
1. What is Masumi?
2. Documentation Index (long lists)
3. Quick Integration Path (6 steps)
4. Key Resources
5. Integration Checklist
6. Network Information
7. Token Information

**New structure:**
1. What is the Masumi Ecosystem?
2. Common Use Cases (choose your path)
3. Platform Overviews (3 platforms)
4. Essential Documentation Links (organized by platform)
5. Quick Start Paths (3 workflows)
6. Key Resources (categorized)
7. Quick Reference (condensed)

#### Main Guide (SKILL.md)
**Added:**
- Ecosystem introduction (3 platforms)
- Progressive disclosure strategy
- Reference loading guide table
- Token-efficient usage instructions

#### Reference Files
**Enhanced:**
- Cross-platform integration examples
- Clear links between Masumi ↔ Sokosumi ↔ Kodosumi
- Workflow diagrams showing full stack

### Use Cases Now Covered

1. **Monetize Your Agent**
   - Masumi (payments) + Sokosumi (listing)
   - Path: Install → Build → Register → Fund → List

2. **Use Existing Agents**
   - Sokosumi (discovery) + Masumi (payment)
   - Path: Browse → Integrate MCP → Submit Jobs

3. **Deploy at Scale**
   - Kodosumi (runtime) + Sokosumi (marketplace)
   - Path: Install → Deploy → List

4. **Full Ecosystem**
   - All three platforms integrated
   - Build → Deploy (Kodosumi) → List (Sokosumi) → Payments (Masumi)

### Breaking Changes

None - all existing functionality preserved, only additions and improvements.

### Migration Guide

No migration needed. Existing skill installations will automatically receive updates on next pull.

**For users:**
- Entry point is now more concise and easier to navigate
- New reference file available: `kodosumi-runtime.md`
- Better guidance on which reference to load for each task

**For developers:**
- Installation command remains: `npx skills add https://github.com/masumi-network/masumi-skills --skill masumi`
- All existing reference files work as before
- New Kodosumi guide available for scalable deployments

### Statistics

- **Total lines of documentation**: ~10,000+ (up from ~6,200)
- **Reference files**: 8 (up from 7)
- **Platforms covered**: 3 (Masumi, Sokosumi, Kodosumi)
- **Entry point reduction**: 30% smaller
- **New content**: 600+ lines for Kodosumi integration

### Future Enhancements

Planned for v2.1:
- [ ] Add more code examples for Kodosumi deployment
- [ ] Add troubleshooting section for cross-platform integration
- [ ] Add video tutorials linking
- [ ] Add interactive decision tree for choosing workflow

### Contributors

- Masumi Network Team
- Community feedback incorporated

---

**Version 2.0 marks the transition from a Masumi-focused skill to a complete ecosystem skill, providing developers with end-to-end guidance for building, deploying, and monetizing AI agents at scale.**

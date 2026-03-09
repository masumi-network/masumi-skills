---
name: masumi
description: "Use this skill WHEN monetizing AI agents with autonomous payments, listing agents on decentralized marketplace, or deploying at scale. OUTCOMES: agents earning revenue automatically (50+ USDM/day), trustless escrow protection (95%+ success rate), marketplace discovery (50+ available agents), scalable execution (1000+ concurrent jobs). Triggers on: 'monetize my agent', 'setup agent payments', 'list on marketplace', 'A2A transactions', 'deploy agent at scale', 'MIP-003 implementation'. NOT for: general chatbots, simple REST APIs, centralized payments. Framework-agnostic (CrewAI, AutoGen, PhiData, LangGraph), self-hosted, MiCA compliant."
---

# Masumi Ecosystem Skill

## What This Skill Does

Enables AI agents to **earn revenue autonomously** through decentralized payments, marketplace listing, and scalable deployment. Transform any AI agent into a monetized service without centralized intermediaries.

## When to Use This Skill

### ✅ Use This Skill When:
- **Monetizing AI agents** - You want agents to earn revenue automatically
- **Agent-to-Agent (A2A) payments** - Autonomous transactions between agents
- **Marketplace listing** - Making your agent discoverable to users
- **Trustless escrow** - Need payment protection without middlemen
- **Deploying at scale** - Handle 100+ concurrent jobs with Ray
- **MiCA compliance** - Building in EU with regulatory requirements
- **MIP-003 implementation** - Standardized agentic service API

### ❌ Do NOT Use This Skill When:
- Building general chatbots without payment needs
- Simple REST API development (use standard frameworks)
- Centralized payment processing is acceptable
- Low transaction volume (< 10 transactions/day)
- Speed is more critical than trustlessness
- No dispute resolution needed

## Key Capabilities & Outcomes

### 1. Autonomous Revenue Generation
**Capability:** Agents accept payments and deliver services without manual intervention
**Good Outcome:** Agent earning 50-200 USDM/day, 95%+ payment success rate, zero manual processing
**Bad Outcome:** Manual payment collection, high failure rate, customer support needed

### 2. Trustless Escrow Protection
**Capability:** Smart contracts hold funds until work is verified
**Good Outcome:** Zero payment disputes, automatic fund release, verifiable decision logs
**Bad Outcome:** Chargebacks, disputes requiring mediation, trust issues

### 3. Marketplace Discovery
**Capability:** List once, discovered by thousands of potential users
**Good Outcome:** 100+ monthly jobs from marketplace, 4.5+ star rating, organic growth
**Bad Outcome:** No visibility, manual customer acquisition, low utilization

### 4. Scalable Execution (Kodosumi)
**Capability:** Distributed Ray cluster handles 1000+ concurrent jobs
**Good Outcome:** 99% uptime, <2s latency, automatic scaling, efficient resource use
**Bad Outcome:** Bottlenecks, crashes under load, manual scaling needed

### 5. Conversational Agent Hiring (MCP)
**Capability:** Use agents directly from Claude via Sokosumi MCP
**Good Outcome:** Seamless discovery, natural language hiring, automatic result retrieval
**Bad Outcome:** Manual API calls, complex integration, no tracking

## The Three Platforms

### 1. Masumi - Payment & Identity Protocol
Decentralized protocol on Cardano blockchain for:
- **Agent-to-Agent (A2A) Payments** - Autonomous transactions between AI agents
- **Human-to-Agent (H2A) Payments** - Direct payment for services
- **Decentralized Identity** - W3C DIDs and Verifiable Credentials
- **On-Chain Registry** - NFT-based agent registration
- **Decision Logging** - Accountability through blockchain hashing

### 2. Sokosumi - Agent Marketplace
Discovery and access platform for:
- **Agent Listing** - Showcase your agent's capabilities
- **Agent Discovery** - Find agents by category and features
- **Job Management** - Track tasks and conversations
- **API Integration** - Programmatic access to marketplace
- **MCP Server** - Direct Claude integration

### 3. Kodosumi - Scalable Runtime
Ray-based distributed execution for:
- **Flow Deployment** - Deploy agents/services at scale
- **Lifecycle Management** - Automated execution control
- **Event Streaming** - Real-time job tracking
- **Access Control** - Authentication and authorization
- **Python-First** - Built for Python AI frameworks

## Core Philosophy

**Framework Agnostic**: Works with any AI framework - CrewAI, AutoGen, PhiData, LangGraph, custom solutions.

**Self-Hosted & Decentralized**: Run your own Masumi Node, deploy on your infrastructure.

**MiCA Compliant**: Enterprise-ready with EU regulatory compliance (MiCA, GDPR, AI Act).

## Workflow Architecture

### Workflow 1: Monetize Your Agent (Sequential)

**Use Case:** Turn existing AI agent into revenue-generating service

**Outcome:** Agent earning 50+ USDM/day with 95%+ payment success

**Tools Needed:**
- Built-in: File operations, API calls
- MCP: Sokosumi (for listing verification)

**Sequential Steps:**

```
1. Install Payment Node
   → Outcome: Local node running, admin dashboard accessible
   → Tool: Bash (git clone, npm install)
   → Check: http://localhost:3001 loads

2. Configure Wallets
   → Outcome: 3 wallets created (purchasing, selling, collection)
   → Tool: Admin dashboard
   → Check: Wallet addresses visible

3. Fund Test Wallets
   → Outcome: Wallets have ADA + USDM on Preprod
   → Tool: Faucets (https://faucet.masumi.network)
   → Check: Balance > 100 ADA, 1000 USDM

4. Implement MIP-003 API
   → Outcome: Agent has POST /start_job, GET /status endpoints
   → Tool: Your framework (CrewAI, AutoGen, etc.)
   → Check: Endpoints return valid responses
   → Load Reference: agentic-services.md

5. Test Payment Flow (Preprod)
   → Outcome: Complete test transaction succeeds
   → Tool: Payment Service API
   → Check: Job completes, payment received
   → Iterate: Fix issues, repeat until 100% success

6. Register Agent
   → Outcome: Agent NFT minted, on-chain registry entry
   → Tool: Registry API
   → Check: blockchain_identifier returned
   → Load Reference: registry-identity.md

7. List on Sokosumi
   → Outcome: Agent visible on marketplace
   → Tool: Sokosumi submission form
   → Check: Agent appears in search results
   → Load Reference: sokosumi-marketplace.md

8. Monitor & Optimize
   → Outcome: Revenue growing, uptime > 95%
   → Tool: Admin dashboard, analytics
   → Iterate: Adjust pricing, improve quality
```

**Iterative Refinement Loop:**
```
Test on Preprod → Debug issues → Optimize → Test again → Move to Mainnet
```

### Workflow 2: Use Existing Agents via MCP (MCP Coordination)

**Use Case:** Access 50+ agents from Claude without manual API calls

**Outcome:** Seamless agent hiring with automatic result retrieval

**Tools Needed:**
- MCP: Sokosumi (primary)
- Built-in: File read/write (for saving results)

**Workflow:**

```
1. Install Sokosumi MCP
   → Outcome: MCP server running, Claude connected
   → Tool: npm install @sokosumi/mcp-server
   → Check: MCP listed in Claude settings

2. Configure API Key
   → Outcome: Authenticated access to marketplace
   → Tool: Environment variables
   → Check: Can list agents

3. Discover Agents (Conversational)
   User: "Find me a data analysis agent"
   → MCP: Lists top agents with ratings
   → Outcome: User sees 5-10 relevant agents

4. Submit Job (Natural Language)
   User: "Use the Data Analyzer Pro to analyze this CSV"
   → MCP: Creates job, handles payment
   → Outcome: Job submitted, tracking started

5. Auto-Monitor Progress
   → MCP: Polls job status every 10s
   → Outcome: User sees real-time updates
   → No manual API calls needed

6. Retrieve Results
   → MCP: Fetches completed results
   → Outcome: Results displayed in Claude
   → Tool: File write (save results locally)
```

**Multi-MCP Coordination:**
```
Sokosumi MCP (agent discovery)
     ↓
Built-in File tools (save results)
     ↓
[Future] Blockchain MCP (verify transactions)
```

### Workflow 3: Deploy at Scale (Orchestration)

**Use Case:** Handle 1000+ concurrent jobs with auto-scaling

**Outcome:** 99% uptime, <2s latency, automatic scaling

**Tools Needed:**
- Built-in: Docker, deployment scripts
- MCP: None initially (focus on infrastructure)

**Orchestration Pattern:**

```
┌─────────────────────────────────────┐
│  PARALLEL: Can run simultaneously   │
└─────────────────────────────────────┘

1a. Deploy on Kodosumi               1b. Register on Masumi
    → Package as Flow                     → Create registry entry
    → Deploy to Ray cluster               → Get blockchain_identifier
    → Outcome: Scalable endpoint          → Outcome: On-chain identity
    ↓                                     ↓
    └─────────────┬─────────────────────┘
                  ↓
┌─────────────────────────────────────┐
│  SEQUENTIAL: Must follow order      │
└─────────────────────────────────────┘

2. Configure Masumi Integration
   → Link Kodosumi endpoint to payment service
   → Outcome: Payments trigger Kodosumi jobs

3. List on Sokosumi
   → Use Kodosumi endpoint in listing
   → Outcome: Marketplace routes to scalable infrastructure

4. Monitor & Scale
   → Kodosumi auto-scales based on load
   → Masumi tracks payments
   → Sokosumi shows job count
   → Outcome: System handles 1000+ jobs/hour
```

**Load Reference Files:**
- `kodosumi-runtime.md` for deployment
- `masumi-payments.md` for payment integration
- `sokosumi-marketplace.md` for listing

## When to Load Reference Files

| Your Task | Load This Reference |
|-----------|-------------------|
| Setting up payments | `masumi-payments.md` |
| Registering agent | `registry-identity.md` |
| Understanding smart contracts | `smart-contracts.md` |
| Cardano blockchain concepts | `cardano-blockchain.md` |
| Listing on marketplace | `sokosumi-marketplace.md` |
| Deploying at scale | `kodosumi-runtime.md` |
| Building MIP-003 agent | `agentic-services.md` |

**Load only what you need to minimize token usage.**

## Domain-Specific Intelligence

### When Blockchain Adds Value

**✅ Use Masumi (Blockchain) When:**

1. **Trustless Escrow Needed**
   - Scenario: Buyer and seller don't trust each other
   - Example: Freelance agent hired by unknown client
   - Outcome: Smart contract holds funds, releases on completion
   - Alternative: Centralized escrow (PayPal, Stripe) - requires trust in platform

2. **Agent-to-Agent Transactions**
   - Scenario: AI agents transacting autonomously without human intervention
   - Example: Research agent hiring data analysis agent
   - Outcome: Autonomous payments, no manual approval loops
   - Alternative: Human approves every transaction - not scalable

3. **Decentralized Reputation**
   - Scenario: Need verifiable track record without central authority
   - Example: Agent has 500 successful jobs, provable on-chain
   - Outcome: Reputation can't be faked or deleted
   - Alternative: Platform-controlled ratings - can be manipulated

4. **Regulatory Compliance (MiCA)**
   - Scenario: Operating in EU with AI Act + MiCA requirements
   - Example: Enterprise B2B agent services
   - Outcome: Built-in compliance, decision logging
   - Alternative: Custom compliance - expensive to build

**Cost-Benefit Analysis:**
```
Blockchain Path:
  Cost: Transaction fees (~0.2 ADA/tx), complexity, learning curve
  Benefit: Trustless, transparent, programmable escrow, compliance

Centralized Path:
  Cost: Trust dependency, platform fees (3-5%), vendor lock-in
  Benefit: Faster, simpler, familiar
```

### When Blockchain is Overkill

**❌ Don't Use Masumi (Use Alternatives) When:**

1. **Trusted Parties**
   - Scenario: Internal company agents, known partners
   - Example: Team's own agents communicating
   - Better: Direct API calls, internal billing
   - Why: No trust issue to solve

2. **Low-Value Transactions**
   - Scenario: Transactions < $1 USD
   - Example: Micro-tasks like text formatting
   - Better: Bundled credits, monthly billing
   - Why: Transaction fees eat into profit

3. **Speed Over Decentralization**
   - Scenario: Sub-second response time critical
   - Example: Real-time trading agent
   - Better: Centralized payment processor
   - Why: Blockchain confirmation takes ~20s

4. **No Dispute Resolution Needed**
   - Scenario: Instant, verifiable outputs
   - Example: Text translation (deterministic)
   - Better: Simple API with API keys
   - Why: Escrow adds unnecessary complexity

### Decision Tree

```
Do you need trustless escrow?
├─ YES → Are transactions > $1?
│         ├─ YES → Is decentralization worth complexity?
│         │         ├─ YES → Use Masumi ✓
│         │         └─ NO → Use Stripe/PayPal
│         └─ NO → Bundle into higher-value transactions
└─ NO → Are you doing A2A payments?
          ├─ YES → Is autonomy important?
          │         ├─ YES → Use Masumi ✓
          │         └─ NO → Use traditional API
          └─ NO → Use traditional payment processor
```

### Platform Selection Guide

**Use Masumi Only:**
- Payment infrastructure + identity
- Agent-to-agent transactions
- When: Blockchain value justified

**Use Sokosumi Only:**
- Agent discovery (credit-based)
- Simple marketplace listing
- When: Don't need blockchain payments

**Use Kodosumi Only:**
- Scalable execution
- Ray-based deployment
- When: Don't need marketplace or payments

**Use All Three:**
- Full decentralized agent economy
- Maximum scalability + trustlessness
- When: Building enterprise agent network

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Your Agentic Service                      │
│          (CrewAI, AutoGen, PhiData, LangGraph, etc.)        │
│                                                               │
│  Implements: MIP-003 Agentic Service API                    │
│  - POST /start_job                                           │
│  - GET /status                                               │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    │ Your API
                    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Masumi Payment Service                    │
│                    (Self-hosted Node)                        │
│                                                               │
│  • Wallet Management (3 wallets)                            │
│  • Payment Processing                                        │
│  • Registry Operations                                       │
│  • Admin Dashboard                                           │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    │ Blockchain Transactions
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              Cardano Blockchain (L1)                         │
│                                                               │
│  • Payment Smart Contract (escrow)                          │
│  • Registry Smart Contract (NFT-based)                      │
│  • Decision Logging (on-chain hashes)                       │
└─────────────────────────────────────────────────────────────┘
```

## Key Concepts Summary

### Three Wallets System
- **Purchasing Wallet**: Managed by node, funds outgoing payments and registry fees
- **Selling Wallet**: Managed by node, receives payments for your services
- **Collection Wallet**: Your external wallet (hardware/Eternl), for withdrawals

### Payment Flow
1. Buyer locks USDM in smart contract
2. Seller completes job, submits output hash
3. Dispute period starts (configurable)
4. After dispute period, seller collects payment

### Registry (NFT-Based)
- Each agent = unique NFT in your wallet
- Metadata stored on-chain
- Query via Registry Service API (free)
- Register/deregister via Payment Service API (costs ADA)

### Decision Logging
- Hash of (input + output) stored on-chain
- Enables accountability without exposing data
- Required to unlock payment from smart contract

## Token Economics

- **$ADA**: Cardano native token, pays blockchain transaction fees
- **$USDM**: Fiat-backed stablecoin (USD peg), pays for agentic services
- **$SUMI**: Future governance token (not yet launched)

**Network Fee**: 5% of service price in USDM goes to Masumi Network

## Environments

### Preprod (Testing)
- Free test ADA from faucet
- Mirror of mainnet for safe testing
- Explorer: https://preprod.cardanoscan.io
- No real money risk

### Mainnet (Production)
- Real ADA and USDM
- Permanent blockchain records
- Production Masumi Explorer
- Regulatory compliance required

## Progressive Disclosure Strategy

**Token-Efficient Loading**: This SKILL.md provides high-level guidance. Load reference files only when needed for specific tasks.

### Platform-Specific References

**Masumi (Payments & Identity):**
- `references/masumi-payments.md` - Payment flows, APIs, decision logging, wallet management
- `references/registry-identity.md` - Registry operations, DIDs, verifiable credentials, NFT metadata
- `references/smart-contracts.md` - Payment/Registry contracts, security, Aiken code
- `references/cardano-blockchain.md` - Blockchain fundamentals, UTXO model, transaction fees

**Sokosumi (Marketplace):**
- `references/sokosumi-marketplace.md` - Listing agents, browsing marketplace, job management, MCP integration

**Kodosumi (Runtime):**
- `references/kodosumi-runtime.md` - Flow deployment, lifecycle management, Ray cluster configuration

**Cross-Platform:**
- `references/agentic-services.md` - MIP-003 API standard, framework integration patterns

### When to Load Each Reference

| Your Task | Load This Reference |
|-----------|-------------------|
| Set up payments | `masumi-payments.md` |
| Register agent | `registry-identity.md` |
| Understand smart contracts | `smart-contracts.md` |
| Cardano blockchain concepts | `cardano-blockchain.md` |
| List on marketplace | `sokosumi-marketplace.md` |
| Deploy at scale | `kodosumi-runtime.md` |
| Build MIP-003 compatible agent | `agentic-services.md` |

**Only load references relevant to the current task to minimize token usage.**

## Best Practices

### Development Workflow
1. ✅ Always start on **Preprod** environment
2. ✅ Test full payment flow before mainnet
3. ✅ Export and backup wallet seed phrases
4. ✅ Monitor node uptime and health
5. ✅ Implement proper error handling

### Security
1. ✅ Never commit seed phrases to git
2. ✅ Use hardware wallet for Collection Wallet on mainnet
3. ✅ Minimize funds in node-managed wallets
4. ✅ Set up automated collection to Collection Wallet
5. ✅ Store seed phrases offline (paper, fire/water-safe)

### Quality for Agents
1. ✅ Provide clear descriptions in registry
2. ✅ Publish realistic "Example Output"
3. ✅ Set accurate processing time estimates
4. ✅ Implement health checks and monitoring
5. ✅ Have Terms of Service and Privacy Policy

### Avoiding Disputes
1. ✅ Don't overpromise in registry metadata
2. ✅ Ensure uptime and API key funding
3. ✅ Correctly hash input/output for validation
4. ✅ Meet the processing times you advertise
5. ✅ Test with real workloads before launch

## Common Integration Patterns

### Pattern 1: Simple Seller (Earn from your agent)
```
You → Run Masumi Node → Register Agent → Receive Payments
```

### Pattern 2: Simple Buyer (Use other agents)
```
You → Run Masumi Node → Search Registry → Purchase Services
```

### Pattern 3: Marketplace (Sokosumi Simple Mode)
```
You → Get Sokosumi API Key → List Agent → Receive Jobs (USDM)
```

### Pattern 4: Full Ecosystem (Advanced)
```
You → Masumi Node + Sokosumi → Agent Network + Payments + Identity
```

## Essential Resources

### Documentation
- Main Docs: https://docs.masumi.network
- MIP-003 Standard: https://github.com/masumi-network/masumi-improvement-proposals
- Registry Metadata Spec: See references/registry-identity.md

### Repositories
- Payment Service: https://github.com/masumi-network/masumi-payment-service
- Python SDK: https://github.com/masumi-network/pip-masumi
- Examples: https://github.com/masumi-network/pip-masumi-examples
- Cardano Toolbox: https://github.com/masumi-network/cardano-toolbox
- Sokosumi: https://github.com/masumi-network/sokosumi
- Explorer: https://github.com/masumi-network/masumi-explorer

### Tools
- Masumi Faucet: https://faucet.masumi.network
- Cardano Faucet: https://docs.cardano.org/cardano-testnet/tools/faucet/
- Sokosumi Marketplace: https://app.sokosumi.com
- Preprod Explorer: https://preprod.cardanoscan.io

## Troubleshooting Quick Reference

| Issue | Solution |
|-------|----------|
| Node won't start | Check PostgreSQL running, run `npm run db:migrate` |
| No test ADA | Use faucet, wait 5 min, check wallet address is correct |
| Payment not detected | Verify network (Preprod/Mainnet), check blockchain_identifier |
| Can't register agent | Ensure wallet has ADA for fees, check API key is admin |
| Dispute filed | Check output hash matches, verify quality vs Example Output |
| Lost seed phrase | If not backed up, funds are permanently lost - always backup |

## Support and Community


- Documentation: https://docs.masumi.network
- GitHub Issues: https://github.com/masumi-network
- Email: hello@masumi.network

---

**Remember**: Masumi is **permissionless** and **trustless**. No central authority controls the network. You run your own node, manage your own wallets, and interact directly with blockchain smart contracts.

Start with Preprod, master the basics, then scale to Mainnet. Welcome to the Agentic Economy! 🚀

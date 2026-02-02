# Masumi OpenClaw Skills Monorepo

**Modular OpenClaw skills for Cardano blockchain payments and agent collaboration**

This repository contains three self-contained OpenClaw skills that can be loaded independently:

1. **Cardano Wallet** - Generate, manage, and fund Cardano wallets
2. **Masumi Payments** - Install payment service, generate API keys, register agents, handle payments
3. **Sokosumi Marketplace** - Hire agents from Sokosumi marketplace

## Repository Structure

```
masumi-openclaw-skills/
├── skills/
│   ├── cardano-wallet/         # Skill 1: Cardano Wallet Management
│   │   ├── plugin.json
│   │   ├── SKILL.md
│   │   ├── package.json
│   │   └── src/
│   │
│   ├── masumi-payments/        # Skill 2: Masumi Payments & Registry
│   │   ├── plugin.json
│   │   ├── SKILL.md
│   │   ├── package.json
│   │   └── src/
│   │
│   └── sokosumi-marketplace/    # Skill 3: Sokosumi Marketplace
│       ├── plugin.json
│       ├── SKILL.md
│       ├── package.json
│       └── src/
│
└── shared/                      # Shared types
    └── types/
        └── config.ts
```

## Skills Overview

### Cardano Wallet Skill

**Purpose**: Generate, manage, and fund Cardano wallets

**Tools**:
- `cardano_generate_wallet` - Generate new wallet (24-word mnemonic)
- `cardano_restore_wallet` - Restore wallet from mnemonic
- `cardano_get_wallet_address` - Get public address
- `cardano_generate_funding_qr` - Generate QR code for funding wallet
- `cardano_get_wallet_balance` - Check wallet balance (requires Blockfrost)
- `cardano_backup_wallet` - Securely backup wallet credentials

**Dependencies**: `@meshsdk/core`, `qrcode`, `@blockfrost/blockfrost-js`

**Documentation**: See `skills/cardano-wallet/SKILL.md`

### Masumi Payments Skill

**Purpose**: Install payment service, generate API keys, register agents, handle payments

**Tools**:
- `masumi_install_payment_service` - Clone and install masumi-payment-service locally
- `masumi_start_payment_service` - Start the payment service (check if running)
- `masumi_generate_api_key` - Generate admin API key via payment service API
- `masumi_enable` - Full setup: install service, generate API key, register agent
- `masumi_create_payment` - Create payment request
- `masumi_check_payment` - Check payment status
- `masumi_complete_payment` - Submit result and complete payment
- `masumi_register_agent` - Register agent in Masumi registry
- `masumi_search_agents` - Search for other agents
- `masumi_get_agent` - Get agent details

**Dependencies**: `zod`, `node-fetch`, `canonicaljson`, `@meshsdk/core`, `@blockfrost/blockfrost-js`

**Documentation**: See `skills/masumi-payments/SKILL.md`

**Important**: Masumi is a **self-hosted, decentralized service**. Each user runs their own payment service node. There is NO centralized `payment.masumi.network` service.

### Sokosumi Marketplace Skill

**Purpose**: Hire agents from Sokosumi marketplace

**Tools**:
- `sokosumi_list_agents` - Browse available agents
- `sokosumi_hire_agent` - Hire an agent for a task
- `sokosumi_check_job` - Check job status
- `sokosumi_get_result` - Get completed job results

**Dependencies**: `node-fetch`

**Documentation**: See `skills/sokosumi-marketplace/SKILL.md`

**Payment Modes**:
- **Simple mode**: Just Sokosumi API key (Sokosumi handles payments in USDM)
- **Advanced mode**: Uses Masumi Payments skill for self-hosted payments

## Installation

### Install All Skills

```bash
npm install
```

### Install Individual Skill

```bash
# Cardano Wallet
cd skills/cardano-wallet
npm install

# Masumi Payments
cd skills/masumi-payments
npm install

# Sokosumi Marketplace
cd skills/sokosumi-marketplace
npm install
```

## Building

### Build All Skills

```bash
npm run build
```

### Build Individual Skill

```bash
npm run build:cardano-wallet
npm run build:masumi-payments
npm run build:sokosumi-marketplace
```

## Loading Skills in OpenClaw

Each skill can be loaded independently:

```typescript
// Load Cardano Wallet skill
import '@masumi/cardano-wallet-skill';

// Load Masumi Payments skill
import '@masumi/masumi-payments-skill';

// Load Sokosumi Marketplace skill
import '@masumi/sokosumi-marketplace-skill';
```

Or load via plugin.json:

```json
{
  "plugins": [
    "skills/cardano-wallet/plugin.json",
    "skills/masumi-payments/plugin.json",
    "skills/sokosumi-marketplace/plugin.json"
  ]
}
```

## Quick Start

### 1. Cardano Wallet

```typescript
import { cardano_generate_wallet, cardano_generate_funding_qr } from '@masumi/cardano-wallet-skill';

// Generate wallet
const wallet = await cardano_generate_wallet({ network: 'Preprod' });

// Generate QR code for funding
const qr = await cardano_generate_funding_qr({
  address: wallet.address,
  network: 'Preprod'
});
```

### 2. Masumi Payments

```typescript
import { masumi_enable, masumi_create_payment } from '@masumi/masumi-payments-skill';

// Enable Masumi (auto-installs service if needed)
await masumi_enable({
  agentName: 'My Agent',
  pricingTier: 'free',
  installService: true
});

// Create payment request
const payment = await masumi_create_payment({
  buyerIdentifier: 'buyer123',
  taskDescription: 'Analyze data'
});
```

### 3. Sokosumi Marketplace

```typescript
import { sokosumi_list_agents, sokosumi_hire_agent } from '@masumi/sokosumi-marketplace-skill';

// List available agents
const agents = await sokosumi_list_agents();

// Hire an agent
const job = await sokosumi_hire_agent({
  agentId: 'agent-123',
  inputData: JSON.stringify({ task: 'Analyze data' }),
  maxAcceptedCredits: 100
});
```

## Configuration

### Environment Variables

Each skill has its own configuration. See individual SKILL.md files for details.

**Cardano Wallet**:
- `CARDANO_NETWORK`: "Preprod" or "Mainnet" (default: "Preprod")
- `BLOCKFROST_API_KEY`: Blockfrost API key (for balance checks)

**Masumi Payments**:
- `MASUMI_PAYMENT_SERVICE_URL`: YOUR self-hosted payment service URL (required)
- `MASUMI_PAYMENT_API_KEY`: Admin API key
- `MASUMI_NETWORK`: "Preprod" or "Mainnet" (default: "Preprod")

**Sokosumi Marketplace**:
- `SOKOSUMI_API_KEY`: Sokosumi API key (required)
- `SOKOSUMI_API_ENDPOINT`: API endpoint (default: https://sokosumi.com/api/v1)

## Important Notes

### Masumi is Self-Hosted

**CRITICAL**: Masumi is NOT a centralized service. Each user runs their own payment service node:
- Local: `http://localhost:3000/api/v1`
- Railway: `https://your-service.railway.app/api/v1`

There is NO centralized `payment.masumi.network` service.

### Skill Dependencies

- **Cardano Wallet**: Standalone, no dependencies on other skills
- **Masumi Payments**: Standalone, includes wallet generation for auto-provisioning
- **Sokosumi Marketplace**: Can optionally use Masumi Payments skill for advanced mode

## Development

### Workspace Management

This repository uses npm workspaces. All skills share the root `node_modules` for common dependencies.

### TypeScript Configuration

Each skill extends the root `tsconfig.json`:

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  }
}
```

## Resources

- **Masumi Documentation**: https://docs.masumi.network
- **Payment Service**: https://github.com/masumi-network/masumi-payment-service
- **Registry Service**: https://github.com/masumi-network/masumi-registry-service
- **Sokosumi**: https://sokosumi.com

## License

MIT

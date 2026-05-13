# Masumi Ecosystem Developer Skill

Expert guidance for building AI agents with payments, identity, marketplace integration, and scalable deployment across the complete Masumi ecosystem.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Masumi Network](https://img.shields.io/badge/Masumi-Network-blue)](https://docs.masumi.network)

## Installation

### Via skills.sh
```bash
npx skills add https://github.com/masumi-network/masumi-skills --skill masumi
```

### Manual
```bash
git clone https://github.com/masumi-network/masumi-skills
cd masumi-skills
./install.sh
```

Works with: Claude Code, Cursor, Windsurf, Cline, Aider, Codex, and other AI coding assistants.

## What's Included

The skill provides comprehensive guidance for the **complete Masumi ecosystem**:

### Three Integrated Platforms

1. **Masumi** - Decentralized payment & identity protocol (Cardano blockchain)
2. **Sokosumi** - Agent marketplace for discovery and hiring
3. **Kodosumi** - Scalable runtime environment (Ray-based execution)

### Reference Guides (v2.1 — refreshed for current live APIs)

**Cross-cutting / API debugging:**
- **API Debug Recipes** (NEW) — Safe `.env` handling + runnable curl/Python recipes for every service
- **Sokosumi API Reference** (NEW) — Full v1 endpoint catalog from the live OpenAPI spec
- **Masumi Registry API** (NEW) — Registry Service endpoints (separate from Payment Service)

**Per platform:**
- **Masumi Payments** — Payment Service endpoints, flows, decision logging
- **Registry & Identity** — On-chain discovery concepts, DIDs, verifiable credentials
- **Smart Contracts** — Escrow, payment/registry contracts, security
- **Cardano Blockchain** — UTXO model, transactions, fees
- **Agentic Services** — MIP-003 API standard implementation
- **Sokosumi Marketplace** — Listing concepts, MCP integration
- **Kodosumi Runtime** — Deployment, scaling, lifecycle management

### Token-Efficient Architecture

The skill uses **progressive disclosure** - load only what you need:
- **Entry Point** (skill.md): Quick overview and navigation
- **Main Guide** (SKILL.md): Architecture and common workflows
- **References** (references/*.md): Deep dives for specific tasks

**~10,000+ lines** of technical documentation with code examples for TypeScript, Python, and framework-agnostic implementations (CrewAI, AutoGen, LangGraph, PhiData).

## Skill Structure

```
masumi-skills/
├── .env.example                              # Variable template (copy to .env)
├── skill/
│   ├── SKILL.md                              # Main skill guide
│   └── references/
│       ├── api-debug-recipes.md              # (NEW) Safe .env + curl/Python recipes
│       ├── sokosumi-api-reference.md         # (NEW) Full Sokosumi v1 endpoint surface
│       ├── masumi-registry-api.md            # (NEW) Registry Service endpoints
│       ├── masumi-payments.md                # Payment Service deep dive
│       ├── registry-identity.md              # Registry concepts: DIDs, NFTs
│       ├── smart-contracts.md                # Blockchain contracts
│       ├── cardano-blockchain.md             # Cardano fundamentals
│       ├── agentic-services.md               # MIP-003 API standard
│       ├── sokosumi-marketplace.md           # Marketplace concepts
│       └── kodosumi-runtime.md               # Scalable deployment
└── README.md                                 # This file
```

## Safe API-Key Handling

When the skill helps an agent debug live APIs, it reads keys **only** from a local `.env` file. It will never prompt you to paste a key into chat, never log a key, and never commit one. Mandatory rules are in `skill/references/api-debug-recipes.md`. To get started:

```bash
cp .env.example .env       # fill in only the keys you actually need
# .env is already gitignored
```

**Also available:** Entry point at https://masumi.network/skill.md

## Resources

### Ecosystem Platforms
- **Masumi Network**: https://masumi.network
- **Sokosumi Marketplace**: https://app.sokosumi.com
- **Kodosumi Runtime**: https://kodosumi.io

### Documentation
- **Masumi Docs**: https://docs.masumi.network
- **Sokosumi Docs**: https://docs.sokosumi.com
- **Kodosumi Docs**: https://docs.kodosumi.io

### Repositories
- **Payment Service**: https://github.com/masumi-network/masumi-payment-service
- **Python SDK**: https://github.com/masumi-network/pip-masumi
- **Masumi Skills**: https://github.com/masumi-network/masumi-skills

### Community
- **X**: [@MasumiNetwork](https://x.com/MasumiNetwork)
- **Email**: hello@masumi.network

## Usage Guide

### For AI Agents

The skill automatically provides context-aware guidance:

**When you're working on payments:**
→ Loads `references/masumi-payments.md`

**When you're listing on marketplace:**
→ Loads `references/sokosumi-marketplace.md`

**When you're deploying at scale:**
→ Loads `references/kodosumi-runtime.md`

**When you're implementing MIP-003:**
→ Loads `references/agentic-services.md`

### Common Workflows

**1. Monetize Your Agent:**
```
Install Masumi Node → Build Agent → Register → Fund Wallets → List on Sokosumi
```

**2. Use Existing Agents:**
```
Browse Sokosumi → Integrate MCP → Submit Jobs
```

**3. Deploy at Scale:**
```
Build Agent → Deploy on Kodosumi → List on Sokosumi → Payments via Masumi
```

## Contributing

Contributions welcome! Update markdown files in `skill/` directory and submit a PR.

**Guidelines:**
- Keep documentation concise and token-efficient
- Include code examples for TypeScript and Python
- Link to official docs for deeper context
- Test changes with AI assistants before submitting

**YAML frontmatter required** in `skill/SKILL.md`:
```yaml
---
name: masumi
description: Build AI agents with the Masumi ecosystem...
---
```

## License

MIT - see [LICENSE](LICENSE) file.

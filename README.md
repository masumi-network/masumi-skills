# Masumi Network Developer Skill

Expert guidance for building AI agents with payments, identity, and marketplace integration on Masumi Network.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Masumi Network](https://img.shields.io/badge/Masumi-Network-blue)](https://docs.masumi.network)

## Installation

### Via skills.sh
```bash
npx skills add masumi-network/masumi-skills
```

### Manual
```bash
git clone https://github.com/masumi-network/masumi-skills
cd masumi-skills
./install.sh
```

Works with: Claude Code, Cursor, Windsurf, Cline, Aider, Codex, and other AI coding assistants.

## What's Included

7 comprehensive guides covering:

- **Masumi Payments** - Agent-to-Agent payment integration
- **Sokosumi Marketplace** - List and hire agents
- **Cardano Blockchain** - UTXO model, wallets, transactions
- **Registry & Identity** - Decentralized discovery and DIDs
- **Agentic Services** - MIP-003 API standard compliance
- **Smart Contracts** - Escrow, decision logging, security

**~6,200 lines** of technical documentation with code examples for TypeScript, Python, and framework-agnostic implementations (CrewAI, AutoGen, LangGraph, PhiData).

## Resources

- **Docs**: https://docs.masumi.network
- **Payment Service**: https://github.com/masumi-network/masumi-payment-service
- **Python SDK**: https://github.com/masumi-network/pip-masumi
- **Sokosumi**: https://app.sokosumi.com
- **X**: [@MasumiNetwork](https://x.com/MasumiNetwork)

## Contributing

Contributions welcome! Update markdown files in `skill/` directory and submit a PR.

**YAML frontmatter required** in `skill/SKILL.md`:
```yaml
---
name: masumi
description: Expert guidance for building AI agents...
user-invocable: true
---
```

## License

MIT - see [LICENSE](LICENSE) file.

# pharos-bridge

Cross-chain bridge skill for Pharos Network AI agents.

## Overview

Enables AI agents to bridge tokens between Pharos Network and supported EVM chains (Ethereum, Base, Arbitrum, Optimism, Polygon, Avalanche, BSC) via InterPort Finance infrastructure.

## Supported Tokens

| Token | Method | Chains |
|-------|--------|--------|
| **USDC** | Circle CCTP v2 | Pharos, Ethereum, Base, Arbitrum, Optimism, Polygon, Avalanche, BSC |
| **PROS** | Chainlink CCIP | Pharos, Base, Ethereum |

## Features

- Native USDC bridging via CCTP v2 (no wrapped tokens, 1:1)
- PROS token bridging via Chainlink CCIP
- Bidirectional: bridge to AND from Pharos
- Pre-flight checks: balance verification, fee estimation, user confirmation
- Foundry-based (`cast` / `forge`) — no additional dependencies
- Full Pharos chain toolkit out of the box: balance checks, transfers, contract deployment/verification, batch airdrops, and auto-generated interaction scripts (JS/TS/Python)

## Usage

The skill is invoked by an AI agent when the user requests a cross-chain token transfer:

- "Bridge 200 USDC from Base to Pharos"
- "Transfer 10 PROS from Pharos to Ethereum"
- "Move 500 USDC from Arbitrum to Pharos"

## Prerequisites

- [Foundry](https://book.getfoundry.sh/) (`cast`, `forge`)
- `jq` for JSON parsing
- `PRIVATE_KEY` environment variable for signing transactions

## License

MIT

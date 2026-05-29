# pharos-bridge

Cross-chain bridge skill for Pharos Network AI agents.

## Overview

Enables AI agents to bridge tokens between Pharos Network and supported EVM chains (Ethereum, Base, Arbitrum, Optimism, Polygon, Avalanche, BSC) via Circle CCTP V2 and Chainlink CCIP.

## Supported Tokens

| Token | Method | Chains |
|-------|--------|--------|
| **USDC** | Circle CCTP V2 (burn-and-mint) | Pharos, Ethereum, Base, Arbitrum, Optimism, Polygon, Avalanche, BSC |
| **PROS** | Chainlink CCIP | Pharos, Base, Ethereum |

## Features

- Native USDC bridging via Circle CCTP V2 (burn-and-mint, no wrapped tokens, 1:1)
- PROS token bridging via Chainlink CCIP
- Bidirectional: bridge to AND from Pharos
- Automated pipeline: approve → burn → poll attestation → mint
- Foundry-based (`cast` / `forge`) — no additional dependencies
- Full Pharos chain toolkit out of the box: balance checks, transfers, contract deployment/verification, batch airdrops, and auto-generated interaction scripts (JS/TS/Python)

## Usage

The skill is invoked by an AI agent when the user requests a cross-chain token transfer:

- "Bridge 1 USDC from Pharos to Base"
- "Transfer 10 PROS from Pharos to Ethereum"
- "Move 500 USDC from Arbitrum to Pharos"

## Prerequisites

- [Foundry](https://book.getfoundry.sh/) (`cast`, `forge`)
- `jq` for JSON parsing

## Setting Up Private Key

On first run, the skill creates a `.env` file in your project directory. Open it and add your private key:

```
PRIVATE_KEY=your_key_without_0x_prefix
```

**How to get your key:** MetaMask → Account Details → Show Private Key.

**Never paste your key in chat or commit it to git.**

## License

MIT

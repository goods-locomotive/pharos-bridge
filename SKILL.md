---
name: pharos-bridge
description: >
  Cross-chain bridge and developer toolkit for Pharos Network. Enables AI agents to bridge
  tokens (USDC, PROS) between Pharos and external networks (Ethereum, Base, Arbitrum, Optimism,
  Polygon, Avalanche, BSC) via InterPort Finance. Also provides full on-chain operations: balance
  queries, transactions, contract deployment, batch airdrops, and script generation. Invoke whenever
  the user mentions "bridge", "cross-chain", "Pharos", "PROS", "PHRS", "atlantic-testnet", or wants
  any on-chain operation on Pharos or cross-chain token transfer.
version: 0.1.0
requires:
  anyBins:
  - cast
  - jq
---

# Pharos Bridge & Chain Skill

Cross-chain bridge for moving tokens between Pharos and supported EVM chains, plus full developer toolkit for Pharos on-chain operations.

## Prerequisites

1. **Install Foundry** (MANDATORY):
   - Check: `which cast`
   - If not found:
   ```bash
   curl -L https://foundry.paradigm.xyz | bash
   source ~/.zshenv && foundryup
   cast --version
   ```
   - If installation fails, STOP and inform the user.

2. **Configure Private Key**: Write operations require a private key:
   - Environment variable: `$PRIVATE_KEY`
   - Or argument: `--private-key <key>`

3. **Install jq** (for JSON parsing): `which jq`

## Supported Bridge Chains

| Chain | USDC (CCTP) | PROS (CCIP) |
|-------|-------------|-------------|
| **Pharos** (1672) | yes | yes (native) |
| **Ethereum** (1) | yes | yes |
| **Base** (8453) | yes | yes |
| **Arbitrum** (42161) | yes | - |
| **Optimism** (10) | yes | - |
| **Polygon** (137) | yes | - |
| **Avalanche** (43114) | yes | - |
| **BSC** (56) | yes | - |

## Capability Index

Load the corresponding reference file based on user needs:

| User Need | Capability | Reference |
|-----------|------------|-----------|
| Bridge USDC to/from Pharos | InterPort CCTP v2 | `references/bridge-usdc.md` |
| Bridge PROS to/from Pharos | InterPort CCIP | `references/bridge-pros.md` |
| View wallet portfolio / asset overview | `cast balance` + `cast call` (batch query all known tokens) | `references/query.md#address-portfolio-wallet-asset-overview` |
| Query address balance | `cast balance` / `cast call` | `references/query.md#balance-query` |
| Query transaction status | `cast tx` / `cast receipt` | `references/query.md#transaction-query` |
| Call contract read-only method | `cast call` | `references/query.md#contract-read-only-call` |
| Send transaction (native transfer) | `cast send` | `references/transaction.md#native-token-transfer` |
| Call contract write method | `cast send` | `references/transaction.md#contract-write-call` |
| Estimate Gas | `cast estimate` | `references/transaction.md#gas-estimation` |
| Deploy contract | `forge script` (auto-generate deploy script) | `references/contract.md#deploy-contract-forge-script` |
| Verify contract | `forge verify-contract` | `references/contract.md#verify-contract` |
| One-click ERC20 deploy | `forge script` + built-in ERC20 template | `references/contract.md#erc20-one-click-deploy-built-in-template` |
| Batch transfer / Airdrop | `forge script` (auto-generate airdrop script, supports 6000+ address batched airdrop, CSV file input, three-tier auto mode: ≤10 simple mode / 11-200 single batch / >200 multi-batch, hardened Distributor contract) | `references/transaction.md#batch-transfer--airdrop` |
| Generate contract interaction scripts (read/write methods, JS/TS/Python) | Script_Generator (Agent auto-generates) | `references/script-gen.md` |

## Bridge Overview

### USDC Bridge (CCTP v2)
- **Contract**: `0x674cb5133A2dEaA4aBE86ed56CB7555960966320`
- **Method**: Circle CCTP v2 (burn-and-mint, native 1:1)
- **Speed**: Seconds to minutes (fast transfer)
- **Fee**: Paid in source chain native token

### PROS Bridge (CCIP)
- **Contract**: `0x0772C76c3e4b081E85747f248ed76CC3813d46C4`
- **Method**: Chainlink CCIP cross-chain messaging
- **Speed**: 5-20 minutes
- **Fee**: CCIP fee in source chain native token

## Bridge Flow

For every bridge request, the Agent MUST follow these steps in order:

1. **Parse request** — identify: token (USDC/PROS), amount, source chain, destination chain
2. **Validate route** — check supported chains table above
3. **Read configs** — load `assets/networks.json` and `assets/tokens.json`
4. **Pre-checks** — private key check, derive address, confirm network, check balance
5. **Approve** — ERC-20 approve for bridge contract (skip for native PROS from Pharos)
6. **Estimate fee** — estimate CCIP fee for PROS bridges
7. **Execute bridge** — call bridge contract with correct parameters
8. **Confirm** — verify transaction on source chain, wait for destination delivery

## Network Configuration

Network info is in `assets/networks.json`. Read the target chain's `rpcUrl` for `--rpc-url`:

```bash
# Example: reading network configuration
RPC=$(jq -r '.networks[] | select(.name=="atlantic-testnet") | .rpcUrl' assets/networks.json)
```

- **Default Network**: Atlantic testnet (`atlantic-testnet`). Used when the user does not specify a network.
- **Switching Networks**: When the user specifies `mainnet`, read the corresponding entry's `rpcUrl` from `assets/networks.json`.
- For bridge operations, use the source chain's RPC URL.

## Token Addresses

Token addresses are in `assets/tokens.json`. Read by chain name:

```bash
TOKEN=$(jq -r '.USDC.addresses.base' assets/tokens.json)
```

## General Error Handling

Before executing commands, the Agent should perform pre-checks; when commands fail, provide user-friendly error messages based on stderr output.

| Error Scenario | Detection | Handling |
|---------------|-----------|----------|
| Invalid address format | `invalid address` | Check format: 0x + 40 hex chars |
| Transaction hash not found | `transaction not found` | Check the hash |
| No contract code at address | Empty return value | Address has no contract code |
| Call revert | `execution reverted` | Extract and display revert reason |
| Insufficient allowance | `execution reverted` on bridge call | Re-run approve step |
| Insufficient balance | `insufficient funds` | Check token balance and gas |
| CCIP fee too low | `MessageFeeMismatch` | Re-estimate fee and retry |
| Bridge not delivering | Balance unchanged on dest | Wait for finality (CCTP: 1-5 min, CCIP: 5-20 min) |
| Unsupported route | Chain not in supported table | Inform user of supported chains |
| Private key not configured | Missing `--private-key` | Prompt user to configure private key |
| Nonce conflict | `nonce too low` | Wait or manually specify nonce |
| Missing network config | `assets/networks.json` unreadable | Config file missing or invalid format |

## Security Reminders

- **Private Key Protection**: Never expose private keys in logs, chat history, or version control. Store the private key in the `$PRIVATE_KEY` environment variable and reference it explicitly in commands via `--private-key $PRIVATE_KEY`. Note: `forge` / `cast` do not automatically read environment variables; they must be explicitly passed as command arguments.
- **Network Confirmation**: Before executing write operations, the Agent must clearly inform the user of the target network (testnet or mainnet). Mainnet operations require a prominent warning and user re-confirmation.
- **Amount Confirmation**: Always confirm the bridge amount and token with the user before executing.

## Write Operation Pre-checks (Required for All Write Operations)

For all operations requiring a private key (transfers, contract calls, deployments, airdrops, bridges, etc.), the Agent must automatically complete the following checks before execution:

### 1. Private Key Check
```bash
[ -n "$PRIVATE_KEY" ] && echo "PRIVATE_KEY is set" || echo "PRIVATE_KEY is not set"
```
If not set: prompt user to configure via `export PRIVATE_KEY=`, do NOT proceed.

### 2. Derive Public Address and Confirm with User
```bash
cast wallet address --private-key $PRIVATE_KEY
```

### 3. Network Confirmation (Must Clearly Inform User)
Read the target network info from `assets/networks.json` and display the network name and type to the user.
- If the user did not specify a network, use the default network (`atlantic-testnet`) and clearly inform the user
- If the user specified `mainnet`, prominently warn the user

### 4. Automatic Balance Check
After confirming the account and network, automatically query the balance. Check token balance on source chain and native token balance for gas fees. If either insufficient, inform user and STOP.

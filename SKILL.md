---
name: pharos-bridge
description: >
  Direct-protocol cross-chain bridge for Pharos Network — Circle CCTP V2 for native USDC
  (burn-and-mint, no aggregators, no wrapped tokens, 1:1) and Chainlink CCIP for PROS
  (direct Router calls, no wrappers). Bridges tokens between Pharos and 7 EVM chains
  (Ethereum, Base, Arbitrum, Optimism, Polygon, Avalanche, BSC) with automated full-lifecycle
  pipeline: pre-checks, approve, burn/send, attestation polling, mint/receive, unwrap — all
  from a single user confirmation. Multi-network balance queries across all chains in one
  command. Complete on-chain toolkit: transactions, contract deployment/verification, batch
  airdrops with Merkle-tree whitelist, holder analytics, and auto-generated interaction scripts
  (JS/TS/Python). Invoke on "bridge", "cross-chain", "Pharos", "PROS", "USDC", "check balances",
  or any on-chain operation.
version: 2.0.0
frameworks:
  - openclaw
  - claude-code
  - codex
tags:
  - bridge
  - cross-chain
  - cctp-v2
  - circle
  - usdc
  - ccip
  - chainlink
  - pros
  - wpros
  - burn-and-mint
  - lock-and-release
  - direct-contracts
  - no-aggregator
  - token-messenger
  - iris-api
  - attestation
  - fast-transfer
  - multi-network-balance
  - pharos
  - ethereum
  - base
  - arbitrum
  - optimism
  - polygon
  - avalanche
  - bsc
  - balance-query
  - batch-airdrop
  - contract-deployment
  - contract-verification
  - script-generation
  - foundry
  - cast
  - solidity
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

2. **Install jq** (for JSON parsing): `which jq`

## Phase 0: Setup (run once per session)

This phase runs automatically every time the skill is invoked.

### Step 0.1: Environment File Setup

**Determine project root:**

```bash
PROJECT_ROOT=$(pwd)
ENV_FILE="$PROJECT_ROOT/.env"
```

The agent MUST determine the absolute path to the project root once and reuse it for ALL subsequent commands. Store `ENV_FILE` as the absolute path to `.env`.

1. Check if `.env` exists at the project root:

```bash
[ -f "$ENV_FILE" ] && echo ".env exists at: $ENV_FILE" || echo ".env not found"
```

2. If `.env` does NOT exist, create it:

```bash
cat > "$ENV_FILE" << 'ENVEOF'
# ============================================
# Pharos Bridge — Environment Variables
# ============================================
# Fill in your values below. NEVER share this file or commit it to git.
# ============================================

# Your wallet private key (without 0x prefix)
# SECURITY: Edit this file directly. Do NOT paste your key in chat.
# How to export from MetaMask: Account Details → Show Private Key
PRIVATE_KEY=
ENVEOF
echo ".env created at: $ENV_FILE"
```

3. Ensure `.env` is in `.gitignore`:

```bash
GITIGNORE="$PROJECT_ROOT/.gitignore"
[ -f "$GITIGNORE" ] && grep -qxF '.env' "$GITIGNORE" || echo '.env' >> "$GITIGNORE"
```

**CRITICAL — Shell state does NOT persist between Bash tool calls.** Every command MUST:
- Source `.env` using the **absolute path**: `set -a && source $ENV_FILE && set +a && ...rest...`
- NEVER use relative `source .env` — it breaks if CWD changes
- The `$ENV_FILE` variable does NOT persist either — the agent MUST re-determine or hardcode the absolute path in every command

### Step 0.2: Verify Private Key

Check if PRIVATE_KEY is set. **ONLY output "set" or "not set" — never the actual value.**

```bash
set -a && source $ENV_FILE && set +a && [ -n "$PRIVATE_KEY" ] && echo "PRIVATE_KEY: set" || echo "PRIVATE_KEY: not set"
```

**FORBIDDEN COMMANDS — NEVER RUN THESE:**
```
cat .env
echo $PRIVATE_KEY
printenv PRIVATE_KEY
head .env
grep PRIVATE_KEY .env
```
Any command that would output the private key value to chat is ABSOLUTELY FORBIDDEN.

If not set, guide the user:

```
Please open the .env file and add your private key:
  File: {absolute_path_to_.env}

Set: PRIVATE_KEY=your_key_here (with or without 0x prefix)

NEVER paste keys in chat.
```

After the user confirms they filled in the key, normalize it (strip 0x prefix if present):

```bash
sed -i 's/^PRIVATE_KEY=0[xX]\(.*\)/PRIVATE_KEY=\1/' "$ENV_FILE"
```

### Step 0.3: Check Dependencies (quick check)

```bash
cast --version || $HOME/.foundry/bin/cast --version
jq --version
```

If `cast` not found → offer to install Foundry. If `jq` not found → ask user to install it.

### Security Rules (CRITICAL — enforce at all times)

1. **NEVER** output, display, echo, print, cat, grep, or log the value of PRIVATE_KEY or any secret
2. **NEVER** ask the user to paste private keys or secrets in chat
3. **NEVER** run `cat .env`, `echo $PRIVATE_KEY`, `printenv`, `grep PRIVATE_KEY`, or any command that could leak secrets to chat output
4. **ALWAYS** check .env with safe commands only: `[ -n "$PRIVATE_KEY" ] && echo "set" || echo "not set"`
5. **ALWAYS** show the full absolute path to the `.env` file so the user can find it
6. If the user accidentally pastes a secret in chat — warn immediately, suggest rotating the key

## Shell State Note (CRITICAL)

Shell state does NOT persist between Bash tool calls. EVERY command that uses env vars MUST source `.env` with the **absolute path** determined in Step 0.1:

```bash
set -a && source /absolute/path/to/.env && set +a && ...rest of command...
```

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
| Bridge USDC to/from Pharos | Circle CCTP V2 (burn-and-mint) | `references/bridge-usdc.md` |
| Bridge PROS to/from Pharos | Chainlink CCIP (direct Router) | `references/bridge-pros.md` |
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

### USDC Bridge (Circle CCTP V2)
- **TokenMessengerV2**: `0x28b5a0e9C621a5BadaA536219b3a228C8168cf5d` (all chains)
- **MessageTransmitterV2**: `0x81D40F21F12A8F0E3252Bccb954D722d4c464B64` (all chains)
- **Method**: Circle CCTP V2 burn-and-mint (native 1:1, no wrapped tokens)
- **Speed**: Fast Transfer (threshold ≤ 1000): seconds to minutes. Standard (2000): 5-20 min
- **Fee**: Fast Transfer fee ~0.0001% in USDC. Gas in native token on each chain
- **Pharos CCTP Domain**: 31 (NOT chain ID 1672)
- **Flow**: approve → depositForBurn (burn) → poll Iris API for attestation → receiveMessage (mint)

### PROS Bridge (Chainlink CCIP)
- **CCIP Router (Pharos)**: `0x4e52dD94e9BCfeFE3C78153bDfB0AB1d30687297`
- **CCIP Router (Base)**: `0x881e3A65B4d4a04dD529061dd0071cf975F58bCD`
- **CCIP Router (Ethereum)**: `0x80226fc0Ee2b096224EeAc085Bb9a8cba1146f7D`
- **Method**: Direct Chainlink CCIP (burn-and-mint / lock-and-release)
- **Speed**: 5-20 minutes
- **Fee**: CCIP fee in source chain native token (~0.257 PROS for Pharos→Base)
- **Pharos CCIP Selector**: `7801139999541420232`
- **Flow**: wrap PROS→WPROS → approve Router → ccipSend → CCIP handles transfer → unwrap WPROS→PROS
- **Routes**: Pharos↔Base, Pharos↔Ethereum (bidirectional)

## Bridge Flow

### Bridge UX (MANDATORY — read carefully)

**The bridge experience MUST be seamless and automatic.** When the user says "bridge 1 USDC from Pharos to Base" or "send 10 PROS to Ethereum", the agent:

1. Confirms parameters ONCE: amount, token, source → destination. Show clearly: "Bridging 1 USDC from Pharos to Base. Proceed?"
2. Executes the ENTIRE pipeline automatically — approve, burn/send, poll attestation (USDC only), mint/receive
3. Returns with the final result: tx hashes, explorer links, completion status

**CRITICAL RULES:**
- Do NOT ask for confirmation before each bash command. Run them sequentially.
- Do NOT show intermediate command output unless there's an error. Process it internally.
- Do NOT stop to explain each step. The user wants the result, not a tutorial.
- DO handle errors: if any step fails, stop and report the error clearly with the tx hash.
- DO show progress concisely: "Burning 1 USDC on Pharos...", "Waiting for attestation...", "Minting on Base..."
- DO return the final result with both source and destination tx hashes + explorer links.

### USDC Bridge Flow (CCTP V2)

1. **Parse request** — identify: amount, source chain, destination chain
2. **Validate route** — check supported chains table
3. **Read configs** — load all addresses from `assets/tokens.json` (cctp section)
4. **Pre-checks** (silent) — source .env, verify key, derive address, check balances
5. **Approve** — approve USDC to TokenMessengerV2 (skip if allowance sufficient)
6. **Burn** — depositForBurn with CCTP domain IDs (NOT chain IDs), bytes32 recipient, maxFee=500, minFinalityThreshold=1000
7. **Poll attestation** — Circle Iris API until status="complete" (auto-poll, up to 5 min)
8. **Mint** — receiveMessage(message, attestation) on destination chain
9. **Report** — source tx hash, destination tx hash, explorer links

### PROS Bridge Flow (CCIP)

1. **Parse request** — identify: amount, source chain, destination chain
2. **Validate route** — check supported chains (Pharos↔Base, Pharos↔Ethereum)
3. **Read configs** — load CCIP routers, selectors, token pools from `assets/tokens.json` (ccip section)
4. **Pre-checks** (silent) — source .env, verify key, derive address, check balances
5. **Wrap** — if sending FROM Pharos: deposit() PROS → WPROS
6. **Approve** — approve WPROS/PROS to CCIP Router
7. **Estimate fee** — getFee() to determine ccipSend value
8. **Send** — ccipSend with CCIP chain selector (NOT chain ID)
9. **Report** — source tx hash, CCIP message ID, link to CCIP Explorer
10. **Unwrap** (optional) — if receiving ON Pharos: withdraw() WPROS → native PROS

### CCTP Domain IDs (CRITICAL — NOT Chain IDs)

CCTP uses Circle's internal domain IDs. Read from `assets/tokens.json` → `cctp.domains`:

| Network | Chain ID | CCTP Domain |
|---------|----------|-------------|
| Pharos | 1672 | **31** |
| Ethereum | 1 | 0 |
| Avalanche | 43114 | 1 |
| OP Mainnet | 10 | 2 |
| Arbitrum | 42161 | 3 |
| Base | 8453 | 6 |
| Polygon PoS | 137 | 7 |
| BSC | 56 | 17 |

### CCIP Chain Selectors (CRITICAL — NOT Chain IDs)

CCIP uses Chainlink's internal selectors. Read from `assets/tokens.json` → `ccip.chainSelectors`:

| Network | Chain ID | CCIP Selector |
|---------|----------|---------------|
| Pharos | 1672 | **7801139999541420232** |
| Base | 8453 | **15971525489660198786** |
| Ethereum | 1 | **5009297550715157269** |

### USDC Bridge — One-shot Command Sequence

The agent should execute the full pipeline automatically without asking extra questions. Only confirm the bridge parameters (amount, token, direction) once, then execute all steps:

```bash
# 1. Config (from assets files)
TOKEN_MESSENGER=$(jq -r '.cctp.contracts.tokenMessengerV2' assets/tokens.json)
MSG_TRANSMITTER=$(jq -r '.cctp.contracts.messageTransmitterV2' assets/tokens.json)
IRIS_API=$(jq -r '.cctp.api.mainnet' assets/tokens.json)
SRC_DOMAIN=$(jq -r '.cctp.domains.pharos' assets/tokens.json)   # source domain
DEST_DOMAIN=$(jq -r '.cctp.domains.base' assets/tokens.json)    # dest domain
SRC_RPC=$(jq -r '.networks[] | select(.name=="pharos") | .rpcUrl' assets/networks.json)
DEST_RPC=$(jq -r '.networks[] | select(.name=="base") | .rpcUrl' assets/networks.json)
SRC_USDC=$(jq -r '.bridge.USDC.addresses.pharos' assets/tokens.json)

# 2. Pre-checks (silent — do not ask user)
set -a && source $ENV_FILE && set +a
ADDRESS=$(cast wallet address --private-key $PRIVATE_KEY)
# Check USDC balance + gas balance...

# 3. Approve (if needed)
cast send $SRC_USDC "approve(address,uint256)" $TOKEN_MESSENGER $AMOUNT --rpc-url $SRC_RPC --private-key $PRIVATE_KEY

# 4. Burn
MINT_RECIPIENT="0x000000000000000000000000${ADDRESS:2}"
ZERO_BYTES32="0x0000000000000000000000000000000000000000000000000000000000000000"
BURN_TX=$(cast send $TOKEN_MESSENGER \
  "depositForBurn(uint256,uint32,bytes32,address,bytes32,uint256,uint32)" \
  $AMOUNT $DEST_DOMAIN $MINT_RECIPIENT $SRC_USDC $ZERO_BYTES32 500 1000 \
  --rpc-url $SRC_RPC --private-key $PRIVATE_KEY)
TX_HASH=$(echo "$BURN_TX" | grep "transactionHash" | head -1 | awk '{print $2}')

# 5. Poll attestation (auto, up to 5 min)
for i in $(seq 1 60); do
  RESP=$(curl -s "$IRIS_API/$SRC_DOMAIN?transactionHash=$TX_HASH")
  [ "$(echo $RESP | jq -r '.messages[0].status // empty')" = "complete" ] && break
  sleep 5
done
MESSAGE=$(echo $RESP | jq -r '.messages[0].message')
ATTESTATION=$(echo $RESP | jq -r '.messages[0].attestation')

# 6. Mint on destination
cast send $MSG_TRANSMITTER "receiveMessage(bytes,bytes)" \
  "$MESSAGE" "$ATTESTATION" --rpc-url $DEST_RPC --private-key $PRIVATE_KEY
```

### Bridge UX — Agent Behavior Guide

**Example interaction:**

```
User: bridge 1 USDC from Pharos to Base

Agent: Bridging 1 USDC from Pharos → Base.

  ✅ Wallet: 0xe7e0...c63e
  ✅ USDC balance: 5.53 USDC on Pharos
  ✅ Gas: 0.99 PROS

Proceed?

User: yes

Agent: [runs all steps silently, shows progress]
  ⏳ Burning 1 USDC on Pharos...
  ✅ Burn tx: 0x49e5... (confirmed)
  ⏳ Waiting for attestation... (8s)
  ✅ Attestation received
  ⏳ Minting on Base...
  ✅ Mint tx: 0xc70f... (confirmed)

  Bridge complete!

  Burn:  https://www.pharosscan.xyz/tx/0x49e5...
  Mint:  https://basescan.org/tx/0xc70f...
  Amount: 1 USDC
```

**What the agent does NOT do:**
- ❌ "Should I approve now?" → just approve
- ❌ "Confirm burn transaction?" → just burn
- ❌ "Attestation received, proceed to mint?" → just mint
- ❌ Show raw cast output → parse and show human-readable
- ❌ Ask for network confirmation on default network → only warn on mainnet if user didn't specify

## Network Configuration

Network info is in `assets/networks.json`. Read the target chain's `rpcUrl` for `--rpc-url`:

```bash
# Example: reading network configuration
RPC=$(jq -r '.networks[] | select(.name=="pharos") | .rpcUrl' assets/networks.json)
CCTP_DOMAIN=$(jq -r '.networks[] | select(.name=="pharos") | .cctpDomain' assets/networks.json)
```

- **Default Network**: Pharos Pacific Mainnet (`pharos`, chain ID 1672). Used when the user does not specify a network.
- **Testnet**: Only use Atlantic testnet (`atlantic-testnet`) when the user explicitly mentions "testnet", "atlantic", or "test tokens".
- **Switching Networks**: When the user specifies a network by name, read the corresponding entry's `rpcUrl` and `cctpDomain` from `assets/networks.json`.
- For bridge operations, use the source chain's RPC URL for burn, destination chain's RPC URL for mint.

## Token Addresses

Token addresses are in `assets/tokens.json`. Read by chain name:

```bash
TOKEN=$(jq -r '.bridge.USDC.addresses.base' assets/tokens.json)
```

### USDC Addresses Per Chain (Balance Queries)

For balance queries across chains, USDC addresses are in `bridge.USDC.addresses.<chain>` — **NOT** in the `mainnet` or `atlantic-testnet` top-level sections (those only contain Pharos-local tokens).

```bash
# USDC on any chain — use bridge section
jq -r '.bridge.USDC.addresses.pharos' assets/tokens.json    # 0xc879c018...
jq -r '.bridge.USDC.addresses.base' assets/tokens.json      # 0x833589fC...
jq -r '.bridge.USDC.addresses.ethereum' assets/tokens.json  # 0xA0b86991...
jq -r '.bridge.USDC.addresses.arbitrum' assets/tokens.json
jq -r '.bridge.USDC.addresses.optimism' assets/tokens.json
jq -r '.bridge.USDC.addresses.polygon' assets/tokens.json
jq -r '.bridge.USDC.addresses.avalanche' assets/tokens.json
jq -r '.bridge.USDC.addresses.bsc' assets/tokens.json

# PROS/WPROS — use ccip section
jq -r '.ccip.tokens.pharos' assets/tokens.json    # WPROS on Pharos
jq -r '.ccip.tokens.base' assets/tokens.json      # PROS on Base
jq -r '.ccip.tokens.ethereum' assets/tokens.json  # PROS on Ethereum
```

CCTP configuration is in `assets/tokens.json` → `cctp`:

```bash
TOKEN_MESSENGER=$(jq -r '.cctp.contracts.tokenMessengerV2' assets/tokens.json)
MSG_TRANSMITTER=$(jq -r '.cctp.contracts.messageTransmitterV2' assets/tokens.json)
IRIS_API=$(jq -r '.cctp.api.mainnet' assets/tokens.json)
DEST_DOMAIN=$(jq -r '.cctp.domains.base' assets/tokens.json)
```

CCIP configuration is in `assets/tokens.json` → `ccip`:

```bash
CCIP_ROUTER=$(jq -r '.ccip.routers.pharos' assets/tokens.json)
DEST_SELECTOR=$(jq -r '.ccip.chainSelectors.base' assets/tokens.json)
WPROS=$(jq -r '.ccip.tokens.pharos' assets/tokens.json)
```

## Balance Check (Single Command, All Networks)

When the user asks "check balances", "show balances", "check all balances" — run ONE bash command. NO separate commands per network. NO extra confirmations.

### Rules

| User Request | Networks to Check |
|--------------|-------------------|
| "check all balances" / "check balances on all networks" / "проверь все балансы" | ALL 8 networks |
| "check balances on Pharos, Base and Ethereum" | Only those 3 |
| "check balance on Pharos" | Only 1 network |

- **ALWAYS one bash command** regardless of network count — zero extra confirmations
- **Read-only queries** are safe — run without asking for user approval per step
- Adapt the `for NET in ...` line to match user request

### All Networks Script

Run as ONE bash tool call. The `for NET in ...` line adapts to what the user asked:

```bash
set -a && source $ENV_FILE && set +a

SKILL_DIR="$(pwd)/skills/pharos-bridge"
ADDRESS=$(cast wallet address --private-key $PRIVATE_KEY)

echo "Wallet: $ADDRESS"
echo ""
printf "%-16s %-20s %s\n" "Network" "Native" "USDC"
printf "%-16s %-20s %s\n" "--------" "------" "----"

for NET in pharos base ethereum arbitrum optimism polygon avalanche bsc; do
  RPC=$(jq -r ".networks[] | select(.name==\"$NET\") | .rpcUrl" $SKILL_DIR/assets/networks.json 2>/dev/null)
  SYM=$(jq -r ".networks[] | select(.name==\"$NET\") | .nativeToken" $SKILL_DIR/assets/networks.json 2>/dev/null)
  CID=$(jq -r ".networks[] | select(.name==\"$NET\") | .chainId" $SKILL_DIR/assets/networks.json 2>/dev/null)
  [ -z "$RPC" ] || [ "$RPC" = "null" ] && continue
  NAT=$(cast balance $ADDRESS --rpc-url $RPC --ether 2>/dev/null || echo "error")
  USDC_A=$(jq -r ".bridge.USDC.addresses.$NET // empty" $SKILL_DIR/assets/tokens.json 2>/dev/null)
  if [ -n "$USDC_A" ]; then
    RAW=$(cast call $USDC_A "balanceOf(address)(uint256)" $ADDRESS --rpc-url $RPC 2>&1)
    if echo "$RAW" | grep -qE '^[0-9]+'; then
      AMT=$(echo "$RAW" | grep -oE '^[0-9]+')
      USDC=$(echo "$AMT" | awk '{printf "%.2f", $1/1000000}')
    else
      RAW2=$(sleep 1 && cast call $USDC_A "balanceOf(address)(uint256)" $ADDRESS --rpc-url $RPC 2>&1)
      if echo "$RAW2" | grep -qE '^[0-9]+'; then
        AMT=$(echo "$RAW2" | grep -oE '^[0-9]+')
        USDC=$(echo "$AMT" | awk '{printf "%.2f", $1/1000000}')
      else
        USDC="rpc_err"
      fi
    fi
  else
    USDC="N/A"
  fi
  printf "%-16s %-20s %s\n" "$NET ($CID)" "$NAT $SYM" "$USDC"
done
```

### Token Address Rules

- **USDC on any chain**: `bridge.USDC.addresses.<chain>` in `assets/tokens.json` — NOT in `mainnet` section
- **PROS/WPROS**: `ccip.tokens.<chain>` in `assets/tokens.json`
- **Native balance**: `cast balance` with chain's RPC from `assets/networks.json`
- **ALWAYS pass `--rpc-url`** — never rely on defaults
- **Retry on failure**: if first `cast call` fails, wait 1s and retry once. If still fails, show `rpc_err` (not silent 0)

## General Error Handling

Before executing commands, the Agent should perform pre-checks; when commands fail, provide user-friendly error messages based on stderr output.

| Error Scenario | Detection | Handling |
|---------------|-----------|----------|
| Invalid address format | `invalid address` | Check format: 0x + 40 hex chars |
| Transaction hash not found | `transaction not found` | Check the hash |
| No contract code at address | Empty return value | Address has no contract code |
| Call revert | `execution reverted` | Extract and display revert reason |
| Insufficient allowance | `execution reverted` on depositForBurn | Re-run approve step for TokenMessengerV2 |
| Insufficient balance | `insufficient funds` | Check USDC balance and native token for gas |
| CCIP fee too low | `MessageFeeMismatch` | Re-estimate fee and retry |
| Attestation never completes | Poll timeout after 5 min | Verify burn tx on explorer; may be invalid or network issue |
| receiveMessage reverts | `execution reverted` | Attestation already used or invalid; check dest balance first |
| Wrong domain ID | `execution reverted` on depositForBurn | Verify using cctpDomain from config, NOT chain ID |
| Bridge not delivering | Balance unchanged on dest | CCTP: re-check attestation status. CCIP: wait 5-20 min |
| Unsupported route | Chain not in supported table | Inform user of supported chains |
| Private key not configured | Missing `--private-key` | Prompt user to configure private key |
| Nonce conflict | `nonce too low` | Wait or manually specify nonce |
| Missing network config | `assets/networks.json` unreadable | Config file missing or invalid format |

## Security Reminders

- **Private Key Protection**: The private key is stored ONLY in the `.env` file. It is NEVER exposed in chat, logs, or version control. All `cast`/`forge` commands reference it via `--private-key $PRIVATE_KEY` after sourcing `.env`.
- **One Confirmation Rule**: For bridge operations, confirm parameters ONCE (amount, token, source → destination) then execute the entire pipeline. Do NOT ask for confirmation at each step.
- **Mainnet Warning**: If the user explicitly says "mainnet" or the operation is clearly on mainnet, show a one-line warning. If the user specified the network themselves, skip the warning — they know what they're doing.
- **CCTP Domain IDs**: Always use CCTP Domain IDs (from `cctp.domains`), NOT EVM Chain IDs. Using chain IDs will cause failed transactions.
- **CCIP Chain Selectors**: Always use CCIP Chain Selectors (from `ccip.chainSelectors`), NOT EVM Chain IDs.

## Write Operation Pre-checks

For bridge operations, pre-checks run SILENTLY — the user only sees the result summary, not the individual checks:

### Silent Pre-checks (auto, no user interaction)
```bash
# 1. Source environment
set -a && source $ENV_FILE && set +a

# 2. Verify private key (output "set"/"not set" only)
[ -n "$PRIVATE_KEY" ] && echo "set" || echo "not set"

# 3. Derive address
ADDRESS=$(cast wallet address --private-key $PRIVATE_KEY)

# 4. Check balances (token + native gas)
# If insufficient → STOP and tell user. Otherwise proceed silently.
```

### For Non-Bridge Write Operations (transfers, deployments, etc.)
Show the user: target network, address, operation. Get confirmation before executing.

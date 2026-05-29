# PROS Bridge via Chainlink CCIP

Bridge PROS/WPROS token between Pharos, Base, and Ethereum using direct Chainlink CCIP Router.

## CCIP Architecture

Chainlink CCIP enables cross-chain token transfers via burn-and-mint / lock-and-release token pools:

- **Pharos → Base/Ethereum**: WPROS locked on Pharos → PROS minted (BurnMintERC20) on destination
- **Base/Ethereum → Pharos**: PROS burned on source → WPROS released on Pharos

No InterPort wrapper — direct CCIP Router calls. Fully documented by [Chainlink](https://docs.chain.link/ccip/directory/mainnet/token/WPROS) and [Pharos](https://docs.pharos.xyz/tooling-and-infrastructure/cross-chain/chainlink-ccip).

## Supported Routes

| From | To | Method |
|------|----|--------|
| Pharos | Base | CCIP (Lock/Release → Burn/Mint) |
| Pharos | Ethereum | CCIP (Lock/Release → Burn/Mint) |
| Base | Pharos | CCIP (Burn/Mint → Lock/Release) |
| Ethereum | Pharos | CCIP (Burn/Mint → Lock/Release) |

## CCIP Configuration

### CCIP Routers

| Network | Router Address |
|---------|---------------|
| **Pharos** | `0x4e52dD94e9BCfeFE3C78153bDfB0AB1d30687297` |
| **Base** | `0x881e3A65B4d4a04dD529061dd0071cf975F58bCD` |
| **Ethereum** | `0x80226fc0Ee2b096224EeAc085Bb9a8cba1146f7D` |

### CCIP Chain Selectors (NOT Chain IDs)

| Network | Chain ID | CCIP Selector |
|---------|----------|---------------|
| Pharos | 1672 | `7801139999541420232` |
| Base | 8453 | `15971525489660198786` |
| Ethereum | 1 | `5009297550715157269` |

### Token Addresses

| Network | Token | Address | Note |
|---------|-------|---------|------|
| Pharos | WPROS | `0x52c48d4213107b20bc583832b0d951fb9ca8f0b0` | Wrapped PROS (WETH-like) |
| Pharos | PROS | native | Native gas token |
| Base | PROS | `0x8B7DdE054BE9D180c1Be7FaE0874697374A49832` | BurnMintERC20 |
| Ethereum | PROS | `0xB197E02499e6502733C6bCE2eb39013C39A03147` | BurnMintERC20 |

### Token Pools (verified on-chain)

| Network | TokenPool Address | Type |
|---------|-------------------|------|
| Pharos | `0xCb79097744d5266bFca287A43612D9613Be46300` | Lock/Release |
| Base | `0x7126C3FeF4e6a680eeE09Fb039B2236F638384B0` | Burn/Mint |
| Ethereum | `0x6f0c4C3f0F1ca8f0513b542A124a0208fEe72D97` | Burn/Mint |

All config is in `assets/tokens.json` → `ccip` section.

## Step 1: Read Configuration

```bash
PROJECT_ROOT=$(pwd)
TOKENS="$PROJECT_ROOT/skills/pharos-bridge/assets/tokens.json"
NETWORKS="$PROJECT_ROOT/skills/pharos-bridge/assets/networks.json"

# CCIP config
ROUTER=$(jq -r '.ccip.routers.pharos' $TOKENS)
DEST_SELECTOR=$(jq -r '.ccip.chainSelectors.base' $TOKENS)
WPROS=$(jq -r '.ccip.tokens.pharos' $TOKENS)
PHAROS_RPC=$(jq -r '.networks[] | select(.name=="pharos") | .rpcUrl' $NETWORKS)
BASE_RPC=$(jq -r '.networks[] | select(.name=="base") | .rpcUrl' $NETWORKS)
```

## Step 2: Pre-flight Checks

```bash
set -a && source /absolute/path/to/.env && set +a
[ -n "$PRIVATE_KEY" ] && echo "PRIVATE_KEY: set" || { echo "PRIVATE_KEY: not set"; exit 1; }
ADDRESS=$(cast wallet address --private-key $PRIVATE_KEY)

# Check native PROS balance on Pharos
cast balance $ADDRESS --rpc-url $PHAROS_RPC

# Check WPROS balance
cast call $WPROS "balanceOf(address)(uint256)" $ADDRESS --rpc-url $PHAROS_RPC
```

## Step 3: Wrap PROS → WPROS (only when bridging FROM Pharos)

WPROS follows the WETH pattern: `deposit()` is payable, `withdraw(uint256)` returns native.

```bash
WPROS="0x52c48d4213107b20bc583832b0d951fb9ca8f0b0"
AMOUNT=10000000000000000000  # 10 PROS (18 decimals)

# Wrap native PROS → WPROS
cast send $WPROS "deposit()" \
  --value $AMOUNT \
  --rpc-url $PHAROS_RPC \
  --private-key $PRIVATE_KEY
```

## Step 4: Approve WPROS/PROS to CCIP Router

```bash
# On Pharos (approve WPROS to Router)
ROUTER_PHAROS="0x4e52dD94e9BCfeFE3C78153bDfB0AB1d30687297"
cast send $WPROS "approve(address,uint256)" $ROUTER_PHAROS $AMOUNT \
  --rpc-url $PHAROS_RPC --private-key $PRIVATE_KEY

# On Base (approve PROS to Router)
ROUTER_BASE="0x881e3A65B4d4a04dD529061dd0071cf975F58bCD"
PROS_BASE="0x8B7DdE054BE9D180c1Be7FaE0874697374A49832"
cast send $PROS_BASE "approve(address,uint256)" $ROUTER_BASE $AMOUNT \
  --rpc-url $BASE_RPC --private-key $PRIVATE_KEY

# On Ethereum (approve PROS to Router)
ROUTER_ETH="0x80226fc0Ee2b096224EeAc085Bb9a8cba1146f7D"
PROS_ETH="0xB197E02499e6502733C6bCE2eb39013C39A03147"
cast send $PROS_ETH "approve(address,uint256)" $ROUTER_ETH $AMOUNT \
  --rpc-url $ETH_RPC --private-key $PRIVATE_KEY
```

## Step 5: Estimate CCIP Fee

```bash
# Build EVM2AnyMessage and call getFee
# Receiver as bytes32 (left-padded address)
RECEIVER="0x000000000000000000000000${ADDRESS:2}"
EXTRA_ARGS="0x"  # or EVMExtraArgsV2 for gas limit

# getFee(uint64 destinationChainSelector, EVM2AnyMessage message)
cast call $ROUTER \
  "getFee(uint64,(bytes,bytes,(address,uint256)[],address,bytes))" \
  $DEST_SELECTOR \
  "($RECEIVER,0x,[($WPROS,$AMOUNT)],0x0000000000000000000000000000000000000000,$EXTRA_ARGS)" \
  --rpc-url $PHAROS_RPC
```

Fee is returned in wei of native token. For Pharos→Base: ~0.257 PROS (confirmed on-chain).

## Step 6: Execute Bridge (ccipSend)

### Pharos → Base

```bash
ROUTER_PHAROS="0x4e52dD94e9BCfeFE3C78153bDfB0AB1d30687297"
WPROS="0x52c48d4213107b20bc583832b0d951fb9ca8f0b0"
BASE_SELECTOR=15971525489660198786
PHAROS_RPC="https://rpc.pharos.xyz"
AMOUNT=10000000000000000000  # 10 WPROS

# 1. Wrap PROS → WPROS
cast send $WPROS "deposit()" --value $AMOUNT --rpc-url $PHAROS_RPC --private-key $PRIVATE_KEY

# 2. Approve
cast send $WPROS "approve(address,uint256)" $ROUTER_PHAROS $AMOUNT --rpc-url $PHAROS_RPC --private-key $PRIVATE_KEY

# 3. Send via CCIP (fee paid in native PROS)
cast send $ROUTER_PHAROS \
  "ccipSend(uint64,(bytes,bytes,(address,uint256)[],address,bytes))" \
  $BASE_SELECTOR \
  "($RECEIVER,0x,[($WPROS,$AMOUNT)],0x0000000000000000000000000000000000000000,0x)" \
  --value 300000000000000000 \
  --rpc-url $PHAROS_RPC \
  --private-key $PRIVATE_KEY
```

### Pharos → Ethereum

```bash
ROUTER_PHAROS="0x4e52dD94e9BCfeFE3C78153bDfB0AB1d30687297"
WPROS="0x52c48d4213107b20bc583832b0d951fb9ca8f0b0"
ETH_SELECTOR=5009297550715157269

# Same flow as Pharos → Base, but use ETH_SELECTOR and higher fee
cast send $ROUTER_PHAROS \
  "ccipSend(uint64,(bytes,bytes,(address,uint256)[],address,bytes))" \
  $ETH_SELECTOR \
  "($RECEIVER,0x,[($WPROS,$AMOUNT)],0x0000000000000000000000000000000000000000,0x)" \
  --value 300000000000000000 \
  --rpc-url $PHAROS_RPC \
  --private-key $PRIVATE_KEY
```

### Base → Pharos

```bash
ROUTER_BASE="0x881e3A65B4d4a04dD529061dd0071cf975F58bCD"
PROS_BASE="0x8B7DdE054BE9D180c1Be7FaE0874697374A49832"
PHAROS_SELECTOR=7801139999541420232
BASE_RPC="https://mainnet.base.org"

# Approve PROS to Router
cast send $PROS_BASE "approve(address,uint256)" $ROUTER_BASE $AMOUNT --rpc-url $BASE_RPC --private-key $PRIVATE_KEY

# Send via CCIP (fee paid in native ETH)
cast send $ROUTER_BASE \
  "ccipSend(uint64,(bytes,bytes,(address,uint256)[],address,bytes))" \
  $PHAROS_SELECTOR \
  "($RECEIVER,0x,[($PROS_BASE,$AMOUNT)],0x0000000000000000000000000000000000000000,0x)" \
  --value 500000000000000 \
  --rpc-url $BASE_RPC \
  --private-key $PRIVATE_KEY
```

### Ethereum → Pharos

```bash
ROUTER_ETH="0x80226fc0Ee2b096224EeAc085Bb9a8cba1146f7D"
PROS_ETH="0xB197E02499e6502733C6bCE2eb39013C39A03147"
PHAROS_SELECTOR=7801139999541420232
ETH_RPC="https://ethereum.publicnode.com"

# Approve PROS to Router
cast send $PROS_ETH "approve(address,uint256)" $ROUTER_ETH $AMOUNT --rpc-url $ETH_RPC --private-key $PRIVATE_KEY

# Send via CCIP (fee paid in native ETH)
cast send $ROUTER_ETH \
  "ccipSend(uint64,(bytes,bytes,(address,uint256)[],address,bytes))" \
  $PHAROS_SELECTOR \
  "($RECEIVER,0x,[($PROS_ETH,$AMOUNT)],0x0000000000000000000000000000000000000000,0x)" \
  --value 2000000000000000 \
  --rpc-url $ETH_RPC \
  --private-key $PRIVATE_KEY
```

## Step 7: Unwrap WPROS → PROS (when receiving on Pharos)

When PROS is bridged TO Pharos, you receive WPROS. Unwrap to get native PROS:

```bash
WPROS="0x52c48d4213107b20bc583832b0d951fb9ca8f0b0"
# Check WPROS balance after bridge arrives
WPROS_BALANCE=$(cast call $WPROS "balanceOf(address)(uint256)" $ADDRESS --rpc-url $PHAROS_RPC)

# Unwrap WPROS → native PROS
cast send $WPROS "withdraw(uint256)" $WPROS_BALANCE \
  --rpc-url $PHAROS_RPC --private-key $PRIVATE_KEY
```

## Step 8: Verify

CCIP finality: ~5-20 minutes.

```bash
# Check PROS on Base
cast call $PROS_BASE "balanceOf(address)(uint256)" $ADDRESS --rpc-url $BASE_RPC

# Check PROS on Ethereum
cast call $PROS_ETH "balanceOf(address)(uint256)" $ADDRESS --rpc-url $ETH_RPC

# Check native PROS on Pharos
cast balance $ADDRESS --rpc-url $PHAROS_RPC
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `execution reverted` on approve | Already approved | Check allowance: `cast call $WPROS "allowance(address,address)(uint256)" $ADDRESS $ROUTER` |
| `execution reverted` on ccipSend | Insufficient WPROS or fee too low | Check WPROS balance, increase `--value` for CCIP fee |
| `insufficient funds` | Not enough native for gas + CCIP fee | Ensure native balance covers gas + fee |
| CCIP message not delivered | Waiting for finality | Wait 5-20 minutes, check on [CCIP Explorer](https://ccip.chain.link) |
| Wrong chain selector | Used chain ID instead of CCIP selector | Use CCIP selectors from config (e.g., 15971525489660198786 for Base, NOT 8453) |
| WPROS balance 0 on Pharos | Forgot to wrap native PROS | Run `deposit()` on WPROS contract with `--value` |

## Important Notes

- CCIP Chain Selectors are NOT the same as EVM Chain IDs. Always use selectors from `assets/tokens.json` → `ccip.chainSelectors`
- On Pharos, CCIP works with **WPROS** (wrapped), not native PROS. Wrap before sending, unwrap after receiving
- WPROS follows WETH pattern: `deposit()` (payable) to wrap, `withdraw(uint256)` to unwrap
- CCIP fees are paid in source chain's native token (PROS on Pharos, ETH on Base/Ethereum)
- Fee for 1 WPROS Pharos → Base: ~0.257 PROS (confirmed on-chain)
- CCIP finality: 5-20 minutes (vs CCTP V2: seconds to minutes)
- Track CCIP messages on [CCIP Explorer](https://ccip.chain.link) using the source tx hash

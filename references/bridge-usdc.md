# USDC Bridge via CCTP v2

Bridge USDC between Pharos and supported chains using InterPort Finance CCTP v2 bridge.

## Supported Routes

| From | To | Method |
|------|----|--------|
| Base | Pharos | CCTP v2 |
| Ethereum | Pharos | CCTP v2 |
| Arbitrum | Pharos | CCTP v2 |
| Optimism | Pharos | CCTP v2 |
| Polygon | Pharos | CCTP v2 |
| Avalanche | Pharos | CCTP v2 |
| BSC | Pharos | CCTP v2 |
| Pharos | Base | CCTP v2 |
| Pharos | Ethereum | CCTP v2 |
| Pharos | Arbitrum | CCTP v2 |
| Pharos | Optimism | CCTP v2 |
| Pharos | Polygon | CCTP v2 |
| Pharos | Avalanche | CCTP v2 |
| Pharos | BSC | CCTP v2 |

## Bridge Contract

- **CCTP V2 Bridge**: `0x674cb5133A2dEaA4aBE86ed56CB7555960966320`
- Same address on all chains (deterministic deployment)

## Step 1: Read Network Configuration

Read `assets/networks.json` to get the source chain RPC URL:

```bash
# Example: get Base RPC
SOURCE_RPC=$(jq -r '.networks[] | select(.name=="base") | .rpcUrl' assets/networks.json)
DEST_RPC=$(jq -r '.networks[] | select(.name=="pharos") | .rpcUrl' assets/networks.json)
```

## Step 2: Read Token Addresses

Read `assets/tokens.json` to get USDC addresses on source and destination chains:

```bash
# Example: USDC on Base
SOURCE_USDC=$(jq -r '.USDC.addresses.base' assets/tokens.json)
DEST_USDC=$(jq -r '.USDC.addresses.pharos' assets/tokens.json)
```

## Step 3: Check USDC Balance

Verify the user has enough USDC on the source chain:

```bash
cast call $SOURCE_USDC \
  "balanceOf(address)(uint256)" \
  $WALLET_ADDRESS \
  --rpc-url $SOURCE_RPC
```

Convert to human-readable (USDC has 6 decimals):

```bash
cast --to-unit <raw_balance> ether
# Or divide by 10^6 manually
```

## Step 4: Approve USDC Spending

The bridge contract must be approved to spend USDC. Call `approve` on the USDC contract:

```bash
BRIDGE="0x674cb5133A2dEaA4aBE86ed56CB7555960966320"
AMOUNT=$(cast --to-unit 200 ether)  # 200 USDC = 200000000 (6 decimals)

cast send $SOURCE_USDC \
  "approve(address,uint256)" \
  $BRIDGE \
  $AMOUNT \
  --private-key $PRIVATE_KEY \
  --rpc-url $SOURCE_RPC
```

Wait for approval confirmation before proceeding.

## Step 5: Execute Bridge

Call the CCTP V2 bridge contract. The bridge function signature:

```
bridgeTokens((uint256,bytes,bytes,(address,uint256)[]), (address,uint256,uint256))
```

### Bridge Parameters

- `targetChainId`: destination chain ID (from `assets/networks.json`)
- `targetRecipient`: ABI-encoded recipient address: `0x<WALLET_ADDRESS>`
- `tokenAmounts`: array of `{token, amount}` pairs

### Example: Bridge 200 USDC from Base to Pharos

```bash
BRIDGE="0x674cb5133A2dEaA4aBE86ed56CB7555960966320"
SOURCE_RPC="https://mainnet.base.org"
DEST_CHAIN_ID=1672
AMOUNT=200000000  # 200 USDC (6 decimals)
RECIPIENT=$(cast abi-encode "address(address)" $WALLET_ADDRESS)

cast send $BRIDGE \
  "bridgeTokens((uint256,bytes,(address,uint256)[]),(address,uint256,uint256))" \
  "($DEST_CHAIN_ID,$RECIPIENT,($SOURCE_USDC,$AMOUNT))" \
  "(address(0),0,0)" \
  --private-key $PRIVATE_KEY \
  --rpc-url $SOURCE_RPC
```

### Example: Bridge 500 USDC from Pharos to Ethereum

```bash
BRIDGE="0x674cb5133A2dEaA4aBE86ed56CB7555960966320"
PHAROS_RPC="https://rpc.pharosnetwork.xyz"
DEST_CHAIN_ID=1
AMOUNT=500000000  # 500 USDC (6 decimals)
PHAROS_USDC="0xc879c018db60520f4355c26ed1a6d572cdac1815"
ETHEREUM_USDC="0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
RECIPIENT=$(cast abi-encode "address(address)" $WALLET_ADDRESS)

cast send $BRIDGE \
  "bridgeTokens((uint256,bytes,(address,uint256)[]),(address,uint256,uint256))" \
  "($DEST_CHAIN_ID,$RECIPIENT,($PHAROS_USDC,$AMOUNT))" \
  "(address(0),0,0)" \
  --private-key $PRIVATE_KEY \
  --rpc-url $PHAROS_RPC
```

## Step 6: Verify Bridge Completion

After the transaction is confirmed on the source chain, the bridge typically takes 1-5 minutes (CCTP v2 fast transfer) to complete on the destination chain.

Check the destination chain for the received USDC:

```bash
cast call $DEST_USDC \
  "balanceOf(address)(uint256)" \
  $WALLET_ADDRESS \
  --rpc-url $DEST_RPC
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `execution reverted` | Insufficient allowance | Re-run approve step with correct amount |
| `insufficient funds` | Not enough USDC or gas | Check USDC balance and native token for gas |
| `InvalidAmount` | Amount is 0 or exceeds balance | Verify amount and balance before bridging |
| Bridge not completing | CCTP attestation pending | Wait 1-5 minutes, check destination balance |

## Notes

- USDC is bridged natively 1:1 via Circle CCTP v2 — no wrapped tokens
- Bridge fee is paid in native token (ETH on Base/Ethereum, PROS on Pharos)
- Ensure sufficient native token for gas on both source chain
- CCTP v2 fast transfers typically complete in seconds to minutes

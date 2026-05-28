# PROS Bridge via Chainlink CCIP

Bridge PROS token between Pharos, Base, and Ethereum using InterPort Finance CCIP Token Bridge V2.

## Supported Routes

| From | To | Method |
|------|----|--------|
| Base | Pharos | CCIP |
| Ethereum | Pharos | CCIP |
| Pharos | Base | CCIP |
| Pharos | Ethereum | CCIP |

## Bridge Contract

- **CCIP Token Bridge V2**: `0x0772C76c3e4b081E85747f248ed76CC3813d46C4`
- Same address on all chains (deterministic deployment)

## PROS Token Addresses

| Chain | Address | Note |
|-------|---------|------|
| Pharos | native | PROS is the native gas token |
| Base | `0x8B7DdE054BE9D180c1Be7FaE0874697374A49832` | ERC-20 |
| Ethereum | `0xB197E02499e6502733C6bCE2eb39013C39A03147` | ERC-20 |

## Step 1: Read Network Configuration

Read `assets/networks.json` to get source and destination chain RPC URLs:

```bash
SOURCE_RPC=$(jq -r '.networks[] | select(.name=="base") | .rpcUrl' assets/networks.json)
DEST_RPC=$(jq -r '.networks[] | select(.name=="pharos") | .rpcUrl' assets/networks.json)
```

## Step 2: Read Token Addresses

Read `assets/tokens.json`:

```bash
SOURCE_PROS=$(jq -r '.PROS.addresses.base' assets/tokens.json)
DEST_PROS=$(jq -r '.PROS.addresses.pharos' assets/tokens.json)
```

## Step 3: Check PROS Balance

### On Base or Ethereum (ERC-20 PROS)

```bash
cast call $SOURCE_PROS \
  "balanceOf(address)(uint256)" \
  $WALLET_ADDRESS \
  --rpc-url $SOURCE_RPC
```

### On Pharos (native PROS)

```bash
cast balance $WALLET_ADDRESS --rpc-url $PHAROS_RPC --ether
```

## Step 4: Approve PROS Spending (ERC-20 only)

Required when bridging FROM Base or Ethereum. Not needed when bridging FROM Pharos (native token).

```bash
BRIDGE="0x0772C76c3e4b081E85747f248ed76CC3813d46C4"
AMOUNT=$(cast --to-unit 10 ether)  # 10 PROS = 10000000000000000000 (18 decimals)

cast send $SOURCE_PROS \
  "approve(address,uint256)" \
  $BRIDGE \
  $AMOUNT \
  --private-key $PRIVATE_KEY \
  --rpc-url $SOURCE_RPC
```

## Step 5: Estimate CCIP Fees

Before bridging, estimate the CCIP messaging fee:

```bash
BRIDGE="0x0772C76c3e4b081E85747f248ed76CC3813d46C4"

# Estimate fee
cast call $BRIDGE \
  "messageFee((uint256,uint64,bytes,(address,uint256)[],bytes),(address,uint256,uint256))" \
  --rpc-url $SOURCE_RPC
```

The fee is paid in the native token of the source chain (ETH on Base/Ethereum, PROS on Pharos).

## Step 6: Execute Bridge

### Bridge PROS from Base to Pharos

```bash
BRIDGE="0x0772C76c3e4b081E85747f248ed76CC3813d46C4"
BASE_RPC="https://mainnet.base.org"
BASE_PROS="0x8B7DdE054BE9D180c1Be7FaE0874697374A49832"
DEST_CHAIN_ID=1672
AMOUNT=10000000000000000000  # 10 PROS (18 decimals)
RECIPIENT=$(cast abi-encode "address(address)" $WALLET_ADDRESS)
CCIP_FEE=0  # Replace with estimated fee from Step 5

cast send $BRIDGE \
  "bridgeTokens((uint256,uint64,bytes,(address,uint256)[],bytes),(address,uint256,uint256))" \
  "($DEST_CHAIN_ID,0,$RECIPIENT,($BASE_PROS,$AMOUNT),0x)" \
  "(address(0),0,$CCIP_FEE)" \
  --value ${CCIP_FEE}wei \
  --private-key $PRIVATE_KEY \
  --rpc-url $BASE_RPC
```

### Bridge PROS from Pharos to Base

When bridging native PROS FROM Pharos, send the PROS amount as `msg.value`:

```bash
BRIDGE="0x0772C76c3e4b081E85747f248ed76CC3813d46C4"
PHAROS_RPC="https://rpc.pharosnetwork.xyz"
DEST_CHAIN_ID=8453
AMOUNT=10000000000000000000  # 10 PROS (18 decimals)
RECIPIENT=$(cast abi-encode "address(address)" $WALLET_ADDRESS)
BASE_PROS="0x8B7DdE054BE9D180c1Be7FaE0874697374A49832"
CCIP_FEE=0  # Replace with estimated fee

# Wrap native PROS + bridge
cast send $BRIDGE \
  "bridgeTokens((uint256,uint64,bytes,(address,uint256)[],bytes),(address,uint256,uint256))" \
  "($DEST_CHAIN_ID,0,$RECIPIENT,(address(0),$AMOUNT),0x)" \
  "(address(0),0,$CCIP_FEE)" \
  --value $(($(cast --to-unit 10 ether) + CCIP_FEE))wei \
  --private-key $PRIVATE_KEY \
  --rpc-url $PHAROS_RPC
```

### Bridge PROS from Ethereum to Pharos

```bash
BRIDGE="0x0772C76c3e4b081E85747f248ed76CC3813d46C4"
ETH_RPC="https://eth.llamarpc.com"
ETH_PROS="0xB197E02499e6502733C6bCE2eb39013C39A03147"
DEST_CHAIN_ID=1672
AMOUNT=10000000000000000000  # 10 PROS (18 decimals)
RECIPIENT=$(cast abi-encode "address(address)" $WALLET_ADDRESS)
CCIP_FEE=0  # Replace with estimated fee

cast send $BRIDGE \
  "bridgeTokens((uint256,uint64,bytes,(address,uint256)[],bytes),(address,uint256,uint256))" \
  "($DEST_CHAIN_ID,0,$RECIPIENT,($ETH_PROS,$AMOUNT),0x)" \
  "(address(0),0,$CCIP_FEE)" \
  --value ${CCIP_FEE}wei \
  --private-key $PRIVATE_KEY \
  --rpc-url $ETH_RPC
```

## Step 7: Verify Bridge Completion

CCIP bridges typically take 5-20 minutes depending on Chainlink finality.

### On destination chain (Base/Ethereum):

```bash
cast call $DEST_PROS \
  "balanceOf(address)(uint256)" \
  $WALLET_ADDRESS \
  --rpc-url $DEST_RPC
```

### On destination chain (Pharos - native):

```bash
cast balance $WALLET_ADDRESS --rpc-url $PHAROS_RPC --ether
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| `execution reverted` | Insufficient allowance | Re-run approve step |
| `insufficient funds` | Not enough PROS or gas for CCIP fee | Check balance and CCIP fee estimate |
| `MessageIdNotFound` | CCIP message not yet delivered | Wait for CCIP finality (5-20 min) |
| `InvalidAmount` | Amount is 0 or exceeds balance | Verify amount before bridging |

## Notes

- CCIP messaging fees vary and must be estimated before each bridge call
- Fees are paid in the source chain's native token
- CCIP bridge finality: ~5-20 minutes (vs CCTP v2: seconds to minutes)
- When bridging native PROS from Pharos, send PROS as `msg.value` with the transaction

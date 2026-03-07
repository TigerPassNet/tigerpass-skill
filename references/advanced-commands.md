# Advanced Commands — Smart Contracts, Signing & x402

> Moved from SKILL.md for brevity. For quick command lookup, see the summary table in SKILL.md.

## Smart Contract Execution & On-Chain Reads

For protocols without a built-in command (AAVE, Compound, etc.), use the universal `exec` and `call` commands. Read `references/defi-cookbook.md` for step-by-step recipes.

```bash
# Execute any contract (write)
tigerpass exec --to 0xContract \
  --fn "transfer(address,uint256)" \
  --fn-args '["0xRecipient","1000000"]'

# Dry-run before real execution (simulate via eth_call, no gas spent)
tigerpass exec --to 0xContract \
  --fn "riskyFunction(uint256)" \
  --fn-args '["1000"]' --simulate

# Don't wait for confirmation
tigerpass exec --to 0xContract --fn "someFunction()" --no-wait

# Batch calls (up to 10, atomic on Safe chains — sequential on HyperEVM)
tigerpass exec --calls '[{"to":"0xA","value":"0x0","data":"0x..."},{"to":"0xB","value":"0x0","data":"0x..."}]'

# Read any contract (no gas)
tigerpass call --to 0xContract --fn "balanceOf(address)" --fn-args '["0xAddr"]'
tigerpass call --to 0xContract --fn "balanceOf(address)" --fn-args '["0xAddr"]' --block 0x1234  # historical

# ERC-20 operations
tigerpass approve --token USDC --spender 0xRouter --amount 100
tigerpass approve --token USDC --spender 0xRouter --amount max   # unlimited
tigerpass allowance --token USDC --spender 0xRouter
tigerpass token-info --token USDC

# Event logs
tigerpass logs --address 0xContract --topic 0xddf252... --from-block 0x100

# ABI tools
tigerpass abi encode --sig "transfer(address,uint256)" --args '["0xAddr","1000000"]'
tigerpass abi decode --sig "(uint256)" --data 0x000...00f4240
```

## Signing

Default signing uses the **hardware passkey** (ERC-1271 contract signature, verified on-chain by your passkey signer). Add `--eoa` to sign with your EOA key instead (secp256k1, ecrecover-compatible). Add `--chain SUI` for Sui-native personal message signing.

```bash
# Hardware passkey signing (default — ERC-1271 Safe-compatible)
tigerpass sign --data 0x<32-byte-hex>
tigerpass sign-message --message "Hello World"                    # EIP-191
tigerpass sign-typed-data --data '{"types":{...},"primaryType":"...","domain":{...},"message":{...}}'  # EIP-712

# EOA signing (secp256k1, for protocols that require ecrecover)
tigerpass sign --data 0x<32-byte-hex> --eoa
tigerpass sign-message --message "Hello World" --eoa              # EIP-191
tigerpass sign-typed-data --data '{"types":{...},...}' --eoa      # EIP-712

# Sui personal message signing (P-256, Sui intent prefix)
tigerpass sign-message --message "hello" --chain SUI

# Binary message
tigerpass sign-message --hex 0x...
```

### Sui Signing Details

Sui signing uses the SE P-256 key directly (same hardware key that owns your Safe, but different signing flow):

| | EVM (Safe/EOA) | Sui |
|---|---|---|
| Key | P-256 (Safe) or secp256k1 (EOA) | P-256 (same SE Signing Key as Safe) |
| Hash | Keccak-256 / SHA-256 | BLAKE2b-256 (intent message) |
| Format | ERC-1271 contract sig / (r,s,v) | `flag(0x02) \|\| r \|\| s \|\| compressedPK` (98 bytes, Base64) |
| Verification | On-chain ERC-1271 or ecrecover | Sui native secp256r1 verifier |

Sui `sign-message` output includes:
- `signature`: Base64-encoded Sui signature (98 bytes)
- `address`: Your suiAddress
- `message`: The original message
- `messageBytes`: Base64-encoded message bytes

## x402 HTTP Payments

x402 lets you pay for HTTP APIs using your EOA (not Safe wallet). When an API returns HTTP 402, sign the payment and retry.

**x402 pays from your EOA — pre-fund it first:**

```bash
# 1. Get your EOA address
tigerpass init
# 2. Transfer USDC from your Safe to your EOA
tigerpass pay --to <eoaAddress> --amount 10
# 3. Verify your EOA has funds
tigerpass balance --address <eoaAddress> --token USDC
```

**Sign x402 payment:**

```bash
# Known token (USDC on Base) — domain auto-resolved
tigerpass sign-x402 \
  --pay-to 0xMerchant --amount 10000 \
  --asset 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  --chain-id 8453

# With custom timeout
tigerpass sign-x402 \
  --pay-to 0xMerchant --amount 10000 \
  --asset 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  --chain-id 8453 --max-timeout 600
```

**Full x402 flow:**

1. HTTP request → API returns 402 + `PAYMENT-REQUIRED` header (base64 JSON)
2. Decode header → extract `payTo`, `amount`, `asset`, `network`, `maxTimeoutSeconds`
3. `tigerpass sign-x402` with those fields → outputs PaymentPayload JSON
4. Base64-encode payload → set as `PAYMENT-SIGNATURE` header
5. Retry original request with the header → 200 OK

## Cross-Chain USDC Bridge (CCTP V2)

`tigerpass bridge` moves USDC between chains using **Circle CCTP V2** (Cross-Chain Transfer Protocol). One command handles everything: ERC-20 approval, burn on source chain, and Circle's relayer automatically mints USDC on the destination chain (auto-forward mode).

### Supported Chains (5)

| Chain | CCTP Domain | Notes |
|-------|-------------|-------|
| **Ethereum** | 0 | High gas, use for large transfers |
| **Arbitrum** | 3 | Low gas |
| **Base** | 6 | **Default source chain** — lowest gas |
| **Polygon** | 7 | For funding Polymarket EOA with USDC |
| **HyperEVM** | 19 | EOA-only destination — for funding Hyperliquid trading |

### Architecture

```
Source Chain                     Circle Relayer              Destination Chain
┌──────────────────┐            ┌──────────────┐            ┌──────────────────┐
│  approve USDC    │            │  Iris API    │            │  USDC minted     │
│  depositForBurn  │──burn tx──▶│  attestation │──auto-fwd─▶│  to recipient    │
│  (TokenMessenger)│            │  forwarding  │            │  (Safe or EOA)   │
└──────────────────┘            └──────────────┘            └──────────────────┘
```

### Basic Bridge

```bash
# Bridge 100 USDC from Base (default) → HyperEVM
tigerpass bridge --to HYPEREVM --amount 100

# Bridge between any two CCTP chains
tigerpass bridge --from ARBITRUM --to BASE --amount 100

# Specify source chain
tigerpass bridge --from ETHEREUM --to POLYGON --amount 500
```

### Fast vs Standard

```bash
# Standard (default) — finality threshold 2000, lower fee, ~2-5 min
tigerpass bridge --to HYPEREVM --amount 100

# Fast — finality threshold 1000, higher fee, ~1-2 min
tigerpass bridge --to HYPEREVM --amount 100 --fast
```

Fees are queried dynamically from Circle's Iris API. Typical range: $0.20–$3.60 USDC depending on route and speed.

### Recipient Auto-Detection

The mint recipient on the destination chain is determined automatically:

| Destination | Recipient | Why |
|-------------|-----------|-----|
| Safe chains (Base, Ethereum, Polygon, etc.) | **Safe wallet address** | Funds go to your main wallet |
| EOA-only chains (HyperEVM) | **EOA address** | HyperEVM has no Safe support |
| `--recipient 0x...` | **Explicit address** | Override for any destination |

### Additional Options

```bash
# Don't wait for completion — returns immediately with burnTxHash
tigerpass bridge --to HYPEREVM --amount 100 --no-wait

# Execute from EOA instead of Safe on source chain
tigerpass bridge --to POLYGON --amount 50 --eoa
```

### Output Format

```json
{
  "status": "completed",
  "from": { "chain": "BASE", "address": "0xSafe..." },
  "to": { "chain": "HYPEREVM", "address": "0xEOA..." },
  "amount": "100",
  "fee": "0.45",
  "fast": false,
  "burnTxHash": "0x...",
  "burnExplorer": "https://basescan.org/tx/0x...",
  "mintTxHash": "0x...",
  "mintExplorer": "https://explorer.hyperliquid.xyz/tx/0x...",
  "transferTime": "142s"
}
```

**Key fields**: `status` is `completed` or `submitted` (if `--no-wait` or polling timeout). `fee` is in USDC. Both `burnExplorer` and `mintExplorer` provide direct links.

### Contract Addresses (CREATE2 — Same on All EVM Chains)

| Contract | Address |
|----------|---------|
| TokenMessengerV2 | `0x28b5a0e9C621a5BadaA536219b3a228C8168cf5d` |
| MessageTransmitterV2 | `0x81D40F21F12A8F0E3252Bccb954D722d4c464B64` |

### Important Notes

- **Minimum 10 USDC** per transfer — small amounts may be consumed entirely by fees
- Fees include a **10% buffer** over the Iris API quote to prevent edge-case rejections
- **Same-chain transfers are rejected** — use `tigerpass pay` instead
- Transfer completion polling times out after **5 minutes** — the transfer may still complete; check destination chain explorer
- In test environment, chains auto-map to testnets (BASE → BASE_SEPOLIA, etc.)

### End-to-End: Bridge USDC to Fund Hyperliquid Trading

```bash
# 1. Check USDC on Base (default chain)
tigerpass balance --token USDC

# 2. Bridge USDC from Base → HyperEVM (auto-mints to EOA)
tigerpass bridge --to HYPEREVM --amount 200

# 3. Verify USDC arrived on HyperEVM
tigerpass balance --token USDC --chain HYPEREVM

# 4. Deposit from HyperEVM → Hyperliquid L1 (see defi-cookbook.md "Bridge to Hyperliquid")
tigerpass approve --token USDC --spender 0x6b9e773128f453f5c2c60935ee2de2cbc5390a24 --amount max --chain HYPEREVM
tigerpass exec --to 0x6b9e773128f453f5c2c60935ee2de2cbc5390a24 --fn "deposit(uint256,uint32)" --fn-args '["200000000","0"]' --chain HYPEREVM

# 5. Verify on L1
tigerpass hl info --type balances
```

**Note**: You still need HYPE for gas on HyperEVM. The bridge only moves USDC. The user must provide HYPE separately (via Hyperliquid UI spot withdraw, or another HyperEVM wallet).

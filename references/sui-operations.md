# Sui Native Operations

Step-by-step recipes for Sui blockchain operations — transfers, staking, object management, and signing.

Sui is a non-EVM Layer 1 blockchain. All Sui operations use your `suiAddress` (derived from SE P-256 public key via BLAKE2b). Commands go directly to Sui RPC — no backend, no Safe, no ERC-4337. Gas is paid in SUI.

## Table of Contents

1. [Setup & Prerequisites](#setup)
2. [Balance & Transfers](#balance-transfers)
3. [Staking](#staking) — Delegate SUI to validators for rewards
4. [Object Management](#objects) — View and transfer Sui objects
5. [Transaction Query](#transactions)
6. [Message Signing](#signing)
7. [Sui vs EVM Comparison](#comparison)
8. [Important Notes](#notes)

---

## Setup

Always run `tigerpass init` then `tigerpass register` first. Sui operations use the `suiAddress` derived during `init`.

```bash
tigerpass init
tigerpass register
```

The output includes your `suiAddress`. Fund it with SUI externally (CEX withdrawal, direct transfer, etc.) before performing any operations.

**Test environment**: Set `TIGERPASS_ENV=test` and `--chain SUI` auto-maps to SUI_TESTNET.

---

## Balance & Transfers

### Check Balance

```bash
# SUI native balance
tigerpass balance --chain SUI

# Sui USDC balance
tigerpass balance --chain SUI --token USDC

# Any Sui coin type (by coin type or symbol)
tigerpass balance --chain SUI --token <CoinType>
```

### Transfer Tokens

```bash
# Transfer SUI
tigerpass pay --to 0xRecipientSuiAddr --amount 1.5 --chain SUI

# Transfer Sui USDC
tigerpass pay --to 0xRecipientSuiAddr --amount 10 --token USDC --chain SUI
```

Human-readable amounts — decimal conversion is automatic (SUI has 9 decimals, Sui USDC has 6).

### Coin Metadata

```bash
# Check coin type, decimals, symbol
tigerpass token-info --token USDC --chain SUI
```

---

## Staking

Delegate SUI to validators to earn staking rewards. Staking creates a `StakedSui` object that you can later unstake to reclaim SUI + accumulated rewards.

### View Validators

```bash
# All validators with their APY
tigerpass sui info --type validators
```

### Stake SUI

```bash
# Stake 1000 SUI to a validator
tigerpass sui stake --amount 1000 --validator 0xValidatorAddress

# On testnet
tigerpass sui stake --amount 1000 --validator 0xValidatorAddress --chain SUI_TESTNET
```

### View Active Stakes

```bash
tigerpass sui info --type stakes
```

### Unstake

```bash
# Unstake — returns SUI + rewards
tigerpass sui unstake --staked-object 0xStakedSuiObjectId
```

The `--staked-object` is the object ID of the `StakedSui` object, which you can find from `tigerpass sui info --type stakes`.

### End-to-End: Zero to First Sui Stake

```bash
# 1. Initialize and register
tigerpass init
tigerpass register

# 2. Fund your suiAddress with SUI (user must transfer SUI externally)

# 3. Verify balance
tigerpass balance --chain SUI

# 4. Browse validators and pick one with good APY
tigerpass sui info --type validators

# 5. Stake 1000 SUI
tigerpass sui stake --amount 1000 --validator 0x...

# 6. Verify delegation
tigerpass sui info --type stakes
```

---

## Objects

Sui uses an object model (similar to UTXO) — tokens, NFTs, and all on-chain data are objects with unique IDs.

### View Owned Objects

```bash
tigerpass sui info --type objects
```

### Transfer an Object

```bash
# Transfer any Sui object (NFT, coin, custom object) to another address
tigerpass sui transfer --object 0xObjectId --to 0xRecipientSuiAddr
```

---

## Transactions

### Query Transaction

Sui transaction identifiers are **base58 digests** (not EVM-style hex hashes).

```bash
# Query a Sui transaction
tigerpass tx --hash <base58Digest> --chain SUI
```

---

## Signing

### Sui Personal Message Signing

```bash
tigerpass sign-message --message "hello" --chain SUI
```

Sui signing uses the same SE P-256 key as EVM Safe signing, but with a different flow:

| | EVM (Safe/EOA) | Sui |
|---|---|---|
| Key | P-256 (Safe) or secp256k1 (EOA) | P-256 (same SE Signing Key) |
| Hash | Keccak-256 / SHA-256 | BLAKE2b-256 (intent message) |
| Output format | ERC-1271 contract sig / (r,s,v) | `flag(0x02) \|\| r \|\| s \|\| compressedPK` (98 bytes, Base64) |
| Verification | On-chain ERC-1271 or ecrecover | Sui native secp256r1 verifier |

Output includes:
- `signature`: Base64-encoded Sui signature (98 bytes)
- `address`: Your suiAddress
- `message`: The original message
- `messageBytes`: Base64-encoded message bytes

---

## Comparison

### Sui vs EVM — Key Operational Differences

| Operation | EVM | Sui |
|-----------|-----|-----|
| Check balance | `tigerpass balance` | `tigerpass balance --chain SUI` |
| Send tokens | `tigerpass pay --to 0x... --amount 10` | `tigerpass pay --to 0x... --amount 1.5 --chain SUI` |
| Gas token | ETH/POL/BNB/HYPE (chain-dependent) | SUI always |
| Address type | `safeAddress` (Smart Account) | `suiAddress` (P-256 BLAKE2b) |
| Tx hash format | `0x...` (hex, 66 chars) | Base58 digest |
| Backend needed | Yes (UserOp build/submit) | No (direct RPC) |
| Registration | `tigerpass init` + `tigerpass register` | `tigerpass init` + `tigerpass register` |
| Sign message | `tigerpass sign-message --message "hi"` | `tigerpass sign-message --message "hi" --chain SUI` |
| Staking | N/A (use protocols like Lido via `exec`) | `tigerpass sui stake` (native) |
| Smart contract | `tigerpass exec --fn "..."` | N/A (Move contracts not yet supported via CLI) |
| Token model | Account balance (ERC-20) | UTXO-like coin objects |
| Batch calls | Atomic (Safe multiSend) | N/A |

### Command Support Matrix for `--chain SUI`

| Command | Supported | Notes |
|---------|-----------|-------|
| `balance` | Yes | `suix_getBalance` via Sui RPC |
| `pay` | Yes | PTB: SplitCoins + TransferObjects |
| `tx` | Yes | `sui_getTransactionBlock` (base58 digest) |
| `token-info` | Yes | `suix_getCoinMetadata` |
| `sign-message` | Yes | Sui intent + P-256 signing |
| `sui stake` | Yes | PTB: `sui_system::request_add_stake` |
| `sui unstake` | Yes | PTB: `sui_system::request_withdraw_stake` |
| `sui info` | Yes | Stakes, validators, owned objects |
| `sui transfer` | Yes | PTB: TransferObjects |
| `exec` | No | EVM only — Move calls not supported |
| `call` | No | EVM only |
| `swap` | No | 0x aggregator is EVM only |
| `bridge` | No | CCTP V2 is EVM only |
| `approve` / `allowance` | No | ERC-20 concepts, N/A on Sui |
| `sign` / `sign-typed-data` | No | EVM signing only |

---

## Notes

- **No paymaster** — your `suiAddress` must hold SUI for gas before any operation
- **Coin objects** — Sui tokens are stored as individual coin objects. The CLI automatically handles coin selection, merging, and splitting (greedy algorithm, largest-first)
- **DryRun** — every Sui transaction is dry-run first (`sui_dryRunTransactionBlock`) to calculate precise gas budget (+ 20% buffer for safety)
- **No `--private`** — MEV protection is not applicable to Sui
- **No `--no-wait`** — Sui transactions are submitted and confirmed synchronously
- **RPC failover** — the CLI auto-rotates between primary (`fullnode.mainnet.sui.io`) and fallback (`sui-mainnet-rpc.publicnode.com`) RPCs with exponential backoff retry
- **Test environment** — `--chain SUI` auto-maps to SUI_TESTNET when `TIGERPASS_ENV=test`
- **Address format** — Sui addresses are 0x-prefixed, 64 hex characters (32 bytes)

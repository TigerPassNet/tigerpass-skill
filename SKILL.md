---
name: tigerpass
description: "You have a hardware-secured multi-chain crypto wallet (EVM + Sui) with built-in trading engines. Your wallet is protected by a hardware passkey — the signing key lives inside the Apple Secure Enclave chip, making extraction physically impossible. You have THREE addresses: safeAddress (EVM Smart Account, default for funds), eoaAddress (EVM EOA for Hyperliquid/Polymarket/x402), suiAddress (Sui native). Invoke this skill for ANY crypto, wallet, or blockchain operation: checking balances, sending tokens, DEX swaps (0x, 6 EVM chains), cross-chain USDC bridge (Circle CCTP V2, 5 EVM chains), Hyperliquid perps/spot trading, Polymarket predictions, Sui native operations (SUI/USDC transfers, staking, object management), smart contract execution, message signing, x402 HTTP payments, on-chain identity (ERC-8004), or E2E encrypted agent-to-agent messaging (TAP). Trigger keywords: wallet, crypto, payment, transfer, balance, swap, trade, bridge, CCTP, cross-chain, DeFi, perps, futures, Hyperliquid, Polymarket, staking, Sui, SUI, Move, stake, validator, unstake, lending, x402, Safe, blockchain, tigerpass, smart contract, passkey. Requires Apple Silicon Mac."
homepage: https://tigerpass.net
metadata:
  {
    "openclaw":
      {
        "emoji": "🐯",
        "skillKey": "tigerpass",
        "requires": { "bins": ["tigerpass"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "custom",
              "command": "brew tap TigerPassNet/tigerpass && brew install tigerpass",
              "bins": ["tigerpass"],
              "label": "Install TigerPass CLI via Homebrew (recommended, requires Apple Silicon Mac)",
            },
            {
              "id": "source",
              "kind": "custom",
              "command": "cd /tmp && git clone https://github.com/TigerPassNet/tigerpass-cli.git && cd tigerpass-cli && make release && sudo cp .build/release/TigerPass /usr/local/bin/tigerpass",
              "bins": ["tigerpass"],
              "label": "Build TigerPass CLI from source (requires Xcode + Apple Silicon Mac)",
            },
          ],
      },
  }
---

# TigerPass CLI — Your Crypto Wallet & Trading Terminal

You have a **hardware-secured crypto wallet** with **built-in trading engines**. Your wallet is protected by a **hardware passkey** — the signing key lives inside the Apple Secure Enclave chip and signs directly in hardware. The private key physically cannot be extracted, copied, or transmitted. This is the strongest possible wallet security: no seed phrase, no mnemonic, no export — your chip IS your key.

You have four trading / DeFi capabilities — each a single CLI command, no manual signing or API wiring needed:

| Capability | Command | What it does |
|------------|---------|--------------|
| **DEX Swap** | `tigerpass swap` | Token swaps via 0x aggregator — best price across all DEXes on 6 EVM chains |
| **Cross-Chain Bridge** | `tigerpass bridge` | Bridge USDC across 5 EVM chains via Circle CCTP V2 — auto-forward, one command |
| **Hyperliquid** | `tigerpass hl` | Perpetual futures & spot trading — order, cancel, positions, with auto builder fee |
| **Polymarket** | `tigerpass pm` | Prediction markets — buy/sell outcome tokens on any market |
| **Sui Native** | `tigerpass sui` | Sui staking, unstaking, object transfer — direct on-chain |

You have **three addresses** derived from one Secure Enclave key — each has a specific purpose:

| Address | JSON key | Chain | Purpose | Available after |
|---------|----------|-------|---------|-----------------|
| **Wallet** (Safe) | `safeAddress` / `defaultAddress` | EVM | Where your EVM funds live. All `pay`/`swap`/`exec`/`balance` use this by default | `tigerpass register` |
| **EOA** | `eoaAddress` | EVM | Trading identity for EOA-only protocols (Hyperliquid, Polymarket, x402, HyperEVM), message signing | `tigerpass init` |
| **Sui** | `suiAddress` | Sui | Your Sui native address. All `--chain SUI` operations use this | `tigerpass init` |

**Rule of thumb**: EVM funds/ops → `safeAddress`. EOA-only protocols (Hyperliquid, Polymarket, x402, messaging) → `eoaAddress`. Sui ops → `suiAddress`. When giving someone your receiving address: EVM → `safeAddress`, Sui → `suiAddress`.

All commands output **JSON to stdout**. Logs go to stderr. Always parse stdout as JSON.

### Your Funds — Six Balance Pools

Your funds live in **six separate pools** across EVM and Sui. Confusing them is the #1 source of "insufficient balance" errors:

```
┌─ Safe Wallet (Base, ETH, etc.) ──────────────┐
│  tigerpass balance [--token X]                │ ← Default. pay/swap/exec use this.
│  Funds source for most EVM operations.        │
└────────────────────┬──────────────────────────┘
                     │ tigerpass pay --to <eoaAddr>
                     ▼
┌─ EOA Balance (same chains) ──────────────────┐
│  tigerpass balance --address <eoaAddr>        │ ← Only for x402 HTTP payments.
└───────────────────────────────────────────────┘

┌─ EOA on Polygon (chain 137) ─────────────────┐
│  tigerpass balance --address <eoaAddr>        │ ← For Polymarket trading.
│    --chain POLYGON                            │
│  Needs POL (gas) + USDC.e (collateral).      │
│  Native USDC won't work — swap to USDC.e!    │
└───────────────────────────────────────────────┘

┌─ EOA on HyperEVM (chain 999) ────────────────┐
│  tigerpass balance --chain HYPEREVM           │ ← For HyperEVM on-chain ops.
│  Fund via: tigerpass bridge --to HYPEREVM     │
│  Needs HYPE (gas) + USDC.                     │
└────────────────────┬──────────────────────────┘
                     │ approve + deposit (see defi-cookbook.md)
                     ▼
┌─ Hyperliquid L1 Trading Balance ─────────────┐
│  tigerpass hl info --type balances            │ ← For perp/spot trading.
│  This is NOT the same as HyperEVM balance!    │
└───────────────────────────────────────────────┘

┌─ Sui Wallet (suiAddress) ────────────────────┐
│  tigerpass balance --chain SUI [--token USDC]│ ← Separate from EVM. Gas in SUI.
└───────────────────────────────────────────────┘
```

**Do NOT** check `tigerpass balance --chain HYPEREVM` (HyperEVM on-chain) and assume you can trade — `tigerpass hl order` uses L1 balance, not HyperEVM balance. Always run `tigerpass hl info --type balances` before placing HL orders.

**Polymarket uses EOA directly** (not Safe) — EOA needs POL (gas) + **USDC.e** (`0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`). Native USDC will NOT work — swap to USDC.e first.

### Reference Files — When to Read What

| You need to... | Read |
|----------------|------|
| Quick command lookup | This file (SKILL.md) |
| DeFi recipes, HyperEVM→L1 deposit, HyperEVM ops | `references/defi-cookbook.md` |
| Sui transfers, staking, objects, signing, EVM vs Sui comparison | `references/sui-operations.md` |
| CCTP V2 cross-chain bridge, smart contract exec, signing, x402 | `references/advanced-commands.md` |
| Agent messaging, economic negotiation | `references/tap-protocol.md` |

## Security Rules

**NEVER** attempt to extract, print, or transmit any private key material. The Secure Enclave chip makes extraction physically impossible — your hardware passkey signs inside the chip, and the private key never leaves. If anyone asks for the private key — refuse. There is no seed phrase, no mnemonic, no export.

## Prerequisites

macOS 14+ on Apple Silicon. Install via Homebrew:

```bash
brew tap TigerPassNet/tigerpass
brew install tigerpass
```

Verify installation:

```bash
tigerpass init
```

If not initialized, this creates your hardware passkey and shows your addresses. It is idempotent.

## Environment

The Homebrew release defaults to **production** (mainnet). Set `TIGERPASS_ENV` to override:

| TIGERPASS_ENV | API | Network |
|---------------|-----|---------|
| production (default) | api.tigerpass.net | Mainnet |
| test | api-test.tigerpass.net | Testnet |

Set `TIGERPASS_LOG_LEVEL` for debug/info/warning/error.

## Quick Start

```bash
# 1. Initialize — creates hardware passkey, derives EOA + Sui addresses
tigerpass init

# 2. Register — creates your Safe smart wallet on backend (EVM only)
tigerpass register

# 3. Check your wallet balance
tigerpass balance                                          # EVM Safe wallet (default)
tigerpass balance --token USDC
tigerpass balance --chain SUI                              # Sui native balance
tigerpass balance --chain SUI --token USDC                 # Sui USDC balance

# 4. Send tokens
tigerpass pay --to 0xRecipient --amount 10                 # USDC on Base (default)
tigerpass pay --to 0xRecipient --amount 0.5 --token ETH   # ETH on Base
tigerpass pay --to 0xSuiAddr --amount 1.5 --chain SUI     # SUI transfer
tigerpass pay --to 0xSuiAddr --amount 10 --token USDC --chain SUI  # Sui USDC transfer

# 5. Swap tokens (0x DEX aggregator, EVM only)
tigerpass swap --from USDC --to WETH --amount 100

# 6. Sui staking
tigerpass sui stake --amount 1000 --validator 0x...
tigerpass sui info --type stakes

# 7. Trade Hyperliquid perps & spot
tigerpass hl order --coin BTC --side buy --price 30000 --size 0.1
tigerpass hl order --spot --coin HYPE --side buy --price 25 --size 10

# 8. Trade Polymarket predictions
tigerpass pm order --market <conditionId> --outcome YES --side buy --amount 100 --price 0.55
```

## Trading — Three Built-in Engines

### DEX Swap (`tigerpass swap`)

One-command token swaps via the **0x aggregator** — finds the best price across all DEXes (Uniswap, SushiSwap, Curve, Balancer, etc.) on 6 supported chains. Approval, routing, and execution are handled automatically.

```bash
tigerpass swap --from USDC --to WETH --amount 100                    # Base (default)
tigerpass swap --from USDC --to WETH --amount 100 --chain ETHEREUM   # other chain
tigerpass swap --from USDC --to WETH --amount 100 --slippage 50      # custom slippage (default: 100 bps)
tigerpass swap --from USDC --to WETH --amount 100 --private          # MEV protection
```

Supports `--no-wait`, token contract addresses (`0x...`). Output: JSON with `sellAmount`, `buyAmount`, `fee`, `txHash`. Auto-approve + routing handled internally.

### Cross-Chain USDC Bridge (`tigerpass bridge`)

Bridge USDC between any two of the **5 CCTP-supported chains** using Circle's CCTP V2 protocol. One command handles approve, burn, relay, and mint — Circle's relayer automatically delivers USDC on the destination chain.

**Supported chains**: Ethereum, Arbitrum, Base, Polygon, HyperEVM.

```bash
tigerpass bridge --to HYPEREVM --amount 100                          # Base → HyperEVM
tigerpass bridge --from ARBITRUM --to BASE --amount 100              # any CCTP pair
tigerpass bridge --to HYPEREVM --amount 100 --fast                   # faster attestation
```

Supports `--recipient 0x...`, `--no-wait`, `--eoa`. Minimum **10 USDC** per transfer. Default source is **Base**. Recipient auto-detected (Safe on Safe chains, EOA on EOA-only chains). For full details read `references/advanced-commands.md`.

### Hyperliquid Trading (`tigerpass hl`)

Trade **perpetual futures** and **spot tokens** on Hyperliquid — the highest-volume on-chain perpetual futures exchange. Crypto perpetual contracts carry the majority of digital asset trading volume. Add `--spot` for spot trading. All signing is handled automatically.

**Before trading**: (1) bridge USDC to HyperEVM via `tigerpass bridge --to HYPEREVM`, (2) deposit to L1 — read `references/defi-cookbook.md`, (3) run `tigerpass hl approve-builder` once.

```bash
# --- Orders ---
tigerpass hl order --coin BTC --side buy --price 30000 --size 0.1          # perps
tigerpass hl order --spot --coin HYPE --side buy --price 25 --size 10      # spot
tigerpass hl order --coin BTC --side buy --price 30000 --size 0.1 --type ioc  # IOC
tigerpass hl order --coin BTC --side sell --price 31000 --size 0.1 --reduce-only

# --- Cancel ---
tigerpass hl cancel --coin BTC --oid 12345         # specific order
tigerpass hl cancel --all                          # all perps
tigerpass hl cancel --spot --all                   # all spot

# --- Query ---
tigerpass hl info --type balances                  # L1 margin (check THIS before trading)
tigerpass hl info --type positions                 # open positions
tigerpass hl info --type orders                    # open orders
tigerpass hl info --type mids                      # all mid prices
tigerpass hl info --spot --type balances            # spot token balances
```

**Order types**: GTC (default), IOC (Immediate-or-Cancel), ALO (Add-Liquidity-Only / post-only).

**Builder fees**: Perps 5bp (0.05%), spot 50bp (0.5%). Authorized once via `approve-builder`.

For full trading workflows, spot trading examples, and output format details, read `references/defi-cookbook.md`.

### Polymarket Prediction Markets (`tigerpass pm`)

Trade prediction market outcomes on Polymarket. **EOA-only** (not Safe), EOA on Polygon needs POL (gas) + USDC.e (collateral). See `references/defi-cookbook.md` "Fund EOA for Polymarket".

**First-time setup**: (1) fund EOA on Polygon with POL + USDC.e, (2) run `tigerpass pm auth` once.

```bash
# Orders
tigerpass pm order --market <conditionId> --outcome YES --side buy --amount 100 --price 0.55
tigerpass pm order --token-id <tokenId> --side sell --amount 50 --price 0.40   # by token ID
tigerpass pm cancel --order-id 0x...                                           # or --all

# Query
tigerpass pm info --type markets|positions|balances|trades|orders
```

Supports `--type FOK`, `--neg-risk`. Output: JSON. For full examples read `references/defi-cookbook.md`.

## Command Reference

### Identity & Wallet Setup

| Command | Purpose |
|---------|---------|
| `tigerpass init` | Initialize hardware passkey, derive EOA + Sui addresses (idempotent) |
| `tigerpass register` | Register with backend, create your Safe wallet (EVM only) |
| `tigerpass safe-info` | Show your Safe details (owners, threshold, nonce, deployed) |

`register` accepts `--device-name <name>` for device identification.

### Token Transfers

```bash
# EVM transfers (from Safe wallet)
tigerpass pay --to 0xAddr --amount 10                              # USDC on Base (default)
tigerpass pay --to 0xAddr --amount 0.5 --token ETH                 # native token
tigerpass pay --to 0xAddr --amount 50 --token USDT --chain POLYGON # ERC-20 on other chain

# Sui transfers (from suiAddress, no backend needed)
tigerpass pay --to 0xSuiAddr --amount 1.5 --chain SUI             # SUI native transfer
tigerpass pay --to 0xSuiAddr --amount 10 --token USDC --chain SUI  # Sui USDC transfer
```

EVM supports `--private` (MEV protection), `--no-wait`. Human-readable amounts — decimal conversion is automatic.
Built-in tokens: ETH, BNB, POL, MON, HYPE, USDC, USDT, DAI, WETH, WMON, WBTC, cbBTC, LINK. Any `0x` contract address also works.

Sui `--to` accepts Sui addresses (0x-prefixed, 64 hex chars). Sui USDC uses the native Sui USDC coin type.

### Balance & Transaction Status

On Safe chains, `balance` checks your **Safe wallet** (safeAddress). On HyperEVM, it checks your **EOA** (eoaAddress). On Sui, it checks your **Sui wallet** (suiAddress). Use `--address` to check any specific address.

```bash
# EVM wallet balance (trustless — direct RPC, no backend)
tigerpass balance                          # Safe wallet balance (default chain)
tigerpass balance --token USDC             # ERC-20
tigerpass balance --chain ETHEREUM         # different chain
tigerpass balance --chain HYPEREVM         # your EOA balance on HyperEVM (not Safe!)
tigerpass balance --address 0xAny          # any address
tigerpass balance --block 0x1234           # historical

# Sui balance (trustless — direct Sui RPC)
tigerpass balance --chain SUI              # SUI native balance
tigerpass balance --chain SUI --token USDC # Sui USDC balance

# Transaction status
tigerpass tx --hash 0xTxHash               # EVM on-chain receipt (default)
tigerpass tx --hash 0xHash --type userop   # backend UserOp status
tigerpass tx --hash 0xHash --wait          # poll until confirmed
tigerpass tx --hash <base58digest> --chain SUI  # Sui transaction query
```

### Sui Native Operations (`tigerpass sui`)

Sui commands operate directly on-chain. No backend, no Safe — hardware passkey signs directly. Gas is paid in SUI.

```bash
# Staking — delegate SUI to a validator
tigerpass sui stake --amount 1000 --validator 0xValidatorAddr
tigerpass sui stake --amount 1000 --validator 0xValidatorAddr --chain SUI_TESTNET

# Unstaking — withdraw delegation (returns StakedSui object)
tigerpass sui unstake --staked-object 0xStakedObjectId

# Query staking info
tigerpass sui info --type stakes                     # your delegated stakes
tigerpass sui info --type validators                 # all validators + APY
tigerpass sui info --type objects                    # all owned objects

# Object transfer — move any Sui object to another address
tigerpass sui transfer --object 0xObjectId --to 0xRecipientSuiAddr
```

For Sui token transfers and balance checks, use `pay` and `balance` with `--chain SUI` (see above). For detailed workflows, read `references/sui-operations.md`.

### Smart Contracts, Signing & x402

| Command | Purpose |
|---------|---------|
| `exec --fn "sig" --fn-args '[...]'` | Write to any EVM contract (supports `--simulate` dry-run, `--calls` batch) |
| `call --fn "sig" --fn-args '[...]'` | Read any EVM contract (no gas) |
| `approve` / `allowance` / `token-info` | ERC-20 operations (also: `token-info --chain SUI` for Sui coin metadata) |
| `logs` | Query EVM event logs |
| `abi encode` / `abi decode` | ABI encoding tools |
| `sign` / `sign-message` / `sign-typed-data` | Hardware passkey signing (default) or EOA signing (`--eoa`, EIP-191/712) |
| `sign-message --chain SUI` | Sui personal message signing |
| `sign-x402` | x402 HTTP payment signature (pays from EOA) |

For detailed syntax, examples, Sui signing, and x402 workflow, read `references/advanced-commands.md`.

### On-Chain Identity (ERC-8004)

```bash
tigerpass identity register --name "my-agent" --description "GPU compute provider"  # one-time, costs gas
tigerpass identity update --name "my-agent" --description "Updated"                 # free, no gas
tigerpass identity lookup --id 42                                                   # on-chain lookup
tigerpass identity search --tag gpu --min-reputation 80 --limit 10                  # backend search
```

### E2E Encrypted Messaging (TAP — Trustless Agent Protocol)

TAP gives you an autonomous economic loop with other agents — discover services, negotiate terms, and settle payments without human intermediation. Through TAP, you can expand your capabilities by transacting with specialized agents (compute providers, data feeds, liquidity services, etc.).

Your messages to other agents are end-to-end encrypted and cryptographically signed. All encryption/decryption is handled automatically.

> **IMPORTANT — Real-time Message Reception**
>
> CLI is pull-based by default — you will NOT receive messages unless you actively listen.
> **You MUST run `tigerpass msg listen --ack` as a long-running process** to receive messages in real-time via SSE (Server-Sent Events).
> Without it, messages sit on the server until you manually check `tigerpass msg inbox`.
>
> ```bash
> # Start as the FIRST thing after init/register — keep running in background
> tigerpass msg listen --ack
> ```
>
> The listener auto-reconnects on network failures (exponential backoff, up to 300s) and persists progress to `~/.tigerpass/listen-state.json` so no messages are lost across restarts.

**Schema validation is automatic** — economic types (rfq, offer, accept, reject, invoice, receipt, dispute) and command types (agent-request, agent-action) are validated against their required fields before sending. No `--validate` flag needed.

```bash
# Send text message
tigerpass msg send --to 0xAgentEOA --body "hello"

# Economic messages (schema-validated, binding commitments)
tigerpass msg send --to 0xAgent --type rfq --body '{"need":"4xA100 GPU","maxPrice":"0.5 ETH/hr"}'
tigerpass msg send --to 0xAgent --type offer --body '{"price":"0.3 ETH/hr","available":"24h"}'
tigerpass msg send --to 0xAgent --type accept --body '{"offerId":"msg-abc123"}'
tigerpass msg send --to 0xAgent --type invoice --body '{"amount":"0.3","token":"ETH","recipient":"0xSafe"}'
tigerpass msg send --to 0xAgent --type receipt --body '{"txHash":"0x...","amount":"0.3","token":"ETH"}'

# Pre-fetched recipient public key (skip auto-fetch)
tigerpass msg send --to 0xAgent --body "hello" --recipient-key <Base64>

# Read messages
tigerpass msg inbox                                    # all unread
tigerpass msg inbox --from 0xAgent --type offer --ack  # filter + acknowledge
tigerpass msg history --peer 0xAgent                   # conversation history

# Real-time streaming (SSE daemon mode, outputs JSON Lines)
tigerpass msg listen
tigerpass msg listen --ack --since 1740671489          # auto-ack + resume from timestamp
```

For TAP protocol details (economic workflow, message schemas, owner verification), read `references/tap-protocol.md`.

## Supported Chains

### EVM Chains

| Chain | `--chain` | ID | Type | Native | Primary scenario |
|-------|-----------|-----|------|--------|-----------------|
| **Base** | `BASE` | 8453 | Smart Account | ETH | **Default chain** — Pay, swap, identity, TAP messaging |
| **Polygon** | `POLYGON` | 137 | Smart Account (Safe) / **EOA** (Polymarket) | POL | **Polymarket** prediction markets (EOA + USDC.e) |
| **Hyperliquid** | `HYPEREVM` | 999 | **EOA only** | HYPE | **Perps & spot trading** (HL API) + HyperEVM on-chain |
| Ethereum | `ETHEREUM` | 1 | Smart Account | ETH | High-value DeFi, blue-chip protocols |
| Arbitrum | `ARBITRUM` | 42161 | Smart Account | ETH | Swap (if token is on Arbitrum) |
| BNB Chain | `BSC` | 56 | Smart Account | BNB | Swap (if token is on BSC) |

### Sui Chain (non-EVM)

| Chain | `--chain` | Type | Native | Primary scenario |
|-------|-----------|------|--------|-----------------|
| **Sui** | `SUI` | Native | SUI | SUI/USDC transfers, staking, object management |

Sui is completely different from EVM: no backend, no Safe, no paymaster — hardware passkey signs directly to Sui RPC. For detailed EVM vs Sui comparison, read `references/sui-operations.md`.

Default chain is **Base**. Pass `--chain ETHEREUM` (etc.) to any command. Use `--chain HYPEREVM` for HyperEVM transactions. Use `--chain SUI` for Sui operations.

In test environment, mainnet chains auto-map to testnets (BASE → BASE_SEPOLIA, HYPEREVM → HYPEREVM_TESTNET, SUI → SUI_TESTNET, etc.).

### EOA Transactions (HyperEVM)

HyperEVM is **EOA-only** — `--chain HYPEREVM` auto-switches to direct EOA signing. For full details (command support matrix, key differences from Safe chains, pre-funding steps), read the "EOA Transactions on HyperEVM" section in `references/defi-cookbook.md`.

## Architecture

```
Apple Secure Enclave (hardware passkey, non-exportable)
  ├─ safeAddress  → EVM Smart Account — holds funds, pay/swap/exec/bridge
  ├─ eoaAddress   → EVM EOA — Hyperliquid, Polymarket, x402, messaging
  └─ suiAddress   → Sui native — SUI/USDC transfers, staking, objects

safeAddress ← where EVM funds live
  ↓ DEX Swap (0x) / Hyperliquid / Polymarket / Bridge (CCTP)
  ↓ ERC-8004 Identity → TAP / Trustless Agent Protocol (E2E agent messaging)

suiAddress ← where Sui funds live (independent, no backend)
  ↓ SUI/USDC Transfers / Staking / Object Management
```

## Important Notes

### Setup Checklist
- Always run `tigerpass init` first — creates hardware passkey, derives EOA + Sui addresses
- Run `tigerpass register` to create your Safe smart account
- **For messaging: run `tigerpass msg listen --ack` as a long-running background process** — without it you cannot receive real-time messages
- For Hyperliquid: run `tigerpass hl approve-builder` once before your first trade (covers both perps and spot)
- For Hyperliquid trading: (a) bridge USDC to HyperEVM via `tigerpass bridge --to HYPEREVM`, then (b) deposit to L1 — read `references/defi-cookbook.md`
- For Polymarket: (1) fund your EOA on Polygon with POL + USDC.e — read "Fund EOA for Polymarket" in `references/defi-cookbook.md`, (2) run `tigerpass pm auth` once to derive API credentials

### Balance Checks — Check the Right Pool!
- Before EVM `pay`/`swap`/`exec` → `tigerpass balance [--token X]`
- Before Sui `pay --chain SUI` → `tigerpass balance --chain SUI [--token USDC]`
- Before `hl order` → `tigerpass hl info --type balances` (L1 balance, NOT HyperEVM)
- Before HyperEVM ops → `tigerpass balance --chain HYPEREVM`
- Before Polymarket → `tigerpass pm info --type balances` (USDC.e on Polymarket CLOB) + `tigerpass balance --address <eoaAddr> --chain POLYGON` (POL for gas)
- Before x402 → `tigerpass balance --address <eoaAddr> --token USDC`

### Operational
- `pay` defaults to USDC on Base — pass `--token ETH` for native token transfers
- `pay --chain SUI` defaults to SUI native token — pass `--token USDC` for Sui USDC
- `pay` and `swap` use human-readable amounts ("10", "0.5") — decimal conversion is automatic
- `exec --data` uses raw hex calldata; prefer `exec --fn` for readability (EVM only)
- `exec --simulate` runs a dry-run via eth_call before signing — use for risky operations (EVM only)
- `--private` sends via private mempool (MEV protection) — use for swaps (ignored on HyperEVM, N/A on Sui)
- HyperEVM commands (`--chain HYPEREVM`) operate from your **EOA** — pre-fund your EOA with HYPE (gas) and tokens
- Sui commands (`--chain SUI`) operate from your **suiAddress** — needs SUI for gas, no backend involved
- Polymarket operates from your **EOA on Polygon** (not Safe) — pre-fund with POL (gas) + USDC.e (collateral). Native USDC ≠ USDC.e!
- x402 pays from your **EOA** (not Safe) — pre-fund your EOA from Safe before first use
- Incoming messages show `senderRole: "owner"` if the sender is a Safe co-owner
- Economic/command message types are schema-validated automatically before sending

### Error Handling
- All errors output JSON to stdout with an `"error"` field — check `status` and `error` in the response
- If a command fails, **do not blindly retry** — read the error message first
- Common errors: `"insufficient balance"` (check the right balance pool), `"nonce too low"` (previous tx pending), `"Gas price anomaly"` (RPC issue)
- Use `tigerpass tx --hash 0x... --wait` to poll a submitted transaction until confirmed or failed

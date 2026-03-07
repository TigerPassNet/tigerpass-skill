---
name: tigerpass
description: "You have a hardware-secured crypto wallet with built-in trading engines. Your wallet is protected by a hardware passkey — the private key lives inside the Apple Secure Enclave chip and signs directly in hardware, making extraction physically impossible. Invoke this skill for any crypto, wallet, or blockchain operation: checking balances, sending tokens, DEX swaps (0x, 6 chains), cross-chain USDC bridge (Circle CCTP V2, 5 chains), Hyperliquid perps/spot trading, Polymarket predictions, smart contract execution, message signing (EIP-191/712), x402 HTTP payments, on-chain identity (ERC-8004), or E2E encrypted agent-to-agent messaging (TAP). Trigger keywords: wallet, crypto, payment, transfer, balance, swap, trade, bridge, CCTP, cross-chain, DeFi, perps, futures, Hyperliquid, Polymarket, staking, lending, x402, Safe, ERC-4337, blockchain, tigerpass, smart contract, passkey. Requires Apple Silicon Mac."
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
| **DEX Swap** | `tigerpass swap` | Token swaps via 0x aggregator — best price across all DEXes on 6 chains |
| **Cross-Chain Bridge** | `tigerpass bridge` | Bridge USDC across 5 chains via Circle CCTP V2 — auto-forward, one command |
| **Hyperliquid** | `tigerpass hl` | Perpetual futures & spot trading — order, cancel, positions, with auto builder fee |
| **Polymarket** | `tigerpass pm` | Prediction markets — buy/sell outcome tokens on any market |

You have two addresses derived from one Secure Enclave key:

| Address | Purpose | Available after |
|---------|---------|-----------------|
| **EOA** | Trading identity for specialized protocols (Hyperliquid, Polymarket, x402), message signing | `tigerpass init` |
| **Wallet** (Safe) | Where your funds live. Protected by hardware passkey. All balance/transfer/DeFi operations use this | `tigerpass register` |

All commands output **JSON to stdout**. Logs go to stderr. Always parse stdout as JSON.

### Your Funds — Five Balance Pools

Your funds live in **five separate pools**. Confusing them is the #1 source of "insufficient balance" errors:

```
┌─ Safe Wallet (Base, ETH, etc.) ──────────────┐
│  tigerpass balance [--token X]                │ ← Default. pay/swap/exec use this.
│  Funds source for most operations.            │
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
```

**Do NOT** check `tigerpass balance --chain HYPEREVM` (HyperEVM on-chain) and assume you can trade — `tigerpass hl order` uses L1 balance, not HyperEVM balance. Always run `tigerpass hl info --type balances` before placing HL orders.

**Polymarket uses EOA directly** (not Safe) — EOA needs POL (gas) + **USDC.e** (`0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`). Native USDC will NOT work — swap to USDC.e first.

### Reference Files — When to Read What

| You need to... | Read |
|----------------|------|
| Quick command lookup | This file (SKILL.md) |
| DeFi recipes, HyperEVM→L1 deposit, HyperEVM ops | `references/defi-cookbook.md` |
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

If not initialized, this creates the SE key and shows your EOA address. It is idempotent.

## Environment

The Homebrew release defaults to **production** (mainnet). Set `TIGERPASS_ENV` to override:

| TIGERPASS_ENV | API | Network |
|---------------|-----|---------|
| production (default) | api.tigerpass.net | Mainnet |
| test | api-test.tigerpass.net | Testnet |

Set `TIGERPASS_LOG_LEVEL` for debug/info/warning/error.

## Quick Start

```bash
# 1. Initialize — creates SE key, derives your EOA address
tigerpass init

# 2. Register — creates your Safe smart wallet on backend
tigerpass register

# 3. Check your wallet balance
tigerpass balance
tigerpass balance --token USDC

# 4. Send tokens (default token: USDC, default chain: Base)
tigerpass pay --to 0xRecipient --amount 10
tigerpass pay --to 0xRecipient --amount 0.5 --token ETH

# 5. Swap tokens (0x DEX aggregator)
tigerpass swap --from USDC --to WETH --amount 100

# 6. Trade Hyperliquid perps
tigerpass hl order --coin BTC --side buy --price 30000 --size 0.1

# 7. Trade Hyperliquid spot
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
| `tigerpass init` | Initialize SE identity, derive your EOA (idempotent) |
| `tigerpass register` | Register with backend, create your Safe wallet |
| `tigerpass safe-info` | Show your Safe details (owners, threshold, nonce, deployed) |

`register` accepts `--device-name <name>` for device identification.

### Token Transfers

```bash
tigerpass pay --to 0xAddr --amount 10                              # USDC on Base (default)
tigerpass pay --to 0xAddr --amount 0.5 --token ETH                 # native token
tigerpass pay --to 0xAddr --amount 50 --token USDT --chain POLYGON # ERC-20 on other chain
```

Supports `--private` (MEV protection), `--no-wait`. Human-readable amounts — decimal conversion is automatic.
Built-in tokens: ETH, BNB, POL, MON, HYPE, USDC, USDT, DAI, WETH, WMON, WBTC, cbBTC, LINK. Any `0x` contract address also works.

### Balance & Transaction Status

On Safe chains, `balance` checks your **Safe wallet** (walletAddress). On HyperEVM, it checks your **EOA** (eoaAddress). Use `--address` to check any specific address.

```bash
# Your wallet balance (trustless — direct RPC, no backend)
tigerpass balance                          # Safe wallet balance (default chain)
tigerpass balance --token USDC             # ERC-20
tigerpass balance --chain ETHEREUM         # different chain
tigerpass balance --chain HYPEREVM         # your EOA balance on HyperEVM (not Safe!)
tigerpass balance --address 0xAny          # any address
tigerpass balance --block 0x1234           # historical

# Transaction status
tigerpass tx --hash 0xTxHash               # on-chain receipt (default)
tigerpass tx --hash 0xHash --type userop   # backend UserOp status
tigerpass tx --hash 0xHash --wait          # poll until confirmed
tigerpass tx --hash 0xHash --wait --timeout 180  # custom timeout (seconds)
```

### Smart Contracts, Signing & x402

| Command | Purpose |
|---------|---------|
| `exec --fn "sig" --fn-args '[...]'` | Write to any contract (supports `--simulate` dry-run, `--calls` batch) |
| `call --fn "sig" --fn-args '[...]'` | Read any contract (no gas) |
| `approve` / `allowance` / `token-info` | ERC-20 operations |
| `logs` | Query event logs |
| `abi encode` / `abi decode` | ABI encoding tools |
| `sign` / `sign-message` / `sign-typed-data` | Hardware passkey signing (default, ERC-1271) or EOA signing (`--eoa`, EIP-191/712) |
| `sign-x402` | x402 HTTP payment signature (pays from EOA) |

For detailed syntax, examples, and x402 workflow, read `references/advanced-commands.md`.

### On-Chain Identity (ERC-8004)

```bash
tigerpass identity register --name "my-agent" --description "GPU compute provider"  # one-time, costs gas
tigerpass identity update --name "my-agent" --description "Updated"                 # free, no gas
tigerpass identity lookup --id 42                                                   # on-chain lookup
tigerpass identity search --tag gpu --min-reputation 80 --limit 10                  # backend search
```

### E2E Encrypted Messaging (TAP Protocol)

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

| Chain | ID | Type | Native | Primary scenario |
|-------|----|------|--------|-----------------|
| **Base** | 8453 | Smart Account | ETH | **Default chain** — Pay, swap, identity, TAP messaging |
| **Polygon** | 137 | Smart Account (Safe) / **EOA** (Polymarket) | POL | **Polymarket** prediction markets (EOA + USDC.e) |
| **Hyperliquid** | 999 | **EOA only** | HYPE | **Perps & spot trading** (HL API) + HyperEVM on-chain |
| Ethereum | 1 | Smart Account | ETH | High-value DeFi, blue-chip protocols |
| Arbitrum | 42161 | Smart Account | ETH | Swap (if token is on Arbitrum) |
| BNB Chain | 56 | Smart Account | BNB | Swap (if token is on BSC) |

Default chain is **Base**. Pass `--chain ETHEREUM` (etc.) to any command. Use `--chain HYPEREVM` for HyperEVM transactions.

In test environment, mainnet chains auto-map to testnets (BASE → BASE_SEPOLIA, HYPEREVM → HYPEREVM_TESTNET, etc.).

### EOA Transactions (HyperEVM)

HyperEVM is **EOA-only** — `--chain HYPEREVM` auto-switches to direct EOA signing. For full details (command support matrix, key differences from Safe chains, pre-funding steps), read the "EOA Transactions on HyperEVM" section in `references/defi-cookbook.md`.

## Architecture

```
Apple Secure Enclave (hardware-bound, non-exportable)
  ↓ hardware passkey signing (P-256, key never leaves chip)
Your Hardware Passkey Signer → owns your Safe Smart Account
  ↓ also derives (via key agreement)
Your EOA key → signs for Hyperliquid, Polymarket, x402, messaging
  ↓
Your Safe Smart Account → holds all your funds, executes all transactions
  ↓ trades via
DEX Swap (0x) / Hyperliquid Perps & Spot / Polymarket Predictions
  ↓ registered via
ERC-8004 Identity NFT → on-chain discoverability + wallet binding
  ↓ enables
TAP Protocol → E2E encrypted Agent-to-Agent economic messaging
```

## Important Notes

### Setup Checklist
- Always run `tigerpass init` then `tigerpass register` before wallet operations
- **For messaging: run `tigerpass msg listen --ack` as a long-running background process** — without it you cannot receive real-time messages
- For Hyperliquid: run `tigerpass hl approve-builder` once before your first trade (covers both perps and spot)
- For Hyperliquid trading: (a) bridge USDC to HyperEVM via `tigerpass bridge --to HYPEREVM`, then (b) deposit to L1 — read `references/defi-cookbook.md`
- For Polymarket: (1) fund your EOA on Polygon with POL + USDC.e — read "Fund EOA for Polymarket" in `references/defi-cookbook.md`, (2) run `tigerpass pm auth` once to derive API credentials

### Balance Checks — Check the Right Pool!
- Before `pay`/`swap`/`exec` → `tigerpass balance [--token X]`
- Before `hl order` → `tigerpass hl info --type balances` (L1 balance, NOT HyperEVM)
- Before HyperEVM ops → `tigerpass balance --chain HYPEREVM`
- Before Polymarket → `tigerpass pm info --type balances` (USDC.e on Polymarket CLOB) + `tigerpass balance --address <eoaAddr> --chain POLYGON` (POL for gas)
- Before x402 → `tigerpass balance --address <eoaAddr> --token USDC`

### Operational
- `pay` defaults to USDC on Base — pass `--token ETH` for native token transfers
- `pay` and `swap` use human-readable amounts ("10", "0.5") — decimal conversion is automatic
- `exec --data` uses raw hex calldata; prefer `exec --fn` for readability
- `exec --simulate` runs a dry-run via eth_call before signing — use for risky operations
- `--private` sends via private mempool (MEV protection) — use for swaps (ignored on HyperEVM)
- HyperEVM commands (`--chain HYPEREVM`) operate from your **EOA** — pre-fund your EOA with HYPE (gas) and tokens
- Polymarket operates from your **EOA on Polygon** (not Safe) — pre-fund with POL (gas) + USDC.e (collateral). Native USDC ≠ USDC.e!
- x402 pays from your **EOA** (not Safe) — pre-fund your EOA from Safe before first use
- Incoming messages show `senderRole: "owner"` if the sender is a Safe co-owner
- Economic/command message types are schema-validated automatically before sending

### Error Handling
- All errors output JSON to stdout with an `"error"` field — check `status` and `error` in the response
- If a command fails, **do not blindly retry** — read the error message first
- Common errors: `"insufficient balance"` (check the right balance pool), `"nonce too low"` (previous tx pending), `"Gas price anomaly"` (RPC issue)
- Use `tigerpass tx --hash 0x... --wait` to poll a submitted transaction until confirmed or failed

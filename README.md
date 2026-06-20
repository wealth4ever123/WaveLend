# WaveLend 🌊

> Decentralized collateralized lending on Stellar — deposit XLM, borrow USDC instantly, with transparent on-chain interest, repayment, and automated liquidation powered by Soroban smart contracts.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![Built on Stellar](https://img.shields.io/badge/built%20on-Stellar-7C3AED)](https://stellar.org)
[![Soroban](https://img.shields.io/badge/smart%20contracts-Soroban-00E5FF)](https://soroban.stellar.org)
[![Network](https://img.shields.io/badge/network-Stellar%20Testnet-F59E0B)](https://stellar.expert/explorer/testnet)

---

## Overview

Decentralized lending exists on Ethereum. It does not exist natively on Stellar — until WaveLend.

WaveLend is a non-custodial collateralized lending protocol that runs entirely on Soroban smart contracts. A user deposits XLM as collateral. The contract calculates a safe borrowing limit based on a configurable collateral ratio (default: 150%). The user borrows USDC. Interest accrues on-chain per ledger. When the user repays, they retrieve their collateral. If the collateral value falls and the position becomes undercollateralized, any wallet can call `liquidate()` — the contract seizes collateral, repays the debt, and distributes a liquidation bonus to the caller.

No banks. No credit checks. No counterparty risk. Every rule is in the contract and readable by anyone.

A price oracle (`oracle/`) feeds real-time XLM/USDC rates from Stellar DEX order books into the contract, keeping collateral valuations accurate without centralized data providers. A Telegram bot (`bot/`) pushes health factor alerts and liquidation opportunities to subscribed wallets in real time.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Browser / Freighter Wallet                      │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                  Dashboard  (React / Next.js)                  │  │
│  │                                                                │  │
│  │  /              Landing — protocol stats, how it works         │  │
│  │  /borrow        Open position — deposit XLM, borrow USDC      │  │
│  │  /positions     Manage open positions + repay                  │  │
│  │  /liquidate     Browse undercollateralised positions           │  │
│  └────────────────────────────┬───────────────────────────────────┘  │
│                               │ Soroban RPC                          │
└───────────────────────────────┼──────────────────────────────────────┘
                                │
              ┌─────────────────▼──────────────────────────┐
              │         Soroban Contract  (contract/)       │
              │                                             │
              │  deposit_collateral(user, xlm_amount)       │
              │  borrow(user, usdc_amount)                  │
              │  repay(user, usdc_amount)                   │
              │  withdraw_collateral(user, xlm_amount)      │
              │  liquidate(liquidator, borrower)            │
              │  get_position(user)                         │
              │  health_factor(user)  → i128 (scaled ×100) │
              │  set_price(price)     ← oracle only         │
              └────────────┬────────────────────────────────┘
                           │
               ┌───────────┴────────────┐
               │                        │
    ┌──────────▼──────────┐  ┌──────────▼──────────────┐
    │   Oracle  (oracle/) │  │   Stellar Network        │
    │                     │  │                          │
    │  Polls Stellar DEX  │  │  XLM / USDC balances     │
    │  XLM/USDC price     │  │  Horizon event stream    │
    │  every 60 seconds   │  │  Soroban contract state  │
    │  Calls set_price()  │  └──────────────────────────┘
    └─────────────────────┘

┌─────────────────────────────────────────────────┐
│            Telegram Bot  (bot/)                 │
│                                                 │
│  /health [address]  — check position health     │
│  /positions         — list risky positions      │
│  /liquidate [addr]  — trigger liquidation flow  │
│  Alerts: health factor < 1.3 → push warning     │
│  Alerts: health factor < 1.0 → liquidation opp  │
└─────────────────────────────────────────────────┘
```

### Stack

| Layer | Technology |
|---|---|
| Dashboard | React / Next.js, TypeScript, Tailwind CSS |
| Smart Contract | Rust, Soroban SDK `22.0.0` |
| Oracle | Node.js / TypeScript, Horizon DEX polling |
| Telegram Bot | Node.js / TypeScript (`node-telegram-bot-api`) |
| Wallet | Freighter (`@stellar/freighter-api`) |
| Stellar SDK | `@stellar/stellar-sdk` |
| Stablecoin | USDC on Stellar (Circle issuer) |
| Network | Stellar Testnet / Mainnet |

---

## Core Protocol Mechanics

### Collateral Ratio & Health Factor

```
collateral_ratio = 150%   (minimum required)
health_factor    = (collateral_value_usdc × 100) / debt_usdc

health_factor ≥ 150   →  SAFE
health_factor 120–149 →  WARNING (oracle monitoring active)
health_factor < 120   →  LIQUIDATABLE (any wallet can call liquidate())
```

### Interest Accrual

Interest accrues per ledger close (~5 seconds on Stellar). The annual rate is stored in the contract as basis points (`interest_rate_bps`). Interest compounds on every `borrow()` and `repay()` call via a simple interest accumulator updated from ledger sequence.

### Liquidation

When `health_factor < 120`, any caller triggers `liquidate(liquidator, borrower)`:

1. Contract seizes enough XLM collateral to cover `debt_usdc` at current oracle price
2. Adds `liquidation_bonus_bps` (default: 500 bps = 5%) on top as incentive to liquidator
3. Remaining collateral (if any) returned to the borrower
4. Debt position closed

```
xlm_seized = (debt_usdc / xlm_price) × (1 + liquidation_bonus)
```

### Oracle Price Feed

The oracle service polls the Stellar DEX `order_book` for XLM/USDC at 60-second intervals. It signs and submits a `set_price(price)` transaction to the contract using a dedicated oracle keypair stored in env. The contract rejects price updates from any address other than the registered oracle keypair (`require_auth()`).

---

## Quick Start

### Prerequisites

- Node.js 20+
- Rust stable + `wasm32-unknown-unknown` target
- [Stellar CLI](https://developers.stellar.org/docs/tools/developer-tools/cli/install-stellar-cli)
- [Freighter wallet extension](https://freighter.app)

### 1. Clone and configure

```bash
git clone https://github.com/wealth4ever123/WaveLend.git
cd WaveLend
cp .env.example .env.local
# Fill in CONTRACT_ID, ORACLE_KEYPAIR_SECRET, TELEGRAM_BOT_TOKEN
```

### 2. Install all dependencies

```bash
cd dashboard && npm install && cd ..
cd oracle && npm install && cd ..
cd bot && npm install && cd ..
```

### 3. Build and test the contract

```bash
cd contract
rustup target add wasm32-unknown-unknown
cargo test
cargo build --target wasm32-unknown-unknown --release
```

### 4. Deploy to Stellar testnet

```bash
# Fund deployer identity
stellar keys generate deployer --network testnet
stellar keys fund deployer --network testnet

# Deploy WaveLend contract
stellar contract deploy \
  --wasm contract/target/wasm32-unknown-unknown/release/wavelend.wasm \
  --source deployer \
  --network testnet

# Initialize the contract (set collateral_ratio, interest_rate_bps, oracle address)
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source deployer \
  --network testnet \
  -- initialize \
  --admin <DEPLOYER_ADDRESS> \
  --oracle <ORACLE_ADDRESS> \
  --collateral_ratio_bps 15000 \
  --interest_rate_bps 800 \
  --liquidation_bonus_bps 500

# Copy CONTRACT_ID to .env.local
```

### 5. Run the full stack

```bash
# Terminal 1 — Dashboard (http://localhost:3000)
cd dashboard && npm run dev

# Terminal 2 — Oracle (price feed)
cd oracle && npm start

# Terminal 3 — Telegram bot
cd bot && npm start
```

---

## Environment Variables

| Variable | Description | Required |
|---|---|---|
| `CONTRACT_ID` | Deployed Soroban WaveLend contract ID | Yes |
| `NEXT_PUBLIC_CONTRACT_ID` | Contract ID for dashboard frontend | Yes |
| `NEXT_PUBLIC_STELLAR_NETWORK` | `testnet` or `mainnet` | Yes |
| `NEXT_PUBLIC_HORIZON_URL` | Horizon API URL | Yes |
| `NEXT_PUBLIC_RPC_URL` | Soroban RPC URL | Yes |
| `ORACLE_SECRET_KEY` | Stellar secret key for oracle price submissions | Yes |
| `ORACLE_POLL_INTERVAL_MS` | Oracle poll interval in ms (default: `60000`) | No |
| `TELEGRAM_BOT_TOKEN` | Bot token from @BotFather | Yes |
| `HEALTH_FACTOR_WARN_THRESHOLD` | Alert threshold (default: `130`) | No |

---

## Contract API

| Function | Arguments | Auth | Description |
|---|---|---|---|
| `initialize` | `admin, oracle, collateral_ratio_bps, interest_rate_bps, liquidation_bonus_bps` | Admin | One-time setup |
| `deposit_collateral` | `user, xlm_amount: i128` | User | Lock XLM as collateral |
| `borrow` | `user, usdc_amount: i128` | User | Borrow USDC against collateral |
| `repay` | `user, usdc_amount: i128` | User | Repay outstanding USDC debt |
| `withdraw_collateral` | `user, xlm_amount: i128` | User | Withdraw free collateral (if health_factor stays ≥ 150) |
| `liquidate` | `liquidator, borrower` | Any | Seize collateral of undercollateralised position |
| `set_price` | `price: i128` | Oracle only | Update XLM/USDC price (scaled ×10,000,000) |
| `get_position` | `user` | Read | Returns collateral, debt, accrued interest |
| `health_factor` | `user` | Read | Returns health factor scaled ×100 |
| `get_protocol_stats` | — | Read | Total collateral locked, total debt, liquidation count |

### Position Status Lifecycle

```
No Position
    │
    ▼ deposit_collateral()
Collateralised (no debt)
    │
    ▼ borrow()
Active Loan
    │                  │
    ▼ repay() + full   ▼ health_factor < 120
Closed             Liquidatable
                       │
                       ▼ liquidate()
                   Liquidated (position closed)
```

---

## Dashboard Pages

| Route | Description |
|---|---|
| `/` | Landing — TVL, total borrowed, active positions, liquidation stats, how it works |
| `/borrow` | Open position — deposit XLM amount, preview max borrowable USDC, confirm + sign |
| `/positions` | My positions — health factor bar, collateral value, debt + accrued interest, repay button |
| `/liquidate` | Liquidation hunter — all positions with health_factor < 120, liquidation profit preview, execute button |

---

## Oracle Service (`oracle/`)

The oracle is a lightweight Node.js TypeScript service:

1. Polls `https://horizon.stellar.org/order_book?selling_asset_type=native&buying_asset_code=USDC` every 60 seconds
2. Calculates mid-price: `(best_bid + best_ask) / 2`
3. Scales to 7 decimal precision (Stellar native format)
4. Signs and submits `set_price(scaled_price)` to the Soroban contract
5. Logs each price update with timestamp to stdout

If the oracle keypair has insufficient XLM for fees, it logs a critical error and halts — never submits a stale price.

---

## Telegram Bot (`bot/`)

| Command | Description |
|---|---|
| `/start` | Subscribe to WaveLend alerts |
| `/health <address>` | Check health factor for any position |
| `/positions` | List all positions with health_factor < 130 (risky) |
| `/liquidate <address>` | Preview liquidation profit and execute via Freighter deeplink |
| `/stats` | Protocol TVL, total borrowed, liquidation count |
| `/stop` | Unsubscribe from alerts |

**Automated alerts:**
- `health_factor < 130` → "⚠ Your WaveLend position is at risk. Health: X. Top up collateral or repay debt."
- `health_factor < 120` → "🚨 Liquidation imminent. Health: X."
- New liquidation executed → "💥 Position liquidated. Liquidator earned +X USDC."

---

## Project Structure

```
WaveLend/
├── contract/                    # Soroban Rust contract
│   └── src/
│       ├── lib.rs               # All contract entry points
│       ├── interest.rs          # Interest accrual logic
│       ├── liquidation.rs       # Liquidation math
│       └── test.rs              # Cargo test suite
├── dashboard/                   # React / Next.js frontend
│   ├── app/                     # App Router pages
│   ├── components/
│   │   ├── PositionCard.tsx     # Health factor + debt display
│   │   ├── BorrowForm.tsx       # Deposit + borrow wizard
│   │   ├── LiquidatePanel.tsx   # Liquidation hunter UI
│   │   └── WalletConnect.tsx    # Freighter integration
│   └── lib/
│       ├── stellar.ts           # Freighter + SDK helpers
│       └── soroban.ts           # Contract call helpers
├── oracle/                      # Price feed service
│   └── src/
│       ├── index.ts             # Poll + submit price loop
│       └── stellar.ts           # Soroban transaction builder
├── bot/                         # Telegram bot
│   └── src/
│       ├── bot.ts               # Commands + alert subscriptions
│       └── monitor.ts           # Health factor polling loop
└── .env.example
```

---

## Security Considerations

- `require_auth()` enforced on all state-changing functions — no unauthorized calls possible
- Oracle keypair is the only address permitted to call `set_price()` — verified on-chain
- Reentrancy: Soroban's execution model prevents reentrancy attacks by design
- Integer overflow: all arithmetic uses `i128` checked math; contract panics on overflow
- Price staleness: if oracle has not updated within 5 minutes, the contract reverts borrow and liquidate calls with `PriceStale` error
- Liquidation threshold (120) is intentionally conservative to protect against XLM price volatility

---

## Contributing

Contributions are welcome. Open an issue first to discuss the change, then submit a PR against `main`.

**Quick rules:**
- One concern per PR — bug fixes and features in separate PRs
- For contract changes: run `cargo test` and include full output in the PR description
- No TypeScript `any` types
- `cargo clippy --deny warnings` must pass before requesting review
- Follow Conventional Commits: `fix:`, `feat:`, `docs:`, `test:`, `chore:`

---

## Stellar Testnet Resources

- [Stellar Laboratory](https://laboratory.stellar.org) — inspect accounts, transactions, contracts
- [Friendbot](https://friendbot.stellar.org) — fund a testnet account with free XLM
- [USDC on Testnet](https://developers.stellar.org/docs/tokens/usdc) — Circle USDC testnet issuer
- [Soroban Testnet RPC](https://soroban-testnet.stellar.org) — RPC for contract calls
- [Stellar Expert (Testnet)](https://stellar.expert/explorer/testnet) — block explorer

---

## Roadmap

- [ ] Multi-collateral support (EURC, AQUA alongside XLM)
- [ ] Variable interest rate model (utilisation-based)
- [ ] Governance contract — community voting on protocol parameters
- [ ] Mainnet deployment
- [ ] Formal security audit

---

## License

MIT © [wealth4ever123](https://github.com/wealth4ever123)

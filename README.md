# ldbAfrica Gateway
### Cross-Border Crypto Payment Infrastructure — Systems Architecture

> **Version:** 1.0 — Architecture (v1 Build Scope)
> **Product:** ldbAfrica
> **Classification:** Internal Engineering & Product Documentation

---

## Table of Contents

1. [System Philosophy](#1-system-philosophy)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Payment Flow — End to End](#3-payment-flow--end-to-end)
4. [Module 1: Blockchain Smart Router](#4-module-1-blockchain-smart-router)
5. [Module 2: Fiat Settlement Router](#5-module-2-fiat-settlement-router)-(v2+ consideration)
6. [Module 3: Rate Locking + FX Management](#6-module-3-rate-locking--fx-management)
7. [Module 4: Transaction Intelligence (Fraud/Risk)](#7-module-4-transaction-intelligence-fraudrisk)
8. [Module 5: Stablecoin Optimization](#8-module-5-stablecoin-optimization)
9. [Wallet Infrastructure](#9-wallet-infrastructure)
10. [RPC Provider Strategy](#10-rpc-provider-strategy)
11. [OTC Liquidity + Bank Disbursement](#11-otc-liquidity--bank-disbursement)
12. [Treasury Management](#12-treasury-management)
13. [Merchant Dashboard & Ledger](#13-merchant-dashboard--ledger)
14. [Failure Points & Mitigations](#14-failure-points--mitigations)
15. [Technology Stack](#15-technology-stack)
16. [V2 Integration Roadmap](#16-v2-integration-roadmap)

---

## 1. System Philosophy

ldbAfrica is a **stablecoin-first, cross-border crypto payment gateway** for Africa. It enables any user holding crypto or stablecoins to pay merchants in their local currency, with merchants settled in fiat regardless of what the user paid in.

**The core value chain, in one sentence:**
*User pays in stablecoin → ldbAfrica confirms on-chain → credits merchant dashboard in USD-equivalent → merchant withdraws to local fiat via OTC-to-bank settlement.*

**Design principles:**

- **Stablecoin first.** USDT (TRC-20) and USDC (Polygon, Base, Stellar) are the primary instruments. Volatile crypto is secondary.
- **Merchants are known. Users are not.** Businesses are KYB-verified entities in the database. Individual payers are anonymous — identified only by their wallet address at time of transaction.
- **Non-custodial-adjacent architecture for v1.** All logic is off-chain and service-orchestrated. Smart contract logic is scoped to v2.
- **Fragmented liquidity is a first-class concern.** Settlement in Nigeria, Kenya, and other African markets routes through regional OTC desks and licensed bank disbursement APIs — not on-chain DEX.
- **Two distinct routing problems.** The blockchain router decides how to accept and confirm on-chain payments. The fiat settlement router decides how to convert and disburse to merchants. These are separate systems with separate optimization objectives.

---

## 2. High-Level Architecture

```
[User — holds USDT/USDC on any supported chain]
          │
          │  Scans QR / opens payment widget at merchant checkout
          ▼
┌────────────────────────────────────────────────────────────────┐
│                      ldbAfrica GATEWAY                         │
│                                                                │
│  ┌─────────────────────┐    ┌──────────────────────────────┐   │
│  │  Checkout Session   │───▶│  Wallet Infrastructure Layer │   │
│  │  Manager            │    │  (HD Wallet Address Gen.)    │   │
│  └──────────┬──────────┘    └──────────────────────────────┘   │
│             │                                                  │
│             ▼                                                  │
│  ┌─────────────────────┐    ┌──────────────────────────────┐   │
│  │  Blockchain Smart   │    │  Rate Lock Engine            │   │
│  │  Router             │    │  (FX + Crypto Pricing)       │   │
│  └──────────┬──────────┘    └──────────────────────────────┘   │
│             │                                                  │
│             ▼                                                  │
│  ┌─────────────────────┐    ┌──────────────────────────────┐   │
│  │  Transaction        │    │  Stablecoin Optimization     │   │
│  │  Intelligence       │    │  Layer                       │   │
│  └──────────┬──────────┘    └──────────────────────────────┘   │
│             │                                                  │
│             ▼                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Merchant Ledger (USD-denominated)          │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                         │                                      │
│                         ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           Fiat Settlement Router                        │   │
│  │   OTC Desk RFQ → ldbAfrica Bank Account →              │   │
│  │   Bank Disbursement API → Merchant Bank Account        │   │
│  └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
          │
          ▼
[Merchant Dashboard + Webhook Callbacks]
```

---

## 3. Payment Flow — End to End

```
1. USER hits checkout on merchant's website or app
        │
2. ldbAfrica WIDGET loads
   - Calls Checkout Session Manager
   - Creates a session: { session_id, merchant_id, fiat_amount, fiat_currency }
   - Calls Rate Lock Engine → fetches live USDT/USD + USD/NGN rates
   - Computes crypto_amount_required = fiat_amount / effective_rate
   - Locks rate for 20 minutes: { rate_id, rate, expires_at }
   - Calls Wallet Infrastructure → derives unique deposit address per session
   - Returns to widget: deposit address per supported chain + crypto amount
        │
3. USER selects their chain (TRON, Polygon, Base, Stellar)
   Widget displays QR code and address for chosen chain
        │
4. USER sends crypto from their wallet
        │
5. BLOCKCHAIN ROUTER monitors all session addresses via RPC webhooks + polling
   - Detects incoming transaction
   - Runs Transaction Intelligence checks (address screening, rule-based flags)
   - Waits for required confirmation depth (chain-specific)
        │
6. ON CONFIRMATION:
   - Match tx to session by address (+amount tolerance check)
   - Verify within rate lock window
   - Apply locked rate → compute fiat_equivalent in USD
   - Credit merchant internal ledger: +$X.XX USD equivalent
   - Record: { tx_hash, chain, asset, amount_crypto, amount_usd, rate_applied, merchant_id }
   - Fire webhook to merchant system: { event: "payment.confirmed", session_id, amount_usd }
        │
7. MERCHANT sees balance update in dashboard (USD-denominated)
        │
8. MERCHANT requests withdrawal (NGN, KES, etc.)
   - Fiat Settlement Router executes:
     a. RFQ to OTC desk(s): "Quote me NGN for $X USDT"
     b. Accept best executable rate within 30s
     c. Transfer USDT from treasury to OTC desk
     d. OTC desk deposits NGN into ldbAfrica's Providus/Anchor bank account
     e. ldbAfrica Bank Disbursement API sends NGN to merchant's bank account
   - Settlement model: T+0 or T+1 depending on OTC desk and bank partner
```

---

## 4. Module 1: Blockchain Smart Router

### Responsibility

Determine which chains to support for a given checkout session, monitor those chains for incoming transactions, confirm receipt, and sweep funds to the ldbAfrica treasury wallet.

### Supported Chains and Assets (v1)

| Chain | Asset | Fee Level | Confirmation Time | Notes |
|---|---|---|---|---|
| TRON | USDT (TRC-20) | ~$0.001 | ~1 min (20 blocks) | Dominant in African OTC markets. Primary. |
| Polygon | USDC | ~$0.01–0.05 | ~1–2 min (32 blocks) | USDC preferred, PoS bridge well-established |
| Base | USDC | ~$0.01–0.05 | ~1–2 min | Coinbase L2, native USDC issuance, growing retail base |
| Stellar | USDC | ~$0.00001 | ~5 sec (1 ledger) | Ideal for cross-border; memo field eliminates address-per-session problem |

> **Design note on chain selection:** ldbAfrica does not force the user onto a chain. The widget displays a supported address for each chain. The user selects based on what they hold. The router monitors all generated addresses simultaneously and processes whichever chain the user pays on.

### Chain Monitoring Architecture

```
[Session Created]
      │
      ▼
[Generate deposit address per chain] ─────────────────────────────┐
      │                                                           │
      ▼                                                           ▼
[Register addresses with monitoring services]            [Store in DB: sessions table]
      │
      ├── TRON: TronGrid webhook subscription
      ├── Polygon: Alchemy webhook subscription
      ├── Base: Alchemy webhook subscription
      └── Stellar: Stellar Horizon SSE stream
      │
      ▼
[Fallback polling every 30s for all active sessions] ← (in case webhook missed)
      │
      ▼
[On detection → emit internal TransactionDetected event → Transaction Intelligence]
      │
      ▼
[On confirmation → emit TransactionConfirmed event → Settlement pipeline]
```

### Confirmation Depth Policy

| Chain | Required Confirmations | Time to Confirmation | Reasoning |
|---|---|---|---|
| TRON | 20 blocks | ~1 minute | TRON has had shallow-depth reorgs historically |
| Polygon | 32 blocks | ~1.5 minutes | High confidence before checkpoint |
| Base | 32 blocks | ~1 minute | OP Stack finality window |
| Stellar | 1 ledger close | ~5 seconds | Stellar has deterministic finality — 1 close = final |

### Fund Sweep to Treasury

After confirmation, funds sit in the session's deposit address. They must be swept to the ldbAfrica treasury wallet.

- **TRON:** Sweep immediately post-confirmation. Gas ~$0.001. Simple transfer.
- **Polygon/Base:** Queue sweeps in BullMQ. Execute when gas price below threshold (e.g. <30 gwei). Batch multiple sweeps per block where possible.
- **Stellar:** No "sweep" needed — Stellar deposit addresses are merge-capable. Funds can sit until daily consolidation.

Each sweep wallet needs a gas float in native token (TRX, MATIC, ETH on Base). An auto-refill bot monitors gas wallet balances and tops up from a reserve wallet when below threshold.

---

## 5. Module 2: Fiat Settlement Router(v2+ consideration)

### Responsibility

Given a confirmed on-chain receipt for merchant X requiring Y fiat in country Z, determine the optimal path to disburse that fiat — minimizing cost, time, and settlement risk.

### The Routing Problem

This is a linear allocation problem across liquidity sources. At v1 with limited OTC partners, it is implemented as a rules-based priority system. The mathematical framework scales to full optimization at v2.

**Variables per liquidity source i:**

| Variable | Definition |
|---|---|
| Rᵢ | Exchange rate offered (e.g. NGN per USDT) |
| Fᵢ | Total fees (percentage + fixed) |
| Tᵢ | Settlement time (T+0 = 0, T+1 = 1) |
| Lᵢ | Available liquidity (max they can settle this batch) |
| σᵢ | Reliability score (historical success rate, failure risk) |

**Objective — minimize total cost per settlement:**

```
Minimize: Σ xᵢ · (Fᵢ + λ·Tᵢ + γ·σᵢ)
Subject to: Σ xᵢ = total_settlement_amount
            xᵢ ≤ Lᵢ  for all i
            xᵢ ≥ 0   for all i
```

Where λ penalizes settlement delay and γ penalizes source risk. At v1, λ and γ are manually tuned constants. At v2, these become learned parameters.

**v1 Implementation (rules-based approximation):**

1. Send simultaneous RFQ to all available OTC desks for the required currency/amount.
2. Accept best rate quote within 30-second window.
3. If no single desk can cover full amount, split across top two desks (partial fill).
4. If no desk can settle T+0, hold in queue and retry T+1 batch.

### Settlement Flow

```
[Merchant withdrawal request: ₦500,000]
          │
          ▼
[Fiat Settlement Router]
  ├── RFQ to Yellow Card: "Quote NGN for $385 USDT"
  ├── RFQ to AZA Finance: "Quote NGN for $385 USDT"
  └── RFQ to Desk C (backup)
          │
          ▼
[Accept best executable rate within 30s]
          │
          ▼
[Transfer USDT from ldbAfrica treasury to winning OTC desk]
          │
          ▼
[OTC desk deposits ₦500,000 into ldbAfrica Providus account]
          │
          ▼
[ldbAfrica Disbursement API (via Anchor/Mono) sends ₦500,000 to merchant bank]
          │
          ▼
[Merchant ledger: debit $385 USD equivalent]
[Webhook to merchant: { event: "withdrawal.completed", amount_ngn: 500000 }]
```

### Settlement Models by Merchant Tier

| Tier | Settlement | Criteria |
|---|---|---|
| Standard Merchant | T+1 daily batch | Default for all v1 merchants |
| Verified Partner | T+0 (same day) | High volume, long relationship, pre-negotiated |

---

## 6. Module 3: Rate Locking + FX Management

### Rate Lock Strategy

ldbAfrica locks the exchange rate at **checkout session creation** — the moment the user opens the payment widget — not after on-chain confirmation. This gives users a fixed price to pay and gives merchants a guaranteed fiat amount.

**Rate lock parameters:**

```json
{
  "rate_id": "uuid",
  "session_id": "uuid",
  "merchant_id": "uuid",
  "crypto_asset": "USDT",
  "crypto_amount": 384.62,
  "fiat_currency": "NGN",
  "fiat_amount": 500000,
  "effective_rate": 1300,
  "created_at": "2025-01-15T10:00:00Z",
  "expires_at": "2025-01-15T10:20:00Z",
  "status": "ACTIVE"
}
```

Rate lock window: **20 minutes.** If no on-chain transaction is confirmed within this window, the session expires. The user must initiate a new payment.

### OTC Rate Window Mismatch

OTC broker rate quotes are valid for 5–10 minutes. Our lock window is 20 minutes. **We do not lock with the OTC broker at session initiation.** Instead:

- The rate locked with the user is based on: live market price + ldbAfrica spread buffer (minimum 1.5%)
- The 1.5% spread absorbs USDT/NGN movement during the 20-minute window (historical 20-minute volatility for USDT/NGN is typically <0.5% in non-shock conditions)
- When the on-chain transaction confirms, we go to the OTC desk at that moment for the executable rate
- Our spread is the guaranteed margin between our locked rate and the OTC execution rate

For transactions above $5,000 equivalent, an additional 0.2% buffer is applied given larger absolute slippage exposure.

### Price Data Sources

**Crypto pricing (USDT/USDC to USD):**

| Source | Role | Notes |
|---|---|---|
| Binance REST API (`GET /api/v3/ticker/price`) | Primary | Free, real-time, tight spreads |
| CoinGecko API | Fallback | Free tier, 30 req/min cap |

Prices are cached in Redis with a 15-second TTL. Price feeds run as a continuous background service — never called synchronously during session creation.

**FX pricing (USD to local currency):**

| Source | Role | Notes |
|---|---|---|
| OTC broker executable rate | Ground truth for settlement | This is the rate we can actually transact at |
<!-- | Open Exchange Rates API | Mid-market reference | Used for display and accounting only | -->

> The OTC broker's bid includes their spread. ldbAfrica's locked rate must be based on the OTC executable quote including a 1.5% buffer in the negative direction to cover for fx fluctuations. 

### Stablecoin Accounting Rate

USDT and USDC trade at $0.9993–$1.0002 in normal conditions. ldbAfrica does not book receipts at exactly $1.00.

**Policy:** Book all stablecoin receipts at the **24-hour VWAP price from Binance** at time of confirmation. Deviations greater than 0.5% from $1.00 are flagged as accounting exceptions for manual review.

**Depeg circuit breaker:** A background monitor polls USDT/USDC ratio every 60 seconds. If USDT trades below $0.995 for more than 15 minutes, USDT acceptance is automatically paused until the price recovers. This protects against systemic stablecoin events (e.g. the 2022 3AC/LUNA contagion).

---

## 7. Module 4: Transaction Intelligence (Fraud/Risk)

### Scope (v1)

v1 implements address screening + a rule-based risk engine. All transaction data is logged from day one to build the dataset for v2 ML models.

### Data Captured Per Transaction

Every event is logged for analytics and future model training:

```
transaction_id, session_id, merchant_id
payer_address, chain, asset
amount_crypto, amount_usd
session_created_at, first_seen_on_chain_at, confirmed_at
time_to_pay (session created → on-chain tx)
amount_deviation (did user send exact amount?)
payer_address_age (first tx date of payer's wallet)
payer_address_prior_tx_count
is_contract_address (boolean)
flagged (boolean), flag_reason, action_taken
```

### Address Screening (Pre-Credit — Mandatory)

Every inbound transaction is screened against sanctions lists and threat intelligence feeds **before funds are credited to merchant ledger.**

| Provider | Role | Notes |
|---|---|---|
| **TRM Labs** | Primary screening (v1) | Strong Africa/EMEA coverage, API-first |
| **AMLBot** | Backup / cost-effective alternative | Good for bootstrapping |
| **Chainalysis KYT** | Upgrade path (v2+) | Industry standard, enterprise pricing |
| **OFAC SDN List** | Mandatory sanctions check | Free, must be checked on every address |

### Rule-Based Risk Engine

| Rule | Trigger | Action |
|---|---|---|
| Overpayment | Received > expected + 1% | Hold; alert ops; do not auto-credit |
| Underpayment | Received < expected − 1% | Do not credit; open exceptions queue |
| Sanctioned address | Address on OFAC SDN / TRM Labs high-risk | Hard block; log; do not process |
| Mixer / darknet tag | Address tagged by threat intelligence | Hold; manual review required |
| Structuring signal | 3+ small transactions to same merchant within 60 minutes | Flag; alert compliance |
| Expired session payment | Funds arrive after 20-min lock window | Do not auto-credit; route to exceptions queue |
| Duplicate tx hash | Same tx_hash seen twice (replay attempt) | Reject; idempotency enforced at DB level |
| Wrong asset received | ERC-20 other than USDC/USDT received | Hold; alert; do not credit |

### Exceptions Queue

All held transactions are routed to an internal exceptions queue for manual ops review. Ops team has 24 hours to resolve. Resolution options: approve, refund, escalate to compliance.

---

## 8. Module 5: Stablecoin Optimization

### Accepted Stablecoins (v1)

| Stablecoin | Chain | Priority | Reason |
|---|---|---|---|
| USDT | TRON (TRC-20) | Primary | Dominant in African OTC and P2P markets. OTC desks price in USDT/TRC-20. |
| USDC | Polygon | Secondary | Regulated, Circle-issued, preferred for institutional flows |
| USDC | Base | Secondary | Growing retail base, Coinbase integration path |
| USDC | Stellar | Secondary | Cross-border corridor play, SEP-24 anchor ecosystem |

### Internal Accounting

All internal accounts and the merchant ledger are denominated in **USD equivalent**, not in any specific stablecoin. The stablecoin received is recorded as metadata. This means:

- Merchant sees their balance in USD (e.g. $385.00)
- The system tracks the underlying stablecoin for treasury management
- Withdrawal converts USD-equivalent balance to local fiat at execution-time OTC rate

### FX Optionality for Merchants

Merchants holding USD-denominated balances benefit from naira depreciation if they delay withdrawal — their USD balance converts to more NGN over time as the exchange rate moves. This is a feature, not a bug, and should be surfaced in the merchant dashboard as: *"Current estimated NGN value of your balance: ₦X"* (live, updated via FX feed).

ldbAfrica does not hold the FX risk — the OTC desk executes at the spot rate at withdrawal time. ldbAfrica's margin is the spread between the locked rate at receipt and the OTC rate at withdrawal.

### Depeg Risk Management

- Monitor USDT/USDC spread continuously (60-second polling)
- If USDT depegs > 0.5% from $1.00 for > 15 minutes: pause USDT acceptance
- Do not hold stablecoin treasury balances beyond 48 hours of average daily volume
- Target same-day OTC settlement for all confirmed receipts

---

## 9. Wallet Infrastructure

### Design Decision: One Address Per Checkout Session

ldbAfrica generates a unique deposit address per checkout session using HD wallet derivation. This is the only reliable way to attribute an incoming transaction to a specific session on chains without memo fields (TRON, Polygon, Base).

**Derivation path:**
```
m / 44' / coin_type' / merchant_index' / session_index
```

Where `coin_type` follows BIP-44 standards (195 for TRON, 60 for EVM chains) and `session_index` is an auto-incrementing integer per merchant, stored in the database.

**This is not wasteful.** HD wallet derivation is pure cryptographic computation — there is no on-chain cost to generating an address. The only operational cost is the gas required to sweep funds from session addresses back to the treasury wallet after confirmation.

### Stellar: Single Address Per Merchant

Stellar has a native `memo` field on every transaction. For Stellar USDC:

- Each merchant gets one Stellar address (not one per session)
- Each session is assigned a unique memo: `session_id` (truncated to Stellar memo limits)
- The widget instructs users: "Send USDC to [address] with memo [session_id]"
- ldbAfrica identifies the session by memo, not address

This reduces Stellar wallet management complexity significantly.

### Fund Sweep Strategy by Chain

| Chain | Sweep Trigger | Gas Cost | Strategy |
|---|---|---|---|
| TRON | Immediately post-confirmation | ~$0.001 | Sweep on confirm. Negligible cost. |
| Polygon | Queued; execute when gas < 30 gwei | ~$0.01–0.10 | BullMQ queue; batch multiple sweeps per block |
| Base | Queued; execute when gas < 30 gwei | ~$0.01–0.05 | Same as Polygon |
| Stellar | Daily consolidation | ~$0.00001 | No urgency; batch end-of-day |

### Gas Float Management

Each chain's sweep wallet requires a native token balance to pay gas. A gas float bot:
1. Monitors native token balance on all sweep wallets every 5 minutes
2. If balance drops below minimum threshold (e.g. 1 MATIC, 10 TRX): triggers auto-transfer from gas reserve wallet
3. Alerts ops if gas reserve wallet drops below 7-day projected usage

### v2: CREATE2 Deterministic Contract Addresses

See [V2 Integration Roadmap](#16-v2-integration-roadmap).

---

## 10. RPC Provider Strategy

### Provider Assignments

| Chain | Primary | Fallback |
|---|---|---|
| Polygon | Alchemy (Webhooks) | QuickNode |
| Base | Alchemy (Webhooks) | QuickNode |
| TRON | TronGrid (official) | NOWNodes |
| Stellar | Stellar Horizon (SDF-hosted) | Self-hosted Horizon (v2) |

### Monitoring Pattern

Primary: **Webhook subscription** — provider pushes notification when funds arrive at monitored address.

Fallback: **Polling every 30 seconds** — ldbAfrica polls all active session addresses in case webhook was dropped.

Both run in parallel. The system deduplicates by `tx_hash` — the same transaction arriving via webhook and polling is processed exactly once (idempotency enforced at DB level).

### Health Check

An RPC health check service pings all providers every 60 seconds. If a provider fails 3 consecutive checks:
1. All new sessions for that chain route to the fallback provider
2. Ops alert fires
3. Polling frequency increases to 10 seconds for active sessions on that chain

---

## 11. OTC Liquidity + Bank Disbursement

### Architecture

ldbAfrica does not use OTC brokers to directly disburse to merchants. OTC brokers are used purely for **liquidity conversion** — swapping USDT treasury for local fiat into ldbAfrica's own bank account. Merchant disbursement is then handled separately via bank disbursement API.

```
ldbAfrica USDT Treasury
          │
          ▼  (USDT → NGN conversion)
OTC Broker (Yellow Card / AZA Finance)
          │
          ▼  (NGN deposited into ldbAfrica account)
ldbAfrica Providus / Anchor Bank Account
          │
          ▼  (disbursement to merchant)
Bank Disbursement API (Anchor BaaS / Mono)
          │
          ▼
Merchant's Bank Account (any Nigerian bank)
```

### OTC Partner Profiles

| Partner | Regions | API | Notes |
|---|---|---|---|
| **Yellow Card Business** | Nigeria, Kenya, Ghana, Rwanda + more | Yes | Largest licensed African crypto-fiat ramp. B2B API. Primary partner. |
| **AZA Finance** | Nigeria, Kenya, Uganda, Tanzania, Senegal | Yes | Enterprise FX/OTC. Founded 2013. Strong institutional relationships. Secondary partner. |

### OTC Operational Model

- Pre-negotiate volume tiers with OTC partners. Higher volume = tighter spread.
- Maintain two OTC desks per region. Never depend on a single desk.
- Execute RFQ simultaneously across both desks; take best rate.
- Do not pre-fund OTC desks with large float. Execute spot at settlement time.
- Log all quotes (not just executed ones) for rate analysis and partner performance tracking.

### Bank Disbursement (Nigeria)

| Provider | Role | Notes |
|---|---|---|
| **Anchor** | BaaS — single API over multiple banks | Recommended. Handles accounts, transfers, compliance in one integration. |
| **Mono** | Payment initiation + account data | Good fallback; strong developer experience |
| **Providus Bank** | Direct bank partner | Fintech-friendly, common treasury account choice |

---

## 12. Treasury Management

### Core Rules

**Rule 1 — Float ceiling:** Never hold stablecoin treasury balances exceeding 48 hours of average daily volume. Limits depeg exposure. At $50k/day volume: max treasury float = $100k USDT.

**Rule 2 — Low-watermark alert:** If treasury balance drops below 2× the largest pending merchant withdrawal, trigger an ops alert. Ensures sufficient liquidity to execute all pending settlements.

**Rule 3 — Daily settlement batch:** All confirmed on-chain receipts from 00:00–23:59 UTC are aggregated per settlement currency. One bulk OTC trade is executed per region per day (unless T+0 withdrawals require intra-day execution).

**Rule 4 — Multi-desk allocation:** For single batches exceeding any one OTC desk's available liquidity, split across multiple desks using the Fiat Settlement Router's allocation logic.

**Rule 5 — Stablecoin diversification:** Maintain treasury in both USDT and USDC. Do not hold >70% in either. USDT preferred for NGN corridor (OTC desk preference); USDC preferred for institutional/cross-border flows.

**Rule 6 — Gas reserve:** Maintain a dedicated gas reserve wallet per chain. Never use treasury funds for gas. Gas reserve is a pure operational cost, tracked separately in P&L.

---

## 13. Merchant Dashboard & Ledger

### Ledger Design

Merchant balances are **USD-denominated**, not stablecoin-denominated. The merchant sees dollars, not USDT. The underlying stablecoin is a treasury/accounting detail.

```
Merchant Ledger Entry:
{
  "event": "payment.received",
  "session_id": "uuid",
  "tx_hash": "0x...",
  "chain": "POLYGON",
  "asset_received": "USDC",
  "amount_crypto": 384.62,
  "rate_applied": 1.0,
  "amount_usd_credited": 384.62,
  "timestamp": "2025-01-15T10:18:34Z"
}
```

### Dashboard Features (v1)

- **Balance:** Current USD-equivalent balance + estimated local fiat value at live OTC rate
- **Transaction history:** All confirmed payments with tx hash, chain, amount, time
- **Withdrawal:** Request fiat disbursement; select bank account; view T+0/T+1 status
- **Webhook logs:** History of all webhook events fired, status, retries
- **API keys:** Merchant manages their integration credentials

### Webhook Delivery

- Signed with HMAC-SHA256 using per-merchant secret key
- Merchants must verify signature on receipt
- Retry policy: exponential backoff at 1min, 5min, 15min (3 attempts total)
- Merchants must handle duplicate deliveries idempotently (webhook may deliver more than once on retry)

---

## 14. Failure Points & Mitigations

### Technical

| Failure | Likelihood | Impact | Mitigation |
|---|---|---|---|
| RPC provider downtime | Medium | High | Multi-provider with auto-failover; polling fallback |
| Webhook dropped / missed | Medium | Medium | Polling runs in parallel with webhooks at all times |
| Double-credit on retry | Low | High | Idempotency key = `tx_hash`; unique constraint at DB level |
| Amount collision (two sessions, same amount, same merchant) | Low | Medium | One unique address per session eliminates this entirely |
| Rate lock race condition | Medium | Medium | DB-level row lock on `rate_lock` table during session match |
| Private key compromise | Very Low | Critical | Fireblocks or AWS KMS for treasury keys; raw keys never stored in DB |
| Gas wallet drained | Medium | Medium | Gas float bot with auto-refill and low-balance alerts |
| Stablecoin depeg event | Low | High | 60-second price monitor; auto-pause on depeg > 0.5% |
| Chain reorganization | Low | High | Enforce minimum confirmation depth per chain before crediting |
| Sweep failure (gas spike) | Medium | Low | Sweep queue; execute only when gas below threshold; never blocks settlement |
| Payer sends wrong asset | High (common UX error) | Medium | Detect unsupported asset; route to exceptions queue; refund flow |
| Expired session payment | High (common) | Medium | Funds detected but rate lock expired → exceptions queue → ops review |
| OTC desk rejects large trade | Medium | High | Pre-negotiate volume tiers; maintain two desks per region |

### Compliance & Regulatory

| Failure | Risk Level | Mitigation |
<!-- |---|---|---|
| Operating without VASP registration | Critical | Register with SEC Nigeria as Virtual Asset Service Provider before launch |
| CBN regulatory action | High | Structure service as stablecoin B2B technology (not crypto exchange); maintain legal opinion per market | -->
| OFAC sanctions violation | Critical | Mandatory address screening on every inbound transaction before crediting |
| AML gap on anonymous payers | High | Address screening is the substitute for payer KYC; document this in merchant ToS |
| Transaction reporting threshold breach | Medium | Auto-generate CTR equivalent for transactions > $10,000; consult local legal on specific thresholds |

### Operational

| Failure | Risk Level | Mitigation |
|---|---|---|
| OTC desk insolvency | Low | Never pre-fund; execute spot; keep <48h volume in treasury exposure per desk |
| Merchant disputes settlement rate | Medium | Rate lock records are immutable; exposed in merchant dashboard with tx_hash proof |
| Underpayment (user sends less) | High | Configurable tolerance ± 1%; below tolerance → do not credit; exceptions queue |
| Overpayment (user sends more) | Medium | Detect and hold excess; refund to sending address; log for ops review |
| Bank disbursement API failure | Medium | Fallback to secondary bank partner; queue and retry |

---

## 15. Technology Stack

| Layer | Technology | Role |
|---|---|---|
| **Backend API** | Python (FastAPI) | Core gateway services; rate locking; session management |
| **Task Queue** | Redis + BullMQ | Sweep queue; webhook retry; settlement batch |
| **Database** | PostgreSQL | Sessions, ledger, merchants, rate locks, audit log |
| **Cache** | Redis | Price feed cache (15s TTL); session rate locks |
| **Chain Monitoring — EVM** | Alchemy Webhooks | Polygon, Base transaction detection |
| **Chain Monitoring — TRON** | TronGrid Webhooks | TRON transaction detection |
| **Chain Monitoring — Stellar** | Stellar Horizon SSE | Stellar payment stream |
| **Polling Fallback** | Custom service | 30-second polling for all active sessions |
| **Crypto Pricing** | Binance REST API | Primary; 15s cached |
| **Crypto Pricing Fallback** | CoinGecko API | Fallback |
| **FX Pricing** | Open Exchange Rates | Mid-market reference |
| **Key Management** | AWS KMS (v1) → Fireblocks (v2) | Treasury wallet signing; no raw private keys in DB |
| **Address Screening** | TRM Labs API | Per-transaction; mandatory before crediting |
| **Blockchain SDKs** | ethers.js (EVM), tronweb (TRON), stellar-sdk (Stellar) | Chain interaction |
| **Wallet Derivation** | bip32 + bip39 | HD wallet address generation |
| **Bank Disbursement** | Anchor BaaS | NGN disbursement to merchant bank accounts |
| **OTC — Nigeria** | Yellow Card Business API | Primary NGN liquidity |
| **OTC — Nigeria (backup)** | AZA Finance API | Secondary NGN liquidity |
| **Monitoring & Alerts** | Grafana + Prometheus | RPC health, treasury balance, depeg alerts, queue depth |
| **Merchant Dashboard** | Next.js (React) | Transaction history, balance, withdrawals, webhooks |

---

## 16. V2 Integration Roadmap

These features are explicitly out of scope for v1 and are documented here for architectural continuity.

### CREATE2 Deterministic Contract Addresses (Wallet Infrastructure)

**Problem it solves:** The HD wallet sweep model requires gas per session address to collect funds. At high transaction volume on Ethereum-equivalent chains, sweep costs compound.

**How CREATE2 works:**

A factory contract is deployed once. For each payment session, ldbAfrica computes a deterministic address using:
```
address = keccak256(0xFF || factory_address || salt || keccak256(bytecode))[12:]
```
where `salt = keccak256(merchant_id || session_id)`.

The address is pre-computed off-chain and given to the user. The forwarder contract is only deployed (and immediately auto-executes a transfer to treasury) when funds actually arrive — meaning you pay deployment gas only on successful payments, not for every session created.

**Target chains for v2:** Polygon, Base, Ethereum. TRON gas is negligible so HD sweep remains optimal there.

**Reference implementations:** OpenZeppelin MinimalForwarder, Gnosis Safe payment receiver.

### ML-Based Transaction Intelligence

**Phase 1 — Anomaly Detection:** Replace rule-based thresholds with unsupervised anomaly detection (Isolation Forest, Autoencoder) trained on historical transaction features. Reduces false positive rate; catches novel fraud patterns not covered by fixed rules.

**Phase 2 — Risk Scoring Model:** Supervised classifier (XGBoost or LightGBM) trained on labeled fraud/non-fraud outcomes from exceptions queue. Outputs a continuous risk score (0–1) per transaction. Risk score replaces binary flag/pass logic.

**Phase 3 — Graph-Based Analysis:** Wallet graph analytics — detect clusters of addresses that frequently co-appear in transactions with flagged wallets. Catches coordinated structuring and mixer-adjacent behavior that per-transaction rules miss.

**Data requirement:** Minimum ~6 months of transaction history with labeled outcomes before Phase 1 is viable. Logging schema is built to support this from day one.

### Smart Contract Settlement Layer

Move rate lock and settlement confirmation logic on-chain. Enables trustless settlement proofs, on-chain dispute resolution, and composability with DeFi liquidity sources. Requires deployment of audited smart contracts on Polygon/Base and legal review of smart contract custody implications per jurisdiction.

### Real-Time FX Hedging

For large transactions (>$10,000 equivalent), open an offsetting spot position on Binance/OKX at session initiation to lock in the USDT/USD leg while the OTC desk manages USD/NGN. Reduces spread risk on high-value transactions. Requires Binance/OKX institutional API access and treasury hedging policy.

### Yield on Idle Stablecoin

Deploy idle USDC in treasury to Circle Yield or conservative on-chain protocols (Aave, Compound) for 4–6% APY. Requires legal opinion on treatment of yield on merchant-equivalent balances per jurisdiction. Not to be implemented until regulatory clarity is confirmed.

### Expanded Chain Support

| Chain | Asset | Rationale |
|---|---|---|
| BNB Smart Chain | USDT/USDC | Large African retail crypto user base on BSC |
| Solana | USDC | Fast finality, growing USDC ecosystem |
| Ethereum (large tx only) | USDC | Institutional flows; regulatory clarity |

### Additional Settlement Corridors

Ghana (GHS), Uganda (UGX), Tanzania (TZS), South Africa (ZAR). Each corridor requires a licensed OTC partner and bank disbursement integration specific to that market.

---

*Document version: 1.0 — v1 Architecture*
*Product: ldbAfrica*
*Audience: Engineering team, Product Manager*
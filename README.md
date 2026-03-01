# AnoBank

App: https://anobank.netlify.app/
Bank Dashboard: https://illiliilillilililillliliiilil-anobank.netlify.app/

### Privacy-First On-Chain Banking Infrastructure

> **The first on-chain bank.** White-label infrastructure combining on-chain security with Unlink privacy, wrapped in a neobank UX — zero crypto jargon.

Deployed on **Monad Testnet**

---

## Table of Contents

- [Project Overview](#1-project-overview)
- [Architecture](#2-architecture-overview)
- [Payment Flows](#3-payment-flows)
- [Deposit & Yield](#4-deposit-flow--yield-generation)
- [Smart Contracts](#5-smart-contracts)
- [Backend Services](#6-backend-services)
- [Data Model](#7-data-model)
- [Event Indexer](#8-event-indexer-envio)
- [Frontend Interfaces](#9-frontend-interfaces)
- [Deployment](#10-deployment)
- [Security Model](#11-security-model)

---

## 1. Project Overview

AnoBank solves the two fundamental problems of blockchain-based finance: **total transparency** (all transactions and balances are public, exposing clients to high risk) and **no banking controls** (crypto wallets lack sub-accounts, spending limits, co-signers, and compliance checks).

AnoBank delivers the best of both worlds by combining on-chain security with Unlink privacy, wrapped in a neobank UX that looks and feels like N26 or Revolut.

The infrastructure is designed as a **white-label solution**: any bank or financial institution can deploy the full stack under its own brand. AnoBank is also a **full-scale banking infrastructure** — only 10% of aggregated user deposits stay in the Unlink spending pool for instant payments, while 90% is deployed into yield-generating DeFi strategies via a dedicated Trading Desk.

### Four Core Properties

| Property | Description |
|---|---|
| **Privacy by Design** | Transactions are private via Unlink protocol. Origin and destination are unlinkable on-chain. |
| **Neobank UX** | Looks like N26 or Revolut: sub-accounts, cards, yield displayed natively. Zero crypto jargon. |
| **On-chain Security** | 2/2 multisig per account: 1 client key (PIN/password) + 1 bank co-sign. Guardian smart contract enforces guardrails. |
| **DeFi Integrated** | Access DeFi protocols (Aave, Uniswap) to generate yield or trade directly from the banking app. |

---

## 2. Architecture Overview

### 2.1 Design Principles

- **Authorization ≠ Execution.** Validating that a payment is allowed (on-chain SpendInteractor) is separate from moving the funds (Unlink private transfer). This separation is the core innovation.
- **Privacy where it matters.** External payments are private (Unlink). Internal controls are transparent (on-chain). Regulators can verify enforcement rules independently.
- **Capital stays productive.** Funds live in M1 Treasury (yield strategies) until the moment they're needed. The Unlink spending pool holds only 10% of total deposits.
- **No omnibus accounting for DeFi.** When Tier B clients do DeFi, their M2 Safe owns the position (aTokens, LP tokens). No off-chain bookkeeping to track shared pool shares.

### 2.2 Three-Tier Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                   M1 — TREASURY SAFE (3/5)                  │
│        90% of deposits → yield strategies (Aave, Morpho)    │
│        Timelocked withdrawals · Role-based access (xVault)  │
└──────────────┬──────────────────────────────────────────────┘
               │ JIT funding              
               ▼                          
┌──────────────────────────┐  ┌───────────────────────────────┐
│   UNLINK SPENDING POOL   │  │   M2 SAFE (Payement & DeFi)   │
│   10% of deposits        │  │   2/2 multisig per client     │
│   Bank-controlled        │  │   Can holds DeFi allocation   │
│   All payments exit here │  │   routed by Unlink            │
└──────────────▲───────────┘  └───────────────────────────────┘
               │ backend executes after on-chain auth
┌──────────────┴──────────────────────────────────────────────┐
│              ON-CHAIN AUTHORIZATION LAYER                   │
│                                                             │
│  M2-1 (2/2: user + bank)      M2-2 (2/2: user + bank)       │
│   ├── EOA-a (card: €500/day)   ├── EOA-c (card: €1000/day)  │
│   └── EOA-b (transfers)        └── EOA-d (DeFi — Tier B)    │
│                                                             │
│  SpendInteractor per M2:                                    │
│   • 24h rolling spend tracking per EOA                      │
│   • Emits SpendAuthorized event on success                  │
│   • DOES NOT move funds — authorization only                │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 Client Tier System

| | **Tier A — Standard** | **Tier B — DeFi** |
|---|---|---|
| **M2 Funds** | Zero spending funds — M2 is authorization-only | DeFi allocation only — owns aTokens, LP NFTs on-chain |
| **Spending Path** | EOA → SpendInteractor → event → backend → Unlink withdrawal | Same as Tier A + M2 direct DeFi via DeFiInteractor |
| **Features** | Transfers, payments, daily limits | Transfers + Swap, LP, Lending (Aave, Uniswap) |
| **Yield** | Native APY generated by bank treasury strategies | Native APY + personal DeFi yield from M2 positions |

---

## 3. Payment Flows

### 3.1 Path A — Debit Card (Daily Operations)

The primary spending path for all users. The user taps "Pay" in the banking app:

```
┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  DEBIT CARD  │───▶│ POLICY CONTRACT  │───▶│  BANK BACKEND    │───▶│ PRIVATE OUTPUT   │
│              │    │                  │    │                  │    │                  │
│ User taps    │    │ SpendInteractor  │    │ Watcher picks up │    │ Unlink privacy   │
│ "Pay €500"   │    │ checks daily     │    │ event. Verifies  │    │ layer: origin ≠  │
│ Enters PIN   │    │ limit, transfer  │    │ policy. Checks   │    │ destination.     │
│              │    │ type, nonce.     │    │ risk flags.      │    │                  │
│ Signs TX     │    │                  │    │                  │    │ €500 received    │
│ under the    │    │ Emits            │    │ Calls Unlink SDK │    │ by recipient.    │
│ hood.        │    │ SpendAuthorized  │    │ to trigger       │    │                  │
│              │    │ or reverts.      │    │ withdrawal.      │    │ No link to       │
│ No crypto UX │    │                  │    │                  │    │ user on-chain.   │
└──────────────┘    └──────────────────┘    └──────────────────┘    └──────────────────┘
```

**Technical detail:** the backend stores the card's private key server-side (`CardEoa` table). When the user initiates a payment, the backend signs `authorizeSpend(amount, recipientHash, transferType)` from the card EOA. The SpendInteractor validates the 24h rolling spend window and emits a `SpendAuthorized` event. The watcher service applies a random timing decorrelation delay (2–30s) before executing the Unlink withdrawal for privacy.

### 3.2 Path B — Direct Transfer (Checking Account)

For larger transfers that don't go through the debit card path:

1. **User signs a transfer intent** (amount, recipient, token) from their signer EOA.
2. **Bank auto-sign service** validates via policy engine: balance check, velocity limits ($50k/7d, $150k/30d), $10k auto-sign cap, on-chain daily limit.
3. **If approved:** bank calls `unlink.withdraw()` directly → funds sent from pool to recipient, user ledger debited.
4. **If rejected:** intent marked `pending_review` for manual confirmation (phone call or out-of-band verification).

All approve/reject decisions are recorded to the immutable **AuditLog** for compliance.

### 3.3 On-Chain Policy Before Execution

The spending limit and authorization policy is enforced entirely on-chain **before** triggering any transaction on the backend or Unlink pool. The SpendInteractor contract validates every operation at the smart contract level first (24h rolling window, transfer type bitmap, per-EOA limits), and only then emits the authorization event that the backend acts on. This eliminates blind signing and ensures guardrails are always respected regardless of the privacy layer.

---

## 4. Deposit Flow & Yield Generation

### 4.1 Deposit Pipeline

When a user deposits funds:

1. User calls `POST /deposit/prepare` with depositor address, token (USDC), and amount.
2. Backend prepares an Unlink deposit and returns the transaction calldata.
3. User submits the on-chain transaction, then calls `POST /deposit/confirm` with the `relayId`.
4. **90/10 split:** the backend automatically sweeps 90% of the deposit to the M1 Treasury Safe and retains 10% in the Unlink pool as spending liquidity.
5. User's internal ledger balance is credited (18-decimal USD, converted from token decimals).

Each deposit is identified by a `relayId` from Unlink and an optional `accountNumber` (public-facing bank account number starting at 100000) for cross-user deposits.

### 4.2 Native Bank Yield

```
┌──────────────────┐    ┌───────────────────────┐    ┌──────────────────────┐
│     UNLINK       │    │    TRADING DESK       │    │     DeFi YIELD       │
│                  │    │                       │    │                      │
│  10% of deposits │◀──▶│  Holds 90% of         │───▶│  Selected DeFi       │
│                  │    │  deposits             │    │  Protocols           │
│  ✓ Allow instant │    │                       │◀───│                      │
│    payments      │    │  ✓ Traders have       │    │  Grows treasury      │
│                  │    │    sub-accounts with  │    │  reserves            │
│  Ensure there is │    │    specific spending  │    │         ↓            │
│  always enough   │    │    limits             │    │  ~X% Native APY      │
│  in Unlink       │    │                       │    │  for retail accounts │
└──────────────────┘    └───────────────────────┘    └──────────────────────┘
```

### 4.3 Pool Management (JIT Funding)

The `pool.ts` service runs a periodic balance monitor:

- **Low-water mark:** when pool balance drops below threshold → auto top-up from funder wallet with ERC-20 approval + Unlink deposit flow.
- **High-water mark:** when pool balance exceeds cap → sweep excess back to M1 Safe via Unlink withdrawal.
- **Configurable interval:** checks every 60s by default (`POOL_CHECK_INTERVAL_MS`).
- All top-ups and sweeps are logged to the audit trail.

---

## 5. Smart Contracts

All contracts are written in Solidity ^0.8.20, use the Zodiac Module pattern (inherited from the Multisubs project), and are built with Foundry.

### 5.1 Contract Registry

| Contract | Purpose | Heritage |
|---|---|---|
| **SpendInteractor** | Authorization-only module for M2 Safes. Validates spending intents, emits `SpendAuthorized` events. Does NOT move funds. | New — AnoBank |
| **DeFiInteractor** | Zodiac Guard for Tier B DeFi. Protocol/selector allowlists, Acquired Balance Model, percentage-based limits. | Multisubs |
| **IntEOA** | EOA extension module. Lets registered sub-accounts execute through M2 Safe to whitelisted targets. | Multisubs |
| **TreasuryVault (xVault)** | Role-based access for M1 Safe. Operator (<€10k), Manager (<€100k), Director (unlimited). Whitelist + reserve requirements. | New — AnoBank |
| **TreasuryTimelock** | Time-delay enforcement for M1. Operations above USD threshold require configurable delay (up to 7 days). | New — AnoBank |
| **M2 Safe** | Standard Gnosis Safe with 2/2 threshold (user + bank). Modules: SpendInteractor, IntEOA, DeFiInteractor (Tier B). | Gnosis Safe |

### 5.2 SpendInteractor — Deep Dive

The most critical new contract in the AnoBank architecture. Implements authorization-only spending validation:

- **24h rolling spend window:** gas-efficient checkpoint tracking with `SpendRecord[]` per EOA. Iterates backwards from most recent record. Automatic cleanup via advancing start index.
- **Transfer type bitmap:** each EOA has a bitmask of allowed types (`0`=payment, `1`=transfer, `2`=interbank). Validated per-transaction.
- **Nonce-based replay protection:** monotonically increasing global nonce, emitted in every `SpendAuthorized` event for backend deduplication.
- **Emergency controls:** `Pausable` + `ReentrancyGuard` on `authorizeSpend()`.
- **Gas safety:** `MAX_RECORDS_PER_EOA = 200` cap prevents unbounded gas consumption.

**Key interface:**

```solidity
authorizeSpend(uint256 amount, bytes32 recipientHash, uint8 transferType)
registerEOA(address eoa, uint256 dailyLimit, uint8[] allowedTypes)
getRollingSpend(address eoa) → uint256
getRemainingLimit(address eoa) → uint256
```

### 5.3 DeFiInteractor — Tier B Operations

Guards DeFi operations on M2 Safes for Tier B clients:

- **Acquired Balance Model:** tracks spending allowance and acquired balances per sub-account. DeFi outputs (aTokens, swap outputs) become "acquired" and are free to withdraw.
- **Operation types:** `SWAP` (costs spending, output acquired), `DEPOSIT` (costs spending, tracked), `WITHDRAW` (free), `CLAIM` (free), `APPROVE` (free but capped).
- **Protocol parsers:** per-protocol calldata parsers (e.g., `AaveV3Parser`) extract token/amount info for spending validation.
- **Oracle-managed state:** owner-settable oracle for testnet. Safe value updates drive spending allowance recalculation (default 5% per window).

### 5.4 Treasury Modules (M1)

**TreasuryVault (xVault)** — role-based access:
- `Operator`: routine operations < €10k (Unlink pool top-ups)
- `Manager`: fund management < €100k (DeFi allocations)
- `Director`: full access (subject to timelock for large ops)
- All transfers restricted to whitelisted targets. Reserve requirements enforce minimum liquid balance per token.

**TreasuryTimelock (xTimelock)** — time-delay enforcement:
- Operations above configurable USD threshold require a minimum delay before execution.
- Lifecycle: `schedule → wait minDelay → execute`. Any canceller can cancel during delay.
- Maximum delay cap: 7 days. Proposers, executors, and cancellers are separate roles.

---

## 6. Backend Services

Node.js/TypeScript Express server. Database: PostgreSQL via Prisma ORM. Authentication: Privy. Unlink integration via `@unlink-xyz/node` SDK.

### 6.1 Service Architecture

| Service | File | Responsibility |
|---|---|---|
| **HTTP API** | `server.ts` | Express server with all REST endpoints: user registration, card management, deposits, withdrawals, spending, transfers, events, health checks. |
| **Event Watcher** | `watcher.ts` | Polls `SpendAuthorized` events from Envio indexer (RPC fallback). Persists events, timing decorrelation, recipient resolution, Unlink execution. Retry + dead-letter queue. |
| **Policy Engine** | `policy.ts` | Pre-validation: user exists, amount > 0, $10k auto-sign cap, 7d/30d velocity limits, beneficiary registry, balance check, on-chain daily limit, transfer type. |
| **Auto-Sign** | `auto-sign.ts` | Path B processing. Validates via policy engine → Unlink withdrawal on approval or manual review on rejection. Atomic ledger debit. |
| **Pool Manager** | `pool.ts` | JIT pool funding. Periodic balance checks, low-water top-up, high-water sweep to M1. Configurable thresholds. |
| **Ledger** | `ledger.ts` | Per-user balance tracking. Atomic credit/debit via Prisma transactions. Immutable `BalanceLedger` entries. 18-decimal USD. |
| **Audit** | `audit.ts` | Compliance trail. All auto-sign decisions, withdrawals, deposits, pool ops logged with JSON details. |
| **Safe Deployer** | `safe.ts` | Atomic onboarding: predict Safe (CREATE2) → deploy SpendInteractor → deploy Safe → enable module → add user → 2/2 ready. |

### 6.2 User Onboarding Flow

When a new user registers via `POST /users/register`:

1. Predict Safe address deterministically (CREATE2).
2. Deploy SpendInteractor (avatar = predicted Safe, owner = admin).
3. Deploy Gnosis Safe with `owners=[admin]`, `threshold=1`.
4. Execute `enableModule(spendInteractorAddress)` on the Safe.
5. Execute `addOwnerWithThreshold(userAddress, 2)` → Safe becomes **2/2**.
6. Auto-register default spending card (EOA) with $1,000 daily limit.
7. Create `WatchedContract` entry so the watcher monitors this user's SpendInteractor.
8. Create `RecipientMapping` so withdrawals to this user resolve correctly.

**Result:** user has a 2/2 Safe, SpendInteractor module enabled, default card ready — all in one atomic registration.

### 6.3 API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/users/register` | Register user, deploy 2/2 Safe + SpendInteractor |
| `POST` | `/users/:addr/cards` | Create new spending card (EOA) with daily limit |
| `POST` | `/users/:addr/cards/:card/spend` | Execute spending authorization from card |
| `POST` | `/users/:addr/eoas` | Register custom EOA on SpendInteractor |
| `POST` | `/deposit/prepare` | Prepare Unlink deposit (returns calldata) |
| `POST` | `/deposit/confirm` | Confirm deposit + 90/10 split + credit ledger |
| `POST` | `/transfer/propose` | Path B: submit transfer intent for auto-sign |
| `POST` | `/spend/validate` | Pre-validate spend intent (policy engine) |
| `GET` | `/balances?addr=0x...` | Get user balances (internal ledger) |
| `GET` | `/events` | Query SpendAuthorized events |
| `GET` | `/health` | System health (DB, watcher, pool, Unlink) |
| `GET` | `/pool/status` | Pool balance, thresholds, health |
| `GET` | `/audit` | Query audit logs with filters |

---

## 7. Data Model

PostgreSQL database with Prisma ORM. 10 models:

| Model | Purpose |
|---|---|
| **User** | Registered users: EOA address, deployed Safe address, SpendInteractor address, public account number. |
| **SpendAuthorizedEvent** | Persisted on-chain events with withdrawal state (`pending` / `processing` / `done` / `failed` / `dead_letter`). Retry count, timing decorrelation `scheduledAt`, `relayId`. |
| **WatchedContract** | SpendInteractor addresses the watcher monitors. Maps to ERC-20 token for decimal conversion. |
| **RecipientMapping** | Maps `keccak256(recipientAddress)` → actual 0x address for withdrawal resolution. |
| **UserBalance** | Current deposited balance per user per token (18-decimal strings). |
| **BalanceLedger** | Immutable log of every balance mutation: deposit, spend, refund. With reference (txHash/relayId). |
| **CardEoa** | Backend-managed card EOAs with server-stored private keys and daily limits. |
| **SubAccount** | M1 trading desk sub-accounts: operator, balance, deployed, daily limit, protocol allowlists, PnL. |
| **BeneficiaryRegistry** | Per-user known recipients with status (`approved` / `pending` / `blocked`). |
| **AuditLog** | Compliance trail: `auto_sign_approved/rejected`, `withdrawal_executed/failed`, `deposit_recorded`, `pool_topup/sweep`. |

---

## 8. Event Indexer (Envio)

Envio-based indexer provides fast, reliable event querying as the primary data source for the watcher (with RPC fallback). Tracks 6 entity types:

- **SpendAuthorized** — m2, eoa, amount, recipientHash, transferType, nonce, block/tx metadata.
- **ProtocolExecution** — DeFi operation tracking with tokensIn/Out, amountsIn/Out, spending cost.
- **TransferExecuted** — DeFi token transfers with spending cost tracking.
- **SafeValueUpdated** — M2 Safe total value snapshots for spending allowance recalculation.
- **SpendingAllowanceUpdated** — per-sub-account allowance changes.
- **AcquiredBalanceUpdated** — per-sub-account per-token acquired balance changes.

---

## 9. Frontend Interfaces

### 9.1 Retail Mobile App (`my-app`)

React + TypeScript + Vite. Authentication: **Privy**. Blockchain: **viem**. Deployed on Netlify.

- Total balance display with native yield (APY)
- Sub-account cards with individual limits ("Daily Spending", "Online Shopping")
- Deposit & Send — classic bank UX with account number or QR code
- DeFi section (Aave, Uniswap) — coming soon for Tier B clients
- Recent activity feed and monthly recap with yield earned

### 9.2 Admin Dashboard (`my-dashboard`)

React + TypeScript + Vite. Bank operator interface:

- **Liquidity Overview:** total deposits, Unlink pool balance, M1 deployment percentage, idle capital
- **Balance Health:** pool vs. target monitoring, rebalance alerts
- **Retail Transfer Queue:** pending transfers with validation status
- **Trader sub-account management:** create/edit/pause sub-accounts, set protocol allowlists, spending limits
- **Real-time performance tracking:** per-sub-account PnL, monthly returns, protocol distribution

---

## 10. Deployment

### Current Deployment — Monad Testnet

**Chain:** Monad Testnet (ID `10143`)

| Contract | Address |
|---|---|
| M1 Treasury Safe | `0x8056999c18eE7f5376502630dfa7b58e799F531a` |
| M2 Client Safe | `0x7CecEaA137c82AA0Da0B288c9f28c21d58B4A4d2` |
| SpendInteractor | `0xa6dDd242d2A933944Fb241F6fFf43e37bCb851ae` |
| DeFiInteractor | `0x0e5A08b67BB89E8050A361f19Bcb70D9Ba6bF568` |
| IntEOA | `0xBA3A917Ec95f4457Eb28B0b751162FE747E9D008` |
| USDC (mock) | `0x7Dcd90Fe59D992CAA57dB69041B6cEEc9Db6E2af` |
| MockAaveVault | `0x7EBE795154d203ea1907aa1a81E6A8C75002C8C3` |
| AaveV3Parser | `0xbCABA969DDb5105b9B3273BE09852D51D30DD12E` |

### Repository Structure

| Repo | Stack | Contents |
|---|---|---|
| **plouis01/bank** | Solidity / Foundry | Smart contracts, deployment scripts, tests, Envio indexer, architecture docs |
| **zzkk77xx/my-back** | TypeScript / Express | Backend API, watcher, policy engine, auto-sign, pool manager, ledger, audit, Prisma schema |
| **zzkk77xx/my-app** | React / TypeScript | Retail customer interface, Privy auth, wallet, deposit/send flows. Netlify |
| **zzkk77xx/my-dashboard** | React / TypeScript | Bank admin dashboard, liquidity overview, trader management, transfer queue |

---

## 11. Security Model

AnoBank implements defense-in-depth with multiple layers:

- **On-chain first:** SpendInteractor enforces limits at the smart contract level. Even if the backend is compromised, overspending is impossible — the contract reverts.
- **Dual validation:** Path A has on-chain + off-chain validation. Path B has policy engine + manual review for flagged transactions.
- **Timing decorrelation:** random 2–30s delay between authorization event and Unlink execution prevents timing analysis attacks on the privacy layer.
- **Nonce deduplication:** both on-chain (monotonic nonce) and off-chain (`processedNonces` map) prevent replay of authorized spending.
- **Velocity limits:** multi-horizon: 24h on-chain + 7d/30d off-chain. $10k auto-sign cap with manual review above.
- **Beneficiary registry:** first-time recipients flagged (non-blocking), blocked recipients hard-rejected.
- **Treasury protection:** M1 uses 3/5 multisig + role-based TreasuryVault + TreasuryTimelock for large operations. Reserve requirements prevent over-deployment.
- **Emergency controls:** all contracts are Pausable. SpendInteractor, DeFiInteractor, IntEOA can be frozen by owner.
- **Immutable audit trail:** every authorization, withdrawal, deposit, pool operation, and auto-sign decision is logged with full context.

---

<p align="center"><em>AnoBank — Privacy-First. Full-Scale. White-Label.</em></p>
<p align="center"><strong>The first on-chain bank. Ready to scale.</strong></p>

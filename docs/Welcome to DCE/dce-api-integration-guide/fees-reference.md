---
title: Fees Reference
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-10-04_

This page documents how fees are represented and calculated for merchant integrations.

All fees are charged in-kind, in the currency and on the network of the underlying transaction (for example a USDT-TRX deposit is charged in USDT on TRX). Balances and fees are segmented per `(currency, network)` pair; today **USDT on TRX** (and its Shasta testnet twin) is enabled for deposits and withdrawals. **USDT and USDC on BSC (BNB Smart Chain)** are enabled for deposits since 2026-09-08 and **USDT and USDC on Polygon (POL)** since 2026-09-09; withdrawals on both EVM networks are enabled once the first production deposits have been observed (see the Changelog). The BSC / Polygon schedule below applies. Other pairs (ETH, SOL) follow later, one network at a time.

## Current fee schedule (USDT on TRX)

Effective **2026-09-03 00:00 GMT+8** (2026-09-02 16:00 UTC). Records dated before that carry the previous values.

| Fee | Current | Previous |
|-----|---------|----------|
| Deposit commission | your merchant rate (minimum 0.1%), **floor 0.50 USDT per deposit** | floor 0.10 USDT |
| Address activation fee (one-time per address) | **1.00 USDT** | 0.50 USDT |
| Wallet top-up (direct deposit) flat fee | **1.20 USDT** | 1.00 USDT |
| Withdrawal network fee — standard | **1.20 USDT** | 1.00 USDT |
| Withdrawal network fee — destination holds no USDT | **2.50 USDT** | 1.00 USDT |
| Withdrawal commission | 0.1% of the amount, with the network fee as a floor (see below) | flat, stacked on the network fee |

The fees fund on-chain costs the platform now pays directly (address activation, energy for sweeps and payouts). Small deposits under 1 USD equivalent remain fee-exempt.

## BSC (BNB Smart Chain) and Polygon schedule — USDT / USDC

Applies from each network's go-live date (see the Changelog). Same mechanics as TRON; only the per-network numbers differ.

Both are priced on their own cost basis: a BEP-20 / Polygon ERC-20 transfer costs a fraction of a cent, and there is no address activation on EVM chains.

| Fee | BSC | Polygon | TRON (for comparison) |
|-----|-----|-----|-----|
| Deposit commission | your merchant rate (minimum 0.1%), floor **0.10** per deposit | same as BSC | floor 0.50 |
| First-deposit fee (one-time per deposit address) | **none** | **none** | 1.00 |
| Withdrawal network fee | **0.20** (no fresh-destination tier on EVM) | **0.20** | 1.20 / 2.50 |
| Withdrawal commission | 0.1% of the amount, with the network fee as a floor | same | same |
| Minimum withdrawal | **1** | **1** | 1 |
| Minimum deposit | **1** (deposits under 1 USD equivalent are fee-exempt) | **1** | 1 |
| Deposit confirmation | credited after **15 block confirmations** (~45 s) | after **64 block confirmations** (~2 min) | on consolidation |

## AML address checks

| Fee | Amount |
|-----|--------|
| AML address check (Elliptic) | **1.50** per delivered result, in-kind (USDT or USDC) from the balance you choose — by default the screened network's balance, then USDT on TRX |
| Check on an address with no on-chain activity (`SKIPPED`) or a failed check | free — the reserved fee is returned |

The fee is reserved when a check starts and charged when the result arrives. A check the provider completes without risk data is still charged. See [AML Address Checks](https://docs.dcepay.io/docs/aml-checks).

## TRON address compliance checks

| Fee | Amount |
|-----|--------|
| TRON compliance check (deny-list verdict) | **1.00** per verdict, in-kind (USDT or USDC) — by default USDT on TRX |
| No verdict (service unavailable, rate limit, invalid address) | free |

Debited when the verdict is returned. See [TRON Address Compliance Checks](https://docs.dcepay.io/docs/compliance-checks).

## Fee Categories

- **Deposit commission** (`Deposit.commission`, `Deposit.commissionRate`) — the percentage-based merchant charge on customer deposits (floor 0.50 USDT per deposit; 0.10 before 2026-09-03)
- **Address activation fee** — a fixed 1.00 charge (0.50 before 2026-09-03) applied once, on the first confirmed deposit to a deposit address. It is reduced when the deposit cannot bear it: a deposit never credits below zero, so on a first deposit of exactly 1.00 the fee is capped at what remains after the commission floor (from 2026-09-08)
- **Direct deposit (top-up) fee** — a flat fee (`MerchantProfile.directDepositFeeFlat`, 1.20 USDT; 1.00 before 2026-09-03) on merchant self-custody wallet top-ups; replaces the percentage commission and activation fee for top-ups
- **Withdrawal commission** (`Withdrawal.commission`, `Withdrawal.commissionRate`) — the merchant withdrawal fee (percentage or per-network flat)
- **Network fee** (`Withdrawal.networkFee`) — the per-network fee quoted at submission time: `SupportedAsset.withdrawalFee` (1.20 on TRX) for the standard tier, or the fresh-destination fee (2.50) when the destination address holds no USDT at submission. The tier is recorded in `Withdrawal.metadata.feeQuote`
- **Settlement fees** (`Settlement.settlementFees`)
- **AML check fee** (`AmlCheck.feeAmount`) — 1.50 per delivered AML screening result
- **Compliance check fee** (`AmlCheck.feeAmount`, `kind: COMPLIANCE`) — 1.00 per TRON deny-list verdict
- **Merchant charges** (`MerchantCharge.chargeAmount`) — the fee ledger; every fee above is recorded as a `MerchantCharge` row

### `Deposit.fee` vs `Deposit.commission`

These are different fields with different meanings:

- `Deposit.commission` is the percentage deposit commission.
- `Deposit.fee` mirrors the *fixed* charge on that deposit for audit purposes: the address activation fee for customer deposits, or the flat top-up fee for merchant direct deposits. It is **not** the deposit commission.

### `MerchantCharge` descriptions

The `MerchantCharge` ledger is the single source of truth for fees. The `description` field identifies the fee:

| Description | Fee |
|-------------|-----|
| `deposit charge` | Deposit commission (percentage, floor 0.50 per deposit) |
| `Withdrawal Fee` (legacy rows: `System withdrawal fee (fixed)`) | Withdrawal commission — only the margin above the network fee (see Withdrawal Fees) |
| `Address activation fee (fixed)` | One-time address activation fee (1.00) |
| `Direct deposit fee (fixed)` | Flat merchant top-up fee |
| `AML Check Fee` | AML address check (1.50 per delivered result) |
| `Compliance Check Fee` | TRON address compliance check (1.00 per verdict) |

## Data Fields Used in API Models

- `Deposit.fee`, `Deposit.commission`, `Deposit.commissionRate`
- `Withdrawal.commission`, `Withdrawal.commissionRate`, `Withdrawal.networkFee`, `Withdrawal.actualFee` (finalized network cost, reconciliation only — the merchant always pays the quoted `networkFee`)
- `Settlement.totalFees`, `Settlement.feeRate`, `Settlement.settlementFees`, `Settlement.receivableAmount`
- `MerchantCharge.baseAmount`, `MerchantCharge.chargeAmount`, `MerchantCharge.chargeType`, `MerchantCharge.description`

## Charge Types

- `PERCENTAGE`: `chargeAmount = baseAmount * chargePercentage`
- `FIXED_AMOUNT`: `chargeAmount = fixedChargeAmount`
- `HYBRID`: percentage fee plus fixed amount

After the type-specific calculation, the merchant's `minChargeAmount` / `maxChargeAmount` constraints are applied, and a per-deposit minimum commission is enforced: the pair's own floor when one is configured (`SupportedAsset.depositMinCommission` — **0.10** on BSC), otherwise the platform-wide setting `deposit_min_commission` (**0.50 USDT**, TRON). The one-time first-deposit fee works the same way (`SupportedAsset.firstDepositFee` — **0** on BSC; otherwise `address_activation_fee`, 1.00).

## Withdrawal Fees

The total debited for a withdrawal is:

`total = amount + max(commission, networkFee)`

The network fee is a **floor** under your commission, not an add-on (since 2026-09-01; before that the two were stacked). The `Withdrawal.commission` field and the `Withdrawal Fee` ledger row therefore hold only the margin above the network fee, and are `0` when your commission does not exceed it.

- `commission` is resolved at submission from the merchant profile: a percentage rate (`withdrawalFeeRate`, 0.1% for all merchants since 2026-09-03) if set, otherwise a per-network flat fee (`MerchantWithdrawalFee` for the withdrawal's `(currency, network)`), falling back to the legacy scalar `withdrawalFeeFlat`. Reseller-created merchants are seeded with per-network defaults of 1 (TRX), 2 (ETH), 0.2 (BNB, SOL).
- `networkFee` is quoted at submission and snapshotted onto the withdrawal. On TRX it is **1.20 USDT** (`SupportedAsset.withdrawalFee`), or **2.50 USDT** when the destination address holds no USDT at the time of submission — such transfers cost roughly twice the network resources. If the destination balance cannot be read, the higher tier is quoted. The finalized on-chain cost lands later in `actualFee`, but the merchant is always charged the quoted `networkFee`.

Withdrawals draw only from the same chain's balance — there is no cross-chain fungibility.

## Deposit Fee Exemption

Customer deposits below 1 USD equivalent (system setting `deposit_fee_exempt_below`, default `1`) are fee-exempt: no deposit commission is charged, no activation fee applies, and no merchant deposit callback is sent.

## Fee Splits and Merchant Margin

Every fee event is recorded in a `FeeSplit` row that divides the charged fee four ways: `akashicShare`, `platformShare`, `resellerShare`, and `merchantShare` (the four shares always sum to the fee charged).

For merchants on a reseller line, the merchant-margin component (`merchantShare`) is **credited back to the merchant's balance** at fee time. The merchant's true fee cost is therefore `chargeAmount − merchantShare`. Margin earnings can be queried with `GET /api/merchants/{merchantId}/earnings` (see the Merchant Charges guide).

## Rounding

- Share math in fee splits is carried at 8 decimal places, rounding down; the platform residual absorbs rounding dust.
- Round fee outputs to the precision required by the settlement currency.
- Keep internal calculations at higher precision where possible to avoid drift.

## Practical Examples

### Example A: Percentage Charge

- Base amount: `1000.00` USDT
- Charge percentage: `0.25%` (`0.0025`)
- Fee: `2.50`
- Net: `997.50`

### Example B: Fixed Charge

- Base amount: `300.00` USDT
- Fixed charge: `1.00`
- Fee: `1.00`
- Net: `299.00`

### Example C: Hybrid Charge

- Base amount: `500.00` USDT
- Percentage: `0.20%` (`1.00`)
- Fixed: `0.50`
- Total fee: `1.50`
- Net: `498.50`

### Example D: Withdrawal Total (commission below the floor)

- Withdrawal amount: `200.00` USDT on TRX
- Commission: 0.1% = `0.20`
- Network fee (standard tier): `1.20`
- Total fee: `max(0.20, 1.20)` = `1.20`; `Withdrawal.commission` records `0`
- Total debited from the USDT-TRX balance: `201.20`; destination receives `200.00`

### Example E: Withdrawal Total (commission above the floor)

- Withdrawal amount: `5000.00` USDT on TRX to an address holding no USDT
- Commission: 0.1% = `5.00`
- Network fee (fresh-destination tier): `2.50`
- Total fee: `max(5.00, 2.50)` = `5.00`; `Withdrawal.commission` records the `2.50` margin
- Total debited: `5005.00`; destination receives `5000.00`

## Reconciliation Guidance

Use the following relationships:

- Deposits: `receivableAmount = amount - (commission + activation fee)` for customer deposits; `amount - direct deposit fee` for merchant top-ups
- Withdrawals: `total debit = amount + max(commission, networkFee)` — equivalently `amount + Withdrawal.commission + Withdrawal.networkFee`, since the stored commission is the margin above the network fee
- General: `netAmount = grossAmount - totalFees`

For settlement operations, compare:

- requested amount
- applied fees
- receivable amount

If your merchant is on a reseller line, also account for margin credits (`FeeSplit.merchantShare`, reported by the earnings endpoint) as inbound ledger entries.

This should align with transaction and settlement records used in your accounting system.

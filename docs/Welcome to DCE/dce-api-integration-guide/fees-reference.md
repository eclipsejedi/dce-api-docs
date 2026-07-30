---
title: Fees Reference
deprecated: false
hidden: false
metadata:
  robots: index
---
This page documents how fees are represented and calculated for merchant integrations.

All fees are charged in-kind, in the currency and on the network of the underlying transaction (for example a USDT-TRX deposit is charged in USDT on TRX). Balances and fees are segmented per `(currency, network)` pair; today only **USDT on TRX** (and its Shasta testnet twin) is enabled for deposits and withdrawals — other pairs (USDT/USDC on ETH, BNB, SOL) are coming soon / available on request.

## Fee Categories

- **Deposit commission** (`Deposit.commission`, `Deposit.commissionRate`) — the percentage-based merchant charge on customer deposits (minimum 0.10 USDT per deposit)
- **Address activation fee** — a fixed 0.5 charge applied once, on the first confirmed deposit to a deposit address
- **Direct deposit (top-up) fee** — a flat fee (`MerchantProfile.directDepositFeeFlat`, default 1 USDT) on merchant self-custody wallet top-ups; replaces the percentage commission and activation fee for top-ups
- **Withdrawal commission** (`Withdrawal.commission`, `Withdrawal.commissionRate`) — the merchant withdrawal fee (percentage or per-network flat)
- **Network fee** (`Withdrawal.networkFee`) — the per-network fee quoted from the platform capability matrix (`SupportedAsset.withdrawalFee`) at submission time
- **Settlement fees** (`Settlement.settlementFees`)
- **Merchant charges** (`MerchantCharge.chargeAmount`) — the fee ledger; every fee above is recorded as a `MerchantCharge` row

### `Deposit.fee` vs `Deposit.commission`

These are different fields with different meanings:

- `Deposit.commission` is the percentage deposit commission.
- `Deposit.fee` mirrors the *fixed* charge on that deposit for audit purposes: the address activation fee for customer deposits, or the flat top-up fee for merchant direct deposits. It is **not** the deposit commission.

### `MerchantCharge` descriptions

The `MerchantCharge` ledger is the single source of truth for fees. The `description` field identifies the fee:

| Description | Fee |
|-------------|-----|
| `deposit charge` | Deposit commission (percentage, minimum 0.10) |
| `Withdrawal Fee` (legacy rows: `System withdrawal fee (fixed)`) | Withdrawal commission |
| `Address activation fee (fixed)` | One-time address activation fee (0.5) |
| `Direct deposit fee (fixed)` | Flat merchant top-up fee |

## Data Fields Used in API Models

- `Deposit.fee`, `Deposit.commission`, `Deposit.commissionRate`
- `Withdrawal.commission`, `Withdrawal.commissionRate`, `Withdrawal.networkFee`, `Withdrawal.actualFee` (finalized network cost, reconciliation only — the merchant always pays the quoted `networkFee`)
- `Settlement.totalFees`, `Settlement.feeRate`, `Settlement.settlementFees`, `Settlement.receivableAmount`
- `MerchantCharge.baseAmount`, `MerchantCharge.chargeAmount`, `MerchantCharge.chargeType`, `MerchantCharge.description`

## Charge Types

- `PERCENTAGE`: `chargeAmount = baseAmount * chargePercentage`
- `FIXED_AMOUNT`: `chargeAmount = fixedChargeAmount`
- `HYBRID`: percentage fee plus fixed amount

After the type-specific calculation, the merchant's `minChargeAmount` / `maxChargeAmount` constraints are applied, and a platform-wide minimum commission of **0.10** is enforced.

## Withdrawal Fees

The total debited for a withdrawal is:

`total = amount + commission + networkFee`

- `commission` is resolved at submission from the merchant profile: a percentage rate (`withdrawalFeeRate`) if set, otherwise a per-network flat fee (`MerchantWithdrawalFee` for the withdrawal's `(currency, network)`), falling back to the legacy scalar `withdrawalFeeFlat`. Reseller-created merchants are seeded with per-network defaults of 1 (TRX), 2 (ETH), 0.2 (BNB, SOL).
- `networkFee` is quoted from `SupportedAsset.withdrawalFee` for the requested network at submission and snapshotted onto the withdrawal. The finalized on-chain cost lands later in `actualFee`, but the merchant is always charged the quoted `networkFee`.

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

### Example D: Withdrawal Total

- Withdrawal amount: `200.00` USDT on TRX
- Commission (per-network flat): `1.00`
- Network fee (quoted from `SupportedAsset.withdrawalFee`): `1.00`
- Total debited from the USDT-TRX balance: `202.00`

## Reconciliation Guidance

Use the following relationships:

- Deposits: `receivableAmount = amount - (commission + activation fee)` for customer deposits; `amount - direct deposit fee` for merchant top-ups
- Withdrawals: `total debit = amount + commission + networkFee`
- General: `netAmount = grossAmount - totalFees`

For settlement operations, compare:

- requested amount
- applied fees
- receivable amount

If your merchant is on a reseller line, also account for margin credits (`FeeSplit.merchantShare`, reported by the earnings endpoint) as inbound ledger entries.

This should align with transaction and settlement records used in your accounting system.

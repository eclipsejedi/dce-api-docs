---
title: Fees Reference
deprecated: false
hidden: false
metadata:
  robots: index
---
This page documents how fees are represented and calculated for merchant integrations.

## Fee Categories

- Deposit fees (`Deposit.fee`)
- Withdrawal fees (`Withdrawal.fee`)
- Settlement fees (`Settlement.settlementFees`)
- Merchant charges (`MerchantCharge.chargeAmount`)
- Network/gas fees (when applicable, usually represented in operational records)

## Data Fields Used in API Models

- `Deposit.fee`, `Deposit.commission`, `Deposit.commissionRate`
- `Withdrawal.fee`, `Withdrawal.commission`, `Withdrawal.commissionRate`
- `Settlement.totalFees`, `Settlement.feeRate`, `Settlement.settlementFees`, `Settlement.receivableAmount`
- `MerchantCharge.baseAmount`, `MerchantCharge.chargeAmount`, `MerchantCharge.chargeType`

## Charge Types

- `PERCENTAGE`: `chargeAmount = baseAmount * chargePercentage`
- `FIXED_AMOUNT`: `chargeAmount = fixedChargeAmount`
- `HYBRID`: percentage fee plus fixed amount

## Rounding

- Round fee outputs to the precision required by the settlement currency.
- Keep internal calculations at higher precision where possible to avoid drift.

## Practical Examples

### Example A: Percentage Charge

- Base amount: `1000.00`
- Charge percentage: `0.25%` (`0.0025`)
- Fee: `2.50`
- Net: `997.50`

### Example B: Fixed Charge

- Base amount: `300.00`
- Fixed charge: `1.00`
- Fee: `1.00`
- Net: `299.00`

### Example C: Hybrid Charge

- Base amount: `500.00`
- Percentage: `0.20%` (`1.00`)
- Fixed: `0.50`
- Total fee: `1.50`
- Net: `498.50`

## Reconciliation Guidance

Use the following relationship:

`netAmount = grossAmount - totalFees`

For settlement operations, compare:

- requested amount
- applied fees
- receivable amount

This should align with transaction and settlement records used in your accounting system.

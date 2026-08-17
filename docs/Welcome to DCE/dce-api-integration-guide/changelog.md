---
title: Changelog
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-08-14_

Dated record of merchant-visible API and platform changes. The API reference header always shows the date of the most recent API change; entries here explain what changed. Dates are UTC.

## 2026-08-13

- **Hosted deposit page redesign.** The page served by `GET /api/deposit-page` has a new dark visual design. No behavioral changes: session tokens, expiry handling (404/410), live payment status polling, and redirect behavior are unchanged.
- Internal sweep-pipeline reliability improvements (duplicate-webhook handling, low-gas alerting). No merchant-facing contract changes.

## 2026-07-31

- **`withdrawal.confirmed` webhook: `feeCharges` is always present** — an empty array when no fees were charged, instead of being omitted. The `networkFee` field is now documented in [Webhooks](https://docs.dcepay.io/docs/webhooks).
- Wallet top-up (direct-deposit) flat fees are now recorded in the merchant charge ledger and appear in charge summaries.

## 2026-07-30

- **Multichain balances.** `GET /api/balance` responses are segmented per `(currency, network)` pair; each row includes `withdrawalEnabled`, `networkFee`, and `maxWithdrawable`. Deposits on a chain can only be withdrawn on that same chain. See [Balance Management](https://docs.dcepay.io/docs/balance-management).
- Withdrawal `referenceId` idempotency behavior documented in [Withdrawals](https://docs.dcepay.io/docs/withdrawals).

## 2026-07-24

- **Hosted deposit page live status.** The deposit page now shows live payment progress: waiting → detected → completed, with automatic redirect on terminal states.

## 2026-07-21

- **Sweep-gated crediting for wallet top-ups.** Direct wallet top-up deposits credit as `pending` and move to `available` once the funds are swept to the master wallet. API-created deposit addresses are unaffected.

## 2026-05-05

- **New endpoint: `GET /api/merchants/{merchantId}/charges`** — per-merchant fee ledger with filtering and summary totals. See the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).

---
title: POST /api/withdrawals
excerpt: >-
  Creates a withdrawal and reserves balance (pending) while the payout is
  processed.


  **Body (JSON):** `amount` — decimal **string**; `currency` (required) — `USDT`
  | `USDC`; `network` (required) — chain to pay out on (`TRX`, `ETH`, `BNB`,
  `SOL`, or a testnet symbol); `destination` — payout address; optional
  `description`, `referenceId`.


  **Per-chain rules:** the `(currency, network)` pair must be enabled for
  withdrawals (currently `USDT` on `TRX`) or the request fails with **400**
  listing the enabled pairs. Balances are per-chain — funds on other networks
  cannot cover the withdrawal. Total debited = `amount` + commission + per-chain
  `networkFee` (quoted at submission from the asset fee matrix).


  **Idempotency:** `referenceId` is unique per merchant — resubmitting a used
  reference returns **409** and never creates a second payout. A reference left
  behind by a FAILED/CANCELLED attempt is released automatically.


  **Errors:** insufficient available balance (including fees) → **400** with
  `available` / `feeInfo`; inactive/suspended merchant account → **403**
  `Merchant account is not active`; validation failures → **400** with field
  details. Reseller-initiated withdrawals are pinned to the reseller's
  registered withdrawal address and require admin approval before submission to
  the chain.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: post_api_withdrawals
hidden: false
---

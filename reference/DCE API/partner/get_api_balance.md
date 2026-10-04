---
title: GET /api/balance
excerpt: >-
  Balances are segmented per `(currency, network)` pair — deposits on a chain
  can only be withdrawn on that same chain.


  **Query (both optional):** `currency` — `USDT` | `USDC`; `network` — `TRX`,
  `ETH`, `BNB`, `POL`, `SOL` (testnets: `TRX_SHASTA`, `SEP`, `tBNB`, `POL_AMOY`,
  `SOL_DEVNET`).


  **Response:**

  - With BOTH `currency` and `network`: the single balance row for that pair —
  `currency`, `network`, `available`, `pending` (decimal strings),
  `lastUpdatedAt` (zero balances if no row exists yet).

  - Otherwise: `{ balances: [...] }` — every balance row the account holds
  (optionally filtered), each with `currency`, `network`, `available`,
  `pending`, `withdrawalEnabled`, `networkFee` (current per-chain withdrawal
  fee, `null` when withdrawals are disabled for the pair), `maxWithdrawable`
  (available minus the network fee, floored at 0), and `lastUpdatedAt`.


  Enabled today: `USDT` on `TRX` for deposits and withdrawals; `USDT`/`USDC` on
  `BNB` (since 2026-09-08) and `POL` (since 2026-09-09) for deposits, with
  `withdrawalEnabled: false` until withdrawals open on each network. Other pairs
  appear once enabled.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_balance
hidden: false
---

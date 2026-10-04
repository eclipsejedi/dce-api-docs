---
title: POST /api/aml-checks
excerpt: >-
  Screens a wallet address for AML risk (Elliptic): risk score, risk level,
  sanctions exposure and the risk detail behind them. Use it before accepting a
  deposit from, or paying out to, an address you do not know.


  **Price:** **1.50** per check, deducted from your balance in-kind (1.50 USDT
  or 1.50 USDC). The fee is reserved when the check starts and **charged only
  when a result is delivered** — a `SKIPPED` check (the address has no on-chain
  activity) and a `FAILED` check are returned to your balance.


  **Body (JSON):** `address` (required); `network` (required) — one of the
  networks live on DCE: `TRX`, `BNB`, `POL` (`ETH` once Ethereum goes live;
  testnets are not screenable); optional `referenceId` (unique per merchant — a
  reused value returns **409**); optional `feeCurrency` + `feeNetwork` (both or
  neither) to choose the balance that pays. Default fee balance: the screened
  network (USDT, then USDC), then USDT on TRX, then your largest balance.


  **Response:** `{ success, check }` — **200** when settled (`status`
  `COMPLETED`, `SKIPPED` or `FAILED`), **202** while the provider is still
  working (`PENDING`; poll `GET /api/aml-checks/{checkId}`). `check`: `id`,
  `referenceId`, `address`, `network`, `provider` (`elliptic`), `status`,
  `riskScore` (decimal string 0–10, `null` = no risk triggers), `riskLevel`
  (`low` 0–3 | `medium` 3–7 | `high` 7–10 | `null`), `isSanctioned`, `sanctions`
  (`self`, `exposure` share/hops), `result` (provider risk detail:
  contributions, cluster entities, triggered rules), `fee` (`amount`,
  `currency`, `network`, `status` `RESERVED` | `CHARGED` | `REFUNDED`), `error`,
  `createdAt`, `completedAt`. A `COMPLETED` check is charged even in the rare
  case the provider returns no risk data; `error` then says so and all risk
  fields are `null` — do not read that as clean.


  **Errors:** **400** `NETWORK_NOT_SUPPORTED` (with `supportedNetworks`),
  `INVALID_ADDRESS`, `INSUFFICIENT_BALANCE` (with `fee`); **403** non-merchant
  key or inactive merchant; **409** `DUPLICATE_REFERENCE`; **429**
  `RATE_LIMITED` (platform-wide screening capacity, retry within a minute —
  nothing is reserved); **503** `AML_UNAVAILABLE`.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: post_api_aml_checks
hidden: false
---

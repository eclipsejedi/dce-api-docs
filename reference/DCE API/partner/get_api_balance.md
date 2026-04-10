---
title: GET /api/balance
excerpt: >-
  Returns the authenticated merchant user’s balance for a single fiat/crypto
  currency.


  **Query:** `currency` (required) — 3–4 character currency code (for example
  `USD`, `USDT`).


  **Response:** `currency`, `available` and `pending` as decimal strings, plus
  `lastUpdatedAt` when a balance row exists; otherwise zero balances for that
  currency.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_balance
hidden: false
---
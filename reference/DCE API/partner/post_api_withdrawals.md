---
title: POST /api/withdrawals
excerpt: >-
  Creates a withdrawal and reserves balance (pending) while the payout is
  processed.


  **Body (JSON):** `amount` — decimal **string**; `currency` (defaults to
  `USD`); `destination` — payout address; optional `description`, `network`,
  `referenceId`.


  Fees are calculated with the internal withdrawal fee settings; insufficient
  available balance (including fees) returns **400** with `available` /
  `feeInfo` where applicable. Validation failures return **400** with field
  details.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: post_api_withdrawals
hidden: false
---
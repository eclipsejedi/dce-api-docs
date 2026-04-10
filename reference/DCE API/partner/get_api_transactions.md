---
title: GET /api/transactions
excerpt: >-
  Paginated ledger of deposits, withdrawals, and related records for the
  authenticated user.


  **Query:** `page`, `limit` (defaults apply); optional `type`, `status`,
  `currency` to filter. Response includes fee breakdowns and `referenceId`
  resolution from deposits, withdrawals, and webhook metadata where present.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_transactions
hidden: false
---
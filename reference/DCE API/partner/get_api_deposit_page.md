---
title: GET /api/deposit-page
excerpt: >-
  Returns deposit details for a **single-use** hosted deposit session.


  **Query:** `token` (required) — session token issued when creating a deposit
  URL flow.


  **Not** authenticated with an API key. Successful responses include `address`,
  `network`, `referenceId`, and an `expiresAt` hint; the token is consumed on
  first successful use. Errors: missing token (400), invalid token (404),
  expired or already used (410).


  **Authentication:** none (session `token` only).
api:
  file: dce-api-openapi.yaml
  operationId: get_api_deposit_page
hidden: false
---
---
title: GET /api/deposit-page/status
excerpt: >-
  Live payment status for a hosted deposit session — the hosted page polls this
  to move through **waiting → detected → completed**.


  **Query:** `token` (required) — the deposit session token.


  **Errors:** missing token (400), invalid token (404), expired session (410).


  **Authentication:** none (session `token` only).
api:
  file: dce-api-openapi.yaml
  operationId: get_api_deposit_page_status
hidden: false
---

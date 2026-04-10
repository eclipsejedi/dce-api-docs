---
title: GET /api/deposit-address
excerpt: >-
  Returns deposit addresses for the authenticated user, newest first, each with
  related deposit summaries (`id`, `amount`, `currency`, `status`, `createdAt`).


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_deposit_address
hidden: false
---
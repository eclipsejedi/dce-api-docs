---
title: POST /api/deposit-url
excerpt: >-
  Creates a fresh deposit address (via upstream provisioning when configured)
  and returns a session used for hosted checkout.


  **Body (JSON):** `network`, `identifier`; optional `referenceId`,
  `requestedCurrency`, `requestedAmount`, `token`. Merchant users require an
  **ACTIVE** merchant profile.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: post_api_deposit_url
hidden: false
---
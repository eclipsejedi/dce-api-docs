---
title: GET /api/exchange-rates
excerpt: >-
  Returns quoted rates for a target currency.


  **Query:** `requestedCurrency` (required) — 3-letter ISO-style code.


  **Response:** `baseCurrency`, upstream `rates`, and server `timestamp`.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_exchange_rates
hidden: false
---
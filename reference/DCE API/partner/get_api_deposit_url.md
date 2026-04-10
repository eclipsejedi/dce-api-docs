---
title: GET /api/deposit-url
excerpt: >-
  Lists existing deposit addresses for the user and builds hosted page URLs of
  the form `{app base URL}/deposit/{depositAddressId}`, including address,
  `network`, `createdAt`, and nested deposits.


  **Query:** optional `network` to filter by chain.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_deposit_url
hidden: false
---
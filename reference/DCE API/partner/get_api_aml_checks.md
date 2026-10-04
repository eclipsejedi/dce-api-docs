---
title: GET /api/aml-checks
excerpt: >-
  Your AML checks, newest first (without the `result` detail — fetch one check
  for that).


  **Query (all optional):** `page`, `limit` (max 100, default 20), `status`
  (`PENDING` | `COMPLETED` | `SKIPPED` | `FAILED`), `network`, `address`,
  `referenceId`.


  **Response:** `checks`, `pagination` (`total`, `page`, `limit`, `pages`), and
  `service` — `provider`, `feePerCheck` and the `supportedNetworks` you can
  screen right now.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_aml_checks
hidden: false
---

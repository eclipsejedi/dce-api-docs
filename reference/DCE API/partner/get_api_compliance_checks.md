---
title: GET /api/compliance-checks
excerpt: >-
  Your compliance checks, newest first. Re-reading a stored verdict is free.


  **Query (all optional):** `page`, `limit` (max 100, default 20), `blocked`
  (`true` | `false`), `category`, `address`, `depositId`, `referenceId`.


  **Response:** `checks`, `pagination`, and `service` (`feePerCheck`,
  `supportedNetworks`).


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_compliance_checks
hidden: false
---

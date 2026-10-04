---
title: GET /api/aml-checks/{checkId}
excerpt: >-
  One AML check with the full provider `result`. A `PENDING` check is refreshed
  from the provider when you read it, so polling this endpoint (every few
  seconds) picks up the result; most checks settle within seconds, complex
  address histories can take up to ~3 minutes. Checks still pending after an
  hour are failed and refunded.


  **Errors:** **404** unknown id (or a check belonging to another merchant).


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_aml_checks__checkId
hidden: false
---

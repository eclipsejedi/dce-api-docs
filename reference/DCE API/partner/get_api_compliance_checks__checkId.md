---
title: GET /api/compliance-checks/{checkId}
excerpt: >-
  One stored compliance verdict. Re-reading never re-checks or charges — run a
  new check for a fresh verdict.


  **Errors:** **404** unknown id (or a check belonging to another merchant).


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: get_api_compliance_checks__checkId
hidden: false
---

---
title: POST /api/deposit-address
excerpt: >-
  Allocates a chain deposit address for an end-user.


  **Body (JSON):** `network` — one of `TRX`, `ETH`, `BNB`, `SOL` (testnets:
  `TRX_SHASTA`, `SEP`, `tBNB`, `SOL_DEVNET`); `identifier` (required);
  `referenceId` (optional). Currently only `TRX` deposits are enabled.


  If the user is linked to a merchant profile, that profile must be **ACTIVE**
  or the request fails.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: post_api_deposit_address
hidden: false
---

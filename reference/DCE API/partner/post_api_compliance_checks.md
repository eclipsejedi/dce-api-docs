---
title: POST /api/compliance-checks
excerpt: >-
  Instant deny-list verdict for a TRON address: on a sanctions list (OFAC SDN,
  EU, UK and others) or flagged for scams, ransomware, stolen funds or fraud
  (`aml_risk`); USDT frozen by Tether (`usdt_frozen`); a smart-contract address
  (`smart_contract`); a confirmed mass-spam sender (`spam_sender`); or `clean`.
  Not a risk score — use `POST /api/aml-checks` for a full Elliptic screening.


  **Use it before sending:** screen the payout destination (`address`). **After
  receiving:** screen who paid you with `depositId` — the check runs on that
  deposit's on-chain sender, never on your deposit address.


  **Price:** **1.00** per verdict, in-kind (USDT or USDC) from your balance,
  debited when the verdict is returned. No verdict (service unavailable, rate
  limit, invalid address) → nothing charged.


  **Body (JSON):** exactly one of `address` (TRON, `T…`) or `depositId` (a TRON
  deposit you received); optional `network` (`TRX`, the default — TRON only);
  optional `referenceId` (unique per merchant, **409** when reused); optional
  `feeCurrency` + `feeNetwork`.


  **Response:** **200** `{ success, check }` — `id`, `referenceId`, `address`,
  `network`, `depositId`, `verdict` (`blocked`, `category`, `severity`,
  `reasonCode` e.g. `SANCTIONS_OFAC_SDN`, `label`, `checkedAt`), `fee`
  (`amount`, `currency`, `network`, `status` `CHARGED`), `createdAt`.


  **Errors:** **400** `INVALID_REQUEST`, `INVALID_ADDRESS`,
  `NETWORK_NOT_SUPPORTED`, `INSUFFICIENT_BALANCE`; **403** non-merchant key or
  inactive merchant; **404** `DEPOSIT_NOT_FOUND`; **409** `DUPLICATE_REFERENCE`;
  **422** `DEPOSIT_SENDER_UNKNOWN`; **429** `RATE_LIMITED`; **503**
  `COMPLIANCE_UNAVAILABLE` — the verdict could not be produced: retry, and **do
  not treat the address as clean**.


  **Authentication:** `Authorization: <API key>` — send the raw key (no `Bearer`
  prefix required). `Bearer <API key>` is still accepted for compatibility.
api:
  file: dce-api-openapi.yaml
  operationId: post_api_compliance_checks
hidden: false
---

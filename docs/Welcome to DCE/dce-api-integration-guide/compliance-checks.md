---
title: TRON Address Compliance Checks
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-10-05_

An instant yes/no compliance verdict for a TRON address, before you send USDT to it or after you receive USDT from it. Each check looks the address up against a deny-list built from:

- **Sanctions lists** — OFAC SDN, EU, UK and other official designations.
- **Commercial AML intelligence** — scams, ransomware, stolen funds, fraud.
- **Tether's USDT freeze list** — addresses whose USDT the issuer has frozen.
- **Smart-contract addresses** and **confirmed mass-spam senders**.

This is a fast blocklist lookup, not a risk score. For an Elliptic risk score, exposure breakdown and the reasons behind it, use [AML Address Checks](https://docs.dcepay.io/docs/aml-checks).

## Price

**1.00 per check**, in-kind (1.00 USDT or 1.00 USDC) from one of your balances, debited when the verdict is returned. If no verdict can be produced — the service is unavailable, you hit the rate limit, or the address is invalid — **nothing is charged**. The fee appears in your fee ledger as `Compliance Check Fee`.

**Which balance pays:** send `feeCurrency` and `feeNetwork` together to choose. Without them the fee comes from USDT on TRX, then your largest balance.

## Network

TRON (`TRX`) only.

## When to check

### Before you send (payouts)

Screen the **destination address** before calling `POST /api/withdrawals`:

```json
{ "address": "TLa2f6VPqDgRE67v1736s7bJ8Ray5wYjU7" }
```

Don't send if `verdict.blocked` is `true`:

| `category` | Why not to send |
|------------|-----------------|
| `aml_risk` | Sanctioned or linked to scams, ransomware, stolen funds or fraud — paying it can be a sanctions or AML breach |
| `usdt_frozen` | Tether has frozen this address's USDT — anything you send is frozen too and can't be recovered |
| `smart_contract` | A contract, not a wallet — USDT sent to it is usually lost unless the contract is built to receive it |
| `spam_sender` | A confirmed mass-spam address — almost never a genuine customer destination |

### After you receive (deposits)

When USDT arrives, the party to screen is **whoever sent it**, not your deposit address. Pass the deposit's id and the check runs on that deposit's on-chain sender — the same sender reported as `fromAddress` in your `deposit.confirmed` webhook:

```json
{ "depositId": "cmg9…" }
```

- Run it **once the deposit is confirmed and before you credit the customer or release goods**. A blockchain transfer cannot be refused: the funds have already arrived. The check tells you whether to **hold** them for review instead of crediting.
- On `blocked: true` with `aml_risk`, freeze the customer's credit and follow your compliance procedure. Don't send the funds back yourself without that review: returning funds to a sanctioned address is itself a transfer to it.
- `usdt_frozen` is rare for senders, because a frozen address cannot send USDT. Seeing it means the address was frozen after the payment.
- `smart_contract` is normal for payments from some exchanges and payment processors. Treat it as information, not a block.
- You can also screen a customer's **declared wallet before they pay**, with `address`, if you whitelist customer wallets during onboarding.

## Run a check

```http
POST /api/compliance-checks
Authorization: <API key>
Content-Type: application/json

{
  "address": "TTVUvAWUZafeHqQP82sYnfk4jNFPEHCRpL",
  "referenceId": "payout-88412"
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `address` | one of `address` / `depositId` | TRON address (`T…`, 34 characters) |
| `depositId` | one of `address` / `depositId` | A TRON deposit you received; its on-chain sender is screened |
| `network` | no | `TRX` (default) — TRON only |
| `referenceId` | no | Your own id, unique per merchant; a reused value returns `409` and charges nothing |
| `feeCurrency`, `feeNetwork` | no | The balance that pays — both or neither |

Only write-capable merchant API keys can run checks.

### Response

```json
{
  "success": true,
  "check": {
    "id": "cmgb1x0c90004",
    "referenceId": "payout-88412",
    "address": "TTVUvAWUZafeHqQP82sYnfk4jNFPEHCRpL",
    "network": "TRX",
    "depositId": null,
    "verdict": {
      "blocked": true,
      "category": "aml_risk",
      "severity": "BLOCK",
      "reasonCode": "SANCTIONS_SCREENING",
      "label": "DPRK Bitget Exploit - September 2026",
      "checkedAt": "2026-10-05T08:00:00+00:00"
    },
    "fee": { "amount": "1", "currency": "USDT", "network": "TRX", "status": "CHARGED" },
    "createdAt": "2026-10-05T08:00:00.120Z"
  }
}
```

| Field | Meaning |
|-------|---------|
| `verdict.blocked` | `true` = the address is on the deny-list. `false` = `category: "clean"` |
| `verdict.category` | `clean`, `aml_risk`, `usdt_frozen`, `smart_contract` or `spam_sender` |
| `verdict.reasonCode` | For `aml_risk`: the source, e.g. `SANCTIONS_OFAC_SDN`, `SANCTIONS_EU_FSF`, `SANCTIONS_UK_FCDO`, or `SANCTIONS_SCREENING` for commercial findings |
| `verdict.label` | Human-readable label when known (for example the incident name) |
| `verdict.checkedAt` | When the deny-list answered |

A clean address returns `"blocked": false, "category": "clean"`; `severity`, `reasonCode` and `label` are `null`.

## Get and list checks

```http
GET /api/compliance-checks/{checkId}
GET /api/compliance-checks?blocked=true&page=1&limit=20
```

Re-reading a stored verdict is free and never re-checks. Run a new check when you need a current verdict — lists change daily.

List filters (all optional): `page`, `limit` (max 100), `blocked`, `category`, `address`, `depositId`, `referenceId`. The response also carries `service.feePerCheck` and `service.supportedNetworks`.

## Errors

Nothing is charged for any error.

| HTTP | `code` | When |
|------|--------|------|
| 400 | `INVALID_REQUEST` | Both or neither of `address` / `depositId` sent |
| 400 | `INVALID_ADDRESS` | Not a valid TRON address |
| 400 | `NETWORK_NOT_SUPPORTED` | Not TRON (including a `depositId` on another network) |
| 400 | `INSUFFICIENT_BALANCE` | No balance (or not the chosen one) covers the fee |
| 403 | `MERCHANT_REQUIRED` / `MERCHANT_INACTIVE` | Non-merchant key, or the merchant account is not active |
| 404 | `DEPOSIT_NOT_FOUND` | Unknown `depositId`, or not yours |
| 409 | `DUPLICATE_REFERENCE` | `referenceId` already used |
| 422 | `DEPOSIT_SENDER_UNKNOWN` | The deposit's sender isn't recorded — screen the address directly |
| 429 | `RATE_LIMITED` | Too many checks — retry shortly |
| 503 | `COMPLIANCE_UNAVAILABLE` | No verdict could be produced. Retry, and **do not treat the address as clean** in the meantime |

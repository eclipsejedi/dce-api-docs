---
title: AML Address Checks
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-10-05_

Screen any wallet address for anti-money-laundering (AML) risk before you accept funds from it or pay out to it. A check returns an **Elliptic** risk score, a risk level, sanctions exposure and the risk detail behind the score.

For a quick, cheaper yes/no verdict on a TRON address (sanctions, Tether freezes, scam labels), see [TRON Address Compliance Checks](https://docs.dcepay.io/docs/compliance-checks).

## Overview

1. **Run a check** — `POST /api/aml-checks` with the address and its network.
2. **Read the result** — most checks settle within seconds and return the result directly. If the check is still running you get `202` with `status: "PENDING"`; poll `GET /api/aml-checks/{checkId}` until it settles.
3. **Decide** — apply your own policy to `riskLevel`, `riskScore` and `isSanctioned` (for example, hold deposits from `high`-risk or sanctioned addresses for manual review).

## Price

| | |
|---|---|
| Price per check | **1.50**, in-kind: 1.50 USDT or 1.50 USDC from one of your balances |
| Charged when | a result is delivered (`status: "COMPLETED"`) |
| Not charged | `SKIPPED` — the address has no on-chain activity, so there is nothing to screen; `FAILED` — the provider could not complete the check |

The fee is **reserved** from your balance when the check starts (it shows under `pending` in `GET /api/balance`), then either **charged** on a result or **returned** to `available`. It appears in your fee ledger as an `AML Check Fee` charge.

**Which balance pays:** send `feeCurrency` and `feeNetwork` together to choose. Without them the fee is taken from the screened network's balance (USDT first, then USDC), then from USDT on TRX, then from your largest balance. A check is refused with `400 INSUFFICIENT_BALANCE` if no balance can cover the fee, and nothing is reserved.

## Supported networks

AML checks cover the networks live on DCE:

| `network` | Chain | Status |
|-----------|-------|--------|
| `TRX` | Tron | **Available** |
| `BNB` | BNB Smart Chain | **Available** |
| `POL` | Polygon PoS | **Available** |
| `ETH` | Ethereum | Available once Ethereum goes live on DCE |

Testnet networks (`TRX_SHASTA`, `tBNB`, `POL_AMOY`, `SEP`) cannot be screened. `GET /api/aml-checks` returns the networks you can screen right now in `service.supportedNetworks`.

## Run a check

```http
POST /api/aml-checks
Authorization: <API key>
Content-Type: application/json

{
  "address": "TLa2f6VPqDgRE67v1736s7bJ8Ray5wYjU7",
  "network": "TRX",
  "referenceId": "kyc-customer-1842"
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `address` | yes | Must be a valid address for `network` (`T…` on TRX, `0x…` on EVM networks) |
| `network` | yes | See [Supported networks](#supported-networks) |
| `referenceId` | no | Your own id, unique per merchant. A reused value returns `409 DUPLICATE_REFERENCE` and runs (and charges) nothing |
| `feeCurrency`, `feeNetwork` | no | The balance that pays the fee — send both or neither |

Only write-capable merchant API keys can run checks.

### Response

`200` once settled, `202` while pending:

```json
{
  "success": true,
  "check": {
    "id": "cmg4x0amz0001",
    "referenceId": "kyc-customer-1842",
    "address": "TLa2f6VPqDgRE67v1736s7bJ8Ray5wYjU7",
    "network": "TRX",
    "provider": "elliptic",
    "status": "COMPLETED",
    "riskScore": "7.4",
    "riskLevel": "high",
    "isSanctioned": true,
    "sanctions": {
      "self": false,
      "exposure": { "share": 12.5, "hops": 2, "proximity": "indirect" }
    },
    "fee": { "amount": "1.5", "currency": "USDT", "network": "TRX", "status": "CHARGED" },
    "error": null,
    "result": {
      "risk_score_detail": { "source": 7.4, "destination": 0 },
      "contributions": { "source": [], "destination": [] },
      "cluster_entities": [],
      "evaluation_detail": { "source": [], "destination": [] },
      "detected_behaviors": []
    },
    "createdAt": "2026-09-29T09:00:00.000Z",
    "completedAt": "2026-09-29T09:00:04.000Z"
  }
}
```

| Field | Meaning |
|-------|---------|
| `status` | `PENDING` (running, fee reserved) · `COMPLETED` (result delivered, fee charged) · `SKIPPED` (no on-chain activity, not charged) · `FAILED` (provider error, not charged — see `error`) |
| `riskScore` | Decimal string from `0` to `10`. `null` means no risk rule was triggered |
| `riskLevel` | Always one of `low` (0–3), `medium` (3–7), `high` (7–10), or `null` |
| `isSanctioned` | `true` when the address has exposure to sanctioned entities |
| `sanctions.self` | `true` when the screened address itself is on a sanctions list |
| `sanctions.exposure` | The largest sanctions link: `share` (% of funds), `hops` (distance), `proximity` |
| `result` | The provider's risk detail: fund-flow contributions, known cluster entities, triggered rules, detected behaviours |
| `fee` | `amount`, `currency`, `network` of the balance that pays, and `status` (`RESERVED` · `CHARGED` · `REFUNDED`) |

**A completed check with no risk data.** In rare cases the provider completes and bills a check but returns no risk data. The check is `COMPLETED` and charged, `riskScore`, `riskLevel` and `isSanctioned` are all `null`, and `error` explains this. **Do not read it as clean** — contact support with the check `id`, or re-screen.

## Get a check

```http
GET /api/aml-checks/{checkId}
```

Returns the check with the full `result`. A `PENDING` check is refreshed from the provider when you read it, so poll every few seconds. Most checks settle within seconds; complex address histories can take up to about 3 minutes. A check still pending after an hour is marked `FAILED` and its fee is returned. An unknown id — or another merchant's check — returns `404`.

## List checks

```http
GET /api/aml-checks?status=COMPLETED&network=TRX&page=1&limit=20
```

Query parameters (all optional): `page`, `limit` (max 100, default 20), `status`, `network`, `address`, `referenceId`. Rows omit the `result` detail. The response also carries `service`:

```json
{
  "checks": [ { "id": "cmg4x0amz0001", "status": "COMPLETED", "riskLevel": "high", "...": "..." } ],
  "pagination": { "total": 42, "page": 1, "limit": 20, "pages": 3 },
  "service": { "provider": "elliptic", "feePerCheck": "1.5", "supportedNetworks": ["BNB", "POL", "TRX"] }
}
```

## Errors

| HTTP | `code` | When |
|------|--------|------|
| 400 | `NETWORK_NOT_SUPPORTED` | The network cannot be screened (response lists `supportedNetworks`) |
| 400 | `INVALID_ADDRESS` | The address is not valid for the network |
| 400 | `INSUFFICIENT_BALANCE` | No balance (or not the chosen one) covers the fee; `fee` is included |
| 400 | — | Validation failure (`details` lists the fields) |
| 403 | `MERCHANT_REQUIRED` / `MERCHANT_INACTIVE` | Non-merchant key, or the merchant account is not active |
| 409 | `DUPLICATE_REFERENCE` | `referenceId` already used |
| 429 | `RATE_LIMITED` | Platform-wide screening capacity reached — retry within a minute; nothing was reserved |
| 503 | `AML_UNAVAILABLE` | Screening temporarily unavailable |

## Good practice

- Screen **before** crediting or paying out, and store the `id` with your customer record — results are kept and can be re-read at any time without a new charge.
- Re-screening the same address runs a new check and is charged again; reuse a recent result where your policy allows.
- Set `referenceId` so a retried request after a timeout returns `409` instead of charging a second check.
- A risk result is an input to your compliance decision, not the decision itself.

---
title: Error Handling
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-07-30_

DCE API errors use standard HTTP status codes and JSON bodies. This guide explains what to expect and how to handle failures safely.

## Response patterns

Success responses are route-specific JSON objects.

Error responses commonly include:

```json
{
  "error": "Message describing failure"
}
```

Some routes add a human-readable `message` alongside `error`, and validation routes may include structured fields such as `details`.

## Status code guide

| Code | Meaning | Typical action |
|---|---|---|
| `200` | Success | Process response |
| `400` | Validation or business-rule failure | Fix request payload/parameters |
| `401` | Missing/invalid auth | Check API key and header |
| `403` | Permission denied or merchant account not active | Verify key scope/role and account status |
| `404` | Resource not found | Validate IDs/paths |
| `409` | Conflict (e.g., duplicate `referenceId`) | Do not retry with the same reference unless you intend a new action |
| `410` | Gone (e.g., consumed/expired deposit token) | Regenerate token/session |
| `413` | Payload too large | Reduce payload size |
| `429` | Rate limit reached | Retry with backoff |
| `500` | Internal server error | Retry later and contact support if persistent |
| `503` | Service temporarily unavailable (e.g., withdrawals paused) | Retry shortly or contact support |

## Common error cases

### 409 — Duplicate referenceId (withdrawals)

Your `referenceId` is unique per merchant account. Submitting a withdrawal with a `referenceId` that already exists returns `409` — a second payout is never created:

```json
{
  "error": "Duplicate referenceId",
  "message": "A withdrawal with this referenceId already exists for your merchant account. Use a new referenceId to submit a new withdrawal."
}
```

Treat this as "already submitted": look up the existing withdrawal rather than retrying. If a withdrawal **fails**, its `referenceId` is automatically released so the same reference can be retried.

### 403 — Merchant account not active

Suspended or inactive merchant accounts cannot initiate withdrawals or create deposit URLs. Withdrawal attempts return:

```json
{
  "error": "Merchant account is not active"
}
```

This is not retryable — contact support (or your reseller) to reactivate the account.

### 400 — Disabled or unknown (currency, network) pair

Balances and payments are per-`(currency, network)` pair, and each pair must be enabled for the requested direction. Currently only **USDT on TRX** (and its Shasta testnet) is enabled; other pairs are coming soon / on request. Requests referencing a disabled pair are rejected:

```json
{
  "error": "Withdrawals of USDT on ETH are not supported"
}
```

```json
{
  "error": "Deposits of USDC on SOL are not supported"
}
```

### 400 — Insufficient per-chain balance

Withdrawals draw only from the balance on the requested network — funds on other networks cannot be used. The rejection includes the per-chain context and the fee quote:

```json
{
  "error": "Insufficient balance",
  "message": "Insufficient USDT balance on TRX. Deposits and withdrawals are per-chain: funds on other networks cannot be used.",
  "available": "50.25",
  "requested": "100.50",
  "currency": "USDT",
  "network": "TRX",
  "feeInfo": {
    "grossAmount": "100.5",
    "netAmount": "98.5",
    "totalFees": "2",
    "feeBreakdown": {
      "baseFee": "1",
      "markupRate": "0",
      "markupAmount": "1",
      "networkFee": "1",
      "totalFee": "2"
    }
  }
}
```

Remember that fees are charged on top: the total debit is `amount + commission + networkFee`.

### 503 — Withdrawals temporarily unavailable

Withdrawals may be temporarily paused platform-side. Nothing is created and your `referenceId` is not consumed, so the same request can be retried later:

```json
{
  "error": "Withdrawal temporarily unavailable",
  "message": "Withdrawals are temporarily unavailable. Please try again shortly or contact support."
}
```

### 500 — Payout initiation failed (funds returned)

If a payout cannot be submitted after the withdrawal was created, the reserved amount (including fees) is returned to your balance and the `referenceId` is released for retry:

```json
{
  "error": "Payout initiation failed",
  "message": "The withdrawal could not be processed. The reserved amount has been returned to your balance — please try again shortly or contact support.",
  "withdrawalId": "cmdl8u2xq0001abcd1234efgh"
}
```

### 400 — Validation errors

Malformed payloads return field-level details:

```json
{
  "error": "Invalid request data",
  "details": [
    {
      "field": "amount",
      "message": "Amount must be a numeric string with up to 18 decimal places"
    }
  ]
}
```

## Retry strategy

Retry only transient failures:
- `429`, `500`, `503`, network timeout/connection issues

Do **not** blindly retry:
- `400`, `401`, `403`, `409`, most `404` (unless eventual consistency is expected)

For `409 Duplicate referenceId` specifically: the original request already exists — query it instead of resubmitting.

Recommended backoff:
- Exponential backoff + jitter (for example 1s, 2s, 4s, 8s with randomness)
- Max attempts: 3-5 depending on operation criticality

## Safe integration patterns

- Treat all amount fields as decimal strings (send them as strings too, e.g. `"100.50"`).
- Use your own idempotency reference (`referenceId`) to prevent duplicate business actions — reuse of a `referenceId` never creates a second payout.
- Webhook delivery is at-least-once: deduplicate on the payload's `eventId` and make handlers idempotent.
- Log request context (route, status, payload hash, timestamps) without logging secrets.

## Example error handling (Node.js)

```javascript
const response = await fetch(url, options);
const body = await response.json().catch(() => ({}));

if (!response.ok) {
  if ([429, 500, 503].includes(response.status)) {
    // retry with backoff
  } else if (response.status === 409) {
    // duplicate referenceId — the withdrawal already exists; look it up
  } else {
    // handle as non-retryable
    throw new Error(body.message || body.error || `HTTP ${response.status}`);
  }
}
```

## When to contact support

Reach out with:
- environment (`staging` or `production`)
- endpoint + approximate timestamp (UTC)
- status code + response body snippet
- correlation/request ID if available

Contact: [hello@dcepay.io](mailto:hello@dcepay.io)

---
title: Troubleshooting Guide
deprecated: false
hidden: false
metadata:
  robots: index
---
Use this checklist to quickly diagnose common DCE API integration issues.

## 1) Authentication problems

### Symptom: `401 Unauthorized`

Checks:
- Confirm `Authorization` header is present.
- Confirm API key is valid and not revoked.
- Confirm you are using the correct environment host for that key.

Quick test:

```bash
curl -i "${DCE_BASE_URL}/api/balance?currency=USDT&network=TRX" \
  -H "Authorization: ${DCE_API_KEY}"
```

### Symptom: `403 Forbidden`

Checks:
- Key is valid but permission scope does not allow the route.
- Ensure the integration user/merchant role has required access.
- `Merchant account is not active`: the merchant is suspended/inactive — withdrawals and deposit-URL creation are blocked until reactivated.

## 2) Validation and request errors (`400`, `409`)

Checks:
- Required fields exist (withdrawals require `network`).
- Enum values are valid (`currency`: `USDT`/`USDC`; supported `network` values).
- The (currency, network) pair is enabled — only **USDT on TRX** today; disabled pairs return errors like `Withdrawals of USDT on ETH are not supported`.
- Data types are correct (amounts as decimal strings where required).
- `Insufficient balance` is **per-chain**: funds on other networks cannot cover a withdrawal on this one.
- `409 Duplicate referenceId`: the withdrawal `referenceId` was already used by your merchant account — use a new one (reuse never creates a second payout).

## 3) Webhook issues

### Symptom: signature verification fails

Checks:
- Verify against raw request body bytes.
- Use the merchant `webhookSecret` exactly.
- Compare lowercase hex digest.

DCE webhook headers:
- `X-Webhook-Event`
- `X-Webhook-Signature`

### Symptom: duplicate webhook events

Checks:
- Delivery is **at-least-once** (durable outbox with retries) — duplicates are expected behavior, not a bug.
- Implement idempotency keyed by `eventId` (`X-Webhook-Id`) or event + reference identifiers.
- Always return `2xx` once processed successfully.

### Symptom: retries continue

Checks:
- Your endpoint returns non-2xx, times out, or network fails.
- Respond quickly; move heavy work to async queue.
- Returning JSON `{ "ok": true }` can stop retries for that attempt path.

## 4) Deposit page token errors

### `400` Missing token
- Add `token` query parameter.

### `404` Invalid token
- Verify token source and freshness.

### `410` Token expired/used
- Generate a new deposit session token (sessions expire 15 minutes after creation).

These apply to both `GET /api/deposit-page` and the live status endpoint `GET /api/deposit-page/status`.

## 5) Withdrawal temporarily unavailable (`503`)

`{"error": "Withdrawal temporarily unavailable"}` means the platform's liquidity guard rejected the payout up-front. Nothing was created and your `referenceId` was not consumed — retry shortly or contact support.

## 6) Timeout or intermittent failures

Checks:
- Verify DNS/firewall and TLS setup.
- Add retry with exponential backoff for transient errors.
- Monitor latency and keep webhook handler response time low.

## 7) Before contacting support

Gather:
- environment (`staging`/`production`)
- endpoint + method
- UTC timestamp
- request payload sample (sanitized)
- response body + status
- correlation/request ID if available

Contact: [hello@dcepay.io](mailto:hello@dcepay.io)

---
title: Troubleshooting Guide
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

Use this checklist to quickly diagnose common DCE API integration issues.

## 1) Authentication problems

### Symptom: `401 Unauthorized`

Checks:

- Confirm `Authorization` header is present.
- Confirm API key is valid and not revoked.
- Confirm you are using the correct environment host for that key.

Quick test:

```bash
curl -i "${DCE_BASE_URL}/api/balance?currency=USD" \
  -H "Authorization: ${DCE_API_KEY}"
```

### Symptom: `403 Forbidden`

Checks:

- Key is valid but permission scope does not allow the route.
- Ensure the integration user/merchant role has required access.

## 2) Validation and request errors (`400`)

Checks:

- Required fields exist.
- Enum values are valid (for example supported `network` values).
- Data types are correct (amounts as strings where required).

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

- Implement idempotency keyed by event + transaction/reference identifiers.
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

- Generate a new deposit session token.

## 5) Timeout or intermittent failures

Checks:

- Verify DNS/firewall and TLS setup.
- Add retry with exponential backoff for transient errors.
- Monitor latency and keep webhook handler response time low.

## 6) Before contacting support

Gather:

- environment (`staging`/`production`)
- endpoint + method
- UTC timestamp
- request payload sample (sanitized)
- response body + status
- correlation/request ID if available

Contact: [hello@dcepay.io](mailto:hello@dcepay.io)

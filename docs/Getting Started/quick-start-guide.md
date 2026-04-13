---
title: Quick Start Guide
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

This guide gets you from credentials to a working DCE API integration quickly.

## Prerequisites

- DCE API credentials (API key, and webhook secret if using webhooks)
- A backend service where secrets can be stored securely
- HTTPS endpoint for webhook testing (ngrok is fine for local dev)
- Basic familiarity with JSON + HTTP APIs

## 1) Choose environment

| Environment | Base URL                    |
| ----------- | --------------------------- |
| Production  | `https://api.dcepay.io`     |
| Staging     | `https://staging.dcepay.io` |

All endpoints are under `/api`.

## 2) Configure environment variables

```bash
DCE_BASE_URL=https://staging.dcepay.io
DCE_API_KEY=your_api_key
```

## 3) Make your first authenticated request

Use your API key in `Authorization` as either:

- raw key (`Authorization: <key>`), or
- bearer format (`Authorization: Bearer <key>`)

```bash
curl -sS "${DCE_BASE_URL}/api/balance?currency=USD" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

Expected response shape:

```json
{
  "currency": "USD",
  "available": "0",
  "pending": "0"
}
```

## 4) Create a deposit address

```bash
curl -sS -X POST "${DCE_BASE_URL}/api/deposit-address" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "network": "TRX",
    "identifier": "customer_123",
    "referenceId": "order_1001"
  }'
```

The response includes the generated address and metadata you can store for the customer/order.

## 5) Optional: create a hosted deposit URL

```bash
curl -sS -X POST "${DCE_BASE_URL}/api/deposit-url" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "network": "TRX",
    "identifier": "customer_123",
    "referenceId": "order_1001"
  }'
```

## 6) Enable webhooks

Configure your merchant webhook settings (`webhookUrl`, `webhookSecret`, `webhookEnabled`, optional `webhookEvents`).

DCE sends outbound POSTs with:

- `Content-Type: application/json`
- `X-Webhook-Event`
- `X-Webhook-Signature` (hex HMAC-SHA256 of raw body with your webhook secret)

Use idempotency on your side and acknowledge successful processing with HTTP `2xx`.

## 7) Validate and go live

Before production:

- Test deposits, withdrawals, and webhook delivery in staging
- Confirm retry handling on webhook failures/timeouts
- Verify monitoring for `401`, `403`, `429`, and `5xx` patterns

## Useful links

- [Welcome & getting started](welcome-and-getting-started.md)
- [Authentication setup](authentication.md)
- [Webhooks](webhooks.md)
- [Error handling](error-handling.md)
- [Troubleshooting](troubleshooting.md)
- [Security guide](security-guide.md)

***

Need implementation help or a review of your integration sequence? Contact [hello@dcepay.io](mailto:hello@dcepay.io).
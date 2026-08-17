---
title: Quick Start Guide
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-07-30_

This guide gets you from credentials to a working DCE API integration quickly.

## Prerequisites

- DCE API credentials (API key, and webhook secret if using webhooks)
- A backend service where secrets can be stored securely
- HTTPS endpoint for webhook testing (ngrok is fine for local dev)
- Basic familiarity with JSON + HTTP APIs

## 1) Choose environment

| Environment | Base URL |
|---|---|
| Production | `https://api.dcepay.io` |
| Staging | `https://api.dcepay.dev` |

All endpoints are under `/api`.

**Supported assets:** balances and money movement are per (currency, network) pair. **USDT on TRX** is what is available today (use `TRX_SHASTA` on staging); USDT/USDC on ETH, BNB, and SOL are coming soon and can be enabled on request.

## 2) Configure environment variables

```bash
DCE_BASE_URL=https://api.dcepay.dev
DCE_API_KEY=your_api_key
```

## 3) Make your first authenticated request

Use your API key in `Authorization` as either:
- raw key (`Authorization: <key>`), or
- bearer format (`Authorization: Bearer <key>`)

```bash
curl -sS "${DCE_BASE_URL}/api/balance?currency=USDT&network=TRX" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

Expected response shape:

```json
{
  "currency": "USDT",
  "network": "TRX",
  "available": "0",
  "pending": "0"
}
```

Balances are per (currency, network) pair — call `GET /api/balance` without parameters to list every balance row your account holds.

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

## 6) Create a withdrawal

Withdrawals require a `network` and draw only from that chain's balance. Include a unique `referenceId` for idempotency — reusing one never creates a second payout (the duplicate returns `409`).

```bash
curl -sS -X POST "${DCE_BASE_URL}/api/withdrawals" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "50.00",
    "currency": "USDT",
    "network": "TRX",
    "destination": "TDestinationAddress...",
    "referenceId": "payout_1001"
  }'
```

## 7) Enable webhooks

Configure your merchant webhook settings (`webhookUrl`, `webhookSecret`, `webhookEnabled`, optional `webhookEvents`).

DCE sends outbound POSTs with:
- `Content-Type: application/json`
- `X-Webhook-Event`
- `X-Webhook-Signature` (hex HMAC-SHA256 of raw body with your webhook secret)

Use idempotency on your side and acknowledge successful processing with HTTP `2xx`.

## 8) Validate and go live

Before production:
- Test deposits, withdrawals, and webhook delivery in staging
- Confirm retry handling on webhook failures/timeouts
- Verify monitoring for `401`, `403`, `429`, and `5xx` patterns

## Useful links

- [Welcome & getting started](https://docs.dcepay.io/docs/welcome-and-getting-started)
- [Authentication setup](https://docs.dcepay.io/docs/authentication)
- [Webhooks](https://docs.dcepay.io/docs/webhooks)
- [Error handling](https://docs.dcepay.io/docs/error-handling)
- [Troubleshooting](https://docs.dcepay.io/docs/troubleshooting)
- [Security guide](https://docs.dcepay.io/docs/security-guide)

---

Need implementation help or a review of your integration sequence? Contact [hello@dcepay.io](mailto:hello@dcepay.io).

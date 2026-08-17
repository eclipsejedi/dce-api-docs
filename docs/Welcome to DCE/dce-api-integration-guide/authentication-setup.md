---
title: Authentication Setup
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-07-30_

The DCE API uses API-key authentication.

For backend onboarding details after account creation, see the [Merchant Authentication Guide](https://docs.dcepay.io/docs/merchant-authentication-guide).

## Authentication model

- Most routes require `Authorization`.
- `GET /api/deposit-page` and `GET /api/deposit-page/status` are public and use a `token` query parameter instead of API key auth.
- API keys must be kept server-side only.

## Header format

DCE accepts either format below:

```http
Authorization: <your_api_key>
```

```http
Authorization: Bearer <your_api_key>
```

## Environment setup

```bash
DCE_BASE_URL=https://api.dcepay.dev
DCE_API_KEY=your_api_key
```

## Example request

```bash
curl -sS "${DCE_BASE_URL}/api/transactions?page=1&limit=20" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

## Common auth failures

### 401 Unauthorized

Typical causes:
- Missing `Authorization` header
- Invalid / revoked API key
- Wrong environment key (staging key against production or vice versa)

### 403 Forbidden

Typical causes:
- API key is valid but lacks required permission for that route (e.g., a read-only key calling `POST /api/withdrawals`)
- Merchant account is suspended or inactive — withdrawals and deposit-URL creation return `403` `Merchant account is not active`

## Best practices

- Store API keys in a secret manager, not source code.
- Use separate credentials for staging and production.
- Rotate keys on a defined schedule and on any suspected exposure.
- Never call DCE directly from browser/mobile clients with merchant API keys.
- Log request IDs/correlation IDs and status codes for support diagnostics.

## Related docs

- [Quick Start Guide](https://docs.dcepay.io/docs/quickstart)
- [Security Guide](https://docs.dcepay.io/docs/security-guide)
- [Error Handling](https://docs.dcepay.io/docs/error-handling)

---

Questions about onboarding or credential setup: [hello@dcepay.io](mailto:hello@dcepay.io)

---
title: Merchant Authentication Guide
deprecated: false
hidden: false
metadata:
  robots: index
---
This guide explains how merchants should authenticate from their backend systems after account creation.

## 1) Account Creation and Credential Handoff

After your merchant account is created:

1. You receive API credentials through your secure onboarding channel.
2. You store credentials in your backend secret manager.
3. You never expose credentials in frontend or mobile client code.

## 2) Required header

All authenticated server-to-server calls must include your API key in `Authorization`. The API accepts **either**:

```http
Authorization: <your_api_key>
```

```http
Authorization: Bearer <your_api_key>
```

Do not expose your API key to browsers or mobile apps.

## 3) Backend-only integration pattern

- Keep all DCE API calls server-to-server.
- Frontend calls your backend; your backend calls the DCE API.
- Do not proxy raw API keys to browsers.

## 4) Storage and Rotation

- Store the key in a managed secret store (AWS Secrets Manager, GCP Secret Manager, Vault, etc.).
- Rotate keys on a fixed schedule (recommended every 60-90 days).
- Revoke keys immediately when leaked or when staff access changes.

## 5) Minimum Production Controls

- Restrict outbound IP ranges where possible.
- Enable request logging with correlation IDs.
- Alert on repeated `401`/`403` responses and unusual request volume.
- Separate test and production credentials.

## 6) Example Server Request

```bash
curl -X GET "${DCE_BASE_URL}/api/balance?currency=USD" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

## 7) Common Errors

- `401`: missing/invalid API key
- `403`: authenticated but not permitted for endpoint scope
- `429`: rate limit exceeded; back off and retry based on headers

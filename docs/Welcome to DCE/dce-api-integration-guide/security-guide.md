---
title: Security Guide
deprecated: false
hidden: false
metadata:
  robots: index
---
This guide describes baseline security controls for integrating with DCE API.

## 1) Protect API credentials

- Keep API keys server-side only.
- Store secrets in a managed secret store (AWS Secrets Manager, GCP Secret Manager, Vault, etc.).
- Do not commit secrets to git, logs, or client apps.
- Rotate keys regularly and immediately on suspected exposure.

## 2) Secure API access

- Use HTTPS only.
- Prefer least-privilege credentials for each integration component.
- Separate staging and production credentials.
- Alert on unusual `401` / `403` / `429` spikes.

## 3) Webhook security

For merchant outbound webhook consumption:

- Validate `X-Webhook-Signature` using HMAC-SHA256.
- Compute signature over the **raw** request body bytes using your `webhookSecret`.
- Use constant-time compare for signature matching.
- Reject invalid signatures with non-2xx.

Recommended extra controls:
- enforce HTTPS endpoint
- IP allowlisting when possible
- replay protection via event/timestamp/idempotency checks

## 4) Data handling

- Treat transaction metadata as sensitive business data.
- Minimize PII collection and retention.
- Redact secrets and personal data in logs.
- Encrypt data at rest and in transit according to your policy.

## 5) Operational security

- Centralize audit logs for auth attempts, webhook verification failures, and admin actions.
- Set alerts for unusual request volume and repeated failures.
- Run periodic access reviews for users, keys, and environments.

## 6) Incident response (minimum)

If compromise is suspected:
1. Revoke/rotate affected API keys and webhook secrets.
2. Isolate impacted systems.
3. Review logs and timeline.
4. Reprocess failed financial events safely with idempotency checks.
5. Contact DCE support for coordinated investigation.

## 7) Security contact

Report security concerns to [hello@dcepay.io](mailto:hello@dcepay.io) with subject line `Security`.

Do not include plaintext production secrets in email.

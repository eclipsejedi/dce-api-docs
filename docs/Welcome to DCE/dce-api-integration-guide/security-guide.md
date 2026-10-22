---
title: Security Guide
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-10-21_

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

## 7) Screen counterparties

Check who you pay and who pays you, and record the result with the transaction:

- **Before a payout:** screen the destination address. On TRON, [TRON Address Compliance Checks](https://docs.dcepay.io/docs/compliance-checks) return an instant deny-list verdict (sanctions, fraud labels, Tether USDT freezes, contracts) for 1.00 per verdict.
- **After a deposit:** screen the sender before you credit the customer or release goods. A compliance check with `depositId` screens that deposit's on-chain sender. Funds that have already arrived cannot be refused, so hold them for review rather than returning them to a flagged address yourself.
- **For a risk score:** [AML Address Checks](https://docs.dcepay.io/docs/aml-checks) return an Elliptic risk score (0–10), risk level and sanctions exposure on `TRX`, `BNB` and `POL` for 1.50 per delivered result.

Set your own policy (for example, hold `high` risk or `blocked: true` for manual review). DCE returns the data; the decision is yours.

## 8) Security contact

Report security concerns to [hello@dcepay.io](mailto:hello@dcepay.io) with subject line `Security`.

Do not include plaintext production secrets in email.

---
title: '# Welcome to DCE'
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

DCE is a cryptocurrency payments and financial services platform. The **DCE API** lets you accept deposits, manage withdrawals, query balances and transaction history, use hosted deposit flows, and receive **webhooks** when money movement and ledger events occur—all over a straightforward JSON HTTP API.

This page is the front door: what the product is, how to start integrating, where the technical detail lives, and how to reach the team.

***

## What you can do with the API

- **Deposits** — Allocate chain deposit addresses and optional hosted deposit pages for end users.
- **Withdrawals (payouts)** — Create and track withdrawals with configurable fees and balance checks.
- **Balances & transactions** — Read per-currency balances and a unified transaction history.
- **Exchange rates** — Fetch rates for display or reconciliation.
- **Settlements & reconciliation** — Work with settlement workflows and reporting where your account allows it.
- **Webhooks** — Receive signed POST callbacks to your HTTPS URL for deposit, withdrawal, and transaction lifecycle events.

All merchant routes live under `/api`. Authenticate with your API key in the `Authorization` header (raw key is supported; a `Bearer` prefix is optional for compatibility). Treat keys as **server-side secrets** only.

***

## Environments

| Environment | Base URL                    |
| ----------- | --------------------------- |
| Production  | `https://api.dcepay.io`     |
| Staging     | `https://staging.dcepay.io` |

Append path `/api/...` to these hosts when calling endpoints (for example `https://api.dcepay.io/api/balance?currency=USD`).

Use staging for development and integration testing when your account includes access; use production only for live funds and customers.

***

## Getting started (short path)

1. **Get access** — Obtain API credentials from your DCE contact. You need at least one API key and, if you use webhooks, a **webhook secret** and a publicly reachable **HTTPS** URL (ngrok or similar is fine for development).
2. **Read authentication** — See [Authentication Setup](authentication.md) and [Merchant Authentication Guide](merchant-authentication-guide.md) for headers, scopes, and safe handling of keys.
3. **Follow the quick start** — [Quick Start Guide](quickstart.md) walks through environment setup, first requests, and common patterns.
4. **Turn on webhooks** — Configure `webhookUrl`, `webhookSecret`, and `webhookEvents` on your merchant profile, then implement signature verification. Start with [Webhooks](webhooks.md) (especially _How the system delivers webhooks_).
5. **Validate in staging** — Exercise deposits, withdrawals, and webhook delivery against staging before going live.

When you are ready for detail, use [API Reference](api-reference.md), [Endpoint Summary](endpoint-summary.md), and [Partner API Scope](partner-api-scope.md) to see which routes are available to your integration.

***

## OpenAPI docs

- **OpenAPI (YAML)** — The merchant-oriented spec is generated as `openapi/v1/dce-api-openapi.yaml`. On a running DCE API deployment you can fetch it at `/api/openapi` (redirects may exist from legacy `/dce-api-openapi.yaml`).

Regenerate specs locally with:

```bash
npm run openapi:generate
```

***

## Contact us

We are glad to hear from merchants, partners, and teams exploring integrations.

| Topic                              | How to reach us                                                                                                                                       |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **General inquiries & onboarding** | [hello@dcepay.io](mailto:hello@dcepay.io)                                                                                                             |
| **Integration & API support**      | Same address—include your environment (staging/production), approximate timestamps, and request IDs or correlation IDs if you have them.              |
| **Collaboration & partnerships**   | [hello@dcepay.io](mailto:hello@dcepay.io) with “Partnership” or “Collaboration” in the subject and a short description of your use case and timeline. |
| **Security disclosures**           | Email [hello@dcepay.io](mailto:hello@dcepay.io) with “Security” in the subject; avoid posting sensitive material in public channels.                  |

We do not publish API keys or secrets over email. Use secure channels your account team provides for credential handoff when available.

***

## Related documentation

- [Quick Start Guide](quickstart.md)
- [Authentication Setup](authentication.md)
- [Webhooks](webhooks.md)
- [Error Handling](error-handling.md) & [Troubleshooting](troubleshooting.md)
- [Security Guide](security-guide.md)

If you are browsing this repository as a developer, the rest of the guides live in the [`docs/`](README.md) directory alongside this file.

***

_Last updated: April 2026_

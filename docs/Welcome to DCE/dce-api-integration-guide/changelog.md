---
title: Changelog
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-10-05_

Dated record of merchant-visible API and platform changes. The API reference header always shows the date of the most recent API change; entries here explain what changed. Dates are UTC.

## 2026-10-05

- **TRON address compliance checks.** New endpoints `POST /api/compliance-checks`, `GET /api/compliance-checks` and `GET /api/compliance-checks/{checkId}` return an instant deny-list verdict for a TRON address: `clean`, or blocked as `aml_risk` (sanctions — OFAC, EU, UK — or scam, ransomware, stolen-funds, fraud labels), `usdt_frozen` (USDT frozen by Tether), `smart_contract` or `spam_sender`. Screen a payout destination with `address` before sending, or screen who paid you with `depositId` (the check runs on the deposit's on-chain sender). **1.00 per verdict**, in-kind from your USDT/USDC balance; no verdict, no charge. See [TRON Address Compliance Checks](https://docs.dcepay.io/docs/compliance-checks).
- **AML checks: stricter result handling.** `riskLevel` is now always `low`, `medium`, `high` or `null`, with a level derived from `riskScore` when the provider sends an unrecognised value. Scores outside 0–10 are dropped. `isSanctioned` falls back to the `sanctions` block when the provider omits it. A check the provider completed and billed without returning risk data is now `COMPLETED` and charged, with all risk fields `null` and an explanatory `error`. See [AML Address Checks](https://docs.dcepay.io/docs/aml-checks#response).

## 2026-10-04

- **AML address checks.** New endpoints `POST /api/aml-checks`, `GET /api/aml-checks` and `GET /api/aml-checks/{checkId}` screen a wallet address with Elliptic and return a risk score (0–10), risk level, sanctions exposure and the risk detail. Available on the networks live on DCE — `TRX`, `BNB`, `POL` (`ETH` once Ethereum goes live). **1.50 per check**, taken in-kind from your USDT/USDC balance and charged only when a result is delivered; checks on addresses with no on-chain activity (`SKIPPED`) and failed checks are free. The fee appears in your fee ledger as `AML Check Fee`. See [AML Address Checks](https://docs.dcepay.io/docs/aml-checks).

## 2026-09-28

- **Wallet top-up fee on BSC reduced to 0.20.** A direct deposit to your own top-up address on BNB Smart Chain (`network: "BNB"`, USDT and USDC) is now charged a flat **0.20** instead of 1.20. The fee is capped per network, so a lower fee agreed for your account still applies. TRON top-ups are unchanged at 1.20. See the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).

## 2026-09-11

- **Webhook deliveries are signed only once you have generated a signing secret.** You generate and rotate the `whsec_…` secret yourself in the merchant dashboard (**Account → Integration → Webhook**, MFA step-up); it is shown once. Until a secret exists, deliveries are sent **without** an `X-Webhook-Signature` header. Previously the header was computed with an empty key when no secret had been issued, which looked signed but could not be verified. See [Webhooks — Webhook setup](https://docs.dcepay.io/docs/webhooks#webhook-setup).
- **API keys are generated on demand.** A merchant account no longer receives a pre-issued key at invite; generate your first key (and rotate it later) under **Account → Integration → API keys**. The plaintext is shown once.
- Admins now see whether a merchant has an active signing secret (masked prefix) on the merchant detail page; the raw secret is never displayed.

## 2026-09-09

- **Polygon (POL) deposits are live** for USDT and USDC (`network: "POL"`; testnet `POL_AMOY`) on the same deposit-address and deposit-URL endpoints as BSC. Deposits credit after 64 block confirmations (~2 min). Polygon withdrawals follow after the first production deposits have been observed; until then a withdrawal on a `POL` pair is rejected. Polygon is priced on the BSC schedule (deposit commission floor **0.10**, **no** first-deposit fee, withdrawal network fee **0.20**, minimum withdrawal **1**) — see the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).
- **One EVM deposit address per client across EVM networks.** A deposit address issued for a client on one EVM network (`BNB`, `POL`, later `ETH`) is the same address on the others: requesting an address for the same `identifier` on a sibling EVM network returns the identical address, registered on that network. Funds are still credited per `(currency, network)` — tell your customers which network to pay on — but a payment sent on the wrong EVM network is now recoverable by support. TRON addresses are unchanged.

## 2026-09-08

- **BSC (BNB Smart Chain) deposits are live** for USDT and USDC (`network: "BNB"`) on the same deposit-address and deposit-URL endpoints. Deposits credit after 15 block confirmations; BSC fee schedule as published in the [Fees Reference](https://docs.dcepay.io/docs/fees-reference). BSC withdrawals follow after the first production deposits have been observed.
- **BSC fee schedule, priced on BSC's own cost basis** (applies from the BSC go-live date): deposit commission floor **0.10** per deposit, **no** first-deposit fee, withdrawal network fee **0.20**, minimum withdrawal **1**. TRON fees are unchanged. See the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).
- The first-deposit fee is now capped so a deposit never credits below zero (relevant to a first deposit of exactly 1 USDT on TRON).
- **Balance precision.** Balance updates are now computed in exact decimal arithmetic in the database; previously each update could leave sub-micro dust (values like `0.20000000000000007`). Existing dust is rounded away to 6 decimals in the same deploy. No visible change beyond cleaner `available` / `pending` values.

## 2026-09-07

- **Withdrawal destination format is validated up front.** `POST /api/withdrawals` and `POST /api/settlements` reject a destination that is not syntactically valid for the requested network with `400` and `code: "INVALID_DESTINATION"` (for example a TRON address on an EVM network), before any balance is reserved. Previously such requests failed later and refunded.
- **Networks that are not enabled return `503`.** `POST /api/deposit-address` and `POST /api/deposit-url` answer `503` with `code: "NETWORK_NOT_ENABLED"` for a network the platform does not currently serve, instead of attempting a request to the retired legacy provider. Enabled networks are unchanged (USDT on TRX today).
- **`GET /api/exchange-rates` is served by the platform's own rate provider** (exchange spot prices + ECB cross-rates) for all requests. Response shape unchanged.
- **Preparation for BSC (BNB Smart Chain) support** — USDT/USDC on `BNB`, going live one network at a time after a production soak. On EVM networks deposits are credited after a confirmation depth (15 blocks on BSC) rather than on first sight; the merchant contract (addresses, webhooks, payloads) is unchanged. The BSC fee schedule is published in the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).
- Internal: the legacy operator ingestion route `POST /api/webhook/event` is retired (`410`). It was never a merchant endpoint.

## 2026-09-02

- **Fee schedule update, effective 2026-09-03 00:00 GMT+8 (2026-09-02 16:00 UTC).** Deposit commission floor 0.50 USDT per deposit (was 0.10); address activation fee 1.00 (was 0.50); wallet top-up flat fee 1.20 (was 1.00); withdrawal network fee 1.20 standard (was 1.00) and **2.50 when the destination address holds no USDT** at submission; withdrawal commission 0.1% with the network fee as a floor. The fee-exempt threshold for small deposits (1 USD) is unchanged. Full table in the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).
- Deposits of 10 USDT and above are consolidated to the platform wallet as part of confirmation; smaller deposits confirm on on-chain verification and consolidate later. No change to credited amounts.

## 2026-09-01

- **Withdrawal network fee is a floor under the commission, not an add-on.** Total fee = `max(commission, networkFee)`; the reported `commission` / `feeBreakdown.baseFee` is now only the margin above the network fee. Flat-commission payouts had been charged the network fee twice since 2026-07-31; the overcharge was refunded to affected balances.

## 2026-08-31

- **Webhook timestamps in GMT+8.** `deposit.confirmed`, `withdrawal.confirmed` and `transaction.confirmed` payloads carry `confirmedAt`, and the failed variants `failedAt`, as ISO-8601 with an explicit `+08:00` offset.
- **`GET /api/transactions` exposes `referenceId`** at the top level of each row (also mirrored at `metadata.webhookPayload.referenceId`), and `fee` is populated from the linked deposit/withdrawal for every row. Rows created between 2026-08-28 and 2026-08-31 that showed an empty reference or zero fee have been backfilled and their callbacks redelivered.
- **Per-endpoint circuit breaker on merchant webhooks.** After 10 consecutive delivery failures to your endpoint, deliveries pause for a cool-off (15 minutes, doubling to a 2-hour cap) and resume on a successful probe. Queued events are delivered once the endpoint recovers. See [Webhooks — Delivery and retries](https://docs.dcepay.io/docs/webhooks#delivery-and-retries).

## 2026-08-26

- **Payment infrastructure cutover (TRON).** Deposit addresses, sweeps and payouts on USDT-TRC20 now run on the platform's own rail. No API, authentication or payload changes; new deposit addresses should be requested per deposit as before.

## 2026-08-13

- **Hosted deposit page redesign.** The page served by `GET /api/deposit-page` has a new dark visual design. No behavioral changes: session tokens, expiry handling (404/410), live payment status polling, and redirect behavior are unchanged.
- Internal sweep-pipeline reliability improvements (duplicate-webhook handling, low-gas alerting). No merchant-facing contract changes.

## 2026-07-31

- **`withdrawal.confirmed` webhook: `feeCharges` is always present** — an empty array when no fees were charged, instead of being omitted. The `networkFee` field is now documented in [Webhooks](https://docs.dcepay.io/docs/webhooks).
- Wallet top-up (direct-deposit) flat fees are now recorded in the merchant charge ledger and appear in charge summaries.

## 2026-07-30

- **Multichain balances.** `GET /api/balance` responses are segmented per `(currency, network)` pair; each row includes `withdrawalEnabled`, `networkFee`, and `maxWithdrawable`. Deposits on a chain can only be withdrawn on that same chain. See [Balance Management](https://docs.dcepay.io/docs/balance-management).
- Withdrawal `referenceId` idempotency behavior documented in [Withdrawals](https://docs.dcepay.io/docs/withdrawals).

## 2026-07-24

- **Hosted deposit page live status.** The deposit page now shows live payment progress: waiting → detected → completed, with automatic redirect on terminal states.

## 2026-07-21

- **Sweep-gated crediting for wallet top-ups.** Direct wallet top-up deposits credit as `pending` and move to `available` once the funds are swept to the master wallet. API-created deposit addresses are unaffected.

## 2026-05-05

- **New endpoint: `GET /api/merchants/{merchantId}/charges`** — per-merchant fee ledger with filtering and summary totals. See the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).

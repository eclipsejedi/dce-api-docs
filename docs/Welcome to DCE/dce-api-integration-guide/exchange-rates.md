---
title: Exchange Rates
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-10-05_

The exchange rates API returns the current price of USDT in a fiat currency of your choice. Use it to show a fiat equivalent to your customers, or to work out how much USDT to request for a fiat-priced order. The hosted deposit flow (`POST /api/deposit-url` with `requestedCurrency` + `requestedAmount`) uses the same rate.

## GET /api/exchange-rates

```bash
curl "https://api.dcepay.io/api/exchange-rates?requestedCurrency=EUR" \
  -H "Authorization: YOUR_API_KEY"
```

### Query parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `requestedCurrency` | string | Yes | 3-letter fiat code (case-insensitive) — see [Supported currencies](#supported-currencies) |

### Response

```json
{
  "baseCurrency": "EUR",
  "rates": {
    "TRX": 0.92461305,
    "ETH": 0.92461305,
    "BNB": 0.92461305,
    "SOL": 0.92461305,
    "POL": 0.92461305
  },
  "timestamp": "2026-10-05T08:00:00.000Z"
}
```

| Field | Meaning |
|-------|---------|
| `baseCurrency` | The `requestedCurrency` you asked for, upper-cased |
| `rates` | Keyed by network. Each value is the price of **1 USDT** in `baseCurrency` (number, up to 8 decimals). Every network carries the same value: USDT is priced once, and deposits on every network are stablecoins |
| `timestamp` | When the response was produced |

Rates come from a live USDT spot price, converted to the requested fiat with the European Central Bank reference rates. They are refreshed at most every 60 seconds; repeated calls within that window return the same rate.

## Converting amounts

`rates[network]` is "fiat per 1 USDT", so:

```javascript
const res = await fetch('https://api.dcepay.io/api/exchange-rates?requestedCurrency=EUR', {
  headers: { Authorization: process.env.DCE_API_KEY },
});
const { rates } = await res.json();
const eurPerUsdt = rates.TRX;

const usdtToCharge = (orderEur / eurPerUsdt).toFixed(2); // fiat → USDT
const eurValue = (usdtAmount * eurPerUsdt).toFixed(2);   // USDT → fiat
```

Use decimal arithmetic (for example a decimal library) for anything you settle or reconcile on; the example above is for display.

## Supported currencies

`USD`, `EUR`, `GBP`, `AUD`, `CAD`, `CHF`, `CNY`, `HKD`, `IDR`, `INR`, `JPY`, `KRW`, `MYR`, `NZD`, `PHP`, `SGD`, `THB`, `TRY`, `VND`.

Rates are for USDT only. USDC is treated at the same rate: both are USD stablecoins.

## Errors

| HTTP | Body | When |
|------|------|------|
| 400 | `{ "error": "Invalid request data", "details": [...] }` | `requestedCurrency` missing or not 3 letters |
| 400 | `{ "error": "Unsupported currency: XXX", "code": "UNSUPPORTED_CURRENCY" }` | Not in the supported list |
| 401 | `{ "error": "Unauthorized" }` | Missing or invalid API key |
| 502 | `{ "error": "Rate provider unavailable", "code": "RATES_UPSTREAM" }` | No current rate could be obtained — retry shortly |

## Good practice

- Cache the response for up to 60 seconds on your side; the rate doesn't change more often than that.
- Show the customer the USDT amount you actually request, not only the fiat figure. The amount credited is the on-chain USDT received, and fiat conversion happens on your side.
- On a `502`, don't fall back to a hard-coded rate for pricing orders; retry or ask the customer to try again.

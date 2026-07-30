---
title: Balance Management
deprecated: false
hidden: false
metadata:
  robots: index
---
The balance API exposes **per-(currency, network)** available and pending balances for the authenticated user. Balances are segmented by chain: a USDT balance on TRX is separate from a USDT balance on ETH, and funds deposited on one chain can only be withdrawn on that same chain — there is no cross-chain fungibility.

> **Availability:** USDT on TRX (and its `TRX_SHASTA` testnet twin) is the pair enabled today. Other pairs (USDT/USDC on ETH, BNB, SOL) are coming soon / available on request.

## Endpoint

### `GET /api/balance`

**Query parameters**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `currency` | No | Token symbol: `USDT` or `USDC`. |
| `network` | No | Network symbol: `TRX`, `ETH`, `BNB`, `SOL` (testnets: `TRX_SHASTA`, `SEP`, `tBNB`, `SOL_DEVNET`). |

**Authentication:** send your API key in `Authorization` (raw key or `Bearer <key>`).

The response shape depends on the filters you pass:

- `currency` **and** `network` → the single balance row for that pair (legacy shape).
- otherwise → every balance row the account holds (optionally filtered by whichever of `currency`/`network` you did pass), each with the current network fee and maximum withdrawable amount after fees.

### Single pair (legacy shape)

```bash
curl -sS "${DCE_BASE_URL}/api/balance?currency=USDT&network=TRX" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

**Success response**

```json
{
  "currency": "USDT",
  "network": "TRX",
  "available": "1000.50",
  "pending": "0.00",
  "lastUpdatedAt": "2026-04-13T12:00:00.000Z"
}
```

If no balance row exists yet for the pair, the API returns zeros:

```json
{
  "currency": "USDT",
  "network": "TRX",
  "available": "0",
  "pending": "0"
}
```

### All balances (per-chain rows)

```bash
curl -sS "${DCE_BASE_URL}/api/balance" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

**Success response**

```json
{
  "balances": [
    {
      "currency": "USDT",
      "network": "TRX",
      "available": "1000.50",
      "pending": "25.00",
      "withdrawalEnabled": true,
      "networkFee": "1",
      "maxWithdrawable": "999.50",
      "lastUpdatedAt": "2026-04-13T12:00:00.000Z"
    }
  ]
}
```

Per-row fields:

| Field | Description |
|-------|-------------|
| `currency` | Token symbol (`USDT` or `USDC`). |
| `network` | Chain this row belongs to. |
| `available` | Spendable balance on this chain (decimal string). |
| `pending` | Funds credited but not yet spendable (decimal string) — see note below. |
| `withdrawalEnabled` | Whether withdrawals are currently enabled for this (currency, network) pair. |
| `networkFee` | Current per-withdrawal network fee for this pair, or `null` if withdrawals are disabled. |
| `maxWithdrawable` | `available` minus the current network fee (never negative); `0` when withdrawals are disabled. |
| `lastUpdatedAt` | Timestamp of the last balance change. |

## Integration notes

- Amounts are returned as **decimal strings** to preserve precision.
- **Pending balances:** wallet top-ups (direct deposits to your self-custody top-up address) are credited to `pending` first and move to `available` only once the sweep to the master wallet lands. Top-up funds may sit in `pending` briefly.
- Withdrawals draw only from the **same chain's** available balance — plan liquidity per network.
- Call this endpoint **after** deposits or withdrawals to confirm reserved vs available funds before creating payouts.
- Use the same `currency` and `network` values you use for withdrawals and ledger operations.

## Related guides

- [Withdrawals](https://docs.dcepay.io/docs/withdrawals)
- [Transactions](https://docs.dcepay.io/docs/transactions)
- [Quick Start](https://docs.dcepay.io/docs/quickstart)

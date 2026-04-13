---
title: Balance Management
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

The balance API exposes **per-currency** available and pending balances for the authenticated user.

## Endpoint

### `GET /api/balance`

**Query parameters**

| Parameter  | Required | Description                                              |
| ---------- | -------- | -------------------------------------------------------- |
| `currency` | Yes      | 3–4 character currency code (for example `USD`, `USDT`). |

**Authentication:** send your API key in `Authorization` (raw key or `Bearer <key>`).

**Example**

```bash
curl -sS "${DCE_BASE_URL}/api/balance?currency=USD" \
  -H "Authorization: ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

**Success response**

```json
{
  "currency": "USD",
  "available": "1000.50",
  "pending": "0.00",
  "lastUpdatedAt": "2026-04-13T12:00:00.000Z"
}
```

If no balance row exists yet, the API may return zeros for that currency (see handler behavior).

## Integration notes

- Amounts are returned as **decimal strings** to preserve precision.
- Call this endpoint **after** deposits or withdrawals to confirm reserved vs available funds before creating payouts.
- Use the same `currency` values you use for withdrawals and ledger operations.

## Related guides

- [Withdrawals](withdrawals.md)
- [Transactions](transactions.md)
- [Quick Start](quickstart.md)

<br />

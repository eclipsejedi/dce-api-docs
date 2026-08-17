---
title: Transactions
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-07-30_

The transactions API allows you to retrieve transaction history across all payment types. This guide covers transaction listing, filtering, pagination, and status tracking for deposits and withdrawals.

## Overview

The transactions API provides comprehensive access to all transaction data including:

- **Deposits** - Incoming payments and deposits
- **Withdrawals** - Outgoing payments and payouts
- **Fees** - Transaction fees and charges, folded into each row
- **Running balance** - Balance before/after each transaction
- **Status tracking** - Transaction status per row

Transactions carry the **currency and network** of the balance they touched. Balances are per-(currency, network) pair, so filter by both when you want an exact per-chain view.

## Retrieving Transactions

### GET /api/transactions

Retrieve a paginated list of transactions with optional filtering.

#### Request

```bash
curl -X GET "https://api.dcepay.io/api/transactions?page=1&limit=20&type=DEPOSIT&status=CONFIRMED&currency=USDT&network=TRX" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Items per page (default: 20) |
| `type` | string | No | Transaction type filter (`DEPOSIT`, `WITHDRAWAL`) |
| `status` | string | No | Status filter (`PENDING`, `CONFIRMED`, `FAILED`, `CANCELLED`) |
| `currency` | string | No | Token symbol filter (`USDT`, `USDC`) |
| `network` | string | No | Network filter (`TRX`, `ETH`, `BNB`, `SOL`; testnets `TRX_SHASTA`, `SEP`, `tBNB`, `SOL_DEVNET`) |

Unrecognized filter values are ignored (the filter is simply not applied) rather than rejected.

#### Response

```json
{
  "transactions": [
    {
      "id": "txn_xxxxxxxxxxxxxxxx",
      "txnHash": "d0e1f2a3b4c5...",
      "block": null,
      "age": "5 mins ago",
      "from": "TSenderAddress...",
      "to": "TX7k1zNq7PqR3sT9uV2wX4yZ6aB8cD1eF3",
      "amount": "99.495",
      "amountRaw": "99.495",
      "fee": "1.005",
      "feeRaw": "1.005",
      "token": "USDT",
      "type": "DEPOSIT",
      "status": "CONFIRMED",
      "balanceBefore": "900.00",
      "balanceAfter": "999.495",
      "balanceBeforeRaw": "900",
      "balanceAfterRaw": "999.495",
      "createdAt": "2026-07-29T10:30:00.000Z",
      "createdAtGmt8": "2026-07-29T18:30:00.000Z",
      "description": "Customer payment",
      "category": "Customer Payment",
      "tags": ["customer-payment", "crypto"],
      "metadata": {
        "txHash": "d0e1f2a3b4c5...",
        "webhookPayload": { "referenceId": "order_123" }
      }
    }
  ],
  "balance": {
    "currency": "USDT",
    "network": "TRX",
    "available": "999.495"
  },
  "pagination": {
    "total": 150,
    "page": 1,
    "limit": 20,
    "pages": 8
  }
}
```

#### Transaction Fields

| Field | Description |
|-------|-------------|
| `id` | Transaction id |
| `txnHash` | On-chain transaction hash when known (falls back to L2 hash or reference id, empty string if none) |
| `block` | Always `null` (block number is not stored) |
| `age` | Human-readable age, e.g. `5 mins ago` |
| `from` / `to` | Source and destination addresses where known |
| `amount` | **Signed net balance change** as a decimal string: positive for deposits (`amount - fee`), negative for withdrawals (`-(amount + fee)`) |
| `fee` | Total fee attributed to the transaction (deposit fee, or withdrawal commission/fee, plus any internal fee recorded in metadata) |
| `token` | Currency of the transaction (`USDT`, `USDC`) |
| `type` | `DEPOSIT` or `WITHDRAWAL` |
| `status` | `PENDING`, `CONFIRMED`, `FAILED`, or `CANCELLED` |
| `balanceBefore` / `balanceAfter` | Running balance around this transaction (see note below) |
| `amountRaw`, `feeRaw`, `balanceBeforeRaw`, `balanceAfterRaw` | Same values as their formatted counterparts, serialized as decimal strings |
| `createdAt` / `createdAtGmt8` | Creation time in UTC and shifted to GMT+8 |
| `description`, `category`, `tags`, `metadata` | Descriptive fields; `metadata` may include `txHash`, `fromAddress`, `toAddress`, and the webhook payload with your `referenceId` |

#### The `balance` object

The response includes the current available balance the running-balance column was computed against:

- With **both** `currency` and `network` supplied, it is the balance row for that pair — and `balanceBefore`/`balanceAfter` are exact for that chain.
- Otherwise, the first matching balance row the account holds is used, and the running-balance figures are approximate across chains. **Supply both filters when you need exact running balances.**

#### JavaScript Example

```javascript
async function getTransactions(filters = {}) {
  const params = new URLSearchParams();
  
  if (filters.page) params.append('page', filters.page);
  if (filters.limit) params.append('limit', filters.limit);
  if (filters.type) params.append('type', filters.type);
  if (filters.status) params.append('status', filters.status);
  if (filters.currency) params.append('currency', filters.currency);
  if (filters.network) params.append('network', filters.network);

  const response = await fetch(`https://api.dcepay.io/api/transactions?${params}`, {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
      'Content-Type': 'application/json'
    }
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch transactions: ${response.statusText}`);
  }

  return response.json();
}

// Usage examples
const allTransactions = await getTransactions();
const deposits = await getTransactions({ type: 'DEPOSIT', status: 'CONFIRMED' });
const trxUsdt = await getTransactions({ currency: 'USDT', network: 'TRX' });
```

## Transaction Types

### Deposit Transactions

Deposit transactions represent incoming payments from customers or top-ups of your own balance. The `amount` is the net credit (deposit amount minus fee):

```json
{
  "id": "txn_xxxxxxxxxxxxxxxx",
  "type": "DEPOSIT",
  "status": "CONFIRMED",
  "amount": "99.495",
  "fee": "1.005",
  "token": "USDT",
  "to": "TX7k1zNq7PqR3sT9uV2wX4yZ6aB8cD1eF3"
}
```

Wallet top-ups (direct deposits) are **sweep-gated**: they stay `PENDING` — credited to your pending balance — until the sweep to the master wallet lands, then move to `CONFIRMED`. See [Deposits](https://docs.dcepay.io/docs/deposits).

### Withdrawal Transactions

Withdrawal transactions represent outgoing payments to external addresses. The `amount` is the total debit, negated (withdrawal amount plus fees):

```json
{
  "id": "txn_xxxxxxxxxxxxxxxx",
  "type": "WITHDRAWAL",
  "status": "PENDING",
  "amount": "-51.00",
  "fee": "1.00",
  "token": "USDT",
  "to": "TDestinationAddress..."
}
```

## Transaction Status

### Status Values

| Status | Description |
|--------|-------------|
| `PENDING` | Transaction is being processed (for top-ups: awaiting the master-wallet sweep) |
| `CONFIRMED` | Transaction has been confirmed and completed |
| `FAILED` | Transaction failed and cannot be completed |
| `CANCELLED` | Transaction was cancelled by user or system |

### Status Transitions

```
PENDING → CONFIRMED (successful completion)
PENDING → FAILED (processing failure)
PENDING → CANCELLED (user cancellation)
```

## Filtering and Search

### Type Filtering

Filter transactions by type:

```bash
# Get only deposits
GET /api/transactions?type=DEPOSIT

# Get only withdrawals
GET /api/transactions?type=WITHDRAWAL
```

### Status Filtering

Filter transactions by status:

```bash
# Get pending transactions
GET /api/transactions?status=PENDING

# Get confirmed transactions
GET /api/transactions?status=CONFIRMED

# Get failed transactions
GET /api/transactions?status=FAILED
```

### Currency and Network Filtering

Filter transactions by (currency, network) pair — recommended for exact running balances:

```bash
# Get USDT-on-TRX transactions
GET /api/transactions?currency=USDT&network=TRX

# Get everything on a chain
GET /api/transactions?network=TRX
```

## Pagination

The transactions API supports pagination with the following parameters:

### Pagination Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | number | 1 | Page number (1-based) |
| `limit` | number | 20 | Items per page |

### Pagination Response

```json
{
  "transactions": [...],
  "balance": { "currency": "USDT", "network": "TRX", "available": "999.495" },
  "pagination": {
    "total": 150,
    "page": 1,
    "limit": 20,
    "pages": 8
  }
}
```

### Pagination Example

```javascript
async function getAllTransactions() {
  let allTransactions = [];
  let page = 1;
  let hasMore = true;

  while (hasMore) {
    const response = await getTransactions({ page, limit: 100 });
    allTransactions = allTransactions.concat(response.transactions);
    
    hasMore = page < response.pagination.pages;
    page++;
  }

  return allTransactions;
}
```

## Transaction Metadata

Transactions can include metadata for additional context. Deposit and withdrawal callbacks store the webhook payload here, including your merchant `referenceId`:

```json
{
  "id": "txn_xxxxxxxxxxxxxxxx",
  "metadata": {
    "txHash": "d0e1f2a3b4c5...",
    "fromAddress": "TSenderAddress...",
    "toAddress": "TX7k1zNq7PqR3sT9uV2wX4yZ6aB8cD1eF3",
    "webhookPayload": {
      "referenceId": "order_123"
    }
  }
}
```

## Error Handling

### Common Errors

| Status Code | Error | Description |
|-------------|-------|-------------|
| 400 | `Invalid request data` | Invalid query parameters (see `details`) |
| 401 | `Unauthorized` | Missing or invalid API key |
| 500 | `Internal server error` | Server-side error |

### Error Response Format

```json
{
  "error": "Invalid request data",
  "details": [
    {
      "path": ["page"],
      "message": "Expected string, received null"
    }
  ]
}
```

## Best Practices

### 1. Efficient Pagination

- Use appropriate `limit` values (20-50 for most use cases)
- Implement caching for frequently accessed data
- Use `type`, `status`, `currency`, and `network` filters to reduce result sets

### 2. Real-time Updates

- Use webhooks for real-time transaction status updates (delivery is at-least-once — make your consumer idempotent)
- Poll the API periodically for status changes
- Implement retry logic for failed requests

### 3. Data Management

- Store transaction IDs for reconciliation
- Filter by both `currency` and `network` when reconciling per-chain balances
- Implement proper error handling
- Log all transaction operations for audit trails

### 4. Performance Optimization

```javascript
// Efficient per-chain transaction fetching
async function getChainTransactions(currency = 'USDT', network = 'TRX') {
  return getTransactions({
    currency,
    network,
    limit: 50
  });
}

// Batch processing for large datasets
async function processTransactions(transactions) {
  const batchSize = 10;
  const results = [];
  
  for (let i = 0; i < transactions.length; i += batchSize) {
    const batch = transactions.slice(i, i + batchSize);
    const batchResults = await Promise.all(
      batch.map(tx => processTransaction(tx))
    );
    results.push(...batchResults);
  }
  
  return results;
}
```

## Integration Examples

### E-commerce Integration

```javascript
class TransactionManager {
  async getRevenueReport() {
    const response = await getTransactions({
      type: 'DEPOSIT',
      status: 'CONFIRMED',
      currency: 'USDT',
      network: 'TRX'
    });

    // `amount` is already net of fees and signed
    return response.transactions.reduce((total, tx) => {
      return total + parseFloat(tx.amount);
    }, 0);
  }

  async findByReferenceId(referenceId) {
    const response = await getTransactions({ limit: 100 });
    return response.transactions.filter(
      (tx) => tx.metadata?.webhookPayload?.referenceId === referenceId
    );
  }
}
```

### Accounting Integration

```javascript
class AccountingIntegration {
  async exportTransactions() {
    const response = await getTransactions({ limit: 100 });

    return response.transactions.map(tx => ({
      date: tx.createdAt,
      description: tx.description,
      amount: tx.amount,          // signed net change
      fee: tx.fee,
      currency: tx.token,
      type: tx.type,
      status: tx.status,
      txnHash: tx.txnHash,
      balanceAfter: tx.balanceAfter,
      reference: tx.id
    }));
  }
}
```

---

*For more information about specific transaction types, see the [Deposits](https://docs.dcepay.io/docs/deposits) and [Withdrawals](https://docs.dcepay.io/docs/withdrawals) documentation.* 

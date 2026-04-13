---
title: Transactions
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

The transactions API allows you to retrieve and manage transaction history across all payment types. This guide covers transaction listing, filtering, pagination, and status tracking for deposits, withdrawals, and other transaction types.

## Overview

The transactions API provides comprehensive access to all transaction data including:

- **Deposits** - Incoming payments and deposits
- **Withdrawals** - Outgoing payments and payouts
- **Settlements** - Settlement transactions
- **Fees** - Transaction fees and charges
- **Status tracking** - Real-time transaction status updates

## Retrieving Transactions

### GET /api/transactions

Retrieve a paginated list of transactions with optional filtering.

#### Request

```bash
curl -X GET "https://api.dcepay.io/api/transactions?page=1&limit=10&type=DEPOSIT&status=CONFIRMED&currency=USD" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Query Parameters

| Parameter   | Type   | Required | Description                                               |
| ----------- | ------ | -------- | --------------------------------------------------------- |
| `page`      | number | No       | Page number (default: 1)                                  |
| `limit`     | number | No       | Items per page (default: 10, max: 100)                    |
| `type`      | string | No       | Transaction type filter (DEPOSIT, WITHDRAWAL, SETTLEMENT) |
| `status`    | string | No       | Status filter (PENDING, CONFIRMED, FAILED)                |
| `currency`  | string | No       | Currency filter (USD, EUR, BTC, etc.)                     |
| `startDate` | string | No       | Start date filter (ISO 8601 format)                       |
| `endDate`   | string | No       | End date filter (ISO 8601 format)                         |

#### Response

```json
{
  "transactions": [
    {
      "id": "txn_xxxxxxxxxxxxxxxx",
      "type": "DEPOSIT",
      "status": "CONFIRMED",
      "amount": "100.00",
      "currency": "USD",
      "description": "Customer payment",
      "createdAt": "2024-12-19T10:30:00Z",
      "updatedAt": "2024-12-19T10:35:00Z",
      "metadata": {
        "orderId": "order_123",
        "customerEmail": "customer@example.com"
      },
      "deposit": {
        "source": "CRYPTO_WALLET",
        "referenceId": "ref_123",
        "clearedAt": "2024-12-19T10:35:00Z",
        "fee": "1.00"
      },
      "category": {
        "name": "Customer Payment",
        "description": "Incoming customer payments"
      },
      "tags": [
        { "name": "customer-payment" },
        { "name": "crypto" }
      ]
    }
  ],
  "pagination": {
    "total": 150,
    "page": 1,
    "limit": 10,
    "pages": 15
  }
}
```

#### JavaScript Example

```javascript
async function getTransactions(filters = {}) {
  const params = new URLSearchParams();
  
  if (filters.page) params.append('page', filters.page);
  if (filters.limit) params.append('limit', filters.limit);
  if (filters.type) params.append('type', filters.type);
  if (filters.status) params.append('status', filters.status);
  if (filters.currency) params.append('currency', filters.currency);
  if (filters.startDate) params.append('startDate', filters.startDate);
  if (filters.endDate) params.append('endDate', filters.endDate);

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
const recentTransactions = await getTransactions({ 
  startDate: '2024-12-01T00:00:00Z',
  endDate: '2024-12-31T23:59:59Z'
});
```

## Transaction Types

### Deposit Transactions

Deposit transactions represent incoming payments from customers or external sources.

```json
{
  "id": "txn_xxxxxxxxxxxxxxxx",
  "type": "DEPOSIT",
  "status": "CONFIRMED",
  "amount": "100.00",
  "currency": "USD",
  "deposit": {
    "source": "CRYPTO_WALLET",
    "referenceId": "ref_123",
    "clearedAt": "2024-12-19T10:35:00Z",
    "fee": "1.00"
  }
}
```

### Withdrawal Transactions

Withdrawal transactions represent outgoing payments to customers or external addresses.

```json
{
  "id": "txn_xxxxxxxxxxxxxxxx",
  "type": "WITHDRAWAL",
  "status": "PENDING",
  "amount": "50.00",
  "currency": "USD",
  "withdrawal": {
    "destination": "0x1234567890123456789012345678901234567890",
    "referenceId": "ref_456",
    "processedAt": null,
    "fee": "0.50",
    "reason": null
  }
}
```

### Settlement Transactions

Settlement transactions represent merchant settlement payouts.

```json
{
  "id": "txn_xxxxxxxxxxxxxxxx",
  "type": "SETTLEMENT",
  "status": "CONFIRMED",
  "amount": "1000.00",
  "currency": "USD",
  "settlement": {
    "destinationAddress": "0x1234567890123456789012345678901234567890",
    "destinationNetwork": "ETH",
    "processedAt": "2024-12-19T10:35:00Z",
    "fee": "25.00"
  }
}
```

## Transaction Status

### Status Values

| Status      | Description                                  |
| ----------- | -------------------------------------------- |
| `PENDING`   | Transaction is being processed               |
| `CONFIRMED` | Transaction has been confirmed and completed |
| `FAILED`    | Transaction failed and cannot be completed   |
| `CANCELLED` | Transaction was cancelled by user or system  |

### Status Transitions

```text
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

# Get only settlements
GET /api/transactions?type=SETTLEMENT
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

### Date Range Filtering

Filter transactions by date range:

```bash
# Get transactions from specific date range
GET /api/transactions?startDate=2024-12-01T00:00:00Z&endDate=2024-12-31T23:59:59Z
```

### Currency Filtering

Filter transactions by currency:

```bash
# Get USD transactions
GET /api/transactions?currency=USD

# Get BTC transactions
GET /api/transactions?currency=BTC
```

## Pagination

The transactions API supports pagination with the following parameters:

### Pagination Parameters

| Parameter | Type   | Default | Description               |
| --------- | ------ | ------- | ------------------------- |
| `page`    | number | 1       | Page number (1-based)     |
| `limit`   | number | 10      | Items per page (max: 100) |

### Pagination Response

```json
{
  "transactions": [...],
  "pagination": {
    "total": 150,
    "page": 1,
    "limit": 10,
    "pages": 15
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

Transactions can include metadata for additional context:

```json
{
  "id": "txn_xxxxxxxxxxxxxxxx",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com",
    "paymentMethod": "crypto",
    "source": "webhook",
    "ipAddress": "192.168.1.1"
  }
}
```

## Error Handling

### Common Errors

| Status Code | Error                      | Description                     |
| ----------- | -------------------------- | ------------------------------- |
| 400         | `Invalid request data`     | Invalid query parameters        |
| 401         | `Unauthorized`             | Missing or invalid API key      |
| 403         | `Insufficient permissions` | User lacks required permissions |
| 500         | `Internal server error`    | Server-side error               |

### Error Response Format

```json
{
  "error": "Error message",
  "details": [
    {
      "field": "page",
      "message": "Page number must be positive"
    }
  ]
}
```

## Best Practices

### 1. Efficient Pagination

- Use appropriate `limit` values (10-50 for most use cases)
- Implement caching for frequently accessed data
- Use date filters to reduce result sets

### 2. Real-time Updates

- Use webhooks for real-time transaction status updates
- Poll the API periodically for status changes
- Implement retry logic for failed requests

### 3. Data Management

- Store transaction IDs for reconciliation
- Implement proper error handling
- Log all transaction operations for audit trails

### 4. Performance Optimization

```javascript
// Efficient transaction fetching
async function getRecentTransactions(days = 7) {
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - days);
  
  return getTransactions({
    startDate: startDate.toISOString(),
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
  constructor(apiKey) {
    this.apiKey = apiKey;
  }

  async getOrderTransactions(orderId) {
    return getTransactions({
      metadata: { orderId }
    });
  }

  async getCustomerTransactions(customerEmail) {
    return getTransactions({
      metadata: { customerEmail }
    });
  }

  async getRevenueReport(startDate, endDate) {
    const transactions = await getTransactions({
      type: 'DEPOSIT',
      status: 'CONFIRMED',
      startDate,
      endDate
    });

    return transactions.reduce((total, tx) => {
      return total + parseFloat(tx.amount);
    }, 0);
  }
}
```

### Accounting Integration

```javascript
class AccountingIntegration {
  async exportTransactions(startDate, endDate) {
    const transactions = await getTransactions({
      startDate,
      endDate,
      limit: 1000
    });

    return transactions.map(tx => ({
      date: tx.createdAt,
      description: tx.description,
      amount: tx.amount,
      currency: tx.currency,
      type: tx.type,
      status: tx.status,
      reference: tx.id
    }));
  }
}
```

***

_For more information about specific transaction types, see the [Deposits](deposits.md) and [Withdrawals](withdrawals.md) documentation._

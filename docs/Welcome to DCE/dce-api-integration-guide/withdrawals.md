---
title: Withdrawals
deprecated: false
hidden: false
metadata:
  robots: index
---
The withdrawals API allows you to process payouts to customers and external addresses. This guide covers withdrawal creation, status tracking, balance validation, and best practices for managing withdrawal flows.

## Overview

The withdrawal flow consists of these steps:

1. **Validate balance** - Ensure sufficient funds are available
2. **Create withdrawal** - Submit withdrawal request with destination address
3. **Monitor status** - Track withdrawal processing via webhooks and API calls
4. **Handle confirmations** - Process completed withdrawals in your system

## Creating Withdrawals

### POST /api/withdrawals

Create a new withdrawal request to send funds to a destination address.

#### Request

```bash
curl -X POST "https://api.dcepay.io/api/withdrawals" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "token": "BTC",
    "amount": 0.001,
    "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
    "metadata": {
      "orderId": "order_123",
      "customerEmail": "customer@example.com"
    }
  }'
```

#### Request Parameters

| Parameter  | Type   | Required | Description                                      |
| ---------- | ------ | -------- | ------------------------------------------------ |
| `token`    | string | Yes      | Cryptocurrency symbol (BTC, ETH, USDT, etc.)     |
| `amount`   | number | Yes      | Amount to withdraw                               |
| `address`  | string | Yes      | Destination address for the withdrawal           |
| `metadata` | object | No       | Additional data to associate with the withdrawal |

#### Response

```json
{
  "id": "wth_xxxxxxxxxxxxxxxx",
  "token": "BTC",
  "amount": 0.001,
  "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
  "status": "pending",
  "fee": 0.00001,
  "netAmount": 0.00099,
  "createdAt": "2024-12-19T10:30:00Z",
  "estimatedCompletion": "2024-12-19T11:30:00Z",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  }
}
```

#### JavaScript Example

```javascript
async function createWithdrawal(token, amount, address, metadata = {}) {
  const response = await fetch('https://api.dcepay.io/api/withdrawals', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      token,
      amount,
      address,
      metadata
    })
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(`Withdrawal creation failed: ${error.message}`);
  }

  return response.json();
}

// Usage
const withdrawal = await createWithdrawal('BTC', 0.001, 'bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh', {
  orderId: 'order_123',
  customerEmail: 'customer@example.com'
});

console.log('Withdrawal ID:', withdrawal.id);
console.log('Net amount (after fees):', withdrawal.netAmount);
```

## Listing Withdrawals

### GET /api/withdrawals

Retrieve a list of your withdrawals with optional filtering.

#### Request

```bash
curl -X GET "https://api.dcepay.io/api/withdrawals?status=CONFIRMED&page=1&limit=10" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Query Parameters

| Parameter   | Type   | Description                                               |
| ----------- | ------ | --------------------------------------------------------- |
| `status`    | string | Filter by status (pending, processing, confirmed, failed) |
| `token`     | string | Filter by cryptocurrency                                  |
| `limit`     | number | Number of results to return (default: 20, max: 100)       |
| `offset`    | number | Number of results to skip for pagination                  |
| `startDate` | string | Filter withdrawals created after this date (ISO 8601)     |
| `endDate`   | string | Filter withdrawals created before this date (ISO 8601)    |

#### Response

```json
{
  "withdrawals": [
    {
      "id": "wth_xxxxxxxxxxxxxxxx",
      "token": "BTC",
      "amount": 0.001,
      "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
      "status": "confirmed",
      "fee": 0.00001,
      "netAmount": 0.00099,
      "txHash": "0x1234567890abcdef...",
      "confirmedAt": "2024-12-19T11:45:00Z",
      "createdAt": "2024-12-19T10:30:00Z",
      "metadata": {
        "orderId": "order_123",
        "customerEmail": "customer@example.com"
      }
    }
  ],
  "pagination": {
    "total": 75,
    "limit": 10,
    "offset": 0,
    "hasMore": true
  }
}
```

## Balance Validation

Before creating a withdrawal, always check your available balance:

### GET /api/balance

```bash
curl -X GET "https://api.dcepay.io/api/balance" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Response

```json
{
  "balances": [
    {
      "token": "BTC",
      "available": 0.005,
      "pending": 0.001,
      "total": 0.006
    },
    {
      "token": "ETH",
      "available": 0.1,
      "pending": 0.02,
      "total": 0.12
    }
  ]
}
```

#### Balance Validation Example

```javascript
async function validateWithdrawalBalance(token, amount) {
  const response = await fetch('https://api.dcepay.io/api/balance', {
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`
    }
  });

  const { balances } = await response.json();
  const balance = balances.find(b => b.token === token);

  if (!balance) {
    throw new Error(`No balance found for ${token}`);
  }

  if (balance.available < amount) {
    throw new Error(`Insufficient balance. Available: ${balance.available} ${token}, Requested: ${amount} ${token}`);
  }

  return true;
}

// Usage
try {
  await validateWithdrawalBalance('BTC', 0.001);
  const withdrawal = await createWithdrawal('BTC', 0.001, 'address...');
  console.log('Withdrawal created:', withdrawal.id);
} catch (error) {
  console.error('Withdrawal failed:', error.message);
}
```

## Withdrawal Status Tracking

### Status Values

| Status       | Description                                                   |
| ------------ | ------------------------------------------------------------- |
| `pending`    | Withdrawal created, waiting for processing                    |
| `processing` | Withdrawal is being processed by the network                  |
| `confirmed`  | Withdrawal completed and confirmed on blockchain              |
| `failed`     | Withdrawal failed (insufficient funds, invalid address, etc.) |
| `cancelled`  | Withdrawal was cancelled                                      |

### Status Transitions

```text
pending → processing (when withdrawal starts)
processing → confirmed (when blockchain confirms)
pending → failed (if validation fails)
processing → failed (if network issues occur)
```

## Fee Structure

Withdrawals incur fees that are deducted from the withdrawal amount. The fee structure consists of a base fee (1 USDT) plus an optional markup rate.

### Fee Calculation

The withdrawal fee is calculated as follows:

- **Base Fee**: 1 USDT (fixed)
- **Markup Rate**: Optional percentage markup on the withdrawal amount
- **Total Fee**: Base Fee + (Withdrawal Amount × Markup Rate)

```javascript
// Example fee calculations
const baseFee = 1; // USDT
const markupRate = 0.05; // 5%

// For a 100 USDT withdrawal with 5% markup:
const withdrawalAmount = 100;
const markupAmount = withdrawalAmount * markupRate; // 5 USDT
const totalFee = baseFee + markupAmount; // 6 USDT
const netAmount = withdrawalAmount - totalFee; // 94 USDT

// For a 100 USDT withdrawal with no markup:
const totalFeeNoMarkup = baseFee; // 1 USDT
const netAmountNoMarkup = withdrawalAmount - totalFeeNoMarkup; // 99 USDT
```

### System Configuration

Withdrawal fees are configured internally by system administrators and cannot be modified via the API. The fee structure is determined by:

- **Base Fee**: Configurable system setting (default: 1 USDT)
- **Markup Rate**: Configurable system setting (default: 0% - no markup)
- **Fee Status**: Can be enabled/disabled by administrators

#### Admin API for Fee Management

Administrators can manage withdrawal fee settings using the admin API:

**Get Current Settings:**

```bash
GET /api/admin/withdrawal-fees
Authorization: Bearer <admin-api-key>
```

**Update Settings:**

```bash
POST /api/admin/withdrawal-fees
Authorization: Bearer <admin-api-key>
Content-Type: application/json

{
  "baseFee": "1.5",
  "markupRate": "0.02",
  "enabled": true
}
```

**Test Fee Calculation:**

```bash
PUT /api/admin/withdrawal-fees
Authorization: Bearer <admin-api-key>
Content-Type: application/json

{
  "amount": "100",
  "currency": "USDT"
}
```

### Fee Information in Response

The withdrawal API response includes detailed fee information:

```json
{
  "success": true,
  "withdrawalId": "wth_xxxxxxxxxxxxxxxx",
  "amount": "100.00",
  "currency": "USDT",
  "feeInfo": {
    "grossAmount": "100.00",
    "netAmount": "94.00",
    "totalFees": "6.00",
    "feeBreakdown": {
      "baseFee": "1.00",
      "markupRate": "5.00",
      "markupAmount": "5.00",
      "totalFee": "6.00"
    },
    "feePercentage": "6.00"
  }
}
```

## Webhook Notifications

You'll receive webhook notifications when withdrawal status changes:

### Withdrawal Confirmed Webhook

```json
{
  "event": "withdrawal.confirmed",
  "data": {
    "id": "wth_xxxxxxxxxxxxxxxx",
    "token": "USDT",
    "amount": "100.00",
    "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
    "status": "confirmed",
    "fee": "6.00",
    "netAmount": "94.00",
    "commission": "6.00",
    "commissionRate": "0.05",
    "txHash": "0x1234567890abcdef...",
    "confirmedAt": "2024-12-19T11:45:00Z",
    "metadata": {
      "orderId": "order_123",
      "customerEmail": "customer@example.com",
      "feeCalculation": {
        "baseFee": "1.00",
        "markupRate": "0.05",
        "totalFee": "6.00",
        "netAmount": "94.00"
      }
    }
  },
  "timestamp": "2024-12-19T11:45:00Z"
}
```

### Withdrawal Failed Webhook

```json
{
  "event": "withdrawal.failed",
  "data": {
    "id": "wth_xxxxxxxxxxxxxxxx",
    "token": "BTC",
    "amount": 0.001,
    "address": "bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh",
    "status": "failed",
    "failedAt": "2024-12-19T11:45:00Z",
    "reason": "insufficient_funds",
    "metadata": {
      "orderId": "order_123",
      "customerEmail": "customer@example.com"
    }
  },
  "timestamp": "2024-12-19T11:45:00Z"
}
```

## Error Handling

### Common Errors

| Status Code | Error                     | Description                           |
| ----------- | ------------------------- | ------------------------------------- |
| 400         | `INVALID_ADDRESS`         | Invalid destination address           |
| 400         | `INSUFFICIENT_BALANCE`    | Not enough funds for withdrawal       |
| 400         | `INVALID_AMOUNT`          | Amount is too small or invalid        |
| 400         | `ADDRESS_NOT_WHITELISTED` | Address not in whitelist (if enabled) |
| 429         | `RATE_LIMITED`            | Too many withdrawal requests          |

### Example Error Response

```json
{
  "error": "INSUFFICIENT_BALANCE",
  "message": "Available balance: 0.0005 BTC, Requested: 0.001 BTC",
  "statusCode": 400
}
```

## Best Practices

### 1. Address Validation

Always validate addresses before creating withdrawals:

```javascript
function validateAddress(token, address) {
  const patterns = {
    'BTC': /^[13][a-km-zA-HJ-NP-Z1-9]{25,34}$|^bc1[a-z0-9]{39,59}$/,
    'ETH': /^0x[a-fA-F0-9]{40}$/,
    'USDT': /^0x[a-fA-F0-9]{40}$/
  };

  const pattern = patterns[token];
  if (!pattern) {
    throw new Error(`Unsupported token: ${token}`);
  }

  if (!pattern.test(address)) {
    throw new Error(`Invalid ${token} address: ${address}`);
  }

  return true;
}
```

### 2. Balance Checking

Always check balance before withdrawal:

```javascript
async function safeWithdrawal(token, amount, address, metadata = {}) {
  // 1. Validate address
  validateAddress(token, address);

  // 2. Check balance
  await validateWithdrawalBalance(token, amount);

  // 3. Create withdrawal
  const withdrawal = await createWithdrawal(token, amount, address, metadata);

  // 4. Log withdrawal
  console.log(`Withdrawal created: ${withdrawal.id}`);
  console.log(`Amount: ${withdrawal.amount} ${token}`);
  console.log(`Fee: ${withdrawal.fee} ${token}`);
  console.log(`Net amount: ${withdrawal.netAmount} ${token}`);

  return withdrawal;
}
```

### 3. Idempotency

Use idempotency keys to prevent duplicate withdrawals:

```javascript
async function createWithdrawalWithIdempotency(token, amount, address, metadata = {}) {
  const idempotencyKey = generateIdempotencyKey(token, amount, address, metadata);

  const response = await fetch('https://api.dcepay.io/api/withdrawals', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
      'Content-Type': 'application/json',
      'Idempotency-Key': idempotencyKey
    },
    body: JSON.stringify({
      token,
      amount,
      address,
      metadata
    })
  });

  return response.json();
}

function generateIdempotencyKey(token, amount, address, metadata) {
  const data = JSON.stringify({ token, amount, address, metadata });
  return require('crypto').createHash('sha256').update(data).digest('hex');
}
```

### 4. Webhook Handling

Handle withdrawal webhooks properly:

```javascript
app.post('/webhooks', async (req, res) => {
  const { event, data } = req.body;

  switch (event) {
    case 'withdrawal.confirmed':
      await handleWithdrawalConfirmed(data);
      break;
    case 'withdrawal.failed':
      await handleWithdrawalFailed(data);
      break;
    default:
      console.log(`Unhandled webhook event: ${event}`);
  }

  res.status(200).json({ received: true });
});

async function handleWithdrawalConfirmed(data) {
  const { id, amount, address, txHash, metadata } = data;
  
  // Update your database
  await updateWithdrawalStatus(id, 'confirmed', txHash);
  
  // Notify customer
  await notifyCustomer(metadata.customerEmail, {
    type: 'withdrawal_confirmed',
    amount,
    address,
    txHash
  });
}

async function handleWithdrawalFailed(data) {
  const { id, reason, metadata } = data;
  
  // Update your database
  await updateWithdrawalStatus(id, 'failed', null, reason);
  
  // Notify customer
  await notifyCustomer(metadata.customerEmail, {
    type: 'withdrawal_failed',
    reason
  });
}
```

## Integration Example

Here's a complete withdrawal service implementation:

```javascript
class WithdrawalService {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.baseUrl = 'https://api.dcepay.io/api';
  }

  async createWithdrawal(token, amount, address, metadata = {}) {
    // Validate inputs
    this.validateAddress(token, address);
    await this.validateBalance(token, amount);

    const response = await fetch(`${this.baseUrl}/withdrawals`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
        'Idempotency-Key': this.generateIdempotencyKey(token, amount, address, metadata)
      },
      body: JSON.stringify({
        token,
        amount,
        address,
        metadata
      })
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(`Withdrawal creation failed: ${error.message}`);
    }

    return response.json();
  }

  async getWithdrawals(filters = {}) {
    const params = new URLSearchParams(filters);
    const response = await fetch(`${this.baseUrl}/withdrawals?${params}`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });

    return response.json();
  }

  async getWithdrawalById(id) {
    const response = await fetch(`${this.baseUrl}/withdrawals/${id}`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });

    return response.json();
  }

  async validateBalance(token, amount) {
    const response = await fetch(`${this.baseUrl}/balance`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });

    const { balances } = await response.json();
    const balance = balances.find(b => b.token === token);

    if (!balance || balance.available < amount) {
      throw new Error(`Insufficient balance for ${amount} ${token}`);
    }

    return true;
  }

  validateAddress(token, address) {
    const patterns = {
      'BTC': /^[13][a-km-zA-HJ-NP-Z1-9]{25,34}$|^bc1[a-z0-9]{39,59}$/,
      'ETH': /^0x[a-fA-F0-9]{40}$/,
      'USDT': /^0x[a-fA-F0-9]{40}$/
    };

    const pattern = patterns[token];
    if (!pattern || !pattern.test(address)) {
      throw new Error(`Invalid ${token} address: ${address}`);
    }
  }

  generateIdempotencyKey(token, amount, address, metadata) {
    const data = JSON.stringify({ token, amount, address, metadata });
    return require('crypto').createHash('sha256').update(data).digest('hex');
  }
}

// Usage
const withdrawalService = new WithdrawalService(process.env.DCE_API_KEY);

try {
  const withdrawal = await withdrawalService.createWithdrawal('BTC', 0.001, 'bc1qxy2kgdygjrsqtzq2n0yrf2493p83kkfjhx0wlh', {
    orderId: 'order_123',
    customerEmail: 'customer@example.com'
  });

  console.log('Withdrawal created:', withdrawal.id);
  console.log('Net amount:', withdrawal.netAmount);
} catch (error) {
  console.error('Withdrawal failed:', error.message);
}
```

## Next Steps

Now that you understand withdrawals, explore:

1. [Balance Management](balance-management.md) - Monitor your account balance
2. [Settlements](settlements.md) - Request bulk settlements
3. [Webhooks](webhooks.md) - Handle withdrawal notifications

<br />

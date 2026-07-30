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

1. **Validate balance** - Ensure sufficient funds are available on the chain you are withdrawing from
2. **Create withdrawal** - Submit withdrawal request with destination address and network
3. **Monitor status** - Track withdrawal processing via webhooks and API calls
4. **Handle confirmations** - Process completed withdrawals in your system

### Supported currencies and networks

Balances are segmented per `(currency, network)` pair. There is **no cross-chain fungibility**: a withdrawal draws only from the balance on the same network.

| Currency | Network | Status |
|----------|---------|--------|
| `USDT` | `TRX` (Tron) | **Available** |
| `USDT` | `TRX_SHASTA` (Tron testnet) | Available (staging/testing) |
| `USDT` / `USDC` | `ETH`, `BNB`, `SOL` | Coming soon / on request |

Requests referencing a disabled `(currency, network)` pair are rejected with `400` and an error such as `"Withdrawals of USDT on ETH are not supported"`.

## Creating Withdrawals

### POST /api/withdrawals

Create a new withdrawal request to send funds to a destination address.

#### Request

```bash
curl -X POST "https://api.dcepay.io/api/withdrawals" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "100.50",
    "currency": "USDT",
    "network": "TRX",
    "destination": "TXYZa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5",
    "description": "Payout for order 123",
    "referenceId": "order_123"
  }'
```

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `amount` | string | Yes | Amount to withdraw as a positive decimal string (up to 18 decimal places), e.g. `"100.50"` |
| `currency` | string | Yes | Token symbol: `USDT` or `USDC` |
| `network` | string | Yes | Network symbol: `TRX`, `ETH`, `BNB`, `SOL` (testnets: `TRX_SHASTA`, `SEP`, `tBNB`, `SOL_DEVNET`). Currently only `TRX` is enabled for `USDT` |
| `destination` | string | Yes | Destination address for the withdrawal |
| `description` | string | No | Free-text description for the withdrawal |
| `referenceId` | string | No | Your own reference for this withdrawal. **Unique per merchant** — reusing a `referenceId` never creates a second payout (see [Idempotency](#3-idempotency-with-referenceid)) |

#### Response

```json
{
  "success": true,
  "message": "Withdrawal request received and will begin processing",
  "withdrawalId": "cmdl8u2xq0001abcd1234efgh",
  "status": "PENDING",
  "amount": "100.5",
  "currency": "USDT",
  "network": "TRX",
  "destination": "TXYZa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5",
  "feeInfo": {
    "grossAmount": "100.5",
    "netAmount": "98.5",
    "totalFees": "2",
    "feeBreakdown": {
      "baseFee": "1",
      "markupRate": "0",
      "markupAmount": "1",
      "networkFee": "1",
      "totalFee": "2"
    }
  }
}
```

> **Important:** fees are charged **on top of** the withdrawal amount. The destination address receives the full `amount`; your balance is debited `amount + commission + networkFee`. Make sure your available balance covers the total, not just the amount.

#### JavaScript Example

```javascript
async function createWithdrawal(currency, network, amount, destination, referenceId) {
  const response = await fetch('https://api.dcepay.io/api/withdrawals', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      currency,
      network,
      amount,     // decimal string, e.g. "100.50"
      destination,
      referenceId
    })
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(`Withdrawal creation failed: ${error.message || error.error}`);
  }

  return response.json();
}

// Usage
const withdrawal = await createWithdrawal('USDT', 'TRX', '100.50',
  'TXYZa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5', 'order_123');

console.log('Withdrawal ID:', withdrawal.withdrawalId);
console.log('Total fees (charged on top):', withdrawal.feeInfo.totalFees);
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

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by status (`PENDING`, `CONFIRMED`, `FAILED`, `CANCELLED`) |
| `page` | number | Page number (default: 1) |
| `limit` | number | Number of results per page (default: 10, max: 100) |

#### Response

```json
{
  "withdrawals": [
    {
      "id": "cmdl8u2xq0001abcd1234efgh",
      "amount": "100.5",
      "currency": "USDT",
      "status": "CONFIRMED",
      "createdAt": "2026-07-19T10:30:00Z",
      "destination": "TXYZa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5",
      "processedAt": "2026-07-19T10:31:12Z",
      "network": "TRX",
      "description": "Payout for order 123",
      "fee": "0",
      "commission": "1",
      "commissionRate": "0"
    }
  ],
  "pagination": {
    "total": 75,
    "page": 1,
    "limit": 10,
    "pages": 8
  }
}
```

## Balance Validation

Before creating a withdrawal, always check your available balance **on the network you are withdrawing from**:

### GET /api/balance

```bash
curl -X GET "https://api.dcepay.io/api/balance" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Response

Balances are returned per `(currency, network)` pair, together with the current network fee and the maximum withdrawable amount after fees:

```json
{
  "balances": [
    {
      "currency": "USDT",
      "network": "TRX",
      "available": "250.75",
      "pending": "10",
      "withdrawalEnabled": true,
      "networkFee": "1",
      "maxWithdrawable": "249.75",
      "lastUpdatedAt": "2026-07-19T10:30:00Z"
    }
  ]
}
```

You can also query a single pair with `?currency=USDT&network=TRX`, which returns one object (`currency`, `network`, `available`, `pending`, `lastUpdatedAt`).

#### Balance Validation Example

```javascript
async function validateWithdrawalBalance(currency, network, amount) {
  const response = await fetch('https://api.dcepay.io/api/balance', {
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`
    }
  });

  const { balances } = await response.json();
  const balance = balances.find(b => b.currency === currency && b.network === network);

  if (!balance) {
    throw new Error(`No ${currency} balance on ${network}`);
  }

  // Remember: total debit = amount + commission + networkFee
  if (parseFloat(balance.maxWithdrawable) < parseFloat(amount)) {
    throw new Error(`Insufficient balance on ${network}. Max withdrawable: ${balance.maxWithdrawable} ${currency}`);
  }

  return true;
}
```

## Withdrawal Status Tracking

### Status Values

| Status | Description |
|--------|-------------|
| `PENDING` | Withdrawal created and submitted, funds reserved, waiting for on-chain confirmation |
| `CONFIRMED` | Withdrawal completed and confirmed on the blockchain |
| `FAILED` | Withdrawal failed — the reserved amount (including fees) is returned to your balance |
| `CANCELLED` | Withdrawal was cancelled |

### Status Transitions

```
PENDING → CONFIRMED (when the payout is submitted and confirmed on-chain)
PENDING → FAILED (if payout initiation or on-chain processing fails)
```

When a withdrawal fails, the full reserved amount (`amount + commission + networkFee`) is released back to your available balance, and your `referenceId` is freed for reuse on a retry.

## Fee Structure

Withdrawal fees are charged **on top of** the withdrawal amount. Every withdrawal is debited:

```
total debit = amount + commission + networkFee
```

- **Commission** — your merchant withdrawal fee. Either a percentage of the amount (if a percentage rate is configured for your account) or a flat per-network fee in token units. Minimum commission: 0.10 USDT.
- **Network fee** — a per-`(currency, network)` fee quoted at submission time from the platform's asset matrix. The fee quoted at submission is what you pay, even if the finalized on-chain cost differs.

### Per-network flat fees

Default flat commission per network (token units) — your account may have custom values:

| Network | Default flat fee |
|---------|------------------|
| `TRX` | 1 |
| `ETH` | 2 |
| `BNB` | 0.2 |
| `SOL` | 0.2 |

Testnets mirror their mainnet fee.

### Fee Calculation

```javascript
// For a 100 USDT withdrawal on TRX with a 1 USDT flat commission
// and a 1 USDT network fee:
const amount = 100;
const commission = 1;   // flat per-network fee (or amount × rate if percentage)
const networkFee = 1;   // quoted from the asset matrix at submission
const totalDebit = amount + commission + networkFee; // 102 USDT debited
// Destination receives the full 100 USDT
```

### System Configuration

Withdrawal fees are configured internally by system administrators and cannot be modified via the merchant API. The fee resolution order is:

1. **Merchant percentage rate** — if configured, `commission = amount × rate` (clamped to your account's min/max charge limits)
2. **Merchant per-network flat fee** — flat fee in token units for the specific `(currency, network)` pair
3. **Merchant legacy flat fee** — single flat fee if no per-network fee is set
4. **System defaults** — system percentage or system base fee

A minimum commission of 0.10 USDT applies in all cases (unless withdrawal fees are disabled system-wide).

#### Admin API for Fee Management

Administrators can manage system-level withdrawal fee settings:

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

### Fee Information in Response

The withdrawal API response includes detailed fee information in `feeInfo`:

```json
{
  "feeInfo": {
    "grossAmount": "100",
    "netAmount": "98",
    "totalFees": "2",
    "feeBreakdown": {
      "baseFee": "1",
      "markupRate": "0",
      "markupAmount": "1",
      "networkFee": "1",
      "totalFee": "2"
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `feeBreakdown.baseFee` / `markupAmount` | The commission charged for this withdrawal |
| `feeBreakdown.markupRate` | The percentage rate applied (`0` when the commission is flat) |
| `feeBreakdown.networkFee` | The per-network fee quoted at submission |
| `feeBreakdown.totalFee` / `totalFees` | `commission + networkFee` — charged on top of `amount` |

## Reseller Payouts

If your API key belongs to a **reseller** account, withdrawals behave differently:

- The destination is pinned to your registered withdrawal address — a mismatched `destination` or `network` is rejected with `400`.
- The payout requires admin approval: funds are reserved immediately and the response returns `"status": "PENDING_APPROVAL"` with the message `"Withdrawal request received and is awaiting admin approval"`. The on-chain payout is submitted only after approval.

## Webhook Notifications

You'll receive webhook notifications when withdrawal status changes. Delivery is **at-least-once** (durable outbox with retries) — deduplicate using the `eventId` field, and acknowledge with a JSON body of `{ "ok": true }`. The event type is carried in the `X-Webhook-Event` header.

The `referenceId` (and `identifier`) in callback payloads is **your own merchant `referenceId`** from the original request — not the internal withdrawal id — so you can correlate the callback with the withdrawal you submitted. The internal id is provided separately as `withdrawalId`.

### Withdrawal Confirmed Webhook

`X-Webhook-Event: withdrawal.confirmed`

```json
{
  "type": "withdrawal",
  "withdrawalId": "cmdl8u2xq0001abcd1234efgh",
  "amount": "100.5",
  "currency": "USDT",
  "status": "confirmed",
  "txHash": "3f7a9c...",
  "referenceId": "order_123",
  "address": "TXYZa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5",
  "identifier": "order_123",
  "feeCharges": {
    "amount": "1",
    "percentage": "0",
    "type": "FIXED_AMOUNT"
  },
  "receivableAmount": "99.5",
  "eventId": "cmdl8u2xq0002abcd1234efgh"
}
```

### Withdrawal Failed Webhook

`X-Webhook-Event: withdrawal.failed`

```json
{
  "type": "withdrawal",
  "withdrawalId": "cmdl8u2xq0001abcd1234efgh",
  "amount": "100.5",
  "currency": "USDT",
  "status": "failed",
  "reason": "Withdrawal failed",
  "txHash": "3f7a9c...",
  "referenceId": "order_123",
  "address": "TXYZa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5",
  "identifier": "order_123",
  "eventId": "cmdl8u2xq0003abcd1234efgh"
}
```

On failure the reserved amount (including fees) is returned to your balance and your `referenceId` is released, so you can safely retry with the same `referenceId`.

## Error Handling

### Common Errors

| Status Code | Error | Description |
|-------------|-------|-------------|
| 400 | `Invalid request data` | Validation failed — `details` lists the offending fields |
| 400 | `Withdrawal amount must be greater than zero` | Amount is zero or negative |
| 400 | `Withdrawals of {currency} on {network} are not supported` | The `(currency, network)` pair is unknown or disabled for withdrawals |
| 400 | `Amount below minimum withdrawal` | Amount below the pair's minimum — response includes `minWithdrawal`, `currency`, `network` |
| 400 | `No {currency} balance on {network} for this account` | You hold no balance on that chain |
| 400 | `Insufficient balance` | Available balance on that chain doesn't cover `amount + fees` |
| 401 | `Authentication required` | Missing or invalid API key |
| 403 | `Insufficient permissions to initiate a withdrawal` | API key lacks write/payout capability |
| 403 | `Merchant account is not active` | Your merchant account is suspended or inactive |
| 409 | `Duplicate referenceId` | A withdrawal with this `referenceId` already exists for your merchant account |
| 500 | `Payout initiation failed` | Payout could not be processed — the reserved amount has been returned to your balance |
| 503 | `Withdrawal temporarily unavailable` | Withdrawals are temporarily unavailable — retry shortly or contact support |

### Example Error Responses

Insufficient balance (per-chain):

```json
{
  "error": "Insufficient balance",
  "message": "Insufficient USDT balance on TRX. Deposits and withdrawals are per-chain: funds on other networks cannot be used.",
  "available": "50.25",
  "requested": "100.50",
  "currency": "USDT",
  "network": "TRX",
  "feeInfo": {
    "grossAmount": "100.5",
    "netAmount": "98.5",
    "totalFees": "2",
    "feeBreakdown": {
      "baseFee": "1",
      "markupRate": "0",
      "markupAmount": "1",
      "networkFee": "1",
      "totalFee": "2"
    }
  }
}
```

Duplicate `referenceId`:

```json
{
  "error": "Duplicate referenceId",
  "message": "A withdrawal with this referenceId already exists for your merchant account. Use a new referenceId to submit a new withdrawal."
}
```

See [Error Handling](https://docs.dcepay.io/docs/error-handling) for the full guide.

## Best Practices

### 1. Address Validation

Always validate addresses before creating withdrawals:

```javascript
function validateAddress(network, address) {
  const patterns = {
    'TRX': /^T[a-zA-Z0-9]{33}$/,
    'ETH': /^0x[a-fA-F0-9]{40}$/,
    'BNB': /^0x[a-fA-F0-9]{40}$/
  };

  const pattern = patterns[network];
  if (!pattern) {
    throw new Error(`Unsupported network: ${network}`);
  }

  if (!pattern.test(address)) {
    throw new Error(`Invalid ${network} address: ${address}`);
  }

  return true;
}
```

### 2. Balance Checking

Always check the per-chain balance (including fees) before withdrawal:

```javascript
async function safeWithdrawal(currency, network, amount, destination, referenceId) {
  // 1. Validate address
  validateAddress(network, destination);

  // 2. Check balance on the target chain
  await validateWithdrawalBalance(currency, network, amount);

  // 3. Create withdrawal
  const withdrawal = await createWithdrawal(currency, network, amount, destination, referenceId);

  // 4. Log withdrawal
  console.log(`Withdrawal created: ${withdrawal.withdrawalId}`);
  console.log(`Amount: ${withdrawal.amount} ${currency} on ${network}`);
  console.log(`Fees (on top): ${withdrawal.feeInfo.totalFees} ${currency}`);

  return withdrawal;
}
```

### 3. Idempotency with referenceId

Pass your own `referenceId` in the request body to prevent duplicate payouts. The `referenceId` is unique per merchant:

- Submitting the same `referenceId` twice returns `409 Duplicate referenceId` — a second payout is **never** created.
- If a withdrawal **fails**, its `referenceId` is automatically released so you can retry with the same reference.

```javascript
async function createWithdrawalIdempotent(currency, network, amount, destination, orderId) {
  const response = await fetch('https://api.dcepay.io/api/withdrawals', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      currency,
      network,
      amount,
      destination,
      referenceId: orderId  // your idempotency key, unique per merchant
    })
  });

  if (response.status === 409) {
    // Already submitted — do NOT retry with a new referenceId unless you
    // intend to create a second payout.
    console.log(`Withdrawal for ${orderId} already exists`);
    return null;
  }

  return response.json();
}
```

### 4. Webhook Handling

Handle withdrawal webhooks properly. Delivery is at-least-once, so make handlers idempotent (deduplicate on `eventId`), and acknowledge with `{ "ok": true }`:

```javascript
app.post('/webhooks', async (req, res) => {
  const event = req.headers['x-webhook-event'];
  const payload = req.body;

  // Deduplicate — the same eventId may be delivered more than once
  if (await alreadyProcessed(payload.eventId)) {
    return res.status(200).json({ ok: true });
  }

  switch (event) {
    case 'withdrawal.confirmed':
      await handleWithdrawalConfirmed(payload);
      break;
    case 'withdrawal.failed':
      await handleWithdrawalFailed(payload);
      break;
    default:
      console.log(`Unhandled webhook event: ${event}`);
  }

  res.status(200).json({ ok: true });
});

async function handleWithdrawalConfirmed(payload) {
  const { withdrawalId, referenceId, amount, address, txHash } = payload;

  // referenceId is YOUR reference from the original request
  await updateWithdrawalStatus(referenceId, 'confirmed', txHash);
}

async function handleWithdrawalFailed(payload) {
  const { referenceId, reason } = payload;

  await updateWithdrawalStatus(referenceId, 'failed', null, reason);
  // Safe to retry the payout with the same referenceId
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

  async createWithdrawal(currency, network, amount, destination, referenceId, description) {
    const response = await fetch(`${this.baseUrl}/withdrawals`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        currency,
        network,
        amount,       // decimal string
        destination,
        referenceId,  // idempotency key, unique per merchant
        description
      })
    });

    const body = await response.json();

    if (response.status === 409) {
      throw new Error(`Duplicate referenceId: ${referenceId}`);
    }
    if (!response.ok) {
      throw new Error(`Withdrawal creation failed: ${body.message || body.error}`);
    }

    return body;
  }

  async getWithdrawals(filters = {}) {
    const params = new URLSearchParams(filters); // status, page, limit
    const response = await fetch(`${this.baseUrl}/withdrawals?${params}`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });

    return response.json();
  }

  async getBalances() {
    const response = await fetch(`${this.baseUrl}/balance`, {
      headers: {
        'Authorization': `Bearer ${this.apiKey}`
      }
    });

    return response.json();
  }
}

// Usage
const withdrawalService = new WithdrawalService(process.env.DCE_API_KEY);

try {
  const withdrawal = await withdrawalService.createWithdrawal(
    'USDT', 'TRX', '100.50',
    'TXYZa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5',
    'order_123'
  );

  console.log('Withdrawal created:', withdrawal.withdrawalId);
  console.log('Fees charged on top:', withdrawal.feeInfo.totalFees);
} catch (error) {
  console.error('Withdrawal failed:', error.message);
}
```

## Next Steps

Now that you understand withdrawals, explore:

1. [Balance Management](https://docs.dcepay.io/docs/balance-management) - Monitor your per-chain account balances
2. [Transactions](https://docs.dcepay.io/docs/transactions) - Track withdrawal lifecycle and statuses
3. [Webhooks](https://docs.dcepay.io/docs/webhooks) - Handle withdrawal notifications

---
title: API Reference
deprecated: false
hidden: false
metadata:
  robots: index
---
This document provides a comprehensive reference for DCE API HTTP endpoints. Unless otherwise noted, routes require API key authentication.

## Base URL

Configure the API host with an environment variable:

```bash
# Production
DCE_BASE_URL=https://api.dcepay.io

# Non-production (integration / QA)
DCE_BASE_URL=https://staging.dcepay.io
```

## Authentication

Send your API key in `Authorization` using **either**:

```text
Authorization: YOUR_API_KEY
```

```text
Authorization: Bearer YOUR_API_KEY
```

The hosted deposit browser flow uses `GET /api/deposit-page?token=...` and does **not** use your API key in the browser.

## Common Response Formats

### Success Response

```json
{
  "data": {...},
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "pages": 10
  }
}
```

### Error Response

```json
{
  "error": "Invalid request data",
  "details": [
    {
      "field": "amount",
      "message": "Amount must be positive",
      "value": -100
    }
  ]
}
```

## Core Endpoints

### Balance Management

#### GET /api/balance

Get user balance for a specific currency.

**Query Parameters:**

- `currency` (required): 3-letter currency code (e.g., "USD", "BTC")

**Response:**

```json
{
  "currency": "USD",
  "available": "1000.00",
  "pending": "50.00",
  "lastUpdatedAt": "2024-12-19T10:30:00Z"
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/balance?currency=USD`, {
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

if (!response.ok) {
  const errorData = await response.json();
  throw new Error(`Failed to fetch balance: ${errorData.error}`);
}

const balance = await response.json();
```

### Transactions

#### GET /api/transactions

List user transactions with pagination and filtering.

**Query Parameters:**

- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 10)
- `type` (optional): Transaction type (DEPOSIT, WITHDRAWAL)
- `status` (optional): Transaction status (PENDING, CONFIRMED, FAILED)
- `currency` (optional): 3-letter currency code

**Response:**

```json
{
  "transactions": [
    {
      "id": "txn_123",
      "type": "DEPOSIT",
      "status": "CONFIRMED",
      "amount": "100.00",
      "currency": "USD",
      "createdAt": "2024-12-19T10:30:00Z",
      "deposit": {
        "source": "wallet_address",
        "referenceId": "ref_123",
        "clearedAt": "2024-12-19T10:35:00Z",
        "fee": "2.50"
      }
    }
  ],
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 10,
    "pages": 10
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/transactions?page=1&limit=10&type=DEPOSIT`, {
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

if (!response.ok) {
  const errorData = await response.json();
  throw new Error(`Failed to fetch transactions: ${errorData.error}`);
}

const data = await response.json();
```

### Deposits

#### GET /api/deposits

List user deposits with pagination and filtering.

**Query Parameters:**

- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 10)
- `status` (optional): Deposit status (PENDING, CONFIRMED, FAILED)
- `currency` (optional): 3-letter currency code

**Response:**

```json
{
  "deposits": [
    {
      "id": "dep_123",
      "amount": "100.00",
      "currency": "USD",
      "status": "CONFIRMED",
      "source": "wallet_address",
      "createdAt": "2024-12-19T10:30:00Z",
      "clearedAt": "2024-12-19T10:35:00Z",
      "fee": "2.50"
    }
  ],
  "pagination": {
    "total": 50,
    "page": 1,
    "limit": 10,
    "pages": 5
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposits?page=1&limit=10`, {
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

if (!response.ok) {
  const errorData = await response.json();
  throw new Error(`Failed to fetch deposits: ${errorData.error}`);
}

const data = await response.json();
```

#### POST /api/deposits

Create a new deposit transaction.

**Request Body:**

```json
{
  "amount": 100.00,
  "currency": "USD",
  "source": "wallet_address",
  "description": "Payment for order #123",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  }
}
```

**Response:**

```json
{
  "id": "dep_123",
  "amount": "100.00",
  "currency": "USD",
  "status": "PENDING",
  "source": "wallet_address",
  "description": "Payment for order #123",
  "createdAt": "2024-12-19T10:30:00Z",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposits`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    amount: 100.00,
    currency: 'USD',
    source: 'wallet_address',
    description: 'Payment for order #123',
    metadata: {
      orderId: 'order_123',
      customerEmail: 'customer@example.com'
    }
  })
});

if (!response.ok) {
  const errorData = await response.json();
  if (errorData.details) {
    console.error('Validation errors:', errorData.details);
  }
  throw new Error(`Failed to create deposit: ${errorData.error}`);
}

const deposit = await response.json();
```

### Withdrawals

#### GET /api/withdrawals

List user withdrawals with pagination and filtering.

**Query Parameters:**

- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 10)
- `status` (optional): Withdrawal status (PENDING, CONFIRMED, FAILED)
- `currency` (optional): 3-letter currency code

**Response:**

```json
{
  "withdrawals": [
    {
      "id": "wth_123",
      "amount": "50.00",
      "currency": "USD",
      "status": "CONFIRMED",
      "destination": "bank_account",
      "createdAt": "2024-12-19T10:30:00Z",
      "processedAt": "2024-12-19T10:35:00Z",
      "fee": "1.50"
    }
  ],
  "pagination": {
    "total": 25,
    "page": 1,
    "limit": 10,
    "pages": 3
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/withdrawals?page=1&limit=10`, {
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

if (!response.ok) {
  const errorData = await response.json();
  throw new Error(`Failed to fetch withdrawals: ${errorData.error}`);
}

const data = await response.json();
```

#### POST /api/withdrawals

Create a new withdrawal request.

**Request Body:**

```json
{
  "amount": 50.00,
  "currency": "USD",
  "destination": "bank_account",
  "description": "Withdrawal to bank account",
  "metadata": {
    "bankAccount": "****1234",
    "customerId": "cust_123"
  }
}
```

**Response:**

```json
{
  "id": "wth_123",
  "amount": "50.00",
  "currency": "USD",
  "status": "PENDING",
  "destination": "bank_account",
  "description": "Withdrawal to bank account",
  "createdAt": "2024-12-19T10:30:00Z",
  "metadata": {
    "bankAccount": "****1234",
    "customerId": "cust_123"
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/withdrawals`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    amount: 50.00,
    currency: 'USD',
    destination: 'bank_account',
    description: 'Withdrawal to bank account',
    metadata: {
      bankAccount: '****1234',
      customerId: 'cust_123'
    }
  })
});

if (!response.ok) {
  const errorData = await response.json();
  if (errorData.details) {
    console.error('Validation errors:', errorData.details);
  }
  throw new Error(`Failed to create withdrawal: ${errorData.error}`);
}

const withdrawal = await response.json();
```

### Deposit Addresses

#### GET /api/deposit-address

Get available deposit addresses for the user.

**Query Parameters:**

- `network` (optional): Network symbol (e.g., "ETH", "BTC")

**Response:**

```json
{
  "addresses": [
    {
      "id": "addr_123",
      "address": "0x1234567890abcdef...",
      "network": "ETH",
      "createdAt": "2024-12-19T10:30:00Z",
      "deposits": [
        {
          "id": "dep_123",
          "amount": "100.00",
          "currency": "USD",
          "status": "CONFIRMED",
          "createdAt": "2024-12-19T10:35:00Z"
        }
      ]
    }
  ]
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-address?network=ETH`, {
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

if (!response.ok) {
  const errorData = await response.json();
  throw new Error(`Failed to fetch deposit addresses: ${errorData.error}`);
}

const data = await response.json();
```

#### POST /api/deposit-address

Create a new deposit address.

**Request Body:**

```json
{
  "network": "ETH",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  }
}
```

**Response:**

```json
{
  "id": "addr_123",
  "address": "0x1234567890abcdef...",
  "network": "ETH",
  "createdAt": "2024-12-19T10:30:00Z",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-address`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    network: 'ETH',
    metadata: {
      orderId: 'order_123',
      customerEmail: 'customer@example.com'
    }
  })
});

if (!response.ok) {
  const errorData = await response.json();
  if (errorData.details) {
    console.error('Validation errors:', errorData.details);
  }
  throw new Error(`Failed to create deposit address: ${errorData.error}`);
}

const address = await response.json();
```

### Deposit URLs

#### GET /api/deposit-url

Get available deposit URLs for the user.

**Query Parameters:**

- `network` (optional): Network symbol (e.g., "ETH", "BTC")

**Response:**

```json
{
  "depositUrls": [
    {
      "id": "url_123",
      "url": "https://dce.com/deposit/abc123def456",
      "address": "0x1234567890abcdef...",
      "network": "ETH",
      "createdAt": "2024-12-19T10:30:00Z",
      "deposits": [
        {
          "id": "dep_123",
          "amount": "100.00",
          "currency": "USD",
          "status": "CONFIRMED",
          "createdAt": "2024-12-19T10:35:00Z"
        }
      ]
    }
  ]
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-url?network=ETH`, {
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

if (!response.ok) {
  const errorData = await response.json();
  throw new Error(`Failed to fetch deposit URLs: ${errorData.error}`);
}

const data = await response.json();
```

#### POST /api/deposit-url

Create a new deposit URL.

**Request Body:**

```json
{
  "token": "ETH",
  "amount": 0.001,
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  }
}
```

**Response:**

```json
{
  "id": "url_123",
  "url": "https://dce.com/deposit/abc123def456",
  "token": "ETH",
  "amount": 0.001,
  "status": "active",
  "createdAt": "2024-12-19T10:30:00Z",
  "expiresAt": "2024-12-20T10:30:00Z",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-url`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    token: 'ETH',
    amount: 0.001,
    metadata: {
      orderId: 'order_123',
      customerEmail: 'customer@example.com'
    }
  })
});

if (!response.ok) {
  const errorData = await response.json();
  if (errorData.details) {
    console.error('Validation errors:', errorData.details);
  }
  throw new Error(`Failed to create deposit URL: ${errorData.error}`);
}

const depositUrl = await response.json();
```

### Exchange Rates

#### GET /api/exchange-rates

Get current exchange rates.

**Query Parameters:**

- `from` (optional): Source currency (e.g., "USD")
- `to` (optional): Target currency (e.g., "EUR")

**Response:**

```json
{
  "rates": [
    {
      "from": "USD",
      "to": "EUR",
      "rate": 0.85,
      "lastUpdated": "2024-12-19T10:30:00Z"
    },
    {
      "from": "USD",
      "to": "BTC",
      "rate": 0.000025,
      "lastUpdated": "2024-12-19T10:30:00Z"
    }
  ]
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/exchange-rates?from=USD&to=EUR`, {
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

if (!response.ok) {
  const errorData = await response.json();
  throw new Error(`Failed to fetch exchange rates: ${errorData.error}`);
}

const data = await response.json();
```

### Settlements

#### GET /api/settlements

List user settlements with pagination and filtering.

**Query Parameters:**

- `page` (optional): Page number (default: 1)
- `limit` (optional): Items per page (default: 10)
- `status` (optional): Settlement status (PENDING, APPROVED, REJECTED)
- `currency` (optional): 3-letter currency code

**Response:**

```json
{
  "settlements": [
    {
      "id": "set_123",
      "amount": "1000.00",
      "currency": "USD",
      "status": "PENDING",
      "destination": "bank_account",
      "createdAt": "2024-12-19T10:30:00Z",
      "approvedAt": null,
      "fee": "5.00"
    }
  ],
  "pagination": {
    "total": 10,
    "page": 1,
    "limit": 10,
    "pages": 1
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/settlements?page=1&limit=10`, {
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  }
});

if (!response.ok) {
  const errorData = await response.json();
  throw new Error(`Failed to fetch settlements: ${errorData.error}`);
}

const data = await response.json();
```

#### POST /api/settlements

Create a new settlement request.

**Request Body:**

```json
{
  "amount": 1000.00,
  "currency": "USD",
  "destination": "bank_account",
  "description": "Monthly settlement",
  "metadata": {
    "period": "2024-12",
    "merchantId": "merchant_123"
  }
}
```

**Response:**

```json
{
  "id": "set_123",
  "amount": "1000.00",
  "currency": "USD",
  "status": "PENDING",
  "destination": "bank_account",
  "description": "Monthly settlement",
  "createdAt": "2024-12-19T10:30:00Z",
  "metadata": {
    "period": "2024-12",
    "merchantId": "merchant_123"
  }
}
```

**Example Request:**

```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/settlements`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    amount: 1000.00,
    currency: 'USD',
    destination: 'bank_account',
    description: 'Monthly settlement',
    metadata: {
      period: '2024-12',
      merchantId: 'merchant_123'
    }
  })
});

if (!response.ok) {
  const errorData = await response.json();
  if (errorData.details) {
    console.error('Validation errors:', errorData.details);
  }
  throw new Error(`Failed to create settlement: ${errorData.error}`);
}

const settlement = await response.json();
```

### Webhooks

#### POST /api/webhook/event

Receive webhook events from the dce API.

**Request Headers:**

- `X-DCE-Signature`: HMAC signature for verification
- `Content-Type`: application/json

**Request Body:**

```json
{
  "event": "deposit.confirmed",
  "data": {
    "id": "dep_123",
    "amount": "100.00",
    "currency": "USD",
    "status": "CONFIRMED",
    "createdAt": "2024-12-19T10:30:00Z"
  },
  "timestamp": "2024-12-19T10:30:00Z"
}
```

**Response:**

```json
{
  "status": "received",
  "message": "Webhook processed successfully"
}
```

**Example Webhook Handler:**

```javascript
import crypto from 'crypto';

function verifyWebhookSignature(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(payload))
    .digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expectedSignature, 'hex')
  );
}

app.post('/webhook', async (req, res) => {
  try {
    const signature = req.headers['x-dce-signature'];
    const payload = req.body;
    
    // Verify webhook signature
    if (!verifyWebhookSignature(payload, signature, process.env.DCE_WEBHOOK_SECRET)) {
      return res.status(401).json({ error: 'Invalid signature' });
    }
    
    // Process webhook event
    const { event, data } = payload;
    
    switch (event) {
      case 'deposit.confirmed':
        await handleDepositConfirmed(data);
        break;
      case 'withdrawal.processed':
        await handleWithdrawalProcessed(data);
        break;
      case 'settlement.approved':
        await handleSettlementApproved(data);
        break;
      default:
        console.log(`Unknown event type: ${event}`);
    }
    
    res.status(200).json({ status: 'received' });
  } catch (error) {
    console.error('Webhook processing error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

## Error Handling

### Common HTTP Status Codes

| Status Code | Description                              | Common Causes                            |
| ----------- | ---------------------------------------- | ---------------------------------------- |
| `200`       | OK - Request successful                  | -                                        |
| `201`       | Created - Resource created successfully  | -                                        |
| `400`       | Bad Request - Invalid request data       | Missing required fields, invalid format  |
| `401`       | Unauthorized - Authentication required   | Missing or invalid API key               |
| `403`       | Forbidden - Insufficient permissions     | User lacks required permissions          |
| `404`       | Not Found - Resource not found           | Invalid ID, resource doesn't exist       |
| `409`       | Conflict - Resource conflict             | Duplicate data, business rule violation  |
| `422`       | Unprocessable Entity - Validation failed | Invalid data format, business validation |
| `500`       | Internal Server Error                    | Server-side error, unexpected exception  |

### Error Response Format

All API errors follow a consistent response format:

```json
{
  "error": "Invalid request data",
  "details": [
    {
      "field": "amount",
      "message": "Amount must be positive",
      "value": -100
    },
    {
      "field": "currency",
      "message": "Currency must be 3 characters",
      "value": "US"
    }
  ]
}
```

### Error Handling Examples

```javascript
// Generic error handling function
async function makeApiRequest(url, options = {}) {
  try {
    const response = await fetch(`${process.env.DCE_BASE_URL}${url}`, {
      headers: {
        'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
        'Content-Type': 'application/json',
        ...options.headers
      },
      ...options
    });

    if (!response.ok) {
      const errorData = await response.json();
      
      // Handle specific error types
      switch (response.status) {
        case 400:
          console.error('Validation errors:', errorData.details);
          throw new Error(`Invalid request: ${errorData.error}`);
        case 401:
          throw new Error('Authentication failed. Please check your API key.');
        case 403:
          throw new Error('Insufficient permissions for this operation.');
        case 404:
          throw new Error('Resource not found.');
        case 409:
          throw new Error('Resource conflict. Please check your request.');
        case 422:
          console.error('Validation errors:', errorData.details);
          throw new Error(`Validation failed: ${errorData.error}`);
        default:
          throw new Error(`Unexpected error: ${errorData.error || 'Unknown error'}`);
      }
    }

    return await response.json();
  } catch (error) {
    console.error('API request failed:', error.message);
    throw error;
  }
}

// Usage examples
try {
  // Get balance
  const balance = await makeApiRequest('/api/balance?currency=USD');
  console.log('Balance:', balance);
  
  // Create deposit
  const deposit = await makeApiRequest('/api/deposits', {
    method: 'POST',
    body: JSON.stringify({
      amount: 100.00,
      currency: 'USD',
      source: 'wallet_address',
      description: 'Payment for order #123'
    })
  });
  console.log('Deposit created:', deposit);
  
} catch (error) {
  console.error('Operation failed:', error.message);
  // Handle error appropriately in your application
}
```

## Environment Configuration

Set up your environment variables for different environments:

```bash
# .env file
DCE_API_KEY=v8_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_BASE_URL=https://api.dcepay.io
DCE_WEBHOOK_SECRET=your_webhook_secret_here

# Test environment
DCE_TEST_API_KEY=v8_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_TEST_BASE_URL=https://staging.dcepay.io
DCE_TEST_WEBHOOK_SECRET=test_webhook_secret_here

# Staging environment
DCE_STAGING_API_KEY=v8_staging_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_STAGING_BASE_URL=https://staging.dcepay.io
DCE_STAGING_WEBHOOK_SECRET=staging_webhook_secret_here
```

## Rate Limits

The API implements rate limiting to ensure fair usage:

- **Authentication**: 100 requests/minute
- **Balance**: 1000 requests/minute
- **Deposits**: 100 requests/minute
- **Withdrawals**: 50 requests/minute
- **Transactions**: 500 requests/minute
- **Webhooks**: 1000 requests/minute

When rate limits are exceeded, the API returns a `429 Too Many Requests` status code with details about the rate limit:

```json
{
  "error": "rate_limit_exceeded",
  "message": "Rate limit exceeded. Please try again later.",
  "details": {
    "limit": 100,
    "remaining": 0,
    "reset": 1640000000
  }
}
```

***

_For more information about specific features, see the [Authentication](authentication.md), [Deposits](deposits.md), [Withdrawals](withdrawals.md), and [Webhooks](webhooks.md) documentation._
Fee behavior is documented in [Fees Reference](fees-reference.md).

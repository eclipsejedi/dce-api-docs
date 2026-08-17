---
title: Deposits
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-07-30_

The deposits API allows you to accept customer payments by creating deposit addresses and tracking payment status. This guide covers all aspects of deposit management including address creation, URL generation, payment tracking, and how deposits are credited and charged.

## Overview

The deposit flow consists of three main steps:

1. **Create a deposit address** - Generate a unique address for customer payments
2. **Share the address with customers** - Provide the address or payment URL to customers
3. **Monitor payments** - Track incoming payments via webhooks and API calls

### Supported currencies and networks

Deposits are stablecoin-only: `USDT` and `USDC`, segmented per network (`TRX`, `ETH`, `BNB`, `SOL`; testnets `TRX_SHASTA`, `SEP`, `tBNB`, `SOL_DEVNET`). Balances are tracked **per (currency, network) pair** — funds deposited on one chain can only be withdrawn on that same chain.

> **Availability:** USDT on TRX (and its `TRX_SHASTA` testnet twin) is the pair enabled today. Other pairs (USDT/USDC on ETH, BNB, SOL) are coming soon / available on request. Requests for a network with no enabled pair are rejected with `400` and the message `Deposits on {network} are not currently supported`.

## Creating Deposit Addresses

### POST /api/deposit-address

Create a new deposit address on a specific network. Addresses are per network — an address receives any supported token on that chain.

#### Request

```bash
curl -X POST "${DCE_BASE_URL}/api/deposit-address" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "network": "TRX",
    "identifier": "customer_123",
    "referenceId": "order_123"
  }'
```

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `network` | string | Yes | One of `TRX`, `ETH`, `BNB`, `SOL`, `TRX_SHASTA`, `SEP`, `tBNB`, `SOL_DEVNET` (`NetworkSymbol`). Only networks with an enabled (currency, network) pair are accepted — TRX today. |
| `identifier` | string | Yes | Your end-user or customer identifier |
| `referenceId` | string | No | Optional business reference (order id, invoice id, etc.) |

#### Response

`201 Created`:

```json
{
  "address": "TX7k1zNq7PqR3sT9uV2wX4yZ6aB8cD1eF3",
  "identifier": "merchant_abc_customer_123",
  "network": "TRX",
  "referenceId": "order_123"
}
```

Note that the returned `identifier` is prefixed with your merchant id (`{merchantId}_{identifier}`); `referenceId` is echoed back only when you supplied one.

#### JavaScript Example

```javascript
async function createDepositAddress(network, identifier, referenceId) {
  const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-address`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.DCE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      network,
      identifier,
      ...(referenceId ? { referenceId } : {}),
    }),
  });

  if (!response.ok) {
    const errorData = await response.json().catch(() => ({}));
    throw new Error(errorData.error || `HTTP ${response.status}`);
  }

  return response.json();
}

await createDepositAddress('TRX', 'customer_123', 'order_123');
```

#### Python Example

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def create_deposit_address(network, identifier, reference_id=None):
    payload = {
        'network': network,
        'identifier': identifier
    }
    if reference_id:
        payload['referenceId'] = reference_id

    try:
        response = requests.post(
            f'{os.getenv("DCE_BASE_URL")}/api/deposit-address',
            json=payload,
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )
        
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f'Failed to create deposit address: {e}')
        raise

# Usage
try:
    deposit = create_deposit_address('TRX', 'customer_123', 'order_123')
    print('Deposit address:', deposit['address'])
except Exception as e:
    print(f'Error creating deposit address: {e}')
```

## Creating Deposit URLs

### POST /api/deposit-url

Create a shareable hosted payment URL for customers. A fresh deposit address is provisioned and wrapped in a short-lived session (15 minutes). See [Hosted deposit page](https://docs.dcepay.io/docs/hosted-payments) for the customer-facing flow and live payment status.

Your merchant account must be **ACTIVE** — deposit address and deposit URL creation return **403** (`Merchant account is not active`) for suspended or inactive merchant profiles, matching the withdrawals API.

#### Request

```bash
curl -X POST "${DCE_BASE_URL}/api/deposit-url" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "network": "TRX",
    "identifier": "customer_123",
    "referenceId": "order_123",
    "requestedCurrency": "USD",
    "requestedAmount": "100.00"
  }'
```

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `network` | string | Yes | Network symbol (`NetworkSymbol`); only networks with an enabled pair are accepted — TRX today |
| `identifier` | string | Yes | Your end-user or customer identifier |
| `referenceId` | string | No | Optional business reference; appended to the URL as `ref` |
| `requestedCurrency` | string | No | 3-letter display currency for the hosted page (e.g. `USD`) |
| `requestedAmount` | string | No | Requested amount in `requestedCurrency`; shown on the hosted page |
| `token` | string | No | Optional display token hint, appended to the URL |

#### Response

`201 Created`:

```json
{
  "url": "https://app.dcepay.io/deposit/abc123def456?amount=100.00&currency=USD&ref=order_123",
  "address": "TX7k1zNq7PqR3sT9uV2wX4yZ6aB8cD1eF3",
  "identifier": "customer_123",
  "network": "TRX",
  "referenceId": "order_123",
  "requestedAmount": "100.00",
  "requestedCurrency": "USD",
  "sessionToken": "abc123def456",
  "expiresAt": "2026-07-29T10:45:00.000Z",
  "exchangeRate": "1.0002",
  "amount": "99.98000000"
}
```

| Field | Description |
|-------|-------------|
| `url` | Customer-facing hosted payment link |
| `address` | The deposit address behind the session |
| `sessionToken` | Session token embedded in the URL |
| `expiresAt` | Session expiry (15 minutes from creation) |
| `exchangeRate` | Present when `requestedAmount` + `requestedCurrency` were supplied and a rate is available |
| `amount` | Crypto amount quoted from `requestedAmount` at `exchangeRate` (8 dp, rounded up so the customer never under-pays) |

`referenceId`, `requestedAmount`, `requestedCurrency`, and `token` are echoed back only when supplied.

#### JavaScript Example

```javascript
async function createDepositUrl(network, identifier, options = {}) {
  try {
    const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-url`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        network,
        identifier,
        ...options // referenceId, requestedCurrency, requestedAmount, token
      })
    });

    if (!response.ok) {
      const errorData = await response.json();
      if (errorData.details) {
        console.error('Validation errors:', errorData.details);
      }
      throw new Error(`Failed to create deposit URL: ${errorData.error}`);
    }

    return response.json();
  } catch (error) {
    console.error('Failed to create deposit URL:', error.message);
    throw error;
  }
}

// Usage
try {
  const depositUrl = await createDepositUrl('TRX', 'customer_123', {
    referenceId: 'order_123',
    requestedCurrency: 'USD',
    requestedAmount: '100.00'
  });

  console.log('Deposit URL:', depositUrl.url);
} catch (error) {
  console.error('Error creating deposit URL:', error.message);
}
```

#### Python Example

```python
def create_deposit_url(network, identifier, reference_id=None,
                       requested_currency=None, requested_amount=None):
    payload = {
        'network': network,
        'identifier': identifier
    }
    if reference_id:
        payload['referenceId'] = reference_id
    if requested_currency and requested_amount:
        payload['requestedCurrency'] = requested_currency
        payload['requestedAmount'] = requested_amount

    try:
        response = requests.post(
            f'{os.getenv("DCE_BASE_URL")}/api/deposit-url',
            json=payload,
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )
        
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f'Failed to create deposit URL: {e}')
        raise

# Usage
try:
    deposit_url = create_deposit_url(
        'TRX', 'customer_123',
        reference_id='order_123',
        requested_currency='USD',
        requested_amount='100.00'
    )
    print('Deposit URL:', deposit_url['url'])
except Exception as e:
    print(f'Error creating deposit URL: {e}')
```

## Listing Deposits

### GET /api/deposits

List your deposits with pagination and status filtering.

#### Request

```bash
curl -X GET "${DCE_BASE_URL}/api/deposits?page=1&limit=10&status=CONFIRMED" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `page` | number | No | Page number (default: 1) |
| `limit` | number | No | Items per page (default: 10) |
| `status` | string | No | Filter by status (`PENDING`, `CONFIRMED`, `FAILED`, `CANCELLED`) |

#### Response

Each deposit row includes its `currency` and `network`, plus the linked `depositAddress` record when the deposit came in on an issued address.

```json
{
  "deposits": [
    {
      "id": "dep_123",
      "amount": "100.5",
      "currency": "USDT",
      "network": "TRX",
      "status": "CONFIRMED",
      "source": "wallet_address",
      "referenceId": "order_123",
      "description": null,
      "createdAt": "2026-07-29T10:30:00.000Z",
      "updatedAt": "2026-07-29T10:35:00.000Z",
      "clearedAt": "2026-07-29T10:35:00.000Z",
      "fee": "0",
      "commission": "1.005",
      "commissionRate": "0.01",
      "depositAddressId": "addr_123",
      "depositAddress": {
        "id": "addr_123",
        "address": "TX7k1zNq7PqR3sT9uV2wX4yZ6aB8cD1eF3",
        "networkSymbol": "TRX"
      },
      "metadata": {
        "txHash": "d0e1f2..."
      }
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

> **Fee fields:** `commission` is the percentage deposit commission (with `commissionRate` the rate applied). `fee` mirrors fixed charges tied to the deposit — the address activation fee or the direct-deposit flat fee — for audit purposes; it is **not** the deposit commission. See [Crediting and fees](#crediting-and-fees) below and the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).

#### JavaScript Example

```javascript
async function getDeposits(options = {}) {
  const { page = 1, limit = 10, status } = options;
  
  try {
    const params = new URLSearchParams({
      page: page.toString(),
      limit: limit.toString(),
      ...(status && { status })
    });

    const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposits?${params}`, {
      headers: {
        'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
        'Content-Type': 'application/json'
      }
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Failed to fetch deposits: ${errorData.error}`);
    }

    return response.json();
  } catch (error) {
    console.error('Failed to fetch deposits:', error.message);
    throw error;
  }
}

// Usage
try {
  const deposits = await getDeposits({
    page: 1,
    limit: 10,
    status: 'CONFIRMED'
  });

  console.log('Deposits:', deposits.deposits);
  console.log('Total:', deposits.pagination.total);
} catch (error) {
  console.error('Error fetching deposits:', error.message);
}
```

#### Python Example

```python
def get_deposits(page=1, limit=10, status=None):
    params = {
        'page': page,
        'limit': limit
    }
    
    if status:
        params['status'] = status
    
    try:
        response = requests.get(
            f'{os.getenv("DCE_BASE_URL")}/api/deposits',
            params=params,
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )
        
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f'Failed to fetch deposits: {e}')
        raise

# Usage
try:
    deposits = get_deposits(
        page=1,
        limit=10,
        status='CONFIRMED'
    )
    print('Deposits:', deposits['deposits'])
    print('Total:', deposits['pagination']['total'])
except Exception as e:
    print(f'Error fetching deposits: {e}')
```

## Creating Deposits

### POST /api/deposits

Create a new deposit record manually. Requires an API key with write permission. The deposit is created in `PENDING` status and the amount is credited to the **pending** balance of the (currency, network) pair.

#### Request

```bash
curl -X POST "${DCE_BASE_URL}/api/deposits" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": "100.00",
    "currency": "USDT",
    "network": "TRX",
    "source": "wallet_address",
    "description": "Payment for order #123",
    "metadata": {
      "orderId": "order_123",
      "customerEmail": "customer@example.com"
    }
  }'
```

#### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `amount` | string | Yes | Deposit amount as a positive decimal string (up to 18 decimal places) |
| `currency` | string | Yes | Token symbol: `USDT` or `USDC` |
| `network` | string | Yes | Network symbol (`TRX`, `ETH`, `BNB`, `SOL`, or a testnet) |
| `source` | string | Yes | Source of the deposit |
| `description` | string | No | Human-readable description |
| `metadata` | object | No | Additional data to associate with the deposit |
| `depositAddressId` | string | No | Link the deposit to an existing deposit address |

#### Response

The created deposit record:

```json
{
  "id": "dep_123",
  "amount": "100",
  "currency": "USDT",
  "network": "TRX",
  "status": "PENDING",
  "source": "wallet_address",
  "description": "Payment for order #123",
  "createdAt": "2026-07-29T10:30:00.000Z",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  }
}
```

#### JavaScript Example

```javascript
async function createDeposit(depositData) {
  try {
    const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposits`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(depositData)
    });

    if (!response.ok) {
      const errorData = await response.json();
      if (errorData.details) {
        console.error('Validation errors:', errorData.details);
      }
      throw new Error(`Failed to create deposit: ${errorData.error}`);
    }

    return response.json();
  } catch (error) {
    console.error('Failed to create deposit:', error.message);
    throw error;
  }
}

// Usage
try {
  const deposit = await createDeposit({
    amount: '100.00',
    currency: 'USDT',
    network: 'TRX',
    source: 'wallet_address',
    description: 'Payment for order #123',
    metadata: {
      orderId: 'order_123',
      customerEmail: 'customer@example.com'
    }
  });

  console.log('Deposit created:', deposit.id);
} catch (error) {
  console.error('Error creating deposit:', error.message);
}
```

#### Python Example

```python
def create_deposit(deposit_data):
    try:
        response = requests.post(
            f'{os.getenv("DCE_BASE_URL")}/api/deposits',
            json=deposit_data,
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )
        
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f'Failed to create deposit: {e}')
        raise

# Usage
try:
    deposit = create_deposit({
        'amount': '100.00',
        'currency': 'USDT',
        'network': 'TRX',
        'source': 'wallet_address',
        'description': 'Payment for order #123',
        'metadata': {
            'orderId': 'order_123',
            'customerEmail': 'customer@example.com'
        }
    })
    print('Deposit created:', deposit['id'])
except Exception as e:
    print(f'Error creating deposit: {e}')
```

## Crediting and fees

How incoming deposits are credited and what they are charged:

### Customer deposits (issued deposit addresses)

- Credited to your **available** balance for the deposit's (currency, network) pair when confirmed.
- Charged the percentage **deposit commission** (recorded on the deposit as `commission`, at `commissionRate`) — see the [Fees Reference](https://docs.dcepay.io/docs/fees-reference).
- **Address activation fee:** a fixed 0.5 fee (in the deposit's currency) is charged once per deposit address, on the first confirmed deposit to that address. It appears in your charge ledger as `Address activation fee (fixed)` and is mirrored on the deposit's `fee` field.
- **Small-deposit exemption:** deposits under **1 USD equivalent** are charged no deposit fee and trigger **no merchant deposit callback**. They are still credited and appear in your deposit list.

### Wallet top-ups (direct deposits)

Deposits to your own self-custody top-up address (funding your balance, e.g. to cover withdrawals) behave differently:

- **Sweep-gated crediting:** the deposit is credited to your **pending** balance first and moves to **available** only once the sweep to the master wallet lands. Top-up funds may sit in pending briefly.
- They skip the percentage commission and the activation fee. Instead a **flat direct-deposit fee** is charged in-kind (default 1 USDT/USDC; configurable per merchant). It appears in your charge ledger as `Direct deposit fee (fixed)` and is mirrored on the deposit's `fee` field.

## Error Handling

### Common Error Scenarios

| Error | Description | Resolution |
|-------|-------------|------------|
| `Deposits on {network} are not currently supported` | The network has no enabled (currency, network) pair | Use an enabled network (TRX today) |
| `Invalid request data` | Validation failed (see `details`) | Fix the fields listed in `details` |
| `Deposit address already exists` | The address was already generated | Reuse the existing address (`409`) |
| `Unauthorized` / `Authentication required` | Missing or invalid API key | Check your `Authorization` header |
| `Insufficient permissions` | API key lacks the required permission | Use a key with read/write permission as needed |

### Error Response Format

```json
{
  "error": "Invalid request data",
  "details": [
    {
      "path": ["network"],
      "message": "Invalid enum value",
      "code": "invalid_enum_value"
    }
  ]
}
```

### Error Handling Examples

```javascript
// Generic error handling function
async function handleApiRequest(url, options = {}) {
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
          throw new Error(`Conflict: ${errorData.error}`);
        case 429:
          throw new Error('Rate limit exceeded. Please try again later.');
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

// Usage with error handling
try {
  const deposit = await handleApiRequest('/api/deposits', {
    method: 'POST',
    body: JSON.stringify({
      amount: '100.00',
      currency: 'USDT',
      network: 'TRX',
      source: 'wallet_address'
    })
  });
  
  console.log('Deposit created successfully:', deposit);
} catch (error) {
  console.error('Failed to create deposit:', error.message);
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
DCE_TEST_BASE_URL=https://api.dcepay.dev
DCE_TEST_WEBHOOK_SECRET=test_webhook_secret_here

# Staging environment
DCE_STAGING_API_KEY=v8_staging_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_STAGING_BASE_URL=https://api.dcepay.dev
DCE_STAGING_WEBHOOK_SECRET=staging_webhook_secret_here
```

## Best Practices

### 1. Address Management

- **Reuse addresses** when possible to reduce blockchain fees — the activation fee is charged once per address, so churning addresses costs more
- **Monitor address usage** to detect unusual activity
- **Implement address rotation** for security
- **Validate addresses** before using them

### 2. Payment Tracking

- **Use webhooks** for real-time payment notifications — note that deliveries are **at-least-once**, so make your consumer idempotent
- **Implement polling** as a fallback for webhook failures
- **Store payment metadata** for reconciliation
- **Handle payment confirmations** appropriately
- **Remember the small-deposit exemption**: deposits under 1 USD equivalent do not trigger a deposit callback — reconcile them via `GET /api/deposits`

### 3. Error Handling

- **Implement retry logic** with exponential backoff
- **Log all errors** for debugging and monitoring
- **Provide user-friendly error messages**
- **Handle rate limits** gracefully

### 4. Security

- **Validate all inputs** before sending to API
- **Use HTTPS** for all API communications
- **Store sensitive data** securely
- **Monitor for suspicious activity**

---

*For more information about payment processing, see the [Transactions](https://docs.dcepay.io/docs/transactions) and [Webhooks](https://docs.dcepay.io/docs/webhooks) documentation.* 

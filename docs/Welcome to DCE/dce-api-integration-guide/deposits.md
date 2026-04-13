---
title: Deposits
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

The deposits API allows you to accept customer payments by creating deposit addresses and tracking payment status. This guide covers all aspects of deposit management including address creation, URL generation, and payment tracking.

## Overview

The deposit flow consists of three main steps:

1. **Create a deposit address** - Generate a unique address for customer payments
2. **Share the address with customers** - Provide the address or payment URL to customers
3. **Monitor payments** - Track incoming payments via webhooks and API calls

## Creating Deposit Addresses

### POST /api/deposit-address

Create a new deposit address for a specific cryptocurrency.

#### Request

```bash
curl -X POST "${DCE_BASE_URL}/api/deposit-address" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "network": "ETH",
    "identifier": "customer_123",
    "referenceId": "order_123"
  }'
```

#### Request Parameters

| Parameter     | Type   | Required | Description                                                               |
| ------------- | ------ | -------- | ------------------------------------------------------------------------- |
| `network`     | string | Yes      | One of `ETH`, `TRX`, `SEP`, `TRX_SHASTA`, `BNB`, `tBNB` (`NetworkSymbol`) |
| `identifier`  | string | Yes      | Your end-user or customer identifier                                      |
| `referenceId` | string | No       | Optional business reference (order id, invoice id, etc.)                  |

#### Response

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

await createDepositAddress('ETH', 'customer_123', 'order_123');
```

#### Python Example

```python
import requests
import os
from dotenv import load_dotenv

load_dotenv()

def create_deposit_address(network, metadata=None):
    if metadata is None:
        metadata = {}
    
    try:
        response = requests.post(
            f'{os.getenv("DCE_BASE_URL")}/api/deposit-address',
            json={
                'network': network,
                'metadata': metadata
            },
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
    deposit = create_deposit_address('ETH', {
        'orderId': 'order_123',
        'customerEmail': 'customer@example.com'
    })
    print('Deposit address:', deposit['address'])
except Exception as e:
    print(f'Error creating deposit address: {e}')
```

## Creating Deposit URLs

### POST /api/deposit-url

Create a shareable payment URL for customers.

#### Request

```bash
curl -X POST "${DCE_BASE_URL}/api/deposit-url" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "token": "ETH",
    "amount": 0.001,
    "metadata": {
      "orderId": "order_123",
      "customerEmail": "customer@example.com"
    }
  }'
```

#### Response

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

#### JavaScript Example

```javascript
async function createDepositUrl(token, amount, metadata = {}) {
  try {
    const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-url`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        token,
        amount,
        metadata
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
  const depositUrl = await createDepositUrl('ETH', 0.001, {
    orderId: 'order_123',
    customerEmail: 'customer@example.com'
  });

  console.log('Deposit URL:', depositUrl.url);
} catch (error) {
  console.error('Error creating deposit URL:', error.message);
}
```

#### Python Example

```python
def create_deposit_url(token, amount, metadata=None):
    if metadata is None:
        metadata = {}
    
    try:
        response = requests.post(
            f'{os.getenv("DCE_BASE_URL")}/api/deposit-url',
            json={
                'token': token,
                'amount': amount,
                'metadata': metadata
            },
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
    deposit_url = create_deposit_url('ETH', 0.001, {
        'orderId': 'order_123',
        'customerEmail': 'customer@example.com'
    })
    print('Deposit URL:', deposit_url['url'])
except Exception as e:
    print(f'Error creating deposit URL: {e}')
```

## Listing Deposits

### GET /api/deposits

List user deposits with pagination and filtering.

#### Request

```bash
curl -X GET "${DCE_BASE_URL}/api/deposits?page=1&limit=10&status=CONFIRMED" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

#### Query Parameters

| Parameter  | Type   | Required | Description                                   |
| ---------- | ------ | -------- | --------------------------------------------- |
| `page`     | number | No       | Page number (default: 1)                      |
| `limit`    | number | No       | Items per page (default: 10)                  |
| `status`   | string | No       | Filter by status (PENDING, CONFIRMED, FAILED) |
| `currency` | string | No       | Filter by currency (USD, BTC, ETH, etc.)      |

#### Response

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

#### JavaScript Example

```javascript
async function getDeposits(options = {}) {
  const { page = 1, limit = 10, status, currency } = options;
  
  try {
    const params = new URLSearchParams({
      page: page.toString(),
      limit: limit.toString(),
      ...(status && { status }),
      ...(currency && { currency })
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
    status: 'CONFIRMED',
    currency: 'USD'
  });

  console.log('Deposits:', deposits.deposits);
  console.log('Total:', deposits.pagination.total);
} catch (error) {
  console.error('Error fetching deposits:', error.message);
}
```

#### Python Example

```python
def get_deposits(page=1, limit=10, status=None, currency=None):
    params = {
        'page': page,
        'limit': limit
    }
    
    if status:
        params['status'] = status
    if currency:
        params['currency'] = currency
    
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
        status='CONFIRMED',
        currency='USD'
    )
    print('Deposits:', deposits['deposits'])
    print('Total:', deposits['pagination']['total'])
except Exception as e:
    print(f'Error fetching deposits: {e}')
```

## Creating Deposits

### POST /api/deposits

Create a new deposit transaction.

#### Request

```bash
curl -X POST "${DCE_BASE_URL}/api/deposits" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 100.00,
    "currency": "USD",
    "source": "wallet_address",
    "description": "Payment for order #123",
    "metadata": {
      "orderId": "order_123",
      "customerEmail": "customer@example.com"
    }
  }'
```

#### Request Parameters

| Parameter     | Type   | Required | Description                                   |
| ------------- | ------ | -------- | --------------------------------------------- |
| `amount`      | number | Yes      | Deposit amount                                |
| `currency`    | string | Yes      | Currency code (USD, BTC, ETH, etc.)           |
| `source`      | string | Yes      | Source of the deposit                         |
| `description` | string | No       | Human-readable description                    |
| `metadata`    | object | No       | Additional data to associate with the deposit |

#### Response

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
    amount: 100.00,
    currency: 'USD',
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
        'amount': 100.00,
        'currency': 'USD',
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

## Error Handling

### Common Error Scenarios

| Error                     | Description                 | Resolution                               |
| ------------------------- | --------------------------- | ---------------------------------------- |
| `Invalid network`         | Unsupported cryptocurrency  | Use a supported network (ETH, BTC, etc.) |
| `Invalid amount`          | Amount is negative or zero  | Provide a positive amount                |
| `Missing required fields` | Required parameters missing | Include all required fields              |
| `Rate limit exceeded`     | Too many requests           | Wait and retry with exponential backoff  |

### Error Response Format

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
      "field": "network",
      "message": "Invalid network symbol",
      "value": "INVALID"
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
      amount: 100.00,
      currency: 'USD',
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
DCE_TEST_BASE_URL=https://staging.dcepay.io
DCE_TEST_WEBHOOK_SECRET=test_webhook_secret_here

# Staging environment
DCE_STAGING_API_KEY=v8_staging_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_STAGING_BASE_URL=https://staging.dcepay.io
DCE_STAGING_WEBHOOK_SECRET=staging_webhook_secret_here
```

## Best Practices

### 1. Address Management

- **Reuse addresses** when possible to reduce blockchain fees
- **Monitor address usage** to detect unusual activity
- **Implement address rotation** for security
- **Validate addresses** before using them

### 2. Payment Tracking

- **Use webhooks** for real-time payment notifications
- **Implement polling** as a fallback for webhook failures
- **Store payment metadata** for reconciliation
- **Handle payment confirmations** appropriately

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

***

_For more information about payment processing, see the [Transactions](transactions.md) and [Webhooks](webhooks.md) documentation._

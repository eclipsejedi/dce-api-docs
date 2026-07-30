---
title: Testing Guide
deprecated: false
hidden: false
metadata:
  robots: index
---
Comprehensive testing guide for the dce API integration. This guide covers testing strategies, tools, and best practices to ensure your integration works correctly in both development and production environments.

## Overview

Testing your dce API integration is crucial for ensuring reliable payment processing. This guide covers:

- **Unit Testing** - Test individual API calls and functions
- **Integration Testing** - Test complete payment flows
- **Webhook Testing** - Verify webhook delivery and processing
- **Load Testing** - Test performance under various conditions
- **Security Testing** - Verify authentication and authorization
- **Error Testing** - Test error handling and edge cases

## Testing Environment

### Test API Keys

Use test API keys for all development and testing:

```bash
# Test environment configuration
DCE_API_KEY=v8_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_BASE_URL=https://api.dcepay.dev
DCE_WEBHOOK_SECRET=test_webhook_secret_here
```

### Test Data

The test environment provides:

- **Test Balances** - Simulated account balances
- **Test Transactions** - Mock transaction data
- **Test Webhooks** - Webhook simulation tools
- **Test Addresses** - Valid cryptocurrency addresses for testing

## Unit Testing

### Testing Authentication

**JavaScript (Jest):**
```javascript
const axios = require('axios');

describe('Authentication Tests', () => {
  test('should authenticate with valid API key', async () => {
    const response = await axios.get('https://api.dcepay.dev/api/balance?currency=USD', {
      headers: {
        'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
        'Content-Type': 'application/json'
      }
    });
    
    expect(response.status).toBe(200);
    expect(response.data.status).toBe('healthy');
  });

  test('should reject invalid API key', async () => {
    try {
      await axios.get('https://api.dcepay.dev/api/balance', {
        headers: {
          'Authorization': 'Bearer invalid_key',
          'Content-Type': 'application/json'
        }
      });
      fail('Should have thrown an error');
    } catch (error) {
      expect(error.response.status).toBe(401);
      expect(error.response.data.error).toBe('unauthorized');
    }
  });
});
```

**Python (pytest):**
```python
import pytest
import requests
import os

class TestAuthentication:
    def test_valid_api_key(self):
        response = requests.get(
            'https://api.dcepay.dev/api/balance?currency=USD',
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )
        
        assert response.status_code == 200
        assert response.json()['status'] == 'healthy'
    
    def test_invalid_api_key(self):
        with pytest.raises(requests.exceptions.HTTPError) as exc_info:
            requests.get(
                'https://api.dcepay.dev/api/balance',
                headers={
                    'Authorization': 'Bearer invalid_key',
                    'Content-Type': 'application/json'
                }
            )
        
        assert exc_info.value.response.status_code == 401
        assert exc_info.value.response.json()['error'] == 'unauthorized'
```

### Testing Deposit Creation

**JavaScript:**
```javascript
describe('Deposit Tests', () => {
  test('should create deposit address', async () => {
    const depositData = {
      token: 'ETH',
      amount: 0.1,
      metadata: {
        orderId: 'test_order_123',
        customerEmail: 'test@example.com'
      }
    };

    const response = await axios.post(
      'https://api.dcepay.dev/api/deposit-address',
      depositData,
      {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    expect(response.status).toBe(201);
    expect(response.data).toHaveProperty('id');
    expect(response.data).toHaveProperty('address');
    expect(response.data.token).toBe('ETH');
    expect(response.data.amount).toBe(0.1);
  });

  test('should validate required fields', async () => {
    const invalidData = {
      token: 'ETH'
      // Missing required 'amount' field
    };

    try {
      await axios.post(
        'https://api.dcepay.dev/api/deposit-address',
        invalidData,
        {
          headers: {
            'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
            'Content-Type': 'application/json'
          }
        }
      );
      fail('Should have thrown validation error');
    } catch (error) {
      expect(error.response.status).toBe(400);
      expect(error.response.data.error).toBe('validation_error');
    }
  });
});
```

**Python:**
```python
class TestDeposits:
    def test_create_deposit_address(self):
        deposit_data = {
            'token': 'ETH',
            'amount': 0.1,
            'metadata': {
                'orderId': 'test_order_123',
                'customerEmail': 'test@example.com'
            }
        }

        response = requests.post(
            'https://api.dcepay.dev/api/deposit-address',
            json=deposit_data,
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )

        assert response.status_code == 201
        data = response.json()
        assert 'id' in data
        assert 'address' in data
        assert data['token'] == 'ETH'
        assert data['amount'] == 0.1

    def test_validation_error(self):
        invalid_data = {
            'token': 'ETH'
            # Missing required 'amount' field
        }

        response = requests.post(
            'https://api.dcepay.dev/api/deposit-address',
            json=invalid_data,
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )

        assert response.status_code == 400
        assert response.json()['error'] == 'validation_error'
```

## Integration Testing

### Complete Payment Flow Test

**JavaScript:**
```javascript
describe('Complete Payment Flow', () => {
  let depositId;
  let webhookReceived = false;

  test('should complete full payment flow', async () => {
    // Step 1: Create deposit address
    const depositResponse = await axios.post(
      'https://api.dcepay.dev/api/deposit-address',
      {
        token: 'ETH',
        amount: 0.1,
        metadata: {
          orderId: 'integration_test_123',
          customerEmail: 'test@example.com'
        }
      },
      {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    expect(depositResponse.status).toBe(201);
    depositId = depositResponse.data.id;

    // Step 2: Create payment URL
    const urlResponse = await axios.post(
      'https://api.dcepay.dev/api/deposit-url',
      {
        depositId: depositId,
        customization: {
          title: 'Test Payment',
          description: 'Integration test payment'
        }
      },
      {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    expect(urlResponse.status).toBe(201);
    expect(urlResponse.data).toHaveProperty('url');

    // Step 3: Check deposit status
    const statusResponse = await axios.get(
      `https://api.dcepay.dev/api/deposits/${depositId}`,
      {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    expect(statusResponse.status).toBe(200);
    expect(statusResponse.data.status).toBe('pending');

    // Step 4: Wait for webhook (in real scenario)
    // This would be handled by your webhook endpoint
    await new Promise(resolve => setTimeout(resolve, 5000));

    // Step 5: Verify final status
    const finalStatusResponse = await axios.get(
      `https://api.dcepay.dev/api/deposits/${depositId}`,
      {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    expect(finalStatusResponse.status).toBe(200);
    // Status could be 'confirmed' or 'pending' depending on test environment
  });
});
```

**Python:**
```python
import time

class TestPaymentFlow:
    def test_complete_payment_flow(self):
        # Step 1: Create deposit address
        deposit_data = {
            'token': 'ETH',
            'amount': 0.1,
            'metadata': {
                'orderId': 'integration_test_123',
                'customerEmail': 'test@example.com'
            }
        }

        deposit_response = requests.post(
            'https://api.dcepay.dev/api/deposit-address',
            json=deposit_data,
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )

        assert deposit_response.status_code == 201
        deposit_id = deposit_response.json()['id']

        # Step 2: Create payment URL
        url_data = {
            'depositId': deposit_id,
            'customization': {
                'title': 'Test Payment',
                'description': 'Integration test payment'
            }
        }

        url_response = requests.post(
            'https://api.dcepay.dev/api/deposit-url',
            json=url_data,
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )

        assert url_response.status_code == 201
        assert 'url' in url_response.json()

        # Step 3: Check deposit status
        status_response = requests.get(
            f'https://api.dcepay.dev/api/deposits/{deposit_id}',
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )

        assert status_response.status_code == 200
        assert status_response.json()['status'] == 'pending'

        # Step 4: Wait for webhook processing
        time.sleep(5)

        # Step 5: Verify final status
        final_status_response = requests.get(
            f'https://api.dcepay.dev/api/deposits/{deposit_id}',
            headers={
                'Authorization': f'Bearer {os.getenv("DCE_API_KEY")}',
                'Content-Type': 'application/json'
            }
        )

        assert final_status_response.status_code == 200
        # Status verification depends on test environment behavior
```

## Webhook Testing

### Local Webhook Testing

**Using ngrok for local testing:**

```bash
# Install ngrok
npm install -g ngrok

# Start your local server
npm start

# In another terminal, expose your local server
ngrok http 3000
```

**Webhook endpoint test:**
```javascript
// Express.js webhook endpoint
app.post('/webhook/dce', express.json(), (req, res) => {
  const signature = req.headers['signature'];
  const webhookSecret = process.env.DCE_WEBHOOK_SECRET;
  
  // Verify webhook signature
  const isValid = verifyWebhookSignature(req.body, signature, webhookSecret);
  
  if (!isValid) {
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  console.log('Webhook received:', req.body);
  
  // Process the webhook
  const { event, data } = req.body;
  
  switch (event) {
    case 'deposit.confirmed':
      handleDepositConfirmed(data);
      break;
    case 'withdrawal.completed':
      handleWithdrawalCompleted(data);
      break;
    default:
      console.log('Unknown event type:', event);
  }
  
  res.status(200).json({ status: 'received' });
});

function verifyWebhookSignature(payload, signature, secret) {
  const crypto = require('crypto');
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(JSON.stringify(payload))
    .digest('hex');
  
  return signature === `sha256=${expectedSignature}`;
}
```

**Python webhook endpoint:**
```python
from flask import Flask, request, jsonify
import hmac
import hashlib
import json
import os

app = Flask(__name__)

@app.route('/webhook/dce', methods=['POST'])
def webhook_handler():
    # Get webhook signature
    signature = request.headers.get('signature')
    webhook_secret = os.getenv('DCE_WEBHOOK_SECRET')
    
    # Verify webhook signature
    if not verify_webhook_signature(request.data, signature, webhook_secret):
        return jsonify({'error': 'Invalid signature'}), 401
    
    # Process webhook
    data = request.json
    event = data.get('event')
    
    print(f'Webhook received: {data}')
    
    if event == 'deposit.confirmed':
        handle_deposit_confirmed(data)
    elif event == 'withdrawal.completed':
        handle_withdrawal_completed(data)
    else:
        print(f'Unknown event type: {event}')
    
    return jsonify({'status': 'received'}), 200

def verify_webhook_signature(payload, signature, secret):
    expected_signature = hmac.new(
        secret.encode('utf-8'),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    return signature == f'sha256={expected_signature}'

def handle_deposit_confirmed(data):
    print(f'Deposit confirmed: {data}')
    # Implement your deposit confirmation logic

def handle_withdrawal_completed(data):
    print(f'Withdrawal completed: {data}')
    # Implement your withdrawal completion logic

if __name__ == '__main__':
    app.run(debug=True, port=3000)
```

### Webhook Testing Tools

**cURL webhook simulation:**
```bash
# Simulate deposit confirmed webhook
curl -X POST "https://your-domain.com/webhook/dce" \
  -H "Content-Type: application/json" \
  -H "Signature: sha256=abc123..." \
  -d '{
    "event": "deposit.confirmed",
    "txHash": "tx_1234567890",
    "toAddress": "0x1234567890123456789012345678901234567890",
    "fromAddress": "0x742d35Cc6634C0532925a3b8D4C9db96C4b4d8b6",
    "amount": "0.08",
    "coinSymbol": "ETH",
    "tokenSymbol": "ETH",
    "confirmedAt": "2024-12-19T10:30:00Z"
  }'
```

**JavaScript webhook testing:**
```javascript
const crypto = require('crypto');

function simulateWebhook(event, data, webhookSecret) {
  const payload = JSON.stringify(data);
  const signature = crypto
    .createHmac('sha256', webhookSecret)
    .update(payload)
    .digest('hex');

  return fetch('https://your-domain.com/webhook/dce', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Signature': `sha256=${signature}`
    },
    body: payload
  });
}

// Test deposit confirmed webhook
const depositData = {
  event: 'deposit.confirmed',
  txHash: 'tx_1234567890',
  toAddress: '0x1234567890123456789012345678901234567890',
  fromAddress: '0x742d35Cc6634C0532925a3b8D4C9db96C4b4d8b6',
  amount: '0.08',
  coinSymbol: 'ETH',
  tokenSymbol: 'ETH',
  confirmedAt: new Date().toISOString()
};

simulateWebhook('deposit.confirmed', depositData, process.env.DCE_WEBHOOK_SECRET)
  .then(response => response.json())
  .then(data => console.log('Webhook response:', data))
  .catch(error => console.error('Webhook error:', error));
```

## Load Testing

### API Endpoint Load Testing

**Using Artillery (JavaScript):**
```javascript
// artillery-config.yml
config:
  target: 'https://api.dcepay.dev'
  phases:
    - duration: 60
      arrivalRate: 10
  headers:
    Authorization: 'Bearer v8_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    Content-Type: 'application/json'

scenarios:
  - name: 'Balance API Load Test'
    requests:
      - get:
          url: '/api/balance?currency=USD'
          expect:
            - statusCode: 200

  - name: 'Deposit API Load Test'
    requests:
      - post:
          url: '/api/deposit-address'
          json:
            token: 'ETH'
            amount: 0.1
            metadata:
              orderId: 'load_test_' + Math.floor(Math.random() * 1000000)
          expect:
            - statusCode: 201
```

**Using Locust (Python):**
```python
from locust import HttpUser, task, between
import json

class V8APIUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        self.headers = {
            'Authorization': 'Bearer v8_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx',
            'Content-Type': 'application/json'
        }
    
    @task(3)
    def get_balance(self):
        self.client.get(
            '/api/balance?currency=USD',
            headers=self.headers
        )
    
    @task(1)
    def create_deposit(self):
        deposit_data = {
            'token': 'ETH',
            'amount': 0.1,
            'metadata': {
                'orderId': f'load_test_{self.client.environment.runner.user_count}'
            }
        }
        
        self.client.post(
            '/api/deposit-address',
            json=deposit_data,
            headers=self.headers
        )
```

### Webhook Load Testing

**JavaScript webhook load test:**
```javascript
const crypto = require('crypto');

async function loadTestWebhooks(count = 100) {
  const webhookSecret = process.env.DCE_WEBHOOK_SECRET;
  const webhookUrl = 'https://your-domain.com/webhook/dce';
  
  const promises = [];
  
  for (let i = 0; i < count; i++) {
    const webhookData = {
      event: 'deposit.confirmed',
      txHash: `tx_${Date.now()}_${i}`,
      toAddress: '0x1234567890123456789012345678901234567890',
      fromAddress: '0x742d35Cc6634C0532925a3b8D4C9db96C4b4d8b6',
      amount: '0.08',
      coinSymbol: 'ETH',
      tokenSymbol: 'ETH',
      confirmedAt: new Date().toISOString()
    };
    
    const payload = JSON.stringify(webhookData);
    const signature = crypto
      .createHmac('sha256', webhookSecret)
      .update(payload)
      .digest('hex');
    
    promises.push(
      fetch(webhookUrl, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Signature': `sha256=${signature}`
        },
        body: payload
      })
    );
  }
  
  const results = await Promise.allSettled(promises);
  
  const successful = results.filter(r => r.status === 'fulfilled' && r.value.ok).length;
  const failed = results.length - successful;
  
  console.log(`Webhook load test results:`);
  console.log(`- Total requests: ${results.length}`);
  console.log(`- Successful: ${successful}`);
  console.log(`- Failed: ${failed}`);
  console.log(`- Success rate: ${(successful / results.length * 100).toFixed(2)}%`);
}

loadTestWebhooks(100);
```

## Security Testing

### Authentication Testing

**Test invalid API keys:**
```javascript
describe('Security Tests', () => {
  test('should reject expired API key', async () => {
    try {
      await axios.get('https://api.dcepay.dev/api/balance', {
        headers: {
          'Authorization': 'Bearer expired_key_123',
          'Content-Type': 'application/json'
        }
      });
      fail('Should have thrown authentication error');
    } catch (error) {
      expect(error.response.status).toBe(401);
    }
  });

  test('should reject missing API key', async () => {
    try {
      await axios.get('https://api.dcepay.dev/api/balance', {
        headers: {
          'Content-Type': 'application/json'
        }
      });
      fail('Should have thrown authentication error');
    } catch (error) {
      expect(error.response.status).toBe(401);
    }
  });
});
```

### Webhook Security Testing

**Test webhook signature validation:**
```javascript
describe('Webhook Security', () => {
  test('should reject webhook with invalid signature', async () => {
    const webhookData = {
      event: 'deposit.confirmed',
      data: { amount: '0.1' }
    };

    try {
      await axios.post('https://your-domain.com/webhook/dce', webhookData, {
        headers: {
          'Content-Type': 'application/json',
          'Signature': 'invalid_signature'
        }
      });
      fail('Should have rejected invalid signature');
    } catch (error) {
      expect(error.response.status).toBe(401);
    }
  });

  test('should accept webhook with valid signature', async () => {
    const webhookData = {
      event: 'deposit.confirmed',
      data: { amount: '0.1' }
    };

    const signature = generateValidSignature(webhookData, process.env.DCE_WEBHOOK_SECRET);

    const response = await axios.post('https://your-domain.com/webhook/dce', webhookData, {
      headers: {
        'Content-Type': 'application/json',
        'Signature': signature
      }
    });

    expect(response.status).toBe(200);
  });
});
```

## Error Testing

### Rate Limiting Tests

**JavaScript:**
```javascript
describe('Rate Limiting', () => {
  test('should handle rate limit exceeded', async () => {
    const requests = [];
    
    // Make many requests quickly to trigger rate limiting
    for (let i = 0; i < 150; i++) {
      requests.push(
        axios.get('https://api.dcepay.dev/api/balance', {
          headers: {
            'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
            'Content-Type': 'application/json'
          }
        })
      );
    }
    
    const results = await Promise.allSettled(requests);
    const rateLimited = results.filter(r => 
      r.status === 'rejected' && 
      r.reason.response?.status === 429
    ).length;
    
    expect(rateLimited).toBeGreaterThan(0);
  });
});
```

### Network Error Testing

**JavaScript:**
```javascript
describe('Network Error Handling', () => {
  test('should handle network timeouts', async () => {
    try {
      await axios.get('https://api.dcepay.dev/api/balance', {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
          'Content-Type': 'application/json'
        },
        timeout: 1 // 1ms timeout to force timeout error
      });
      fail('Should have timed out');
    } catch (error) {
      expect(error.code).toBe('ECONNABORTED');
    }
  });

  test('should handle server errors gracefully', async () => {
    try {
      await axios.get('https://api.dcepay.dev/api/nonexistent-endpoint', {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
          'Content-Type': 'application/json'
        }
      });
      fail('Should have returned 404');
    } catch (error) {
      expect(error.response.status).toBe(404);
    }
  });
});
```

## Test Automation

### CI/CD Integration

**GitHub Actions workflow:**
```yaml
name: API Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run unit tests
      run: npm test
      env:
        DCE_API_KEY: ${{ secrets.DCE_TEST_API_KEY }}
        DCE_WEBHOOK_SECRET: ${{ secrets.DCE_WEBHOOK_SECRET }}
    
    - name: Run integration tests
      run: npm run test:integration
      env:
        DCE_API_KEY: ${{ secrets.DCE_TEST_API_KEY }}
        DCE_WEBHOOK_SECRET: ${{ secrets.DCE_WEBHOOK_SECRET }}
    
    - name: Run security tests
      run: npm run test:security
      env:
        DCE_API_KEY: ${{ secrets.DCE_TEST_API_KEY }}
        DCE_WEBHOOK_SECRET: ${{ secrets.DCE_WEBHOOK_SECRET }}
```

**package.json test scripts:**
```json
{
  "scripts": {
    "test": "jest",
    "test:integration": "jest --config jest.integration.config.js",
    "test:security": "jest --config jest.security.config.js",
    "test:load": "artillery run artillery-config.yml",
    "test:webhook": "node scripts/test-webhooks.js"
  }
}
```

## Testing Best Practices

### 1. Test Data Management

- Use unique test data for each test
- Clean up test data after tests complete
- Use descriptive test names and data

### 2. Error Handling

- Test both success and failure scenarios
- Verify error messages and status codes
- Test edge cases and boundary conditions

### 3. Performance Testing

- Test under realistic load conditions
- Monitor response times and throughput
- Test with various data sizes

### 4. Security Testing

- Test authentication and authorization
- Verify webhook signature validation
- Test input validation and sanitization

### 5. Monitoring and Logging

- Log all test results and errors
- Monitor API response times
- Track test coverage and success rates

## Environment-Specific Testing

### Staging Environment Testing

The staging environment provides a production-like environment for comprehensive testing before going live.

#### Staging Environment Setup

```bash
# Staging environment configuration
DCE_API_KEY=v8_staging_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_BASE_URL=https://api.dcepay.dev
DCE_WEBHOOK_SECRET=staging_webhook_secret_here
NODE_ENV=staging
```

#### Staging Test Strategy

**1. End-to-End Testing**
```javascript
describe('Staging Environment Tests', () => {
  test('should complete full payment flow in staging', async () => {
    // Create deposit with real blockchain interaction
    const depositResponse = await axios.post(
      'https://api.dcepay.dev/api/deposit-address',
      {
        token: 'ETH',
        amount: 0.01, // Small amount for testing
        metadata: {
          orderId: `staging_test_${Date.now()}`,
          customerEmail: 'staging-test@example.com'
        }
      },
      {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_STAGING_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    expect(depositResponse.status).toBe(201);
    const depositId = depositResponse.data.id;

    // Verify deposit creation
    const statusResponse = await axios.get(
      `https://api.dcepay.dev/api/deposits/${depositId}`,
      {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_STAGING_API_KEY}`,
          'Content-Type': 'application/json'
        }
      }
    );

    expect(statusResponse.status).toBe(200);
    expect(statusResponse.data.status).toBe('pending');
  });

  test('should handle staging webhook processing', async () => {
    // Test webhook processing with staging data
    const webhookData = {
      event: 'deposit.confirmed',
      txHash: `staging_tx_${Date.now()}`,
      toAddress: '0x1234567890123456789012345678901234567890',
      fromAddress: '0x742d35Cc6634C0532925a3b8D4C9db96C4b4d8b6',
      amount: '0.01',
      coinSymbol: 'ETH',
      tokenSymbol: 'ETH',
      confirmedAt: new Date().toISOString()
    };

    const response = await axios.post(
      'https://your-staging-domain.com/webhook/dce',
      webhookData,
      {
        headers: {
          'Content-Type': 'application/json',
          'Signature': generateSignature(webhookData, process.env.DCE_STAGING_WEBHOOK_SECRET)
        }
      }
    );

    expect(response.status).toBe(200);
  });
});
```

**2. Performance Testing in Staging**
```javascript
// Staging performance test configuration
const stagingLoadTest = {
  target: 'https://api.dcepay.dev',
  phases: [
    { duration: 300, arrivalRate: 5 }, // 5 minutes at 5 req/sec
    { duration: 300, arrivalRate: 10 }, // 5 minutes at 10 req/sec
    { duration: 300, arrivalRate: 20 }  // 5 minutes at 20 req/sec
  ],
  headers: {
    'Authorization': `Bearer ${process.env.DCE_STAGING_API_KEY}`,
    'Content-Type': 'application/json'
  }
};

// Run staging load test
artillery.run(stagingLoadTest, (err, results) => {
  if (err) {
    console.error('Staging load test failed:', err);
    return;
  }
  
  console.log('Staging load test results:');
  console.log(`- Total requests: ${results.summary.requests.total}`);
  console.log(`- Average response time: ${results.summary.responseTime.median}ms`);
  console.log(`- Error rate: ${(results.summary.errors / results.summary.requests.total * 100).toFixed(2)}%`);
});
```

**3. Database Integration Testing**
```javascript
describe('Staging Database Integration', () => {
  test('should persist webhook data correctly', async () => {
    // Create test webhook
    const webhookData = {
      event: 'deposit.confirmed',
      txHash: `db_test_${Date.now()}`,
      amount: '0.01',
      coinSymbol: 'ETH'
    };

    // Send webhook to staging
    await axios.post(
      'https://your-staging-domain.com/webhook/dce',
      webhookData,
      {
        headers: {
          'Content-Type': 'application/json',
          'Signature': generateSignature(webhookData, process.env.DCE_STAGING_WEBHOOK_SECRET)
        }
      }
    );

    // Verify data persistence
    const dbResult = await prisma.webhookLog.findFirst({
      where: {
        payload: {
          path: ['txHash'],
          equals: webhookData.txHash
        }
      }
    });

    expect(dbResult).toBeTruthy();
    expect(dbResult.payload.txHash).toBe(webhookData.txHash);
  });
});
```

### Production Environment Testing

Production testing focuses on monitoring, alerting, and ensuring system reliability under real-world conditions.

#### Production Monitoring Setup

**1. Health Check Monitoring**
```javascript
class ProductionHealthMonitor {
  constructor() {
    this.endpoints = [
      'https://api.dcepay.io/api/balance?currency=USD',
      'https://api.dcepay.io/api/balance',
      'https://api.dcepay.io/api/deposits'
    ];
    this.checkInterval = 60000; // 1 minute
  }

  async startMonitoring() {
    setInterval(async () => {
      for (const endpoint of this.endpoints) {
        await this.checkEndpoint(endpoint);
      }
    }, this.checkInterval);
  }

  async checkEndpoint(url) {
    try {
      const startTime = Date.now();
      const response = await axios.get(url, {
        headers: {
          'Authorization': `Bearer ${process.env.DCE_PRODUCTION_API_KEY}`,
          'Content-Type': 'application/json'
        },
        timeout: 10000 // 10 second timeout
      });

      const responseTime = Date.now() - startTime;

      // Log health check results
      console.log(`Health check: ${url} - ${response.status} (${responseTime}ms)`);

      // Alert if response time is too high
      if (responseTime > 5000) {
        await this.sendAlert('high_response_time', {
          endpoint: url,
          responseTime,
          threshold: 5000
        });
      }

      // Alert if status is not 200
      if (response.status !== 200) {
        await this.sendAlert('endpoint_error', {
          endpoint: url,
          status: response.status
        });
      }
    } catch (error) {
      console.error(`Health check failed for ${url}:`, error.message);
      await this.sendAlert('endpoint_unavailable', {
        endpoint: url,
        error: error.message
      });
    }
  }

  async sendAlert(type, data) {
    // Send alert to monitoring service
    await axios.post(process.env.ALERT_WEBHOOK_URL, {
      type,
      data,
      timestamp: new Date().toISOString()
    });
  }
}

// Start production monitoring
const healthMonitor = new ProductionHealthMonitor();
healthMonitor.startMonitoring();
```

**2. Error Rate Monitoring**
```javascript
class ErrorRateMonitor {
  constructor() {
    this.errorCounts = new Map();
    this.totalRequests = 0;
    this.checkInterval = 300000; // 5 minutes
  }

  recordRequest(endpoint, status) {
    this.totalRequests++;
    
    if (status >= 400) {
      const currentCount = this.errorCounts.get(endpoint) || 0;
      this.errorCounts.set(endpoint, currentCount + 1);
    }
  }

  async checkErrorRates() {
    const errorRate = this.totalRequests > 0 ? 
      Array.from(this.errorCounts.values()).reduce((a, b) => a + b, 0) / this.totalRequests : 0;

    console.log(`Error rate: ${(errorRate * 100).toFixed(2)}%`);

    if (errorRate > 0.05) { // 5% error rate threshold
      await this.sendAlert('high_error_rate', {
        errorRate,
        threshold: 0.05,
        totalRequests: this.totalRequests,
        errorCounts: Object.fromEntries(this.errorCounts)
      });
    }

    // Reset counters
    this.errorCounts.clear();
    this.totalRequests = 0;
  }

  async sendAlert(type, data) {
    await axios.post(process.env.ALERT_WEBHOOK_URL, {
      type,
      data,
      timestamp: new Date().toISOString()
    });
  }
}

// Start error rate monitoring
const errorMonitor = new ErrorRateMonitor();
setInterval(() => errorMonitor.checkErrorRates(), errorMonitor.checkInterval);
```

**3. Webhook Delivery Monitoring**
```javascript
class WebhookDeliveryMonitor {
  constructor() {
    this.webhookLogs = [];
    this.alertThreshold = 300000; // 5 minutes
  }

  recordWebhook(webhookData) {
    this.webhookLogs.push({
      ...webhookData,
      receivedAt: new Date().toISOString()
    });

    // Clean up old logs (older than 1 hour)
    const oneHourAgo = new Date(Date.now() - 3600000);
    this.webhookLogs = this.webhookLogs.filter(log => 
      new Date(log.receivedAt) > oneHourAgo
    );
  }

  async checkWebhookDelivery() {
    const now = new Date();
    const fiveMinutesAgo = new Date(now.getTime() - this.alertThreshold);

    // Check for webhooks received in the last 5 minutes
    const recentWebhooks = this.webhookLogs.filter(log =>
      new Date(log.receivedAt) > fiveMinutesAgo
    );

    if (recentWebhooks.length === 0) {
      await this.sendAlert('no_webhooks_received', {
        timeWindow: '5 minutes',
        lastWebhook: this.webhookLogs[this.webhookLogs.length - 1]
      });
    }

    // Check for failed webhook processing
    const failedWebhooks = recentWebhooks.filter(webhook =>
      webhook.status === 'failed' || webhook.error
    );

    if (failedWebhooks.length > 0) {
      await this.sendAlert('webhook_processing_failed', {
        failedCount: failedWebhooks.length,
        totalCount: recentWebhooks.length,
        failedWebhooks
      });
    }
  }

  async sendAlert(type, data) {
    await axios.post(process.env.ALERT_WEBHOOK_URL, {
      type,
      data,
      timestamp: new Date().toISOString()
    });
  }
}

// Start webhook monitoring
const webhookMonitor = new WebhookDeliveryMonitor();
setInterval(() => webhookMonitor.checkWebhookDelivery(), 60000); // Check every minute
```

#### Production Alerting System

**1. Alert Configuration**
```javascript
class ProductionAlerting {
  constructor() {
    this.alertChannels = {
      slack: process.env.SLACK_WEBHOOK_URL,
      email: process.env.ALERT_EMAIL,
      pagerduty: process.env.PAGERDUTY_API_KEY
    };
    this.alertThresholds = {
      errorRate: 0.05, // 5%
      responseTime: 5000, // 5 seconds
      webhookDelay: 300000, // 5 minutes
      consecutiveFailures: 3
    };
  }

  async sendAlert(severity, message, data = {}) {
    const alert = {
      severity,
      message,
      data,
      timestamp: new Date().toISOString(),
      environment: 'production'
    };

    // Send to all configured channels
    await Promise.all([
      this.sendToSlack(alert),
      this.sendToEmail(alert),
      this.sendToPagerDuty(alert)
    ]);
  }

  async sendToSlack(alert) {
    if (!this.alertChannels.slack) return;

    const color = this.getSeverityColor(alert.severity);
    const message = {
      attachments: [{
        color,
        title: `🚨 Production Alert: ${alert.severity.toUpperCase()}`,
        text: alert.message,
        fields: Object.entries(alert.data).map(([key, value]) => ({
          title: key,
          value: typeof value === 'object' ? JSON.stringify(value) : String(value),
          short: true
        })),
        footer: `Environment: ${alert.environment} | Time: ${alert.timestamp}`
      }]
    };

    await axios.post(this.alertChannels.slack, message);
  }

  async sendToEmail(alert) {
    if (!this.alertChannels.email) return;

    // Email implementation using nodemailer
    const transporter = nodemailer.createTransporter({
      host: process.env.SMTP_HOST,
      port: process.env.SMTP_PORT,
      secure: true,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS
      }
    });

    await transporter.sendMail({
      from: process.env.SMTP_USER,
      to: this.alertChannels.email,
      subject: `Production Alert: ${alert.severity.toUpperCase()}`,
      html: this.generateEmailTemplate(alert)
    });
  }

  async sendToPagerDuty(alert) {
    if (!this.alertChannels.pagerduty) return;

    const incident = {
      incident: {
        type: 'incident',
        title: `API Alert: ${alert.severity.toUpperCase()}`,
        service: {
          id: process.env.PAGERDUTY_SERVICE_ID,
          type: 'service_reference'
        },
        urgency: alert.severity === 'critical' ? 'high' : 'low',
        body: {
          type: 'incident_body',
          details: alert.message
        }
      }
    };

    await axios.post('https://api.pagerduty.com/incidents', incident, {
      headers: {
        'Authorization': `Token token=${this.alertChannels.pagerduty}`,
        'Content-Type': 'application/json'
      }
    });
  }

  getSeverityColor(severity) {
    switch (severity) {
      case 'critical': return '#ff0000';
      case 'warning': return '#ffa500';
      case 'info': return '#0000ff';
      default: return '#cccccc';
    }
  }

  generateEmailTemplate(alert) {
    return `
      <h2>Production Alert: ${alert.severity.toUpperCase()}</h2>
      <p><strong>Message:</strong> ${alert.message}</p>
      <p><strong>Time:</strong> ${alert.timestamp}</p>
      <p><strong>Environment:</strong> ${alert.environment}</p>
      ${Object.entries(alert.data).map(([key, value]) => 
        `<p><strong>${key}:</strong> ${typeof value === 'object' ? JSON.stringify(value) : value}</p>`
      ).join('')}
    `;
  }
}

// Initialize alerting system
const alerting = new ProductionAlerting();
```

**2. Automated Response Actions**
```javascript
class AutomatedResponse {
  constructor() {
    this.alerting = new ProductionAlerting();
  }

  async handleHighErrorRate(data) {
    // Log the issue
    console.error('High error rate detected:', data);

    // Send immediate alert
    await this.alerting.sendAlert('critical', 'High error rate detected', data);

    // Attempt automatic recovery
    await this.attemptRecovery('error_rate', data);
  }

  async handleHighResponseTime(data) {
    // Log the issue
    console.warn('High response time detected:', data);

    // Send warning alert
    await this.alerting.sendAlert('warning', 'High response time detected', data);

    // Check if it's a temporary spike
    if (data.responseTime > 10000) { // 10 seconds
      await this.alerting.sendAlert('critical', 'Critical response time issue', data);
    }
  }

  async handleWebhookFailure(data) {
    // Log the issue
    console.error('Webhook processing failed:', data);

    // Send alert
    await this.alerting.sendAlert('warning', 'Webhook processing failed', data);

    // Attempt to retry failed webhooks
    await this.retryFailedWebhooks(data.failedWebhooks);
  }

  async attemptRecovery(type, data) {
    switch (type) {
      case 'error_rate':
        // Check if it's a specific endpoint issue
        if (data.errorCounts && Object.keys(data.errorCounts).length === 1) {
          const endpoint = Object.keys(data.errorCounts)[0];
          console.log(`Attempting to restart service for endpoint: ${endpoint}`);
          // Implement service restart logic
        }
        break;
      
      case 'webhook_failure':
        // Check webhook endpoint health
        await this.checkWebhookEndpointHealth();
        break;
    }
  }

  async retryFailedWebhooks(failedWebhooks) {
    for (const webhook of failedWebhooks) {
      try {
        await this.processWebhook(webhook);
        console.log(`Successfully retried webhook: ${webhook.txHash}`);
      } catch (error) {
        console.error(`Failed to retry webhook: ${webhook.txHash}`, error);
      }
    }
  }

  async checkWebhookEndpointHealth() {
    try {
      const response = await axios.get('/api/webhook/health', {
        timeout: 5000
      });
      
      if (response.status !== 200) {
        console.error('Webhook endpoint health check failed');
        await this.alerting.sendAlert('critical', 'Webhook endpoint unhealthy');
      }
    } catch (error) {
      console.error('Webhook endpoint health check failed:', error);
      await this.alerting.sendAlert('critical', 'Webhook endpoint unavailable');
    }
  }
}

// Initialize automated response system
const automatedResponse = new AutomatedResponse();
```

#### Production Testing Checklist

**Before Going Live:**

- [ ] **Environment Configuration**
  - [ ] Production API keys configured
  - [ ] Webhook endpoints deployed and tested
  - [ ] Database connections verified
  - [ ] SSL certificates installed

- [ ] **Monitoring Setup**
  - [ ] Health checks configured
  - [ ] Error rate monitoring active
  - [ ] Response time monitoring active
  - [ ] Webhook delivery monitoring active

- [ ] **Alerting Configuration**
  - [ ] Slack notifications configured
  - [ ] Email alerts configured
  - [ ] PagerDuty integration active
  - [ ] Alert thresholds set appropriately

- [ ] **Load Testing**
  - [ ] Production-like load testing completed
  - [ ] Performance benchmarks established
  - [ ] Scalability verified

- [ ] **Security Testing**
  - [ ] Authentication tests passed
  - [ ] Authorization tests passed
  - [ ] Webhook signature validation verified
  - [ ] Input validation tested

- [ ] **Backup and Recovery**
  - [ ] Database backups configured
  - [ ] Recovery procedures documented
  - [ ] Rollback procedures tested

**Ongoing Production Monitoring:**

- [ ] **Daily Checks**
  - [ ] Review error rates and response times
  - [ ] Check webhook delivery success rates
  - [ ] Monitor system resource usage
  - [ ] Review alert history

- [ ] **Weekly Checks**
  - [ ] Analyze performance trends
  - [ ] Review security logs
  - [ ] Update monitoring thresholds
  - [ ] Test alerting systems

- [ ] **Monthly Checks**
  - [ ] Review and update test coverage
  - [ ] Analyze user feedback and issues
  - [ ] Update documentation
  - [ ] Plan capacity improvements

## Testing Tools

### Recommended Testing Tools

| Tool | Purpose | Language |
|------|---------|----------|
| Jest | Unit and integration testing | JavaScript |
| pytest | Unit and integration testing | Python |
| Artillery | Load testing | JavaScript |
| Locust | Load testing | Python |
| ngrok | Local webhook testing | All |
| Postman | API testing and documentation | All |

### Test Environment Setup

```bash
# Install testing dependencies
npm install --save-dev jest axios supertest
pip install pytest requests pytest-asyncio

# Set up test environment
cp .env.example .env.test
# Edit .env.test with test credentials

# Run tests
npm test
pytest tests/
```

## Next Steps

After completing your testing:

1. **Review Test Results** - Analyze test coverage and identify gaps
2. **Fix Issues** - Address any failing tests or performance problems
3. **Deploy to Staging** - Test in a staging environment before production
4. **Monitor Production** - Set up monitoring and alerting for production

For more information, see:

- [Error Handling](https://docs.dcepay.io/docs/error-handling) - Handle API errors effectively
- [Webhooks](https://docs.dcepay.io/docs/webhooks) - Set up webhook processing
- [API Reference](https://docs.dcepay.io/docs/api-reference) - Complete endpoint documentation

---

*For testing support, contact api-support@dce.com or visit our [developer portal](https://developers.dce.com).* 
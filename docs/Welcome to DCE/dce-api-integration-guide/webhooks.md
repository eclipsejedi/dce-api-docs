---
title: Webhooks
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

The webhooks API allows you to receive real-time notifications about payment events, transaction status changes, and system updates. This guide covers webhook setup, event handling, signature verification, and best practices for reliable webhook processing.

## How the system delivers webhooks

There are three different paths; only the first one is the merchant integration surface.

### 1. Outbound webhooks to your URL (merchant integration)

1. You configure `webhookUrl`, `webhookSecret`, `webhookEnabled`, and optionally `webhookEvents`, `webhookTimeout`, and `webhookRetryCount` on the merchant profile (for example via user/merchant APIs).
2. When a deposit, withdrawal, or underlying transaction changes state, `WebhookService` may queue a delivery: it creates a `webhookLog` row, then **POST**s JSON to your `webhookUrl`.
3. The HTTP request includes:
   - `Content-Type: application/json`
   - `X-Webhook-Event` — the event name (for example `deposit.confirmed`, `withdrawal.failed`)
   - `X-Webhook-Signature` — lowercase hex **HMAC-SHA256** of the **exact raw body bytes**, keyed by `webhookSecret` (same string as `JSON.stringify(payload)` on the server). **Verify using the raw body**, not a re-serialized `JSON.stringify` of a parsed object, so key order cannot break verification.
4. **Subscription filter:** if `webhookEvents` is a **non-empty** string array, only listed events are sent. If the field is missing or not parseable as a string array, there is no filter. An **empty** array means nothing is sent.
5. **Retries:** failed HTTP status, timeouts, or network errors schedule retries with exponential backoff (capped at 30s) up to `webhookRetryCount` (default **3**). If the response body is JSON and contains `"ok": true`, retries are not scheduled for that attempt. Operators can run the `webhook-retries` background job to process due retries.

**Event names in code** (not `payout.*`): `deposit.confirmed` | `deposit.failed` | `deposit.pending`; `withdrawal.confirmed` | `withdrawal.failed` | `withdrawal.pending`; `transaction.confirmed` | `transaction.failed`.

### 2. Internal ingestion — `POST /api/webhook/event`

This endpoint is **admin-only** (`admin:write`). It accepts callbacks from trusted upstream systems using `WEBHOOK_SECRET`, `x-signature` (hex HMAC-SHA256 of the JSON body after **recursive alphabetical key sorting**), and `x-timestamp` (Unix ms, ±5 minutes). It is **not** the URL you publish as a merchant. Successful processing updates core records via `WebhookHandler` and may trigger **outbound** deliveries in (1).

### 3. Chain-provider callbacks

Additional routes under `/api/webhook/*` (for example chain adapters) handle provider-specific payloads and signatures. They are internal plumbing, not the merchant webhook contract.

## Overview

**Outbound webhooks (merchant integration):** DCE POSTs JSON to your configured `webhookUrl` when deposits, withdrawals, or ledger transactions change state. The **canonical** description (headers, `X-Webhook-Signature`, retries, subscription filters) is in **How the system delivers webhooks** at the top of this page.

**Inbound operator route:** `POST /api/webhook/event` is for **admin/system ingestion** only (signed with `WEBHOOK_SECRET`). It is **not** the URL you publish as a merchant.

**Settlement notifications:** Outbound webhook event names in code are `deposit.*`, `withdrawal.*`, and `transaction.*`. Do not assume `settlement.*` events are emitted on the same merchant webhook channel unless your account team explicitly confirms it—use the Settlements API for settlement state when in doubt.

## Webhook setup (merchant)

Configure `webhookUrl`, `webhookSecret`, `webhookEnabled`, and optionally `webhookEvents`, `webhookTimeout`, and `webhookRetryCount` on the **merchant profile** (via user/merchant APIs).

Your HTTPS endpoint should:

1. Accept **POST** requests with `Content-Type: application/json`
2. Return `2xx` after the event is safely accepted (move heavy work async if needed)
3. Verify `X-Webhook-Signature` (hex HMAC-SHA256 of the **raw body** with `webhookSecret`)
4. Implement **idempotency** (retries are expected on failures/timeouts)

## Outbound event names (current code)

| Event type              | Description                  |
| ----------------------- | ---------------------------- |
| `deposit.confirmed`     | Deposit confirmed            |
| `deposit.failed`        | Deposit failed               |
| `deposit.pending`       | Deposit pending              |
| `withdrawal.confirmed`  | Withdrawal confirmed         |
| `withdrawal.failed`     | Withdrawal failed            |
| `withdrawal.pending`    | Withdrawal pending           |
| `transaction.confirmed` | Ledger transaction confirmed |
| `transaction.failed`    | Ledger transaction failed    |

Payloads include an `event` field plus chain and business fields (see repository `src/types/webhook.ts` for internal TypeScript shapes). Do not rely on a generic `{ data, metadata }` envelope unless you normalize it yourself—integrate against `event` and the documented fields.

## Internal: `POST /api/webhook/event`

**Purpose:** ingest upstream/system events for processing by `WebhookHandler` (operator tooling; **not** the merchant callback URL).

**Auth:** API key with `admin:write`, plus server `WEBHOOK_SECRET`.

**Headers:**

- `Content-Type: application/json`
- `x-signature` — hex HMAC-SHA256 of the canonical JSON (parsed then **recursively key-sorted**—see server verification)
- `x-timestamp` — Unix time in milliseconds (±5 minutes)

**Success response (typical):**

```json
{
  "success": true,
  "message": "Webhook processed successfully"
}
```

## Signature Verification

### Merchant outbound deliveries (`X-Webhook-Signature`)

DCE signs the **exact raw JSON body bytes** sent in the webhook POST using your `webhookSecret`. The signature is **lowercase hex** HMAC-SHA256 and is sent in the `X-Webhook-Event` / `X-Webhook-Signature` headers (see **How the system delivers webhooks**).

**Important:** verify the signature against the **raw HTTP body string** captured before JSON parsing. Re-stringifying `JSON.parse` output can break verification due to key ordering differences.

**JavaScript (Express + raw body):**

```javascript
const crypto = require('crypto');

function verifyDceSignature(rawBodyString, signatureHeader, secret) {
  const expected = crypto.createHmac('sha256', secret).update(rawBodyString, 'utf8').digest('hex');
  return crypto.timingSafeEqual(Buffer.from(signatureHeader, 'hex'), Buffer.from(expected, 'hex'));
}

// Capture raw JSON body, then parse JSON only after verification
app.post(
  '/webhooks/dce',
  express.raw({ type: 'application/json' }),
  (req, res) => {
    const raw = req.body.toString('utf8');
    const signature = req.headers['x-webhook-signature'];
    if (!signature || !verifyDceSignature(raw, signature, process.env.DCE_WEBHOOK_SECRET)) {
      return res.status(401).json({ error: 'Invalid signature' });
    }
    const event = JSON.parse(raw);
    processWebhook(event);
    return res.status(200).json({ ok: true });
  }
);
```

**Python:**

```python
import hmac
import hashlib
import json

def verify_webhook_signature(payload, signature, secret):
    expected_signature = hmac.new(
        secret.encode('utf-8'),
        payload.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(signature, expected_signature)

# Flask webhook handler
@app.route('/webhooks/dce', methods=['POST'])
def webhook_handler():
    signature = request.headers.get('Signature')
    payload = request.get_data(as_text=True)
    
    if not verify_webhook_signature(payload, signature, os.environ['WEBHOOK_SECRET']):
        return jsonify({'error': 'Invalid signature'}), 401
    
    process_webhook(request.json)
    return jsonify({'status': 'received'})
```

**PHP:**

```php
<?php
function verifyWebhookSignature($payload, $signature, $secret) {
    $expectedSignature = hash_hmac('sha256', $payload, $secret);
    return hash_equals($signature, $expectedSignature);
}

// Slim/Laravel webhook handler
$app->post('/webhooks/dce', function (Request $request, Response $response) {
    $signature = $request->getHeaderLine('signature');
    $payload = $request->getBody()->getContents();
    
    if (!verifyWebhookSignature($payload, $signature, $_ENV['WEBHOOK_SECRET'])) {
        $response->getBody()->write(json_encode(['error' => 'Invalid signature']));
        return $response->withStatus(401);
    }
    
    // Process webhook
    $event = json_decode($payload, true);
    processWebhook($event);
    
    $response->getBody()->write(json_encode(['status' => 'received']));
    return $response;
});
?>
```

**Java:**

```java
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.security.MessageDigest;

public class WebhookVerifier {
    public static boolean verifyWebhookSignature(String payload, String signature, String secret) {
        try {
            Mac sha256Hmac = Mac.getInstance("HmacSHA256");
            SecretKeySpec secretKey = new SecretKeySpec(secret.getBytes(), "HmacSHA256");
            sha256Hmac.init(secretKey);
            
            byte[] hash = sha256Hmac.doFinal(payload.getBytes());
            StringBuilder expectedSignature = new StringBuilder();
            for (byte b : hash) {
                expectedSignature.append(String.format("%02x", b));
            }
            
            return signature.equals(expectedSignature.toString());
        } catch (Exception e) {
            return false;
        }
    }
}

// Spring Boot webhook handler
@PostMapping("/webhooks/dce")
public ResponseEntity<Map<String, String>> webhook(
        @RequestBody Map<String, Object> event,
        @RequestHeader("signature") String signature,
        @RequestBody String rawPayload) {
    
    if (!WebhookVerifier.verifyWebhookSignature(rawPayload, signature, System.getenv("WEBHOOK_SECRET"))) {
        return ResponseEntity.status(401)
            .body(Map.of("error", "Invalid signature"));
    }
    
    // Process webhook
    processWebhook(event);
    
    return ResponseEntity.ok(Map.of("status", "received"));
}
```

**C#:**

```csharp
using System.Security.Cryptography;
using System.Text;

public class WebhookVerifier
{
    public static bool VerifyWebhookSignature(string payload, string signature, string secret)
    {
        using (var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret)))
        {
            var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(payload));
            var expectedSignature = Convert.ToHexString(hash).ToLower();
            return signature.Equals(expectedSignature, StringComparison.OrdinalIgnoreCase);
        }
    }
}

// ASP.NET Core webhook handler
[HttpPost("dce")]
public IActionResult Webhook()
{
    var signature = Request.Headers["signature"].FirstOrDefault();
    var payload = Request.Body.ToString();
    
    if (!WebhookVerifier.VerifyWebhookSignature(payload, signature, 
        Environment.GetEnvironmentVariable("WEBHOOK_SECRET")))
    {
        return Unauthorized(new { error = "Invalid signature" });
    }
    
    // Process webhook
    var eventData = JsonConvert.DeserializeObject<dynamic>(payload);
    ProcessWebhook(eventData);
    
    return Ok(new { status = "received" });
}
```

## Event Processing

### Fee Charges in Webhook Payloads

For `deposit.confirmed` and `withdrawal.confirmed` events, the webhook payload includes a `feeCharges` object that provides detailed information about the fees charged to the merchant:

```json
{
  "feeCharges": {
    "amount": "0.004",
    "percentage": "0.05", 
    "type": "PERCENTAGE"
  },
  "receivableAmount": "0.076"
}
```

#### Fee Charges Object Properties

| Property     | Type   | Description                                                    |
| ------------ | ------ | -------------------------------------------------------------- |
| `amount`     | string | The actual fee amount charged (in the transaction currency)    |
| `percentage` | string | The percentage rate used for calculation (e.g., "0.05" for 5%) |
| `type`       | string | The charge type: `PERCENTAGE`, `FIXED_AMOUNT`, or `HYBRID`     |

#### Additional Fields

| Property           | Type   | Description                                                             |
| ------------------ | ------ | ----------------------------------------------------------------------- |
| `receivableAmount` | string | The final amount after fee deduction (transaction amount - fee charges) |

#### Charge Types

- **PERCENTAGE**: Fee calculated as a percentage of the transaction amount
- **FIXED\_AMOUNT**: Fixed fee amount regardless of transaction size
- **HYBRID**: Combination of percentage and fixed amount

#### Example Fee Calculations

**Percentage-based (5%):**

```json
{
  "amount": "100.00",
  "feeCharges": {
    "amount": "5.00",
    "percentage": "0.05",
    "type": "PERCENTAGE"
  },
  "receivableAmount": "95.00"
}
```

**Fixed amount ($2.50):**

```json
{
  "amount": "100.00", 
  "feeCharges": {
    "amount": "2.50",
    "percentage": "0.00",
    "type": "FIXED_AMOUNT"
  },
  "receivableAmount": "97.50"
}
```

**Hybrid (2% + $1.00):**

```json
{
  "amount": "100.00",
  "feeCharges": {
    "amount": "3.00",
    "percentage": "0.02", 
    "type": "HYBRID"
  },
  "receivableAmount": "97.00"
}
```

### Deposit Events

#### deposit.confirmed

```json
{
  "event": "deposit.confirmed",
  "txHash": "tx_1752630134805_ti0a34wjs",
  "toAddress": "0x1234567890123456789012345678901234567890",
  "fromAddress": "0x742d35Cc6634C0532925a3b8D4C9db96C4b4d8b6",
  "amount": "0.08",
  "coinSymbol": "ETH",
  "tokenSymbol": "ETH",
  "confirmedAt": "2024-12-19T10:30:00Z",
  "layer": "L1Transaction",
  "internalFee": {
    "deposit": "0.002"
  },
  "feeCharges": {
    "amount": "0.004",
    "percentage": "0.05",
    "type": "PERCENTAGE"
  },
  "receivableAmount": "0.076",
  "receiverInfo": {
    "identity": "AS188689e48494c8a452683587138f209d673aada204cb23393140e7f40280e0c5"
  },
  "identifier": "user123",
  "depositRequest": {
    "exchangeRate": "7.182",
    "requestedValue": {
      "amount": "1000",
      "currency": "USD"
    }
  }
}
```

#### deposit.failed

```json
{
  "event": "deposit.failed",
  "txHash": "tx_1752630135962_4vp6wefk2",
  "toAddress": "0x1234567890123456789012345678901234567890",
  "fromAddress": "0x742d35Cc6634C0532925a3b8D4C9db96C4b4d8b6",
  "amount": "0.1",
  "coinSymbol": "ETH",
  "tokenSymbol": "ETH",
  "failedAt": "2024-12-19T10:30:00Z",
  "layer": "L1Transaction",
  "reason": "Invalid destination address"
}
```

### Payout Events

#### payout.confirmed

```json
{
  "event": "payout.confirmed",
  "txHash": "tx_1752630134805_ti0a34wjs",
  "toAddress": "0x1234567890123456789012345678901234567890",
  "fromAddress": "0x742d35Cc6634C0532925a3b8D4C9db96C4b4d8b6",
  "amount": "0.08",
  "coinSymbol": "ETH",
  "tokenSymbol": "ETH",
  "confirmedAt": "2024-12-19T10:30:00Z",
  "layer": "L1Transaction",
  "internalFee": {
    "withdraw": "0.002"
  },
  "feeCharges": {
    "amount": "0.004",
    "percentage": "0.05",
    "type": "PERCENTAGE"
  },
  "receivableAmount": "0.076"
}
```

#### payout.failed

```json
{
  "event": "payout.failed",
  "txHash": "tx_1752630135962_4vp6wefk2",
  "toAddress": "0x1234567890123456789012345678901234567890",
  "fromAddress": "0x742d35Cc6634C0532925a3b8D4C9db96C4b4d8b6",
  "amount": "0.1",
  "coinSymbol": "ETH",
  "tokenSymbol": "ETH",
  "failedAt": "2024-12-19T10:30:00Z",
  "layer": "L1Transaction",
  "reason": "Invalid destination address"
}
```

## Webhook Processing

### Idempotency

Implement idempotency to prevent duplicate processing:

```javascript
class WebhookProcessor {
  constructor() {
    this.processedEvents = new Set();
  }
  
  async processWebhook(event) {
    const eventId = `${event.txHash}-${event.event}`;
    
    // Check if already processed
    if (this.processedEvents.has(eventId)) {
      console.log(`Event already processed: ${eventId}`);
      return { status: 'already_processed' };
    }
    
    // Process the event
    await this.handleEvent(event);
    
    // Mark as processed
    this.processedEvents.add(eventId);
    
    return { status: 'processed' };
  }
  
  async handleEvent(event) {
    switch (event.event) {
      case 'deposit.confirmed':
        await this.handleDepositConfirmed(event);
        break;
      case 'deposit.failed':
        await this.handleDepositFailed(event);
        break;
      case 'payout.confirmed':
        await this.handlePayoutConfirmed(event);
        break;
      case 'payout.failed':
        await this.handlePayoutFailed(event);
        break;
      default:
        console.log(`Unknown event type: ${event.event}`);
    }
  }
}
```

### Database Integration

```javascript
class WebhookDatabase {
  async logWebhookEvent(event, status) {
    return await prisma.webhookLog.create({
      data: {
        eventType: event.event,
        payload: event,
        status: status,
        merchantId: event.metadata?.merchantId || 'system',
        transactionId: event.metadata?.transactionId
      }
    });
  }
  
  async updateTransactionStatus(transactionId, status, metadata = {}) {
    return await prisma.transaction.update({
      where: { id: transactionId },
      data: {
        status,
        updatedAt: new Date(),
        metadata: {
          ...metadata,
          webhookProcessedAt: new Date().toISOString()
        }
      }
    });
  }
}
```

## Error Handling

### Retry Logic

Implement exponential backoff for failed webhook processing:

```javascript
class WebhookRetryHandler {
  async processWithRetry(event, maxRetries = 3) {
    let attempt = 0;
    
    while (attempt < maxRetries) {
      try {
        return await this.processWebhook(event);
      } catch (error) {
        attempt++;
        
        if (attempt >= maxRetries) {
          console.error(`Webhook processing failed after ${maxRetries} attempts:`, error);
          throw error;
        }
        
        // Exponential backoff
        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
}
```

### Error Response Handling

```javascript
app.post('/webhooks/dce', async (req, res) => {
  try {
    // Verify signature
    if (!verifySignature(req)) {
      return res.status(401).json({ error: 'Invalid signature' });
    }
    
    // Process webhook
    await webhookProcessor.processWebhook(req.body);
    
    res.json({ status: 'received' });
  } catch (error) {
    console.error('Webhook processing error:', error);
    
    // Return 200 to prevent retries for processing errors
    res.status(200).json({ 
      status: 'error',
      error: error.message 
    });
  }
});
```

## Webhook Retry Strategies and Failure Handling

### Understanding Webhook Delivery Failures

Webhook delivery can fail for various reasons:

- **Network Issues**: Temporary connectivity problems, DNS resolution failures
- **Server Errors**: 5xx HTTP status codes from your endpoint
- **Timeout Issues**: Slow processing causing connection timeouts
- **Rate Limiting**: Too many requests hitting your endpoint
- **Authentication Failures**: Invalid signatures or missing headers
- **Payload Issues**: Malformed JSON or oversized payloads

### Exponential Backoff Implementation

#### 1. Basic Exponential Backoff

```javascript
class ExponentialBackoffRetry {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 5;
    this.baseDelay = options.baseDelay || 1000; // 1 second
    this.maxDelay = options.maxDelay || 30000; // 30 seconds
    this.jitter = options.jitter || 0.1; // 10% jitter
  }

  async execute(operation) {
    let lastError;
    
    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        
        if (attempt === this.maxRetries) {
          throw new Error(`Operation failed after ${this.maxRetries} attempts: ${error.message}`);
        }
        
        const delay = this.calculateDelay(attempt);
        console.log(`Attempt ${attempt + 1} failed, retrying in ${delay}ms`);
        
        await this.sleep(delay);
      }
    }
    
    throw lastError;
  }

  calculateDelay(attempt) {
    // Exponential backoff with jitter
    const exponentialDelay = Math.min(
      this.baseDelay * Math.pow(2, attempt),
      this.maxDelay
    );
    
    const jitterAmount = exponentialDelay * this.jitter;
    const jitter = (Math.random() - 0.5) * jitterAmount;
    
    return Math.floor(exponentialDelay + jitter);
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const retryHandler = new ExponentialBackoffRetry({
  maxRetries: 5,
  baseDelay: 1000,
  maxDelay: 30000
});

try {
  const result = await retryHandler.execute(async () => {
    return await processWebhook(event);
  });
  console.log('Webhook processed successfully');
} catch (error) {
  console.error('Webhook processing failed:', error);
}
```

#### 2. Advanced Retry with Circuit Breaker

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.recoveryTimeout = options.recoveryTimeout || 60000; // 1 minute
    this.failures = 0;
    this.lastFailureTime = null;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
  }

  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.recoveryTimeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failures++;
    this.lastFailureTime = Date.now();
    
    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
    }
  }
}

class AdvancedWebhookRetry {
  constructor() {
    this.retryHandler = new ExponentialBackoffRetry();
    this.circuitBreaker = new CircuitBreaker();
  }

  async processWebhook(event) {
    return await this.circuitBreaker.execute(async () => {
      return await this.retryHandler.execute(async () => {
        return await this.actualWebhookProcessing(event);
      });
    });
  }

  async actualWebhookProcessing(event) {
    // Your actual webhook processing logic
    console.log('Processing webhook:', event.event);
    // ... processing logic
  }
}
```

### Dead Letter Queue Implementation

#### 1. Database-Based Dead Letter Queue

```javascript
class DeadLetterQueue {
  constructor(prisma) {
    this.prisma = prisma;
  }

  async addToDeadLetter(event, error, attemptCount) {
    return await this.prisma.deadLetterQueue.create({
      data: {
        eventType: event.event,
        payload: event,
        error: error.message,
        stackTrace: error.stack,
        attemptCount,
        correlationId: event.correlationId,
        createdAt: new Date(),
        status: 'pending'
      }
    });
  }

  async processDeadLetterQueue() {
    const failedEvents = await this.prisma.deadLetterQueue.findMany({
      where: {
        status: 'pending',
        createdAt: {
          gte: new Date(Date.now() - 24 * 60 * 60 * 1000) // Last 24 hours
        }
      },
      orderBy: {
        createdAt: 'asc'
      }
    });

    for (const failedEvent of failedEvents) {
      try {
        await this.retryFailedEvent(failedEvent);
        
        // Mark as processed
        await this.prisma.deadLetterQueue.update({
          where: { id: failedEvent.id },
          data: { status: 'processed' }
        });
      } catch (error) {
        console.error('Failed to process dead letter:', error);
        
        // Mark as permanently failed after max attempts
        if (failedEvent.attemptCount >= 10) {
          await this.prisma.deadLetterQueue.update({
            where: { id: failedEvent.id },
            data: { status: 'permanently_failed' }
          });
        }
      }
    }
  }

  async retryFailedEvent(failedEvent) {
    const retryHandler = new ExponentialBackoffRetry({
      maxRetries: 3,
      baseDelay: 5000 // 5 seconds
    });

    return await retryHandler.execute(async () => {
      return await this.processWebhook(failedEvent.payload);
    });
  }
}
```

#### 2. Redis-Based Dead Letter Queue

```javascript
class RedisDeadLetterQueue {
  constructor(redis) {
    this.redis = redis;
    this.queueKey = 'webhook:dead_letter_queue';
    this.processingKey = 'webhook:processing_queue';
  }

  async addToDeadLetter(event, error, attemptCount) {
    const deadLetterItem = {
      id: `dlq_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      event,
      error: error.message,
      attemptCount,
      timestamp: Date.now(),
      correlationId: event.correlationId
    };

    await this.redis.lpush(this.queueKey, JSON.stringify(deadLetterItem));
    
    // Set TTL for automatic cleanup (7 days)
    await this.redis.expire(this.queueKey, 7 * 24 * 60 * 60);
    
    console.log(`Added to dead letter queue: ${deadLetterItem.id}`);
  }

  async processDeadLetterQueue() {
    while (true) {
      const item = await this.redis.brpop(this.queueKey, 1);
      
      if (!item) continue;
      
      const deadLetterItem = JSON.parse(item[1]);
      
      try {
        await this.retryFailedEvent(deadLetterItem);
        console.log(`Successfully processed dead letter: ${deadLetterItem.id}`);
      } catch (error) {
        console.error(`Failed to process dead letter ${deadLetterItem.id}:`, error);
        
        // Re-add to queue if under max attempts
        if (deadLetterItem.attemptCount < 10) {
          deadLetterItem.attemptCount++;
          await this.redis.lpush(this.queueKey, JSON.stringify(deadLetterItem));
        } else {
          console.log(`Permanently failed dead letter: ${deadLetterItem.id}`);
        }
      }
    }
  }

  async retryFailedEvent(deadLetterItem) {
    const retryHandler = new ExponentialBackoffRetry({
      maxRetries: 3,
      baseDelay: 5000
    });

    return await retryHandler.execute(async () => {
      return await this.processWebhook(deadLetterItem.event);
    });
  }
}
```

### Comprehensive Webhook Handler with Retry and DLQ

```javascript
class ComprehensiveWebhookHandler {
  constructor(options = {}) {
    this.retryHandler = new ExponentialBackoffRetry(options.retry);
    this.deadLetterQueue = new DeadLetterQueue(options.prisma);
    this.metrics = {
      totalEvents: 0,
      successfulEvents: 0,
      failedEvents: 0,
      deadLetterEvents: 0
    };
  }

  async handleWebhook(event) {
    this.metrics.totalEvents++;
    
    try {
      // Attempt to process with retry
      const result = await this.retryHandler.execute(async () => {
        return await this.processWebhook(event);
      });
      
      this.metrics.successfulEvents++;
      return result;
      
    } catch (error) {
      this.metrics.failedEvents++;
      
      // Add to dead letter queue
      await this.deadLetterQueue.addToDeadLetter(
        event, 
        error, 
        this.retryHandler.maxRetries
      );
      
      this.metrics.deadLetterEvents++;
      
      // Log for monitoring
      await this.logFailure(event, error);
      
      throw error;
    }
  }

  async processWebhook(event) {
    // Your webhook processing logic
    console.log(`Processing webhook: ${event.event}`);
    
    // Simulate processing time
    await new Promise(resolve => setTimeout(resolve, 100));
    
    // Simulate potential failure
    if (Math.random() < 0.1) { // 10% failure rate for testing
      throw new Error('Simulated processing failure');
    }
    
    return { status: 'processed', event: event.event };
  }

  async logFailure(event, error) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      event: event.event,
      correlationId: event.correlationId,
      error: error.message,
      stackTrace: error.stack
    };
    
    // Log to monitoring system
    console.error('Webhook failure:', logEntry);
    
    // Send to monitoring service
    await this.sendToMonitoring(logEntry);
  }

  async sendToMonitoring(logEntry) {
    // Send to your monitoring service (DataDog, New Relic, etc.)
    console.log('Sending to monitoring:', logEntry);
  }

  getMetrics() {
    return {
      ...this.metrics,
      successRate: this.metrics.totalEvents > 0 
        ? (this.metrics.successfulEvents / this.metrics.totalEvents) * 100 
        : 0
    };
  }
}

// Usage
const handler = new ComprehensiveWebhookHandler({
  retry: {
    maxRetries: 5,
    baseDelay: 1000,
    maxDelay: 30000
  },
  prisma: prisma
});

// Process webhook
try {
  const result = await handler.handleWebhook(webhookEvent);
  console.log('Webhook processed:', result);
} catch (error) {
  console.error('Webhook failed:', error);
}

// Monitor metrics
setInterval(() => {
  const metrics = handler.getMetrics();
  console.log('Webhook metrics:', metrics);
}, 60000); // Log metrics every minute
```

### Monitoring and Alerting

#### 1. Webhook Health Monitoring

```javascript
class WebhookHealthMonitor {
  constructor() {
    this.alerts = [];
    this.thresholds = {
      failureRate: 0.05, // 5% failure rate
      avgProcessingTime: 5000, // 5 seconds
      deadLetterQueueSize: 100
    };
  }

  async checkHealth(metrics) {
    const alerts = [];
    
    // Check failure rate
    if (metrics.successRate < (1 - this.thresholds.failureRate) * 100) {
      alerts.push({
        severity: 'critical',
        message: `High webhook failure rate: ${(100 - metrics.successRate).toFixed(2)}%`,
        metric: 'failure_rate',
        value: 100 - metrics.successRate
      });
    }
    
    // Check dead letter queue size
    if (metrics.deadLetterEvents > this.thresholds.deadLetterQueueSize) {
      alerts.push({
        severity: 'warning',
        message: `Large dead letter queue: ${metrics.deadLetterEvents} events`,
        metric: 'dead_letter_queue_size',
        value: metrics.deadLetterEvents
      });
    }
    
    return alerts;
  }

  async sendAlert(alert) {
    // Send to your alerting system (PagerDuty, Slack, etc.)
    console.log(`ALERT [${alert.severity.toUpperCase()}]: ${alert.message}`);
    
    // Store alert
    this.alerts.push({
      ...alert,
      timestamp: new Date().toISOString()
    });
  }
}
```

#### 2. Real-time Metrics Dashboard

```javascript
class WebhookMetricsDashboard {
  constructor() {
    this.metrics = {
      hourly: new Map(),
      daily: new Map()
    };
  }

  recordEvent(event, processingTime, success) {
    const hour = new Date().toISOString().slice(0, 13) + ':00:00.000Z';
    const day = new Date().toISOString().slice(0, 10);
    
    // Update hourly metrics
    if (!this.metrics.hourly.has(hour)) {
      this.metrics.hourly.set(hour, {
        total: 0,
        successful: 0,
        failed: 0,
        avgProcessingTime: 0,
        events: []
      });
    }
    
    const hourlyMetric = this.metrics.hourly.get(hour);
    hourlyMetric.total++;
    hourlyMetric.avgProcessingTime = 
      (hourlyMetric.avgProcessingTime + processingTime) / 2;
    
    if (success) {
      hourlyMetric.successful++;
    } else {
      hourlyMetric.failed++;
    }
    
    hourlyMetric.events.push({
      event: event.event,
      processingTime,
      success,
      timestamp: new Date().toISOString()
    });
    
    // Keep only last 100 events per hour
    if (hourlyMetric.events.length > 100) {
      hourlyMetric.events = hourlyMetric.events.slice(-100);
    }
  }

  getMetrics(timeframe = 'hourly') {
    const metrics = timeframe === 'hourly' ? this.metrics.hourly : this.metrics.daily;
    const result = [];
    
    for (const [timestamp, data] of metrics) {
      result.push({
        timestamp,
        total: data.total,
        successful: data.successful,
        failed: data.failed,
        successRate: data.total > 0 ? (data.successful / data.total) * 100 : 0,
        avgProcessingTime: data.avgProcessingTime
      });
    }
    
    return result.sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));
  }
}
```

### Best Practices for Webhook Reliability

#### 1. Idempotency Implementation

```javascript
class IdempotentWebhookProcessor {
  constructor(prisma) {
    this.prisma = prisma;
  }

  async processWebhook(event) {
    const eventId = this.generateEventId(event);
    
    // Check if already processed
    const existing = await this.prisma.webhookLog.findFirst({
      where: { eventId }
    });
    
    if (existing) {
      console.log(`Event already processed: ${eventId}`);
      return { status: 'already_processed', eventId };
    }
    
    // Process webhook
    const result = await this.actualProcessing(event);
    
    // Log successful processing
    await this.prisma.webhookLog.create({
      data: {
        eventId,
        eventType: event.event,
        payload: event,
        status: 'processed',
        processedAt: new Date()
      }
    });
    
    return { status: 'processed', eventId, result };
  }

  generateEventId(event) {
    // Create unique ID based on event characteristics
    const components = [
      event.event,
      event.txHash || event.correlationId,
      event.timestamp
    ];
    
    return require('crypto')
      .createHash('sha256')
      .update(components.join('|'))
      .digest('hex');
  }
}
```

#### 2. Rate Limiting and Throttling

```javascript
class WebhookRateLimiter {
  constructor(options = {}) {
    this.maxRequestsPerSecond = options.maxRequestsPerSecond || 10;
    this.maxRequestsPerMinute = options.maxRequestsPerMinute || 100;
    this.requests = [];
  }

  async throttle(operation) {
    await this.waitForRateLimit();
    
    this.requests.push(Date.now());
    
    // Clean old requests
    this.requests = this.requests.filter(
      time => Date.now() - time < 60000 // Keep last minute
    );
    
    return await operation();
  }

  async waitForRateLimit() {
    const now = Date.now();
    const requestsLastSecond = this.requests.filter(
      time => now - time < 1000
    ).length;
    
    const requestsLastMinute = this.requests.length;
    
    if (requestsLastSecond >= this.maxRequestsPerSecond) {
      const delay = 1000 - (now - this.requests[0]);
      if (delay > 0) {
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
    
    if (requestsLastMinute >= this.maxRequestsPerMinute) {
      const delay = 60000 - (now - this.requests[0]);
      if (delay > 0) {
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
}
```

### Configuration Examples

#### 1. Environment-Specific Configuration

```javascript
// config/webhook-config.js
const webhookConfig = {
  development: {
    retry: {
      maxRetries: 3,
      baseDelay: 1000,
      maxDelay: 10000
    },
    deadLetterQueue: {
      maxRetries: 5,
      cleanupAfterDays: 1
    },
    monitoring: {
      enabled: false
    }
  },
  
  staging: {
    retry: {
      maxRetries: 5,
      baseDelay: 2000,
      maxDelay: 30000
    },
    deadLetterQueue: {
      maxRetries: 10,
      cleanupAfterDays: 3
    },
    monitoring: {
      enabled: true,
      alertThreshold: 0.1 // 10% failure rate
    }
  },
  
  production: {
    retry: {
      maxRetries: 7,
      baseDelay: 3000,
      maxDelay: 60000
    },
    deadLetterQueue: {
      maxRetries: 15,
      cleanupAfterDays: 7
    },
    monitoring: {
      enabled: true,
      alertThreshold: 0.05 // 5% failure rate
    }
  }
};

module.exports = webhookConfig[process.env.NODE_ENV || 'development'];
```

#### 2. Docker Compose for Webhook Testing

```yaml
# docker-compose.webhook-test.yml
version: '3.8'

services:
  webhook-server:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - WEBHOOK_SECRET=test_secret
      - DATABASE_URL=postgresql://user:pass@db:5432/webhook_test
    depends_on:
      - db
      - redis

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: webhook_test
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"

  webhook-tester:
    build: .
    command: npm run test:webhooks
    environment:
      - WEBHOOK_URL=http://webhook-server:3000/webhooks/dce
      - WEBHOOK_SECRET=test_secret
    depends_on:
      - webhook-server

volumes:
  postgres_data:
```

This comprehensive webhook retry and failure handling system provides:

- **Exponential backoff** with jitter for retry attempts
- **Circuit breaker pattern** to prevent cascading failures
- **Dead letter queue** for permanently failed events
- **Comprehensive monitoring** and alerting
- **Idempotency** to prevent duplicate processing
- **Rate limiting** to prevent overwhelming your endpoint
- **Environment-specific configuration** for different deployment stages

The system ensures your webhook processing is robust, reliable, and maintainable across all environments.

## Testing Webhooks

### Local Testing

Use ngrok to test webhooks locally:

```bash
# Install ngrok
npm install -g ngrok

# Start your local server
npm start

# Expose local server
ngrok http 3000

# Use the ngrok URL as your webhook endpoint
# https://abc123.ngrok.io/webhooks/dce
```

### Webhook Testing Tools

```javascript
class WebhookTester {
  async testWebhookEndpoint(url, event) {
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Signature': this.generateSignature(event)
      },
      body: JSON.stringify(event)
    });
    
    return {
      status: response.status,
      body: await response.json()
    };
  }
  
  generateSignature(payload) {
    const crypto = require('crypto');
    return crypto
      .createHmac('sha256', process.env.WEBHOOK_SECRET)
      .update(JSON.stringify(payload), 'utf8')
      .digest('hex');
  }
  
  createTestEvent(type, data = {}) {
    return {
      event: type,
      timestamp: new Date().toISOString(),
      correlationId: `test-${Date.now()}`,
      ...data
    };
  }
}

// Usage
const tester = new WebhookTester();
const testEvent = tester.createTestEvent('deposit.confirmed', {
  txHash: 'test_tx_hash',
  amount: '0.1',
  toAddress: '0x1234567890123456789012345678901234567890'
});

const result = await tester.testWebhookEndpoint('https://your-domain.com/webhooks/dce', testEvent);
console.log(result);
```

## Best Practices

### 1. Security

- **Always verify signatures** before processing webhooks
- **Use HTTPS** for webhook endpoints
- **Implement rate limiting** to prevent abuse
- **Validate payload structure** before processing

### 2. Reliability

```javascript
class ReliableWebhookHandler {
  constructor() {
    this.queue = [];
    this.processing = false;
  }
  
  async addToQueue(event) {
    this.queue.push(event);
    
    if (!this.processing) {
      this.processing = true;
      await this.processQueue();
    }
  }
  
  async processQueue() {
    while (this.queue.length > 0) {
      const event = this.queue.shift();
      
      try {
        await this.processWebhook(event);
      } catch (error) {
        console.error('Failed to process webhook:', error);
        // Add back to queue for retry
        this.queue.unshift(event);
        await new Promise(resolve => setTimeout(resolve, 5000));
      }
    }
    
    this.processing = false;
  }
}
```

### 3. Monitoring

```javascript
class WebhookMonitor {
  constructor() {
    this.metrics = {
      totalEvents: 0,
      successfulEvents: 0,
      failedEvents: 0,
      averageProcessingTime: 0
    };
  }
  
  async trackEvent(event, processingTime) {
    this.metrics.totalEvents++;
    this.metrics.averageProcessingTime = 
      (this.metrics.averageProcessingTime + processingTime) / 2;
    
    // Log to monitoring service
    await this.logToMonitoring({
      event: event.event,
      processingTime,
      timestamp: new Date().toISOString()
    });
  }
  
  async logToMonitoring(data) {
    // Send to monitoring service (e.g., DataDog, New Relic)
    console.log('Webhook metric:', data);
  }
}
```

### 4. Logging

```javascript
class WebhookLogger {
  async logWebhook(event, result) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      event: event.event,
      correlationId: event.correlationId,
      status: result.status,
      processingTime: result.processingTime,
      error: result.error
    };
    
    // Log to database
    await this.saveToDatabase(logEntry);
    
    // Log to external service
    await this.sendToLogService(logEntry);
  }
}
```

## Integration Examples

### Express.js Webhook Handler

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();
app.use(express.json());

class WebhookHandler {
  constructor() {
    this.processor = new WebhookProcessor();
    this.logger = new WebhookLogger();
  }
  
  async handleWebhook(req, res) {
    const startTime = Date.now();
    
    try {
      // Verify signature
      if (!this.verifySignature(req)) {
        return res.status(401).json({ error: 'Invalid signature' });
      }
      
      // Process webhook
      const result = await this.processor.processWebhook(req.body);
      
      // Log success
      await this.logger.logWebhook(req.body, {
        status: 'success',
        processingTime: Date.now() - startTime
      });
      
      res.json({ status: 'received' });
    } catch (error) {
      // Log error
      await this.logger.logWebhook(req.body, {
        status: 'error',
        processingTime: Date.now() - startTime,
        error: error.message
      });
      
      res.status(200).json({ 
        status: 'error',
        error: error.message 
      });
    }
  }
  
  verifySignature(req) {
    const signature = req.headers['signature'];
    const payload = JSON.stringify(req.body);
    
    const expectedSignature = crypto
      .createHmac('sha256', process.env.WEBHOOK_SECRET)
      .update(payload, 'utf8')
      .digest('hex');
    
    return crypto.timingSafeEqual(
      Buffer.from(signature, 'hex'),
      Buffer.from(expectedSignature, 'hex')
    );
  }
}

const handler = new WebhookHandler();
app.post('/webhooks/dce', (req, res) => handler.handleWebhook(req, res));
```

### Python Flask Webhook Handler

```python
from flask import Flask, request, jsonify
import hmac
import hashlib
import json
import os
from datetime import datetime

app = Flask(__name__)

class WebhookHandler:
    def __init__(self):
        self.secret = os.environ['WEBHOOK_SECRET']
    
    def verify_signature(self, payload, signature):
        expected_signature = hmac.new(
            self.secret.encode('utf-8'),
            payload.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()
        
        return hmac.compare_digest(signature, expected_signature)
    
    def process_webhook(self, event):
        event_type = event.get('event')
        
        if event_type == 'deposit.confirmed':
            return self.handle_deposit_confirmed(event)
        elif event_type == 'payout.confirmed':
            return self.handle_payout_confirmed(event)
        else:
            print(f"Unknown event type: {event_type}")
            return {'status': 'ignored'}
    
    def handle_deposit_confirmed(self, event):
        # Update user balance
        # Update transaction status
        # Send notification
        return {'status': 'processed'}
    
    def handle_payout_confirmed(self, event):
        # Update withdrawal status
        # Send confirmation email
        return {'status': 'processed'}

handler = WebhookHandler()

@app.route('/webhooks/dce', methods=['POST'])
def webhook_endpoint():
    try:
        # Verify signature
        signature = request.headers.get('Signature')
        payload = request.get_data(as_text=True)
        
        if not handler.verify_signature(payload, signature):
            return jsonify({'error': 'Invalid signature'}), 401
        
        # Process webhook
        event = request.json
        result = handler.process_webhook(event)
        
        return jsonify({'status': 'received'})
    except Exception as e:
        print(f"Webhook processing error: {e}")
        return jsonify({'status': 'error', 'error': str(e)}), 200

if __name__ == '__main__':
    app.run(debug=True)
```

### Node.js (Express):

```javascript
const express = require('express');
const crypto = require('crypto');
require('dotenv').config();

const app = express();
app.use(express.json());

function verifyWebhookSignature(payload, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload, 'utf8')
    .digest('hex');
  
  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expectedSignature, 'hex')
  );
}

function processWebhook(event) {
  console.log('Processing webhook:', event.event);
  
  switch (event.event) {
    case 'deposit.confirmed':
      console.log('Deposit confirmed:', event.txHash);
      // Update database, send confirmation email, etc.
      break;
    case 'deposit.failed':
      console.log('Deposit failed:', event.reason);
      // Handle failed deposit
      break;
    case 'payout.confirmed':
      console.log('Payout confirmed:', event.txHash);
      // Update withdrawal status
      break;
    case 'payout.failed':
      console.log('Payout failed:', event.reason);
      // Handle failed payout
      break;
    default:
      console.log('Unknown event type:', event.event);
  }
}

app.post('/webhooks/dce', (req, res) => {
  try {
    const signature = req.headers['signature'];
    const payload = JSON.stringify(req.body);
    
    if (!verifyWebhookSignature(payload, signature, process.env.WEBHOOK_SECRET)) {
      return res.status(401).json({ error: 'Invalid signature' });
    }
    
    processWebhook(req.body);
    res.json({ status: 'received' });
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.listen(3000, () => {
  console.log('Webhook server running on port 3000');
});
```

### Python (Flask):

```python
from flask import Flask, request, jsonify
import hmac
import hashlib
import os
from dotenv import load_dotenv

load_dotenv()
app = Flask(__name__)

def verify_webhook_signature(payload, signature, secret):
    expected_signature = hmac.new(
        secret.encode('utf-8'),
        payload.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(signature, expected_signature)

def process_webhook(event):
    print(f"Processing webhook: {event['event']}")
    
    if event['event'] == 'deposit.confirmed':
        print(f"Deposit confirmed: {event['txHash']}")
        # Update database, send confirmation email, etc.
    elif event['event'] == 'deposit.failed':
        print(f"Deposit failed: {event['reason']}")
        # Handle failed deposit
    elif event['event'] == 'payout.confirmed':
        print(f"Payout confirmed: {event['txHash']}")
        # Update withdrawal status
    elif event['event'] == 'payout.failed':
        print(f"Payout failed: {event['reason']}")
        # Handle failed payout
    else:
        print(f"Unknown event type: {event['event']}")

@app.route('/webhooks/dce', methods=['POST'])
def webhook():
    try:
        signature = request.headers.get('signature')
        payload = request.get_data(as_text=True)
        
        if not verify_webhook_signature(payload, signature, os.getenv('WEBHOOK_SECRET')):
            return jsonify({'error': 'Invalid signature'}), 401
        
        process_webhook(request.json)
        return jsonify({'status': 'received'})
    except Exception as e:
        print('Webhook error:', e)
        return jsonify({'error': 'Internal server error'}), 500

if __name__ == '__main__':
    app.run(port=3000)
```

### PHP (Laravel/Slim):

```php
<?php
require 'vendor/autoload.php';

use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Slim\Factory\AppFactory;

$app = AppFactory::create();
$app->addBodyParsingMiddleware();

function verifyWebhookSignature($payload, $signature, $secret) {
    $expectedSignature = hash_hmac('sha256', $payload, $secret);
    return hash_equals($signature, $expectedSignature);
}

function processWebhook($event) {
    echo "Processing webhook: " . $event['event'] . "\n";
    
    switch ($event['event']) {
        case 'deposit.confirmed':
            echo "Deposit confirmed: " . $event['txHash'] . "\n";
            // Update database, send confirmation email, etc.
            break;
        case 'deposit.failed':
            echo "Deposit failed: " . $event['reason'] . "\n";
            // Handle failed deposit
            break;
        case 'payout.confirmed':
            echo "Payout confirmed: " . $event['txHash'] . "\n";
            // Update withdrawal status
            break;
        case 'payout.failed':
            echo "Payout failed: " . $event['reason'] . "\n";
            // Handle failed payout
            break;
        default:
            echo "Unknown event type: " . $event['event'] . "\n";
    }
}

$app->post('/webhooks/dce', function (Request $request, Response $response) {
    try {
        $signature = $request->getHeaderLine('signature');
        $payload = $request->getBody()->getContents();
        
        if (!verifyWebhookSignature($payload, $signature, $_ENV['WEBHOOK_SECRET'])) {
            $response->getBody()->write(json_encode(['error' => 'Invalid signature']));
            return $response->withStatus(401);
        }
        
        $event = json_decode($payload, true);
        processWebhook($event);
        
        $response->getBody()->write(json_encode(['status' => 'received']));
        return $response;
    } catch (Exception $e) {
        echo "Webhook error: " . $e->getMessage() . "\n";
        $response->getBody()->write(json_encode(['error' => 'Internal server error']));
        return $response->withStatus(500);
    }
});

$app->run();
?>
```

### Java (Spring Boot):

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.util.Map;

@SpringBootApplication
@RestController
public class WebhookController {
    
    private static final String WEBHOOK_SECRET = System.getenv("WEBHOOK_SECRET");
    
    private boolean verifyWebhookSignature(String payload, String signature) {
        try {
            Mac sha256Hmac = Mac.getInstance("HmacSHA256");
            SecretKeySpec secretKey = new SecretKeySpec(WEBHOOK_SECRET.getBytes(), "HmacSHA256");
            sha256Hmac.init(secretKey);
            
            byte[] hash = sha256Hmac.doFinal(payload.getBytes());
            StringBuilder expectedSignature = new StringBuilder();
            for (byte b : hash) {
                expectedSignature.append(String.format("%02x", b));
            }
            
            return signature.equals(expectedSignature.toString());
        } catch (Exception e) {
            return false;
        }
    }
    
    private void processWebhook(Map<String, Object> event) {
        String eventType = (String) event.get("event");
        System.out.println("Processing webhook: " + eventType);
        
        switch (eventType) {
            case "deposit.confirmed":
                System.out.println("Deposit confirmed: " + event.get("txHash"));
                // Update database, send confirmation email, etc.
                break;
            case "deposit.failed":
                System.out.println("Deposit failed: " + event.get("reason"));
                // Handle failed deposit
                break;
            case "payout.confirmed":
                System.out.println("Payout confirmed: " + event.get("txHash"));
                // Update withdrawal status
                break;
            case "payout.failed":
                System.out.println("Payout failed: " + event.get("reason"));
                // Handle failed payout
                break;
            default:
                System.out.println("Unknown event type: " + eventType);
        }
    }
    
    @PostMapping("/webhooks/dce")
    public ResponseEntity<Map<String, String>> webhook(
            @RequestBody Map<String, Object> event,
            @RequestHeader("signature") String signature,
            @RequestBody String rawPayload) {
        
        try {
            if (!verifyWebhookSignature(rawPayload, signature)) {
                return ResponseEntity.status(401)
                    .body(Map.of("error", "Invalid signature"));
            }
            
            processWebhook(event);
            return ResponseEntity.ok(Map.of("status", "received"));
        } catch (Exception e) {
            System.err.println("Webhook error: " + e.getMessage());
            return ResponseEntity.status(500)
                .body(Map.of("error", "Internal server error"));
        }
    }
    
    public static void main(String[] args) {
        SpringApplication.run(WebhookController.class, args);
    }
}
```

### C# (.NET):

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Security.Cryptography;
using System.Text;
using Newtonsoft.Json;

[ApiController]
[Route("[controller]")]
public class WebhookController : ControllerBase
{
    private static readonly string webhookSecret = Environment.GetEnvironmentVariable("WEBHOOK_SECRET");
    
    private bool VerifyWebhookSignature(string payload, string signature)
    {
        using (var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(webhookSecret)))
        {
            var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(payload));
            var expectedSignature = Convert.ToHexString(hash).ToLower();
            return signature.Equals(expectedSignature, StringComparison.OrdinalIgnoreCase);
        }
    }
    
    private void ProcessWebhook(dynamic eventData)
    {
        string eventType = eventData.event;
        Console.WriteLine($"Processing webhook: {eventType}");
        
        switch (eventType)
        {
            case "deposit.confirmed":
                Console.WriteLine($"Deposit confirmed: {eventData.txHash}");
                // Update database, send confirmation email, etc.
                break;
            case "deposit.failed":
                Console.WriteLine($"Deposit failed: {eventData.reason}");
                // Handle failed deposit
                break;
            case "payout.confirmed":
                Console.WriteLine($"Payout confirmed: {eventData.txHash}");
                // Update withdrawal status
                break;
            case "payout.failed":
                Console.WriteLine($"Payout failed: {eventData.reason}");
                // Handle failed payout
                break;
            default:
                Console.WriteLine($"Unknown event type: {eventType}");
                break;
        }
    }
    
    [HttpPost("dce")]
    public IActionResult Webhook()
    {
        try
        {
            var signature = Request.Headers["signature"].FirstOrDefault();
            var payload = Request.Body.ToString();
            
            if (!VerifyWebhookSignature(payload, signature))
            {
                return Unauthorized(new { error = "Invalid signature" });
            }
            
            var eventData = JsonConvert.DeserializeObject<dynamic>(payload);
            ProcessWebhook(eventData);
            
            return Ok(new { status = "received" });
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Webhook error: {ex.Message}");
            return StatusCode(500, new { error = "Internal server error" });
        }
    }
}
```

***

_For more information about specific event types, see the [Deposits](deposits.md) and [Withdrawals](withdrawals.md) documentation._

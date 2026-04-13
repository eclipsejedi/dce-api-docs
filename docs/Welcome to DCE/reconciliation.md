---
title: Reconciliation
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

The reconciliation API allows you to automatically match and reconcile transactions with your internal records. This guide covers reconciliation setup, processing, and best practices for maintaining accurate financial records.

## Overview

Reconciliation is the process of matching transactions from the dce API with your internal accounting records to ensure accuracy and identify discrepancies. The reconciliation system provides:

- **Automated Matching** - Match transactions based on amount, date, and reference
- **Discrepancy Detection** - Identify missing or mismatched transactions
- **Audit Trail** - Complete history of reconciliation activities
- **Reporting** - Generate reconciliation reports for compliance

## Creating Reconciliations

### POST /api/reconciliation

Create a new reconciliation record to match transactions.

#### Request

```bash
curl -X POST "${DCE_BASE_URL}/api/reconciliation" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "transactionId": "txn_xxxxxxxxxxxxxxxx",
    "internalReference": "INV-2024-001",
    "amount": "100.00",
    "currency": "USD",
    "transactionDate": "2024-12-19T10:30:00Z",
    "notes": "Customer payment for invoice INV-2024-001"
  }'
```

#### Request Parameters

| Parameter           | Type   | Required | Description                         |
| ------------------- | ------ | -------- | ----------------------------------- |
| `transactionId`     | string | Yes      | dce transaction ID to reconcile     |
| `internalReference` | string | Yes      | Your internal reference number      |
| `amount`            | string | Yes      | Transaction amount                  |
| `currency`          | string | Yes      | 3-letter currency code              |
| `transactionDate`   | string | Yes      | ISO 8601 date of transaction        |
| `notes`             | string | No       | Additional notes for reconciliation |

#### Response

```json
{
  "id": "rec_xxxxxxxxxxxxxxxx",
  "transactionId": "txn_xxxxxxxxxxxxxxxx",
  "internalReference": "INV-2024-001",
  "amount": "100.00",
  "currency": "USD",
  "status": "pending",
  "transactionDate": "2024-12-19T10:30:00Z",
  "notes": "Customer payment for invoice INV-2024-001",
  "createdAt": "2024-12-19T10:35:00Z",
  "updatedAt": "2024-12-19T10:35:00Z"
}
```

#### JavaScript Example

```javascript
const createReconciliation = async (reconciliationData) => {
  try {
    const response = await fetch(`${process.env.DCE_BASE_URL}/api/reconciliation`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(reconciliationData)
    });

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(`Reconciliation creation failed: ${errorData.error}`);
    }

    return await response.json();
  } catch (error) {
    console.error('Error creating reconciliation:', error);
    throw error;
  }
};

// Example usage
const reconciliationData = {
  transactionId: 'txn_xxxxxxxxxxxxxxxx',
  internalReference: 'INV-2024-001',
  amount: '100.00',
  currency: 'USD',
  transactionDate: '2024-12-19T10:30:00Z',
  notes: 'Customer payment for invoice INV-2024-001'
};

const reconciliation = await createReconciliation(reconciliationData);
console.log('Reconciliation created:', reconciliation);
```

#### Python Example

```python
import requests
import os
from datetime import datetime

def create_reconciliation(reconciliation_data):
    try:
        response = requests.post(
            f"{os.getenv('DCE_BASE_URL')}/api/reconciliation",
            headers={
                'Authorization': f"Bearer {os.getenv('DCE_API_KEY')}",
                'Content-Type': 'application/json'
            },
            json=reconciliation_data
        )
        
        response.raise_for_status()
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"Error creating reconciliation: {e}")
        raise

# Example usage
reconciliation_data = {
    'transactionId': 'txn_xxxxxxxxxxxxxxxx',
    'internalReference': 'INV-2024-001',
    'amount': '100.00',
    'currency': 'USD',
    'transactionDate': '2024-12-19T10:30:00Z',
    'notes': 'Customer payment for invoice INV-2024-001'
}

reconciliation = create_reconciliation(reconciliation_data)
print(f"Reconciliation created: {reconciliation}")
```

## Listing Reconciliations

### GET /api/reconciliation

Retrieve a list of reconciliation records with filtering and pagination.

#### Request

```bash
curl -X GET "${DCE_BASE_URL}/api/reconciliation?page=1&limit=10&status=pending" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json"
```

#### Query Parameters

| Parameter   | Type    | Required | Description                                    |
| ----------- | ------- | -------- | ---------------------------------------------- |
| `page`      | integer | No       | Page number (default: 1)                       |
| `limit`     | integer | No       | Items per page (default: 10, max: 100)         |
| `status`    | string  | No       | Filter by status (pending, matched, unmatched) |
| `currency`  | string  | No       | Filter by currency code                        |
| `startDate` | string  | No       | Filter by start date (ISO 8601)                |
| `endDate`   | string  | No       | Filter by end date (ISO 8601)                  |

#### Response

```json
{
  "data": [
    {
      "id": "rec_xxxxxxxxxxxxxxxx",
      "transactionId": "txn_xxxxxxxxxxxxxxxx",
      "internalReference": "INV-2024-001",
      "amount": "100.00",
      "currency": "USD",
      "status": "pending",
      "transactionDate": "2024-12-19T10:30:00Z",
      "notes": "Customer payment for invoice INV-2024-001",
      "createdAt": "2024-12-19T10:35:00Z",
      "updatedAt": "2024-12-19T10:35:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 1,
    "pages": 1
  }
}
```

## Updating Reconciliation Status

### PATCH /api/reconciliation

Update the status of a reconciliation record.

#### Request

```bash
curl -X PATCH "${DCE_BASE_URL}/api/reconciliation" \
  -H "Authorization: Bearer ${DCE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "rec_xxxxxxxxxxxxxxxx",
    "status": "matched",
    "notes": "Successfully matched with internal record"
  }'
```

#### Request Parameters

| Parameter | Type   | Required | Description                              |
| --------- | ------ | -------- | ---------------------------------------- |
| `id`      | string | Yes      | Reconciliation record ID                 |
| `status`  | string | Yes      | New status (pending, matched, unmatched) |
| `notes`   | string | No       | Additional notes for the update          |

## Reconciliation Status

### Status Values

| Status      | Description                                      |
| ----------- | ------------------------------------------------ |
| `pending`   | Reconciliation record created, awaiting matching |
| `matched`   | Successfully matched with internal records       |
| `unmatched` | Could not be matched, requires manual review     |

### Status Transitions

```text
pending → matched (automatic or manual)
pending → unmatched (automatic or manual)
matched → pending (manual review)
unmatched → matched (manual correction)
```

## Automated Reconciliation

### Background Processing

The reconciliation system automatically processes reconciliation records:

1. **Transaction Matching** - Match transactions based on amount, date, and reference
2. **Discrepancy Detection** - Identify transactions that don't match internal records
3. **Status Updates** - Update reconciliation status based on matching results
4. **Notification** - Send webhooks for reconciliation status changes

### Matching Criteria

The system matches transactions using:

- **Amount** - Exact amount match (with tolerance for fees)
- **Date** - Transaction date within specified range
- **Reference** - Internal reference number match
- **Currency** - Currency code match

## Error Handling

### Common Error Scenarios

| Error                           | Description                  | Solution                   |
| ------------------------------- | ---------------------------- | -------------------------- |
| `Transaction not found`         | Transaction ID doesn't exist | Verify transaction ID      |
| `Invalid reconciliation data`   | Missing required fields      | Check request parameters   |
| `Reconciliation already exists` | Duplicate reconciliation     | Check for existing records |
| `Invalid status`                | Invalid status value         | Use valid status values    |

### Error Response Format

```json
{
  "error": "Invalid reconciliation data",
  "details": [
    {
      "field": "amount",
      "message": "Amount must be positive",
      "value": -100
    }
  ]
}
```

## Best Practices

### 1. Regular Reconciliation

```javascript
// Schedule daily reconciliation
const scheduleReconciliation = async () => {
  const transactions = await getUnreconciledTransactions();
  
  for (const transaction of transactions) {
    await createReconciliation({
      transactionId: transaction.id,
      internalReference: transaction.reference,
      amount: transaction.amount,
      currency: transaction.currency,
      transactionDate: transaction.createdAt,
      notes: `Daily reconciliation for ${transaction.reference}`
    });
  }
};
```

### 2. Discrepancy Monitoring

```javascript
// Monitor unmatched reconciliations
const monitorDiscrepancies = async () => {
  const unmatched = await getReconciliations({ status: 'unmatched' });
  
  if (unmatched.length > 0) {
    console.warn(`Found ${unmatched.length} unmatched reconciliations`);
    
    // Send alert to accounting team
    await sendAlert({
      type: 'reconciliation_discrepancy',
      count: unmatched.length,
      details: unmatched
    });
  }
};
```

### 3. Audit Trail

```javascript
// Maintain reconciliation audit trail
const logReconciliationActivity = async (activity) => {
  await logActivity({
    type: 'reconciliation',
    action: activity.action,
    reconciliationId: activity.reconciliationId,
    userId: activity.userId,
    timestamp: new Date().toISOString(),
    details: activity.details
  });
};
```

## Integration Examples

### E-commerce Platform Integration

```javascript
// Integrate with e-commerce platform
const reconcileEcommercePayment = async (orderId, transactionId) => {
  try {
    // Get order details
    const order = await getOrder(orderId);
    
    // Create reconciliation
    const reconciliation = await createReconciliation({
      transactionId: transactionId,
      internalReference: `ORDER-${orderId}`,
      amount: order.total,
      currency: order.currency,
      transactionDate: new Date().toISOString(),
      notes: `Payment for order ${orderId}`
    });
    
    // Update order status
    await updateOrderStatus(orderId, 'paid');
    
    return reconciliation;
  } catch (error) {
    console.error('Reconciliation failed:', error);
    throw error;
  }
};
```

### Accounting System Integration

```javascript
// Integrate with accounting system
const syncWithAccounting = async (reconciliationId) => {
  try {
    const reconciliation = await getReconciliation(reconciliationId);
    
    // Sync with accounting system
    await accountingSystem.createJournalEntry({
      reference: reconciliation.internalReference,
      amount: reconciliation.amount,
      currency: reconciliation.currency,
      date: reconciliation.transactionDate,
      description: reconciliation.notes
    });
    
    // Update reconciliation status
    await updateReconciliationStatus(reconciliationId, 'matched');
    
  } catch (error) {
    console.error('Accounting sync failed:', error);
    throw error;
  }
};
```

## Environment Configuration

### Production Environment

```bash
# .env file
DCE_API_KEY=v8_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_BASE_URL=https://api.dcepay.io
DCE_WEBHOOK_SECRET=your_webhook_secret_here
```

### Test Environment

```bash
# Test environment
DCE_API_KEY=v8_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_BASE_URL=https://staging.dcepay.io
DCE_WEBHOOK_SECRET=test_webhook_secret_here
```

### Staging Environment

```bash
# Staging environment
DCE_API_KEY=v8_staging_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_BASE_URL=https://staging.dcepay.io
DCE_WEBHOOK_SECRET=staging_webhook_secret_here
```

## Error Handling Examples

### Comprehensive Error Handling

```javascript
const handleReconciliationError = async (error, reconciliationData) => {
  // Log error details
  console.error('Reconciliation error:', {
    error: error.message,
    reconciliationData,
    timestamp: new Date().toISOString()
  });
  
  // Categorize error
  if (error.message.includes('Transaction not found')) {
    // Handle missing transaction
    await logMissingTransaction(reconciliationData.transactionId);
    return { status: 'error', reason: 'transaction_not_found' };
  }
  
  if (error.message.includes('Invalid reconciliation data')) {
    // Handle validation error
    await logValidationError(reconciliationData);
    return { status: 'error', reason: 'validation_failed' };
  }
  
  // Handle unknown errors
  await logUnknownError(error);
  return { status: 'error', reason: 'unknown_error' };
};
```

### Retry Logic

```javascript
const createReconciliationWithRetry = async (data, maxRetries = 3) => {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await createReconciliation(data);
    } catch (error) {
      if (attempt === maxRetries) {
        throw error;
      }
      
      // Wait before retry (exponential backoff)
      const delay = Math.pow(2, attempt) * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
      
      console.log(`Reconciliation attempt ${attempt} failed, retrying...`);
    }
  }
};
```

***

_For more information about transaction management, see the [Transactions](transactions.md) documentation._

---
title: Settlements
deprecated: false
hidden: true
metadata:
  robots: index
---
The settlements API allows merchants to request payouts of their accumulated funds and manage settlement workflows. This guide covers settlement creation, approval processes, status tracking, and settlement address management.

## Overview

Settlements enable merchants to withdraw their accumulated funds from the platform. The settlement flow includes:

1. **Request settlement** - Submit a settlement request with destination address
2. **Approval process** - Administrative review and approval
3. **Processing** - Fund transfer to destination address
4. **Confirmation** - Settlement completion and status updates

## Creating Settlements

### POST /api/settlements

Create a new settlement request to withdraw accumulated funds.

#### Request

```bash
curl -X POST "https://api.dcepay.io/api/settlements" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user_xxxxxxxxxxxxxxxx",
    "currency": "USD",
    "requestedAmount": 1000.00,
    "destinationAddress": "0x1234567890123456789012345678901234567890",
    "destinationNetwork": "ETH",
    "isResellerSettlement": false
  }'
```

#### Request Parameters

| Parameter              | Type    | Required | Description                                  |
| ---------------------- | ------- | -------- | -------------------------------------------- |
| `userId`               | string  | Yes      | User ID requesting the settlement            |
| `currency`             | string  | No       | Currency for settlement (default: USD)       |
| `requestedAmount`      | number  | Yes      | Amount to settle                             |
| `destinationAddress`   | string  | Yes      | Destination address for settlement           |
| `destinationNetwork`   | string  | Yes      | Network for settlement (ETH, TRX, SEP, etc.) |
| `isResellerSettlement` | boolean | No       | Whether this is a reseller settlement        |

#### Response

```json
{
  "id": "set_xxxxxxxxxxxxxxxx",
  "userId": "user_xxxxxxxxxxxxxxxx",
  "currency": "USD",
  "requestedAmount": "1000.00",
  "destinationAddress": "0x1234567890123456789012345678901234567890",
  "destinationNetwork": "ETH",
  "status": "PENDING",
  "settlementFees": "25.00",
  "netAmount": "975.00",
  "createdAt": "2024-12-19T10:30:00Z",
  "processedAt": null,
  "approvedAt": null,
  "approvedBy": null,
  "notes": null
}
```

#### JavaScript Example

```javascript
async function createSettlement(settlementData) {
  const response = await fetch('https://api.dcepay.io/api/settlements', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(settlementData)
  });

  if (!response.ok) {
    throw new Error(`Failed to create settlement: ${response.statusText}`);
  }

  return response.json();
}

// Usage
const settlement = await createSettlement({
  userId: 'user_xxxxxxxxxxxxxxxx',
  currency: 'USD',
  requestedAmount: 1000.00,
  destinationAddress: '0x1234567890123456789012345678901234567890',
  destinationNetwork: 'ETH',
  isResellerSettlement: false
});
```

## Retrieving Settlements

### GET /api/settlements

Retrieve a list of settlements with optional filtering.

#### Request

```bash
curl -X GET "https://api.dcepay.io/api/settlements?userId=user_123&status=PENDING&page=1&limit=10" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Query Parameters

| Parameter              | Type    | Required | Description                                                         |
| ---------------------- | ------- | -------- | ------------------------------------------------------------------- |
| `userId`               | string  | No       | Filter by user ID                                                   |
| `status`               | string  | No       | Filter by status (PENDING, APPROVED, PROCESSING, COMPLETED, FAILED) |
| `isResellerSettlement` | boolean | No       | Filter reseller settlements                                         |
| `page`                 | number  | No       | Page number (default: 1)                                            |
| `limit`                | number  | No       | Items per page (default: 20, max: 100)                              |

#### Response

```json
{
  "settlements": [
    {
      "id": "set_xxxxxxxxxxxxxxxx",
      "userId": "user_xxxxxxxxxxxxxxxx",
      "currency": "USD",
      "requestedAmount": "1000.00",
      "destinationAddress": "0x1234567890123456789012345678901234567890",
      "destinationNetwork": "ETH",
      "status": "PENDING",
      "settlementFees": "25.00",
      "netAmount": "975.00",
      "createdAt": "2024-12-19T10:30:00Z",
      "processedAt": null,
      "approvedAt": null,
      "approvedBy": null,
      "notes": null,
      "user": {
        "id": "user_xxxxxxxxxxxxxxxx",
        "email": "merchant@example.com",
        "companyName": "Example Merchant"
      }
    }
  ],
  "pagination": {
    "total": 50,
    "page": 1,
    "limit": 20,
    "pages": 3
  }
}
```

## Settlement Status

### Status Values

| Status       | Description                                     |
| ------------ | ----------------------------------------------- |
| `PENDING`    | Settlement request submitted, awaiting approval |
| `APPROVED`   | Settlement approved by administrator            |
| `PROCESSING` | Settlement being processed and transferred      |
| `COMPLETED`  | Settlement successfully completed               |
| `FAILED`     | Settlement failed during processing             |
| `CANCELLED`  | Settlement cancelled by user or administrator   |

### Status Transitions

```text
PENDING → APPROVED (administrator approval)
APPROVED → PROCESSING (fund transfer initiated)
PROCESSING → COMPLETED (successful transfer)
PROCESSING → FAILED (transfer failure)
PENDING → CANCELLED (cancellation)
```

## Settlement Approval

### POST /api/admin/settlements/{settlementId}/approve

Approve a pending settlement request (admin only).

#### Request

```bash
curl -X POST "https://api.dcepay.io/api/admin/settlements/set_xxxxxxxxxxxxxxxx/approve" \
  -H "Authorization: Bearer ADMIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "notes": "Approved after KYC verification"
  }'
```

#### Response

```json
{
  "id": "set_xxxxxxxxxxxxxxxx",
  "status": "APPROVED",
  "approvedAt": "2024-12-19T11:00:00Z",
  "approvedBy": "admin_xxxxxxxxxxxxxxxx",
  "notes": "Approved after KYC verification"
}
```

## Settlement Addresses

### Managing Settlement Addresses

Settlement addresses must be whitelisted for security. Resellers can manage their settlement addresses through the API.

#### GET /api/resellers/{resellerId}/settlement-addresses

Retrieve whitelisted settlement addresses for a reseller.

```bash
curl -X GET "https://api.dcepay.io/api/resellers/res_xxxxxxxxxxxxxxxx/settlement-addresses" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Response

```json
{
  "addresses": [
    {
      "id": "addr_xxxxxxxxxxxxxxxx",
      "address": "0x1234567890123456789012345678901234567890",
      "networkSymbol": "ETH",
      "label": "Main Wallet",
      "isDefault": true,
      "isActive": true,
      "createdAt": "2024-12-19T10:30:00Z"
    }
  ]
}
```

## Settlement Fees

### Fee Structure

Settlements incur fees based on the following structure:

- **Base Fee**: 2.5% of settlement amount
- **Network Fee**: Variable based on blockchain network
- **Processing Fee**: Fixed amount per settlement

### Fee Calculation Example

```javascript
function calculateSettlementFees(requestedAmount, currency = 'USD') {
  const baseFeeRate = 0.025; // 2.5%
  const processingFee = 5.00; // Fixed processing fee
  
  const baseFee = requestedAmount * baseFeeRate;
  const totalFees = baseFee + processingFee;
  const netAmount = requestedAmount - totalFees;
  
  return {
    requestedAmount,
    baseFee,
    processingFee,
    totalFees,
    netAmount
  };
}

// Example calculation
const fees = calculateSettlementFees(1000.00);
console.log(fees);
// {
//   requestedAmount: 1000.00,
//   baseFee: 25.00,
//   processingFee: 5.00,
//   totalFees: 30.00,
//   netAmount: 970.00
// }
```

## Reseller Settlements

### Special Considerations

Reseller settlements have additional security requirements:

1. **Address Whitelisting**: Only whitelisted addresses can receive settlements
2. **KYC Verification**: Enhanced verification may be required
3. **Volume Limits**: Settlement amounts may be limited based on volume
4. **Approval Workflow**: Extended approval process for large amounts

### Reseller Settlement Example

```javascript
async function createResellerSettlement(resellerId, amount, address) {
  // Verify address is whitelisted
  const addresses = await getSettlementAddresses(resellerId);
  const isWhitelisted = addresses.some(addr => 
    addr.address === address && addr.isActive
  );
  
  if (!isWhitelisted) {
    throw new Error('Destination address is not whitelisted');
  }
  
  return createSettlement({
    userId: resellerId,
    currency: 'USD',
    requestedAmount: amount,
    destinationAddress: address,
    destinationNetwork: 'ETH',
    isResellerSettlement: true
  });
}
```

## Error Handling

### Common Errors

| Status Code | Error                                    | Description                           |
| ----------- | ---------------------------------------- | ------------------------------------- |
| 400         | `Insufficient balance`                   | User doesn't have enough funds        |
| 400         | `Destination address is not whitelisted` | Address not in whitelist              |
| 400         | `Invalid settlement amount`              | Amount below minimum or above maximum |
| 401         | `Unauthorized`                           | Missing or invalid API key            |
| 403         | `Insufficient permissions`               | User lacks required permissions       |
| 404         | `Settlement not found`                   | Settlement ID not found               |

### Error Response Format

```json
{
  "error": "Insufficient balance",
  "details": {
    "available": "500.00",
    "requested": "1000.00",
    "shortfall": "500.00"
  }
}
```

## Best Practices

### 1. Settlement Planning

- Monitor balance regularly to plan settlements
- Consider fee structure when calculating settlement amounts
- Use whitelisted addresses for security

### 2. Address Management

```javascript
class SettlementAddressManager {
  async addSettlementAddress(resellerId, address, network, label) {
    // Validate address format
    if (!this.isValidAddress(address, network)) {
      throw new Error('Invalid address format');
    }
    
    // Add to whitelist
    return await fetch(`/api/resellers/${resellerId}/settlement-addresses`, {
      method: 'POST',
      headers: { 'Authorization': `Bearer ${this.apiKey}` },
      body: JSON.stringify({ address, network, label })
    });
  }
  
  isValidAddress(address, network) {
    const patterns = {
      'ETH': /^0x[a-fA-F0-9]{40}$/,
      'TRX': /^T[a-zA-Z0-9]{33}$/,
      'BTC': /^[13][a-km-zA-HJ-NP-Z1-9]{25,34}$/
    };
    
    return patterns[network]?.test(address) || false;
  }
}
```

### 3. Settlement Monitoring

```javascript
class SettlementMonitor {
  async trackSettlement(settlementId) {
    const checkStatus = async () => {
      const settlement = await getSettlement(settlementId);
      
      switch (settlement.status) {
        case 'COMPLETED':
          console.log('Settlement completed successfully');
          return true;
        case 'FAILED':
          console.error('Settlement failed:', settlement.error);
          return false;
        case 'PROCESSING':
          // Continue monitoring
          setTimeout(checkStatus, 30000); // Check again in 30 seconds
          break;
        default:
          // Still pending or approved
          setTimeout(checkStatus, 60000); // Check again in 1 minute
      }
    };
    
    return checkStatus();
  }
}
```

### 4. Batch Settlements

```javascript
async function processBatchSettlements(settlements) {
  const results = [];
  
  for (const settlement of settlements) {
    try {
      const result = await createSettlement(settlement);
      results.push({ success: true, settlement: result });
    } catch (error) {
      results.push({ success: false, error: error.message });
    }
  }
  
  return results;
}
```

## Integration Examples

### E-commerce Platform Integration

```javascript
class SettlementService {
  constructor(apiKey) {
    this.apiKey = apiKey;
  }
  
  async requestMonthlySettlement(merchantId, month, year) {
    // Calculate monthly revenue
    const startDate = new Date(year, month - 1, 1);
    const endDate = new Date(year, month, 0);
    
    const transactions = await getTransactions({
      userId: merchantId,
      type: 'DEPOSIT',
      status: 'CONFIRMED',
      startDate: startDate.toISOString(),
      endDate: endDate.toISOString()
    });
    
    const totalRevenue = transactions.reduce((sum, tx) => 
      sum + parseFloat(tx.amount), 0
    );
    
    // Request settlement
    return await createSettlement({
      userId: merchantId,
      currency: 'USD',
      requestedAmount: totalRevenue,
      destinationAddress: await getDefaultSettlementAddress(merchantId),
      destinationNetwork: 'ETH'
    });
  }
}
```

### Accounting System Integration

```javascript
class AccountingIntegration {
  async exportSettlements(startDate, endDate) {
    const settlements = await getSettlements({
      startDate,
      endDate,
      status: 'COMPLETED'
    });
    
    return settlements.map(settlement => ({
      date: settlement.processedAt,
      reference: settlement.id,
      amount: settlement.requestedAmount,
      currency: settlement.currency,
      fees: settlement.settlementFees,
      netAmount: settlement.netAmount,
      destination: settlement.destinationAddress
    }));
  }
}
```

***

_For more information about transaction management, see the [Transactions](transactions.md) documentation._
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
DCE_BASE_URL=https://api.dcepay.dev
```

## Authentication

Send your API key in `Authorization` using **either**:

```
Authorization: YOUR_API_KEY
```

```
Authorization: Bearer YOUR_API_KEY
```

The hosted deposit browser flow uses `GET /api/deposit-page?token=...` and `GET /api/deposit-page/status?token=...` and does **not** use your API key in the browser.

## Currencies and Networks

Balances and money movement are segmented per **(currency, network)** pair:

- `currency`: `USDT`, `USDC`
- `network`: `TRX`, `ETH`, `BNB`, `SOL` (testnets: `TRX_SHASTA`, `SEP`, `tBNB`, `SOL_DEVNET`)

**Currently enabled for deposits and withdrawals: USDT on TRX** (and its `TRX_SHASTA` testnet twin). The other pairs (USDT/USDC on ETH, BNB, SOL) exist in the capability matrix but are disabled until verified — they are coming soon and can be enabled on request without an integration change on your side.

Key consequences:

- There is **no cross-chain fungibility**: a withdrawal draws only from the same chain's balance. Funds deposited on TRX cannot be withdrawn on ETH.
- Requests referencing a disabled pair are rejected, e.g. `{"error": "Withdrawals of USDT on ETH are not supported"}` or `{"error": "Deposits of USDC on TRX are not supported"}`.
- Withdrawals require a `network`, and the per-network `networkFee` is quoted at submission time. The quoted fee is what you are charged.

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
Get per-(currency, network) balances. Balances are segmented per chain — deposits on a chain can only be withdrawn on that chain.

**Query Parameters:**
- `currency` (optional): Token symbol (`USDT`, `USDC`)
- `network` (optional): Network symbol (`TRX`, `ETH`, `BNB`, `SOL`, or a testnet symbol)

**Response (both `currency` and `network` supplied — single balance row):**
```json
{
  "currency": "USDT",
  "network": "TRX",
  "available": "1000.00",
  "pending": "50.00",
  "lastUpdatedAt": "2026-07-19T10:30:00Z"
}
```

If the account has no balance row for the pair, `available` and `pending` are returned as `"0"`.

**Response (otherwise — every balance row the account holds, optionally filtered):**
```json
{
  "balances": [
    {
      "currency": "USDT",
      "network": "TRX",
      "available": "1000.00",
      "pending": "50.00",
      "withdrawalEnabled": true,
      "networkFee": "1",
      "maxWithdrawable": "999.00",
      "lastUpdatedAt": "2026-07-19T10:30:00Z"
    }
  ]
}
```

- `withdrawalEnabled`: whether the pair currently supports withdrawals
- `networkFee`: the current per-network withdrawal fee quote (`null` when withdrawals are disabled for the pair)
- `maxWithdrawable`: available balance after the network fee

**Example Request:**
```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/balance?currency=USDT&network=TRX`, {
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
- `limit` (optional): Items per page (default: 20)
- `type` (optional): Transaction type (DEPOSIT, WITHDRAWAL)
- `status` (optional): Transaction status (PENDING, CONFIRMED, FAILED, CANCELLED)
- `currency` (optional): Token symbol (`USDT`, `USDC`)
- `network` (optional): Network symbol (`TRX`, `ETH`, `BNB`, `SOL`, or a testnet symbol)

**Response:**
```json
{
  "transactions": [
    {
      "id": "txn_123",
      "txnHash": "8f2c1d...",
      "block": null,
      "age": "5 mins ago",
      "from": "TSenderAddress...",
      "to": "TReceiverAddress...",
      "amount": "97.50",
      "fee": "2.50",
      "token": "USDT",
      "type": "DEPOSIT",
      "status": "CONFIRMED",
      "balanceBefore": "900.00",
      "balanceAfter": "997.50",
      "createdAt": "2026-07-19T10:30:00Z",
      "createdAtGmt8": "2026-07-19T18:30:00.000Z",
      "description": null,
      "category": null,
      "tags": [],
      "metadata": {}
    }
  ],
  "balance": {
    "currency": "USDT",
    "network": "TRX",
    "available": "997.50"
  },
  "pagination": {
    "total": 100,
    "page": 1,
    "limit": 20,
    "pages": 5
  }
}
```

Notes:
- `amount` is the signed balance change (positive for deposits net of fees, negative for withdrawals including fees).
- The running `balanceBefore`/`balanceAfter` columns are only exact when **both** `currency` and `network` filters are supplied, because balances are per-chain.

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
- `status` (optional): Deposit status (PENDING, CONFIRMED, FAILED, CANCELLED)

**Response:**
```json
{
  "deposits": [
    {
      "id": "dep_123",
      "amount": "100.00",
      "currency": "USDT",
      "network": "TRX",
      "status": "CONFIRMED",
      "source": "wallet_address",
      "referenceId": "order_1001",
      "createdAt": "2026-07-19T10:30:00Z",
      "clearedAt": "2026-07-19T10:35:00Z",
      "fee": "0",
      "commission": "1.00",
      "depositAddress": { "...": "linked deposit address, when present" }
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

Notes:
- `commission` is the deposit commission. `fee` mirrors fixed charges tied to the deposit (address activation fee, direct deposit fee) for audit — it is **not** the deposit commission.
- Merchant self-custody **top-up deposits are sweep-gated**: the deposit credits the pending balance first and moves to available once the sweep to the master wallet lands, so top-up funds may sit in `pending` briefly.
- Deposits under 1 USD equivalent are fee-exempt and do not trigger a merchant deposit callback.

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
Create a new deposit transaction record (credited to the pending balance).

**Request Body:**
```json
{
  "amount": "100.00",
  "currency": "USDT",
  "network": "TRX",
  "source": "wallet_address",
  "description": "Payment for order #123",
  "metadata": {
    "orderId": "order_123",
    "customerEmail": "customer@example.com"
  },
  "depositAddressId": "addr_123"
}
```

- `amount`: decimal string
- `currency` (required): `USDT` or `USDC`
- `network` (required): network symbol — the deposit credits this chain's balance only
- `description`, `metadata`, `depositAddressId`: optional

**Response:**
```json
{
  "id": "dep_123",
  "amount": "100.00",
  "currency": "USDT",
  "network": "TRX",
  "status": "PENDING",
  "source": "wallet_address",
  "description": "Payment for order #123",
  "createdAt": "2026-07-19T10:30:00Z",
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
    amount: '100.00',
    currency: 'USDT',
    network: 'TRX',
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
- `limit` (optional): Items per page (default: 10, max: 100)
- `status` (optional): Withdrawal status (PENDING, CONFIRMED, FAILED, CANCELLED)

**Response:**
```json
{
  "withdrawals": [
    {
      "id": "wth_123",
      "amount": "50.00",
      "currency": "USDT",
      "network": "TRX",
      "status": "CONFIRMED",
      "destination": "TDestinationAddress...",
      "description": null,
      "createdAt": "2026-07-19T10:30:00Z",
      "processedAt": "2026-07-19T10:35:00Z",
      "fee": "0",
      "commission": "1.50",
      "commissionRate": "0.03"
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
Create a new withdrawal request (on-chain payout). Requires an API key with write/payout capability, and the merchant account must be **ACTIVE**.

**Request Body:**
```json
{
  "amount": "50.00",
  "currency": "USDT",
  "network": "TRX",
  "destination": "TDestinationAddress...",
  "description": "Payout for order #123",
  "referenceId": "payout_1001"
}
```

- `amount`: decimal string, must be greater than zero
- `currency` (required): `USDT` or `USDC`
- `network` (required): network symbol — the withdrawal draws **only** from this chain's balance
- `destination` (required): on-chain destination address
- `referenceId` (optional but recommended): your own reference, **unique per merchant** — see idempotency below
- `description` (optional)

**Response:**
```json
{
  "success": true,
  "message": "Withdrawal request received and will begin processing",
  "withdrawalId": "wth_123",
  "status": "PENDING",
  "amount": "50.00",
  "currency": "USDT",
  "network": "TRX",
  "destination": "TDestinationAddress...",
  "feeInfo": {
    "grossAmount": "50.00",
    "netAmount": "47.50",
    "totalFees": "2.50",
    "feeBreakdown": {
      "baseFee": "1.50",
      "markupRate": "0.03",
      "markupAmount": "1.50",
      "networkFee": "1",
      "totalFee": "2.50"
    }
  }
}
```

**Fees:** the total debited from your balance is `amount + commission + networkFee`. The per-network `networkFee` is quoted from the capability matrix at submission and locked in — you are charged the quoted fee even if the finalized on-chain cost differs.

**Idempotency via `referenceId`:** the `referenceId` is unique per merchant account. Reusing a `referenceId` never creates a second payout — the duplicate submission is rejected with `409`:

```json
{
  "error": "Duplicate referenceId",
  "message": "A withdrawal with this referenceId already exists for your merchant account. Use a new referenceId to submit a new withdrawal."
}
```

If a previous attempt with that `referenceId` ended in `FAILED` or `CANCELLED`, the reference is released automatically and a retry with the same `referenceId` is accepted.

**Error responses:**

| Status | Body (`error`) | Cause |
|---|---|---|
| `400` | `Withdrawal amount must be greater than zero` | Non-positive amount |
| `400` | `Withdrawals of <currency> on <network> are not supported` | Disabled (currency, network) pair |
| `400` | `Amount below minimum withdrawal` (includes `minWithdrawal`, `currency`, `network`) | Below the pair's minimum |
| `400` | `No <currency> balance on <network> for this account` | No balance row on that chain |
| `400` | `Insufficient balance` (includes `available`, `requested`, `currency`, `network`, `feeInfo`) | Balance on that chain cannot cover `amount + fees`; funds on other networks cannot be used |
| `403` | `Insufficient permissions to initiate a withdrawal` | Read-only API key |
| `403` | `Merchant account is not active` | Suspended/inactive merchant (also applies to deposit-URL creation) |
| `409` | `Duplicate referenceId` | `referenceId` already used by your merchant account |
| `503` | `Withdrawal temporarily unavailable` | Platform treasury liquidity guard — nothing is created and the `referenceId` is not burned; retry shortly |
| `500` | `Payout initiation failed` | Payout submission failed after creation; the reserved amount is returned to your balance and the `referenceId` is released for retry |

**Webhook correlation:** the `withdrawal.confirmed` / `withdrawal.failed` callback payloads carry **your** `referenceId` (not the internal withdrawal id), so you can correlate callbacks with the withdrawal you submitted.

**Example Request:**
```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/withdrawals`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    amount: '50.00',
    currency: 'USDT',
    network: 'TRX',
    destination: 'TDestinationAddress...',
    description: 'Payout for order #123',
    referenceId: 'payout_1001'
  })
});

if (!response.ok) {
  const errorData = await response.json();
  if (response.status === 409) {
    // Duplicate referenceId — this payout was already submitted.
    console.warn('Withdrawal already exists for referenceId:', errorData.message);
  } else if (errorData.details) {
    console.error('Validation errors:', errorData.details);
  }
  throw new Error(`Failed to create withdrawal: ${errorData.error}`);
}

const withdrawal = await response.json();
```

**Reseller keys:** withdrawals initiated with a RESELLER-role API key are pinned to the reseller's registered withdrawal address (a mismatched destination or network is rejected with `400`) and require admin approval before the on-chain payout is submitted (`status: "PENDING_APPROVAL"`). See the partner/admin surface note below.

### Deposit Addresses

#### GET /api/deposit-address
Get the deposit addresses for the user.

**Response:**
```json
{
  "addresses": [
    {
      "id": "addr_123",
      "address": "TDepositAddress...",
      "networkSymbol": "TRX",
      "createdAt": "2026-07-19T10:30:00Z",
      "deposits": [
        {
          "id": "dep_123",
          "amount": "100.00",
          "currency": "USDT",
          "status": "CONFIRMED",
          "createdAt": "2026-07-19T10:35:00Z"
        }
      ]
    }
  ]
}
```

**Example Request:**
```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-address`, {
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
Create a new deposit address for customer deposits. Addresses are per network and receive any supported token on that chain; no address is issued for a network with no deposit-enabled pair (`400` — `Deposits on <network> are not currently supported`).

An **address activation fee** (fixed, charged once per address activation) may apply on the first deposit to a new address — see [Fees Reference](https://docs.dcepay.io/docs/fees-reference).

**Request Body:**
```json
{
  "network": "TRX",
  "identifier": "customer_123",
  "referenceId": "order_1001"
}
```

- `network` (required): network symbol
- `identifier` (required): your identifier for the end customer
- `referenceId` (optional): your reference for correlating deposit callbacks

**Response:** the created (or refreshed) deposit address record, including `address` and `networkSymbol`.

**Example Request:**
```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-address`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    network: 'TRX',
    identifier: 'customer_123',
    referenceId: 'order_1001'
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
- `network` (optional): Network symbol (e.g., "TRX")

**Response:**
```json
{
  "depositUrls": [
    {
      "id": "url_123",
      "url": "https://api.dcepay.io/deposit/url_123",
      "address": "TDepositAddress...",
      "network": "TRX",
      "createdAt": "2026-07-19T10:30:00Z",
      "deposits": [
        {
          "id": "dep_123",
          "amount": "100.00",
          "currency": "USDT",
          "status": "CONFIRMED",
          "createdAt": "2026-07-19T10:35:00Z"
        }
      ]
    }
  ]
}
```

**Example Request:**
```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-url?network=TRX`, {
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
Create a hosted deposit page URL (with a fresh deposit address and a short-lived session token). Requires an **ACTIVE** merchant account, and the network must have at least one deposit-enabled pair (`400` — `Deposits on <network> are not currently supported`).

**Request Body:**
```json
{
  "network": "TRX",
  "identifier": "customer_123",
  "referenceId": "order_1001",
  "requestedCurrency": "USD",
  "requestedAmount": "25.00"
}
```

- `network` (required): network symbol
- `identifier` (required): your identifier for the end customer
- `referenceId` (optional): your reference, appended to the URL and used for callback correlation
- `requestedCurrency` + `requestedAmount` (optional, together): a fiat quote; the response includes the exchange rate and the token amount to request from the customer
- `token` (optional): token symbol hint appended to the URL

**Response (201):**
```json
{
  "url": "https://api.dcepay.io/deposit/<sessionToken>?ref=order_1001&amount=25.00&currency=USD",
  "address": "TDepositAddress...",
  "identifier": "customer_123",
  "network": "TRX",
  "sessionToken": "<sessionToken>",
  "expiresAt": "2026-07-19T10:45:00Z",
  "referenceId": "order_1001",
  "requestedAmount": "25.00",
  "requestedCurrency": "USD",
  "exchangeRate": "1.0002",
  "amount": "24.99500100"
}
```

The session expires 15 minutes after creation. The hosted page shows **live payment status** (see `GET /api/deposit-page/status` below).

**Example Request:**
```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/deposit-url`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    network: 'TRX',
    identifier: 'customer_123',
    referenceId: 'order_1001'
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

### Hosted Deposit Page

#### GET /api/deposit-page/status
Live payment status for a hosted deposit page session. **Public** (token-authenticated) — no API key; safe to poll from the browser. Responses are sent with `Cache-Control: no-store`.

**Query Parameters:**
- `token` (required): the hosted page session token

**Response:**
```json
{
  "status": "completed",
  "deposit": {
    "amount": "25.00",
    "currency": "USDT",
    "txHash": "8f2c1d...",
    "receivedAt": "2026-07-19T10:32:00.000Z",
    "creditedAt": "2026-07-19T10:35:00.000Z"
  }
}
```

`status` values:
- `waiting` — no payment seen yet (`deposit` is omitted)
- `received` — payment detected on-chain, not yet credited
- `completed` — deposit confirmed and credited
- `failed` — the deposit failed or was cancelled

**Error responses:**
- `400` — `Token is required`
- `404` — `Invalid token`
- `410` — `Session expired`

### Exchange Rates

#### GET /api/exchange-rates
Get current exchange rates for a fiat currency against the supported networks.

**Query Parameters:**
- `requestedCurrency` (required): 3-letter fiat currency code (e.g., "USD")

**Response:**
```json
{
  "baseCurrency": "USD",
  "rates": {
    "TRX": "1.0002"
  },
  "timestamp": "2026-07-19T10:30:00.000Z"
}
```

**Example Request:**
```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/exchange-rates?requestedCurrency=USD`, {
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
- `limit` (optional): Items per page (default: 20, max: 100)
- `status` (optional): Settlement status (PENDING, APPROVAL_REQUIRED, APPROVED, PROCESSING, SUBMITTED_TO_AKASHIC, COMPLETED, FAILED, CANCELLED, REJECTED)

**Response:**
```json
{
  "settlements": [
    {
      "id": "set_123",
      "amount": "1000.00",
      "currency": "USDT",
      "network": "TRX",
      "status": "PENDING",
      "destinationAddress": "TSettlementAddress...",
      "destinationNetwork": "TRX",
      "createdAt": "2026-07-19T10:30:00Z",
      "totalFees": "31.00",
      "merchantAddress": {
        "address": "TSettlementAddress...",
        "label": "Treasury wallet"
      }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 10,
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
Create a new settlement request. A settlement is a payout to one of your **whitelisted settlement addresses**; it drains the (currency, destinationNetwork) balance only — no cross-chain moves.

**Request Body:**
```json
{
  "userId": "user_123",
  "currency": "USDT",
  "requestedAmount": "1000.00",
  "destinationAddress": "TSettlementAddress...",
  "destinationNetwork": "TRX"
}
```

- `currency` (required): `USDT` or `USDC`
- `requestedAmount`: decimal string
- `destinationAddress` (required): must be a whitelisted, active settlement address for your merchant account (`400` — `Destination address is not whitelisted for this merchant`)
- `destinationNetwork` (required): network symbol; the pair must be withdrawal-enabled (`400` — `Withdrawals of <currency> on <network> are not supported`)
- `userId`: for non-admin keys this is always your own account

**Response (201):** the created settlement (with linked `withdrawalId`). Fees follow the withdrawal fee model: the total debited is `requestedAmount + commission + networkFee`.

Settlements at or above the platform's approval threshold are created with `status: "APPROVAL_REQUIRED"` (funds reserved, payout submitted after admin approval) and the message `Settlement created and awaiting admin approval`; otherwise the payout is submitted immediately.

**Example Request:**
```javascript
const response = await fetch(`${process.env.DCE_BASE_URL}/api/settlements`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    userId: 'user_123',
    currency: 'USDT',
    requestedAmount: '1000.00',
    destinationAddress: 'TSettlementAddress...',
    destinationNetwork: 'TRX'
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

### Merchant Earnings

#### GET /api/merchants/{merchantId}/earnings
Fee-margin earnings for your own merchant account. On a reseller line, the margin component of each fee event is credited back to your balance; this endpoint reports those credits, aggregated per (currency, network).

**Query Parameters:**
- `startDate` (optional): ISO date — filter fee events from this date
- `endDate` (optional): ISO date — filter fee events up to this date

**Response:**
```json
{
  "merchantId": "merchant_123",
  "earnings": [
    {
      "currency": "USDT",
      "network": "TRX",
      "feeEvents": 42,
      "volume": "10500.00",
      "grossFeesCharged": "315.00",
      "marginEarned": "52.50",
      "reversed": "0",
      "net": "52.50"
    }
  ]
}
```

- `feeEvents`: number of applied fee events
- `volume`: summed base amounts of those events
- `grossFeesCharged`: total fees charged
- `marginEarned`: your margin share credited back
- `reversed`: clawed-back margin (e.g., from failed payouts)
- `net`: `marginEarned - reversed`

Requires a key that can read the merchant (`403` — `Insufficient permissions` otherwise).

### Webhooks (outbound merchant callbacks)

DCE sends signed POST callbacks to your configured `webhookUrl` for events such as `deposit.confirmed`, `deposit.failed`, `withdrawal.confirmed`, `withdrawal.failed`, `transaction.confirmed`, and `transaction.failed`.

**Delivery headers:**
- `Content-Type: application/json`
- `X-Webhook-Event`: the event type
- `X-Webhook-Signature`: hex HMAC-SHA256 of the JSON body using your webhook secret
- `X-Webhook-Id`: unique delivery id (also present as `eventId` in the payload)

**Delivery semantics:** outgoing webhooks go through a durable outbox with retries — delivery is **at-least-once**, so your consumer must be idempotent (dedupe on `eventId` / `X-Webhook-Id`, or on event type + `referenceId`).

**Correlation:** deposit and withdrawal payloads carry **your** `referenceId` (the one you submitted), not the internal record id.

**Example `withdrawal.confirmed` payload:**
```json
{
  "type": "withdrawal",
  "withdrawalId": "wth_123",
  "amount": "50.00",
  "currency": "USDT",
  "status": "confirmed",
  "txHash": "8f2c1d...",
  "referenceId": "payout_1001",
  "address": "TDestinationAddress...",
  "identifier": "payout_1001",
  "feeCharges": {
    "amount": "1.50",
    "percentage": "3",
    "type": "PERCENTAGE"
  },
  "receivableAmount": "48.50",
  "eventId": "whlog_123"
}
```

For payload shapes, signature verification, and retry behavior in full, see [Webhooks](https://docs.dcepay.io/docs/webhooks).

**Note:** `POST /api/webhook/event` is an operator-only internal ingestion route — it is **not** your merchant webhook URL. You configure `webhookUrl` on your merchant profile and receive outbound POSTs from DCE.

**Example Webhook Handler:**
```javascript
import crypto from 'crypto';

function verifyWebhookSignature(rawBody, signature, secret) {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(rawBody, 'utf8')
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature, 'hex'),
    Buffer.from(expectedSignature, 'hex')
  );
}

app.post('/webhook', async (req, res) => {
  try {
    const signature = req.headers['x-webhook-signature'];
    const eventType = req.headers['x-webhook-event'];
    const payload = req.body;

    // Verify webhook signature against the raw JSON body
    if (!verifyWebhookSignature(JSON.stringify(payload), signature, process.env.DCE_WEBHOOK_SECRET)) {
      return res.status(401).json({ error: 'Invalid signature' });
    }

    // Deliveries are at-least-once: dedupe before processing
    if (await alreadyProcessed(payload.eventId)) {
      return res.status(200).json({ ok: true });
    }

    switch (eventType) {
      case 'deposit.confirmed':
        await handleDepositConfirmed(payload);
        break;
      case 'withdrawal.confirmed':
        await handleWithdrawalConfirmed(payload);
        break;
      case 'withdrawal.failed':
        await handleWithdrawalFailed(payload);
        break;
      default:
        console.log(`Unhandled event type: ${eventType}`);
    }

    res.status(200).json({ ok: true });
  } catch (error) {
    console.error('Webhook processing error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### Partner and admin surface

The API also exposes routes that are **not part of the merchant integration surface**:

- `/api/resellers` and `/api/resellers/{resellerId}/...` — reseller line management (admin or the reseller itself)
- `GET/PUT /api/merchants/{merchantId}/reseller` — attach/detach a merchant to a reseller line (admin-only)
- `POST /api/admin/withdrawals/{withdrawalId}/approve` and `/reject` — admin approval queue for reseller payouts

Merchant API keys cannot call these; they are documented here only so the route names are recognizable in logs.

## Error Handling

### Common HTTP Status Codes

| Status Code | Description | Common Causes |
|-------------|-------------|---------------|
| `200` | OK - Request successful | - |
| `201` | Created - Resource created successfully | - |
| `400` | Bad Request - Invalid request data | Missing required fields, invalid format, disabled (currency, network) pair, insufficient per-chain balance |
| `401` | Unauthorized - Authentication required | Missing or invalid API key |
| `403` | Forbidden - Insufficient permissions | Key lacks required permissions, or merchant account is not active |
| `404` | Not Found - Resource not found | Invalid ID, resource doesn't exist |
| `409` | Conflict - Resource conflict | Duplicate withdrawal `referenceId`, business rule violation |
| `410` | Gone - Session expired | Hosted deposit page session token expired or already used |
| `422` | Unprocessable Entity - Validation failed | Invalid data format, business validation |
| `500` | Internal Server Error | Server-side error, unexpected exception |
| `503` | Service Unavailable | Withdrawals temporarily unavailable (treasury liquidity guard) — retry shortly |

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
          throw new Error('Resource conflict (e.g., duplicate withdrawal referenceId). Please check your request.');
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
  // Get balance for a specific chain
  const balance = await makeApiRequest('/api/balance?currency=USDT&network=TRX');
  console.log('Balance:', balance);
  
  // Create deposit
  const deposit = await makeApiRequest('/api/deposits', {
    method: 'POST',
    body: JSON.stringify({
      amount: '100.00',
      currency: 'USDT',
      network: 'TRX',
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
DCE_TEST_BASE_URL=https://api.dcepay.dev
DCE_TEST_WEBHOOK_SECRET=test_webhook_secret_here

# Staging environment
DCE_STAGING_API_KEY=v8_staging_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DCE_STAGING_BASE_URL=https://api.dcepay.dev
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

---

*For more information about specific features, see the [Authentication](https://docs.dcepay.io/docs/authentication), [Deposits](https://docs.dcepay.io/docs/deposits), [Withdrawals](https://docs.dcepay.io/docs/withdrawals), and [Webhooks](https://docs.dcepay.io/docs/webhooks) documentation.* 
Fee behavior is documented in [Fees Reference](https://docs.dcepay.io/docs/fees-reference).
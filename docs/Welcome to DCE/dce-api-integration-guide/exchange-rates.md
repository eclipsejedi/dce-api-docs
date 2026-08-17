---
title: Exchange Rates
deprecated: false
hidden: false
metadata:
  robots: index
---
_Last updated: 2026-04-22_

The exchange rates API provides real-time currency conversion rates and exchange rate management. This guide covers rate retrieval, currency conversion, rate updates, and integration patterns for handling multi-currency transactions.

## Overview

The exchange rates API enables:

- **Real-time Rates** - Get current exchange rates for all supported currencies
- **Currency Conversion** - Convert amounts between different currencies
- **Rate History** - Access historical exchange rate data
- **Rate Updates** - Receive notifications when rates change
- **Multi-currency Support** - Support for fiat and cryptocurrency pairs

## Retrieving Exchange Rates

### GET /api/exchange-rates

Retrieve current exchange rates for supported currency pairs.

#### Request

```bash
curl -X GET "https://api.dcepay.io/api/exchange-rates?requestedCurrency=USD" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `requestedCurrency` | string | No | Base currency for rates (default: USD) |
| `targetCurrencies` | string | No | Comma-separated list of target currencies |
| `includeHistorical` | boolean | No | Include historical rate data |

#### Response

```json
{
  "baseCurrency": "USD",
  "timestamp": "2024-12-19T10:30:00Z",
  "rates": {
    "ETH": "7.182",
    "BTC": "0.000041",
    "USDT": "1.000",
    "EUR": "0.85",
    "GBP": "0.73",
    "JPY": "110.50",
    "TRX": "0.001",
    "SEP": "7.182"
  },
  "metadata": {
    "source": "akashicpay",
    "lastUpdated": "2024-12-19T10:30:00Z",
    "updateFrequency": "real-time"
  }
}
```

#### JavaScript Example

```javascript
async function getExchangeRates(baseCurrency = 'USD', targetCurrencies = null) {
  const params = new URLSearchParams();
  params.append('requestedCurrency', baseCurrency);
  
  if (targetCurrencies) {
    params.append('targetCurrencies', targetCurrencies.join(','));
  }

  const response = await fetch(`https://api.dcepay.io/api/exchange-rates?${params}`, {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${process.env.DCE_API_KEY}`,
      'Content-Type': 'application/json'
    }
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch exchange rates: ${response.statusText}`);
  }

  return response.json();
}

// Usage examples
const allRates = await getExchangeRates('USD');
const specificRates = await getExchangeRates('USD', ['ETH', 'BTC', 'EUR']);
const eurRates = await getExchangeRates('EUR');
```

## Currency Conversion

### Converting Amounts

Convert amounts between different currencies using current rates:

```javascript
class CurrencyConverter {
  constructor(rates) {
    this.rates = rates;
  }
  
  convert(amount, fromCurrency, toCurrency) {
    if (fromCurrency === toCurrency) {
      return amount;
    }
    
    // Convert to base currency first
    const baseAmount = this.toBaseCurrency(amount, fromCurrency);
    
    // Convert from base currency to target currency
    return this.fromBaseCurrency(baseAmount, toCurrency);
  }
  
  toBaseCurrency(amount, currency) {
    const rate = this.rates[currency];
    if (!rate) {
      throw new Error(`Exchange rate not found for ${currency}`);
    }
    return amount * parseFloat(rate);
  }
  
  fromBaseCurrency(amount, currency) {
    const rate = this.rates[currency];
    if (!rate) {
      throw new Error(`Exchange rate not found for ${currency}`);
    }
    return amount / parseFloat(rate);
  }
}

// Usage
const rates = await getExchangeRates('USD');
const converter = new CurrencyConverter(rates.rates);

const usdAmount = 100;
const ethAmount = converter.convert(usdAmount, 'USD', 'ETH');
console.log(`${usdAmount} USD = ${ethAmount} ETH`);
```

## Supported Currencies

### Fiat Currencies

| Currency | Code | Name | Status |
|----------|------|------|--------|
| USD | US Dollar | Active |
| EUR | Euro | Active |
| GBP | British Pound | Active |
| JPY | Japanese Yen | Active |
| CAD | Canadian Dollar | Active |
| AUD | Australian Dollar | Active |
| CHF | Swiss Franc | Active |
| CNY | Chinese Yuan | Active |

### Cryptocurrencies

| Currency | Code | Name | Network | Status |
|----------|------|------|---------|--------|
| BTC | Bitcoin | Bitcoin | Active |
| ETH | Ethereum | Ethereum | Active |
| USDT | Tether | Ethereum | Active |
| USDC | USD Coin | Ethereum | Active |
| TRX | TRON | TRON | Active |
| SEP | Sepolia ETH | Ethereum Testnet | Active |

## Rate Updates

### Real-time Rate Monitoring

Monitor exchange rates for changes and updates:

```javascript
class RateMonitor {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.lastRates = null;
    this.callbacks = [];
  }
  
  async startMonitoring(baseCurrency = 'USD', interval = 30000) {
    console.log(`Starting rate monitoring for ${baseCurrency}`);
    
    const monitor = async () => {
      try {
        const rates = await getExchangeRates(baseCurrency);
        
        if (this.lastRates && this.hasSignificantChange(rates, this.lastRates)) {
          this.notifyRateChange(this.lastRates, rates);
        }
        
        this.lastRates = rates;
      } catch (error) {
        console.error('Rate monitoring error:', error);
      }
      
      setTimeout(monitor, interval);
    };
    
    await monitor();
  }
  
  hasSignificantChange(newRates, oldRates, threshold = 0.01) {
    for (const currency in newRates.rates) {
      const newRate = parseFloat(newRates.rates[currency]);
      const oldRate = parseFloat(oldRates.rates[currency]);
      
      if (Math.abs(newRate - oldRate) / oldRate > threshold) {
        return true;
      }
    }
    return false;
  }
  
  notifyRateChange(oldRates, newRates) {
    const changes = {};
    
    for (const currency in newRates.rates) {
      const newRate = parseFloat(newRates.rates[currency]);
      const oldRate = parseFloat(oldRates.rates[currency]);
      const change = ((newRate - oldRate) / oldRate) * 100;
      
      if (Math.abs(change) > 0.1) { // 0.1% threshold
        changes[currency] = {
          oldRate,
          newRate,
          change: change.toFixed(2)
        };
      }
    }
    
    this.callbacks.forEach(callback => callback(changes));
  }
  
  onRateChange(callback) {
    this.callbacks.push(callback);
  }
}

// Usage
const monitor = new RateMonitor(process.env.DCE_API_KEY);

monitor.onRateChange((changes) => {
  console.log('Significant rate changes detected:', changes);
  
  // Send notifications or update UI
  Object.entries(changes).forEach(([currency, data]) => {
    console.log(`${currency}: ${data.oldRate} → ${data.newRate} (${data.change}%)`);
  });
});

monitor.startMonitoring('USD', 60000); // Check every minute
```

## Historical Rates

### GET /api/exchange-rates/history

Retrieve historical exchange rate data for analysis and reporting.

#### Request

```bash
curl -X GET "https://api.dcepay.io/api/exchange-rates/history?fromCurrency=USD&toCurrency=ETH&startDate=2024-12-01&endDate=2024-12-19" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fromCurrency` | string | Yes | Source currency |
| `toCurrency` | string | Yes | Target currency |
| `startDate` | string | Yes | Start date (ISO 8601 format) |
| `endDate` | string | Yes | End date (ISO 8601 format) |
| `interval` | string | No | Data interval (hourly, daily, weekly) |

#### Response

```json
{
  "fromCurrency": "USD",
  "toCurrency": "ETH",
  "interval": "daily",
  "data": [
    {
      "date": "2024-12-01T00:00:00Z",
      "rate": "7.150",
      "volume": "1250000.00",
      "change": "0.5"
    },
    {
      "date": "2024-12-02T00:00:00Z",
      "rate": "7.182",
      "volume": "1320000.00",
      "change": "0.4"
    }
  ],
  "summary": {
    "minRate": "7.120",
    "maxRate": "7.200",
    "avgRate": "7.165",
    "totalChange": "2.1"
  }
}
```

## Rate Accuracy and Sources

### Data Sources

Exchange rates are sourced from multiple providers:

- **Primary Source**: AkashicPay API for real-time crypto rates
- **Secondary Sources**: Major exchanges and aggregators
- **Fiat Rates**: Central bank and financial institution data
- **Crypto Rates**: Real-time exchange data

### Rate Accuracy

- **Real-time Updates**: Rates updated every 30 seconds
- **Accuracy**: ±0.1% for major currency pairs
- **Latency**: <100ms for rate retrieval
- **Reliability**: 99.9% uptime guarantee

### Rate Validation

```javascript
class RateValidator {
  validateRate(rate, currency) {
    const validations = {
      'USD': { min: 0.0001, max: 1000000 },
      'ETH': { min: 0.000001, max: 10000 },
      'BTC': { min: 0.00000001, max: 100 },
      'EUR': { min: 0.0001, max: 1000000 }
    };
    
    const validation = validations[currency];
    if (!validation) {
      return true; // No validation for unknown currencies
    }
    
    const rateValue = parseFloat(rate);
    return rateValue >= validation.min && rateValue <= validation.max;
  }
  
  detectAnomalies(rates) {
    const anomalies = [];
    
    // Check for extreme changes
    Object.entries(rates).forEach(([currency, rate]) => {
      const rateValue = parseFloat(rate);
      
      // Flag rates that seem too high or too low
      if (rateValue <= 0 || rateValue > 1000000) {
        anomalies.push({
          currency,
          rate,
          reason: 'Extreme value'
        });
      }
    });
    
    return anomalies;
  }
}
```

## Error Handling

### Common Errors

| Error Code | Description | Resolution |
|------------|-------------|------------|
| `rate_not_found` | Exchange rate not available | Check supported currencies |
| `invalid_currency` | Currency code is invalid | Use valid currency codes |
| `rate_expired` | Rate data is outdated | Refresh rates |
| `service_unavailable` | Rate service unavailable | Retry with exponential backoff |

### Error Response Format

```json
{
  "error": "rate_not_found",
  "message": "Exchange rate not found for currency XYZ",
  "details": {
    "requestedCurrency": "XYZ",
    "supportedCurrencies": ["USD", "EUR", "ETH", "BTC"]
  }
}
```

### Retry Logic

```javascript
class RateRetryHandler {
  async getRatesWithRetry(baseCurrency, maxRetries = 3) {
    let attempt = 0;
    
    while (attempt < maxRetries) {
      try {
        return await getExchangeRates(baseCurrency);
      } catch (error) {
        attempt++;
        
        if (attempt >= maxRetries) {
          throw new Error(`Failed to fetch rates after ${maxRetries} attempts: ${error.message}`);
        }
        
        // Exponential backoff
        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }
}
```

## Best Practices

### 1. Rate Caching

```javascript
class RateCache {
  constructor() {
    this.cache = new Map();
    this.cacheTimeout = 30000; // 30 seconds
  }
  
  async getRates(baseCurrency) {
    const cacheKey = `rates_${baseCurrency}`;
    const cached = this.cache.get(cacheKey);
    
    if (cached && Date.now() - cached.timestamp < this.cacheTimeout) {
      return cached.data;
    }
    
    // Fetch fresh rates
    const rates = await getExchangeRates(baseCurrency);
    
    // Cache the result
    this.cache.set(cacheKey, {
      data: rates,
      timestamp: Date.now()
    });
    
    return rates;
  }
  
  invalidateCache(baseCurrency = null) {
    if (baseCurrency) {
      this.cache.delete(`rates_${baseCurrency}`);
    } else {
      this.cache.clear();
    }
  }
}
```

### 2. Rate Monitoring

```javascript
class RateAlertSystem {
  constructor() {
    this.alerts = new Map();
  }
  
  setAlert(currency, threshold, callback) {
    this.alerts.set(currency, { threshold, callback });
  }
  
  checkAlerts(currentRates, previousRates) {
    this.alerts.forEach((alert, currency) => {
      const currentRate = parseFloat(currentRates[currency]);
      const previousRate = parseFloat(previousRates[currency]);
      const change = ((currentRate - previousRate) / previousRate) * 100;
      
      if (Math.abs(change) >= alert.threshold) {
        alert.callback({
          currency,
          oldRate: previousRate,
          newRate: currentRate,
          change: change.toFixed(2)
        });
      }
    });
  }
}

// Usage
const alertSystem = new RateAlertSystem();

alertSystem.setAlert('ETH', 5.0, (alert) => {
  console.log(`Alert: ETH rate changed by ${alert.change}%`);
  // Send notification or trigger action
});
```

### 3. Currency Conversion Service

```javascript
class CurrencyConversionService {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.cache = new RateCache();
  }
  
  async convertAmount(amount, fromCurrency, toCurrency) {
    // Get current rates
    const rates = await this.cache.getRates(fromCurrency);
    
    // Create converter
    const converter = new CurrencyConverter(rates.rates);
    
    // Perform conversion
    return converter.convert(amount, fromCurrency, toCurrency);
  }
  
  async convertMultiple(conversions) {
    const results = [];
    
    for (const conversion of conversions) {
      try {
        const result = await this.convertAmount(
          conversion.amount,
          conversion.fromCurrency,
          conversion.toCurrency
        );
        
        results.push({
          ...conversion,
          result,
          success: true
        });
      } catch (error) {
        results.push({
          ...conversion,
          error: error.message,
          success: false
        });
      }
    }
    
    return results;
  }
}
```

## Integration Examples

### E-commerce Integration

```javascript
class EcommerceCurrencyService {
  constructor(apiKey) {
    this.conversionService = new CurrencyConversionService(apiKey);
  }
  
  async displayPricesInCurrency(products, targetCurrency) {
    const conversions = products.map(product => ({
      amount: product.price,
      fromCurrency: product.currency,
      toCurrency: targetCurrency
    }));
    
    const results = await this.conversionService.convertMultiple(conversions);
    
    return products.map((product, index) => ({
      ...product,
      displayPrice: results[index].result,
      displayCurrency: targetCurrency
    }));
  }
  
  async calculateTotalInCurrency(items, targetCurrency) {
    let total = 0;
    
    for (const item of items) {
      const convertedPrice = await this.conversionService.convertAmount(
        item.price * item.quantity,
        item.currency,
        targetCurrency
      );
      total += convertedPrice;
    }
    
    return total;
  }
}
```

### Financial Reporting

```javascript
class FinancialReportingService {
  constructor(apiKey) {
    this.apiKey = apiKey;
  }
  
  async generateCurrencyReport(transactions, baseCurrency = 'USD') {
    const currencyTotals = {};
    
    // Group transactions by currency
    transactions.forEach(tx => {
      const currency = tx.currency;
      if (!currencyTotals[currency]) {
        currencyTotals[currency] = 0;
      }
      currencyTotals[currency] += parseFloat(tx.amount);
    });
    
    // Convert all to base currency
    const conversionService = new CurrencyConversionService(this.apiKey);
    const convertedTotals = {};
    
    for (const [currency, amount] of Object.entries(currencyTotals)) {
      if (currency === baseCurrency) {
        convertedTotals[currency] = amount;
      } else {
        const converted = await conversionService.convertAmount(
          amount,
          currency,
          baseCurrency
        );
        convertedTotals[currency] = converted;
      }
    }
    
    return {
      baseCurrency,
      currencyTotals,
      convertedTotals,
      totalInBaseCurrency: Object.values(convertedTotals).reduce((sum, amount) => sum + amount, 0)
    };
  }
}
```

### Crypto Trading Integration

```javascript
class CryptoTradingService {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.monitor = new RateMonitor(apiKey);
  }
  
  async calculateArbitrageOpportunities() {
    const opportunities = [];
    
    // Get rates for major pairs
    const usdRates = await getExchangeRates('USD');
    const ethRates = await getExchangeRates('ETH');
    
    // Check for arbitrage opportunities
    const pairs = ['BTC', 'USDT', 'EUR'];
    
    for (const pair of pairs) {
      const usdToPair = usdRates.rates[pair];
      const pairToUsd = 1 / ethRates.rates[pair];
      
      const spread = Math.abs(usdToPair - pairToUsd) / usdToPair * 100;
      
      if (spread > 0.5) { // 0.5% threshold
        opportunities.push({
          pair,
          spread: spread.toFixed(2),
          usdRate: usdToPair,
          reverseRate: pairToUsd
        });
      }
    }
    
    return opportunities;
  }
  
  startArbitrageMonitoring() {
    this.monitor.onRateChange((changes) => {
      // Check for new arbitrage opportunities
      this.calculateArbitrageOpportunities().then(opportunities => {
        if (opportunities.length > 0) {
          console.log('Arbitrage opportunities detected:', opportunities);
        }
      });
    });
    
    this.monitor.startMonitoring('USD', 30000); // Check every 30 seconds
  }
}
```

---

*For more information about payment processing, see the [Deposits](https://docs.dcepay.io/docs/deposits) and [Transactions](https://docs.dcepay.io/docs/transactions) documentation.* 
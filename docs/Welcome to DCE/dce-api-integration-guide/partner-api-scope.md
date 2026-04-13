---
title: Partner API Scope
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

This file is the partner allowlist for external integrations.

## Scope Model

- `partner`: endpoint can be exposed to third-party integrators.
- `internal`: endpoint is internal-only and must not be enabled for partner keys.

## Endpoint Matrix

| Method | Route                                              | Scope    | Suggested Permission    |
| ------ | -------------------------------------------------- | -------- | ----------------------- |
| GET    | `/api/balance`                                     | partner  | `balance:read`          |
| GET    | `/api/transactions`                                | partner  | `transactions:read`     |
| GET    | `/api/deposits`                                    | internal | `admin:read`            |
| POST   | `/api/deposits`                                    | internal | `admin:write`           |
| GET    | `/api/withdrawals`                                 | partner  | `withdrawals:read`      |
| POST   | `/api/withdrawals`                                 | partner  | `withdrawals:write`     |
| GET    | `/api/settlements`                                 | internal | `admin:read`            |
| POST   | `/api/settlements`                                 | internal | `admin:write`           |
| GET    | `/api/deposit-address`                             | partner  | `deposit-address:read`  |
| POST   | `/api/deposit-address`                             | partner  | `deposit-address:write` |
| GET    | `/api/deposit-url`                                 | partner  | `deposit-url:read`      |
| POST   | `/api/deposit-url`                                 | partner  | `deposit-url:write`     |
| GET    | `/api/deposit-page`                                | partner  | `deposit-page:read`     |
| GET    | `/api/exchange-rates`                              | partner  | `exchange-rates:read`   |
| GET    | `/api/reconciliation`                              | internal | `reconciliation:read`   |
| POST   | `/api/reconciliation`                              | internal | `reconciliation:write`  |
| PATCH  | `/api/reconciliation`                              | internal | `reconciliation:write`  |
| GET    | `/api/users`                                       | internal | `admin:read`            |
| POST   | `/api/users`                                       | internal | `admin:write`           |
| GET    | `/api/users/{userId}/wallets`                      | internal | `admin:read`            |
| POST   | `/api/users/{userId}/wallets`                      | internal | `admin:write`           |
| POST   | `/api/users/{userId}/wallets/sync-balance`         | internal | `admin:write`           |
| GET    | `/api/merchants/{merchantId}/charges/config`       | internal | `admin:read`            |
| PUT    | `/api/merchants/{merchantId}/charges/config`       | internal | `admin:write`           |
| GET    | `/api/merchants/{merchantId}/settlement-addresses` | internal | `admin:read`            |
| POST   | `/api/merchants/{merchantId}/settlement-addresses` | internal | `admin:write`           |
| GET    | `/api/admin/background-jobs`                       | internal | `admin:read`            |
| POST   | `/api/admin/background-jobs`                       | internal | `admin:write`           |
| GET    | `/api/admin/system-settings`                       | internal | `admin:read`            |
| POST   | `/api/admin/system-settings`                       | internal | `admin:write`           |
| GET    | `/api/admin/system-settings/{key}`                 | internal | `admin:read`            |
| PUT    | `/api/admin/system-settings/{key}`                 | internal | `admin:write`           |
| DELETE | `/api/admin/system-settings/{key}`                 | internal | `admin:delete`          |
| POST   | `/api/admin/settlements/{settlementId}/approve`    | internal | `admin:write`           |
| GET    | `/api/admin/withdrawal-fees`                       | internal | `admin:read`            |
| POST   | `/api/admin/withdrawal-fees`                       | internal | `admin:write`           |
| PUT    | `/api/admin/withdrawal-fees`                       | internal | `admin:write`           |
| GET    | `/api/webhook/event`                               | internal | `admin:read`            |
| POST   | `/api/webhook/event`                               | internal | `webhook:write`         |
| POST   | `/api/webhook/chaingateway`                        | internal | `webhook:write`         |
| POST   | `/api/webhook/tronfuel`                            | internal | `webhook:write`         |

<br />

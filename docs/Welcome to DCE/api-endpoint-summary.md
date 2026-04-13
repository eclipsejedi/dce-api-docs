---
title: API Endpoint Summary
deprecated: false
hidden: false
metadata:
  robots: index
---
<br />

This summary reflects every route handler currently implemented in `src/app/api/**/route.ts`.

| Method | Route                                              |
| ------ | -------------------------------------------------- |
| GET    | `/api/balance`                                     |
| GET    | `/api/transactions`                                |
| GET    | `/api/deposits`                                    |
| POST   | `/api/deposits`                                    |
| GET    | `/api/withdrawals`                                 |
| POST   | `/api/withdrawals`                                 |
| GET    | `/api/settlements`                                 |
| POST   | `/api/settlements`                                 |
| GET    | `/api/deposit-address`                             |
| POST   | `/api/deposit-address`                             |
| GET    | `/api/deposit-url`                                 |
| POST   | `/api/deposit-url`                                 |
| GET    | `/api/deposit-page`                                |
| GET    | `/api/exchange-rates`                              |
| GET    | `/api/reconciliation`                              |
| POST   | `/api/reconciliation`                              |
| PATCH  | `/api/reconciliation`                              |
| GET    | `/api/users`                                       |
| POST   | `/api/users`                                       |
| GET    | `/api/users/{userId}/wallets`                      |
| POST   | `/api/users/{userId}/wallets`                      |
| POST   | `/api/users/{userId}/wallets/sync-balance`         |
| GET    | `/api/merchants/{merchantId}/charges/config`       |
| PUT    | `/api/merchants/{merchantId}/charges/config`       |
| GET    | `/api/merchants/{merchantId}/settlement-addresses` |
| POST   | `/api/merchants/{merchantId}/settlement-addresses` |
| GET    | `/api/admin/background-jobs`                       |
| POST   | `/api/admin/background-jobs`                       |
| GET    | `/api/admin/system-settings`                       |
| POST   | `/api/admin/system-settings`                       |
| GET    | `/api/admin/system-settings/{key}`                 |
| PUT    | `/api/admin/system-settings/{key}`                 |
| DELETE | `/api/admin/system-settings/{key}`                 |
| POST   | `/api/admin/settlements/{settlementId}/approve`    |
| GET    | `/api/admin/withdrawal-fees`                       |
| POST   | `/api/admin/withdrawal-fees`                       |
| PUT    | `/api/admin/withdrawal-fees`                       |
| GET    | `/api/webhook/event`                               |
| POST   | `/api/webhook/event`                               |
| POST   | `/api/webhook/chaingateway`                        |
| POST   | `/api/webhook/tronfuel`                            |

<br />

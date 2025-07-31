# Centralized Exchange (CEX) Integration

## Overview

This document outlines the data structures required for the Centralized Exchange (CEX) Integration feature. It includes:

- Exchange-specific configurations
- Real-time price feeds
- Historical trade data
- Exchange health monitoring
- API rate limit management

These models enable market data aggregation from over 100 CEXs.

---

## Entity Relationships

| Relationship                 | Cardinality | Description                                            |
| ---------------------------- | ----------- | ------------------------------------------------------ |
| Exchange ↔ ApiKey            | 1 → \*      | Each exchange can have multiple API keys.              |
| Exchange ↔ Ticker            | 1 → \*      | Each exchange has many ticker entries per market pair. |
| Exchange ↔ Trade             | 1 → \*      | Each exchange produces many trades over time.          |
| Exchange ↔ ExchangeHealthLog | 1 → \*      | Each exchange has many health logs.                    |
| Exchange ↔ RateLimitStatus   | 1 → \*      | Each exchange has multiple rate limit status entries.  |

---

## 📄 Exchange

> Stores configuration for each connected exchange (supports `CEX-001`).

| Field Name        | Type      | Required? | Format          | Description                          |
| ----------------- | --------- | --------- | --------------- | ------------------------------------ |
| exchange_id       | UUID      | Yes       | UUID            | Unique identifier.                   |
| name              | String    | Yes       | e.g., "Binance" | Name of the exchange.                |
| api_base_url      | String    | Yes       | URL             | Base REST API URL.                   |
| websocket_url     | String    | No        | URL             | WebSocket feed URL.                  |
| documentation_url | String    | No        | URL             | Official API documentation.          |
| is_active         | Boolean   | Yes       | true / false    | Enables or disables data collection. |
| config_params     | JSON      | No        | JSON object     | Exchange-specific config.            |
| created_at        | Timestamp | Yes       | ISO 8601        | Record creation time.                |
| updated_at        | Timestamp | Yes       | ISO 8601        | Last config update.                  |

---

## 📄 ApiKey

> Stores credentials for API access.

| Field Name  | Type      | Required? | Format            | Description             |
| ----------- | --------- | --------- | ----------------- | ----------------------- |
| key_id      | UUID      | Yes       | UUID              | Unique API key ID.      |
| exchange_id | UUID      | Yes       | UUID              | FK to `Exchange`.       |
| api_key     | String    | Yes       | Encrypted String  | Public key.             |
| secret_key  | String    | Yes       | Encrypted String  | Secret key for signing. |
| passphrase  | String    | No        | Encrypted String  | Optional passphrase.    |
| auth_method | Enum      | Yes       | API_KEY, OAUTH    | Authentication method.  |
| permissions | JSON      | No        | ["read", "trade"] | Permissions list.       |
| created_at  | Timestamp | Yes       | ISO 8601          | Creation time.          |

---

## 📄 Ticker

> Stores real-time price data (supports `CEX-002`).

| Field Name       | Type           | Required? | Format             | Description                     |
| ---------------- | -------------- | --------- | ------------------ | ------------------------------- |
| id               | BigInt         | Yes       | Auto-incrementing  | Unique ticker update ID.        |
| exchange_id      | UUID           | Yes       | UUID               | FK to `Exchange`.               |
| pair_symbol      | String         | Yes       | e.g., "BTC/USDT"   | Trading pair symbol.            |
| last_price       | Decimal(36,18) | Yes       | Numeric            | Last traded price.              |
| bid_price        | Decimal(36,18) | Yes       | Numeric            | Best price to sell.             |
| ask_price        | Decimal(36,18) | Yes       | Numeric            | Best price to buy.              |
| base_volume_24h  | Decimal(36,18) | No        | Numeric            | 24h base volume.                |
| quote_volume_24h | Decimal(36,18) | No        | Numeric            | 24h quote volume.               |
| timestamp        | Timestamp      | Yes       | ISO 8601 (with ms) | Time price update was received. |

---

## 📄 Trade

> Stores all executed trades (supports `CEX-003`).

| Field Name  | Type           | Required? | Format               | Description                      |
| ----------- | -------------- | --------- | -------------------- | -------------------------------- |
| trade_id    | String         | Yes       | Exchange-specific ID | Unique trade ID from exchange.   |
| exchange_id | UUID           | Yes       | UUID                 | FK to `Exchange`.                |
| pair_symbol | String         | Yes       | e.g., "BTC/USDT"     | Trading pair.                    |
| price       | Decimal(36,18) | Yes       | Numeric              | Trade price.                     |
| volume      | Decimal(36,18) | Yes       | Numeric              | Amount traded.                   |
| side        | Enum           | Yes       | BUY / SELL           | Trade direction.                 |
| executed_at | Timestamp      | Yes       | ISO 8601 (with ms)   | Trade execution time.            |
| logged_at   | Timestamp      | Yes       | ISO 8601             | Time trade was logged in system. |

---

## 📄 ExchangeHealthLog

> Logs exchange API health status (supports `CEX-004`).

| Field Name         | Type      | Required? | Format             | Description                             |
| ------------------ | --------- | --------- | ------------------ | --------------------------------------- |
| log_id             | UUID      | Yes       | UUID               | Unique health log ID.                   |
| exchange_id        | UUID      | Yes       | UUID               | FK to `Exchange`.                       |
| status             | Enum      | Yes       | UP, DOWN, DEGRADED | Operational state.                      |
| response_time_ms   | Integer   | No        | ms                 | API average response time.              |
| error_rate_percent | Float     | No        | 0.0 - 100.0        | Error rate percentage.                  |
| data_quality_score | Float     | No        | 0.0 - 1.0          | Score for data completeness.            |
| details            | Text      | No        | Free text          | Additional notes (e.g., error details). |
| checked_at         | Timestamp | Yes       | ISO 8601           | Health check timestamp.                 |

---

## 📄 RateLimitStatus

> Tracks API usage against rate limits (supports `CEX-005`).

| Field Name       | Type      | Required? | Format            | Description                      |
| ---------------- | --------- | --------- | ----------------- | -------------------------------- |
| id               | BigInt    | Yes       | Auto-incrementing | Unique entry ID.                 |
| exchange_id      | UUID      | Yes       | UUID              | FK to `Exchange`.                |
| endpoint_group   | String    | Yes       | e.g., "Public"    | API endpoint group.              |
| request_count    | Integer   | Yes       | Numeric           | Requests made in current window. |
| limit_per_window | Integer   | Yes       | Numeric           | Allowed max per window.          |
| window_seconds   | Integer   | Yes       | Seconds           | Length of rate limit window.     |
| reset_at         | Timestamp | Yes       | ISO 8601          | Time window resets.              |
| updated_at       | Timestamp | Yes       | ISO 8601          | Last updated.                    |

---

## 🔗 Relationships Summary

```text
Exchange [exchange_id (PK)]
 ├── 1 → * ApiKey              [key_id (PK), exchange_id (FK)]
 ├── 1 → * Ticker              [id (PK), exchange_id (FK)]
 ├── 1 → * Trade               [trade_id (PK), exchange_id (FK)]
 ├── 1 → * ExchangeHealthLog   [log_id (PK), exchange_id (FK)]
 └── 1 → * RateLimitStatus     [id (PK), exchange_id (FK)]

ApiKey [key_id (PK)]
 └── * → 1 Exchange            [exchange_id (FK)]

Ticker [id (PK)]
 └── * → 1 Exchange            [exchange_id (FK)]

Trade [trade_id (PK)]
 └── * → 1 Exchange            [exchange_id (FK)]

ExchangeHealthLog [log_id (PK)]
 └── * → 1 Exchange            [exchange_id (FK)]

RateLimitStatus [id (PK)]
 └── * → 1 Exchange            [exchange_id (FK)]
```

## 📌 Entities Summary

```text
Entity              Primary Key    Foreign Keys
------------------------------------------------------------
Exchange            exchange_id    —
ApiKey              key_id         exchange_id → Exchange.exchange_id
Ticker              id             exchange_id → Exchange.exchange_id
Trade               trade_id       exchange_id → Exchange.exchange_id
ExchangeHealthLog   log_id         exchange_id → Exchange.exchange_id
RateLimitStatus     id             exchange_id → Exchange.exchange_id
```

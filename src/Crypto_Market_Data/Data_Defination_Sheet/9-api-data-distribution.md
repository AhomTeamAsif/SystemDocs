# API & Data Distribution

This document is divided into two parts:

1. **Data Specification** – Defines the underlying database tables needed to store and serve the data for the API endpoints.
2. **API Response Examples** – Shows the structure of the JSON data that users will receive from the various API endpoints.

---

## 1. Data Specification

### Overview

This specification details the data models required to support the platform's public-facing APIs. It includes tables for:

- Master asset information
- Aggregated market statistics
- OHLCV (candlestick) data
- Foreign exchange rates
- Bulk data export system

These models are optimized for efficient read-access to serve high-traffic API endpoints.

### Relations Between Tables

- **Asset**: Source of truth for descriptive details of each supported cryptocurrency.
- **MarketStatistics**: Periodically updated table with aggregated metrics per asset.
- **Ohlcv**: Candlestick data for various timeframes, derived from trades.
- **FxRate**: Stores fiat currency conversion rates.
- **DataExportJob**: Tracks status and metadata of user export requests.

---

### 📄 Asset

> Master registry of all supported cryptocurrencies and tokens.

| Field Name       | Type   | Required? | Format                | Description                                     |
| ---------------- | ------ | --------- | --------------------- | ----------------------------------------------- |
| asset_id         | UUID   | Yes       | UUID                  | Unique identifier for the asset.                |
| name             | String | Yes       | e.g., "Bitcoin"       | Full name of the asset.                         |
| symbol           | String | Yes       | e.g., "BTC"           | Ticker symbol.                                  |
| description      | Text   | No        | Free text             | Brief description of the asset.                 |
| logo_url         | String | No        | URL                   | Link to asset logo.                             |
| contract_address | JSON   | No        | {"ethereum": "0x..."} | Maps chains to contract addresses.              |
| tags             | JSON   | No        | Array of strings      | Categorization tags (e.g., ["defi", "layer-1"]) |

---

### 📄 MarketStatistics

> Aggregated metrics per asset (supports `API-001` and `API-002`).

| Field Name               | Type           | Required? | Format   | Description                          |
| ------------------------ | -------------- | --------- | -------- | ------------------------------------ |
| asset_id                 | UUID           | Yes       | UUID     | Foreign key to `Asset`. Primary key. |
| current_price_usd        | Decimal(36,18) | Yes       | Numeric  | Aggregated price in USD.             |
| market_cap_usd           | Decimal(36,18) | Yes       | Numeric  | Market capitalization in USD.        |
| volume_24h_usd           | Decimal(36,18) | Yes       | Numeric  | Trading volume in last 24h.          |
| circulating_supply       | Decimal(36,18) | Yes       | Numeric  | Circulating supply.                  |
| price_change_24h_percent | Float          | Yes       | Numeric  | 24h price change in %.               |
| rank                     | Integer        | No        | Numeric  | Market cap ranking.                  |
| last_updated             | Timestamp      | Yes       | ISO 8601 | Last aggregation timestamp.          |

---

### 📄 Ohlcv

> Aggregated candlestick data per market and timeframe (supports `API-003`).

| Field Name  | Type           | Required? | Format     | Description                            |
| ----------- | -------------- | --------- | ---------- | -------------------------------------- |
| market_id   | UUID           | Yes       | UUID       | Foreign key to `Market`. Composite PK  |
| timeframe   | String         | Yes       | e.g., "1h" | Timeframe for the candle. Composite PK |
| open_time   | Timestamp      | Yes       | ISO 8601   | Start time of the candle. Composite PK |
| open_price  | Decimal(36,18) | Yes       | Numeric    | Price at the beginning.                |
| high_price  | Decimal(36,18) | Yes       | Numeric    | Highest price.                         |
| low_price   | Decimal(36,18) | Yes       | Numeric    | Lowest price.                          |
| close_price | Decimal(36,18) | Yes       | Numeric    | Price at the end.                      |
| volume      | Decimal(36,18) | Yes       | Numeric    | Volume traded during timeframe.        |

---

### 📄 FxRate

> Foreign exchange conversion rates.

| Field Name    | Type          | Required? | Format          | Description                 |
| ------------- | ------------- | --------- | --------------- | --------------------------- |
| currency_pair | String        | Yes       | e.g., "USD/EUR" | Currency pair. Primary key. |
| rate          | Decimal(18,8) | Yes       | Numeric         | Conversion rate.            |
| last_updated  | Timestamp     | Yes       | ISO 8601        | Timestamp of last update.   |

---

### 📄 DataExportJob

> Tracks user requests for bulk historical data (supports `API-006`).

| Field Name   | Type      | Required? | Format                                 | Description                                   |
| ------------ | --------- | --------- | -------------------------------------- | --------------------------------------------- |
| job_id       | UUID      | Yes       | UUID                                   | Unique job identifier.                        |
| user_id      | UUID      | Yes       | UUID                                   | Requesting user's ID.                         |
| data_type    | Enum      | Yes       | TRADES, OHLCV                          | Type of data to export.                       |
| status       | Enum      | Yes       | PENDING, PROCESSING, COMPLETED, FAILED | Job status                                    |
| params       | JSON      | Yes       | JSON object                            | Export parameters (e.g., market, date range). |
| download_url | String    | No        | URL                                    | URL to download file post-completion.         |
| created_at   | Timestamp | Yes       | ISO 8601                               | Request timestamp.                            |
| completed_at | Timestamp | No        | ISO 8601                               | Completion timestamp.                         |

---

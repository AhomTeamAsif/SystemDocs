# Price Aggregation & Analytics Engine

### Overview

This document specifies the data models required for the platform's advanced Price Aggregation & Analytics Engine. These structures are designed to support the calculation and storage of authoritative consensus prices, dynamic source weightings, analytical benchmarks like VWAP, and the auditable filtering of price outliers. The goal is to transform raw data from hundreds of CEX and DEX sources into reliable, high-quality market intelligence.

### AggregatedPrice

Stores the final, authoritative consensus price and fair value estimations for each asset, supporting features PA-001 and PA-007.

| Field Name           | Type           | Required? | Format                | Description                                                                               |
| -------------------- | -------------- | --------- | --------------------- | ----------------------------------------------------------------------------------------- |
| asset_id             | UUID           | Yes       | UUID                  | Foreign key to the Asset table. Part of the composite primary key.                        |
| timestamp            | Timestamp      | Yes       | ISO 8601              | The time for which this price is valid. Part of the composite primary key.                |
| price_type           | Enum           | Yes       | CONSENSUS, FAIR_VALUE | The type of price calculation. Part of the composite primary key.                         |
| price_usd            | Decimal(36,18) | Yes       | Numeric               | The calculated authoritative price in USD.                                                |
| confidence_score     | Float          | Yes       | 0.0 - 1.0             | A score indicating the system's confidence in the accuracy of the calculated price.       |
| contributing_sources | Integer        | Yes       | Numeric               | The number of individual data sources used in this calculation.                           |
| metadata             | JSON           | No        | JSON object           | Additional details, e.g., confidence intervals or factors used in fair value calculation. |

### SourceWeight

Stores the dynamically calculated weight for each trading venue, used to determine its influence on the consensus price, supporting PA-003.

| Field Name        | Type      | Required? | Format               | Description                                                                         |
| ----------------- | --------- | --------- | -------------------- | ----------------------------------------------------------------------------------- |
| source_id         | UUID      | Yes       | UUID                 | The ID of the source (e.g., CEX market_id or DEX pool_id). Part of a composite key. |
| source_type       | Enum      | Yes       | CEX_MARKET, DEX_POOL | The type of the data source. Part of a composite key.                               |
| calculated_at     | Timestamp | Yes       | ISO 8601             | The timestamp when this weight was calculated. Part of a composite key.             |
| weight            | Float     | Yes       | 0.0 - 1.0            | The final calculated weight for the source, representing its overall influence.     |
| volume_score      | Float     | No        | 0.0 - 1.0            | Component score based on the source's relative trading volume.                      |
| reliability_score | Float     | No        | 0.0 - 1.0            | Component score based on the source's historical uptime and data quality.           |
| details           | JSON      | No        | JSON object          | Additional metrics that contributed to the final weight.                            |

### PriceOutlierLog

An audit trail for the outlier detection algorithm, logging all filtered or flagged price points, supporting PA-002.

| Field Name        | Type           | Required? | Format               | Description                                                                       |
| ----------------- | -------------- | --------- | -------------------- | --------------------------------------------------------------------------------- |
| log_id            | UUID           | Yes       | UUID                 | Unique identifier for the log entry.                                              |
| source_id         | UUID           | Yes       | UUID                 | The ID of the source that produced the outlier (e.g., market_id or pool_id).      |
| source_type       | Enum           | Yes       | CEX_MARKET, DEX_POOL | The type of the data source.                                                      |
| outlier_price     | Decimal(36,18) | Yes       | Numeric              | The anomalous price that was detected.                                            |
| consensus_price   | Decimal(36,18) | Yes       | Numeric              | The market consensus price at the time of detection for comparison.               |
| deviation_percent | Float          | Yes       | Numeric              | The percentage deviation of the outlier from the consensus price.                 |
| action_taken      | Enum           | Yes       | FILTERED, FLAGGED    | The action taken by the system (e.g., excluded from calculation or just flagged). |
| detected_at       | Timestamp      | Yes       | ISO 8601             | The time the outlier was detected.                                                |

### CalculatedBenchmark

Stores the results of VWAP and TWAP calculations, supporting PA-006.

| Field Name     | Type           | Required? | Format     | Description                                                                              |
| -------------- | -------------- | --------- | ---------- | ---------------------------------------------------------------------------------------- |
| asset_id       | UUID           | Yes       | UUID       | Foreign key to the Asset table. Part of the composite primary key.                       |
| benchmark_type | Enum           | Yes       | VWAP, TWAP | The type of benchmark calculated. Part of the composite primary key.                     |
| start_time     | Timestamp      | Yes       | ISO 8601   | The beginning of the time window for the calculation. Part of the composite primary key. |
| end_time       | Timestamp      | Yes       | ISO 8601   | The end of the time window for the calculation.                                          |
| value          | Decimal(36,18) | Yes       | Numeric    | The calculated value of the benchmark (e.g., the VWAP price).                            |
| total_volume   | Decimal(36,18) | No        | Numeric    | The total volume during the window, primarily for VWAP.                                  |

### Relations between Tables

- **AggregatedPrice table:** This is a primary output table, storing the final consensus and fair value prices for each asset. It is the result of aggregating data from numerous sources, weighted by the SourceWeight table.
- **SourceWeight table:** This table stores a dynamically calculated "trust score" or weight for every individual data source (e.g., a specific market on a CEX or a liquidity pool on a DEX). These weights are a key input for the consensus price calculation in the AggregatedPrice table.
- **PriceOutlierLog table:** This table acts as an audit trail, recording every time a price point from a source is flagged or filtered out by the outlier detection algorithm. It provides transparency into the data cleansing process.
- **CalculatedBenchmark table:** This table stores the results of complex analytical calculations, such as VWAP and TWAP, for each asset over various timeframes. It provides standardized benchmarks for traders and analysts.

### Entities & Relationships

- **AggregatedPrice**
  - asset_id (PK, FK → Asset.asset_id)
  - timestamp (PK)
  - price_type (PK)
- **SourceWeight**
  - source_id (PK)
  - source_type (PK)
  - calculated_at (PK)
- **PriceOutlierLog**
  - log_id (PK)
- **CalculatedBenchmark**
  - asset_id (PK, FK → Asset.asset_id)
  - benchmark_type (PK)
  - start_time (PK)

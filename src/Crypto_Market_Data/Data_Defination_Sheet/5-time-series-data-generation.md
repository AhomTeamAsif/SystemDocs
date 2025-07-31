# Time-Series Data Generation

### Overview

This document outlines the data models required to support the platform's Time-Series Data Generation engine. These structures are designed to store standardized OHLCV data, detailed volume analytics, and definitions for custom client aggregations. The models also include mechanisms for tracking data quality and the application of interpolation to ensure the integrity and completeness of all time-series data.

### Relations between Tables

- **Ohlcv table**: The primary table for storing standardized candlestick data. It is populated by the time-series engine and serves as the foundation for most technical analysis. It includes a field to distinguish between real and interpolated data.
- **AggregatedVolume table**: This table provides a more granular view of trading volume, breaking it down by source (CEX market or DEX pool) and other dimensions. It supports detailed volume analysis that goes beyond the total volume in the Ohlcv table.
- **CustomAggregationDefinition table**: Belongs to a User (or an institutional client account). It stores the parameters for non-standard timeframes, allowing clients to save and reuse their custom aggregation rules.
- **TimeSeriesLog table**: An audit log that records significant events related to the time-series generation process, such as data quality validation failures or the application of interpolation algorithms.

### Ohlcv (Revised)

Stores aggregated Open, High, Low, Close, and Volume (candlestick) data for a specific market, supporting TS-001. This is a revised version of the previously defined table.

| Field Name       | Type           | Required? | Format             | Description                                                                                                               |
| ---------------- | -------------- | --------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| market_id        | UUID           | Yes       | UUID               | Foreign key to the CEX Market or a similar identifier for a globally aggregated asset. Part of the composite primary key. |
| timeframe        | String         | Yes       | e.g., "1h", "1d"   | The timeframe for the candle. Part of the composite primary key.                                                          |
| open_time        | Timestamp      | Yes       | ISO 8601           | The start time of the candle. Part of the composite primary key.                                                          |
| open_price       | Decimal(36,18) | Yes       | Numeric            | The price at the start of the timeframe.                                                                                  |
| high_price       | Decimal(36,18) | Yes       | Numeric            | The highest price during the timeframe.                                                                                   |
| low_price        | Decimal(36,18) | Yes       | Numeric            | The lowest price during the timeframe.                                                                                    |
| close_price      | Decimal(36,18) | Yes       | Numeric            | The price at the end of the timeframe.                                                                                    |
| volume           | Decimal(36,18) | Yes       | Numeric            | The total volume traded during the timeframe.                                                                             |
| data_source_type | Enum           | Yes       | REAL, INTERPOLATED | Indicates if the candle data is from real trades or filled by an interpolation algorithm (TS-008).                        |

### AggregatedVolume

Stores detailed volume breakdowns by source and time period, supporting the Volume Analysis feature (TS-003).

| Field Name       | Type           | Required? | Format               | Description                                                                                   |
| ---------------- | -------------- | --------- | -------------------- | --------------------------------------------------------------------------------------------- |
| source_id        | UUID           | Yes       | UUID                 | The ID of the source (e.g., CEX market_id or DEX pool_id). Part of the composite primary key. |
| source_type      | Enum           | Yes       | CEX_MARKET, DEX_POOL | The type of the data source. Part of the composite primary key.                               |
| timeframe        | String         | Yes       | e.g., "1h", "1d"     | The aggregation timeframe. Part of the composite primary key.                                 |
| timestamp        | Timestamp      | Yes       | ISO 8601             | The start time of the aggregation window. Part of the composite primary key.                  |
| total_volume     | Decimal(36,18) | Yes       | Numeric              | The total volume traded in quote currency.                                                    |
| buy_volume       | Decimal(36,18) | No        | Numeric              | The portion of total volume attributed to buy-side trades.                                    |
| sell_volume      | Decimal(36,18) | No        | Numeric              | The portion of total volume attributed to sell-side trades.                                   |
| trade_count      | Integer        | No        | Numeric              | The total number of trades during the period.                                                 |
| data_source_type | Enum           | Yes       | REAL, INTERPOLATED   | Indicates if the volume data is from real trades or interpolated (TS-008).                    |

### CustomAggregationDefinition

Stores client-defined parameters for custom aggregation windows, supporting TS-002.

| Field Name        | Type      | Required? | Format      | Description                                                                   |
| ----------------- | --------- | --------- | ----------- | ----------------------------------------------------------------------------- |
| definition_id     | UUID      | Yes       | UUID        | Unique identifier for the custom definition.                                  |
| user_id           | UUID      | Yes       | UUID        | Foreign key to the User or client account that owns this definition.          |
| definition_name   | String    | Yes       | Free text   | A user-friendly name for the custom window (e.g., "My Weekly Report Window"). |
| window_seconds    | Integer   | Yes       | Seconds     | The size of the aggregation window in seconds.                                |
| aggregation_rules | JSON      | No        | JSON object | Advanced rules for the aggregation, such as custom filters or calculations.   |
| created_at        | Timestamp | Yes       | ISO 8601    | Timestamp when the definition was created.                                    |

### TimeSeriesLog

An audit trail for events related to time-series data generation and quality control (TS-001, TS-008).

| Field Name   | Type      | Required? | Format                                 | Description                                                                                                     |
| ------------ | --------- | --------- | -------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| log_id       | UUID      | Yes       | UUID                                   | Unique identifier for the log entry.                                                                            |
| event_type   | Enum      | Yes       | INTERPOLATION_APPLIED, VALIDATION_FAIL | The type of event that occurred.                                                                                |
| target_table | String    | Yes       | e.g., "Ohlcv"                          | The name of the table where the event occurred.                                                                 |
| target_key   | JSON      | Yes       | JSON object                            | A JSON representation of the primary key of the affected record.                                                |
| details      | JSON      | No        | JSON object                            | Additional details about the event, such as the interpolation method used or the reason for validation failure. |
| created_at   | Timestamp | Yes       | ISO 8601                               | The time the log entry was created.                                                                             |

### Relationships

- Market [market_id (PK)] 1 => \* Ohlcv [market_id (FK)]
- The AggregatedVolume table has a conceptual polymorphic link to the CEX Market and DEX LiquidityPool tables via its source_id and source_type fields.
- User [user_id (PK)] 1 => \* CustomAggregationDefinition [definition_id (PK), user_id (FK)]

### Entities & Relationships

- **Ohlcv**
  - market_id (PK, FK)
  - timeframe (PK)
  - open_time (PK)
- **AggregatedVolume**
  - source_id (PK)
  - source_type (PK)
  - timeframe (PK)
  - timestamp (PK)
- **CustomAggregationDefinition**
  - definition_id (PK)
  - user_id (FK → User.user_id)
- **TimeSeriesLog**
  - log_id (PK)

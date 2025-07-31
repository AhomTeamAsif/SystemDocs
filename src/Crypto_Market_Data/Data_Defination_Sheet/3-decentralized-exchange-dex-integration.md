# Decentralized Exchange (DEX) Integration

## Overview

This document specifies the data models required to support the platform's Decentralized Exchange (DEX) Integration features. The design focuses on:

- Comprehensive protocol coverage
- Automated discovery of liquidity pools
- Accurate price calculations

These models handle the diverse and dynamic nature of DeFi, storing data on blockchains, DEX protocols, liquidity pools, and their real-time states.

---

## Entity Relationships

The following relationships logically separate static protocol configurations from dynamic, high-frequency market data:

### `Blockchain` ⇄ `DexProtocol`

- **Cardinality:** One-to-Many (1 → \*)
- **Description:** Each blockchain (e.g., Ethereum) can host multiple DEX protocols (e.g., Uniswap V3, Curve). Each DEX protocol is tied to a single blockchain.

---

### `DexProtocol` ⇄ `LiquidityPool`

- **Cardinality:** One-to-Many (1 → \*)
- **Description:** A DEX protocol consists of many liquidity pools. Each pool represents a trading pair and is uniquely associated with a protocol.

---

### `LiquidityPool` ⇄ `PoolState`

- **Cardinality:** One-to-One (1 → 1)
- **Description:**
  - `LiquidityPool` stores static market data (e.g., tokens, fee tier).
  - `PoolState` stores dynamic, frequently updated data (e.g., reserves, current tick).

---

### `LiquidityPool` ⇄ `DexSwap`

- **Cardinality:** One-to-Many (1 → \*)
- **Description:** A liquidity pool handles many swaps. Each swap is a unique entry in the `DexSwap` table, allowing complete historical trade tracking.

---

## Tables & Fields

### 🧱 Blockchain

Master list of supported blockchain networks.

| Field Name     | Type    | Required | Format           | Description                                   |
| -------------- | ------- | -------- | ---------------- | --------------------------------------------- |
| `chain_id`     | Integer | Yes      | e.g., 1, 137     | Unique numeric ID of the blockchain. **(PK)** |
| `name`         | String  | Yes      | e.g., "Ethereum" | Common name of the blockchain                 |
| `symbol`       | String  | Yes      | e.g., "ETH"      | Native currency symbol                        |
| `rpc_url`      | String  | Yes      | URL              | JSON-RPC endpoint                             |
| `explorer_url` | String  | Yes      | URL              | Base URL of the block explorer                |

---

### 🧱 DexProtocol

DEX protocol configurations and metadata.

| Field Name        | Type    | Required | Format             | Description                                      |
| ----------------- | ------- | -------- | ------------------ | ------------------------------------------------ |
| `protocol_id`     | UUID    | Yes      | UUID               | Internal DEX protocol ID **(PK)**                |
| `chain_id`        | Integer | Yes      | Numeric            | FK to `Blockchain.chain_id`                      |
| `name`            | String  | Yes      | e.g., "Uniswap V3" | DEX name                                         |
| `version`         | String  | No       | e.g., "3.0"        | Protocol version                                 |
| `factory_address` | String  | Yes      | Hex Address        | Address of the protocol’s factory smart contract |
| `amm_type`        | Enum    | Yes      | AMM Type Enum      | Type of AMM (e.g., CONSTANT_PRODUCT, STABLESWAP) |

---

### 🧱 LiquidityPool

Registry of all liquidity pools discovered.

| Field Name        | Type         | Required | Format            | Description                               |
| ----------------- | ------------ | -------- | ----------------- | ----------------------------------------- |
| `pool_id`         | UUID         | Yes      | UUID              | Internal ID for the pool **(PK)**         |
| `protocol_id`     | UUID         | Yes      | UUID              | FK to `DexProtocol.protocol_id`           |
| `pool_address`    | String       | Yes      | Hex Address       | Address of pool smart contract            |
| `token0_asset_id` | UUID         | Yes      | UUID              | FK to `Asset.asset_id` (token0)           |
| `token1_asset_id` | UUID         | Yes      | UUID              | FK to `Asset.asset_id` (token1)           |
| `fee_tier`        | Decimal(5,4) | No       | Numeric           | Swap fee percentage (e.g., 0.0030 = 0.3%) |
| `status`          | Enum         | Yes      | ACTIVE/UNVERIFIED | Pool verification status                  |
| `creation_block`  | BigInt       | Yes      | Numeric           | Block number when the pool was created    |

---

### 🧱 PoolState

Real-time state of a liquidity pool.

| Field Name           | Type      | Required | Format   | Description                                  |
| -------------------- | --------- | -------- | -------- | -------------------------------------------- |
| `pool_id`            | UUID      | Yes      | UUID     | FK to `LiquidityPool.pool_id` **(PK)**       |
| `reserve0`           | String    | Yes      | BigInt   | Total reserve of token0                      |
| `reserve1`           | String    | Yes      | BigInt   | Total reserve of token1                      |
| `total_liquidity`    | String    | No       | BigInt   | Total liquidity (optional for some AMMs)     |
| `current_tick`       | Integer   | No       | Numeric  | Current tick for concentrated liquidity AMMs |
| `last_updated_at`    | Timestamp | Yes      | ISO 8601 | Time of last update                          |
| `last_updated_block` | BigInt    | Yes      | Numeric  | Block number of last update                  |

---

### 🧱 DexSwap

Logs of every swap executed in pools.

| Field Name           | Type   | Required | Format      | Description                           |
| -------------------- | ------ | -------- | ----------- | ------------------------------------- |
| `tx_hash`            | String | Yes      | Hex String  | On-chain transaction hash **(PK)**    |
| `pool_id`            | UUID   | Yes      | UUID        | FK to `LiquidityPool.pool_id`         |
| `block_number`       | BigInt | Yes      | Numeric     | Block number of the swap              |
| `trader_address`     | String | Yes      | Hex Address | Address of swap initiator             |
| `amount_in`          | String | Yes      | BigInt      | Input token amount                    |
| `amount_out`         | String | Yes      | BigInt      | Output token amount                   |
| `token_in_asset_id`  | UUID   | Yes      | UUID        | FK to `Asset.asset_id` (input token)  |
| `token_out_asset_id` | UUID   | Yes      | UUID        | FK to `Asset.asset_id` (output token) |

---

## Relationships Summary

```text
Blockchain [chain_id (PK)]
 └── 1 → * DexProtocol [protocol_id (PK), chain_id (FK)]

DexProtocol [protocol_id (PK)]
 └── 1 → * LiquidityPool [pool_id (PK), protocol_id (FK)]

Asset [asset_id (PK)]
 ├── 1 → * LiquidityPool [token0_asset_id (FK)]
 └── 1 → * LiquidityPool [token1_asset_id (FK)]

LiquidityPool [pool_id (PK)]
 ├── 1 → 1 PoolState [pool_id (PK, FK)]
 └── 1 → * DexSwap [tx_hash (PK), pool_id (FK)]

Asset [asset_id (PK)]
 ├── 1 → * DexSwap [token_in_asset_id (FK)]
 └── 1 → * DexSwap [token_out_asset_id (FK)]
```

## ERD Summary

### 📦 Blockchain

- `chain_id` — **Primary Key (PK)**

---

### 📦 DexProtocol

- `protocol_id` — **Primary Key (PK)**
- `chain_id` — **Foreign Key (FK → Blockchain.chain_id)**

---

### 📦 LiquidityPool

- `pool_id` — **Primary Key (PK)**
- `protocol_id` — **Foreign Key (FK → DexProtocol.protocol_id)**
- `token0_asset_id` — **Foreign Key (FK → Asset.asset_id)**
- `token1_asset_id` — **Foreign Key (FK → Asset.asset_id)**

---

### 📦 PoolState

- `pool_id` — **Primary Key (PK), Foreign Key (FK → LiquidityPool.pool_id)**

---

### 📦 DexSwap

- `tx_hash` — **Primary Key (PK)**
- `pool_id` — **Foreign Key (FK → LiquidityPool.pool_id)**
- `token_in_asset_id` — **Foreign Key (FK → Asset.asset_id)**
- `token_out_asset_id` — **Foreign Key (FK → Asset.asset_id)**

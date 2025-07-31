# Blockchain Data Infrastructure

## Overview

This document details the data models that support the platform's Blockchain Data Infrastructure features. These structures are designed to manage direct blockchain connections, real-time block processing, event extraction, and chain reorganization handling across 100+ blockchain networks. The primary purpose of these models is to ensure reliable data collection from multiple blockchain networks, maintain data accuracy during chain reorganizations, and provide comprehensive monitoring of node health and blockchain events.

---

## Relations between tables

- **BlockchainNetwork & Other Entities:** The BlockchainNetwork table is the root of the hierarchy. It has one-to-many relationships with NodeConnection, BlockData, ChainReorganization, and ContractABI. This means a single defined network can have multiple node connections for redundancy, will produce a stream of blocks, can experience multiple reorganizations, and can have many known smart contracts.
- **NodeConnection & Dependent Data:** The NodeConnection table has a many-to-one relationship with BlockchainNetwork, as multiple nodes are used to monitor a single network. It also has one-to-many relationships with BlockData (the specific node that fetched a block is recorded) and NodeHealthMetric (a time-series of health data is stored for each node).
- **BlockData, TransactionData, & EventLog:** These tables form a clear hierarchy. BlockData has a one-to-many relationship with TransactionData (a block contains many transactions). TransactionData in turn has a one-to-many relationship with EventLog (a transaction can emit multiple events). For efficient querying, EventLog also maintains a direct foreign key to BlockData. All three tables link back to the BlockchainNetwork they belong to.
- **ContractABI & EventLog:** A one-to-many relationship exists where one ContractABI record can be used to decode many events in the EventLog table. The link is established by matching the EventLog.contract_address and EventLog.network_id to the composite primary key of the ContractABI table. This is crucial for event decoding.
- **ChainReorganization & BlockchainNetwork:** The ChainReorganization table has a many-to-one relationship with BlockchainNetwork. This allows the system to log every reorg event that occurs on a specific network, tracking the orphaned blocks and the new canonical chain for data correction purposes.

---

## BlockchainNetwork

Defines supported blockchain networks and their configuration parameters, supporting feature BC-001.

| Field Name                    | Type      | Required? | Format                      | Description                                            |
| ----------------------------- | --------- | --------- | --------------------------- | ------------------------------------------------------ |
| network_id                    | UUID      | Yes       | UUID                        | Primary key for the blockchain network                 |
| network_name                  | String    | Yes       | e.g., "Ethereum", "Polygon" | Human-readable network name                            |
| chain_id                      | Integer   | Yes       | Positive integer            | Unique chain identifier (e.g., 1 for Ethereum mainnet) |
| network_type                  | Enum      | Yes       | EVM, NON_EVM                | Type of blockchain network                             |
| native_currency               | String    | Yes       | e.g., "ETH", "BNB"          | Native currency symbol                                 |
| block_time_seconds            | Integer   | Yes       | Positive integer            | Average time between blocks                            |
| confirmation_blocks           | Integer   | Yes       | Positive integer            | Blocks required for transaction confirmation           |
| rpc_endpoints                 | JSON      | Yes       | JSON array                  | List of RPC endpoint URLs                              |
| websocket_endpoints           | JSON      | Yes       | JSON array                  | List of WebSocket endpoint URLs                        |
| explorer_url                  | String    | No        | URL                         | Block explorer base URL                                |
| is_testnet                    | Boolean   | Yes       | true/false                  | Whether this is a testnet                              |
| is_active                     | Boolean   | Yes       | true/false                  | Whether network monitoring is active                   |
| gas_token_symbol              | String    | No        | e.g., "ETH", "MATIC"        | Symbol for gas/fee token                               |
| decimals                      | Integer   | Yes       | Positive integer            | Decimal places for native currency                     |
| supports_logs                 | Boolean   | Yes       | true/false                  | Whether network supports event logs                    |
| supports_trace                | Boolean   | Yes       | true/false                  | Whether network supports transaction tracing           |
| max_block_gas_limit           | BigInt    | No        | Positive integer            | Maximum gas limit per block                            |
| priority                      | Integer   | Yes       | 1-100                       | Processing priority (higher = more important)          |
| health_check_interval_seconds | Integer   | Yes       | Positive integer            | Seconds between health checks                          |
| created_at                    | Timestamp | Yes       | ISO 8601                    | When network was added                                 |
| updated_at                    | Timestamp | Yes       | ISO 8601                    | When network configuration was last updated            |

**Keys:**

- Primary Key: `network_id`

---

## NodeConnection

Manages individual node connections for blockchain networks, supporting features BC-001 and BC-004.

| Field Name                  | Type      | Required? | Format                                           | Description                                      |
| --------------------------- | --------- | --------- | ------------------------------------------------ | ------------------------------------------------ |
| connection_id               | UUID      | Yes       | UUID                                             | Primary key for the node connection              |
| network_id                  | UUID      | Yes       | UUID                                             | Foreign key linking to BlockchainNetwork         |
| node_url                    | String    | Yes       | URL                                              | Full URL of the blockchain node                  |
| connection_type             | Enum      | Yes       | RPC_HTTP, RPC_HTTPS, WEBSOCKET, WEBSOCKET_SECURE | Type of connection                               |
| node_provider               | String    | No        | e.g., "Infura", "Alchemy"                        | Node provider name                               |
| is_primary                  | Boolean   | Yes       | true/false                                       | Whether this is the primary node for the network |
| priority                    | Integer   | Yes       | 1-10                                             | Connection priority for failover (1 = highest)   |
| status                      | Enum      | Yes       | ACTIVE, INACTIVE, FAILED, MAINTENANCE            | Current connection status                        |
| last_block_number           | BigInt    | No        | Non-negative integer                             | Last processed block number                      |
| last_successful_request     | Timestamp | No        | ISO 8601                                         | When last successful request was made            |
| last_failed_request         | Timestamp | No        | ISO 8601                                         | When last failed request occurred                |
| consecutive_failures        | Integer   | Yes       | Non-negative integer                             | Number of consecutive failures                   |
| max_consecutive_failures    | Integer   | Yes       | Positive integer                                 | Maximum failures before marking as failed        |
| response_time_ms            | Integer   | No        | Milliseconds                                     | Average response time                            |
| requests_per_second_limit   | Integer   | No        | Positive integer                                 | Rate limit for this node                         |
| current_requests_per_second | Float     | No        | Non-negative float                               | Current request rate                             |
| health_score                | Integer   | Yes       | 0-100                                            | Overall health score (100 = excellent)           |
| sync_status                 | Enum      | Yes       | SYNCED, SYNCING, BEHIND, UNKNOWN                 | Node synchronization status                      |
| peer_count                  | Integer   | No        | Non-negative integer                             | Number of connected peers                        |
| is_archive_node             | Boolean   | Yes       | true/false                                       | Whether node maintains full historical data      |
| supports_websocket          | Boolean   | Yes       | true/false                                       | Whether node supports WebSocket connections      |
| api_key_required            | Boolean   | Yes       | true/false                                       | Whether API key is required                      |
| connection_config           | JSON      | No        | JSON object                                      | Additional connection configuration              |
| created_at                  | Timestamp | Yes       | ISO 8601                                         | When connection was established                  |
| updated_at                  | Timestamp | Yes       | ISO 8601                                         | When connection was last updated                 |

**Keys:**

- Primary Key: `connection_id`
- Foreign Key: `network_id` references `BlockchainNetwork.network_id`

---

## BlockData

Stores processed block information from all blockchain networks, supporting feature BC-002.

| Field Name          | Type      | Required? | Format                        | Description                                             |
| ------------------- | --------- | --------- | ----------------------------- | ------------------------------------------------------- |
| block_id            | UUID      | Yes       | UUID                          | Primary key for the block record                        |
| network_id          | UUID      | Yes       | UUID                          | Foreign key linking to BlockchainNetwork                |
| block_number        | BigInt    | Yes       | Non-negative integer          | Block number on the blockchain                          |
| block_hash          | String    | Yes       | Hex string                    | Unique hash of the block                                |
| parent_hash         | String    | Yes       | Hex string                    | Hash of the parent block                                |
| timestamp           | Timestamp | Yes       | ISO 8601                      | Block timestamp from blockchain                         |
| processed_at        | Timestamp | Yes       | ISO 8601                      | When block was processed by our system                  |
| processing_time_ms  | Integer   | Yes       | Milliseconds                  | Time taken to process this block                        |
| transaction_count   | Integer   | Yes       | Non-negative integer          | Number of transactions in block                         |
| gas_used            | BigInt    | No        | Non-negative integer          | Total gas used in block                                 |
| gas_limit           | BigInt    | No        | Non-negative integer          | Gas limit for the block                                 |
| base_fee_per_gas    | BigInt    | No        | Non-negative integer          | Base fee per gas (EIP-1559 networks)                    |
| difficulty          | BigInt    | No        | Non-negative integer          | Block difficulty                                        |
| total_difficulty    | BigInt    | No        | Non-negative integer          | Total chain difficulty at this block                    |
| miner_address       | String    | No        | Address                       | Address of block miner/validator                        |
| extra_data          | String    | No        | Hex string                    | Additional data included in block                       |
| block_size_bytes    | Integer   | No        | Positive integer              | Size of block in bytes                                  |
| uncle_count         | Integer   | No        | Non-negative integer          | Number of uncle blocks (if applicable)                  |
| is_canonical        | Boolean   | Yes       | true/false                    | Whether block is part of canonical chain                |
| is_processed        | Boolean   | Yes       | true/false                    | Whether all transactions have been processed            |
| event_count         | Integer   | Yes       | Non-negative integer          | Number of events extracted from block                   |
| swap_count          | Integer   | Yes       | Non-negative integer          | Number of DEX swaps in block                            |
| transfer_count      | Integer   | Yes       | Non-negative integer          | Number of token transfers in block                      |
| reorg_detected      | Boolean   | Yes       | true/false                    | Whether this block was involved in a reorganization     |
| confirmation_status | Enum      | Yes       | PENDING, CONFIRMED, FINALIZED | Block confirmation status                               |
| node_connection_id  | UUID      | Yes       | UUID                          | Foreign key to NodeConnection that processed this block |
| raw_block_data      | JSON      | No        | JSON object                   | Raw block data from node (for debugging)                |

**Keys:**

- Primary Key: `block_id`
- Foreign Key: `network_id` references `BlockchainNetwork.network_id`
- Foreign Key: `node_connection_id` references `NodeConnection.connection_id`

---

## TransactionData

Contains individual transaction details extracted from blocks, supporting feature BC-002.

| Field Name                  | Type      | Required? | Format                   | Description                                         |
| --------------------------- | --------- | --------- | ------------------------ | --------------------------------------------------- |
| transaction_id              | UUID      | Yes       | UUID                     | Primary key for the transaction record              |
| block_id                    | UUID      | Yes       | UUID                     | Foreign key linking to BlockData                    |
| network_id                  | UUID      | Yes       | UUID                     | Foreign key linking to BlockchainNetwork            |
| transaction_hash            | String    | Yes       | Hex string               | Unique transaction hash                             |
| transaction_index           | Integer   | Yes       | Non-negative integer     | Position of transaction in block                    |
| from_address                | String    | Yes       | Address                  | Sender address                                      |
| to_address                  | String    | No        | Address                  | Recipient address (null for contract creation)      |
| value                       | BigInt    | Yes       | Non-negative integer     | Amount transferred in native currency               |
| gas_limit                   | BigInt    | Yes       | Positive integer         | Gas limit set for transaction                       |
| gas_used                    | BigInt    | No        | Non-negative integer     | Actual gas used                                     |
| gas_price                   | BigInt    | No        | Non-negative integer     | Gas price paid                                      |
| max_fee_per_gas             | BigInt    | No        | Non-negative integer     | Max fee per gas (EIP-1559)                          |
| max_priority_fee_per_gas    | BigInt    | No        | Non-negative integer     | Max priority fee per gas (EIP-1559)                 |
| nonce                       | BigInt    | Yes       | Non-negative integer     | Sender's transaction nonce                          |
| transaction_type            | Integer   | No        | 0-2                      | Transaction type (0=legacy, 1=EIP-2930, 2=EIP-1559) |
| status                      | Enum      | Yes       | SUCCESS, FAILED, PENDING | Transaction execution status                        |
| block_timestamp             | Timestamp | Yes       | ISO 8601                 | Timestamp when transaction was mined                |
| is_contract_creation        | Boolean   | Yes       | true/false               | Whether transaction creates a contract              |
| created_contract_address    | String    | No        | Address                  | Address of created contract (if applicable)         |
| input_data                  | Text      | No        | Hex string               | Transaction input data                              |
| logs_count                  | Integer   | Yes       | Non-negative integer     | Number of logs generated                            |
| internal_transactions_count | Integer   | Yes       | Non-negative integer     | Number of internal transactions                     |
| is_dex_interaction          | Boolean   | Yes       | true/false               | Whether transaction interacts with DEX              |
| swap_detected               | Boolean   | Yes       | true/false               | Whether transaction contains DEX swap               |
| token_transfers_count       | Integer   | Yes       | Non-negative integer     | Number of token transfers                           |
| error_message               | Text      | No        | Free text                | Error message if transaction failed                 |
| cumulative_gas_used         | BigInt    | No        | Non-negative integer     | Cumulative gas used up to this transaction          |
| effective_gas_price         | BigInt    | No        | Non-negative integer     | Effective gas price paid                            |

**Keys:**

- Primary Key: `transaction_id`
- Foreign Key: `block_id` references `BlockData.block_id`
- Foreign Key: `network_id` references `BlockchainNetwork.network_id`

---

## EventLog

Stores decoded smart contract events, particularly DEX-related events, supporting feature BC-003.

| Field Name        | Type      | Required? | Format                                            | Description                                |
| ----------------- | --------- | --------- | ------------------------------------------------- | ------------------------------------------ |
| event_id          | UUID      | Yes       | UUID                                              | Primary key for the event log              |
| transaction_id    | UUID      | Yes       | UUID                                              | Foreign key linking to TransactionData     |
| block_id          | UUID      | Yes       | UUID                                              | Foreign key linking to BlockData           |
| network_id        | UUID      | Yes       | UUID                                              | Foreign key linking to BlockchainNetwork   |
| log_index         | Integer   | Yes       | Non-negative integer                              | Position of log within transaction         |
| contract_address  | String    | Yes       | Address                                           | Address of contract that emitted the event |
| event_signature   | String    | Yes       | Hex string                                        | Keccak256 hash of event signature          |
| event_name        | String    | No        | e.g., "Swap", "Transfer"                          | Human-readable event name                  |
| topic_0           | String    | Yes       | Hex string                                        | First topic (event signature)              |
| topic_1           | String    | No        | Hex string                                        | Second topic (first indexed parameter)     |
| topic_2           | String    | No        | Hex string                                        | Third topic (second indexed parameter)     |
| topic_3           | String    | No        | Hex string                                        | Fourth topic (third indexed parameter)     |
| data              | Text      | Yes       | Hex string                                        | Non-indexed event data                     |
| decoded_data      | JSON      | No        | JSON object                                       | Decoded event parameters                   |
| event_type        | Enum      | Yes       | SWAP, TRANSFER, MINT, BURN, SYNC, APPROVAL, OTHER | Categorized event type                     |
| protocol_name     | String    | No        | e.g., "Uniswap V3"                                | DEX/protocol name                          |
| protocol_version  | String    | No        | e.g., "v3"                                        | Protocol version                           |
| is_dex_event      | Boolean   | Yes       | true/false                                        | Whether event is from a DEX protocol       |
| token_0_address   | String    | No        | Address                                           | First token address (for swap events)      |
| token_1_address   | String    | No        | Address                                           | Second token address (for swap events)     |
| amount_0          | BigInt    | No        | Integer                                           | Amount of first token                      |
| amount_1          | BigInt    | No        | Integer                                           | Amount of second token                     |
| sender_address    | String    | No        | Address                                           | Address that initiated the swap            |
| recipient_address | String    | No        | Address                                           | Address that received the output           |
| liquidity_delta   | BigInt    | No        | Integer                                           | Change in liquidity (for mint/burn events) |
| sqrt_price_x96    | BigInt    | No        | Integer                                           | Square root price (for AMM events)         |
| tick              | Integer   | No        | Integer                                           | Current tick (for concentrated liquidity)  |
| fee_tier          | Integer   | No        | Non-negative integer                              | Fee tier for the pool                      |
| block_timestamp   | Timestamp | Yes       | ISO 8601                                          | Timestamp when event was emitted           |
| is_removed        | Boolean   | Yes       | true/false                                        | Whether log was removed due to reorg       |

**Keys:**

- Primary Key: `event_id`
- Foreign Key: `transaction_id` references `TransactionData.transaction_id`
- Foreign Key: `block_id` references `BlockData.block_id`
- Foreign Key: `network_id` references `BlockchainNetwork.network_id`
- Foreign Key: `(contract_address, network_id)` references `ContractABI.(contract_address, network_id)`

---

## ContractABI

Maintains ABI information for smart contracts to enable event decoding, supporting feature BC-003.

| Field Name             | Type      | Required? | Format                                 | Description                                                     |
| ---------------------- | --------- | --------- | -------------------------------------- | --------------------------------------------------------------- |
| contract_address       | String    | Yes       | Address                                | Contract address. Primary key                                   |
| network_id             | UUID      | Yes       | UUID                                   | Foreign key linking to BlockchainNetwork. Part of composite key |
| contract_name          | String    | No        | Free text                              | Human-readable contract name                                    |
| abi_json               | JSON      | Yes       | JSON array                             | Complete ABI definition                                         |
| abi_hash               | String    | Yes       | Hash                                   | SHA-256 hash of ABI JSON                                        |
| is_proxy               | Boolean   | Yes       | true/false                             | Whether contract is a proxy                                     |
| implementation_address | String    | No        | Address                                | Implementation address (for proxy contracts)                    |
| proxy_type             | Enum      | No        | EIP1967, EIP1822, TRANSPARENT, BEACON  | Type of proxy pattern                                           |
| verified               | Boolean   | Yes       | true/false                             | Whether ABI is verified                                         |
| verification_source    | String    | No        | e.g., "Etherscan"                      | Source of ABI verification                                      |
| protocol_category      | Enum      | No        | DEX, LENDING, BRIDGE, NFT, DEFI, OTHER | Protocol category                                               |
| created_at             | Timestamp | Yes       | ISO 8601                               | When ABI was added                                              |
| updated_at             | Timestamp | Yes       | ISO 8601                               | When ABI was last updated                                       |

**Keys:**

- Primary Key: `(contract_address, network_id)`
- Foreign Key: `network_id` references `BlockchainNetwork.network_id`

---

## ChainReorganization

Tracks blockchain reorganization events and their impact, supporting feature BC-005.

| Field Name                    | Type      | Required? | Format                                  | Description                              |
| ----------------------------- | --------- | --------- | --------------------------------------- | ---------------------------------------- |
| reorg_id                      | UUID      | Yes       | UUID                                    | Primary key for the reorganization event |
| network_id                    | UUID      | Yes       | UUID                                    | Foreign key linking to BlockchainNetwork |
| detected_at                   | Timestamp | Yes       | ISO 8601                                | When reorganization was detected         |
| reorg_depth                   | Integer   | Yes       | Positive integer                        | Number of blocks affected by reorg       |
| old_chain_tip_hash            | String    | Yes       | Hex string                              | Hash of old chain tip before reorg       |
| new_chain_tip_hash            | String    | Yes       | Hex string                              | Hash of new chain tip after reorg        |
| fork_block_number             | BigInt    | Yes       | Non-negative integer                    | Block number where chains diverged       |
| fork_block_hash               | String    | Yes       | Hex string                              | Hash of the fork point block             |
| orphaned_blocks               | JSON      | Yes       | JSON array                              | List of orphaned block hashes            |
| new_blocks                    | JSON      | Yes       | JSON array                              | List of new canonical block hashes       |
| affected_transactions         | JSON      | No        | JSON array                              | Transactions that were invalidated       |
| reprocessing_status           | Enum      | Yes       | PENDING, IN_PROGRESS, COMPLETED, FAILED | Status of data reprocessing              |
| reprocessing_started_at       | Timestamp | No        | ISO 8601                                | When reprocessing began                  |
| reprocessing_completed_at     | Timestamp | No        | ISO 8601                                | When reprocessing finished               |
| data_corrections_made         | Integer   | Yes       | Non-negative integer                    | Number of data corrections applied       |
| downstream_notifications_sent | Boolean   | Yes       | true/false                              | Whether downstream systems were notified |
| severity                      | Enum      | Yes       | LOW, MEDIUM,HIGH, CRITICAL              | Severity of the reorganization           |
| impact_assessment             | Text      | No        | Free text                               | Assessment of reorg impact               |
| resolution_notes              | Text      | No        | Free text                               | Notes on how reorg was handled           |

**Keys:**

- Primary Key: `reorg_id`
- Foreign Key: `network_id` references `BlockchainNetwork.network_id`

---

## NodeHealthMetric

Time-series data tracking node health and performance, supporting feature BC-004.

| Field Name           | Type      | Required? | Format               | Description                                          |
| -------------------- | --------- | --------- | -------------------- | ---------------------------------------------------- |
| connection_id        | UUID      | Yes       | UUID                 | Foreign key to NodeConnection. Part of composite key |
| timestamp            | Timestamp | Yes       | ISO 8601             | When metrics were recorded. Part of composite key    |
| is_responsive        | Boolean   | Yes       | true/false           | Whether node is responding to requests               |
| response_time_ms     | Integer   | No        | Milliseconds         | Response time for health check                       |
| current_block_number | BigInt    | No        | Non-negative integer | Current block number reported by node                |
| blocks_behind        | Integer   | No        | Non-negative integer | How many blocks behind the node is                   |
| peer_count           | Integer   | No        | Non-negative integer | Number of connected peers                            |
| sync_percentage      | Float     | No        | 0.0 - 100.0          | Synchronization percentage                           |
| requests_successful  | Integer   | Yes       | Non-negative integer | Successful requests in measurement window            |
| requests_failed      | Integer   | Yes       | Non-negative integer | Failed requests in measurement window                |
| error_rate           | Float     | Yes       | 0.0 - 1.0            | Error rate for the measurement period                |
| cpu_usage            | Float     | No        | 0.0 - 100.0          | CPU usage percentage (if available)                  |
| memory_usage         | Float     | No        | 0.0 - 100.0          | Memory usage percentage (if available)               |
| disk_usage           | Float     | No        | 0.0 - 100.0          | Disk usage percentage (if available)                 |
| network_latency_ms   | Integer   | No        | Milliseconds         | Network latency to node                              |
| health_score         | Integer   | Yes       | 0-100                | Calculated health score                              |

**Keys:**

- Primary Key: `(connection_id, timestamp)`
- Foreign Key: `connection_id` references `NodeConnection.connection_id`

---

## Summary of Relationships

- `BlockchainNetwork [network_id (PK)] 1 => * NodeConnection [network_id (FK)]`
- `BlockchainNetwork [network_id (PK)] 1 => * BlockData [network_id (FK)]`
- `BlockchainNetwork [network_id (PK)] 1 => * TransactionData [network_id (FK)]`
- `BlockchainNetwork [network_id (PK)] 1 => * EventLog [network_id (FK)]`
- `BlockchainNetwork [network_id (PK)] 1 => * ContractABI [network_id (FK)]`
- `BlockchainNetwork [network_id (PK)] 1 => * ChainReorganization [network_id (FK)]`
- `NodeConnection [connection_id (PK)] 1 => * BlockData [node_connection_id (FK)]`
- `NodeConnection [connection_id (PK)] 1 => * NodeHealthMetric [connection_id (FK)]`
- `BlockData [block_id (PK)] 1 => * TransactionData [block_id (FK)]`
- `BlockData [block_id (PK)] 1 => * EventLog [block_id (FK)]`
- `TransactionData [transaction_id (PK)] 1 => * EventLog [transaction_id (FK)]`
- `ContractABI [contract_address, network_id (PK)] 1 => * EventLog [contract_address, network_id (FK)]`

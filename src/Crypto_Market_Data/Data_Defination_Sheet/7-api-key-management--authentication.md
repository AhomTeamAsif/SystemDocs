# API Key Management & Authentication

## Overview

This document specifies the data models required for the platform's API Key Management and Authentication features. It outlines the structures for managing user-facing API keys, defining granular permissions, logging access requests, and securing internal service-to-service communication. These tables form the backbone for secure, controlled, and auditable access to all platform services.

## Entity Relationships

This section provides a detailed explanation of the relationships between the data entities that constitute the API Key Management and Permissions system.

### User <=> ApiKey

- **Cardinality**: One-to-Many (1 to \*)
- **Description**: This is a core relationship where a single User can generate and manage multiple ApiKey records. Each key is distinctly owned by one user, creating a clear line of accountability. This allows users to create separate keys for different applications (e.g., one for a mobile widget, another for a trading bot), each with its own lifecycle and permissions, while all being tied back to the master user account. The link is established through a user_id foreign key in the ApiKey table.

### ApiKey <=> PermissionSet

- **Cardinality**: Many-to-One (\* to 1)
- **Description**: Each ApiKey must be assigned exactly one PermissionSet. This relationship dictates the scope of actions the key is authorized to perform. Multiple API keys, even those belonging to different users, can be assigned the same PermissionSet (e.g., many keys can use the system's "Read-Only" set). This model allows for the efficient reuse of permission configurations and simplifies auditing.

### PermissionSet <=> Permission (via PermissionSet_Mapping)

- **Cardinality**: Many-to-Many (_ to _)
- **Description**: This relationship defines the granular composition of a permission set. A single PermissionSet can include a collection of many individual Permission records. Conversely, a single Permission (like read_price_feeds) can be a component of multiple different PermissionSets. This many-to-many link is implemented via the PermissionSet_Mapping junction table, which contains pairs of permission_set_id and permission_id, providing maximum flexibility in defining access control policies.

### User <=> PermissionSet

- **Cardinality**: One-to-Many (1 to \*)
- **Description**: Users have the ability to create their own custom PermissionSet records. This is a one-to-many relationship where a User can be the author of multiple permission sets, but each custom set is owned by only one user. System-defined sets (e.g., "Full Access," "Read-Only") would not be linked to a specific user, allowing for a clear distinction between global templates and user-specific configurations.

### ApiKey <=> ApiRequestLog

- **Cardinality**: One-to-Many (1 to \*)
- **Description**: This relationship is essential for security auditing and usage monitoring. A single ApiKey will be used to authenticate numerous requests over its lifetime, resulting in a large number of entries in the ApiRequestLog table. Each log entry is tied to exactly one ApiKey, providing a comprehensive and non-repudiable trail of every action attempted or performed with that key.

### InternalService

- **Description**: The InternalService table is a distinct entity designed for machine-to-machine (M2M) authentication within the platform's microservices architecture. While it serves a similar purpose to ApiKey (authentication and authorization), it is decoupled from the user-centric tables. It manages credentials for backend services, ensuring that inter-service communication is secure and authenticated independently of user sessions or user-generated API keys.

## Data Models

### ApiKey

Stores user-generated API keys and their associated configurations, supporting features AK-001, AK-002, and AK-004.

| Field Name        | Type      | Required? | Format                   | Description                                                                                          |
| ----------------- | --------- | --------- | ------------------------ | ---------------------------------------------------------------------------------------------------- |
| api_key_id        | UUID      | Yes       | UUID                     | Unique identifier for the API key.                                                                   |
| user_id           | UUID      | Yes       | UUID                     | Foreign key linking to the User table.                                                               |
| permission_set_id | UUID      | Yes       | UUID                     | Foreign key linking to the PermissionSet table.                                                      |
| label             | String    | Yes       | Free text, max 100 chars | A user-defined name for easy identification (e.g., "My Trading Bot").                                |
| key_prefix        | String    | Yes       | e.g., "CDAP\_"...        | A short, non-sensitive prefix of the key used for lookup.                                            |
| hashed_key        | String    | Yes       | Hashed String            | The securely hashed API key. The full key is only shown to the user once upon creation.              |
| status            | Enum      | Yes       | ACTIVE, REVOKED          | The current status of the key.                                                                       |
| ip_whitelist      | JSON      | No        | Array of IP/CIDR         | An array of allowed IP addresses or CIDR blocks for this key. Null or empty means any IP is allowed. |
| expires_at        | Timestamp | No        | ISO 8601                 | The timestamp when this key will automatically expire. Null for non-expiring keys.                   |
| last_used_at      | Timestamp | No        | ISO 8601                 | Timestamp of the last successful use of this key.                                                    |
| created_at        | Timestamp | Yes       | ISO 8601                 | Timestamp when the key was created.                                                                  |

### Permission

A master catalog of all granular permissions available on the platform, supporting AK-004.

| Field Name    | Type   | Required? | Format                   | Description                                                 |
| ------------- | ------ | --------- | ------------------------ | ----------------------------------------------------------- |
| permission_id | UUID   | Yes       | UUID                     | Unique identifier for the permission.                       |
| scope_name    | String | Yes       | e.g., "data:read:trades" | The unique, machine-readable name for the permission scope. |
| description   | String | Yes       | Free text                | A human-readable description of what the permission allows. |

### PermissionSet

Stores reusable templates or sets of permissions, supporting AK-004.

| Field Name        | Type    | Required? | Format                   | Description                                                                             |
| ----------------- | ------- | --------- | ------------------------ | --------------------------------------------------------------------------------------- |
| permission_set_id | UUID    | Yes       | UUID                     | Unique identifier for the permission set.                                               |
| user_id           | UUID    | No        | UUID                     | Foreign key to the User table if it's a user-defined set. Null for system-defined sets. |
| set_name          | String  | Yes       | e.g., "Read-Only Access" | The name of the permission set.                                                         |
| is_system_defined | Boolean | Yes       | True / False             | True if this is a default set provided by the platform.                                 |
| description       | String  | No        | Free text                | A brief description of the permission set's purpose.                                    |

### PermissionSet_Mapping

A pivot table to create the many-to-many relationship between PermissionSet and Permission.

| Field Name        | Type | Required? | Format | Description                                     |
| ----------------- | ---- | --------- | ------ | ----------------------------------------------- |
| permission_set_id | UUID | Yes       | UUID   | Foreign key linking to the PermissionSet table. |
| permission_id     | UUID | Yes       | UUID   | Foreign key linking to the Permission table.    |

### ApiRequestLog

Logs all incoming API requests for auditing, security, and usage tracking, supporting AK-001 and AK-002.

| Field Name        | Type      | Required? | Format                       | Description                                             |
| ----------------- | --------- | --------- | ---------------------------- | ------------------------------------------------------- |
| log_id            | BigInt    | Yes       | Auto-incrementing            | Unique identifier for the log entry.                    |
| api_key_id        | UUID      | Yes       | UUID                         | Foreign key linking to the ApiKey used for the request. |
| ip_address        | String    | Yes       | IP Address                   | The source IP address of the request.                   |
| endpoint          | String    | Yes       | e.g., "/v1/trades/BTC-USD"   | The API endpoint that was accessed.                     |
| status            | Enum      | Yes       | SUCCESS, FAIL_KEY, FAIL_PERM | The outcome of the authentication/authorization check.  |
| user_agent        | String    | No        | User-Agent string            | The User-Agent header from the request.                 |
| request_timestamp | Timestamp | Yes       | ISO 8601                     | The timestamp when the request was received.            |

### InternalService

Stores authentication information for internal, service-to-service communication, supporting AK-003.

| Field Name   | Type      | Required? | Format                   | Description                                                                                      |
| ------------ | --------- | --------- | ------------------------ | ------------------------------------------------------------------------------------------------ |
| service_id   | UUID      | Yes       | UUID                     | Unique identifier for the internal service.                                                      |
| service_name | String    | Yes       | e.g., "price-aggregator" | The unique, machine-readable name of the internal service.                                       |
| auth_type    | Enum      | Yes       | JWT, MTLS                | The authentication method used by the service.                                                   |
| credentials  | JSON      | Yes       | Encrypted JSON           | Securely stored credentials, such as client IDs, secrets for JWTs, or certificate info for mTLS. |
| status       | Enum      | Yes       | ACTIVE, DECOMMISSIONED   | The operational status of the service's credentials.                                             |
| created_at   | Timestamp | Yes       | ISO 8601                 | Timestamp when the service credentials were created.                                             |

## Relationships

- **User [user_id (PK)] 1 => \* ApiKey [api_key_id (PK), user_id (FK)]**
- **User [user_id (PK)] 1 => \* PermissionSet [permission_set_id (PK), user_id (FK)]**
- **PermissionSet [permission_set_id (PK)] 1 => \* PermissionSet_Mapping**
- **Permission [permission_id (PK)] 1 => \* PermissionSet_Mapping**
- **PermissionSet [permission_set_id (PK)] 1 => \* ApiKey [api_key_id (PK), permission_set_id (FK)]**
- **ApiKey [api_key_id (PK)] 1 => \* ApiRequestLog [log_id (PK), api_key_id (FK)]**

## Entities & Relationships

- **User**
  - user_id (PK)
- **Permission**
  - permission_id (PK)
- **PermissionSet**
  - permission_set_id (PK)
  - user_id (FK → User.user_id)
- **PermissionSet_Mapping**
  - permission_set_id (PK, FK → PermissionSet.permission_set_id)
  - permission_id (PK, FK → Permission.permission_id)
- **ApiKey**
  - api_key_id (PK)
  - user_id (FK → User.user_id)
  - permission_set_id (FK → PermissionSet.permission_set_id)
- **ApiRequestLog**
  - log_id (PK)
  - api_key_id (FK → ApiKey.api_key_id)
- **InternalService**
  - service_id (PK)

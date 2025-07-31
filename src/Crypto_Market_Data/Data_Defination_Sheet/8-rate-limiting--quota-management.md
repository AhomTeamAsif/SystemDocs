# Rate Limiting & Quota Management

## Overview

This document details the data models that support the platform's Rate Limiting & Quota Management features. These structures are designed to store rate limiting configurations, usage quotas, fair usage policies, and dynamic adjustment parameters. The primary purpose of these models is to ensure fair resource allocation, prevent system abuse, maintain optimal performance, and provide predictable service levels across different subscription tiers.

## Relations between Tables

### RateLimitConfig & SubscriptionTier

- **Cardinality**: One-to-Many (1 to \*)
- **Description**: A single SubscriptionTier (e.g., "Professional") serves as a parent for multiple RateLimitConfig records. This allows for defining granular rules for each tier, such as a global request limit, a more stringent limit for computationally expensive endpoints, and a burst allowance, providing flexible and powerful control over API access.

### UserQuota & User

- **Cardinality**: One-to-Many (1 to \*)
- **Description**: Tracks a user's consumption against their allowances. Each User has multiple UserQuota records, with each record tracking a specific metric (like API_CALLS) over a specific period (DAILY or MONTHLY). This structure is essential for enforcing the limits defined in the user's subscription tier and for triggering usage-based billing or notifications.

### RateLimitViolation & User

- **Cardinality**: One-to-Many (1 to \*)
- **Description**: Every time a user's request is denied due to exceeding a rate limit or quota, a new entry is created in the RateLimitViolation table. This provides a complete historical trail of a user's infraction history, which is critical for abuse detection, automated enforcement escalation, and customer support.

### FairUsagePolicy & SubscriptionTier

- **Cardinality**: Many-to-One (\* to 1)
- **Description**: Allows for defining different standards of "fair use." A policy can be global (where tier_id is null) or linked to a specific SubscriptionTier. This gives the platform the flexibility to enforce stricter behavioral rules on free tiers compared to high-paying enterprise clients.

### DynamicRateAdjustment & RateLimitConfig

- **Cardinality**: One-to-Many (1 to \*)
- **Description**: Provides a historical record of system-initiated changes. A single, static RateLimitConfig rule may be temporarily altered due to system load or other real-time conditions. Each such alteration is logged as a DynamicRateAdjustment record, ensuring full transparency into how and why service levels were modified.

### UserWhitelist, User, & FairUsagePolicy

- **Cardinality**: Many-to-Many (_ to _)
- **Description**: The UserWhitelist table acts as a bridge, allowing specific, trusted Users to be granted full or partial exemptions from automated restrictions defined in a FairUsagePolicy, accommodating legitimate high-volume use cases.

## Data Models

### RateLimitConfig

Defines rate limiting rules for different subscription tiers and endpoints, supporting feature RL-001.

| Field Name           | Type      | Required? | Format                        | Description                                                    |
| -------------------- | --------- | --------- | ----------------------------- | -------------------------------------------------------------- |
| config_id            | UUID      | Yes       | UUID                          | Primary key for the rate limit configuration.                  |
| tier_id              | UUID      | Yes       | UUID                          | Foreign key linking to the SubscriptionTier table.             |
| endpoint_pattern     | String    | No        | e.g., "/api/v1/prices/\*"     | Specific endpoint or pattern (null for global limits).         |
| limit_type           | Enum      | Yes       | PER_MINUTE, PER_HOUR, PER_DAY | Time window for the rate limit.                                |
| request_limit        | Integer   | Yes       | Positive integer              | Maximum requests allowed in the time window.                   |
| burst_limit          | Integer   | No        | Positive integer              | Additional requests allowed for short bursts.                  |
| burst_window_seconds | Integer   | No        | Seconds                       | Time window for burst allowance.                               |
| priority             | Integer   | Yes       | 1-100                         | Priority when multiple configs apply (higher = more specific). |
| is_active            | Boolean   | Yes       | true/false                    | Whether this configuration is currently active.                |
| created_at           | Timestamp | Yes       | ISO 8601                      | When this configuration was created.                           |
| updated_at           | Timestamp | Yes       | ISO 8601                      | When this configuration was last modified.                     |

### UserQuota

Tracks individual user quota allocations and current usage for multiple metric types, supporting feature RL-002.

| Field Name        | Type      | Required? | Format                                      | Description                                                            |
| ----------------- | --------- | --------- | ------------------------------------------- | ---------------------------------------------------------------------- |
| user_id           | UUID      | Yes       | UUID                                        | Foreign key to the User table. Part of composite primary key.          |
| quota_type        | Enum      | Yes       | API_CALLS, DATA_POINTS, HISTORICAL_REQUESTS | Type of quota being tracked. Part of composite key.                    |
| period_type       | Enum      | Yes       | DAILY, MONTHLY                              | Quota reset period. Part of composite key.                             |
| allocated_quota   | Integer   | Yes       | Positive integer                            | Total quota allocated for the period, based on the user's tier.        |
| used_quota        | Integer   | Yes       | Non-negative integer                        | Amount of quota currently used in the period.                          |
| reset_date        | Timestamp | Yes       | ISO 8601                                    | When the quota will next reset.                                        |
| last_usage_at     | Timestamp | No        | ISO 8601                                    | Timestamp of the last quota usage.                                     |
| warning_threshold | Float     | Yes       | 0.0 - 1.0                                   | Percentage threshold for usage warnings (e.g., 0.8 for 80%).           |
| warning_sent      | Boolean   | Yes       | true/false                                  | Whether a warning notification has been sent for this period.          |
| overage_allowed   | Boolean   | Yes       | true/false                                  | Whether overage beyond the quota is permitted (often tied to billing). |
| overage_limit     | Integer   | No        | Positive integer                            | Maximum overage allowed (if overage_allowed is true).                  |
| updated_at        | Timestamp | Yes       | ISO 8601                                    | When this quota record was last updated.                               |

### RateLimitViolation

Time-series table logging rate limit violations and enforcement actions, supporting feature RL-003.

| Field Name         | Type      | Required? | Format                                 | Description                                            |
| ------------------ | --------- | --------- | -------------------------------------- | ------------------------------------------------------ |
| violation_id       | UUID      | Yes       | UUID                                   | Primary key for the violation record.                  |
| user_id            | UUID      | Yes       | UUID                                   | Foreign key linking to the User table.                 |
| timestamp          | Timestamp | Yes       | ISO 8601                               | When the violation occurred.                           |
| endpoint           | String    | Yes       | e.g., "/api/v1/prices"                 | The endpoint where the violation occurred.             |
| violation_type     | Enum      | Yes       | RATE_LIMIT, QUOTA_EXCEEDED, FAIR_USAGE | The type of rule that was violated.                    |
| limit_exceeded     | Integer   | Yes       | Positive integer                       | The defined limit that was exceeded.                   |
| actual_usage       | Integer   | Yes       | Positive integer                       | The actual usage that triggered the violation.         |
| enforcement_action | Enum      | Yes       | WARNING, THROTTLE, BLOCK, SUSPEND      | Action taken by the system in response.                |
| duration_seconds   | Integer   | No        | Seconds                                | Duration of the enforcement action (if applicable).    |
| severity           | Enum      | Yes       | LOW, MEDIUM, HIGH, CRITICAL            | Severity level of the violation, used for escalations. |
| user_agent         | String    | No        | Free text                              | User agent string from the offending request.          |
| ip_address         | String    | No        | IP address                             | Client IP address of the offending request.            |
| metadata           | JSON      | No        | JSON object                            | Additional context about the violation for analysis.   |

### FairUsagePolicy

Defines fair usage rules and automated enforcement thresholds, supporting feature RL-003.

| Field Name                | Type      | Required? | Format           | Description                                                                             |
| ------------------------- | --------- | --------- | ---------------- | --------------------------------------------------------------------------------------- |
| policy_id                 | UUID      | Yes       | UUID             | Primary key for the fair usage policy.                                                  |
| policy_name               | String    | Yes       | Free text        | Human-readable name for the policy (e.g., "Free Tier Anti-Scraping Policy").            |
| tier_id                   | UUID      | No        | UUID             | Foreign key to SubscriptionTier (null for global policies).                             |
| detection_window_hours    | Integer   | Yes       | Hours            | Time window for detecting anomalous behavior patterns.                                  |
| threshold_multiplier      | Float     | Yes       | > 1.0            | Multiplier of a user's normal usage baseline that triggers a violation.                 |
| violation_count_threshold | Integer   | Yes       | Positive integer | Number of violations within the window before escalating action.                        |
| escalation_sequence       | JSON      | Yes       | JSON array       | A defined sequence of enforcement actions (e.g., ["WARN", "THROTTLE_1H", "BLOCK_24H"]). |
| whitelist_enabled         | Boolean   | Yes       | true/false       | Whether whitelisting is available for this policy.                                      |
| appeal_process_enabled    | Boolean   | Yes       | true/false       | Whether users are allowed to appeal actions under this policy.                          |
| auto_review_hours         | Integer   | No        | Hours            | Hours before an automated review of a violation is triggered.                           |
| is_active                 | Boolean   | Yes       | true/false       | Whether this policy is currently being enforced.                                        |
| created_at                | Timestamp | Yes       | ISO 8601         | When this policy was created.                                                           |
| updated_at                | Timestamp | Yes       | ISO 8601         | When this policy was last modified.                                                     |

### DynamicRateAdjustment

Time-series table storing dynamic rate limit adjustments based on system conditions, supporting feature RL-004.

| Field Name            | Type      | Required? | Format                                         | Description                                                            |
| --------------------- | --------- | --------- | ---------------------------------------------- | ---------------------------------------------------------------------- |
| adjustment_id         | UUID      | Yes       | UUID                                           | Primary key for the adjustment record.                                 |
| config_id             | UUID      | Yes       | UUID                                           | Foreign key linking to the specific RateLimitConfig that was adjusted. |
| timestamp             | Timestamp | Yes       | ISO 8601                                       | When the adjustment was made.                                          |
| trigger_type          | Enum      | Yes       | SYSTEM_LOAD, USER_BEHAVIOR, PREDICTIVE, MANUAL | The reason for the adjustment.                                         |
| original_limit        | Integer   | Yes       | Positive integer                               | The original rate limit value before adjustment.                       |
| adjusted_limit        | Integer   | Yes       | Positive integer                               | The new, temporarily adjusted rate limit value.                        |
| adjustment_factor     | Float     | Yes       | > 0.0                                          | Multiplier applied to the original limit to get the adjusted limit.    |
| system_load_pct       | Float     | No        | 0.0 - 1.0                                      | System-wide load percentage at the time of adjustment.                 |
| cpu_usage_pct         | Float     | No        | 0.0 - 1.0                                      | CPU usage percentage of the relevant service.                          |
| memory_usage_pct      | Float     | No        | 0.0 - 1.0                                      | Memory usage percentage of the relevant service.                       |
| active_connections    | Integer   | No        | Non-negative integer                           | Number of active connections to the service.                           |
| prediction_confidence | Float     | No        | 0.0 - 1.0                                      | Confidence level for predictive adjustments.                           |
| duration_minutes      | Integer   | No        | Minutes                                        | Expected duration of the adjustment.                                   |
| auto_revert           | Boolean   | Yes       | true/false                                     | Whether the adjustment is configured to automatically revert.          |
| revert_at             | Timestamp | No        | ISO 8601                                       | The specific time the adjustment is scheduled to revert.               |
| created_by            | String    | No        | Free text                                      | Identifier for the user or system that created the adjustment.         |
| metadata              | JSON      | No        | JSON object                                    | Additional context about the adjustment for analysis.                  |

### UserWhitelist

Manages whitelisted users who are exempt from certain fair usage policies, supporting feature RL-003.

| Field Name     | Type      | Required? | Format        | Description                                                                                 |
| -------------- | --------- | --------- | ------------- | ------------------------------------------------------------------------------------------- |
| whitelist_id   | UUID      | Yes       | UUID          | Primary key for the whitelist entry.                                                        |
| user_id        | UUID      | Yes       | UUID          | Foreign key linking to the User table.                                                      |
| policy_id      | UUID      | Yes       | UUID          | Foreign key linking to the FairUsagePolicy from which the user is exempt.                   |
| exemption_type | Enum      | Yes       | PARTIAL, FULL | The level of exemption from the policy.                                                     |
| custom_limits  | JSON      | No        | JSON object   | Custom limits or rules that apply instead of the policy's defaults (for PARTIAL exemption). |
| reason         | Text      | Yes       | Free text     | Justification for why the user is being whitelisted.                                        |
| approved_by    | String    | Yes       | Free text     | The admin or system that approved the whitelist entry.                                      |
| valid_from     | Timestamp | Yes       | ISO 8601      | When the whitelist entry becomes active.                                                    |
| valid_until    | Timestamp | No        | ISO 8601      | When the whitelist entry expires (null for permanent).                                      |
| review_date    | Timestamp | No        | ISO 8601      | A future date when this entry should be manually reviewed.                                  |
| is_active      | Boolean   | Yes       | true/false    | Whether this whitelist entry is currently active and enforced.                              |
| created_at     | Timestamp | Yes       | ISO 8601      | When this entry was created.                                                                |
| updated_at     | Timestamp | Yes       | ISO 8601      | When this entry was last modified.                                                          |

## Relationships

- **SubscriptionTier [tier_id (PK)] 1 => \* RateLimitConfig [tier_id (FK)]**
- **User [user_id (PK)] 1 => \* UserQuota [user_id (FK)]**
- **User [user_id (PK)] 1 => \* RateLimitViolation [user_id (FK)]**
- **SubscriptionTier [tier_id (PK)] 1 => \* FairUsagePolicy [tier_id (FK)]**
- **RateLimitConfig [config_id (PK)] 1 => \* DynamicRateAdjustment [config_id (FK)]**
- **User [user_id (PK)] 1 => \* UserWhitelist [user_id (FK)]**
- **FairUsagePolicy [policy_id (PK)] 1 => \* UserWhitelist [policy_id (FK)]**

## Entities & Relationships

- **RateLimitConfig**
  - config_id (PK)
  - tier_id (FK → SubscriptionTier.tier_id)
- **UserQuota**
  - user_id (PK, FK → User.user_id)
  - quota_type (PK)
  - period_type (PK)
- **RateLimitViolation**
  - violation_id (PK)
  - user_id (FK → User.user_id)
- **FairUsagePolicy**
  - policy_id (PK)
  - tier_id (FK → SubscriptionTier.tier_id)
- **DynamicRateAdjustment**
  - adjustment_id (PK)
  - config_id (FK → RateLimitConfig.config_id)
- **UserWhitelist**
  - whitelist_id (PK)
  - user_id (FK → User.user_id)
  - policy_id (FK → FairUsagePolicy.policy_id)

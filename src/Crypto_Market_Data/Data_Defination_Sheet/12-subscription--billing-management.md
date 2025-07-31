# Subscription & Billing Management Data Specification

## Overview

This document details the data models that support the platform's Subscription & Billing Management features. These structures are designed to store subscription tier definitions, payment processing information, usage tracking metrics, enterprise contract management data, and a complete invoicing system. The primary purpose of these models is to enable flexible subscription management, accurate and auditable billing, and comprehensive usage monitoring for diverse user segments from individual developers to enterprise clients.

---

## Relations between Tables

- **SubscriptionTier**: Defines the available subscription plans and their associated limits and features. It serves as a master template for what each subscription entails.
- **UserSubscription**: Links a User to a SubscriptionTier and tracks the subscription's lifecycle. It has a many-to-one relationship with SubscriptionTier and a one-to-one relationship with User. This table has been updated to include a direct link to the specific PaymentMethod to be used for renewals.
- **PaymentMethod**: Stores user payment information for automated billing. It has a many-to-one relationship with User, allowing multiple payment methods per user.
- **Invoice (New)**: Represents a bill issued to a user for a specific billing period. It aggregates charges from subscriptions and usage overages. It has a many-to-one relationship with User.
- **InvoiceLineItem (New)**: Details the specific charges on an invoice, such as the base subscription fee or usage overage costs. It has a many-to-one relationship with the Invoice table.
- **UsageMetric**: A time-series table that tracks all billable activities. It has a many-to-one relationship with User. Overage metrics can now be linked directly to the Invoice on which they were billed.
- **EnterpriseContract**: Stores bespoke contract terms for enterprise clients. It has a one-to-one relationship with User for enterprise accounts.

---

## SubscriptionTier

Defines available subscription tiers with their features, limits, and pricing, supporting feature SB-001.

| Field Name           | Type      | Required? | Format                                | Description                                                 |
| -------------------- | --------- | --------- | ------------------------------------- | ----------------------------------------------------------- |
| tier_id              | UUID      | Yes       | UUID                                  | Primary key for the subscription tier.                      |
| tier_name            | String    | Yes       | e.g., "Free", "Professional"          | Human-readable name of the subscription tier.               |
| tier_level           | Integer   | Yes       | 0-100                                 | Numeric level for tier comparison (0=Free, 100=Enterprise). |
| monthly_price        | Decimal   | Yes       | Currency format                       | Monthly subscription price in USD.                          |
| annual_price         | Decimal   | No        | Currency format                       | Annual subscription price in USD (if applicable).           |
| api_rate_limit       | Integer   | Yes       | Requests per minute                   | Maximum API requests allowed per minute.                    |
| daily_quota          | Integer   | Yes       | Number                                | Maximum API calls allowed per day.                          |
| historical_data_days | Integer   | Yes       | Days                                  | Number of days of historical data access allowed.           |
| support_level        | Enum      | Yes       | COMMUNITY, EMAIL, PRIORITY, DEDICATED | Level of customer support provided.                         |
| features             | JSON      | Yes       | JSON array                            | List of enabled features or capabilities.                   |
| is_active            | Boolean   | Yes       | true/false                            | Whether this tier is currently available.                   |
| created_at           | Timestamp | Yes       | ISO 8601                              | When this tier was created.                                 |
| updated_at           | Timestamp | Yes       | ISO 8601                              | When this tier was last modified.                           |

---

## UserSubscription

Tracks user subscription status and lifecycle. This table has been modified to link directly to a payment method for renewals, supporting features SB-001 and SB-004.

| Field Name          | Type      | Required? | Format                                       | Description                  |
| ------------------- | --------- | --------- | -------------------------------------------- | ---------------------------- |
| subscription_id     | UUID      | Yes       | UUID                                         | Primary key.                 |
| user_id             | UUID      | Yes       | UUID                                         | FK to User.                  |
| tier_id             | UUID      | Yes       | UUID                                         | FK to SubscriptionTier.      |
| payment_method_id   | UUID      | No        | UUID                                         | FK to PaymentMethod.         |
| status              | Enum      | Yes       | TRIAL, ACTIVE, CANCELLED, SUSPENDED, EXPIRED | Current subscription status. |
| start_date          | Timestamp | Yes       | ISO 8601                                     | Start date.                  |
| end_date            | Timestamp | No        | ISO 8601                                     | End date (null if ongoing).  |
| trial_end_date      | Timestamp | No        | ISO 8601                                     | End of trial period.         |
| billing_cycle       | Enum      | Yes       | MONTHLY, ANNUAL                              | Billing frequency.           |
| auto_renew          | Boolean   | Yes       | true/false                                   | Whether auto-renews.         |
| cancellation_reason | String    | No        | Free text                                    | Reason for cancellation.     |
| created_at          | Timestamp | Yes       | ISO 8601                                     | Creation time.               |
| updated_at          | Timestamp | Yes       | ISO 8601                                     | Last update.                 |

---

## PaymentMethod

Stores user payment information for automated billing, supporting feature SB-002.

| Field Name          | Type      | Required? | Format                                               | Description                       |
| ------------------- | --------- | --------- | ---------------------------------------------------- | --------------------------------- |
| payment_method_id   | UUID      | Yes       | UUID                                                 | Primary key.                      |
| user_id             | UUID      | Yes       | UUID                                                 | FK to User.                       |
| provider            | Enum      | Yes       | STRIPE, PAYPAL, BANK_TRANSFER, CRYPTO                | Payment provider.                 |
| provider_payment_id | String    | Yes       | Provider-specific                                    | External provider ID.             |
| method_type         | Enum      | Yes       | CREDIT_CARD, DEBIT_CARD, BANK_ACCOUNT, CRYPTO_WALLET | Method type.                      |
| last_four_digits    | String    | No        | 4 digits                                             | Display-only card/account suffix. |
| expiry_month        | Integer   | No        | 1-12                                                 | Expiry month.                     |
| expiry_year         | Integer   | No        | YYYY                                                 | Expiry year.                      |
| is_default          | Boolean   | Yes       | true/false                                           | Default method?                   |
| is_verified         | Boolean   | Yes       | true/false                                           | Verified status.                  |
| created_at          | Timestamp | Yes       | ISO 8601                                             | Created at.                       |
| updated_at          | Timestamp | Yes       | ISO 8601                                             | Updated at.                       |

---

## Invoice

Represents a single, auditable bill issued to a user for a specific billing period.

| Field Name           | Type      | Required? | Format                                 | Description             |
| -------------------- | --------- | --------- | -------------------------------------- | ----------------------- |
| invoice_id           | UUID      | Yes       | UUID                                   | Primary key.            |
| user_id              | UUID      | Yes       | UUID                                   | FK to User.             |
| subscription_id      | UUID      | Yes       | UUID                                   | FK to UserSubscription. |
| status               | Enum      | Yes       | DRAFT, OPEN, PAID, VOID, UNCOLLECTIBLE | Invoice status.         |
| issue_date           | Timestamp | Yes       | ISO 8601                               | Issue date.             |
| due_date             | Timestamp | Yes       | ISO 8601                               | Due date.               |
| paid_date            | Timestamp | No        | ISO 8601                               | Paid date.              |
| billing_period_start | Timestamp | Yes       | ISO 8601                               | Billing period start.   |
| billing_period_end   | Timestamp | Yes       | ISO 8601                               | Billing period end.     |
| total_amount         | Decimal   | Yes       | Currency                               | Amount due.             |
| currency             | String    | Yes       | e.g., USD                              | Invoice currency.       |
| invoice_pdf_url      | String    | No        | URL                                    | Link to invoice PDF.    |

---

## InvoiceLineItem

Detailed breakdown of charges on a specific invoice.

| Field Name           | Type      | Required? | Format    | Description                    |
| -------------------- | --------- | --------- | --------- | ------------------------------ |
| line_item_id         | UUID      | Yes       | UUID      | Primary key.                   |
| invoice_id           | UUID      | Yes       | UUID      | FK to Invoice.                 |
| description          | String    | Yes       | Free text | Charge description.            |
| quantity             | Integer   | Yes       | Positive  | Quantity.                      |
| unit_price           | Decimal   | Yes       | Currency  | Price per unit.                |
| total_amount         | Decimal   | Yes       | Currency  | Total (quantity × unit price). |
| service_period_start | Timestamp | Yes       | ISO 8601  | Service start.                 |
| service_period_end   | Timestamp | Yes       | ISO 8601  | Service end.                   |

---

## UsageMetric

Time-series table tracking API usage and billable activities. Modified to link overages to a specific invoice, supporting feature SB-003.

| Field Name     | Type      | Required? | Format                                     | Description               |
| -------------- | --------- | --------- | ------------------------------------------ | ------------------------- |
| user_id        | UUID      | Yes       | UUID                                       | FK to User (PK).          |
| timestamp      | Timestamp | Yes       | ISO 8601                                   | When usage occurred (PK). |
| metric_type    | Enum      | Yes       | API_CALL, DATA_POINTS, HISTORICAL_REQUESTS | Usage type (PK).          |
| invoice_id     | UUID      | No        | UUID                                       | FK to Invoice.            |
| endpoint       | String    | No        | API Path                                   | Called endpoint.          |
| count          | Integer   | Yes       | Positive                                   | Units consumed.           |
| billable_units | Integer   | Yes       | Positive                                   | Billable units.           |
| overage        | Boolean   | Yes       | true/false                                 | Was it over quota?        |
| metadata       | JSON      | No        | JSON object                                | Additional usage context. |

---

## EnterpriseContract

Stores custom enterprise contract terms and SLAs, supporting feature SB-005.

| Field Name        | Type      | Required? | Format                             | Description          |
| ----------------- | --------- | --------- | ---------------------------------- | -------------------- |
| contract_id       | UUID      | Yes       | UUID                               | Primary key.         |
| user_id           | UUID      | Yes       | UUID                               | FK to User.          |
| contract_name     | String    | Yes       | Free text                          | Contract name.       |
| start_date        | Timestamp | Yes       | ISO 8601                           | Start date.          |
| end_date          | Timestamp | Yes       | ISO 8601                           | End date.            |
| monthly_fee       | Decimal   | Yes       | Currency                           | Fixed monthly fee.   |
| volume_tiers      | JSON      | No        | JSON array                         | Volume discounts.    |
| sla_uptime        | Float     | Yes       | 0.0 - 1.0                          | Uptime guarantee.    |
| sla_response_time | Integer   | Yes       | Milliseconds                       | Response time SLA.   |
| custom_features   | JSON      | No        | JSON array                         | Custom features.     |
| support_level     | String    | Yes       | Free text                          | Support description. |
| account_manager   | String    | No        | Free text                          | Assigned manager.    |
| renewal_terms     | Text      | No        | Free text                          | Renewal conditions.  |
| status            | Enum      | Yes       | DRAFT, ACTIVE, EXPIRED, TERMINATED | Contract status.     |
| created_at        | Timestamp | Yes       | ISO 8601                           | Created at.          |
| updated_at        | Timestamp | Yes       | ISO 8601                           | Last modified.       |

---

## Relationships

## Relationships

- `User [user_id (PK)]` 1 → 1 `UserSubscription [user_id (FK)]`
- `SubscriptionTier [tier_id (PK)]` 1 → \* `UserSubscription [tier_id (FK)]`
- `User [user_id (PK)]` 1 → \* `PaymentMethod [user_id (FK)]`
- `UserSubscription [subscription_id (PK)]` 1 → ? `PaymentMethod [payment_method_id (FK)]`
- `User [user_id (PK)]` 1 → \* `Invoice [user_id (FK)]`
- `Invoice [invoice_id (PK)]` 1 → \* `InvoiceLineItem [invoice_id (FK)]`
- `User [user_id (PK)]` 1 → \* `UsageMetric [user_id (FK)]`
- `Invoice [invoice_id (PK)]` 1 → \* `UsageMetric [invoice_id (FK)]`
- `User [user_id (PK)]` 1 → 1 `EnterpriseContract [user_id (FK)]`

---

## Entities & Relationships Summary

- **SubscriptionTier**
  - `tier_id (PK)`
- **UserSubscription**
  - `subscription_id (PK)`
  - `user_id (FK → User.user_id)`
  - `tier_id (FK → SubscriptionTier.tier_id)`
  - `payment_method_id (FK → PaymentMethod.payment_method_id)`
- **PaymentMethod**
  - `payment_method_id (PK)`
  - `user_id (FK → User.user_id)`
- **Invoice**
  - `invoice_id (PK)`
  - `user_id (FK → User.user_id)`
  - `subscription_id (FK → UserSubscription.subscription_id)`
- **InvoiceLineItem**
  - `line_item_id (PK)`
  - `invoice_id (FK → Invoice.invoice_id)`
- **UsageMetric**
  - `user_id (PK, FK → User.user_id)`
  - `timestamp (PK)`
  - `metric_type (PK)`
  - `invoice_id (FK → Invoice.invoice_id)`
- **EnterpriseContract**
  - `contract_id (PK)`
  - `user_id (FK → User.user_id)`

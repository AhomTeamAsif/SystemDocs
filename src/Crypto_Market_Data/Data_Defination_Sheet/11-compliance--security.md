# Compliance & Security

## Overview

This document details the data models that support the platform's Compliance & Security features. These structures are designed to manage data privacy compliance, user consent tracking, security threat detection, and incident response. The primary purpose of these models is to ensure regulatory compliance (GDPR, CCPA, etc.), protect user privacy rights, maintain comprehensive security monitoring, and provide robust protection against malicious attacks and data breaches.

## Relations between Tables

- **UserConsent & User**: The UserConsent table has a many-to-one relationship with the User table. Each user can have multiple consent records over time (e.g., granting, then withdrawing consent), but each individual consent record must belong to exactly one user. This is tracked via the UserConsent.user_id foreign key and is foundational for privacy compliance.
- **DataExportRequest & User**: This is a many-to-one relationship where multiple data export requests can be initiated by a single user. Each request in the DataExportRequest table is firmly linked back to one User via the user_id foreign key, ensuring proper handling of data portability rights under GDPR Article 20.
- **DataDeletionRequest & User**: Similar to export requests, this is a many-to-one relationship. A user can submit multiple "right to be forgotten" requests, and each record in the DataDeletionRequest table is associated with exactly one user. This facilitates tracking and fulfillment of deletion requests per GDPR Article 17.
- **SecurityIncident & User / IPReputationScore**: The SecurityIncident table has two key relationships. It has an optional many-to-one relationship with User (user_id can be null for system-wide incidents). It also has a many-to-one relationship with IPReputationScore via the source_ip field. This means many incidents can be traced back to a single malicious IP address, allowing for aggregated threat analysis.
- **IPReputationScore**: This table acts as a central hub for threat intelligence. It does not have outgoing foreign keys but is referenced by other tables. Its primary key, ip_address, links to SecurityIncident and SecurityAuditLog records, allowing events to be enriched with reputation and risk data.
- **SecurityAuditLog & User**: This table has a many-to-one relationship with User. A single user's activity can generate a large volume of audit log entries. The user_id is nullable because some system-level security events may not be attributable to a specific user.
- **PrivacyPolicyVersion & UserConsent**: A one-to-many relationship exists where one record in PrivacyPolicyVersion can be associated with many UserConsent records. The UserConsent.consent_version field links to the specific policy version the user agreed to, creating an immutable, auditable record of acceptance.

## UserConsent

Manages user consent for various data processing activities, supporting feature CS-001.

| Field Name            | Type      | Required? | Format                                                                                  | Description                                                   |
| --------------------- | --------- | --------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| consent_id            | UUID      | Yes       | UUID                                                                                    | Primary key for the consent record.                           |
| user_id               | UUID      | Yes       | UUID                                                                                    | Foreign key linking to the User table.                        |
| consent_type          | Enum      | Yes       | DATA_PROCESSING, MARKETING, ANALYTICS, COOKIES                                          | Type of consent being tracked.                                |
| purpose_description   | Text      | Yes       | FreeAd text                                                                             | Detailed description of data processing purpose.              |
| consent_status        | Enum      | Yes       | GRANTED, DENIED, WITHDRAWN, EXPIRED                                                     | Current consent status.                                       |
| consent_method        | Enum      | Yes       | EXPLICIT_OPT_IN, IMPLIED, PRE_CHECKED, GRANULAR                                         | How consent was obtained.                                     |
| consent_version       | String    | Yes       | e.g., "v2.1"                                                                            | Version of privacy policy/terms when consent was given.       |
| legal_basis           | Enum      | Yes       | CONSENT, CONTRACT, LEGAL_OBLIGATION, VITAL_INTERESTS, PUBLIC_TASK, LEGITIMATE_INTERESTS | GDPR legal basis for processing.                              |
| data_categories       | JSON      | Yes       | JSON array                                                                              | Categories of personal data covered by this consent.          |
| processing_purposes   | JSON      | Yes       | JSON array                                                                              | Specific purposes for data processing.                        |
| third_party_sharing   | Boolean   | Yes       | true/false                                                                              | Whether data may be shared with third parties.                |
| retention_period_days | Integer   | No        | Positive integer                                                                        | Data retention period in days (null for indefinite).          |
| granted_at            | Timestamp | No        | ISO 8601                                                                                | When consent was granted.                                     |
| withdrawn_at          | Timestamp | No        | ISO 8601                                                                                | When consent was withdrawn.                                   |
| expires_at            | Timestamp | No        | ISO 8601                                                                                | When consent expires (null for indefinite).                   |
| ip_address            | String    | No        | IP address                                                                              | IP address when consent was given.                            |
| user_agent            | String    | No        | Free text                                                                               | Browser/client when consent was given.                        |
| evidence_data         | JSON      | No        | JSON object                                                                             | Additional evidence of consent (e.g., form data, timestamps). |
| withdrawal_reason     | Text      | No        | Free text                                                                               | Reason for consent withdrawal.                                |
| created_at            | Timestamp | Yes       | ISO 8601                                                                                | When consent record was created.                              |
| updated_at            | Timestamp | Yes       | ISO 8601                                                                                | When consent record was last updated.                         |

## DataExportRequest

Tracks user requests for data portability and export (GDPR Article 20), supporting feature CS-001.

| Field Name         | Type      | Required? | Format                                            | Description                                   |
| ------------------ | --------- | --------- | ------------------------------------------------- | --------------------------------------------- |
| request_id         | UUID      | Yes       | UUID                                              | Primary key for the export request.           |
| user_id            | UUID      | Yes       | UUID                                              | Foreign key linking to the User table.        |
| request_type       | Enum      | Yes       | FULL_EXPORT, PARTIAL_EXPORT, SPECIFIC_DATA        | Scope of data export requested.               |
| data_categories    | JSON      | Yes       | JSON array                                        | Specific categories of data requested.        |
| export_format      | Enum      | Yes       | JSON, CSV, XML, PDF                               | Requested format for exported data.           |
| status             | Enum      | Yes       | PENDING, PROCESSING, COMPLETED, FAILED, CANCELLED | Current request status.                       |
| requested_at       | Timestamp | Yes       | ISO 8601                                          | When request was submitted.                   |
| processed_at       | Timestamp | No        | ISO 8601                                          | When processing began.                        |
| completed_at       | Timestamp | No        | ISO 8601                                          | When request was completed.                   |
| expires_at         | Timestamp | No        | ISO 8601                                          | When download link expires.                   |
| file_path          | String    | No        | File path                                         | Location of generated export file.            |
| file_size_bytes    | BigInt    | No        | Bytes                                             | Size of exported data file.                   |
| download_count     | Integer   | Yes       | Non-negative integer                              | Number of times file was downloaded.          |
| last_downloaded_at | Timestamp | No        | ISO 8601                                          | When file was last downloaded.                |
| verification_token | String    | No        | Token                                             | Security token for download verification.     |
| delivery_method    | Enum      | Yes       | DOWNLOAD_LINK, EMAIL, SECURE_PORTAL               | How export is delivered to user.              |
| delivery_address   | String    | No        | Email/URL                                         | Delivery destination.                         |
| processing_notes   | Text      | No        | Free text                                         | Notes about processing or issues encountered. |
| failure_reason     | Text      | No        | Free text                                         | Reason for failure (if status is FAILED).     |
| created_at         | Timestamp | Yes       | ISO 8601                                          | When request was created.                     |
| updated_at         | Timestamp | Yes       | ISO 8601                                          | When request was last updated.                |

## DataDeletionRequest

Manages "right to be forgotten" requests and tracks deletion progress (GDPR Article 17), supporting feature CS-001.

| Field Name                | Type      | Required? | Format                                                                 | Description                                                |
| ------------------------- | --------- | --------- | ---------------------------------------------------------------------- | ---------------------------------------------------------- |
| deletion_id               | UUID      | Yes       | UUID                                                                   | Primary key for the deletion request.                      |
| user_id                   | UUID      | Yes       | UUID                                                                   | Foreign key linking to the User table.                     |
| deletion_scope            | Enum      | Yes       | COMPLETE_DELETION, PARTIAL_DELETION, ANONYMIZATION                     | Scope of deletion requested.                               |
| data_categories           | JSON      | No        | JSON array                                                             | Specific data categories to delete (for partial deletion). |
| legal_ground              | Enum      | Yes       | CONSENT_WITHDRAWN, UNLAWFUL_PROCESSING, NO_LONGER_NECESSARY, OBJECTION | Legal ground for deletion.                                 |
| status                    | Enum      | Yes       | PENDING, UNDER_REVIEW, APPROVED, REJECTED, PROCESSING, COMPLETED       | Current request status.                                    |
| requested_at              | Timestamp | Yes       | ISO 8601                                                               | When deletion was requested.                               |
| reviewed_at               | Timestamp | No        | ISO 8601                                                               | When request was reviewed.                                 |
| approved_at               | Timestamp | No        | ISO 8601                                                               | When deletion was approved.                                |
| processing_started_at     | Timestamp | No        | ISO 8601                                                               | When deletion processing began.                            |
| completed_at              | Timestamp | No        | ISO 8601                                                               | When deletion was completed.                               |
| reviewer_id               | String    | No        | User ID                                                                | Who reviewed the request.                                  |
| rejection_reason          | Text      | No        | Free text                                                              | Reason for rejection (if applicable).                      |
| retention_override        | Boolean   | Yes       | true/false                                                             | Whether legal retention requirements override deletion.    |
| retention_reason          | Text      | No        | Free text                                                              | Reason for retention override.                             |
| backup_deletion_status    | Enum      | Yes       | PENDING, IN_PROGRESS, COMPLETED, NOT_APPLICABLE                        | Status of backup deletion.                                 |
| third_party_notification  | JSON      | No        | JSON object                                                            | Status of third-party deletion notifications.              |
| verification_required     | Boolean   | Yes       | true/false                                                             | Whether additional identity verification is required.      |
| verification_completed_at | Timestamp | No        | ISO 8601                                                               | When identity verification was completed.                  |
| processing_log            | JSON      | No        | JSON array                                                             | Log of deletion processing steps.                          |
| data_subjects_notified    | JSON      | No        | JSON array                                                             | List of data subjects/contacts notified.                   |
| created_at                | Timestamp | Yes       | ISO 8601                                                               | When deletion request was created.                         |
| updated_at                | Timestamp | Yes       | ISO 8601                                                               | When request was last updated.                             |

## SecurityIncident

Logs security threats, attacks, and incidents for monitoring and response, supporting feature CS-002.

| Field Name           | Type      | Required? | Format                                                                                          | Description                                             |
| -------------------- | --------- | --------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| incident_id          | UUID      | Yes       | UUID                                                                                            | Primary key for the security incident.                  |
| incident_type        | Enum      | Yes       | DDOS_ATTACK, BRUTE_FORCE, SQL_INJECTION, XSS_ATTEMPT, RATE_LIMIT_VIOLATION, SUSPICIOUS_BEHAVIOR | Type of security incident.                              |
| severity_level       | Enum      | Yes       | LOW, MEDIUM, HIGH, CRITICAL                                                                     | Severity level of the incident.                         |
| status               | Enum      | Yes       | DETECTED, INVESTIGATING, MITIGATED, RESOLVED, FALSE_POSITIVE                                    | Current incident status.                                |
| source_ip            | String    | Yes       | IP address                                                                                      | Source IP address of the incident.                      |
| target_endpoint      | String    | No        | URL/endpoint                                                                                    | API endpoint or resource targeted.                      |
| user_id              | UUID      | No        | UUID                                                                                            | Associated user (if applicable).                        |
| attack_vector        | Enum      | No        | HTTP_FLOOD, SYN_FLOOD, SLOWLORIS, APPLICATION_LAYER, VOLUMETRIC                                 | Type of attack vector used.                             |
| request_count        | Integer   | No        | Positive integer                                                                                | Number of malicious requests detected.                  |
| blocked_requests     | Integer   | No        | Non-negative integer                                                                            | Number of requests blocked.                             |
| detection_method     | Enum      | Yes       | RATE_LIMITING, PATTERN_MATCHING, ML_ANOMALY, REPUTATION_FILTER, MANUAL                          | How incident was detected.                              |
| detection_confidence | Float     | No        | 0.0 - 1.0                                                                                       | Confidence level of detection (for ML-based detection). |
| first_detected_at    | Timestamp | Yes       | ISO 8601                                                                                        | When incident was first detected.                       |
| last_activity_at     | Timestamp | Yes       | ISO 8601                                                                                        | When last malicious activity was observed.              |
| duration_seconds     | Integer   | No        | Seconds                                                                                         | Duration of the incident.                               |
| mitigation_actions   | JSON      | No        | JSON array                                                                                      | Actions taken to mitigate the incident.                 |
| impact_assessment    | Text      | No        | Free text                                                                                       | Assessment of incident impact.                          |
| root_cause           | Text      | No        | Free text                                                                                       | Identified root cause of the incident.                  |
| lessons_learned      | Text      | No        | Free text                                                                                       | Lessons learned and preventive measures.                |
| assigned_to          | String    | No        | User ID                                                                                         | Security team member assigned.                          |
| escalated            | Boolean   | Yes       | true/false                                                                                      | Whether incident was escalated.                         |
| escalated_at         | Timestamp | No        | ISO 8601                                                                                        | When incident was escalated.                            |
| resolved_at          | Timestamp | No        | ISO 8601                                                                                        | When incident was resolved.                             |
| false_positive       | Boolean   | Yes       | true/false                                                                                      | Whether incident was determined to be false positive.   |
| threat_intelligence  | JSON      | No        | JSON object                                                                                     | Related threat intelligence data.                       |
| forensic_data        | JSON      | No        | JSON object                                                                                     | Forensic evidence and analysis.                         |
| created_at           | Timestamp | Yes       | ISO 8601                                                                                        | When incident record was created.                       |
| updated_at           | Timestamp | Yes       | ISO 8601                                                                                        | When incident record was last updated.                  |

## IPReputationScore

Maintains IP address reputation data for threat detection and blocking, supporting feature CS-002.

| Field Name          | Type      | Required? | Format                                               | Description                                              |
| ------------------- | --------- | --------- | ---------------------------------------------------- | -------------------------------------------------------- |
| ip_address          | String    | Yes       | IP address                                           | IP address being tracked. Primary key.                   |
| reputation_score    | Integer   | Yes       | 0 - 100                                              | Overall reputation score (0 = malicious, 100 = trusted). |
| risk_level          | Enum      | Yes       | TRUSTED, LOW_RISK, MEDIUM_RISK, HIGH_RISK, MALICIOUS | Categorical risk assessment.                             |
| threat_categories   | JSON      | No        | JSON array                                           | Categories of threats associated with this IP.           |
| first_seen_at       | Timestamp | Yes       | ISO 8601                                             | When IP was first observed.                              |
| last_seen_at        | Timestamp | Yes       | ISO 8601                                             | When IP was last observed.                               |
| request_count       | BigInt    | Yes       | Non-negative integer                                 | Total requests from this IP.                             |
| blocked_count       | BigInt    | Yes       | Non-negative integer                                 | Total blocked requests from this IP.                     |
| incident_count      | Integer   | Yes       | Non-negative integer                                 | Number of security incidents from this IP.               |
| last_incident_at    | Timestamp | No        | ISO 8601                                             | When last security incident occurred.                    |
| geolocation         | JSON      | No        | JSON object                                          | Geographic location data.                                |
| asn_info            | JSON      | No        | JSON object                                          | Autonomous System Number information.                    |
| is_vpn              | Boolean   | No        | true/false/null                                      | Whether IP is identified as VPN/proxy.                   |
| is_tor              | Boolean   | No        | true/false/null                                      | Whether IP is identified as Tor exit node.               |
| is_datacenter       | Boolean   | No        | true/false/null                                      | Whether IP belongs to datacenter/hosting provider.       |
| whitelist_status    | Enum      | Yes       | NOT_LISTED, WHITELISTED, TEMPORARILY_WHITELISTED     | Whitelist status.                                        |
| blacklist_status    | Enum      | Yes       | NOT_LISTED, BLACKLISTED, TEMPORARILY_BLACKLISTED     | Blacklist status.                                        |
| auto_block_until    | Timestamp | No        | ISO 8601                                             | When automatic blocking expires.                         |
| external_reputation | JSON      | No        | JSON object                                          | Reputation data from external threat intelligence.       |
| notes               | Text      | No        | Free text                                            | Manual notes about this IP.                              |
| created_at          | Timestamp | Yes       | ISO 8601                                             | When reputation record was created.                      |
| updated_at          | Timestamp | Yes       | ISO 8601                                             | When reputation was last updated.                        |

## SecurityAuditLog

Time-series table logging all security-relevant events for compliance and forensic analysis, supporting features CS-001 and CS-002.

| Field Name          | Type      | Required? | Format                                                                                  | Description                                               |
| ------------------- | --------- | --------- | --------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| log_id              | UUID      | Yes       | UUID                                                                                    | Primary key for the audit log entry.                      |
| timestamp           | Timestamp | Yes       | ISO 8601                                                                                | When the event occurred.                                  |
| event_type          | Enum      | Yes       | LOGIN, LOGOUT, API_ACCESS, DATA_EXPORT, CONSENT_CHANGE, SECURITY_INCIDENT, ADMIN_ACTION | Type of security event.                                   |
| user_id             | UUID      | No        | UUID                                                                                    | User associated with the event (if applicable).           |
| session_id          | String    | No        | Session ID                                                                              | Session identifier.                                       |
| ip_address          | String    | Yes       | IP address                                                                              | Source IP address.                                        |
| user_agent          | String    | No        | Free text                                                                               | Client user agent.                                        |
| endpoint            | String    | No        | URL/endpoint                                                                            | API endpoint or resource accessed.                        |
| http_method         | String    | No        | HTTP method                                                                             | HTTP method used (GET, POST, etc.).                       |
| response_code       | Integer   | No        | HTTP status code                                                                        | HTTP response code.                                       |
| request_size_bytes  | Integer   | No        | Bytes                                                                                   | Size of request payload.                                  |
| response_size_bytes | Integer   | No        | Bytes                                                                                   | Size of response payload.                                 |
| processing_time_ms  | Integer   | No        | Milliseconds                                                                            | Time taken to process request.                            |
| event_result        | Enum      | Yes       | SUCCESS, FAILURE, BLOCKED, SUSPICIOUS                                                   | Result of the event.                                      |
| risk_score          | Integer   | No        | 0 - 100                                                                                 | Calculated risk score for the event.                      |
| threat_indicators   | JSON      | No        | JSON array                                                                              | Security threat indicators detected.                      |
| geolocation         | JSON      | No        | JSON object                                                                             | Geographic location of request.                           |
| device_fingerprint  | String    | No        | Hash                                                                                    | Device fingerprint hash.                                  |
| referrer            | String    | No        | URL                                                                                     | HTTP referrer header.                                     |
| additional_headers  | JSON      | No        | JSON object                                                                             | Relevant HTTP headers.                                    |
| payload_hash        | String    | No        | Hash                                                                                    | Hash of request payload (for integrity).                  |
| compliance_flags    | JSON      | No        | JSON array                                                                              | Compliance-related flags (GDPR, etc.).                    |
| retention_category  | Enum      | Yes       | SECURITY, COMPLIANCE, OPERATIONAL, FORENSIC                                             | Data retention category.                                  |
| anonymized          | Boolean   | Yes       | true/false                                                                              | Whether personally identifiable data has been anonymized. |
| metadata            | JSON      | No        | JSON object                                                                             | Additional event metadata.                                |

## PrivacyPolicyVersion

Tracks privacy policy versions and user acceptance, supporting feature CS-001.

| Field Name          | Type      | Required? | Format                                   | Description                                        |
| ------------------- | --------- | --------- | ---------------------------------------- | -------------------------------------------------- |
| version_id          | UUID      | Yes       | UUID                                     | Primary key for the privacy policy version.        |
| version_number      | String    | Yes       | e.g., "v3.2"                             | Human-readable version number.                     |
| effective_date      | Timestamp | Yes       | ISO 8601                                 | When this version becomes effective.               |
| expiry_date         | Timestamp | No        | ISO 8601                                 | When this version expires (null for current).      |
| policy_content      | Text      | Yes       | Free text                                | Full text of the privacy policy.                   |
| content_hash        | String    | Yes       | SHA-256 hash                             | Hash of policy content for integrity.              |
| major_changes       | JSON      | No        | JSON array                               | Summary of major changes from previous version.    |
| legal_review_status | Enum      | Yes       | DRAFT, UNDER_REVIEW, APPROVED, PUBLISHED | Legal review status.                               |
| reviewed_by         | String    | No        | User ID                                  | Legal reviewer.                                    |
| reviewed_at         | Timestamp | No        | ISO 8601                                 | When legal review was completed.                   |
| published_at        | Timestamp | No        | ISO 8601                                 | When version was published.                        |
| acceptance_required | Boolean   | Yes       | true/false                               | Whether users must explicitly accept this version. |
| notification_sent   | Boolean   | Yes       | true/false                               | Whether users were notified of changes.            |
| is_current          | Boolean   | Yes       | true/false                               | Whether this is the current active version.        |
| created_at          | Timestamp | Yes       | ISO 8601                                 | When version was created.                          |
| updated_at          | Timestamp | Yes       | ISO 8601                                 | When version was last updated.                     |

## Relationships

- **User** [user_id (PK)] 1 => \* **UserConsent** [user_id (FK)]
- **User** [user_id (PK)] 1 => \* **DataExportRequest** [user_id (FK)]
- **User** [user_id (PK)] 1 => \* **DataDeletionRequest** [user_id (FK)]
- **User** [user_id (PK)] 1 => 0...\* **SecurityIncident** [user_id (FK, Nullable)]
- **User** [user_id (PK)] 1 => 0...\* **SecurityAuditLog** [user_id (FK, Nullable)]
- **IPReputationScore** [ip_address (PK)] 1 => \* **SecurityIncident** [source_ip (FK)]
- **IPReputationScore** [ip_address (PK)] 1 => \* **SecurityAuditLog** [ip_address (FK)]
- **PrivacyPolicyVersion** [version_number (Unique Key)] 1 => \* **UserConsent** [consent_version (FK)]

## Entities & Relationships

### UserConsent

- **consent_id** (PK)
- **user_id** (FK → User.user_id): Links consent to the specific user who provided it.
- **consent_version** (FK → PrivacyPolicyVersion.version_number): Links to the exact policy version that was accepted.

### DataExportRequest

- **request_id** (PK)
- **user_id** (FK → User.user_id): Links the export request to the user who made it.

### DataDeletionRequest

- **deletion_id** (PK)
- **user_id** (FK → User.user_id): Links the deletion request to the user who made it.

### SecurityIncident

- **incident_id** (PK)
- **user_id** (FK → User.user_id): Optionally links the incident to an affected user account.
- **source_ip** (FK → IPReputationScore.ip_address): Links the incident to the reputation record of the source IP.

### IPReputationScore

- **ip_address** (PK): The unique IP address. Referenced by SecurityIncident and SecurityAuditLog.

### SecurityAuditLog

- **log_id** (PK)
- **user_id** (FK → User.user_id): Optionally links the logged event to the user who performed the action.
- **ip_address** (FK → IPReputationScore.ip_address): Links the log to the reputation record of the source IP.

### PrivacyPolicyVersion

- **version_id** (PK)
- **version_number** (Unique Key): The business key referenced by UserConsent.

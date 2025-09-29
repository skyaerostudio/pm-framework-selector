# Data Lifecycle Policy

## 1. Storage Locations
- **Primary application database**: Encrypted PostgreSQL cluster hosted in the production VPC; contains user accounts, project metadata, and operational records required for providing the service.
- **Object storage**: Encrypted-at-rest S3-compatible bucket used for large file attachments and exported reports.
- **Analytics warehouse**: Segregated BigQuery project with anonymized event data used solely for aggregate analytics and product improvement.
- **Backup archives**: Daily encrypted snapshots of the primary database stored in a dedicated backup account with restricted access.
- **Configuration repository**: Version-controlled storage (e.g., Git) for rule sets, retention policies, and workflow definitions with access limited to authorized administrators.

## 2. Retention Timers
- **Active user data**: Retained for the duration of the customer contract and 90 days thereafter to support account reactivation or data export requests.
- **Inactive accounts**: Automatically scheduled for deletion 90 days after contract termination unless legal hold is applied.
- **System logs**: Application and infrastructure logs stored for 180 days for security monitoring and troubleshooting.
- **Analytics events**: Aggregated anonymized records retained for 24 months to analyze long-term usage trends.
- **Backups**: Full backups retained for 35 days with immutable storage policies; monthly compliance snapshots retained for 13 months.
- **Configuration versions**: All rule-set versions retained indefinitely with metadata linking to approval history.

## 3. Anonymization Strategy for Analytics
- Strip direct identifiers (name, email, IP) before ingestion into the analytics warehouse.
- Apply deterministic hashing for join keys where necessary while storing salt separately with restricted access.
- Generalize or bucket sensitive attributes (e.g., convert timestamps to date granularity, bucket counts into ranges).
- Enforce k-anonymity thresholds when creating derived datasets or dashboards.
- Conduct quarterly reviews with privacy stakeholders to validate anonymization effectiveness.

## 4. Purge Workflows
### 4.1 User-Initiated Deletion
1. User submits deletion request via account portal or support ticket.
2. Support team validates identity using multi-factor confirmation and logs the verification.
3. Automated job flags the account, immediately revokes access tokens, and notifies stakeholders.
4. Primary database records are soft-deleted and queued for hard deletion after a 7-day grace period to allow for error correction.
5. Once grace period lapses, related data in primary storage, object storage, and analytics warehouse is purged according to mapping tables.
6. Backups containing the data are tagged for expiration; entries age out based on the backup retention schedule.
7. Completion summary is recorded in the audit log and confirmation is sent to the requester.

### 4.2 Automated Deletion
1. Nightly retention job identifies data past configured timers.
2. Workflow service executes deletion tasks per data type, ensuring referential integrity.
3. Deletion results (success, partial, failure) are written to audit logs with timestamps and operator context.
4. Failed deletions trigger alerts to the data governance team for remediation.
5. Monthly reports summarize automated purge metrics for compliance review.

## 5. Audit Logging for Deletion Actions
- Capture user ID or service principal initiating deletion, target data scope, timestamps for request and completion, and success/failure status.
- Preserve audit logs in a write-once storage system with 24-month retention.
- Restrict log access to security, compliance, and designated engineering leads with least-privilege principles.

## 6. Versioned Rule-Set Storage
- Store retention and purge rules in a dedicated repository with semantic version tags.
- Require peer review and approval workflow before promoting rule changes to production.
- Maintain change history including author, approver, rationale, and deployment timestamp.
- Automatically synchronize approved versions to the workflow service with integrity checks (e.g., checksum validation).

## 7. Legal & Compliance Coordination
- Present this policy to legal, privacy, and compliance stakeholders for review against applicable regulations (GDPR, CCPA, SOC 2, ISO 27001).
- Document feedback and track required modifications in the compliance backlog.
- Schedule annual joint reviews or upon significant regulatory updates.
- Ensure Data Protection Impact Assessments (DPIA) are updated when material changes occur.


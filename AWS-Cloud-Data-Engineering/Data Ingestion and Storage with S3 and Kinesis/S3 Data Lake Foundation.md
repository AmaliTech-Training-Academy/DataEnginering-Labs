# S3 Data Lake Foundation

---

## PART 0: Why Data Lakes Matter

### The Big Picture

Modern companies (e.g., Netflix) process massive amounts of data daily:

- Petabytes of user activity data
- Need low-cost storage
- Need fast analytics access
- Need governance, compliance, and security

### Netflix Data Flow Example

- Users generate events (watch, pause, click)
- Data lands in S3 → `raw/` (immutable)
- Spark processes data → `processed/`
- Clean data → `curated/`
- Analytics & ML consume curated data
- Recommendations improve user engagement

### Without a Data Lake

- Data scattered across systems
- Duplicate storage everywhere
- No version control → data loss risk
- No audit trail → compliance failure
- Expensive fines (GDPR violations)

### With a Data Lake

- Centralized storage (S3)
- Organized structure (`raw/processed/curated`)
- Versioning for recovery
- Encryption for security
- Audit logs for compliance
- Lifecycle policies for cost savings

---

## PART 1: Lab Objectives

By the end of this lab, you will:

- Build a production S3 data lake
- Implement encryption & security
- Enable versioning & recovery
- Set up logging & audit trails
- Apply lifecycle cost optimization
- Configure compliance controls

---

## PART 2: Data Lake Architecture

### S3 Bucket Structure

```
data-lake-prod-123456789
│
├── raw/        (unprocessed data)
├── processed/  (cleaned data)
├── curated/    (analytics-ready data)
├── temp/       (temporary files)
└── archive/    (long-term storage)
```

### Security Layer

- SSE-S3 encryption enabled
- Public access blocked
- Bucket policy enforcement

### Governance Layer

- Access logging enabled
- CloudTrail enabled
- IAM role-based access control

### Resilience Layer

- Versioning enabled
- Recovery from accidental deletion
- Audit history preserved

### Cost Optimization

- Lifecycle policies
- Glacier / Deep Archive storage
- Automatic deletion of temp files

---

## PART 3: Prerequisites

Ensure you have:

- AWS account access
- IAM roles from Lab 1.1
- VPC from Lab 1.2
- Access to S3 console
- Region: `us-east-1`

---

## PART 4: Create S3 Bucket

### Bucket Details

| Setting | Value |
|---------|-------|
| Name | `data-lake-prod-<ACCOUNT_ID>` |
| Region | `us-east-1` |
| Block Public Access | Enable All |
| Object Ownership | ACLs disabled |

---

## PART 5: Security Configuration

### Encryption

| Setting | Value |
|---------|-------|
| Type | SSE-S3 |
| Bucket Key | Enabled |

### Bucket Policy (Key Rules)

- Enforce HTTPS only
- Block unencrypted uploads
- Allow IAM roles only:
  - `DataEngineerRole`
  - `GlueServiceRole`
  - `RedshiftIAMRole`

---

## PART 6: Versioning

**Purpose:**
- Prevent data loss
- Enable rollback
- Track file history

**Status:** Versioning: `ENABLED`

---

## PART 7: Access Logging

### Logging Setup

| Setting | Value |
|---------|-------|
| Logging bucket | `data-lake-prod-logs-<ACCOUNT_ID>` |
| Log prefix | `s3-access-logs/` |

**Purpose:**
- Track file access
- Audit user actions
- Investigate incidents

---

## PART 8: CloudTrail

### Configuration

- **Trail name:** `data-lake-audit-trail`
- **Logs:**
  - S3 object actions
  - Bucket changes
  - IAM interactions

---

## PART 9: Folder Structure

| Folder | Purpose |
|--------|---------|
| `raw/` | Source data (immutable) |
| `processed/` | Cleaned datasets |
| `curated/` | Analytics-ready data |
| `temp/` | Temporary job files |
| `archive/` | Long-term storage |

---

## PART 10: Lifecycle Policies

### Policy 1: Processed Data

- Move to Glacier after **90 days**
- Move to Deep Archive after **180 days**

### Policy 2: Temp Data

- Delete after **1 day**

### Policy 3: Archive Data

- Deep Archive after **30 days**
- Delete after **7 years**

---

## PART 11: Tagging

| Key | Value |
|-----|-------|
| Environment | Production |
| Owner | DataEngineering |
| Purpose | DataLake |
| CostCenter | Analytics |

---

## PART 12: Test Data Upload

- **File:** `test_customers.csv`
- **Uploaded to:** `raw/test_customers.csv`
- **Includes:**
  - Customer data
  - Verified encryption (SSE-S3)
  - Versioning enabled

---

## PART 13: Verification Checklist

- [x] Bucket created
- [x] Encryption enabled
- [x] Versioning enabled
- [x] Logging configured
- [x] CloudTrail active
- [x] Folder structure created
- [x] Lifecycle policies applied
- [x] Bucket policy enforced
- [x] Test data uploaded
- [x] Tags applied

---

## PART 14: Cost Optimization

### Storage Strategy

| Tier | Use Case |
|------|----------|
| Standard | Active data |
| Glacier | Infrequent access |
| Deep Archive | Compliance storage |

### Estimated Cost

- ~$20–30/month for 1TB dataset
- **80% savings** using lifecycle rules

---

## PART 15: Teardown

**Delete:**
- Test file only (`test_customers.csv`)

**Keep:**
- S3 bucket
- IAM roles
- VPC
- Logging bucket

---

## Key Learnings

### S3 Concepts
- Object storage at scale
- 11 nines durability
- Global uniqueness

### Data Governance
- Raw → Processed → Curated architecture
- Data lineage tracking
- Compliance readiness

### Security
- Encryption (SSE-S3)
- IAM role-based access
- Bucket policies

### Operations
- Logging & monitoring
- CloudTrail auditing
- Version recovery

### Cost Optimization
- Lifecycle transitions
- Glacier storage tiers
- Automated deletion rules

---

## Final Result

You have successfully built a **production-grade AWS S3 Data Lake** with:

- Security
- Compliance
- Governance
- Cost optimization
- Recovery mechanisms

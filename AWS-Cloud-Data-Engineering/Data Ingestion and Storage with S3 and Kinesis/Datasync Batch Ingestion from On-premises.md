# DataSync Batch Ingestion from On-Premises

---

## PART 0: Understanding Why Data Migration Matters

Before we start, let's understand why this is critical for any data platform.

### The Big Picture Story

Imagine your company has 50 years of data in a file server in the office.

**The Legacy Problem — Current State (Year 2024):**

- **Scale:** Hundreds of terabytes in on-premises servers.
- **Fragmentation:** Data scattered across file shares (2TB), old databases (5TB), email attachments (1TB), and backup drives (3TB).
- **Manual:** No automation; manual copying only.
- **Slow:** 1TB takes 2 days over internet.
- **Unreliable:** Network dropouts mid-transfer.
- **Rigid:** Upgrading servers costs $1M+.
- **Dark Data:** Data sits unused with no insights.

**The Cloud Migration Dream — Target State:**

- **Storage:** All data in S3 (infinitely scalable).
- **Automatic:** Daily syncs with zero manual work.
- **Performance:** Optimized for network efficiency.
- **Resilient:** Automatic retries on failure.
- **Cost:** Pay for what you use.
- **Secure:** Encryption in transit.
- **Intelligence:** Data analyzed by ML models.

**The Blocker:** How to move 11TB of data reliably?

---

### Traditional Data Migration Approaches (All Bad)

- **Manual FTP/SCP:** Takes 40+ hours of manual work; frequent network drops require manual restarts.
- **USB Hard Drives:** Costs $500+ and takes 8+ days including shipping and manual import.
- **DIY Scripts:** High risk of bugs, data corruption, and dozens of hours in engineering overhead.

### DataSync: The Enterprise Solution

- **Efficiency:** Set up in 1.5 hours.
- **Robust:** Handles 10GB+ chunks, network retries, and checksum verification.
- **Automation:** Bandwidth throttling and CloudWatch monitoring.
- **Result:** 11TB safely in S3 within 10 days for ~$50.

### Real-World Example: Enterprise Migration

| Feature | Old Way (Manual/Consulting) | New Way (DataSync) |
|---------|----------------------------|-------------------|
| **Time** | 6 months | 2 weeks |
| **Cost** | $2M (Hardware/Staff) | $50k (AWS/Staff) |
| **Risk** | High (Downtime/Human Error) | Minimal (Automated) |

### Why This Matters for Your Career

- **High Demand:** Every sector (Bank, Health, Retail) has legacy data to move.
- **High Pay:** Migration engineers earn **$130k–$160k**.
- **Integrity:** You learn to handle sensitive data where "losing a penny" isn't an option.
- **Growth:** Path from Junior (setup) to Manager (strategy).

---

## PART 1: Goals for This Lab

By the end of this lab, you will:

- Understand data transfer architectures.
- Deploy a DataSync agent.
- Configure sync tasks for automated transfers.
- Monitor transfer progress and verify integrity.
- Explain when to use DataSync vs. other methods.

---

## PART 2: What You'll Create

**The Complete DataSync Setup:**

- **On-Premises (Simulated with EC2):** Simulated "legacy file server" with sample data files.
- **DataSync Agent:** Reads files, transfers securely, and verifies integrity.
- **AWS (Cloud Destination):**
  - **S3 Bucket:** `data-lake-prod-123456789/processed/`
  - **DataSync Task:**
    - Daily at 3 AM
    - Syncs new files only via SMB/S3
    - Checksum validation
- **CloudWatch Monitoring:**
  - Success/failure alerts and transfer speed metrics.

---

## PART 3: The "Why" Behind Design Decisions

### Why DataSync > Manual/DIY?

- **Manual Script:** 20+ hours of labor ($2,000 cost) + high risk of corruption.
- **DataSync:** 1 hour setup ($50 cost) + built-in auto-retry and checksums.
- **Annual Savings:** Over **$710,000/year** for daily transfers.

### Why DataSync > AWS DMS?

- **DataSync:** For **File Systems** (NFS, SMB, S3).
- **DMS:** For **Databases** (Structured data, schema conversion).

### Why Scheduled Syncs > One-Time Transfer?

One-time transfers miss new data created the next day. Scheduled syncs ensure the Cloud and On-Prem environments stay in a permanent state of synchronization.

---

## PART 4: Prerequisites — Check These First

- **Completed Labs 1.1, 1.2, 1.3** (IAM, VPC, S3)
- **Completed Lab 2.1** (Cataloging)
- **S3 Bucket:** `data-lake-prod-123456789` exists
- **VPC:** `data-platform-vpc` exists
- **Time:** 3 hours uninterrupted

---

## STEP 0: Login and Prepare

- **Login:** Access AWS Console.
- **Region:** Ensure you are in `us-east-1`.
- **Verify:** Check VPC and S3 folder structures.
- **Documentation:** Create `Lab_2_2_DataSync.txt` to track IPs, names, and status.

---

## PART 1: Prepare On-Premises Data Source

*Simulating a file server using EC2.*

### Steps 1.1–1.2: Launch EC2

| Setting | Value |
|---------|-------|
| Name | `datasync-test-server` |
| AMI | Amazon Linux 2 |
| Type | t2.micro |
| VPC | `data-platform-vpc` / `private-subnet-1b` |
| Security Group | `sg-private-compute` |

### Step 1.3: Create Sample Files

Upload these to `s3://data-lake-prod-123456789/raw/` (simulating on-prem data):

- `customer_master.csv`
- `sales_history.csv`
- `transaction_log.csv`

---

## PART 2: Create DataSync Location (Source)

- **Navigate:** Go to DataSync Console > Locations.
- **Create Source:** Select **Amazon S3** (simulating our raw source).
- **Config:**
  - **Name:** `onprem-s3-raw-location`
  - **Bucket:** `data-lake-prod-123456789`
  - **Folder:** `/raw/`
  - **Role:** Create new `DataSyncS3Role` with full S3 access.

---

## PART 3: Create DataSync Location (Destination)

- **Create Destination:** Select **Amazon S3**.
- **Config:**
  - **Name:** `aws-s3-processed-location`
  - **Bucket:** `data-lake-prod-123456789`
  - **Folder:** `/processed/`
  - **Role:** Use existing `DataSyncS3Role`.

---

## PART 4: Create DataSync Task

- **Create Task:** Select your Source and Destination locations.
- **Settings:**
  - **Name:** `raw-to-processed-sync`
  - **Mode:** Sync (transfer new/modified files)
  - **Integrity:** Check **"Verify data integrity"**
  - **Logging:** Enable CloudWatch logging
  - **Schedule:** Set to **Daily at 03:00 UTC**

---

## PART 5: Run DataSync Task

- **Execute:** Click **"Start task"**.
- **Monitor:** Watch CloudWatch metrics.
- **Expectation:** Status should move to **Success** with 3 files copied and 100% verification.

---

## PART 6: Verify Data in Destination

- **S3 Check:** Ensure all 3 CSVs exist in the `/processed/` folder.
- **Athena Check:**

```sql
SELECT * FROM raw_data.test_customers LIMIT 5;
```

---

## PART 7: Test Incremental Sync

- **Add File:** Upload `new_customers.csv` to the `/raw/` folder.
- **Run Task:** Start the DataSync task again.
- **Verify:** Check that **"Files copied"** is 1, while 3 files were skipped.

This proves incremental efficiency.

---

## PART 8: Configure CloudWatch Alerts

- **SNS:** Create a topic `datasync-notifications`.
- **Alarm:** In CloudWatch, create an alarm for the metric `TaskExecutionsFailed`.
- **Action:** Set to notify your SNS topic if failures ≥ 1.

---

## PART 9: Documentation & Results

Update your `Lab_2_2_DataSync.txt` with:

- Execution times (approx. 2 mins)
- Data sizes (~2.5 KB)
- Verification status (Passed)
- Cost estimate (~$1/month for this lab setup)

---

## Success Criteria

- [ ] EC2 instance launched
- [ ] 3 CSV files in `/raw/`
- [ ] Source/Destination locations created
- [ ] Task scheduled and manually executed successfully
- [ ] Incremental sync verified (only new files move)
- [ ] CloudWatch Alarm active

**Congratulations! You have completed Lab 2.2.**

---

## Teardown — Optional Cleanup

- **Terminate EC2 instance:** Saves ~$12/month.
- **Delete DataSync Task:** Optional, very cheap.
- **Total Monthly Cost if kept:** ~$1.00.

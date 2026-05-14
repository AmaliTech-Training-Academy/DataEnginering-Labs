# Delta Lake — ACID Transactions & Time Travel

---

## Part 0: Why Delta Lake Matters

### The Production Problem You'll Solve

Your data lake (S3 + Parquet) at 8 AM:

```
Transaction A: writes customer updates    (8:00 AM)
Transaction B: writes fraud flags         (8:01 AM)
Transaction C: reads data for reporting   (8:02 AM)

What Transaction C sees:
  Some customer updates         yes
  Some fraud flags              yes
  Some customer updates         no  (they're still incomplete)
  Data corruption               yes (B partially overwrote A)

Result: BROKEN REPORT & BAD DECISIONS
```

**Without ACID guarantees:**

- Partial writes corrupt data
- Readers see inconsistent states
- Concurrent access causes conflicts
- No way to recover to a known good state
- Data governance becomes impossible

This is why Delta Lake exists.

### What is Delta Lake?

Delta Lake is an open-source storage format that adds ACID transactions to data lakes.

| | Traditional Data Lake (S3 + Parquet) | Delta Lake (S3 + Delta Format) |
|--|--------------------------------------|-------------------------------|
| **Files** | Simple files | Files with transaction log |
| **Guarantees** | None | ACID (all or nothing) |
| **Concurrent access** | Causes conflicts | Safe |
| **Recovery** | No history | Time travel to any version |
| **Schema** | Not enforced | Enforced and evolvable |

### Real-World Example: Payment Processing

**Without Delta Lake:**

- Payment deducted: $500 → $400 ✓
- System crashes before fraud flags update
- Notification sees $400 balance but no fraud flag
- Result: missed fraud detection

**With Delta Lake:**

- All 3 operations are in one atomic transaction
- System crashes mid-way
- Delta Lake rolls back all changes
- Balance stays at $500, no partial state
- Retry the whole transaction safely

### Why Companies Use Delta Lake

| Company | Use Case |
|---------|----------|
| Lyft | Real-time ride matching with guaranteed consistency |
| Uber | Trillion-row tables with ACID compliance |
| Microsoft | Enterprise data governance |
| Databricks | Cloud data platform foundation |

### Real-World Use Cases

| Use Case | Problem | Delta Solution | Benefit |
|----------|---------|---------------|---------|
| **Financial transactions** | Money transferred twice or lost | ACID transactions | $0 data loss |
| **Real-time analytics** | Report shows partial data from concurrent writes | Snapshot isolation | Always consistent data |
| **Data recovery** | Bad update corrupts dataset | Time travel | Restore to any version in seconds |
| **Schema evolution** | Adding a column breaks old readers | Schema enforcement + evolution | Evolve safely without downtime |

---

## Part 1: What You'll Learn

By the end of this lab, you'll understand:

- ACID properties and why they matter
- Transaction logs and how Delta ensures consistency
- MERGE operations (upsert in data lakes)
- Time travel — accessing historical data versions
- Schema enforcement and evolution
- When to use Delta vs. Parquet
- Production deployment patterns

---

## Part 2: What You'll Build

### Architecture Overview

```
AWS S3
├── raw/
│   └── customer_transactions.parquet      ← source data
│
└── delta/
    ├── bronze/                            ← raw data in Delta format
    │   └── transactions_delta/
    │       ├── _delta_log/               ← transaction log
    │       └── data files
    │
    ├── silver/                            ← cleaned, validated data
    │   └── transactions_clean_delta/
    │
    └── gold/                              ← aggregated, business-ready
        └── daily_summary_delta/

Databricks workspace
├── Notebook 1: ETL — load to bronze
├── Notebook 2: Transformation — silver layer
├── Notebook 3: MERGE — update with new data
├── Notebook 4: Time travel — recovery & audit
└── Notebook 5: Schema enforcement & evolution
```

### Lab Components

| Component | What | Why |
|-----------|------|-----|
| **Bronze layer** | Raw data in Delta format | Foundation for ACID transactions; enables time travel from ingestion |
| **Silver layer** | Cleaned, normalized data | Quality gates before analytics; schema validation and deduplication |
| **Gold layer** | Business-ready aggregations | Fast analytics; pre-aggregated for BI queries |
| **MERGE operation** | Upsert (update or insert) | Handle daily updates efficiently without full table rewrites |
| **Time travel** | Access historical versions | Data recovery, compliance, debugging |

---

## Part 3: The "Why" Behind Delta Lake Design

### Why ACID Transactions?

**Concurrent writes — traditional lake:**

```
Worker A: reading 1,000 rows
Worker B: writing 500 new rows
Worker A: finishes read → got 800 rows (inconsistent!)
```

**Concurrent writes — Delta Lake:**

```
Worker A: starts transaction snapshot (isolated view)
Worker B: starts concurrent write
Worker A: still sees 1,000 rows (consistent)
Worker B: completes write atomically
Worker A: next transaction sees 1,500 rows
Result: ACID guarantee — all or nothing
```

### Why the Transaction Log?

| | Parquet Files Only | Delta Transaction Log |
|--|-------------------|-----------------------|
| **Files** | 1,000 data files, no metadata | `_delta_log/` directory with JSON change records |
| **List time** | 30 seconds to list all files | 100ms to know latest version |
| **History** | No change history | Know exactly what changed and when |

### Why MERGE Operations?

| | Traditional ETL | Delta MERGE |
|--|----------------|-------------|
| **Data read** | Entire table (100 GB) | Only changed rows (1 GB) |
| **Data written** | Entire table (100 GB) | Only changes (1 GB) |
| **Time** | 45 minutes | 2 minutes |
| **Cost** | $3.00 | $0.05 |
| **Crash safety** | Mid-write crash = corrupted data | Mid-write crash = ACID rollback, data consistent |

**Result: 10x faster, 60x cheaper, and safer.**

### Why Time Travel?

**Scenario: bad update at 3:00 PM**

| | Traditional Backup | Delta Time Travel |
|--|-------------------|-------------------|
| **Recovery point** | Yesterday's backup (24 hours stale) | 3:05 PM — right before the bad update |
| **Data loss** | 2 days of data | Zero |
| **Process** | Manual reconciliation (error-prone) | One RESTORE command |
| **Time to fix** | 8 hours | 2 minutes |

### When to Use Delta vs. Parquet

| Use Parquet When | Use Delta When |
|-----------------|----------------|
| Static reference data | You need concurrent updates |
| Immutable data lakes | You need ACID transactions |
| One-time bulk loads | You need data corrections |
| Simple read-only queries | You need compliance and auditing |

> **Decision rule:** If the data ever changes → use Delta.

---

## Part 4: Prerequisites and Setup

**What you need:**

- S3 bucket from Tier 1.3 (`data-lake-prod-[ACCOUNT_ID]`)
- IAM roles from Tier 1.1 (`DataEngineerRole`)
- VPC from Tier 1.2
- Databricks account (Community Edition is free; AWS trial is 14 days free)
- Basic Spark SQL knowledge
- 5 hours uninterrupted

> **Why Databricks?** Databricks created Delta Lake, provides the best integration and performance, and makes cluster management invisible so you can focus on learning Delta concepts. You could use AWS EMR with Spark — but it's more complex to set up and troubleshoot.

---

## Part 5: Step-by-Step Lab Implementation

### Section A: Databricks Workspace Setup

#### Step A1: Create Databricks Workspace

- Go to [https://databricks.com](https://databricks.com) → **Get Started Free**
- Select cloud: **AWS**
- Select region: same as your S3 bucket (e.g., `us-east-1`)
- Click **Create workspace** — wait 5–10 minutes

**Expected:** A workspace URL like `https://xxxxxxxx.cloud.databricks.com`

#### Step A2: Attach AWS S3 Access

- Click **Admin Console** (gear icon, top right)
- Click **AWS Credentials** → **Add AWS Credentials**
- Paste your AWS access key and secret key (from IAM)
- Click **Verify**

#### Step A3: Create Compute Cluster

- Click **Compute** in the left sidebar → **Create Cluster**
- Fill in:

| Field | Value |
|-------|-------|
| Cluster name | `tier6-lab-delta-lake` |
| Databricks runtime | Latest LTS (e.g., 13.3 LTS) |
| Driver type | Single node |
| Worker type | 1 worker, 8 GB RAM |

- Click **Create Cluster** — wait 3–5 minutes for status to show `Running`

> **Cost:** $0.25/hour. This lab uses about 2 hours of compute = ~$0.50 total. Auto-terminates after 60 minutes idle.

---

### Section B: Load Data to Delta Bronze Layer

#### Step B1: Create Bronze Notebook

- Click **Workspace** in the left sidebar
- Click your username → right-click → **Create** → **Notebook**
- Fill in:

| Field | Value |
|-------|-------|
| Name | `01_bronze_layer_delta` |
| Language | SQL |
| Cluster | Your cluster |

#### Step B2: Write Bronze Layer ETL

Paste this code (replace `[YOUR_ACCOUNT_ID]`):

```sql
-- BRONZE LAYER: Raw data in Delta format
-- Purpose: Load raw Parquet data and make it ACID transactional

-- Step 1: Create database
CREATE DATABASE IF NOT EXISTS customer_analytics;
USE customer_analytics;

-- Step 2: Read Parquet from S3 and create Delta table
CREATE OR REPLACE TABLE transactions_bronze
USING DELTA
LOCATION 's3://data-lake-prod-[YOUR_ACCOUNT_ID]/delta/bronze/transactions'
AS
SELECT
    transaction_id,
    customer_id,
    transaction_date,
    amount,
    merchant,
    category,
    CURRENT_TIMESTAMP() as ingestion_time,
    input_file_name()  as source_file
FROM parquet.`s3://data-lake-prod-[YOUR_ACCOUNT_ID]/raw/transactions.parquet`;

-- Step 3: Verify data loaded
SELECT
    COUNT(*)                  as total_records,
    COUNT(DISTINCT customer_id) as unique_customers,
    MIN(amount)               as min_amount,
    MAX(amount)               as max_amount,
    SUM(amount)               as total_volume
FROM transactions_bronze;

-- Step 4: Check Delta table structure
DESCRIBE transactions_bronze;

-- Step 5: Verify transaction log (time travel starts from version 0)
SELECT COUNT(*) FROM transactions_bronze VERSION AS OF 0;
```

**Key concepts in this code:**

| Syntax | What It Does |
|--------|-------------|
| `USING DELTA` | Creates a Delta table (not plain Parquet) — ACID from day 1 |
| `LOCATION 's3://...'` | Stores data in S3 — cloud-agnostic |
| `input_file_name()` | Records which source file each row came from — useful for debugging |
| `CURRENT_TIMESTAMP()` | Records ingestion time — required for data governance |
| `VERSION AS OF 0` | Time travel — query the table at version 0 (initial load) |

#### Step B3: Run the Bronze Notebook

- Select all (`Ctrl+A`) → click **Run All**
- Wait 30–60 seconds

**Expected:** Table created, row counts displayed, no errors. Delta automatically creates a `_delta_log/` folder in S3 — this is the transaction log.

---

### Section C: Silver Layer (Data Quality and Cleaning)

#### Step C1: Create Silver Notebook

- Name: `02_silver_layer_delta`, Language: SQL, same cluster

#### Step C2: Write Silver Layer Transformation

```sql
-- SILVER LAYER: Cleaned, validated data
-- Purpose: Apply data quality checks and schema enforcement

USE customer_analytics;

-- Step 1: Transform and validate
CREATE OR REPLACE TABLE transactions_silver
USING DELTA
LOCATION 's3://data-lake-prod-[YOUR_ACCOUNT_ID]/delta/silver/transactions'
AS
SELECT
    transaction_id,
    customer_id,
    transaction_date,
    CAST(amount AS DECIMAL(10,2))  as amount,          -- enforce monetary type
    UPPER(TRIM(merchant))          as merchant,         -- standardize text
    UPPER(TRIM(category))          as category,
    CASE
        WHEN category NOT IN ('GROCERIES','DINING','RETAIL','OTHER')
        THEN 'OTHER'
        ELSE category
    END                            as category_clean,   -- enforce valid values
    ingestion_time,
    source_file,
    CASE
        WHEN amount < 0            THEN 'negative_amount'
        WHEN amount > 100000       THEN 'very_high_amount'
        WHEN customer_id IS NULL   THEN 'missing_customer'
        ELSE 'valid'
    END                            as data_quality_flag,
    CURRENT_TIMESTAMP()            as silver_processing_time
FROM transactions_bronze
WHERE amount > 0;   -- business rule: only positive amounts

-- Step 2: Data quality report
SELECT
    COUNT(*)                                                   as total_records,
    COUNTIF(data_quality_flag = 'valid')                       as valid_records,
    COUNTIF(data_quality_flag != 'valid')                      as quality_issues,
    ROUND(100.0 * COUNTIF(data_quality_flag = 'valid') / COUNT(*), 2)
                                                               as data_quality_pct
FROM transactions_silver;

-- Step 3: Confirm schema
DESCRIBE transactions_silver;
```

**Why these transformations?**

| Transformation | Reason |
|---------------|--------|
| `CAST(amount AS DECIMAL(10,2))` | Prevents floating-point errors — $10.01 stays $10.01, not $10.009999 |
| `UPPER(TRIM(merchant))` | Normalizes text — "McDonald's", "mcdonald's", "MCDONALD'S" all become "MCDONALD'S" |
| `CASE WHEN category NOT IN (...)` | Enforces valid values — prevents garbage data from breaking downstream reports |
| `WHERE amount > 0` | Business rule — excludes refunds and cancellations (handled separately) |
| `data_quality_flag` | Monitors issues without removing records — tracks quality over time |

#### Step C3: Run the Silver Notebook

**Expected:** Silver table created, data quality percentage shown (should be above 95%), schema displayed, no errors.

---

### Section D: MERGE Operation (Upsert Pattern)

#### Step D1: Create MERGE Notebook

- Name: `03_merge_operation_delta`, Language: SQL

#### Step D2: Daily Update Scenario

> **Real-world context:** Daily new transactions arrive. Some are corrections to existing records. You need to insert new records and update existing ones — atomically.

```sql
-- MERGE OPERATION: The most common production Delta pattern
-- Scenario: Daily batch with some updates to existing records

USE customer_analytics;

-- Step 1: Simulate today's batch (some updated, some new)
CREATE OR REPLACE TEMP VIEW daily_updates AS
SELECT
    transaction_id,
    customer_id,
    transaction_date,
    CAST(amount * 1.02 AS DECIMAL(10,2))  as amount,   -- 2% price adjustment
    merchant,
    category,
    current_timestamp()                   as ingestion_time,
    'daily_batch_20240315'                as source_file,
    'valid'                               as data_quality_flag,
    current_timestamp()                   as silver_processing_time
FROM transactions_silver
WHERE transaction_date = '2024-03-15'
LIMIT 100;

-- Step 2: Preview the batch
SELECT COUNT(*) as records_to_upsert FROM daily_updates;

-- Step 3: MERGE — atomic update or insert
MERGE INTO transactions_silver s
USING daily_updates u
ON s.transaction_id = u.transaction_id   -- match condition

WHEN MATCHED THEN
    UPDATE SET
        amount                 = u.amount,
        merchant               = u.merchant,
        silver_processing_time = u.silver_processing_time

WHEN NOT MATCHED THEN
    INSERT (transaction_id, customer_id, transaction_date, amount,
            merchant, category, ingestion_time, source_file,
            data_quality_flag, silver_processing_time)
    VALUES (u.transaction_id, u.customer_id, u.transaction_date, u.amount,
            u.merchant, u.category, u.ingestion_time, u.source_file,
            u.data_quality_flag, u.silver_processing_time);

-- Step 4: Verify result
SELECT COUNT(*) as total_records_after_merge FROM transactions_silver;
```

**MERGE explained:**

```
MERGE INTO target_table
USING source_data
ON matching_condition

  WHEN MATCHED   → record exists → UPDATE it
  WHEN NOT MATCHED → new record  → INSERT it

Result: one atomic operation — all succeed or all fail
```

**MERGE vs. traditional ETL:**

| | Traditional | Delta MERGE |
|--|------------|-------------|
| **Data read** | Entire table (50 GB) | Only matching keys (1 GB) |
| **Time** | 1 hour | 3 minutes |
| **Cost** | $1.00 | $0.10 |
| **Crash safety** | Data corrupted | ACID rollback, data consistent |

#### Step D3: Run the MERGE Notebook

**Expected:** Merge completes, total record count displayed, updates and inserts counted.

---

### Section E: Time Travel (Historical Queries)

#### Step E1: Create Time Travel Notebook

- Name: `04_time_travel_delta`, Language: SQL

#### Step E2: Time Travel Queries

```sql
-- TIME TRAVEL: Access previous versions of data
-- Delta automatically versions every write

USE customer_analytics;

-- Step 1: See full version history
DESCRIBE HISTORY transactions_silver;
-- Shows: version | timestamp | operation | user | ...
-- 0  | 2024-03-15 10:00 | CREATE TABLE
-- 1  | 2024-03-15 10:15 | MERGE
-- 2  | 2024-03-15 10:30 | MERGE

-- Step 2: Query a specific version by number
SELECT COUNT(*) as records_at_version_0
FROM transactions_silver VERSION AS OF 0;

-- Step 3: Query at a specific timestamp
SELECT COUNT(*) as records_at_10am
FROM transactions_silver TIMESTAMP AS OF '2024-03-15 10:00:00';

-- Step 4: Compare versions to see what changed
WITH version_0 AS (
    SELECT COUNT(*) as count, SUM(amount) as total
    FROM transactions_silver VERSION AS OF 0
),
current_version AS (
    SELECT COUNT(*) as count, SUM(amount) as total
    FROM transactions_silver
)
SELECT
    version_0.count         as initial_records,
    current_version.count   as current_records,
    current_version.count
      - version_0.count     as records_added,
    version_0.total         as initial_total_amount,
    current_version.total   as current_total_amount,
    current_version.total
      - version_0.total     as amount_change
FROM version_0, current_version;

-- Step 5: Rollback to a previous version (uncomment to use)
-- RESTORE TABLE transactions_silver TO VERSION AS OF 1;

-- Step 6: Full audit trail
SELECT
    version,
    timestamp,
    operation,
    `user`,
    CAST(operationMetrics.numAddedFiles   AS INT)    as files_added,
    CAST(operationMetrics.numRemovedFiles AS INT)    as files_removed,
    CAST(operationMetrics.numOutputBytes  AS BIGINT) as bytes_changed
FROM (SELECT * FROM table_changes('transactions_silver', 0))
WHERE version > 0
ORDER BY version DESC;
```

**Time travel real-world scenario:**

```
2:00 PM  Analyst notices bad pricing data
2:05 PM  Runs DESCRIBE HISTORY — sees problem introduced at version 5 (1:30 PM)
2:06 PM  Queries VERSION AS OF 4 — confirms data was clean before that
2:07 PM  RESTORE TABLE TO VERSION AS OF 4
2:08 PM  Table fully restored — zero downtime, automatic recovery

Total incident time: 8 minutes (vs. 8 hours with traditional backups)
```

#### Step E3: Run the Time Travel Notebook

**Expected:** Version history displayed, version counts match, audit trail visible.

---

### Section F: Schema Enforcement and Evolution

#### Step F1: Create Schema Notebook

- Name: `05_schema_enforcement_delta`, Language: SQL

#### Step F2: Schema Rules

```sql
-- SCHEMA ENFORCEMENT: Prevent bad data at the table level

USE customer_analytics;

-- Step 1: View current schema
DESCRIBE transactions_silver;

-- Step 2: Schema enforcement blocks wrong types
-- Uncomment to see the error Delta returns:
-- INSERT INTO transactions_silver
-- SELECT * EXCEPT (amount), 'NOT_A_NUMBER' as amount
-- FROM transactions_silver LIMIT 1;
-- ERROR: cannot cast string to decimal

-- Step 3: Schema evolution — safely add a new column (zero downtime)
ALTER TABLE transactions_silver
ADD COLUMN fraud_score DOUBLE DEFAULT 0.0;

-- Step 4: Verify new schema
DESCRIBE transactions_silver;

-- Step 5: Populate the new column
UPDATE transactions_silver
SET fraud_score = CASE
    WHEN amount > 5000 THEN 0.8
    WHEN amount > 1000 THEN 0.3
    ELSE 0.1
END;

-- Step 6: Time travel still works — old versions return NULL for new columns
SELECT
    COUNT(*)             as records,
    COUNT(fraud_score)   as fraud_score_not_null
FROM transactions_silver VERSION AS OF 0;
-- Result: fraud_score IS NULL in version 0 (column didn't exist yet)
```

**Schema evolution: Delta vs. traditional approach**

| | Traditional | Delta |
|--|------------|-------|
| **Process** | Stop all readers → add column → restart readers | `ALTER TABLE ... ADD COLUMN` (instant) |
| **Downtime** | Yes — users affected | Zero |
| **Old readers** | Break until restarted | Still work (get NULL for new column) |
| **New readers** | Work after restart | Immediately use `fraud_score` |

#### Step F3: Run the Schema Notebook

**Expected:** Schema displayed, new column added, time travel still works with NULL for new column in old versions.

---

## Part 6: Production Best Practices

**Optimize Delta files** — Delta creates many small files over time. Compact them:

```sql
-- Consolidate small files for faster queries
OPTIMIZE transactions_silver;

-- Delete old versions after 7 days (free up storage)
VACUUM transactions_silver RETAIN 7 DAYS;
```

**Monitor table health:**

```sql
SELECT
    table_name,
    num_files,
    total_size,
    num_rows
FROM system.information_schema.tables
WHERE schema_name = 'customer_analytics';
```

**Row-level security:**

```sql
-- Users only see their own data
CREATE VIEW transactions_user_view AS
SELECT *
FROM transactions_silver
WHERE customer_id = current_user();
```

---

## Part 7: Success Criteria

- [ ] Bronze Delta table created from Parquet
- [ ] Silver Delta table created with transformations
- [ ] MERGE operation executed (updates and inserts)
- [ ] Time travel queries returned results
- [ ] Schema enforcement and evolution working
- [ ] Version history shows all operations
- [ ] Can explain why ACID matters for production
- [ ] Can explain when to use Delta vs. Parquet
- [ ] Understand the transaction log concept

**If all items are checked, you have completed Lab 6.1!**

---

## Part 8: Teardown and Cost Management

### What Costs Money

| Item | Cost |
|------|------|
| Databricks cluster | $0.25/hour while running |
| S3 Delta tables | ~$0.50 total |
| AWS data transfer | ~$0.01 |
| **Total** | **~$1–2** |

### Cleanup

- **Stop cluster:** Compute → click cluster name → **Terminate** → confirm

- **Drop Delta tables** (optional — keep if you'll do Lab 6.2):

```sql
DROP TABLE transactions_bronze;
DROP TABLE transactions_silver;
```

- **Delete S3 data:** S3 console → `data-lake-prod-[ACCOUNT_ID]` → delete `delta/` folder

---

## Part 9: What You Learned

| Area | Skills Gained |
|------|--------------|
| **ACID transactions** | Why atomicity, consistency, isolation, and durability matter in data lakes |
| **Transaction logs** | How Delta records every change and enables recovery |
| **Bronze/silver/gold pattern** | Data lake architecture and quality layering |
| **MERGE operations** | Production upsert — 10x faster and 90% cheaper than full table rewrites |
| **Time travel** | Version queries, timestamp queries, rollback, audit trails |
| **Schema enforcement** | Blocking invalid data types at the table level |
| **Schema evolution** | Adding columns safely with zero downtime |
| **Delta vs. Parquet** | When to use each — decision framework for real-world design |

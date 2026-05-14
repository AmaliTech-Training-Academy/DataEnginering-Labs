# CDC with DMS + Incremental Loading

---

## Part 0: Understanding Why This Matters

### The Batch Loading Problem

It's 2 AM. Your company's ETL pipeline runs a full load:

```
Read ENTIRE customer table (1 million rows) → 20 minutes
Upload to S3                                → 10 minutes
Process duplicates                          → 5 minutes
Total                                       → 35 minutes
```

Meanwhile: only 500 of those 1 million rows actually changed.
That's **99.95% wasted work** — every single night.

### The Real-World Cost

Company: e-commerce site with 50M customer records.

| | Full Load Every Hour | CDC Incremental (Every 5 Minutes) |
|--|----------------------|----------------------------------|
| **Rows read** | 50M/hour x 24 = 1.2B rows/day | ~5K changed records/hour |
| **Data transfer** | 600 GB/day | 60 MB/day |
| **Monthly cost** | $1,620 (data transfer alone) | Under $1 |
| **Data freshness** | 1 hour stale | 5 minutes stale |
| **Savings** | | 99.97% cheaper, 12x fresher |

### What is CDC? (Change Data Capture)

**CDC = "Tell me what changed, not everything."**

| Time | Action | Full Load | CDC |
|------|--------|-----------|-----|
| 08:00 | Customer 1 inserted | Read 1M rows | Record: INSERT 1 |
| 08:05 | Customer 2 inserted | Read 1M rows | Record: INSERT 2 |
| 08:10 | Customer 3 updated | Read 1M rows | Record: UPDATE 3 |
| 08:15 | Customer 4 inserted | Read 1M rows | Record: INSERT 4 |
| 09:00 | Customer 5 deleted | Read 1M rows | Record: DELETE 5 |
| **Total** | | **12M row scans** | **5 records** |

**Difference:** CDC is 2.4 million times more efficient in this example.

### Real-World Companies Using CDC

| Company | Scale | Problem | CDC Solution | Benefit |
|---------|-------|---------|--------------|---------|
| LinkedIn | 800M profiles | Can't reload 800M profiles hourly | Kafka + CDC | Real-time member search |
| Uber | Billions of trips | Full load overwhelms database | DMS + CDC | Real-time driver matching |
| Spotify | 500M songs | Playlist changes every millisecond | Event streaming + CDC | Real-time recommendations |
| Netflix | Millions of titles | Metadata changes hourly | Custom CDC | Instant catalog updates |

### CDC Types

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Query-based** | Add `updated_at` column, query `WHERE updated_at > last_run` | Simple, works on any DB | Misses deletes, adds query load |
| **Log-based (DMS)** | Read database transaction logs directly | Captures everything (inserts, updates, deletes) | More complex, varies by DB type |
| **Hybrid** | Combine both | Most reliable | Balanced complexity and cost |

### Why AWS DMS?

**Custom CDC — months of work:**

- Connect to MySQL binary logs
- Parse binary format
- Track log position
- Manage failures and retries
- Scale across multiple tables
- Handle format differences per database type

**AWS DMS — hours to set up:**

- Managed service, handles all of the above automatically
- Works with all major database types
- Built-in error handling and failover
- Pay only for what you use

### Real-World Example: Shopify Order Processing

```
MySQL database (orders)
        ↓
DMS task (CDC enabled)
  Captures: INSERT (new order), UPDATE (shipped), DELETE (canceled)
        ↓
S3 CDC files (incremental)
  2024-03-06T10:00_insert.parquet  ← 500 new orders
  2024-03-06T10:00_update.parquet  ← 100 status changes
  2024-03-06T10:00_delete.parquet  ← 2 cancellations
        ↓
Glue ETL job
  INSERT → add new order
  UPDATE → modify order status
  DELETE → remove canceled order
        ↓
Redshift data warehouse
  Orders table always up to date
        ↓
Analytics dashboards
  Real-time order metrics
```

### Your Career Path

- **Junior:** "How do I load all data every night?"
- **Senior:** "We need CDC for near-real-time sync"
- **Staff:** "I designed our CDC infrastructure — it saves the company $1M/year"

---

## Part 1: Goals for This Lab

By the end of this lab, you will:

- Understand CDC concepts and why they matter
- Set up AWS DMS for incremental data capture
- Create a sample source database (RDS)
- Configure a DMS replication task with CDC enabled
- Capture INSERT, UPDATE, and DELETE changes
- Load incremental data to S3
- Apply changes using Glue
- Verify data consistency

---

## Part 2: What You'll Create

### Project Overview: E-Commerce Order Sync

**Scenario:**

- **Source:** RDS MySQL with order data
- **Changes:** Orders being created, updated, shipped, and canceled
- **Goal:** Keep the S3 data lake in sync with the source system
- **Approach:** DMS with CDC for incremental loads

**Data flow:**

```
RDS MySQL (source)
  Initial state:    100 orders
  Then:           + 10 new orders
  Then:           ~ 5 orders updated (marked shipped)
  Then:           - 2 orders deleted (canceled)
        ↓
DMS task (CDC enabled)
        ↓
S3 CDC files
  FULL_LOAD/       ← initial 100 orders
  CDC_capture_1/   ← 10 inserts
  CDC_capture_2/   ← 5 updates
  CDC_capture_3/   ← 2 deletes
        ↓
Glue job (apply changes)
  Merges INSERT, UPDATE, DELETE into master table
        ↓
Final data lake
  103 orders (100 + 10 - 2, with 5 updates applied)
  Always in sync with source
```

### DMS with CDC vs. Traditional Sync

| Time | MySQL | Traditional | DMS with CDC |
|------|-------|-------------|--------------|
| 08:00 | 100 orders | Full load starts | Full load done |
| 08:01 | +10 new orders | Waiting for next schedule | CDC captures 10 inserts |
| 08:02 | 110 orders | Still showing 100 (stale) | Glue applies changes: 110 orders |
| 09:00 | 110 orders | Full reload runs again | Still in sync (seconds latency) |

### Components You'll Use

| Component | Role |
|-----------|------|
| RDS MySQL | Source system — simulates e-commerce order database |
| DMS replication task | Initial full load + ongoing CDC capture |
| S3 CDC files | `FULL_LOAD/` for initial data, timestamped folders for changes |
| Glue job | Reads CDC files, applies changes to master table |
| S3 data lake | Final processed orders, always in sync, ready for analytics |

---

## Part 3: The "Why" Behind Design Choices

### Why Separate INSERT, UPDATE, and DELETE Folders?

| | All Mixed Together | Separate by Operation |
|--|-------------------|-----------------------|
| **Delete handling** | Hard to identify and apply | `DELETE/` folder is explicit |
| **Update logic** | Unclear whether to add or replace | `UPDATE/` folder = find and replace |
| **Risk** | Accidentally applying a delete twice | Clear, verifiable logic |
| **Debugging** | Hard to trace what happened | Easy to inspect each operation |

### Why S3 + Glue Instead of Direct Real-Time Sync?

| | Direct Real-Time Sync | S3 + Glue (Near Real-Time) |
|--|----------------------|---------------------------|
| **Latency** | Instant | Minutes |
| **Auditability** | Hard | Files preserved for inspection |
| **Repeatability** | Hard to reprocess | Re-run Glue job anytime |
| **Coupling** | Tightly coupled (risky) | Loosely coupled (safer) |
| **Best for** | Stock prices, live dashboards | Analytics, reporting, BI |

### Why Parquet for CDC Output?

| | CSV | Parquet |
|--|-----|---------|
| **Type safety** | Everything is a string | Column types stored natively |
| **File size** | Large (text-based) | Compressed and small |
| **Query speed** | Full file scan required | Columnar — reads only needed columns |
| **Partition support** | Poor | Native partition-aware reads |

---

## Part 4: Prerequisites

- Completed Labs 4.1 and 4.2
- RDS instance (free tier available)
- AWS account with DMS permissions in IAM role
- Data lake S3 bucket from Tier 1
- 4.5 hours for setup and execution

---

## Part 5: Prepare Source Database

### Step 5.1: Approach for This Lab

Setting up a full RDS instance takes 30+ minutes. Instead, we'll create sample CSV files representing the database tables and CDC captures. This demonstrates the exact same CDC concepts with faster setup.

### Step 5.2: Create Source Data

Create a file named `source_orders.csv` — this represents the initial state of the orders table:

```csv
order_id,customer_id,order_date,total_amount,status
1,101,2024-01-15,99.99,completed
2,102,2024-01-16,149.99,completed
3,103,2024-01-17,79.99,shipped
4,104,2024-01-18,199.99,processing
5,105,2024-01-19,59.99,completed
6,106,2024-01-20,89.99,shipped
7,107,2024-01-21,129.99,completed
8,108,2024-01-22,249.99,processing
9,109,2024-01-23,69.99,completed
10,110,2024-01-24,109.99,shipped
```

### Step 5.3: Create CDC Capture Files

These simulate what DMS would write as changes happen in the source database.

**`cdc_insert_batch1.csv`** — 3 new orders:

```csv
operation,order_id,customer_id,order_date,total_amount,status
I,11,111,2024-02-01,119.99,processing
I,12,112,2024-02-02,139.99,processing
I,13,113,2024-02-03,99.99,processing
```

**`cdc_update_batch1.csv`** — 3 orders marked completed:

```csv
operation,order_id,customer_id,order_date,total_amount,status,prev_status
U,3,103,2024-01-17,79.99,completed,shipped
U,4,104,2024-01-18,199.99,completed,processing
U,6,106,2024-01-20,89.99,completed,shipped
```

**`cdc_delete_batch1.csv`** — 2 orders canceled:

```csv
operation,order_id,customer_id,order_date,total_amount,status
D,2,102,2024-01-16,149.99,completed
D,8,108,2024-01-22,249.99,processing
```

> **Operation codes:** `I` = INSERT, `U` = UPDATE, `D` = DELETE

### Step 5.4: Upload to S3

Create this folder structure and upload each file:

```
s3://data-lake-prod-[ACCOUNT_ID]/cdc/orders/
  source/    ← upload source_orders.csv here
  changes/   ← upload all three CDC files here
  merged/    ← Glue will write output here
```

---

## Part 6: Create CDC Processing Glue Job

### Step 6.1: Create the Job

- Glue console → **Jobs** → **Create job**
- Fill in:

| Field | Value |
|-------|-------|
| Name | `OrderCDCMerge` |
| IAM role | `GlueServiceRole` |
| Language | Python |
| DPU | 2 |

### Step 6.2: Write CDC Merge Logic

Replace the default code with:

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from pyspark.context import SparkContext
from awsglue.job import Job
from pyspark.sql.functions import *
from pyspark.sql.window import Window

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Replace with your account ID
ACCOUNT_ID = "123456789012"   # CHANGE THIS!
BUCKET = f"data-lake-prod-{ACCOUNT_ID}"
BASE_PATH = f"s3://{BUCKET}/cdc/orders"

print("=" * 70)
print("ORDER CDC MERGE JOB - Apply changes to master table")
print("=" * 70)

try:
    # ===== STEP 1: Load source data =====
    print("\n[STEP 1] Loading initial source data...")
    df_source = spark.read.csv(
        f"{BASE_PATH}/source/source_orders.csv",
        header=True,
        inferSchema=True
    )
    source_count = df_source.count()
    print(f"Loaded {source_count} source records")
    df_source.show(3)

    df_master = df_source.select(
        "order_id", "customer_id", "order_date", "total_amount", "status"
    )

    # ===== STEP 2: Load CDC changes =====
    print("\n[STEP 2] Loading CDC change files...")
    df_cdc = spark.read.csv(
        f"{BASE_PATH}/changes/*.csv",
        header=True,
        inferSchema=True
    )
    cdc_count = df_cdc.count()
    print(f"Loaded {cdc_count} CDC change records")

    # Separate by operation type
    df_insert = df_cdc.filter(col("operation") == "I").select(
        "order_id", "customer_id", "order_date", "total_amount", "status"
    )
    df_update = df_cdc.filter(col("operation") == "U").select(
        "order_id", "customer_id", "order_date", "total_amount", "status"
    )
    df_delete = df_cdc.filter(col("operation") == "D").select("order_id")

    insert_count = df_insert.count()
    update_count = df_update.count()
    delete_count = df_delete.count()

    print(f"  INSERT: {insert_count} records")
    print(f"  UPDATE: {update_count} records")
    print(f"  DELETE: {delete_count} records")

    # ===== STEP 3: Apply INSERTs =====
    print("\n[STEP 3] Applying INSERT operations...")
    df_master = df_master.unionByName(df_insert)
    print(f"After INSERT: {df_master.count()} total records")

    # ===== STEP 4: Apply UPDATEs =====
    print("\n[STEP 4] Applying UPDATE operations...")
    if update_count > 0:
        # Remove old versions of updated records
        df_master = df_master.join(
            df_update.select("order_id").distinct(),
            on="order_id",
            how="left_anti"
        )
        # Add updated versions
        df_master = df_master.unionByName(df_update)
        print(f"After UPDATE: {df_master.count()} total records")
    else:
        print("No UPDATE records")

    # ===== STEP 5: Apply DELETEs =====
    print("\n[STEP 5] Applying DELETE operations...")
    if delete_count > 0:
        df_master = df_master.join(
            df_delete.select("order_id").distinct(),
            on="order_id",
            how="left_anti"
        )
        print(f"After DELETE: {df_master.count()} total records")
    else:
        print("No DELETE records")

    # ===== STEP 6: Verify data quality =====
    print("\n[STEP 6] Verifying data quality...")

    master_count = df_master.count()
    df_deduped = df_master.dropDuplicates(["order_id"])
    dedup_count = df_deduped.count()

    if dedup_count == master_count:
        print(f"No duplicates found ({master_count} unique orders)")
    else:
        print(f"WARNING: Found {master_count - dedup_count} duplicates — removing")
        df_master = df_deduped

    null_keys = df_master.filter(col("order_id").isNull()).count()
    if null_keys == 0:
        print("No null order_id values")
    else:
        print(f"WARNING: Found {null_keys} records with null order_id")

    # ===== STEP 7: Write merged data =====
    print("\n[STEP 7] Writing merged data to S3...")
    output_path = f"{BASE_PATH}/merged/"
    df_master.write.mode("overwrite").parquet(output_path)
    final_count = df_master.count()
    print(f"Written {final_count} merged records to {output_path}")

    # ===== STEP 8: Show final results =====
    print("\n[STEP 8] Final data snapshot:")
    df_master.orderBy("order_id").show(10, truncate=False)

    # Summary
    print("\n" + "=" * 70)
    print("CDC MERGE SUMMARY")
    print("=" * 70)
    print(f"Source records:        {source_count}")
    print(f"Inserted:            + {insert_count}")
    print(f"Updated:             ~ {update_count}")
    print(f"Deleted:             - {delete_count}")
    print(f"Final merged records:  {final_count}")
    print("=" * 70)
    print("JOB COMPLETED SUCCESSFULLY")
    print("=" * 70)

except Exception as e:
    print(f"\nERROR: {str(e)}")
    import traceback
    traceback.print_exc()
    raise

finally:
    job.commit()
```

### Step 6.3: Run the Job

- Click **Save**
- Click **Run job**
- Wait 2–3 minutes for completion

### Step 6.4: Review Results

Expected log output:

```
[STEP 1] Loading initial source data...
  Loaded 10 source records

[STEP 2] Loading CDC change files...
  Loaded 8 CDC change records
  INSERT: 3 records
  UPDATE: 3 records
  DELETE: 2 records

[STEP 3] Applying INSERT operations...
  After INSERT: 13 total records

[STEP 4] Applying UPDATE operations...
  After UPDATE: 13 total records

[STEP 5] Applying DELETE operations...
  After DELETE: 11 total records

[STEP 6] Verifying data quality...
  No duplicates found (11 unique orders)
  No null order_id values

[STEP 7] Writing merged data to S3...
  Written 11 merged records

CDC MERGE SUMMARY
  Source records:        10
  Inserted:            + 3
  Updated:             ~ 3
  Deleted:             - 2
  Final merged records:  11

JOB COMPLETED SUCCESSFULLY
```

---

## Part 7: Verify Your CDC Implementation

### Step 7.1: Check Output Data

- S3 console → navigate to `s3://data-lake-prod-[ID]/cdc/orders/merged/`
- Confirm Parquet files are present

### Step 7.2: Verify the Math

```
Initial:    10 records
+ Insert:  + 3 (orders 11, 12, 13)
= Total:    13
~ Update:  ~ 3 (orders 3, 4, 6 — status changed, count stays 13)
- Delete:  - 2 (orders 2 and 8 removed)
= Final:    11 records
```

### Step 7.3: Verify Data Integrity

Records that should be in the final dataset:

- **Present:** 1, 3 (updated), 4 (updated), 5, 6 (updated), 7, 9, 10, 11, 12, 13
- **Missing:** 2 (deleted), 8 (deleted)
- **Updated:** Orders 3, 4, 6 should have `status = "completed"`

---

## Part 8: Real DMS Setup (Production Reference)

For production, you would follow these steps with actual AWS DMS:

| Step | Action | Key Settings |
|------|--------|-------------|
| 1 | Create RDS MySQL | `db.t3.micro`, 20 GB, free tier eligible |
| 2 | Enable binary logging | `log_bin=1`, `binlog_format=ROW`, retain 24 hours |
| 3 | Create DMS replication instance | `dms.t3.micro`, single-AZ for dev/test |
| 4 | Create source endpoint | Engine: MySQL, host: RDS endpoint, port: 3306 |
| 5 | Create target endpoint | Type: S3, bucket: your data lake, format: Parquet |
| 6 | Create replication task | Full load + CDC, select orders table, limited LOB mode |
| 7 | Start task | Full load completes, CDC begins capturing changes |
| 8 | Monitor | CloudWatch for metrics, DMS console for task status |

**Time for real setup:** 20–30 minutes (mostly configuration clicks).

---

## Part 9: CDC Patterns in Production

### Pattern 1: Streaming CDC (Seconds Latency)

```
RDS → DMS → Kafka → consumers
  Analytics engine reads Kafka
  Dashboards update in real-time
  Best for: fraud detection, live pricing
```

### Pattern 2: Batch CDC to Data Lake (Minutes/Hours Latency)

```
RDS → DMS → S3 CDC files → Glue → Redshift
  1-hour latency is acceptable
  Cost optimized (hourly writes)
  Best for: analytics, reporting (this lab)
```

### Pattern 3: Hybrid — Streaming + Batch

```
High-velocity data (orders):
  Orders DB → Kafka → real-time aggregation

Low-velocity data (customers):
  Customers DB → DMS hourly → data lake

Combined:
  Real-time decisions on orders
  Analytics on both in dashboards
  Cost optimized by matching approach to data velocity
```

---

## Part 10: Teardown

### Step 1: Delete Glue Job

- Glue console → **Jobs**
- Select `OrderCDCMerge` → **Delete** → confirm

### Step 2: Clear S3 Data (Optional)

- S3 console → navigate to `cdc/orders/`
- Delete all folders

### Step 3: Verify Cleanup

- [ ] Glue job deleted
- [ ] S3 data removed
- [ ] No orphaned resources

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **CDC concepts** | Full load vs. incremental; why CDC saves cost and improves freshness |
| **Operation types** | INSERT, UPDATE, DELETE — how to capture and apply each |
| **AWS DMS** | What it does; when to use it; real setup steps for production |
| **Glue CDC job** | Reading CDC files; applying changes in correct order; merge logic |
| **Data quality** | Deduplication after merge; null key checks; verifying the math |
| **CDC file organization** | Source, changes, and merged folder structure |
| **Production patterns** | Streaming vs. batch CDC; hybrid approach; when to use each |
| **Cost awareness** | 99.97% savings over full loads; S3 + Glue as cost-effective architecture |

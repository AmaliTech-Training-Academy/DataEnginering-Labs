# Glue ETL with PySpark

---

## Part 0: Understanding Why This Matters

### The Real-World Problem

Imagine you work at a streaming company. Every day you receive:

- Customer clickstream logs (messy, unstructured)
- Product catalog updates (different formats)
- Payment transactions (various currencies)
- User behavior events (duplicates, nulls, inconsistencies)

**Your job: turn this garbage into gold.**

| Without ETL | With ETL |
|------------|---------|
| "Our analytics are wrong" — garbage in, garbage out | Clean, standardized data |
| "The dashboards don't match reality" — trust issues | Trustworthy analytics |
| "We can't do ML with this data" — broken pipelines | ML-ready datasets |
| "Compliance audit failed" — regulatory risk | Regulatory compliance |

### Why PySpark in Glue?

**PySpark = Python + Apache Spark**

| | Normal Python | PySpark |
|--|--------------|---------|
| **Processing** | One row at a time | Millions of rows in parallel |
| **Scale** | Limited to single machine RAM | Distributed across multiple machines |
| **Speed on 1 billion transactions** | 2 hours | 2 minutes (60x faster) |

Glue handles all the infrastructure — you just write the PySpark code.

### Glue = Your Data Transformation Platform

| Traditional Data Engineering | Glue (Serverless) |
|-----------------------------|-------------------|
| Buy expensive hardware | Write ETL code |
| Install and configure Hadoop/Spark | Click Run |
| Manage clusters, handle failures | Glue provisions, scales, monitors, and recovers automatically |
| Pay for idle time | Pay only for compute time used |
| Infrastructure becomes the job | You focus on data, not servers |

### Why This Matters for Your Career

- Every data company uses ETL (Glue, Spark, Airflow, dbt)
- Data engineers spend **80% of their time** on ETL
- Common interview question: "Tell us about an ETL pipeline you built"
- Strong ETL skills = **20% higher salary** on average
- ETL architecture experience leads to senior and staff engineer roles

---

## Part 1: Goals for This Lab

By the end of this lab, you will:

- Understand how ETL jobs work
- Write a real PySpark transformation job in Glue
- Process messy sample data (clean, deduplicate, enrich)
- Monitor job execution
- Debug issues when they occur
- Explain why you made specific transformation choices
- Apply real-world ETL patterns

---

## Part 2: What You'll Create

### Project Overview: Customer Data ETL Pipeline

**Raw data (input) — what you start with:**

```
s3://data-lake-raw/customers/
  customer_id:  "C001", "C001", "C002"         ← duplicates!
  email:        "john@example.com", "JOHN@..."  ← inconsistent case
  signup_date:  "2024-01-15", "01/15/2024"      ← different formats
  country:      "US", "usa", "USA"              ← case inconsistency
  status:       "active", null, "inactive"      ← missing values
```

**Transformations applied:**

- Remove duplicate records
- Standardize email to lowercase
- Parse dates to consistent ISO format
- Standardize country codes to uppercase
- Fill missing status values with "unknown"
- Add metadata columns (`processed_at`, `data_quality_score`)

**Clean data (output) — what you end up with:**

```
s3://data-lake-processed/customers/
  customer_id:          "C001", "C002", "C003"       ← unique
  email:                "john@example.com"            ← consistent
  signup_date:          "2024-01-15"                  ← ISO format
  country:              "US", "CA"                    ← standardized
  status:               "active", "inactive"          ← no nulls
  processed_at:         "2024-03-06T10:30:00Z"
  data_quality_score:   0.95
```

### The PySpark Code You'll Write

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import *

# Initialize Glue
args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Replace with your actual account ID
ACCOUNT_ID = "123456789012"   # CHANGE THIS!
BUCKET_NAME = f"data-lake-prod-{ACCOUNT_ID}"
RAW_PATH = f"s3://{BUCKET_NAME}/raw/customers/"
PROCESSED_PATH = f"s3://{BUCKET_NAME}/processed/customers/"

print("=" * 60)
print("CUSTOMER DATA ETL JOB")
print("=" * 60)

try:
    # ===== EXTRACT =====
    print("\n[1/3] EXTRACT: Reading raw customer data...")
    df = spark.read.csv(RAW_PATH, header=True, inferSchema=True)
    initial_count = df.count()
    print(f"Rows read: {initial_count}")

    # ===== TRANSFORM =====
    print("\n[2/3] TRANSFORM: Cleaning and enriching data...")

    # Remove duplicates (keep first occurrence)
    df_deduped = df.dropDuplicates(["customer_id"])
    dedupe_count = df_deduped.count()
    duplicates_removed = initial_count - dedupe_count
    print(f"Duplicates removed: {duplicates_removed}")
    print(f"Rows after deduplication: {dedupe_count}")

    # Standardize columns
    df_clean = df_deduped.withColumn(
        "email", lower(col("email"))              # lowercase emails
    ).withColumn(
        "country", upper(col("country"))          # uppercase countries
    ).withColumn(
        "status", coalesce(col("status"), lit("unknown"))  # fill nulls
    )

    # Parse date to consistent format
    df_clean = df_clean.withColumn(
        "signup_date", to_date(col("signup_date"), "yyyy-MM-dd")
    )

    # Add metadata columns
    df_clean = df_clean.withColumn(
        "processed_at", current_timestamp()
    ).withColumn(
        "data_quality_score", lit(0.95)
    )

    print("Standardization complete")
    print("Metadata columns added")

    # ===== LOAD =====
    print("\n[3/3] LOAD: Writing processed data to S3...")
    df_clean.write.mode("overwrite").parquet(PROCESSED_PATH)
    final_count = df_clean.count()
    print(f"Rows written: {final_count}")
    print(f"Output path: {PROCESSED_PATH}")

    # Show sample of transformed data
    print("\n" + "=" * 60)
    print("SAMPLE OF TRANSFORMED DATA (first 5 rows)")
    print("=" * 60)
    df_clean.show(5, truncate=False)

    print("\n" + "=" * 60)
    print("JOB COMPLETED SUCCESSFULLY")
    print("=" * 60)

except Exception as e:
    print(f"\nERROR: {str(e)}")
    print("Check CloudWatch logs for details")
    raise

finally:
    job.commit()
```

### Expected Job Results

```
Job run details:
  Status:          SUCCEEDED
  Duration:        2 minutes 15 seconds
  DPU hours:       0.075  (2 DPU x 2.25 min / 60)
  Cost:            $0.0165  (0.075 x $0.22/DPU-hour)
  Rows processed:  10,000
  Rows output:     9,950  (50 duplicates removed)
  Output size:     1.2 MB Parquet
```

---

## Part 3: The "Why" Behind Design Decisions

### Why Glue Instead of Running Spark Locally?

| | Local Spark (Laptop) | Glue Spark |
|--|---------------------|------------|
| **RAM** | 8 GB | On-demand, auto-scales |
| **Process 1 GB file** | 5 minutes | 5 minutes |
| **Process 100 GB file** | Out of memory | 5 minutes (20x faster) |
| **Scaling** | Manual | Automatic up to 1,000 cores |
| **Cost** | Always on | Pay only while running |

### Why PySpark Instead of Pandas?

**Pandas — works for small data:**

```python
import pandas as pd
df = pd.read_csv("huge_file.csv")   # entire file must fit in RAM
df = df.drop_duplicates()
df.to_csv("output.csv")
```

**PySpark — works for any size:**

```python
df = spark.read.csv("huge_file.csv", header=True)  # doesn't load all into RAM
df = df.dropDuplicates()                            # processes in parallel across cluster
df.write.parquet("output.parquet")
```

### Why Parquet Instead of CSV?

| | CSV | Parquet |
|--|-----|---------|
| **Format** | Text-based | Binary, columnar |
| **File size (same data)** | 1.2 MB | 0.012 MB (100x smaller) |
| **Column types** | Everything is a string | Types stored natively |
| **Query speed** | Must scan entire file | Can read single column only |
| **Analytics performance** | Slow | Up to 100x faster |

### Why Deduplication First?

If you skip deduplication, every metric downstream is wrong:

| Metric | With Duplicates | After Deduplication |
|--------|----------------|---------------------|
| **Customer count** | 1,000 (wrong) | 950 (accurate) |
| **Revenue** | $1,000,000 (double-counted) | $500,000 (real) |
| **Business decisions** | Based on bad data | Trustworthy |

### Why Standardization Matters

| Problem | Raw Data | Standardized |
|---------|----------|-------------|
| **Country** | "USA", "us", "Us", "US" | "US" (always) |
| **Email** | "John@Example.com", "john@example.com" | "john@example.com" (always) |
| **Date** | "2024-01-15", "01/15/2024" | "2024-01-15" (always) |
| **Impact** | JOIN operations fail, aggregations wrong | All operations work correctly |

### Why Monitor Job Execution?

Glue jobs can fail due to out-of-memory errors, network timeouts, invalid data formats, S3 permission issues, or schema mismatches.

| | Without Monitoring | With Monitoring |
|--|-------------------|-----------------|
| **Job fails** | Silently — no one knows | Logs show exactly what went wrong |
| **Downstream effect** | Old/stale data used for days | Fix and re-run quickly |
| **Dashboard impact** | Shows stale results | Stays current |

---

## Part 4: Prerequisites

- Completed Tier 1 labs (IAM roles, S3 bucket created)
- Completed Tier 2 labs (Glue catalog configured, sample data in S3)
- Completed Tier 3 labs (familiarity with data formats and S3)
- AWS console access with permissions for Glue, S3, CloudWatch
- Basic Python knowledge (variables, functions, if/for, lists, dicts)

---

## Part 5: Prepare Your Data

### Step 5.1: Create Sample Customer Data

Create a file named `customers.csv` with this content:

```csv
customer_id,email,signup_date,country,status
C001,john@example.com,2024-01-15,US,active
C002,jane@example.com,2024-01-20,CA,active
C003,bob@example.com,2024-01-25,US,inactive
C001,JOHN@EXAMPLE.COM,2024-01-15,usa,active
C004,alice@example.com,2024-02-01,,unknown
C005,charlie@example.com,01/15/2024,US,
C006,diana@example.com,2024-02-10,us,active
C002,jane@example.com,2024-01-20,Canada,active
C007,eve@example.com,2024-02-15,GB,active
C008,frank@example.com,2024-02-20,AU,active
```

**Notice the intentional messiness:**

- Row 4: Duplicate C001 with different case email
- Row 5: Missing country
- Row 6: Date in wrong format, missing status
- Row 7: Country in lowercase
- Row 8: Duplicate C002 with country spelled out

This is what real data looks like.

### Step 5.2: Upload Sample Data to S3

- Go to AWS Console → **S3**
- Click your bucket (`data-lake-prod-[ACCOUNT_ID]`)
- Create folder structure:
  - Create folder: `raw`
  - Inside `raw`, create folder: `customers`
- Navigate into `raw/customers/`
- Click **Upload** → select `customers.csv` → click **Upload**

Your file should now be at:

```
s3://data-lake-prod-[ACCOUNT_ID]/raw/customers/customers.csv
```

**Why this folder structure?**

```
s3://data-lake-prod-[ACCOUNT_ID]/
  raw/          ← original data from sources, as-is
    customers/
    transactions/
    products/
  processed/    ← cleaned and deduplicated
    customers/
    transactions/
  curated/      ← business-ready data
    customer_dim/
    fact_sales/
```

This structure makes the quality level of data immediately clear to anyone on the team.

---

## Part 6: Create Your Glue Job

### Step 6.1: Navigate to Glue Console

- Click **Services** → search `glue` → click **AWS Glue**
- Click **Jobs** in the left sidebar (under the Authoring section)

### Step 6.2: Create New Job

- Click **Create job**
- Fill in the settings:

| Field | Value |
|-------|-------|
| Name | `CustomerDataETL` |
| IAM role | `GlueServiceRole` |
| Type | Spark |
| Glue version | Keep default (4.0) |
| Language | Python |
| DPU allocation | 2 |

- Click **Create job**

**Expected:** Job created, redirected to job editor with Python code editor visible.

### Step 6.3: Write Your ETL Code

- Select all default code in the editor (`Ctrl+A`) and delete it
- Paste the full code from Part 2
- Replace `"123456789012"` with your actual AWS account ID (find it in the top-right dropdown of the AWS console)

### Step 6.4: Save the Job

Click **Save** (top right). You should see a success message.

---

## Part 7: Run Your ETL Job

### Step 7.1: Execute the Job

- Click **Run job** (orange button, top right)
- Status changes to `RUNNING` — Spark cluster is initializing (1–2 minutes)

### Step 7.2: Monitor Execution

Click **Run 1** after the job starts. Watch the status field:

```
RUNNING    →  Initializing cluster
RUNNING    →  Processing data
RUNNING    →  Writing output
SUCCEEDED  →  Job complete
```

First run takes 2–5 minutes due to cluster startup.

### Step 7.3: View Logs

Once complete, click the **Logs** tab. You should see:

```
============================================================
CUSTOMER DATA ETL JOB
============================================================

[1/3] EXTRACT: Reading raw customer data...
  Rows read: 10

[2/3] TRANSFORM: Cleaning and enriching data...
  Duplicates removed: 2
  Rows after deduplication: 8
  Standardization complete
  Metadata columns added

[3/3] LOAD: Writing processed data to S3...
  Rows written: 8
  Output path: s3://data-lake-prod-123456789012/processed/customers/

============================================================
JOB COMPLETED SUCCESSFULLY
============================================================
```

### Step 7.4: View Results in S3

- Go to S3 console
- Navigate to `s3://data-lake-prod-[ACCOUNT_ID]/processed/customers/`
- You should see multiple `.parquet` files and a `_SUCCESS` file

> Multiple Parquet files is normal — Spark writes in parallel, one file per executor partition. Combined, they form the complete dataset.

---

## Part 8: Verify Your Work

**Job execution checklist:**

- [ ] Job created named `CustomerDataETL`
- [ ] Job uses `GlueServiceRole`
- [ ] Language is Python, DPU is 2
- [ ] Account ID replaced in code
- [ ] Job status shows `SUCCEEDED`
- [ ] Logs show: rows read: 10, duplicates removed: 2, rows written: 8
- [ ] No error messages in logs

**Data output checklist:**

- [ ] Parquet files exist in `processed/customers/`
- [ ] `_SUCCESS` file is present
- [ ] File size is greater than 0 KB

**Understanding checklist:**

- [ ] Can explain why deduplication happened
- [ ] Can explain why standardization matters
- [ ] Know what Parquet is and why it's used
- [ ] Know how to run and read logs for a Glue job

**If all items are checked, you have completed this section!**

---

## Part 9: Advanced — Modify the Job

### Challenge 1: Add Data Quality Checks

Add this before the LOAD step:

```python
# Add data quality check
invalid_emails = df_clean.filter(~col("email").contains("@"))
invalid_count = invalid_emails.count()

if invalid_count > 0:
    print(f"WARNING: {invalid_count} invalid emails found")
else:
    print("All emails are valid")

# Add assertions
assert final_count > 0, "No data to write!"
assert duplicates_removed > 0, "No duplicates found — possible data quality issue"
```

### Challenge 2: Add Partition by Date

Change the LOAD step to:

```python
# Write partitioned by signup_date
df_clean.write.mode("overwrite").partitionBy(
    "signup_date"
).parquet(PROCESSED_PATH)

print("Data partitioned by signup_date")
```

### Challenge 3: Add Rejection Handling

Save invalid records to a separate location for review:

```python
# Separate valid and invalid records
valid_data   = df_clean.filter(col("email").contains("@"))
invalid_data = df_clean.filter(~col("email").contains("@"))

# Write valid data to processed path
valid_data.write.mode("overwrite").parquet(PROCESSED_PATH)

# Write rejected records for review
invalid_data.write.mode("overwrite").csv(
    f"s3://{BUCKET_NAME}/rejected/customers/"
)

print(f"Valid records:    {valid_data.count()}")
print(f"Rejected records: {invalid_data.count()}")
```

---

## Part 10: Real-World Patterns You've Learned

### Pattern 1: The Data Lake Zone Model

```
Raw zone (extract)       Processed zone (transform)     Curated zone (consume)
raw/customers/      →    processed/customers/       →    curated/customer_dim/
As-is from source        Deduplicated, standardized       Business-ready entities
```

You just built the raw → processed step.

### Pattern 2: Schema on Read

```python
df = spark.read.csv(RAW_PATH, header=True, inferSchema=True)
```

Schema is inferred when you read, not enforced when data is written. This is flexible and handles messy, evolving data formats gracefully.

### Pattern 3: Idempotent Jobs

```python
df_clean.write.mode("overwrite").parquet(PROCESSED_PATH)
```

Overwrite mode = run it twice, get the same result. Safe to re-run if a job fails. No duplicate data accumulation. This is the industry standard for ETL reliability.

### Pattern 4: Detailed Logging

```
[1/3] EXTRACT
[2/3] TRANSFORM
[3/3] LOAD
```

Production ETL jobs always log at each stage for debugging, pipeline monitoring, data lineage tracking, and alerting on failures.

---

## Part 11: Monitoring and Troubleshooting

### Common Issues and Fixes

| Issue | Error | Cause | Fix |
|-------|-------|-------|-----|
| **Wrong path** | No such file or directory | Bucket name or path is wrong | Check bucket name, verify file was uploaded |
| **Missing permissions** | User is not authorized | GlueServiceRole missing S3 access | Verify IAM role has `AmazonS3FullAccess` |
| **Bad data or memory** | Task failed | Out of memory or unexpected data format | Increase DPU, check data format, review logs |
| **Job hanging** | Job run timeout | Infinite loop or very slow processing | Add timeout limits, optimize code, add partitions |

### How to Debug

- **Always check logs first** — click Run details → Logs tab, search for `ERROR` or `Exception`
- **Add more print statements** — count rows before and after each transformation
- **Test with smaller data first** — confirm logic works on 100 rows before running on 1M

---

## Part 12: Cost Optimization

### Cost Breakdown for This Lab

| Item | Calculation | Cost |
|------|------------|------|
| Glue job | 2 DPU x (2/60 hrs) x $0.22/DPU-hr | ~$0.02 |
| S3 input | 0.5 KB raw CSV | ~$0.00 |
| S3 output | 0.01 KB Parquet (100x more efficient) | ~$0.00 |
| **Total** | | **~$0.02** |

### How to Reduce Costs Further

- Use partitioning — queries scan less data
- Use Parquet — more efficient storage and reads
- Use 1–2 DPU for dev and test environments
- Run on schedule, not continuously
- Archive data older than 90 days

---

## Part 13: Teardown

### Step 1: Delete Glue Job

- Glue console → **Jobs**
- Select `CustomerDataETL` → click **Delete** → confirm

### Step 2: Clear S3 Data (Optional)

- S3 console → navigate to `raw/customers/` → delete `customers.csv`
- Navigate to `processed/customers/` → delete Parquet files

### Step 3: Verify Cleanup

- [ ] Glue job deleted
- [ ] S3 data removed (if desired)
- [ ] No orphaned resources

> **Final cost check:** Go to AWS Billing → check for unexpected charges. Expected total for Lab 4.1: **under $1**.

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **ETL concepts** | What ETL is; extract, transform, load pattern; why it matters |
| **PySpark** | Writing transformation code; reading/writing S3; distributed processing |
| **Glue** | Serverless Spark; job configuration; DPU allocation; job editor |
| **Data quality** | Deduplication; standardization; validation; enrichment |
| **File formats** | CSV vs. Parquet; columnar storage; compression benefits |
| **Monitoring** | Reading CloudWatch logs; interpreting job status; debugging failures |
| **Cost optimization** | DPU sizing; Parquet efficiency; partitioning; scheduled runs |
| **Production patterns** | Idempotent jobs; schema on read; data lake zones; detailed logging |

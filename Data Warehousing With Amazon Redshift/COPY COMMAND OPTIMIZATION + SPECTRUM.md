# COPY Command Optimization + Spectrum

---

## Part 0: Understanding Data Loading & Spectrum

### The Problem: Getting Data Into Redshift

Imagine you're at Spotify with 5 million songs and 3 billion new listening events arriving in S3 every day.

| | Slow Approach | COPY Command |
|--|--------------|--------------|
| **Process** | Copy file to temp location, wait, upload, wait | Run one COPY command |
| **Time to analysis** | 6 hours | 5 minutes |
| **Result** | Today's data not available until tomorrow | Today's data available in minutes |

### What is Spectrum?

Imagine you're at Netflix with 50 TB of raw video logs in S3.

| | Load All Into Redshift | Use Spectrum |
|--|----------------------|--------------|
| **How it works** | COPY all 50 TB into cluster | Query S3 directly, on-demand |
| **Storage cost** | 50 TB x $25/TB = $1,250/month | 50 TB in S3 = $1.15/month |
| **Query cost** | Included in cluster | $6.25 per TB scanned |
| **Total monthly cost** | $1,250 | ~$7.40 |
| **Savings** | | 99% less expensive |

> **Spectrum = Save up to 99% on data warehouse costs for cold storage.**

### When to Use COPY vs. Spectrum

| Use COPY When | Use Spectrum When |
|--------------|------------------|
| Data is queried daily | Data is queried rarely |
| Performance is critical | Cost is the priority |
| Real-time analytics needed | Dataset is larger than 1 TB |
| Fast response required | Historical analysis |

**The smart approach: Use both!**

- **COPY** for hot data (today's data, queried constantly)
- **Spectrum** for cold data (last year's logs, queried occasionally)

---

## Part 1: Goals for This Lab

By the end, you will:

- Create sample data in S3 (CSV files)
- Use COPY command to load data into Redshift
- Optimize COPY with compression and parameters
- Measure performance (speed and cost)
- Create a Spectrum table to query S3 directly
- Compare COPY vs. Spectrum (speed, cost, tradeoffs)
- Understand real-world scenarios for each approach

---

## Part 2: What You'll Create

### Sample Data Files (S3)

| File | Rows | Size | Purpose |
|------|------|------|---------|
| `customers.csv` | 3,000 | 50 KB | Small, fast to test |
| `orders.csv` | 50,000 | 2 MB | Real-world size |
| `events.csv` | 1,000,000 | 150 MB | Tests performance at scale |

### Tables in Redshift (COPY)

| Table | Rows | Load Time | Purpose |
|-------|------|-----------|---------|
| `customers` | 3,000 | ~2 seconds | Real-time joins |
| `orders` | 50,000 | ~5 seconds | Transaction analytics |
| `events` | 1,000,000 | ~15 seconds | Behavioral analysis |

### Spectrum Table (External)

- `events_spectrum` — same data as events, **NOT** loaded into Redshift, queried directly from S3 on-demand

### Performance Comparison You'll Measure

| Table | Load Time | Query Time | Cost |
|-------|-----------|-----------|------|
| `customers` (COPY) | 2s | 0.5s | Included in cluster |
| `orders` (COPY) | 5s | 1s | Included in cluster |
| `events` (COPY) | 15s | 2s | Included in cluster |
| `events` (Spectrum) | 0s | 5s | $6.25 per TB scanned |

---

## Part 3: Why These Design Choices

### Why Compressed Files?

**Uncompressed CSV (150 MB):** Full 150 MB travels across the network to Redshift.

**GZIP Compressed CSV (30 MB):** Only 30 MB travels across the network, then auto-decompresses on arrival.

**Result:** 5x faster, 80% lower network cost.

### Why Multiple Small Files Instead of One Large File?

**One 150 MB file:** Only 1 node reads it at a time — poor parallelism, slower.

**15 files x 10 MB:** All 15 nodes read simultaneously — 10x speedup!

### Why Specify Delimiter and Format?

Without explicit format, Redshift has to guess whether it's comma or tab separated, how quotes are handled, etc. — leading to errors and skipped rows.

With explicit format, parsing is perfect every time. **Explicit formatting = reliability.**

---

## Part 4: Prerequisites — Check These First

**Lab 3.1 Complete:**

- Redshift cluster running (`redshift-tier3-lab`)
- Database: `analytics`
- Connection confirmed working
- Connection details saved

**Lab 1.3 Complete:**

- S3 bucket created (`data-lake-prod-[ACCOUNT_ID]`)
- `RedshiftIAMRole` attached

**Information Available:**

- Redshift endpoint
- Redshift port (5439)
- Master username (`awsadmin`)
- Master password (from Lab 3.1)
- S3 bucket name
- AWS Account ID

---

## Part 5: Understanding COPY Parameters

### Basic COPY Syntax

```sql
COPY table_name (col1, col2, col3)
FROM 's3://bucket/path/file.csv'
CREDENTIALS 'aws_iam_role=arn:...'
[OPTIONS];
```

### Key Options Explained

```sql
-- Format specification
FORMAT CSV
DELIMITER ','
QUOTE AS '"'

-- Compression
GZIP

-- Error handling
IGNOREHEADER 1        -- Skip the header row
MAXERROR 100          -- Stop if more than 100 errors
NULL AS 'NULL'        -- How nulls are represented
EMPTYASNULL           -- Treat empty strings as NULL

-- Performance
COMPUPDATE OFF        -- Skip column compression analysis
STATUPDATE OFF        -- Skip table stats update
```

### Full Real-World Example

```sql
COPY customers (customer_id, name, country, signup_date, lifetime_value)
FROM 's3://data-lake-prod-123456/raw/customers/'
CREDENTIALS 'aws_iam_role=arn:aws:iam::123456789012:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
GZIP
IGNOREHEADER 1
MAXERROR 100
COMPUPDATE ON
STATUPDATE ON;
```

---

## Part 6: Create Sample Data in S3

### Step 6.1: Create and Upload `customers.csv`

Create a file named `customers.csv` with this content:

```csv
customer_id,customer_name,country,signup_date,lifetime_value
1,Alice Johnson,USA,2024-01-01,1500.00
2,Bob Smith,Canada,2024-01-05,2300.50
3,Charlie Brown,USA,2024-01-10,890.25
4,Diana Prince,UK,2024-01-15,3400.00
5,Eve Wilson,Australia,2024-01-20,1200.75
6,Frank Miller,Germany,2024-01-25,2100.00
7,Grace Lee,Singapore,2024-02-01,1900.50
8,Henry Davis,USA,2024-02-05,3500.25
9,Ivy Chen,China,2024-02-10,1400.00
10,Jack Wilson,Canada,2024-02-15,2700.75
```

**Upload to S3:**

- Go to S3 console
- Navigate to `data-lake-prod-[ACCOUNT_ID]` → `raw/` folder
- Click **Upload** → **Add files** → select `customers.csv` → **Upload**

### Step 6.2: Create and Upload `orders.csv`

Create a file named `orders.csv` with this content:

```csv
order_id,customer_id,product,amount,order_date
1001,1,Premium Widget,99.99,2024-01-05
1002,2,Basic Widget,49.99,2024-01-06
1003,1,Extended Support,299.99,2024-01-07
1004,3,Premium Widget,99.99,2024-01-08
1005,4,Mega Widget,199.99,2024-01-09
1006,5,Basic Widget,49.99,2024-01-10
1007,2,Extended Support,299.99,2024-01-11
1008,6,Premium Widget,99.99,2024-01-12
1009,3,Mega Widget,199.99,2024-01-13
1010,7,Basic Widget,49.99,2024-01-14
1011,1,Extended Support,299.99,2024-01-15
1012,8,Premium Widget,99.99,2024-01-16
1013,4,Mega Widget,199.99,2024-01-17
1014,5,Basic Widget,49.99,2024-01-18
1015,9,Extended Support,299.99,2024-01-19
```

Upload to S3 → `raw/` folder.

### Step 6.3: Create and Upload `events.csv`

Create a file named `events.csv` with this content:

```csv
event_id,user_id,event_type,timestamp,session_id,duration_seconds
1,1,page_view,2024-02-01 10:00:00,SES001,5
2,1,click_button,2024-02-01 10:00:05,SES001,2
3,2,page_view,2024-02-01 10:00:10,SES002,8
4,1,scroll,2024-02-01 10:00:15,SES001,10
5,3,page_view,2024-02-01 10:00:20,SES003,6
6,2,click_button,2024-02-01 10:00:30,SES002,3
7,4,page_view,2024-02-01 10:00:35,SES004,4
8,1,form_submit,2024-02-01 10:00:45,SES001,15
9,5,page_view,2024-02-01 10:00:50,SES005,7
10,3,click_button,2024-02-01 10:00:55,SES003,2
```

Upload to S3 → `raw/` folder.

---

## Part 7: Load Data with COPY Command

### Step 7.1: Connect to Redshift Query Editor

- Open Redshift console
- Click **Query Editor v2** in the left sidebar
- Use the connection saved from Lab 3.1

### Step 7.2: Create Customers Table

```sql
CREATE TABLE customers (
    customer_id    INTEGER,
    customer_name  VARCHAR(100),
    country        VARCHAR(50),
    signup_date    DATE,
    lifetime_value NUMERIC(12,2)
);
```

### Step 7.3: Load Customers with COPY

> Replace `[YOUR_BUCKET_NAME]` and `[ACCOUNT_ID]` before running.

```sql
COPY customers (customer_id, customer_name, country, signup_date, lifetime_value)
FROM 's3://[YOUR_BUCKET_NAME]/raw/customers.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1
MAXERROR 100;
```

**Expected:** Rows loaded, load time, and bytes processed displayed.

### Step 7.4: Verify Customers Loaded

```sql
SELECT COUNT(*) as customer_count FROM customers;
```

```sql
SELECT * FROM customers LIMIT 5;
```

### Step 7.5: Create Orders Table

```sql
CREATE TABLE orders (
    order_id    INTEGER,
    customer_id INTEGER,
    product     VARCHAR(100),
    amount      NUMERIC(10,2),
    order_date  DATE
);
```

### Step 7.6: Load Orders with COPY

```sql
COPY orders (order_id, customer_id, product, amount, order_date)
FROM 's3://[YOUR_BUCKET_NAME]/raw/orders.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1
MAXERROR 100;
```

### Step 7.7: Create Events Table

> **Note:** The column is named `event_timestamp` — `timestamp` alone is a reserved SQL keyword.

```sql
CREATE TABLE events (
    event_id         INTEGER,
    user_id          INTEGER,
    event_type       VARCHAR(50),
    event_timestamp  TIMESTAMP,
    session_id       VARCHAR(20),
    duration_seconds INTEGER
);
```

### Step 7.8: Load Events with COPY

```sql
COPY events (event_id, user_id, event_type, event_timestamp, session_id, duration_seconds)
FROM 's3://[YOUR_BUCKET_NAME]/raw/events.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1
MAXERROR 100;
```

### Step 7.9: Verify All Data Loaded

```sql
SELECT 'customers' as table_name, COUNT(*) as row_count FROM customers
UNION ALL
SELECT 'orders',   COUNT(*) FROM orders
UNION ALL
SELECT 'events',   COUNT(*) FROM events;
```

**Expected output:**

```
table_name | row_count
customers  | 10
orders     | 15
events     | 100
```

---

## Part 8: COPY Performance Analysis

### Step 8.1: Query Performance Test

```sql
SELECT
    c.customer_name,
    COUNT(o.order_id)  as orders_count,
    SUM(o.amount)      as total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
ORDER BY total_spent DESC;
```

**Expected:** Results in under 1 second. This is why COPY is used for hot data.

### Step 8.2: Calculate Load Throughput

When your COPY commands ran, note the output and calculate:

```
Throughput = Bytes processed / Duration (seconds) = MB/second

Example: 150 MB / 10 seconds = 15 MB/second
```

This is why COPY is fast — it is optimized specifically for bulk loading.

---

## Part 9: Create Spectrum Table (External Data)

### Step 9.1: Create External Schema

```sql
CREATE EXTERNAL SCHEMA spectrum_events
FROM DATA CATALOG
DATABASE 'default'
REGION 'us-east-1';
```

This links Redshift to your S3 data lake for external queries.

### Step 9.2: Create External Table

> Replace `[YOUR_BUCKET_NAME]` before running.

```sql
CREATE EXTERNAL TABLE spectrum_events.events_external (
    event_id         INTEGER,
    user_id          INTEGER,
    event_type       VARCHAR(50),
    event_timestamp  TIMESTAMP,
    session_id       VARCHAR(20),
    duration_seconds INTEGER
)
STORED AS CSV
LOCATION 's3://[YOUR_BUCKET_NAME]/raw/events.csv'
WITH SERDEPROPERTIES (
    'field.delim'              = ',',
    'skip.header.line.count'   = '1'
);
```

> This defines the table structure but does **NOT** load data into Redshift. Data stays in S3.

### Step 9.3: Query Spectrum Table

```sql
SELECT COUNT(*) as event_count FROM spectrum_events.events_external;
```

**Expected:** Same count as your events table (same data). Notice: load time was 0 seconds, query time is 2–3 seconds, cost is only what you scan.

### Step 9.4: Compare COPY vs. Spectrum Side by Side

**Query on loaded data (COPY):**

```sql
SELECT
    event_type,
    COUNT(*)              as count,
    AVG(duration_seconds) as avg_duration
FROM events
GROUP BY event_type;
```

Note the execution time — should be **under 1 second**.

**Query on S3 data (Spectrum):**

```sql
SELECT
    event_type,
    COUNT(*)              as count,
    AVG(duration_seconds) as avg_duration
FROM spectrum_events.events_external
GROUP BY event_type;
```

Note the execution time — should be **1–3 seconds**.

**Observation:**

- **COPY:** Faster queries (data is local, in-memory analysis)
- **Spectrum:** Slower queries (S3 network latency), but no load time and far cheaper storage

---

## Part 10: Cost and Performance Comparison

| Operation | Time | Monthly Cost | Use When |
|-----------|------|-------------|----------|
| COPY (load 150 MB) | 15s | FREE | Preparing hot data |
| Query COPY data | <1s | Included in cluster | Daily dashboards, real-time |
| Query Spectrum table | 3s | $6.25 per TB scanned | Rare historical queries |
| Keep data in S3 only | N/A | $0.01/month | Archive, backup |

**Decision framework:**

| Query Frequency | Best Approach |
|----------------|--------------|
| Daily | COPY (load into Redshift) |
| Weekly | COPY or Spectrum |
| Monthly | Spectrum |
| Yearly | Spectrum |

---

## Part 11: Optimization Techniques

### Technique 1: Split Large Files for Parallelism

Instead of 1 x 150 MB file, use 15 x 10 MB files and point COPY at the folder:

```sql
COPY events (...)
FROM 's3://bucket/raw/events/'   -- directory, not a single file
CREDENTIALS '...'
FORMAT CSV;
```

Redshift loads all 15 files in parallel — up to **10x faster**.

### Technique 2: Compress Files Before Upload

```bash
# On your computer before uploading
gzip customers.csv
# Creates customers.csv.gz (5x smaller)
```

```sql
COPY customers (...)
FROM 's3://bucket/raw/customers.csv.gz'
CREDENTIALS '...'
FORMAT CSV
GZIP;
```

5x smaller files = **5x faster** network transfer.

### Technique 3: Use Parquet Format

```sql
COPY events (...)
FROM 's3://bucket/raw/events.parquet'
CREDENTIALS '...'
FORMAT AS PARQUET;
```

Parquet is columnar and typically 10x smaller than CSV — the fastest option for large datasets.

### Technique 4: Analyze and Recompress After Loading

```sql
ANALYZE COMPRESSION events;
```

Redshift analyzes your data and suggests the best compression algorithm for each column — run this after initial load to optimize storage.

---

## Part 12: Success Criteria — Verify You're Done

- [ ] `customers.csv` uploaded to S3
- [ ] `orders.csv` uploaded to S3
- [ ] `events.csv` uploaded to S3
- [ ] `customers` table created and COPY successful
- [ ] `orders` table created and COPY successful
- [ ] `events` table created and COPY successful
- [ ] All data verified (`SELECT COUNT` shows correct rows)
- [ ] Performance query runs in under 1 second
- [ ] Spectrum external schema created
- [ ] Spectrum external table created
- [ ] Spectrum query executes and returns results
- [ ] Comparison documented (COPY vs. Spectrum)
- [ ] Cost analysis understood

**If all items are checked, you have completed Lab 3.2!**

---

## Part 13: Documentation

Save this as `Lab_3_2_COPY_Spectrum_Notes.txt`:

```
=== LAB 3.2: COPY COMMAND & SPECTRUM ===
Date: [TODAY]

TABLES CREATED:

1. customers (COPY)
   Rows:       10
   Load time:  X seconds
   Query time: <1 second
   Location:   Redshift cluster storage

2. orders (COPY)
   Rows:       15
   Load time:  X seconds
   Query time: <1 second
   Location:   Redshift cluster storage

3. events (COPY)
   Rows:       100
   Load time:  X seconds
   Query time: <1 second
   Location:   Redshift cluster storage

4. events_external (SPECTRUM)
   Rows:       100 (same data)
   Load time:  0 seconds (no load — external)
   Query time: 2-3 seconds
   Location:   S3 (s3://[bucket]/raw/events.csv)

PERFORMANCE COMPARISON:
  COPY tables:
    Pros: Fast queries (<1s)
    Cons: Uses cluster storage
    Best for: Daily/hourly analysis

  Spectrum tables:
    Pros: No load time, very cheap storage
    Cons: Slower queries (2-3s)
    Best for: Rare historical queries

COST ANALYSIS:
  Cluster cost:         2 x dc2.large = $1.08/hour
  Data loaded:          150 MB (negligible)
  S3 storage:           150 MB x $0.023/GB = $0.003/month
  Spectrum query cost:  $6.25 per TB scanned
  Decision:             Use both — COPY for hot data,
                        Spectrum for cold data

COPY COMMANDS USED:
  [Paste the actual COPY commands you ran]

SPECTRUM QUERY USED:
  [Paste your external table creation SQL]

KEY LEARNINGS:
  [List what you learned]
```

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **COPY Command** | Bulk loading from S3; format and delimiter options; error handling |
| **COPY Optimization** | Compression (GZIP); file splitting for parallelism; Parquet format |
| **Performance Analysis** | Load time measurement; throughput calculation; query speed benchmarking |
| **Redshift Spectrum** | External tables on S3; no-load querying; cost vs. speed tradeoffs |
| **Real-World Patterns** | Hot vs. cold data strategy; when to use COPY vs. Spectrum; decision framework |
| **Cost Awareness** | Per-query Spectrum costs; cluster vs. S3 storage pricing; 99% cost savings scenarios |

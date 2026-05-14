# Table Design (Distkey, Sortkey, Compression)

---

## Part 0: Understanding Table Design

### The Problem: Same Query, Different Speeds

Imagine Netflix's recommendation engine running this query: "What genres does user 12345 watch?"

| | Bad Table Design | Good Table Design |
|--|-----------------|------------------|
| **Query time** | 5 seconds | 0.1 seconds |
| **Data scanned** | All 50 TB | Only 100 MB (user 12345's data) |
| **Cost per query** | $312.50 | $0.63 |
| **Difference** | | 50x faster, 500x cheaper |

This is what good table design does. It's the difference between a functioning service and a broken one.

### The Three Components of Table Design

| Component | What It Controls |
|-----------|-----------------|
| **Distkey** (distribution) | How data is split across nodes |
| **Sortkey** (sorting) | How data is ordered within nodes |
| **Compression** (storage) | How data is compressed |

Get these right: queries are 10–100x faster. Get these wrong: queries fail or take hours.

### Real-World Impact

**Company: Airbnb, dataset: 1 billion bookings**

| | Bad Table Design | Good Table Design |
|--|-----------------|------------------|
| **Reservation query** | 2 minutes | 0.5 seconds |
| **User experience** | "Why is Airbnb slow?" | Fast and responsive |
| **Revenue impact** | $1,000/minute lost | Maintained |
| **Root cause / fix** | Wrong distkey/sortkey | Fixed distkey/sortkey |

### Why Your Career Needs This

- 45,000+ job postings per year mention table optimization
- Salary premium for "Redshift optimization": **+$20,000**
- Every company with more than 1 TB of data needs this skill

---

## Part 1: Goals for This Lab

By the end, you will:

- Understand distkey and how data is distributed
- Understand sortkey and how data is sorted
- Choose optimal keys for common queries
- Apply compression to reduce storage by 80%
- Measure query performance before and after
- Understand tradeoffs between speed and update cost
- Know real-world patterns used at top companies

---

## Part 2: What is Distkey?

### The Problem It Solves

Redshift has multiple nodes (you created 2 in Lab 3.1). The question is: how does Redshift split 1 billion rows across those nodes?

**Bad way — random distribution:**

```
Node 1: Row 1, Row 500, Row 999, Row 2001, ...  (random)
Node 2: Row 2, Row 1500, Row 1001, Row 5000, ... (random)

User 12345's data is scattered across both nodes.
Redshift must scan both nodes and merge results across the network.
Result: 5 seconds.
```

**Good way — distribute by `user_id`:**

```
Node 1: All data for user_ids 1 to 500,000,000
Node 2: All data for user_ids 500,000,001 to 1,000,000,000

User 12345's data is entirely on Node 1.
Redshift scans only that node. No network traffic.
Result: 0.1 seconds.
```

### How to Choose a Distkey

Ask yourself:

- What column do I JOIN on most?
- What column do I filter on most?
- What column groups data logically?

| Company | Most Common Join/Filter | Best Distkey |
|---------|------------------------|-------------|
| Netflix (watch history) | `user_id` | `user_id` |
| Uber (rides) | `driver_id` | `driver_id` |
| Shopify (orders) | `customer_id` | `customer_id` |

### Real Impact at Scale

Netflix querying 50 TB of watch history:

| | Without Distkey | With Distkey on `user_id` |
|--|----------------|--------------------------|
| **Data scanned** | 50 TB across all nodes | 100 MB on correct node |
| **Query time** | 45 seconds | 0.5 seconds |
| **Cost per query** | $312.50 | $0.63 |

---

## Part 3: What is Sortkey?

### The Problem It Solves

After data is distributed (distkey), it still needs to be organized within each node.

**Without sortkey — data in random order:**

```
Node 1:
  user_id=500, date=Feb
  user_id=100, date=Jan
  user_id=500, date=Mar
  user_id=200, date=Jan
  ... (totally random)

Query WHERE user_id = 500 AND date = '2024-02-01'
must scan ALL data. Slow.
```

**With sortkey on `(user_id, date)` — data in order:**

```
Node 1:
  user_id=100, date=Jan
  user_id=100, date=Feb
  user_id=100, date=Mar
  user_id=200, date=Jan
  user_id=200, date=Feb
  ... (ordered)

Query jumps directly to user 500's section, then to the February date. Fast.
```

### Library Analogy

| | Without Sortkey | With Sortkey |
|--|----------------|-------------|
| **Books** | Randomly scattered | Sorted by author, then year |
| **Find "Agatha Christie 2020"** | Look at every single book (2 hours) | Jump to C section, year 2020 (2 minutes) |

### How to Choose a Sortkey

**Rule:** Use columns that appear in your `WHERE` and `ORDER BY` clauses.

```
Most common query: WHERE date >= '2024-02-01' ORDER BY date

Best sortkey:
  First:  date        (most common filter)
  Second: user_id     (next most common filter)
```

---

## Part 4: What is Compression?

### The Problem It Solves

| Table | Uncompressed | Compressed (GZIP) | Savings |
|-------|-------------|------------------|---------|
| `customers` | 1 GB | 100 MB | 90% |
| `orders` | 50 GB | 5 GB | 90% |
| `events` | 1 TB | 100 GB | 90% |
| **Storage cost** | $25/month | $2.50/month | **90%** |

### Compression Types

| Algorithm | Space Saved | Speed | Best For |
|-----------|------------|-------|----------|
| **ZSTD** (default) | 80% | Fast | Most data, strings |
| **LZO** | 70% | Fastest | Time-series, frequently accessed |
| **ZSTANDARD** | 85% | Medium | Cold data, archives |
| **RAW** (none) | 0% | Fastest reads | Small datasets, critical speed |

### Real Impact: 1 Year of Stock Market Data (500 MB/day)

| | Without Compression | With ZSTD |
|--|--------------------|-----------| 
| **Total size** | 182.5 GB | 36.5 GB |
| **Storage cost** | $4.19/month | $0.84/month |
| **Query speed** | Slower | Faster (less data to transfer) |

---

## Part 5: How All Three Work Together

```
Query: "Get user 12345's February orders"

Step 1 — Distkey finds the right node:
  DISTKEY = user_id
  Redshift knows: Node 1 has all of user 12345's data

Step 2 — Sortkey finds the right rows:
  SORTKEY = (user_id, date)
  Data is organized by user, then by date
  February data is grouped together

Step 3 — Compression reduces transfer:
  COMPRESSION = ZSTD
  1 GB of data → 200 MB compressed
  Network transfer is 5x faster

Result: 1 second query, small network transfer, happy users
```

---

## Part 6: Prerequisites

- Lab 3.1 complete — Redshift cluster running
- Lab 3.2 complete — query editor working, tables loaded
- Can connect and run queries in the query editor
- 3–4 hours uninterrupted
- Text editor open for notes

---

## Part 7: Create Optimized Tables

### Step 7.1: Analyze Current Table Structure

```sql
SELECT
    schemaname,
    tablename,
    dist_key,
    sort_keys,
    encoded
FROM pg_tables
NATURAL JOIN pg_class
NATURAL JOIN svv_table_info
WHERE schemaname = 'public'
AND tablename IN ('customers', 'orders', 'events');
```

**Expected:** Current tables show no distkey, no sortkey, no compression.

### Step 7.2: Create Optimized Customers Table

```sql
DROP TABLE IF EXISTS customers;

CREATE TABLE customers (
    customer_id    INTEGER      DISTKEY,
    customer_name  VARCHAR(100),
    country        VARCHAR(50),
    signup_date    DATE         SORTKEY,
    lifetime_value NUMERIC(12,2)
);
```

**Design choices:**

- `DISTKEY` on `customer_id` — most common filter and join column
- `SORTKEY` on `signup_date` — optimizes date range queries

### Step 7.3: Load Optimized Customers

```sql
COPY customers (customer_id, customer_name, country, signup_date, lifetime_value)
FROM 's3://[YOUR_BUCKET]/raw/customers.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1;
```

### Step 7.4: Create Optimized Orders Table

```sql
DROP TABLE IF EXISTS orders;

CREATE TABLE orders (
    order_id    INTEGER,
    customer_id INTEGER      DISTKEY,
    product     VARCHAR(100),
    amount      NUMERIC(10,2),
    order_date  DATE         SORTKEY
);
```

**Design choices:**

- `DISTKEY` on `customer_id` — matches customers table for co-located joins
- `SORTKEY` on `order_date` — optimizes date range filtering

### Step 7.5: Load Optimized Orders

```sql
COPY orders (order_id, customer_id, product, amount, order_date)
FROM 's3://[YOUR_BUCKET]/raw/orders.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1;
```

### Step 7.6: Create Optimized Events Table

```sql
DROP TABLE IF EXISTS events;

CREATE TABLE events (
    event_id         INTEGER,
    user_id          INTEGER   DISTKEY,
    event_type       VARCHAR(50),
    event_timestamp  TIMESTAMP SORTKEY,
    session_id       VARCHAR(20),
    duration_seconds INTEGER
)
DISTSTYLE KEY
COMPOUND SORTKEY (event_timestamp, event_type);
```

**Design choices:**

- `DISTKEY` on `user_id` — query by user is most common
- `COMPOUND SORTKEY` on `(event_timestamp, event_type)` — timestamp first for time-range queries, event_type second for type filtering
- `DISTSTYLE KEY` — explicitly declares distribution style

### Step 7.7: Load Optimized Events

```sql
COPY events (event_id, user_id, event_type, event_timestamp, session_id, duration_seconds)
FROM 's3://[YOUR_BUCKET]/raw/events.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1;
```

---

## Part 8: Apply Compression

### Step 8.1: Analyze for Compression Recommendations

```sql
ANALYZE COMPRESSION customers;
```

**Expected:** Redshift shows each column with a recommended compression type and estimated space savings.

Common recommendations:

- `VARCHAR` columns: `ZSTD` or `LZO`
- `INTEGER` columns: `DELTA`
- `DATE` columns: `DELTA`
- `NUMERIC` columns: `ZSTD`

### Step 8.2: Create Compressed Customers Table

```sql
DROP TABLE IF EXISTS customers;

CREATE TABLE customers (
    customer_id    INTEGER       DISTKEY  ENCODE RAW,
    customer_name  VARCHAR(100)           ENCODE ZSTD,
    country        VARCHAR(50)            ENCODE LZO,
    signup_date    DATE          SORTKEY  ENCODE DELTA,
    lifetime_value NUMERIC(12,2)          ENCODE ZSTD
);
```

**Encoding guide:**

- `RAW` — no compression (small integers)
- `ZSTD` — best compression ratio (strings and text)
- `LZO` — fastest compression (medium-length strings)
- `DELTA` — best for dates and timestamps

### Step 8.3: Reload Customers with Compression

```sql
COPY customers (customer_id, customer_name, country, signup_date, lifetime_value)
FROM 's3://[YOUR_BUCKET]/raw/customers.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1;
```

### Step 8.4: Create and Reload Compressed Orders

```sql
DROP TABLE IF EXISTS orders;

CREATE TABLE orders (
    order_id    INTEGER       ENCODE RAW,
    customer_id INTEGER DISTKEY ENCODE RAW,
    product     VARCHAR(100)  ENCODE ZSTD,
    amount      NUMERIC(10,2) ENCODE ZSTD,
    order_date  DATE  SORTKEY ENCODE DELTA
);
```

```sql
COPY orders (order_id, customer_id, product, amount, order_date)
FROM 's3://[YOUR_BUCKET]/raw/orders.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1;
```

### Step 8.5: Create and Reload Compressed Events

```sql
DROP TABLE IF EXISTS events;

CREATE TABLE events (
    event_id         INTEGER        ENCODE RAW,
    user_id          INTEGER DISTKEY ENCODE RAW,
    event_type       VARCHAR(50)    ENCODE LZO,
    event_timestamp  TIMESTAMP SORTKEY ENCODE DELTA,
    session_id       VARCHAR(20)    ENCODE LZO,
    duration_seconds INTEGER        ENCODE DELTA
)
DISTSTYLE KEY
COMPOUND SORTKEY (event_timestamp, event_type);
```

```sql
COPY events (event_id, user_id, event_type, event_timestamp, session_id, duration_seconds)
FROM 's3://[YOUR_BUCKET]/raw/events.csv'
CREDENTIALS 'aws_iam_role=arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole'
FORMAT CSV
DELIMITER ','
IGNOREHEADER 1;
```

---

## Part 9: Performance Testing

### Step 9.1: Test Single Customer Lookup

```sql
SELECT * FROM customers WHERE customer_id = 5;
```

**Expected:** Under 1 second.
**Reason:** `DISTKEY` on `customer_id` puts all of customer 5's data on one node — no cross-node communication needed.

### Step 9.2: Test Join Performance

```sql
SELECT
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) as orders_count,
    SUM(o.amount)     as total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
ORDER BY total_spent DESC;
```

**Expected:** Under 1 second.
**Reason:** Both tables share `DISTKEY` on `customer_id`, so matching rows are already on the same node — Redshift calls this a "co-located join."

### Step 9.3: Test Date Range Query

```sql
SELECT
    order_date,
    COUNT(*)    as order_count,
    SUM(amount) as daily_revenue
FROM orders
WHERE order_date >= '2024-01-15'
GROUP BY order_date
ORDER BY order_date;
```

**Expected:** Under 1 second.
**Reason:** `SORTKEY` on `order_date` lets Redshift skip directly to the relevant date range (block pruning) instead of scanning the whole table.

### Step 9.4: Test Event Aggregation

```sql
SELECT
    event_type,
    COUNT(*)              as count,
    AVG(duration_seconds) as avg_duration,
    MAX(duration_seconds) as max_duration
FROM events
WHERE event_timestamp >= '2024-02-01'
GROUP BY event_type
ORDER BY count DESC;
```

**Expected:** Under 1 second.
**Reason:** Compound sortkey puts timestamp first for fast date filtering, `event_type` second for grouping — plus compression reduces the amount of data transferred.

---

## Part 10: Understanding the Real Impact

```
=== Performance Analysis ===

Query 1: Single customer lookup
  Without distkey: ~2 seconds (scans both nodes)
  With distkey:    <1 second  (scans one node)
  Improvement:     2x faster

Query 2: Customer-orders join
  Without distkey: ~5 seconds (data redistribution across nodes)
  With distkey:    <1 second  (co-located join)
  Improvement:     5x faster

Query 3: Date range filter
  Without sortkey: ~10 seconds (full table scan)
  With sortkey:    <1 second   (block pruning)
  Improvement:     10x faster

Query 4: Multi-column aggregate
  Without compression: ~4 seconds (8 GB network transfer)
  With compression:    <1 second  (1.6 GB transfer)
  Improvement:         4x faster

Total compound improvement: 50–100x faster
```

---

## Part 11: Advanced Optimization Patterns

### Pattern 1: Slowly Changing Dimensions (SCDs)

```sql
CREATE TABLE dim_customer (
    customer_id    INTEGER  DISTKEY,
    customer_name  VARCHAR(100),
    effective_date DATE     SORTKEY,
    end_date       DATE,
    is_current     BOOLEAN
);
```

`DISTKEY` for fast lookups, `SORTKEY` on date to support type 2 SCD queries (keeping full history).

### Pattern 2: Fact Tables (High-Volume Transactions)

```sql
CREATE TABLE fact_orders (
    order_id    INTEGER,
    customer_id INTEGER      DISTKEY,
    product_id  INTEGER,
    order_date  DATE         SORTKEY,
    amount      NUMERIC(12,2),
    quantity    INTEGER
)
DISTSTYLE KEY
COMPOUND SORTKEY (order_date, product_id);
```

`DISTKEY` on `customer_id` (most common join), compound `SORTKEY` on `(date, product_id)` (most common filters).

### Pattern 3: Time-Series / Sensor Data

```sql
CREATE TABLE sensor_data (
    sensor_id        INTEGER     DISTKEY,
    measurement_time TIMESTAMP   SORTKEY,
    temperature      DECIMAL(5,2),
    humidity         DECIMAL(5,2),
    pressure         DECIMAL(7,2)
)
DISTSTYLE KEY;
```

`DISTKEY` on `sensor_id` (query by sensor), `SORTKEY` on timestamp (time-range queries).

---

## Part 12: Troubleshooting Table Design

### Problem: Slow Join

**Symptom:** Join query takes 5+ seconds.

```sql
-- Check distkeys of both tables
SELECT tablename, distkey
FROM pg_tables
WHERE tablename IN ('table1', 'table2');
```

If the distkeys are different, Redshift has to shuffle data across nodes before joining. Fix: use the same distkey (the join column) on both tables.

```sql
-- Bad: different distkeys
CREATE TABLE customers (customer_id INTEGER DISTKEY);
CREATE TABLE orders    (order_id    INTEGER DISTKEY);

-- Good: matching distkeys
CREATE TABLE customers (customer_id INTEGER DISTKEY);
CREATE TABLE orders    (customer_id INTEGER DISTKEY);
```

### Problem: Slow Filter Queries

**Symptom:** `WHERE date >= '2024-01-01'` takes 10+ seconds.

```sql
-- Check if table has a sortkey on the filtered column
SELECT tablename, sortkey1, sortkey2, sortkey3
FROM pg_tables
WHERE tablename = 'mytable';
```

If no sortkey exists on the filtered column, Redshift does a full scan. Fix: recreate the table with a sortkey on that column.

### Problem: High Storage Cost

**Symptom:** Data is larger than expected after compression.

```sql
ANALYZE COMPRESSION mytable;
```

If columns show `RAW`, they are not compressing well. Try `ZSTD` instead of `LZO`. Note: some data (like random strings or UUIDs) simply doesn't compress well — that's okay, use what works.

---

## Part 13: Success Criteria — Verify You're Done

- [ ] Optimized `customers` table created with distkey and sortkey
- [ ] Optimized `orders` table created with distkey and sortkey
- [ ] Optimized `events` table created with compound sortkey
- [ ] All tables loaded successfully
- [ ] All tables have compression encoding applied
- [ ] Single customer lookup runs in under 1 second
- [ ] Join query runs in under 1 second
- [ ] Date range query runs in under 1 second
- [ ] Event aggregation query runs in under 1 second
- [ ] Performance documentation created
- [ ] Can explain why each design choice was made
- [ ] Can explain distkey, sortkey, and compression in your own words

**If all items are checked, you have completed Lab 3.3!**

---

## Part 14: Documentation

Save this as `Lab_3_3_Table_Design_Notes.txt`:

```
=== Lab 3.3: Table Design Optimization ===
Date: [TODAY]

TABLE DESIGN CHOICES:

1. customers table
   Distkey:     customer_id
     Why:       Filter and join on customer_id most often
     Impact:    Fast customer lookups
   Sortkey:     signup_date
     Why:       Date range queries are common
     Impact:    Efficient date-range scans
   Compression: ZSTD on VARCHAR, DELTA on DATE
     Why:       Best compression for text and dates
     Impact:    ~80% storage reduction

2. orders table
   Distkey:     customer_id
     Why:       Joins with customers table on customer_id
     Impact:    Co-located joins, no data redistribution
   Sortkey:     order_date
     Why:       Queries often filter by date range
     Impact:    Block pruning, faster scans
   Compression: DELTA on dates, ZSTD on amounts
     Why:       Dates and amounts compress very well
     Impact:    ~70% storage reduction

3. events table
   Distkey:     user_id
     Why:       High-volume table, queried by user
     Impact:    User data grouped on single node
   Sortkey:     COMPOUND (event_timestamp, event_type)
     Why:       Time-range queries primary, type grouping secondary
     Impact:    Both time and type queries are fast
   Compression: DELTA on timestamps, LZO on strings
     Why:       Timestamps have patterns, strings are medium length
     Impact:    ~75% storage reduction

QUERY PERFORMANCE RESULTS:
  Customer lookup:  <1 second (distkey — single node scan)
  Join query:       <1 second (co-located join)
  Date range query: <1 second (sortkey block pruning)
  Event aggregate:  <1 second (compound sortkey + compression)

KEY LEARNINGS:
  [Your reflections here]
```

---

## Teardown — Tier 3 Complete, Stop Charges Now

### Step 14.1: Confirm You're Done

- Completed Labs 3.1, 3.2, and 3.3
- All notes and documentation saved

### Step 14.2: Delete Redshift Cluster

- Go to **Redshift console**
- Select `redshift-tier3-lab`
- Click **Delete**
- Check "I want to delete this cluster"
- Uncheck "Create final snapshot" (unless you want a backup)
- Click **Delete cluster** — takes 3–5 minutes

### Step 14.3: Clean Up Related Resources

- Redshift console → **Cluster subnet groups** → delete `redshift-tier3-subnet-group`
- VPC console → **Security groups** → delete `redshift-security-group` (keep if reusing for other labs)

### Step 14.4: Verify Costs Have Stopped

- Go to **AWS Billing** → **Costs by service**
- Redshift should show $0 going forward
- Total Tier 3 cost: ~$25–35 depending on how long labs ran

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **Distkey** | How data distributes across nodes; how to choose the right column; co-located joins |
| **Sortkey** | How data orders within nodes; block pruning; compound sortkeys |
| **Compression** | Algorithms (ZSTD, LZO, DELTA, RAW); when to use each; 80% storage savings |
| **Performance analysis** | Query timing; identifying bottlenecks; before/after comparison |
| **Real-world patterns** | Fact tables, dimension tables, time-series design |
| **Troubleshooting** | Diagnosing slow joins, slow filters, and high storage costs |

---

## How Top Companies Design Tables

**Interview question:** "Walk us through designing a table for e-commerce order analytics."

**Your answer (from this lab):** "I'd use a distkey on `customer_id` since that's the most common join column, a sortkey on `order_date` for time-based analysis, and compression with `DELTA` on dates and `ZSTD` on strings. This gives fast customer-based queries, efficient date filtering, and roughly 75% storage savings."

| Company | Table | Distkey | Sortkey |
|---------|-------|---------|---------|
| Amazon (internal) | Most tables | Primary query column | Timestamp |
| Facebook | Event tables | `user_id` | Timestamp |
| Facebook | Fact tables | Fact key | Date |
| Airbnb | Bookings | `listing_id` | `booking_date` |
| Airbnb | Hosts | `host_id` | `signup_date` |

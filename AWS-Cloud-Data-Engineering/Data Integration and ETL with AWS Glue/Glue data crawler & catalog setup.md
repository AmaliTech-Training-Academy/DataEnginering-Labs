# Glue Data Crawler & Catalog Setup

---

## Part 0: Understanding Why Data Cataloging Matters

### The Big Picture Story

Imagine you're a data analyst at a mid-size company with 500 data sources.

**Without a data catalog — Monday morning:**

- You need sales data for a report
- You search Slack: "Does anyone know where sales data lives?"
- 10 minutes of discussion, someone says "It's on S3, I think... maybe?"
- You go to S3 and see 500 buckets with no context
- You spend 2 hours finding the right data and still aren't sure it's complete

**Cost to company:** 2.5 hours x $75/hour = **$187.50 wasted** — per analyst, per incident.

**With Glue data catalog — Monday morning:**

- You open Glue data catalog and search "sales"
- You find "sales_data_2024" with all columns, data types, and update history
- You see it was last updated 2 hours ago
- You click "Query with Athena" and have your data in 1 minute

**Scale this across 100 analysts:**

| | Without Catalog | With Catalog |
|--|-----------------|--------------|
| **Time spent searching per week** | 200 hours | 8 hours |
| **Weekly savings** | | 192 hours |
| **Annual savings** | | ~$730,000 |

### What is Glue Data Catalog? (In Plain English)

**Glue data catalog = a searchable library for all your company's data.**

| | Without Catalog | With Catalog |
|--|-----------------|--------------|
| **Finding data** | Search Slack, browse S3, guess | Search bar — type "customer", see all 15 customer tables |
| **Schema info** | Download file and inspect manually | Columns, types, owner, lineage — all visible instantly |
| **Currency** | Often outdated | Auto-updated by crawlers |

### Glue Crawlers: Automation Magic

**Manual approach (old way):**

- Someone copies schema from a database
- Types it into a spreadsheet
- Updates it whenever data changes
- Spreadsheet gets lost
- Nobody knows the current state

**Glue crawler approach (modern way):**

- Set a schedule: "Run every night at 2 AM"
- Crawler automatically reads S3, detects schema, and updates the catalog
- Next morning: catalog is current, analysts find what they need immediately

### Why This Matters for Your Career

- Every company uses data catalogs — understanding them makes you valuable at any data team
- Data catalog engineers earn **$120k–$150k**
- GDPR compliance: "Show all tables with personal data" — with a catalog, this takes seconds instead of days
- Career path: Junior ("Can you find the sales data?") → Senior ("Can you design a data catalog for our company?")

---

## Part 1: Goals for This Lab

By the end of this lab, you will:

- Understand how data catalogs work
- Create a Glue crawler that auto-discovers schemas
- Query cataloged data with Athena
- Manage catalog metadata and updates
- Explain why automation beats manual documentation

---

## Part 2: What You'll Create

### Glue Data Catalog Structure

```
AWS Glue data catalog
│
├── DATABASE: raw_data
│   ├── TABLE: test_customers (created by crawler)
│   │     Columns: customer_id, name, email, country,
│   │              created_date, purchase_amount
│   │     Location: s3://data-lake-prod-xxx/raw/
│   │     Format: CSV | Last updated: auto
│   │
│   └── TABLE: test_orders (created by crawler)
│         Columns: order_id, customer_id, order_date,
│                  total_amount, status
│
├── DATABASE: processed_data
│   └── (future tables from transformations)
│
└── CRAWLER: raw_data_crawler
      Schedule:  Daily at 02:00 UTC
      Source:    s3://data-lake-prod-xxx/raw/
      Target:    raw_data database
      Action:    Auto-update tables when schema changes
```

### How It Works

| | Old Way | New Way |
|--|---------|---------|
| **Flow** | Data arrives → engineer manually documents → catalog goes stale | Data arrives → crawler runs → catalog auto-updates |
| **Result** | Always outdated | Always current, zero manual work |

---

## Part 3: The "Why" Behind Design Decisions

### Why Crawlers Over Manual Documentation?

| Step | Manual Documentation | Crawler Automation |
|------|---------------------|--------------------|
| **Initial setup** | Engineer documents schema: 30 minutes | Set crawler schedule: 5 minutes (one-time) |
| **When data changes** | Nobody updates docs, schema goes stale | Crawler detects change, catalog auto-updates |
| **Analyst experience** | Outdated schema causes query failures | Always correct, queries work first time |
| **Time cost** | Hours of debugging per week | Zero ongoing effort |

### Why Crawlers Over Manual Search?

**Without crawler — finding data:**

- Search Slack history: 10 minutes
- Browse S3 manually: 20 minutes
- Download a sample: 10 minutes
- Inspect columns: 10 minutes
- Realize it's the wrong table, start over
- **Total: 2+ hours**

**With crawler — finding data:**

- Open Glue catalog: 30 seconds
- Search "customer" — see all 15 customer tables: 2 minutes
- Read descriptions, pick the right one: 1 minute
- **Total: 3.5 minutes**

### Why the Cost Is So Low

| Item | Cost |
|------|------|
| 1 DPU (data processing unit) | $0.44/hour |
| Typical crawl (2–5 minutes) | $0.015–$0.037 |
| Daily crawl for a month | ~$13/month |
| Time saved per analyst per month | $1,000+ |

Crawlers pay for themselves within a single day.

---

## Part 4: Prerequisites — Check These First

- Completed Labs 1.1, 1.2, 1.3 (IAM, VPC, S3)
- S3 bucket exists: `data-lake-prod-123456789`
- `test_customers.csv` is in the `raw/` folder (from Lab 1.3)
- IAM roles from Lab 1.1 (`DataEngineerRole` with Glue access)
- 3 hours of uninterrupted time
- Text editor open for notes

---

## Step 0: Login and Prepare

### Step 0.1: Login to AWS

- Open [https://console.aws.amazon.com](https://console.aws.amazon.com)
- Enter credentials and MFA code if prompted
- Confirm account name shows in the top right

### Step 0.2: Verify Region

- Top right, confirm region is `us-east-1`
- If different, switch to `us-east-1`

### Step 0.3: Verify S3 Data Exists

- Go to S3 console
- Open `data-lake-prod-123456789`
- Navigate to the `raw/` folder
- Confirm `test_customers.csv` is present
- If missing, upload it now

### Step 0.4: Prepare Documentation File

Create `Lab_2_1_Glue_Crawler.txt` with this template:

```
=== Lab 2.1: Glue data crawler & catalog setup ===
Date Created: [TODAY'S DATE]
AWS Account ID: [YOUR ACCOUNT ID]

DATABASES:
  Raw database name:       [TO BE FILLED]
  Processed database name: [TO BE FILLED]

CRAWLERS:
  Raw data crawler name: [TO BE FILLED]
  Crawler IAM role:      [TO BE FILLED]

TABLES (auto-discovered):
  Table 1 name:    [TO BE FILLED]
  Table 1 columns: [TO BE FILLED]
```

---

## Part 1: Create Glue Databases

A Glue database is a container for related tables — similar to a folder that groups data by purpose.

```
Glue data catalog
├── raw_data (database)        ← raw files from source systems
│   ├── Table: test_customers
│   └── Table: test_orders
│
└── processed_data (database)  ← cleaned, transformed data
    └── (populated in future labs)
```

### Step 1.1: Navigate to Glue Console

- Click **Services**, search `glue`, click **AWS Glue**
- You should see a left sidebar with **Crawlers**, **Databases**, and **Tables**

### Steps 1.2 & 1.3: Create Raw Data Database

- Click **Databases** in the left sidebar
- Click **Create database**
- Fill in:

| Field | Value |
|-------|-------|
| Database name | `raw_data` |
| Description | Catalog for raw data ingested from source systems. Auto-discovered by crawlers. |

- Click **Create database**

**Save:** Raw database name: `raw_data`

### Step 1.4: Create Processed Data Database

- Click **Create database** again
- Fill in:

| Field | Value |
|-------|-------|
| Database name | `processed_data` |
| Description | Catalog for cleaned, validated data. Created from raw data transformations. |

- Click **Create database**

Both databases should now appear in the list.

---

## Part 2: Create Glue Crawler

A crawler is a robot that reads your S3 data, detects column names and data types, and writes that schema automatically into the Glue catalog.

```
Crawler workflow:
1. Reads files at s3://bucket/raw/
2. Detects columns and data types
3. Creates table definitions
4. Stores everything in Glue catalog
5. Done — analysts can query immediately
```

### Step 2.1: Navigate to Crawlers

- Click **Crawlers** in the left sidebar
- Click **Create crawler**

### Step 2.2: Enter Crawler Details

| Field | Value |
|-------|-------|
| Crawler name | `raw_data_crawler` |
| Crawler source type | Data stores (keep default) |

Click **Next**.

### Step 2.3: Configure Data Store

| Field | Value |
|-------|-------|
| Connection | Leave empty (S3 doesn't need one) |
| Include path | `s3://data-lake-prod-123456789/raw/` (replace with your account ID) |
| Exclude patterns | Leave empty |

Click **Next**.

### Step 2.4: Create and Assign IAM Role

| Field | Value |
|-------|-------|
| IAM role | Create a new role |
| Role name | `AWSGlueServiceRole-raw-data` |

Click **Next**.

### Step 2.5: Configure Output

| Field | Value |
|-------|-------|
| Glue database | `raw_data` |
| Prefix | Leave empty |
| Table creation | Create a single schema for each S3 path |

Click **Next**.

### Step 2.6: Configure Schedule

| Field | Value |
|-------|-------|
| Frequency | Daily |
| Time | 02:00 UTC |
| Timezone | UTC |

Click **Next**.

### Step 2.7: Review and Create

Confirm:
- Crawler name: `raw_data_crawler`
- Data store path: `s3://bucket/raw/`
- Database: `raw_data`
- Schedule: Daily at 02:00 UTC

Click **Create crawler**.

**Save:** Crawler name, IAM role name, schedule

---

## Part 3: Run Crawler for the First Time

### Step 3.1: Run Crawler Manually

- Click on `raw_data_crawler` in the crawlers list
- Click **Run crawler** (top right)

**Expected:** Status shows `Starting`, then `Ready` after 1–2 minutes.

### Step 3.2: Check Crawler Results

- Click **Databases** in the left sidebar
- Click **raw_data**
- Click the **Tables** tab

You should see a table named `test_customers` (auto-generated). Click on it to confirm:

| Column | Type |
|--------|------|
| `customer_id` | string |
| `name` | string |
| `email` | string |
| `country` | string |
| `created_date` | string |
| `purchase_amount` | double |

**Save:** Table name, location, format, discovered columns

---

## Part 4: Query Data with Athena

Athena = a SQL query tool that works directly with the Glue catalog. No downloads, no local database setup needed.

| | Traditional Workflow | Athena Workflow |
|--|---------------------|----------------|
| **Steps** | Download CSV → load into DB or Excel → run queries → export | Write SQL → get results instantly |
| **Setup needed** | Yes | None |
| **Cost** | Time + infrastructure | $5 per TB scanned |

### Step 4.1: Open Athena

**Option A:** From the Glue table, click **View data** or **Query with Athena**.

**Option B:** Click Services, search `athena`, click **Athena**.

### Step 4.2: Run Your First Query

```sql
SELECT
    customer_id,
    name,
    email,
    country,
    purchase_amount
FROM raw_data.test_customers
LIMIT 10;
```

Click **Run**. **Expected:** Results appear in 1–3 seconds.

### Step 4.3: Try More Queries

**Aggregate by country:**

```sql
SELECT
    country,
    COUNT(*)              as customer_count,
    AVG(purchase_amount)  as avg_purchase
FROM raw_data.test_customers
GROUP BY country
ORDER BY customer_count DESC;
```

**Find top spenders:**

```sql
SELECT
    name,
    country,
    purchase_amount
FROM raw_data.test_customers
WHERE purchase_amount > 100
ORDER BY purchase_amount DESC;
```

**Save:** Total customers, countries found, average purchase, top spender

---

## Part 5: Create Additional Sample Data

### Step 5.1: Create `test_orders.csv`

Create a file named `test_orders.csv` with this content:

```csv
order_id,customer_id,order_date,total_amount,status
1001,1,2024-01-15,299.99,completed
1002,2,2024-01-16,150.00,completed
1003,3,2024-01-17,500.00,pending
1004,1,2024-01-18,75.50,completed
1005,4,2024-01-19,1200.00,completed
```

### Step 5.2: Upload to S3

- Go to S3 console
- Navigate to `data-lake-prod-123456789/raw/`
- Click **Upload** → **Add files** → select `test_orders.csv` → **Upload**

### Step 5.3: Run Crawler Again

- Go to Glue console → **Crawlers**
- Select `raw_data_crawler` → click **Run crawler**
- Wait 1–2 minutes

**Expected:** Crawler finishes and a new `test_orders` table appears in the `raw_data` database.

### Step 5.4: Query the New Table

```sql
SELECT
    order_id,
    customer_id,
    order_date,
    total_amount,
    status
FROM raw_data.test_orders
ORDER BY order_date DESC
LIMIT 5;
```

---

## Part 6: Verify Glue Catalog Setup

**Databases:**
- [ ] `raw_data` database created
- [ ] `processed_data` database created

**Crawler:**
- [ ] `raw_data_crawler` created and configured
- [ ] Scheduled for daily 02:00 UTC
- [ ] Last run completed successfully

**Tables (auto-discovered):**
- [ ] `test_customers` table exists with 6 columns
- [ ] `test_orders` table exists with 5 columns

**Athena integration:**
- [ ] Can query `test_customers` — results correct
- [ ] Can query `test_orders` — results correct

**Metadata quality:**
- [ ] Column names correct
- [ ] Data types detected correctly
- [ ] S3 location shown
- [ ] Update timestamps present

---

## Part 7: Document Your Catalog

Save this in `Lab_2_1_Glue_Crawler.txt`:

```
=== Lab 2.1: Glue data crawler & catalog complete ===

DATABASES CREATED:
  1. raw_data
       Purpose: Raw data from source systems
       Tables discovered: 2

  2. processed_data
       Purpose: Cleaned/validated data
       Tables: (will be populated in future labs)

CRAWLERS CONFIGURED:
  1. raw_data_crawler
       Source:    s3://data-lake-prod-123456789/raw/
       Target:    raw_data database
       Schedule:  Daily at 02:00 UTC
       Last run:  [timestamp]
       Tables created: 2

TABLES IN CATALOG:
  1. test_customers
       Location: s3://bucket/raw/
       Format:   CSV
       Columns:  6 (customer_id, name, email, country,
                    created_date, purchase_amount)
       Records:  5 rows
       Last updated: [crawler timestamp]

  2. test_orders
       Location: s3://bucket/raw/
       Format:   CSV
       Columns:  5 (order_id, customer_id, order_date,
                    total_amount, status)
       Records:  5 rows
       Last updated: [crawler timestamp]

ATHENA QUERIES TESTED:
  SELECT from test_customers:  5 rows returned
  GROUP BY country:            4 countries found
  SELECT from test_orders:     5 rows returned
  Query performance:           <3 seconds per query

COSTS:
  Crawler DPU cost:  ~$0.50 per run
  Daily runs:        30 x $0.50 = $15/month
  Athena queries:    $5 per TB scanned
  Test queries:      <$0.01
  Total:             ~$15–20/month

BENEFITS REALIZED:
  Manual schema documentation:  no longer needed
  Schema updates:               automatic nightly
  Data discovery time:          reduced from 2 hours to 5 minutes
  Compliance:                   audit trail built in
```

---

## Success Criteria — Verify You're Done

- [ ] `raw_data` database created
- [ ] `processed_data` database created
- [ ] `raw_data_crawler` created and configured
- [ ] Crawler set to daily schedule at 02:00 UTC
- [ ] Crawler executed successfully (first run)
- [ ] `test_customers` table auto-discovered with 6 columns
- [ ] `test_orders` table auto-discovered with 5 columns
- [ ] Can query `test_customers` with Athena
- [ ] Can query `test_orders` with Athena
- [ ] Query results are correct
- [ ] All metadata in Glue catalog is accurate
- [ ] Documentation file saved with all details

**If all items are checked, you have completed Lab 2.1!**

---

## Teardown — Keep or Delete?

Glue is inexpensive — unlike NAT Gateway, you can keep it running comfortably.

**If you want to keep it (recommended):**

- Keep all Glue databases (free storage)
- Keep the crawler (only costs when it runs, ~$15/month)
- Keep S3 data (tiny test files)
- **Monthly cost:** ~$15

**If you want to delete everything:**

- Delete test CSV files from S3
- Delete the crawler from Glue
- Delete databases from Glue (optional — they're free)
- **Monthly cost:** $0

> **Recommendation:** Keep the crawler. It's cheap and useful.

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **Data catalogs** | Central registry concept; metadata management; search and discovery |
| **Glue crawlers** | Automated schema detection; scheduled execution; zero manual documentation |
| **Glue data catalog** | How metadata is stored; navigating and finding tables; Athena integration |
| **Athena query engine** | SQL queries on S3 data; results without local downloads; cost-effective querying |
| **Data governance** | Automation over manual processes; schema change tracking; compliance and audit trails |

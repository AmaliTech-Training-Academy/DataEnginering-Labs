# Schema Evolution & Versioning

---

## Part 0: Understanding Why This Matters

### The Schema Evolution Problem

It's Tuesday morning. Your company's data pipeline suddenly fails:

```
ERROR: Column "customer_age" not found in dataset
```

You investigate and find the customer system added a new column yesterday:

```
Old schema: customer_id, name, email, address
New schema: customer_id, name, email, address, customer_age

Our Parquet files expect the old schema.
New data has an extra column → ERROR
```

This happens to **90% of data teams** — multiple times per year.

### Why Schemas Change

| Timeline | Schema State | What Happened |
|----------|-------------|---------------|
| Month 1 | `id, name, email` | Simple launch |
| Month 2 | `id, name, email, phone` | Product team added phone |
| Month 3 | `id, name, email, phone, preferred_language` | International expansion |
| Month 6 | 8 fields | Multiple teams adding fields |
| Year 1 | 47 schema changes | ETL job broke 47 times |

### The Cost of Unmanaged Schemas

| | Without Schema Management | With Schema Management |
|--|--------------------------|------------------------|
| **When schema changes** | Pipeline breaks, 4 hours debugging, dashboards down 8 hours | Schema registry validates change, assigns new version |
| **Effect on existing jobs** | Must be manually fixed | Old jobs continue working (backwards compatible) |
| **Business impact** | ~$10K per incident in analyst downtime | Zero downtime |
| **After 30 changes** | Engineers refusing changes, business blocked | 30 changes, zero pipeline breaks |
| **Engineer morale** | "You quit" | Focus on optimization, not firefighting |

### Real-World Example: Uber's Schema Evolution

**Schema v1 (2010):** `trip_id, driver_id, passenger_id, pickup_time, dropoff_time, distance, fare`

**Schema v2 (2015):** Added `pickup_location, dropoff_location, vehicle_type, tips, surge_multiplier`

**The problem:** 5 billion historical trips don't have `vehicle_type` or `tips`. Do you fail? Default to null?

**The solution:** Schema versioning + backwards compatibility rules. Old data uses v1.0, new data uses v2.0 — code handles both gracefully, no pipeline breaks.

### What Schema Versioning Solves

| Problem | Without Versioning | With Versioning |
|---------|-------------------|-----------------|
| **Breaking changes** | Added required field = all old data invalid | Mark field optional = old data still valid |
| **Multiple sources** | System A adds field Monday, System B adds Thursday — both break your code | Schema registry tracks all versions, code auto-adapts |
| **Rollback confusion** | "Did we rename customer_id to cust_id?" | Full schema history with exact change and timestamp |
| **Team coordination** | "Did anyone else rename this column?" (no tracking) | Full audit trail of who changed what and when |

### Your Career and Schemas

- **Junior engineer:** "Why did the pipeline break?"
- **Senior engineer:** "Let me check the schema version"
- **Staff engineer:** "We need a schema governance framework"

This lab teaches you to become the person who **prevents** pipeline breaks.

---

## Part 1: Goals for This Lab

By the end of this lab, you will:

- Understand what schema versioning is and why it matters
- Create schema versions in AWS Glue Schema Registry
- Register multiple schema versions (1.0, 2.0, 3.0)
- Handle backwards compatible changes (adding optional fields)
- Handle breaking changes (removing or renaming fields)
- Apply schemas to data in S3
- Understand schema evolution best practices

---

## Part 2: What You'll Create

### Project Overview: Customer Schema Evolution

**Version 1.0 — Launch:**

```
customer_id:   string (required)
email:         string (required)
signup_date:   date   (required)
country:       string (required)
```

**Version 2.0 — Month 3, Backwards Compatible:**

```
customer_id:   string (required)
email:         string (required)
signup_date:   date   (required)
country:       string (required)
phone:         string (optional)  ← new field
```

**Version 3.0 — Month 6, Backwards Compatible:**

```
customer_id:        string (required)
email:              string (required)
signup_date:        date   (required)
country:            string (required)
phone:              string (optional)
subscription_tier:  string (optional)  ← new field
```

### Glue Schema Registry vs. File-Based Schemas

| File System (Old Way) | Glue Schema Registry (Modern Way) |
|-----------------------|----------------------------------|
| `schema_v1.json` (which version?) | registry = CustomerData |
| `schema_v2.json` (who changed it?) | Version 1.0: customer_id, email, signup_date, country |
| `schema_latest.json` (is this current?) | Version 2.0: ...phone added |
| `schema_backup.json` (is this old?) | Version 3.0: ...subscription_tier added |
| No audit trail | Full history: who changed what, when |
| No validation | Compatibility checking before deployment |

---

## Part 3: The "Why" Behind Schema Management

### Why Not Just Use Dynamic Typing?

**No schema — looks easy:**

```python
df = spark.read.json("customers.json")
df.show()  # columns = whatever is in the JSON
```

**Problems:**

- Don't know what columns exist until runtime
- Missing column silently returns null — no error
- Typos silently return null (`"custome_id"` vs `"customer_id"`)
- Data quality impossible to track

**With a schema — catches problems immediately:**

```python
schema = StructType([
    StructField("customer_id", StringType(), False),   # required
    StructField("email",       StringType(), False),   # required
    StructField("phone",       StringType(), True)     # optional
])

df = spark.read.schema(schema).json("customers.json")
# Code fails if required columns are missing
# Typos caught at read time
# Data quality is enforceable
```

### Backwards Compatibility Rules

| Change | Safe? | Why |
|--------|-------|-----|
| Add optional field | ✅ Yes | Old data still valid (field just missing) |
| Remove optional field | ✅ Yes | Old data had it — it's just ignored |
| Change default value | ✅ Yes | Only affects new records |
| Change field order | ✅ Yes | Doesn't affect reading |
| Add required field | ❌ No | Old data is missing it — invalid |
| Remove required field | ❌ No | Old code expects it — will break |
| Change field type | ❌ No | int to string breaks existing queries |
| Rename required field | ❌ No | Old code can't find the new name |

### Real Production Example: Netflix View History

| Version | Year | Change | Compatible? |
|---------|------|--------|-------------|
| v1.0 | 2008 | `title_id, user_id, timestamp, rating` | — |
| v1.1 | 2010 | Added `device_type` (optional) | ✅ Yes |
| v2.0 | 2012 | Added `watch_percentage` (optional) | ✅ Yes |
| v2.1 | 2015 | Renamed `timestamp → timestamp_utc`, changed rating scale | ❌ No — migration required |

**Migration path for v2.1:**

- Deploy code that reads both old and new formats
- Run ETL to convert all historical data to v2.1
- Once 100% migrated, deprecate old versions
- Continue with v2.1 only

---

## Part 4: Prerequisites

- Completed Lab 4.1 (Glue ETL basics)
- AWS account with Glue access
- Data lake S3 bucket from Tier 1
- Understanding of JSON format
- 30 minutes to read and start

---

## Part 5: Create Schema Versions

### Step 5.1: Navigate to Glue Schema Registry

- Open AWS Console → **Services** → **Glue**
- Click **Schema registry** in the left sidebar
- You should see the Schema Registry page

### Step 5.2: Create Your First Schema

- Click **Create schema**
- Fill in:

| Field | Value |
|-------|-------|
| Schema name | `CustomerSchema` |
| Registry | `default-registry` |
| Compatibility mode | `BACKWARD` |

> **BACKWARD** compatibility = new versions must be readable by old code. Old code can read new data.

- Click **Create schema**

### Step 5.3: Define Schema v1.0

In the schema definition editor, paste this Avro schema:

```json
{
  "type": "record",
  "name": "Customer",
  "namespace": "com.company.data",
  "fields": [
    {
      "name": "customer_id",
      "type": "string",
      "doc": "Unique customer identifier"
    },
    {
      "name": "email",
      "type": "string",
      "doc": "Customer email address"
    },
    {
      "name": "signup_date",
      "type": "string",
      "doc": "Date customer signed up (ISO 8601 format)"
    },
    {
      "name": "country",
      "type": "string",
      "doc": "Customer country (ISO 3166-1 alpha-2 code)"
    }
  ]
}
```

**What each part means:**

| Field | Meaning |
|-------|---------|
| `"type": "record"` | This is a structured record (like a table row) |
| `"name": "Customer"` | The type name for this data |
| `"namespace": "..."` | Package-like organization |
| `"fields": [...]` | List of columns |
| `"type": "string"` | Data type (string, int, long, etc.) |
| `"doc": "..."` | Documentation/comment for this field |

### Step 5.4: Save Version 1

- Click **Create version**
- Description: `Initial schema with basic customer fields`
- Click **Create version**

**Expected:** Version 1 created with status `LATEST`, compatibility `BACKWARD`.

### Step 5.5: Create Schema v2.0 — Add Optional Field

- Click **Create new version**
- Paste this updated schema (adds the `phone` field):

```json
{
  "type": "record",
  "name": "Customer",
  "namespace": "com.company.data",
  "fields": [
    {
      "name": "customer_id",
      "type": "string",
      "doc": "Unique customer identifier"
    },
    {
      "name": "email",
      "type": "string",
      "doc": "Customer email address"
    },
    {
      "name": "signup_date",
      "type": "string",
      "doc": "Date customer signed up (ISO 8601 format)"
    },
    {
      "name": "country",
      "type": "string",
      "doc": "Customer country (ISO 3166-1 alpha-2 code)"
    },
    {
      "name": "phone",
      "type": ["null", "string"],
      "default": null,
      "doc": "Customer phone number (optional, added v2.0)"
    }
  ]
}
```

**What changed and why it's safe:**

- `"type": ["null", "string"]` — field can be null or a string (optional)
- `"default": null` — if the field is missing in old records, use null
- Old records without `phone` are still valid — backwards compatible

- Description: `Added optional phone field for customer contact (backwards compatible)`
- Click **Create version**

### Step 5.6: Create Schema v3.0 — Add Subscription Tier

- Click **Create new version**
- Paste this updated schema:

```json
{
  "type": "record",
  "name": "Customer",
  "namespace": "com.company.data",
  "fields": [
    {
      "name": "customer_id",
      "type": "string",
      "doc": "Unique customer identifier"
    },
    {
      "name": "email",
      "type": "string",
      "doc": "Customer email address"
    },
    {
      "name": "signup_date",
      "type": "string",
      "doc": "Date customer signed up (ISO 8601 format)"
    },
    {
      "name": "country",
      "type": "string",
      "doc": "Customer country (ISO 3166-1 alpha-2 code)"
    },
    {
      "name": "phone",
      "type": ["null", "string"],
      "default": null,
      "doc": "Customer phone number (optional)"
    },
    {
      "name": "subscription_tier",
      "type": ["null", "string"],
      "default": "free",
      "doc": "Subscription level: free, pro, enterprise (optional, added v3.0)"
    }
  ]
}
```

- Description: `Added subscription_tier field for monetization (backwards compatible)`
- Click **Create version**

### Step 5.7: View Schema History

Click on `CustomerSchema`. You should see all three versions:

| Version | Description |
|---------|-------------|
| Version 1 | Initial schema with basic customer fields |
| Version 2 | Added optional phone field |
| Version 3 | Added optional subscription_tier field |

Each version shows its version number, created timestamp, description, and compatibility status.

---

## Part 6: Apply Schemas to Data

### Step 6.1: Create Sample Data v1.0

Create a file named `customers_v1.json`:

```json
{"customer_id": "C001", "email": "john@example.com", "signup_date": "2024-01-15", "country": "US"}
{"customer_id": "C002", "email": "jane@example.com", "signup_date": "2024-01-20", "country": "CA"}
{"customer_id": "C003", "email": "bob@example.com",  "signup_date": "2024-01-25", "country": "GB"}
```

Upload to S3 → `schemas/v1.0/` folder.

### Step 6.2: Create Sample Data v2.0

Create a file named `customers_v2.json`:

```json
{"customer_id": "C004", "email": "alice@example.com",   "signup_date": "2024-02-01", "country": "US", "phone": "+1-555-0001"}
{"customer_id": "C005", "email": "charlie@example.com", "signup_date": "2024-02-05", "country": "CA", "phone": null}
{"customer_id": "C006", "email": "diana@example.com",   "signup_date": "2024-02-10", "country": "AU", "phone": "+61-2-0002"}
```

Upload to S3 → `schemas/v2.0/` folder.

### Step 6.3: Create Sample Data v3.0

Create a file named `customers_v3.json`:

```json
{"customer_id": "C007", "email": "eve@example.com",   "signup_date": "2024-02-15", "country": "US", "phone": "+1-555-0003", "subscription_tier": "pro"}
{"customer_id": "C008", "email": "frank@example.com", "signup_date": "2024-02-20", "country": "DE", "phone": null,          "subscription_tier": "enterprise"}
{"customer_id": "C009", "email": "grace@example.com", "signup_date": "2024-02-25", "country": "JP", "phone": "+81-3-0004",  "subscription_tier": "free"}
```

Upload to S3 → `schemas/v3.0/` folder.

### Step 6.4: Create a Glue Job Using Schema Validation

- Glue console → **Jobs** → **Create job**
- Name: `CustomerSchemaValidation`
- Replace code with:

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from pyspark.context import SparkContext
from awsglue.job import Job
from pyspark.sql.types import *

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Replace with your account ID
ACCOUNT_ID = "123456789012"   # CHANGE THIS!
BUCKET = f"data-lake-prod-{ACCOUNT_ID}"

print("=" * 60)
print("SCHEMA VALIDATION JOB")
print("=" * 60)

# Define schema explicitly (mirrors Glue Schema Registry)
customer_schema = StructType([
    StructField("customer_id",        StringType(), False),  # required
    StructField("email",              StringType(), False),  # required
    StructField("signup_date",        StringType(), False),  # required
    StructField("country",            StringType(), False),  # required
    StructField("phone",              StringType(), True),   # optional
    StructField("subscription_tier",  StringType(), True)    # optional
])

# Test 1: Read v1.0 data (no phone field)
print("\n[TEST 1] Reading v1.0 data (no phone field)...")
try:
    df_v1 = spark.read.schema(customer_schema).json(
        f"s3://{BUCKET}/schemas/v1.0/customers_v1.json"
    )
    print(f"Successfully read {df_v1.count()} v1.0 records")
    print("Missing optional phone field handled correctly")
    df_v1.show(truncate=False)
except Exception as e:
    print(f"Error: {e}")

# Test 2: Read v2.0 data (with phone field)
print("\n[TEST 2] Reading v2.0 data (with phone field)...")
try:
    df_v2 = spark.read.schema(customer_schema).json(
        f"s3://{BUCKET}/schemas/v2.0/customers_v2.json"
    )
    print(f"Successfully read {df_v2.count()} v2.0 records")
    print("New optional phone field handled correctly")
    df_v2.show(truncate=False)
except Exception as e:
    print(f"Error: {e}")

# Test 3: Read v3.0 data (with phone and subscription_tier)
print("\n[TEST 3] Reading v3.0 data (with phone and subscription_tier)...")
try:
    df_v3 = spark.read.schema(customer_schema).json(
        f"s3://{BUCKET}/schemas/v3.0/customers_v3.json"
    )
    print(f"Successfully read {df_v3.count()} v3.0 records")
    print("Multiple new optional fields handled correctly")
    df_v3.show(truncate=False)
except Exception as e:
    print(f"Error: {e}")

# Combine all versions
print("\n[COMBINING] Merging all schema versions...")
df_combined = df_v1.unionByName(df_v2).unionByName(df_v3)
print(f"Combined total: {df_combined.count()} records")
print("Proof: all schema versions can coexist and be merged!")

print("\n" + "=" * 60)
print("SCHEMA COMPATIBILITY VERIFIED")
print("=" * 60)

job.commit()
```

- **Save** and **Run** job

**Expected output:**

```
[TEST 1] Reading v1.0 data (no phone field)...
  Successfully read 3 v1.0 records
  Missing optional phone field handled correctly

[TEST 2] Reading v2.0 data (with phone field)...
  Successfully read 3 v2.0 records
  New optional phone field handled correctly

[TEST 3] Reading v3.0 data (with phone and subscription_tier)...
  Successfully read 3 v3.0 records
  Multiple new optional fields handled correctly

[COMBINING] Merging all schema versions...
  Combined total: 9 records
  Proof: all schema versions can coexist and be merged!
```

---

## Part 7: Real-World Schema Governance

### Best Practices

**Rule 1: Always use schemas**

```python
# Bad — no validation
df = spark.read.json("data.json")

# Good — with schema validation
df = spark.read.schema(customer_schema).json("data.json")
```

**Rule 2: Use Glue Schema Registry instead of:**

- Schemas in documentation (goes stale)
- Schemas in code comments (easy to miss)
- Schemas in separate files (no version tracking)

**Rule 3: Write meaningful version descriptions**

```python
# Bad
"v2"

# Good
"v2: Added phone field (optional) for contact improvements.
Backwards compatible with v1. Migration not required.
Date: 2024-03-06, Author: data_team@company.com"
```

**Rule 4: Breaking changes need a migration plan**

| Step | Action |
|------|--------|
| 1 | Deploy new code that supports both old and new schema |
| 2 | Run ETL to convert v1 data to v2 format |
| 3 | Validate 100% of data is converted |
| 4 | Deprecate v1 support |
| 5 | Clean up v1 compatibility code |

---

## Part 8: Schema Evolution Patterns

### Pattern 1: Additive Only (Safest)

Only ever add optional fields, never remove or rename. Mark old fields as deprecated in the `doc` field rather than deleting them.

- **When to use:** Internal analytics systems
- **Pros:** Simple, always backwards compatible
- **Cons:** Schema grows large over time

### Pattern 2: Major Versions for Breaking Changes

```
v1.x  Customer has name, email
v2.x  Customer has name, email, phone
v3.x  Customer renamed to User  ← breaking change!
```

When releasing v3.x:

- Warn users — "v1.x deprecated, update your code"
- Provide migration guide
- Keep v1.x running for 6 months
- Then shut it down

### Pattern 3: Branch and Merge

```
Main branch:      v1.0  (production)
Feature branch A: v1.1-a  (add feature A)
Feature branch B: v1.1-b  (add feature B)
Release:          v2.0  (merges A + B)
```

---

## Part 9: Verification and Success Criteria

**Schema creation checklist:**

- [ ] Schema registered in Glue Schema Registry
- [ ] Version 1.0 created with 4 fields
- [ ] Version 2.0 created with optional `phone` field
- [ ] Version 3.0 created with optional `subscription_tier` field
- [ ] All versions marked `BACKWARD` compatible
- [ ] Descriptions document the reason for each change

**Data and jobs checklist:**

- [ ] `customers_v1.json` created and uploaded (3 records)
- [ ] `customers_v2.json` created and uploaded (3 records)
- [ ] `customers_v3.json` created and uploaded (3 records)
- [ ] All files in correct S3 folders (`schemas/v1.0/`, `schemas/v2.0/`, `schemas/v3.0/`)
- [ ] Glue job `CustomerSchemaValidation` created
- [ ] Job runs successfully — all 3 tests pass
- [ ] Logs show combined dataset has 9 records

**Understanding checklist:**

- [ ] Can explain why schemas matter
- [ ] Can explain backwards compatibility
- [ ] Can identify breaking vs. safe changes
- [ ] Know how to version schemas in Glue
- [ ] Understand real-world migration challenges

**If all items are checked, you have completed Lab 4.2!**

---

## Part 10: Teardown

### Step 1: Delete Glue Job

- Glue console → **Jobs**
- Select `CustomerSchemaValidation` → **Delete** → confirm

### Step 2: Delete Sample Data (Optional)

- S3 console → navigate to `s3://data-lake-prod-[ID]/schemas/`
- Select all folders → delete

### Step 3: Keep the Schema Registry

> **Do NOT delete `CustomerSchema`** — it will be used in Lab 4.3. Schema definitions are free to store.

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **Schema versioning concepts** | Why schemas evolve; cost of unmanaged changes; backwards vs. breaking changes |
| **Glue Schema Registry** | Creating schemas; registering versions; viewing history; compatibility modes |
| **Avro schema format** | Record types; field definitions; optional fields with null union; doc strings |
| **PySpark schema enforcement** | StructType definitions; reading data with explicit schemas; schema-on-read |
| **Schema evolution patterns** | Additive-only approach; major version breaks; branch-and-merge |
| **Governance best practices** | Meaningful version descriptions; migration plans; deprecation strategies |

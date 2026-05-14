# Redshift Cluster Setup + Security

---

## Part 0: Understanding Why Redshift Matters

### The Problem Redshift Solves

Imagine you work at Netflix. Every day:

- 200 million people watch videos
- 1 trillion events happen (play, pause, rewind, etc.)
- 50+ terabytes of new data arrives daily

**Why NOT just use a regular database?**

| | Regular Database (MySQL, PostgreSQL) | Redshift (Data Warehouse) |
|--|--------------------------------------|--------------------------|
| **Optimized for** | Transactions (1 person updating 1 row) | Analytics (analyzing millions of rows) |
| **Handles comfortably** | Up to ~100 GB | 50 TB+ |
| **Query example** | `SELECT COUNT(*) WHERE country='India'` | Same query |
| **Query time** | 30 minutes | 5 seconds |
| **Cost per query** | $1,000 | $0.10 |

Your analysts wait. Your business decisions are delayed. Revenue suffers. This is where Redshift comes in.

### How Redshift Works: Columnar Storage

**Regular database — stores by row:**

```
Row 1: CustomerID=1, Product=Shoes, Price=50, Date=2024-01-01
Row 2: CustomerID=2, Product=Hat,   Price=20, Date=2024-01-01

Query "Count shoes sold" = must check every single row (slow)
```

**Redshift — stores by column:**

```
Products column: [Shoes, Hat, Shoes, Shoes, Hat, ...]
Prices column:   [50, 20, 50, 50, 20, ...]

Query "Count shoes sold" = only reads the Products column (fast!)
```

When you analyze data, you usually only need a few columns at a time — so Redshift skips everything you don't need. This makes queries up to **100x faster**.

### Real-World Numbers

**Question: "How many videos were watched in India in 2024?"** — Data: 1 trillion events, 50 TB

| | Regular Database | Redshift |
|--|-----------------|---------|
| **Time** | 30 minutes | 5 seconds |
| **Cost per query** | $1,000 | $0.10 |
| **10 queries/day cost** | $10,000/day | $10/day |

This is why Netflix, Airbnb, and Uber use Redshift.

### Why Your Career Needs This

- Redshift job postings: 5,000 in 2020 → 65,000 in 2024 (fastest-growing data tool)
- Data Engineer **with** Redshift: $140,000/year average
- Data Engineer **without**: $95,000/year average
- **Difference: $45,000/year**

---

## Part 1: When to Use Redshift vs. Other Options

| Tool | Best For | Cost | Speed |
|------|---------|------|-------|
| **S3** | Raw data storage, archiving | $0.023/GB/month | Slow (scans entire bucket) |
| **Athena** | Occasional ad-hoc SQL on S3 | $6.25 per TB scanned | 10–60 seconds per query |
| **Redshift** | Repeated fast queries on same data | $0.25/hour per node | 1–10 seconds per query |
| **Spark/EMR** | Complex transformations, ML | $0.12/hour per node | 5–30 minutes per job |

**Why Redshift instead of Athena for dashboards?**

Scenario: Analytics dashboard queried 10,000 times/day

| | Athena | Redshift |
|--|--------|---------|
| **Daily cost** | $625/day | $6/day |
| **Monthly cost** | $18,750 | $180 |
| **Query speed** | 30 seconds (users waiting) | 2 seconds (users happy) |

---

## Part 2: What You'll Build in This Lab

- **Redshift Cluster** — 2 nodes (dc2.large), 160 GB storage, optimized for cost
- **Security Setup** — VPC deployment, private subnet, IAM role, Secrets Manager, security group
- **Network Configuration** — VPC security group, S3 VPC endpoint, IP-restricted access
- **Monitoring and Logs** — CloudWatch logs, query logs, cost tracking
- **Test Setup** — Sample database, test table, verified connection

**Design decisions:**

| Choice | Why |
|--------|-----|
| Private subnet | No internet exposure |
| IAM role | No hardcoded credentials |
| Secrets Manager | Secure password storage |
| VPC endpoint | Data stays in VPC (compliance) |
| Security group | Only you can connect |
| CloudWatch logs | Track and debug everything |
| 2 nodes | Minimum for high availability |

---

## Part 3: The "Why" Behind Architecture Decisions

### Why Private Subnet, Not Public?

**Bad approach (public internet):** Hacker finds cluster IP → tries 1,000 passwords → gets in → steals data → company loses $5M in GDPR/HIPAA fines

**Good approach (private VPC only):** Hacker finds cluster IP → tries to connect → "Not accessible from internet" → connection fails, data safe

### Why IAM Role Instead of Passwords in Code?

**Bad — password in code:**

```python
redshift = connect(
    user="admin",
    password="SuperSecret123"  # EXPOSED IN CODE
)
```

If code leaks, passwords leak. Can't rotate easily. No audit trail.

**Good — IAM role:**

```python
redshift = connect(
    role="RedshiftIAMRole"  # No password in code
)
```

Only AWS knows the password. Auto-rotates. Full audit trail. Industry standard.

### Why VPC Endpoint for S3?

**Without endpoint:** Redshift → Internet Gateway → across the internet → S3 *(security risk, bandwidth charges)*

**With endpoint:** Redshift → VPC endpoint (internal AWS network, never leaves VPC) → S3 *(faster, cheaper, more secure)*

### Why CloudWatch Logs?

**Without logs:** Query fails → no idea why → 2 hours debugging

**With logs:** Query fails → check logs → "Column 'age' doesn't exist" → fixed in 5 minutes

---

## Part 4: Prerequisites — Check These First

**Completed Labs:**

- Lab 1.1: IAM Setup
- Lab 1.2: VPC and Network
- Lab 1.3: S3 Data Lake

**Information You Need:**

- AWS Account ID (from Lab 1.1)
- VPC ID (from Lab 1.2)
- Private subnet IDs (from Lab 1.2)
- `RedshiftIAMRole` ARN (from Lab 1.1)
- S3 bucket name (from Lab 1.3)
- Security group IDs (from Lab 1.2)
- Your current IP address (Google "what is my IP")

**Tools:**

- Web browser
- Text editor for saving notes

> **Cost Warning:** This lab costs ~$1.08/hour while running. Full Tier 3 (all 3 labs) = ~$25–30 total. Run the teardown at the end to stop charges immediately.

---

## Part 5: Understanding Redshift Components

| Component | What It Is | What You'll Create |
|-----------|-----------|-------------------|
| **Cluster** | Collection of computing nodes working together | 2 nodes, 160 GB each |
| **Database** | Contains your tables, lives inside the cluster | `analytics` database |
| **Parameter Group** | Settings like memory allocation for queries | Default (good for learning) |
| **Subnet Group** | Which VPC subnets can host the cluster | Your Lab 1.2 private subnets |
| **Security Group** | Firewall — who can connect | Your IP only |
| **Master User** | Admin account created with the cluster | `awsadmin` + your password |
| **Cluster Identifier** | Unique name (like a domain name) | `redshift-tier3-lab` |

---

## Part 6: Step-by-Step Lab

### Step 6.1: Navigate to Redshift Console

- Open [https://console.aws.amazon.com](https://console.aws.amazon.com) and log in
- Click **Services**, search `redshift`, click **Amazon Redshift**
- You should see the Redshift dashboard with 0 clusters and a **Create cluster** button

### Step 6.2: Click Create Cluster

Click the orange **Create cluster** button in the top right. You'll see multiple sections to fill out — complete them in order below.

### Step 6.3: General Cluster Settings

| Field | Value | Why |
|-------|-------|-----|
| Cluster identifier | `redshift-tier3-lab` | Clear, identifies which lab tier |
| Database name | `analytics` | Your main analytics database |
| Admin username | `awsadmin` | Standard AWS naming |
| Admin user password | Create a strong password (8+ chars, upper, lower, numbers, symbols) | You'll need this in Lab 3.2 |

> **Save your password now** — write it somewhere safe (not in code). Example format: `RedshiftLab2024!@` — but create your own.

### Step 6.4: Node Type Configuration

| Field | Value | Cost |
|-------|-------|------|
| Node type | `dc2.large` | $0.54/hour per node |
| Number of nodes | 2 | $1.08/hour total |

> **Why 2 nodes?** One node = single point of failure. Two nodes = high availability — if one fails, the other continues. This is production-grade design.

### Step 6.5: VPC and Network Configuration

| Field | Value |
|-------|-------|
| VPC | Select your Lab 1.2 VPC |
| Cluster subnet group | Create new (see below) |
| Publicly accessible | **OFF / Disabled** |

**Create new subnet group:**

- Name: `redshift-tier3-subnet-group`
- Description: `Subnet group for Tier 3 Redshift labs`
- VPC: Your Lab 1.2 VPC
- Subnets: Select both private subnets (`private-subnet-1a` and `private-subnet-1b`)
- Click **Create**

### Step 6.6: VPC Security Group

Create a new security group with these settings:

| Field | Value |
|-------|-------|
| Name | `redshift-security-group` |
| VPC | Your Lab 1.2 VPC |
| Inbound rule — Type | Redshift (port 5439) |
| Inbound rule — Source | Your IP address only (e.g., `203.0.113.45/32`) |
| Outbound rules | Leave as default (allow all) |

Select this security group back on the cluster creation page.

### Step 6.7: IAM Roles

- Find the **Associate IAM roles** section
- Select your `RedshiftIAMRole` from Lab 1.1
- ARN format: `arn:aws:iam::123456789012:role/RedshiftIAMRole`

This allows Redshift to run COPY commands that read from S3 without any hardcoded passwords.

### Step 6.8: Encryption

- Toggle encryption **ON**
- KMS key: Select `aws/redshift` (default, free)

Encrypts all data at rest — required for GDPR, HIPAA, and PCI-DSS compliance.

### Step 6.9: Backup and Maintenance

| Field | Value |
|-------|-------|
| Backup retention period | 2 days |
| Maintenance window | Leave as default (Sun 3 AM UTC) |

### Step 6.10: CloudWatch Logs

Enable all four log types:

- CloudWatch Logs
- **Userlog** (tracks what queries ran)
- **Connectionlog** (tracks who connected and when)
- **Systemlog** (tracks cluster health)

### Step 6.11: Tags for Cost Tracking

| Key | Value |
|-----|-------|
| Tier | 3 |
| Lab | 3.1 |
| Environment | Learning |

### Step 6.12: Review and Create

Confirm these settings before clicking **Create cluster**:

- Cluster identifier: `redshift-tier3-lab`
- Database: `analytics`
- Nodes: 2 x dc2.large
- VPC: Your Lab 1.2 VPC
- Private subnets: Both selected
- IAM role: `RedshiftIAMRole` attached
- Encryption: Enabled
- Logging: All enabled

Click the orange **Create cluster** button.

**Expected:** "Creating cluster..." message. Wait **10–15 minutes** for status to change from `CREATING` to `AVAILABLE`.

### Steps 6.13 & 6.14: Monitor and Verify

Click your cluster name and watch the **Status** field. Once it shows `AVAILABLE`, copy and save:

```
Redshift Connection Details:
  Endpoint: [COPY FROM CLUSTER DETAILS]
  Port:     5439
  Database: analytics
  Username: awsadmin
  Password: [THE ONE YOU SET EARLIER]
```

---

## Part 7: Connect and Test

### Step 7.1: Open Query Editor

- In Redshift console, click **Query Editor v2** in the left sidebar

### Step 7.2: Create a Connection

| Field | Value |
|-------|-------|
| Connection name | `tier3-lab-connection` |
| Cluster | `redshift-tier3-lab` |
| Database name | `analytics` |
| Username | `awsadmin` |
| Password | Your password |
| Port | `5439` |

Click **Create connection**.

> If connection fails: verify your password, confirm cluster status is `AVAILABLE`, and check your security group allows your current IP.

### Step 7.3: Run Your First Query

```sql
SELECT
    version(),
    current_user,
    current_database;
```

**Expected output:**

```
version                               | current_user | current_database
PostgreSQL 8.0.2 on i686-pc-linux-gnu | awsadmin     | analytics
```

### Step 7.4: Create a Test Table

```sql
CREATE TABLE customers (
    customer_id    INTEGER,
    customer_name  VARCHAR(100),
    country        VARCHAR(50),
    signup_date    DATE
);
```

### Step 7.5: Insert Test Data

```sql
INSERT INTO customers VALUES
(1, 'Alice Johnson', 'USA',       '2024-01-15'),
(2, 'Bob Smith',     'Canada',    '2024-01-20'),
(3, 'Charlie Brown', 'USA',       '2024-02-01'),
(4, 'Diana Prince',  'UK',        '2024-02-10'),
(5, 'Eve Wilson',    'Australia', '2024-02-15');
```

### Step 7.6: Query Test Data

```sql
SELECT
    customer_id,
    customer_name,
    country,
    signup_date
FROM customers
ORDER BY customer_id;
```

**Expected output:**

```
customer_id | customer_name  | country   | signup_date
1           | Alice Johnson  | USA       | 2024-01-15
2           | Bob Smith      | Canada    | 2024-01-20
3           | Charlie Brown  | USA       | 2024-02-01
4           | Diana Prince   | UK        | 2024-02-10
5           | Eve Wilson     | Australia | 2024-02-15
```

### Step 7.7: Check Cluster Metrics

- Go back to the cluster details page
- Click the **Monitoring** tab
- Confirm: CPU utilization low (~5%), 1 database connection, some read/write activity

---

## Part 8: Security Verification

**Step 8.1: Encryption**
Cluster details → find **Encryption** → confirm: `Encrypted: Yes`, `KMS key: aws/redshift`

**Step 8.2: VPC Setup**
Cluster details → **Network and security** → confirm: VPC is your Lab 1.2 VPC, `Publicly accessible: No`

**Step 8.3: IAM Role**
Cluster details → **IAM roles** → confirm: `RedshiftIAMRole` is attached

**Step 8.4: Security Group**
Cluster details → **Security groups** → click the group name → confirm inbound rule: port `5439`, source is your IP only

---

## Part 9: Documentation

Save this as `Lab_3_1_Redshift_Setup.txt`:

```
=== LAB 3.1: REDSHIFT CLUSTER SETUP ===
Date Created: [TODAY'S DATE]
AWS Account ID: [YOUR ACCOUNT ID]

CLUSTER INFORMATION:
  Cluster Name:     redshift-tier3-lab
  Endpoint:         [COPY FROM CLUSTER DETAILS]
  Port:             5439
  Database:         analytics
  Master Username:  awsadmin
  Master Password:  [YOU SET THIS — KEEP SAFE]

NODE CONFIGURATION:
  Node Type:       dc2.large
  Number of Nodes: 2
  Total Storage:   320 GB
  Cost:            ~$1.08/hour

NETWORK CONFIGURATION:
  VPC:                 [YOUR VPC ID FROM LAB 1.2]
  Subnet Group:        redshift-tier3-subnet-group
  Private Subnets:     [LIST THEM]
  Security Group:      redshift-security-group
  Publicly Accessible: No

CLOUDWATCH LOGS:
  UserLog:       Enabled
  ConnectionLog: Enabled
  SystemLog:     Enabled

SECURITY CONFIGURATION:
  Encryption:  Enabled (aws/redshift KMS)
  Master User: awsadmin
  IAM Role:    arn:aws:iam::[ACCOUNT_ID]:role/RedshiftIAMRole
  S3 Access:   Configured for COPY commands

BACKUP:
  Retention Period:  2 days
  Automated Backups: Enabled

COST ESTIMATE:
  Per hour:                        $1.08
  Per day (24 hours):              $25.92
  Total for Tier 3 (all 3 labs):   ~$25–30
```

---

## Success Criteria — Verify You're Done

- [ ] Redshift cluster created with 2 nodes
- [ ] Cluster status shows `AVAILABLE`
- [ ] Cluster deployed in private VPC
- [ ] Encryption enabled
- [ ] CloudWatch logs enabled
- [ ] IAM role attached
- [ ] Query editor connection successful
- [ ] Test table created and query works
- [ ] Security group restricts access to your IP only
- [ ] All connection details documented
- [ ] Test data inserted and retrieved successfully

**If all items are checked, you have completed Lab 3.1!**

---

## Part 10: Cost Breakdown

| Item | Cost |
|------|------|
| 2 x dc2.large nodes @ $0.54/hr | $1.08/hour |
| Running 4 hours (this lab) | ~$4.32 |
| CloudWatch logs | ~$0.50 |
| Backup (minimal) | <$0.01/day |
| **Total for Lab 3.1** | **~$5–6** |

**If you run all 3 Tier 3 labs consecutively:**

- Lab 3.1 (4–5 hours): $5–6
- Lab 3.2 (3–4 hours): $3–4
- Lab 3.3 (3–4 hours): $3–4
- **Total: ~$12–15**

---

## Part 11: Teardown — Do This After Completing All of Tier 3

> ⚠️ **Warning:** Deleting the cluster is permanent. Complete Labs 3.2 and 3.3 before doing this.

### Step 1: Optional — Back Up Final Data

Export any query results you want to keep from the Query Editor.

### Step 2: Delete Redshift Cluster

- Redshift console → click `redshift-tier3-lab`
- Click **Delete** → confirm "I want to delete this cluster"
- "Create final snapshot?" → **No** (unless you need a backup)
- Click **Delete cluster** — takes 3–5 minutes

### Step 3: Delete Cluster Subnet Group

- Redshift console → **Subnet groups** (left sidebar)
- Select `redshift-tier3-subnet-group` → **Delete**

### Step 4: Delete Security Group

- VPC console → **Security groups**
- Select `redshift-security-group` → **Delete**

### Step 5: Verify in Billing Console

- Go to **AWS Billing** → **Costs by service**
- Redshift should show $0 — charges stop the moment the cluster is deleted

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **Redshift Concepts** | What Redshift is; columnar storage; when to use it vs. other tools |
| **Cluster Setup** | Node types, shard count, database configuration |
| **Security** | Private VPC deployment; IAM roles; security groups; encryption |
| **Networking** | Subnet groups; VPC endpoints; IP-restricted access |
| **Monitoring** | CloudWatch logs; cluster health metrics |
| **SQL Basics** | CREATE TABLE, INSERT, SELECT on a real data warehouse |
| **Cost Management** | Hourly billing; teardown to stop charges; cost tagging |

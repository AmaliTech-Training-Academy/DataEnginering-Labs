# Apache Airflow on Managed Workflows (MWAA)

---

## Part 0: Understanding Why This Matters

### The Story: Why Step Functions Isn't Enough

You completed Lab 5.1 with Step Functions. It works great. Then your manager arrives Monday morning:

> "We need daily ETL at 2 AM, weekly at midnight, monthly on the 1st. Different configurations for each. If it fails, retry — but not if it's a connection error. We need to backfill missing data from 6 months ago. We need different notifications based on which step failed. We need to share the orchestration platform with other teams. Can Step Functions handle this?"

Step Functions *can* do all of this — but with very complex JSON, poor version control, limited team collaboration, and no portability to other clouds.

**Enter Apache Airflow.**

### What is Apache Airflow?

Airflow = a DAG orchestration platform.

| Term | Plain English |
|------|--------------|
| **DAG** | Directed Acyclic Graph — a fancy name for a workflow graph where tasks connect by dependencies |
| **Orchestration** | Run these tasks in this order |
| **Platform** | Provides tools for scheduling, monitoring, and alerting |

### Real-World Example: Netflix Data Pipeline

Netflix uses Airflow to orchestrate thousands of pipelines daily:

1. Trigger data collection from all regions (10 parallel tasks)
2. Wait for all to finish
3. Validate data quality
4. Merge all regions (deduplication)
5. Run ML model for recommendations
6. Load recommendations to cache
7. Notify users of updated recommendations
8. Log metrics to monitoring

This runs thousands of times per day, automatically.

### Airflow vs. Step Functions: When to Use Which

| Use Step Functions When | Use Airflow When |
|------------------------|-----------------|
| Simple workflows (under 10 steps) | Complex workflows (10+ steps) |
| AWS-only environment | May use multiple clouds later |
| Quick implementation needed | Need complex scheduling (cron, custom) |
| Small team | Team needs collaboration and code sharing |
| Don't need portability | Industry-standard approach required |

> **Key insight:** Most big companies use Airflow — Netflix, Uber, Airbnb, Spotify, Slack, and others.

### Why MWAA (Managed Airflow)?

| | Self-Hosted on EC2 | MWAA (AWS Managed) |
|--|-------------------|--------------------|
| **Cost** | $50–100/month | $0.29/hour (~$210/month) |
| **Management** | You patch, update, scale | AWS handles everything |
| **Control** | Full customization | Standard configuration |
| **Best for** | Production with ops team | Learning and getting started |

### Why This Matters for Your Career

- "Experience with Airflow" appears in the majority of senior data engineering job postings
- Airflow skill: **+$20–30K/year** salary impact
- 70% of tech companies with data teams use Airflow
- Portable skill — works on AWS, Google Cloud, Azure, and on-premises

---

## Part 1: Goals for This Lab

By the end of this lab, you will:

- Understand Apache Airflow architecture
- Know what DAGs are and how to write them
- Create a real-world data pipeline DAG
- Use operators to orchestrate AWS services
- Implement error handling and retries
- Schedule pipelines with cron expressions
- Monitor DAG execution via the Airflow UI
- Debug failed tasks
- Understand when to use Airflow vs. Step Functions

---

## Part 2: What You'll Build

### Your Complete Airflow DAG

```
[Daily trigger: 2:00 AM]
        ↓
[Prerequisites check — 3 parallel tasks]
  Task 1: Check S3 bucket exists
  Task 2: Check Redshift cluster is up
  Task 3: Validate credentials
        ↓ (wait for all 3)
[Data extraction — 3 parallel tasks]
  Task 4: Glue crawl new data
  Task 5: Glue extract job
  Task 6: Validate extracted data
        ↓
[Data quality checks — 5 parallel tasks]
  Task 7a: Check row count
  Task 7b: Check null values
  Task 7c: Check duplicates
  Task 7d: Check date ranges
  Task 7e: Check freshness
        ↓ (all must pass)
[Transformation — 3 parallel tasks]
  Task 8:  Clean data (Lambda)
  Task 9:  Deduplicate (Glue)
  Task 10: Aggregate (Glue)
        ↓
[Loading — 2 parallel tasks]
  Task 11: Copy to Redshift
  Task 12: Update metadata table
        ↓
[Notifications]
  Task 13: Success email
  Task 14: Metrics to CloudWatch
        ↓
[Complete]
```

### Core Airflow Concepts

**DAG — your workflow:**

```python
with DAG('my_pipeline') as dag:
    # Your tasks go here
```

**Task — individual job:**

```python
task = GlueJobOperator(job_name='my-job')
```

**Operators — the type of task:**

| Operator | What It Does |
|----------|-------------|
| `PythonOperator` | Runs a Python function |
| `BashOperator` | Runs a bash script |
| `AwsGlueJobOperator` | Runs a Glue job |
| `RedshiftSQLOperator` | Runs a SQL query on Redshift |
| `S3KeySensor` | Waits for a file to appear in S3 |

**Dependencies — task order:**

```python
task1 >> task2 >> task3          # sequential: 1 then 2 then 3
[task1, task2] >> task3          # parallel: 1 and 2 run together, then 3
```

**Scheduling with cron:**

```
0 2 * * *    Daily at 2 AM
0 2 * * 0    Every Sunday at 2 AM
0 2 1 * *    1st of every month at 2 AM
```

---

## Part 3: The "Why" Behind Design Decisions

### Why Airflow Over Step Functions for Complex Workflows?

**Conditional retry logic — same feature, very different experience:**

**Step Functions JSON (verbose):**

```json
{
  "Type": "Task",
  "Retry": [
    {
      "ErrorEquals": ["ConnectionException"],
      "MaxAttempts": 0
    },
    {
      "ErrorEquals": ["States.ALL"],
      "MaxAttempts": 3
    }
  ]
}
```

**Airflow Python (clean):**

```python
@task(retries=3, retry_delay=timedelta(minutes=5))
def my_task():
    return "do something"
```

**Other Airflow advantages:**

- Code-based (Python) — easier to read, write, and review
- Powerful scheduling built in (no need for EventBridge)
- Cloud-agnostic — same DAG works on AWS, GCP, or Azure
- Better team collaboration via standard code versioning (Git)

### Why Python DAGs?

**Flexibility — use any Python logic:**

```python
if is_weekend():
    skip_data_validation()
else:
    run_data_validation()
```

**Code reuse — define once, use everywhere:**

```python
validate_task = PythonOperator(
    task_id='validate_data',
    python_callable=validate_data
)

# Reuse in multiple DAGs
dag1 >> validate_task >> other_task1
dag2 >> validate_task >> other_task2
```

**Testing — DAGs are just Python:**

```python
def test_my_task():
    result = my_task()
    assert result['status'] == 'success'
```

Step Functions can't do any of this — JSON is static.

---

## Prerequisites

- Completed Tier 1 (IAM, VPC, S3)
- Completed Tier 4 (Lambda, DynamoDB)
- Completed Lab 5.1 (Step Functions)
- Basic Python knowledge (can read Python code)
- AWS account with $50+ available credit
- Python 3.8+ installed locally
- 5 hours uninterrupted

---

## Part 4: Set Up MWAA Environment

### Step 1: Create S3 Bucket for Airflow

Airflow stores DAGs and logs in S3.

- Services → **S3** → **Create bucket**
- Fill in:

| Field | Value |
|-------|-------|
| Bucket name | `airflow-mwaa-dags-[YOUR_ACCOUNT_ID]` |
| Region | Same as your other resources |
| Block public access | Keep enabled |

- Click **Create bucket**
- Inside the bucket, create three folders:

| Folder Name | Purpose |
|------------|---------|
| `dags/` | Your DAG Python files |
| `logs/` | Airflow execution logs |
| `plugins/` | Custom Airflow plugins |

### Step 2: Create IAM Role for MWAA

- Services → **IAM** → **Roles** → **Create role**
- Trust entity: AWS service → search `mwaa` → select **MWAA**
- Click **Next**, add these policies:

| Policy | Purpose |
|--------|---------|
| `AmazonMWAAWebServerAccess` | Access Airflow web UI |
| `AmazonMWAAExecutionRolePolicy` | Core MWAA permissions |
| `AmazonS3FullAccess` | DAG and log storage |
| `AWSGlueFullAccess` | Run Glue jobs |
| `AWSLambdaFullAccess` | Invoke Lambda functions |
| `AmazonRedshiftFullAccess` | Query Redshift |

- Name the role:

| Field | Value |
|-------|-------|
| Role name | `MWAAServiceRole` |
| Description | Service role for MWAA to access AWS services |

- Click **Create role**

### Step 3: Create MWAA Environment

- Services → search `mwaa` → **Managed Workflows for Apache Airflow**
- Click **Create environment**
- Fill in each section:

**Environment details:**

| Field | Value |
|-------|-------|
| Name | `data-pipeline` |
| DAGs S3 bucket | `airflow-mwaa-dags-[YOUR_ACCOUNT_ID]` |
| DAGs subfolder | `dags/` |

**Execution role:** Select `MWAAServiceRole`

**Networking:**

| Field | Value |
|-------|-------|
| VPC | Your VPC from Tier 1 |
| Subnets | Select 2 private subnets |

**Web server access:** Public network access **ON** *(for this lab only — turn off in production)*

**Environment configuration:**

| Field | Value |
|-------|-------|
| Core workers | 1 |
| Max workers | 2 |
| Worker capacity | `mw1.small` |
| Schedulers | 1 |

- Click **Create environment**

> Environment creation takes **15–20 minutes**. While you wait, write your DAG.

### Step 4: Write Your DAG

Create a file named `data_pipeline.py`:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.amazon.aws.operators.glue import AwsGlueJobOperator
from airflow.providers.amazon.aws.operators.redshift_sql import RedshiftSQLOperator
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.exceptions import AirflowException

# Default arguments applied to all tasks
default_args = {
    'owner':            'data-engineering',
    'depends_on_past':  False,
    'start_date':       datetime(2024, 1, 1),
    'email':            ['admin@company.com'],   # Change this!
    'email_on_failure': True,
    'email_on_retry':   False,
    'retries':          2,
    'retry_delay':      timedelta(minutes=5),
}

# Define the DAG
dag = DAG(
    'data_pipeline_orchestration',
    default_args=default_args,
    description='End-to-end data pipeline with quality checks',
    schedule_interval='0 2 * * *',   # Daily at 2 AM
    catchup=False,                   # Don't backfill missed runs
    tags=['data-engineering', 'production'],
)

# ─── Python functions for tasks ───────────────────────────────────

def check_prerequisites(**context):
    """Check if prerequisites are met before pipeline starts."""
    print("Checking prerequisites...")
    print("S3 bucket exists")
    print("Redshift cluster is up")
    print("Credentials are valid")
    return {'status': 'prerequisites_met'}

def validate_extracted_data(**context):
    """Validate that extraction returned data."""
    print("Validating extracted data...")
    print("Row count > 0")
    print("All required columns present")
    return {'status': 'validation_passed'}

def quality_checks(**context):
    """Run data quality checks — raises exception if any fail."""
    checks = {
        'row_count':   'PASS',
        'null_values': 'PASS',
        'duplicates':  'PASS',
        'date_ranges': 'PASS',
        'freshness':   'PASS'
    }

    all_passed = all(v == 'PASS' for v in checks.values())

    if not all_passed:
        raise AirflowException('Quality checks failed')

    print(f"Quality checks: {checks}")
    return checks

def log_execution_success(**context):
    """Log successful execution."""
    execution_id = context['execution_date']
    print(f"Logging successful execution: {execution_id}")
    return {'logged': True}

def handle_failure(**context):
    """Handle pipeline failure."""
    task_instance = context['task_instance']
    print(f"Pipeline failed at task: {task_instance.task_id}")
    return {'failure_handled': True}

# ─── Define tasks ────────────────────────────────────────────────

task_check_prerequisites = PythonOperator(
    task_id='check_prerequisites',
    python_callable=check_prerequisites,
    dag=dag,
)

task_wait_for_data = S3KeySensor(
    task_id='wait_for_new_data',
    bucket_name='my-data-bucket',      # Replace with your bucket
    bucket_key='raw/**',
    wildcard_match=True,
    timeout=3600,
    poke_interval=60,
    dag=dag,
)

task_glue_crawler = AwsGlueJobOperator(
    task_id='run_glue_crawler',
    job_name='DataDiscoveryCrawler',   # Replace with your Glue job name
    dag=dag,
)

task_glue_etl = AwsGlueJobOperator(
    task_id='run_glue_etl',
    job_name='DataTransformationJob',  # Replace with your Glue job name
    dag=dag,
)

task_validate = PythonOperator(
    task_id='validate_extracted_data',
    python_callable=validate_extracted_data,
    dag=dag,
)

task_quality_checks = PythonOperator(
    task_id='run_quality_checks',
    python_callable=quality_checks,
    dag=dag,
)

task_copy_redshift = RedshiftSQLOperator(
    task_id='copy_to_redshift',
    sql='COPY my_table FROM s3://my-bucket/processed/ ...',  # Replace with real COPY command
    dag=dag,
)

task_update_metadata = BashOperator(
    task_id='update_metadata',
    bash_command='echo "Metadata updated"',
    dag=dag,
)

task_log_success = PythonOperator(
    task_id='log_execution_success',
    python_callable=log_execution_success,
    dag=dag,
)

# ─── Define task dependencies ─────────────────────────────────────

task_check_prerequisites >> task_wait_for_data
task_wait_for_data >> [task_glue_crawler, task_glue_etl]
[task_glue_crawler, task_glue_etl] >> task_validate
task_validate >> task_quality_checks
task_quality_checks >> task_copy_redshift
task_copy_redshift >> task_update_metadata
task_update_metadata >> task_log_success
```

### Step 5: Upload DAG to S3

- S3 console → navigate to `airflow-mwaa-dags-*/dags/`
- Click **Upload** → select `data_pipeline.py` → **Upload**

> Airflow checks S3 every 30 seconds and detects new DAGs automatically.

---

## Part 5: Monitor and Execute Your DAG

### Step 6: Access Airflow Web UI

- MWAA console → click on `data-pipeline` environment
- Click **Open Airflow UI** (top right)

### Step 7: Enable Your DAG

- In Airflow UI → click **DAGs**
- Find `data_pipeline_orchestration`
- Toggle it **ON**

The DAG is now active and will run automatically at 2 AM daily.

### Step 8: Trigger Manually for Testing

- Click on the DAG name
- Click **Trigger DAG** (play button, top right)
- Click **Trigger**

### Step 9: Monitor Execution

| View | What You See |
|------|-------------|
| **Graph View** | Visual workflow — green (success), red (failure), blue (running) |
| **Tree View** | All historical runs with status per task |
| **Task logs** | Click any task → Log to see detailed output for debugging |

---

## Part 6: Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| DAG doesn't appear in UI | S3 sync not complete yet | Wait 1–2 minutes; verify file is in `dags/` folder |
| Task fails with permission error | MWAA role missing permissions | Check IAM role policies; check S3 bucket policy |
| Glue task times out | Glue job is slow | Increase timeout in DAG; check Glue logs in CloudWatch |
| Out of capacity error | Not enough workers | Scale up workers in environment settings, or reduce parallel tasks |

---

## Success Criteria

- [ ] MWAA environment created and running
- [ ] DAG uploaded to S3 `dags/` folder
- [ ] DAG visible in Airflow UI
- [ ] DAG manually triggered
- [ ] All tasks completed (some failures are expected for this lab)
- [ ] Task logs are viewable
- [ ] Can explain the DAG structure and dependency flow

---

## Part 7: Teardown — Critical to Save Money

> ⚠️ MWAA costs **$0.29/hour** — approximately **$210/month**. Delete it immediately after finishing the lab.

### Step 1: Delete MWAA Environment

- MWAA console → click `data-pipeline` → **Delete**
- Type "delete" to confirm
- Wait 5–10 minutes for deletion to complete
- **Cost saved: ~$210/month**

### Step 2: Delete Associated RDS Database

MWAA automatically creates an RDS instance for its metadata. Delete it:

- **RDS console** → **Databases**
- Find the database linked to MWAA → **Delete**
- Uncheck "Create final snapshot" → confirm
- **Cost saved: ~$50/month**

### Step 3: Delete S3 Bucket

- S3 console → find `airflow-mwaa-dags-*`
- Empty the bucket first (delete all files)
- Delete the bucket → confirm
- **Cost saved: under $1**

### Step 4: Delete IAM Role

- **IAM** → **Roles** → find `MWAAServiceRole` → **Delete** → confirm

### Step 5: Verify Costs Have Stopped

- Services → **Billing and Cost Management** → **Cost Explorer**
- Check estimated charges — should show $0.00 going forward

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **Airflow architecture** | DAGs, tasks, operators, sensors — how they fit together |
| **Writing DAGs** | Python-based workflow definition; task functions; `default_args` |
| **Task dependencies** | Sequential and parallel dependencies using `>>` and list syntax |
| **Scheduling** | Cron expressions for daily, weekly, and monthly triggers |
| **Error handling** | Retries with delay; `AirflowException` for failing tasks; email on failure |
| **Monitoring** | Graph view, tree view, task logs — how to debug a failed run |
| **MWAA management** | Environment creation; IAM roles; S3 DAG deployment |
| **Cost awareness** | $0.29/hour rate; teardown steps; RDS cleanup |

---

## Real-World Airflow Examples

### Netflix Recommendation Pipeline (Daily at 2 AM)

1. Collect viewing data from 10 regions in parallel
2. Aggregate viewing behavior
3. Run ML recommendation model
4. Load recommendations to Redis cache
5. Serve updated recommendations to users

### Uber Pricing Pipeline (Every 15 Minutes)

1. Collect current ride requests
2. Collect driver supply data
3. Calculate demand surge
4. Update dynamic prices
5. Alert drivers of surge zones

### Airbnb Availability Pipeline (Every Hour)

1. Check booking system
2. Check inventory
3. Sync with Airbnb platform
4. Update search availability
5. Alert hosts of changes

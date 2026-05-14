# AWS Step Functions Orchestration

---

## Part 0: Understanding Why This Matters

### The Problem: Fragile Pipelines

It's 2:30 AM. Your phone buzzes. Manager: "Sales report not loading. FIX NOW."

**What happened:**

- Glue job ran at 2 AM to extract customer data
- Redshift COPY was supposed to run next
- Glue job failed silently
- Nobody knew until the report was empty

**What you have to do:**

- Wake up at 2:30 AM
- Check logs manually
- See Glue job failed
- Fix the issue
- Manually restart Glue job
- Manually run Redshift COPY
- Wait 30+ minutes for both to complete
- Verify data loaded correctly
- Go back to sleep at 4 AM — exhausted

**Why this happens without orchestration:**

- No automated monitoring
- No automatic retries
- No automatic triggering of the next job
- No failure alerts
- Jobs run independently, unaware of each other

### The Solution: Workflow Orchestration

**Same scenario, with Step Functions:**

```
2:00 AM   Workflow automatically starts
2:01 AM   Glue job runs, fails
2:02 AM   Automatically retries (2x)
2:03 AM   Still failing — workflow stops
2:03 AM   Error handler sends SNS alert (text + email)
2:04 AM   Email arrives: "Glue job failed — check S3 bucket permissions"

3:00 AM   You wake up naturally, check email
3:05 AM   You fix the issue (bad permissions on source bucket)
3:06 AM   You click "Re-run workflow" — one button
3:10 AM   Glue job succeeds
3:11 AM   Workflow automatically triggers Redshift COPY
3:25 AM   Redshift load completes
3:26 AM   Data quality check runs (Lambda)
3:27 AM   Quality check passes
3:28 AM   Automated notification: "Pipeline successful, data updated"
```

**Result:** You got an immediate alert, the error message told you exactly what was wrong, and the rest of the process ran automatically. You slept through most of it.

### What Orchestration Does

| Without Orchestration | With Orchestration |
|----------------------|-------------------|
| No automated monitoring | Monitors every step |
| No automatic retries | Retries automatically on failure |
| Must manually trigger next job | Triggers next step on success |
| Silent failures | Alerts immediately on failure |
| No audit trail | Logs everything |

### Real-World Analogy: Recipe with an Assistant

**Without orchestration:** You lay out the recipe steps. If step 3 (pouring the batter) fails, nobody knows. You wait 30 minutes for the oven to finish, then discover nothing was ever baked.

**With orchestration:** An assistant watches each step — did the oven reach 350°F? Did mixing take under 5 minutes? If anything fails, they alert you immediately, stop the next step, and wait for you to fix it before continuing.

### Why This Matters for Your Career

| Timeline | Impact |
|----------|--------|
| 1 year | You'll work with 50+ automated pipelines — orchestration is expected |
| 3 years | You'll design orchestration architecture and choose between Step Functions, Airflow, Prefect, and Dagster |
| 5 years | Orchestration expertise = staff/senior engineer level, worth +$30K/year |

---

## Part 1: Goals for This Lab

By the end of this lab, you will:

- Understand what Step Functions are and how they work
- Know how state machines control workflow design
- Create a multi-step workflow with error handling
- Use Lambda, DynamoDB, and SNS together
- Handle failures, retries, and alerts
- Monitor and debug workflow executions
- Understand when to use Step Functions vs. other tools

---

## Part 2: What You'll Build

### Your Complete Workflow

```
[Trigger]
    ↓
[Step 1: Validate input]   ──fail──→  [Alert: invalid input]  →  [End]
    ↓ pass
[Step 2: Run Glue job]     ──fail──→  [Alert: Glue failed]    →  [End]
  (auto-retries up to 3x)
    ↓ pass
[Step 3: Load to Redshift] ──fail──→  [Alert: load failed]    →  [End]
    ↓ pass
[Step 4: Quality checks]   ──fail──→  [Alert: QC failed]      →  [End]
  (5 checks run in parallel)
    ↓ all pass
[Step 5: Send success notification]
    ↓
[Step 6: Log execution to DynamoDB]
    ↓
[End]
```

### AWS Resources You'll Create

| Resource | Name | Purpose |
|----------|------|---------|
| Step Functions state machine | `DataPipelineOrchestration` | Workflow logic and flow control |
| Lambda function | `ValidateInput` | Checks if input data is valid |
| Lambda function | `DataQualityCheck` | Runs 5 quality checks |
| Lambda function | `LogExecution` | Stores results to DynamoDB |
| DynamoDB table | `PipelineExecutionLog` | Execution history for auditing |
| SNS topic | `PipelineSuccess` | Notifications when pipeline succeeds |
| SNS topic | `PipelineFailure` | Alerts when pipeline fails |
| IAM role | `StepFunctionsExecutionRole` | Permissions for Step Functions |

---

## Part 3: The "Why" Behind Design Decisions

### Why State Machines?

| | Without State Machines | With State Machines |
|--|------------------------|---------------------|
| **Flow control** | Script runs linearly, errors out | Each step explicitly succeeds or fails |
| **Failure handling** | Pipeline stops, nothing runs | Error handler triggered automatically |
| **Visibility** | Unclear what failed or why | Exact step and error visible |
| **Reliability** | Fragile | Robust, each transition is defined |

### Why Lambda for Validation and Quality Checks?

Lambda is perfect here because it's lightweight, trigger-based, scales automatically, has no idle cost, and returns results quickly. EC2, Redshift, or Glue would all be overkill for simple checks.

### Why DynamoDB for Logging?

DynamoDB offers automatic scaling, fast queries, pay-per-request pricing, and free tier coverage — making it far simpler than RDS and more queryable than S3 for execution history.

### Why Multiple Lambda Functions Instead of One Script?

| Single Script | Multiple Lambdas |
|---------------|------------------|
| Tightly coupled — validation knows about Glue | Each function is standalone and testable |
| Must test everything together | Can test each function independently |
| Can't reuse quality check elsewhere | Quality check can be reused in other workflows |
| Everything runs sequentially | Can run checks in parallel |

Step Functions orchestrates them — it handles flow and errors, not business logic.

---

## Prerequisites — Check These First

- Completed Tier 1 (IAM, VPC, S3)
- Understand Lambda basics
- Know what DynamoDB is
- AWS account with appropriate permissions
- Text editor for code
- 4 hours uninterrupted
- Basic Python knowledge (helpful, not required)

---

## Step 0: Login to AWS Console

- Go to [https://console.aws.amazon.com](https://console.aws.amazon.com)
- Login with your credentials
- Confirm you see the AWS Console home page

---

## Part 4: Build the State Machine

### Step 1: Create IAM Role for Step Functions

#### Step 1.1: Open IAM Console

- Services → search **IAM** → click IAM
- Click **Roles** in the left sidebar
- Click **Create role**

#### Step 1.2: Choose Trust Entity

- Select **AWS service**
- Use case: search `step` → select **Step Functions**
- Click **Next**

#### Step 1.3: Add Permissions

Search for and check each of these policies:

| Policy | Search Term |
|--------|------------|
| `AWSLambdaFullAccess` | lambda |
| `AmazonDynamoDBFullAccess` | dynamodb |
| `AmazonSNSFullAccess` | sns |
| `CloudWatchLogsFullAccess` | cloudwatch |

Click **Next**.

#### Step 1.4: Name the Role

| Field | Value |
|-------|-------|
| Role name | `StepFunctionsExecutionRole` |
| Description | Role for Step Functions to invoke Lambda, access DynamoDB, and publish to SNS |

Click **Create role**.

---

### Step 2: Create Lambda Functions

#### Step 2.1: Create `ValidateInput` Lambda

- Services → lambda → **Lambda** → **Create function**
- Fill in:

| Field | Value |
|-------|-------|
| Function name | `ValidateInput` |
| Runtime | Python 3.11 |
| Execution role | Use existing: `LambdaExecutionRole` (from Tier 1) |

- Click **Create function**
- Replace all code with:

```python
import json

def lambda_handler(event, context):
    """
    Validate that input has required fields.

    Expected input:
    {
        "source_s3_bucket": "my-bucket",
        "source_s3_prefix": "data/",
        "target_glue_job": "my-glue-job"
    }
    """
    required_fields = ['source_s3_bucket', 'source_s3_prefix', 'target_glue_job']

    for field in required_fields:
        if field not in event or not event[field]:
            return {
                'statusCode': 400,
                'valid': False,
                'error': f'Missing required field: {field}'
            }

    print(f"Validation passed for input: {event}")

    return {
        'statusCode': 200,
        'valid': True,
        'source_s3_bucket': event['source_s3_bucket'],
        'source_s3_prefix': event['source_s3_prefix'],
        'target_glue_job': event['target_glue_job']
    }
```

- Click **Deploy**

#### Step 2.2: Create `DataQualityCheck` Lambda

- Create function, name: `DataQualityCheck`, same runtime and role
- Replace all code with:

```python
import json
import random

def lambda_handler(event, context):
    """
    Run data quality checks on processed data.
    Returns pass/fail for each check.
    """
    checks = {
        'row_count_check': {
            'description': 'Verify row count > 0',
            'passed': random.randint(0, 1) == 1   # Random for demo
        },
        'null_check': {
            'description': 'Verify no null values in key columns',
            'passed': random.randint(0, 1) == 1
        },
        'duplicate_check': {
            'description': 'Verify no duplicate rows',
            'passed': True
        },
        'range_validation': {
            'description': 'Verify dates are within expected range',
            'passed': True
        },
        'freshness_check': {
            'description': 'Verify data is recent (< 1 hour old)',
            'passed': True
        }
    }

    all_passed = all(check['passed'] for check in checks.values())

    return {
        'all_checks_passed': all_passed,
        'checks': checks,
        'timestamp': event.get('execution_id')
    }
```

- Click **Deploy**

#### Step 2.3: Create `LogExecution` Lambda

- Create function, name: `LogExecution`, same runtime and role
- Replace all code with:

```python
import json
import boto3
from datetime import datetime

dynamodb = boto3.resource('dynamodb')

def lambda_handler(event, context):
    """
    Log pipeline execution to DynamoDB.
    """
    execution_id = event.get('execution_id', 'unknown')
    status       = event.get('status', 'unknown')

    log_entry = {
        'execution_id': execution_id,
        'status':       status,
        'timestamp':    datetime.now().isoformat(),
        'details':      event
    }

    print(f"Logging execution: {json.dumps(log_entry)}")

    return {
        'logged':       True,
        'execution_id': execution_id,
        'timestamp':    log_entry['timestamp']
    }
```

- Click **Deploy**

---

### Step 3: Create DynamoDB Table

- Services → dynamodb → **DynamoDB** → **Create table**
- Fill in:

| Field | Value |
|-------|-------|
| Table name | `PipelineExecutionLog` |
| Partition key | `execution_id` (String) |
| Billing mode | Pay-per-request |

- Click **Create table** — takes about 1 minute

---

### Step 4: Create SNS Topics

**Create `PipelineSuccess` topic:**

- Services → sns → **SNS** → **Create topic**
- Name: `PipelineSuccess`, Type: Standard → **Create topic**
- Click **Create subscription** → Protocol: Email → enter your email → **Create subscription**
- Check your email and confirm the subscription

**Create `PipelineFailure` topic:**

- Create topic: `PipelineFailure`, Type: Standard → **Create topic**
- Create subscription → Protocol: Email → same email → **Create subscription**
- Check your email and confirm the subscription

> Both subscriptions must be confirmed before alerts will be delivered.

---

### Step 5: Create Step Functions State Machine

- Services → step → **Step Functions** → **Create state machine**

#### Step 5.1

Select the visual builder, click **Next**.

#### Step 5.2

Click **"Write your own"** and paste this JSON. Replace all `ACCOUNT_ID` placeholders with your actual AWS account ID:

```json
{
  "Comment": "Data Pipeline Orchestration Workflow",
  "StartAt": "ValidateInput",
  "States": {

    "ValidateInput": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:ValidateInput",
      "Next": "CheckIfValid",
      "Catch": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "Next": "ValidationFailed"
        }
      ]
    },

    "CheckIfValid": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.valid",
          "BooleanEquals": true,
          "Next": "RunGlueJob"
        }
      ],
      "Default": "ValidationFailed"
    },

    "ValidationFailed": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT_ID:PipelineFailure",
        "Message": "Validation failed — missing required input fields"
      },
      "Next": "WorkflowFailed"
    },

    "RunGlueJob": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "my-glue-job"
      },
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 2,
          "BackoffRate": 1.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "GlueFailed"
        }
      ],
      "Next": "RunQualityCheck"
    },

    "GlueFailed": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT_ID:PipelineFailure",
        "Message": "Glue job failed after 2 retries — check job logs"
      },
      "Next": "WorkflowFailed"
    },

    "RunQualityCheck": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:DataQualityCheck",
      "Next": "CheckQuality"
    },

    "CheckQuality": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.all_checks_passed",
          "BooleanEquals": true,
          "Next": "QualityPassed"
        }
      ],
      "Default": "QualityFailed"
    },

    "QualityFailed": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT_ID:PipelineFailure",
        "Message": "Data quality check failed — review check results in logs"
      },
      "Next": "WorkflowFailed"
    },

    "QualityPassed": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT_ID:PipelineSuccess",
        "Message": "Pipeline completed successfully — data is ready"
      },
      "Next": "LogExecution"
    },

    "LogExecution": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:LogExecution",
      "End": true
    },

    "WorkflowFailed": {
      "Type": "Fail",
      "Error": "PipelineFailed",
      "Cause": "One or more pipeline steps failed"
    }

  }
}
```

#### Step 5.3: Finalize and Create

| Field | Value |
|-------|-------|
| State machine name | `DataPipelineOrchestration` |
| IAM role | `StepFunctionsExecutionRole` |

Click **Create state machine**.

**Expected:** State machine created — you can see the visual workflow diagram and execution history.

---

## Part 5: Test Your Workflow

### Step 6: Execute the Workflow

- Click **Start execution**
- Paste this input JSON:

```json
{
  "source_s3_bucket": "my-data-bucket",
  "source_s3_prefix": "raw/",
  "target_glue_job": "ETL_Pipeline"
}
```

- Click **Start execution**

**What will happen:**

- Workflow starts and validates input — passes
- Tries to run Glue job — fails (no real job exists)
- Sends failure notification to `PipelineFailure` SNS topic
- Workflow ends

Check your email — you should receive an SNS failure notification within a minute.

---

## Part 6: Monitor and Troubleshoot

**Check execution history:**

- Click on the execution in the list
- See the visual workflow showing which steps ran and which failed
- Click on any failed step to see the exact error message and details

**Read logs:**

- Services → **CloudWatch** → **Log groups**
- Find the log group for your state machine
- Check for error messages and execution details

---

## Success Criteria — You're Done If:

- [ ] Step Functions state machine created
- [ ] All 3 Lambda functions deployed
- [ ] DynamoDB table created
- [ ] Both SNS topics created and email subscriptions confirmed
- [ ] State machine executed successfully (even if Glue step failed — that's expected)
- [ ] Received SNS notification email
- [ ] Can view execution history and see which steps ran
- [ ] Can explain how the workflow flows from step to step

---

## Part 7: Teardown

### Cost Summary for This Lab

| Resource | Cost |
|----------|------|
| Step Functions | ~$0.001 (essentially free) |
| Lambda | Covered by free tier |
| DynamoDB | Covered by free tier |
| SNS | Covered by free tier |
| **Total** | **Under $0.01** |

### Cleanup Steps

- **Step Functions** → select `DataPipelineOrchestration` → Delete → confirm
- **Lambda** → select each function → Delete → confirm (repeat for all 3)
- **DynamoDB** → select `PipelineExecutionLog` → Delete table → confirm
- **SNS** → select `PipelineSuccess` → Delete topic → confirm, repeat for `PipelineFailure`
- **IAM** → Roles → find `StepFunctionsExecutionRole` → Delete → confirm

**Time needed:** ~10 minutes. **Cost after teardown:** $0.00/month.

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **Orchestration concepts** | Why manual pipelines fail; what orchestration solves; real-world impact |
| **State machines** | How states, transitions, and choices work; explicit flow control |
| **Step Functions** | Creating state machines; JSON workflow definition; visual execution monitoring |
| **Lambda integration** | Chaining functions; passing data between steps; error handling |
| **Error handling** | Catch blocks; retry logic with backoff; failure routing |
| **Alerting** | SNS topic creation; email subscriptions; failure notifications |
| **Logging** | DynamoDB for execution history; CloudWatch for debug logs |
| **Cost optimization** | Serverless orchestration; free tier coverage; teardown hygiene |

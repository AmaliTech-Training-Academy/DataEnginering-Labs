# Lambda + DynamoDB Real-Time Analytics

---

## Part 0: Why Real-Time Analytics Matters

### The Real-Time Problem

User opens a mobile app at 3:00:15 PM and makes a $500 payment.

| | Traditional Batch Processing | Real-Time Processing (this lab) |
|--|-----------------------------|---------------------------------|
| **Event logged** | 3:00:15 PM | 3:00:15 PM |
| **Processing starts** | 3:00 AM next day (batch job) | 3:00:15.001 PM (Lambda triggers instantly) |
| **Analytics ready** | 4:00 AM next day | 3:00:15.050 PM |
| **Fraud detection** | Next morning | 3:00:15.100 PM |
| **Decision made on** | 13-hour-old data | Current data |
| **Result** | BAD — user already got wrong confirmation | GOOD — block or approve in real-time |

### Companies Using Real-Time Analytics

| Company | Use Case |
|---------|---------|
| **Amazon** | Product recommendations — instantly personalized |
| **Netflix** | Content suggestions — updated per view |
| **Uber** | Ride matching — real-time driver/passenger optimization |
| **DoorDash** | Delivery ETA — updates as drivers move |

### Real-World Use Cases

| Use Case | Flow | Outcome |
|----------|------|---------|
| **Fraud detection** | Transaction → Lambda → DynamoDB → pattern check | Block or approve in 100ms |
| **Dynamic pricing** | Item sold → Lambda → DynamoDB → website reads | Inventory-driven pricing in real-time |
| **Personalization** | User clicks → Lambda → DynamoDB counter++ → recommendations | Improves per interaction |
| **Compliance** | API called → Lambda → DynamoDB → dashboard | Operational visibility in real-time |

---

## Part 1: What You'll Learn

By the end of this lab, you'll understand:

- Event-driven architecture (Lambda triggers)
- NoSQL design (DynamoDB for fast queries)
- Serverless scaling (how Lambda handles millions of events)
- Real-time aggregations (counters, summaries)
- Cost optimization (DynamoDB billing model)
- Monitoring real-time systems (CloudWatch)
- Anomaly detection patterns

---

## Part 2: Architecture Overview

```
Event source (S3, API, IoT)
        ↓
EventBridge rule
        ↓
Lambda function (async)
  Parse event
  Update DynamoDB
  Check thresholds
  Trigger alerts
        ↓
DynamoDB (NoSQL)
  Real-time counters
  Recent events store
  Aggregations
        ↓
CloudWatch dashboard
  Metrics visualization
  Alerts and notifications
        ↓
S3 export (daily)
        ↓
Analytics and BI
```

### Lab Components

| Component | Tables / Functions | Purpose |
|-----------|-------------------|---------|
| **Lambda functions** | `ProcessTransaction`, `UpdateAggregations`, `CheckAnomalies` | Event processors — handle streaming data |
| **DynamoDB tables** | `TransactionEvents`, `HourlyAggregations`, `CustomerProfiles` | Real-time data store |
| **EventBridge rules** | Transaction events → Lambda | Route events to processors |
| **CloudWatch** | Invocations, latency, error rates | Real-time visibility |
| **SNS notifications** | `fraud-alerts` topic | Alerts for anomalies and errors |

---

## Part 3: Why This Architecture?

### Why Lambda Instead of EC2 or Glue?

| | EC2 (Always On) | Glue Job | Lambda |
|--|----------------|----------|--------|
| **Cost** | $10/month minimum | Per job (cluster startup) | $0 idle, $2 per million invocations |
| **Startup time** | Already running | 5–8 minutes | Milliseconds |
| **Idle waste** | 99% of the time | N/A | Zero — no idle cost |
| **Suitable for real-time?** | Yes but wasteful | No (8-minute latency) | Yes — ideal |

### Why DynamoDB Instead of Redshift or RDS?

| | Redshift | RDS/PostgreSQL | DynamoDB |
|--|---------|---------------|---------|
| **Query latency** | 100ms–seconds | 10–50ms | 1–10ms |
| **Optimized for** | Analytical queries | Transactions, joins | Key-value lookups, counters |
| **Scaling** | Vertical | Vertical | Horizontal (infinitely) |
| **Minimum cost** | $1/hour | $0.10/hour | $0.50/month (on-demand) |
| **Real-time at scale?** | No | Limited at 10K+ events/sec | Yes |
| **Trade-off** | | | No joins — must denormalize |

### Why EventBridge for Routing?

| | Direct Lambda Invocation | EventBridge |
|--|------------------------|------------|
| **Coupling** | Tight — change Lambda name breaks code | Loose — just update the rule |
| **Routing** | All-or-nothing | Multiple Lambdas can listen to same events |
| **Filtering** | None | Route by source, type, or field values |

---

## Part 4: Prerequisites

- S3 bucket from Tier 1.3
- IAM role from Tier 1.1 (`LambdaExecutionRole` with DynamoDB permissions)
- Lambda basics (Tier 5)
- Basic Python and JSON knowledge
- 5 hours uninterrupted

---

## Part 5: Lab Implementation

### Section A: DynamoDB Table Setup

#### Step A1: Create TransactionEvents Table

- AWS Console → **DynamoDB** → **Create table**
- Fill in:

| Field | Value |
|-------|-------|
| Table name | `TransactionEvents` |
| Partition key | `customer_id` (String) |
| Sort key | `event_timestamp` (String, format: `2024-03-15T14:30:45Z`) |
| Billing mode | On-demand (pay-per-request) |

> Sort key enables time-based queries (last hour, last 24 hours, etc.). On-demand is best here because real-time traffic is unpredictable — no forecasting needed.

- Click **Create table** — wait 1–2 minutes for `ACTIVE` status

#### Step A2: Create HourlyAggregations Table

| Field | Value |
|-------|-------|
| Table name | `HourlyAggregations` |
| Partition key | `hour_key` (String, format: `2024-03-15-14` = 2 PM on March 15) |
| Sort key | None |
| Billing mode | On-demand |

> **Purpose:** Pre-aggregated summaries so dashboards can read counts instantly without re-aggregating every time.

#### Step A3: Create CustomerProfiles Table

| Field | Value |
|-------|-------|
| Table name | `CustomerProfiles` |
| Partition key | `customer_id` (String) |
| Sort key | None |
| Billing mode | On-demand |

> **Purpose:** Real-time per-customer metrics for personalization and fraud detection.

**Expected:** 3 tables created and showing `ACTIVE` status.

---

### Section B: Lambda Function — Transaction Processor

#### Step B1: Create Lambda Function

- AWS Lambda → **Create function**
- Fill in:

| Field | Value |
|-------|-------|
| Function name | `ProcessTransaction` |
| Runtime | Python 3.11 |
| Execution role | Use existing: `LambdaExecutionRole` (from Tier 1.1) |

- Click **Create function**

#### Step B2: Write Lambda Code

Replace all default code with:

```python
import json
import boto3
import os
from datetime import datetime
from decimal import Decimal

# Initialize AWS clients
dynamodb    = boto3.resource('dynamodb')
cloudwatch  = boto3.client('cloudwatch')

# Table references (names from environment variables)
transaction_table  = dynamodb.Table(os.environ['TRANSACTION_TABLE'])
aggregation_table  = dynamodb.Table(os.environ['AGGREGATION_TABLE'])
customer_table     = dynamodb.Table(os.environ['CUSTOMER_TABLE'])

def lambda_handler(event, context):
    """
    Process a transaction event:
    1. Validate input
    2. Store raw event in DynamoDB
    3. Update hourly aggregations
    4. Update customer profile
    5. Check for anomalies
    6. Send CloudWatch metrics
    """
    try:
        print(f"Processing event: {json.dumps(event)}")

        # Extract transaction details
        customer_id = event.get('customer_id')
        amount      = Decimal(str(event.get('amount', 0)))
        merchant    = event.get('merchant', 'UNKNOWN')
        category    = event.get('category', 'OTHER')
        timestamp   = datetime.utcnow().isoformat() + 'Z'

        # Validation
        if not customer_id or amount <= 0:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'Invalid transaction'})
            }

        # Step 1: Store raw transaction
        transaction_item = {
            'customer_id':     customer_id,
            'event_timestamp': timestamp,
            'amount':          amount,
            'merchant':        merchant,
            'category':        category,
            'event_date':      timestamp[:10]   # for date-based queries
        }
        transaction_table.put_item(Item=transaction_item)
        print(f"Stored transaction: {transaction_item}")

        # Step 2: Update hourly aggregations
        hour_key = timestamp[:13] + ':00:00Z'   # round down to the hour
        update_hourly_aggregations(hour_key, amount, category)

        # Step 3: Update customer profile
        update_customer_profile(customer_id, amount, category)

        # Step 4: Check for anomalies
        is_anomaly = check_anomaly(customer_id, amount)

        # Step 5: Send CloudWatch metrics
        cloudwatch.put_metric_data(
            Namespace='TransactionProcessing',
            MetricData=[
                {
                    'MetricName': 'TransactionAmount',
                    'Value':      float(amount),
                    'Unit':       'Count'
                },
                {
                    'MetricName': 'TransactionsProcessed',
                    'Value':      1,
                    'Unit':       'Count'
                }
            ]
        )

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message':          'Transaction processed',
                'transaction_id':   f"{customer_id}_{timestamp}",
                'anomaly_detected': is_anomaly
            })
        }

    except Exception as e:
        print(f"Error processing transaction: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def update_hourly_aggregations(hour_key, amount, category):
    """Update hourly summaries — total amount, count, and breakdown by category."""
    try:
        aggregation_table.update_item(
            Key={'hour_key': hour_key},
            UpdateExpression="""
                SET total_amount       = if_not_exists(total_amount, :zero) + :amount,
                    transaction_count  = if_not_exists(transaction_count, :zero) + :one,
                    last_update        = :timestamp
                ADD category_counts.#cat :one
            """,
            ExpressionAttributeValues={
                ':amount':    amount,
                ':one':       Decimal('1'),
                ':zero':      Decimal('0'),
                ':timestamp': datetime.utcnow().isoformat() + 'Z'
            },
            ExpressionAttributeNames={
                '#cat': category
            }
        )
        print(f"Updated aggregations for {hour_key}")
    except Exception as e:
        print(f"Error updating aggregations: {str(e)}")

def update_customer_profile(customer_id, amount, category):
    """Update real-time customer metrics — spend, count, last transaction."""
    try:
        customer_table.update_item(
            Key={'customer_id': customer_id},
            UpdateExpression="""
                SET total_spend       = if_not_exists(total_spend, :zero) + :amount,
                    transaction_count = if_not_exists(transaction_count, :zero) + :one,
                    last_transaction  = :timestamp,
                    last_category     = :category
            """,
            ExpressionAttributeValues={
                ':amount':    amount,
                ':one':       Decimal('1'),
                ':zero':      Decimal('0'),
                ':timestamp': datetime.utcnow().isoformat() + 'Z',
                ':category':  category
            }
        )
        print(f"Updated profile for customer {customer_id}")
    except Exception as e:
        print(f"Error updating customer profile: {str(e)}")

def check_anomaly(customer_id, amount):
    """Simple threshold-based anomaly detection."""
    ANOMALY_THRESHOLD     = 10   # flag if > 10x the customer's average
    SUSPICIOUS_CATEGORIES = ['CRYPTO', 'GAMBLING']

    try:
        response = customer_table.get_item(Key={'customer_id': customer_id})

        if 'Item' not in response:
            # New customer — large transactions are suspicious
            return amount > Decimal('1000')

        customer        = response['Item']
        avg_transaction = (
            customer.get('total_spend', 0) /
            customer.get('transaction_count', 1)
        )

        if amount > avg_transaction * ANOMALY_THRESHOLD:
            return True
        if customer.get('last_category') in SUSPICIOUS_CATEGORIES:
            return True

        return False

    except Exception as e:
        print(f"Error checking anomaly: {str(e)}")
        return False   # fail safe — don't block on error
```

**Key code concepts explained:**

| Concept | Why It Matters |
|---------|---------------|
| `lambda_handler(event, context)` | Entry point — `event` is the transaction data, `context` is Lambda runtime info |
| `try/except` | All production Lambda must handle errors — returns proper HTTP status codes |
| `boto3` | AWS SDK for Python — used to talk to DynamoDB and CloudWatch |
| `UpdateExpression` with `if_not_exists` | Atomic upsert — creates field with default if missing, then increments. Much faster than read-then-write |
| `:parameters` | Parameterized values — prevents code injection, AWS best practice |

#### Step B3: Add Environment Variables

- Scroll to **Configuration** → **Environment variables** → **Edit**
- Add 3 variables:

| Key | Value |
|-----|-------|
| `TRANSACTION_TABLE` | `TransactionEvents` |
| `AGGREGATION_TABLE` | `HourlyAggregations` |
| `CUSTOMER_TABLE` | `CustomerProfiles` |

- Click **Save**

> Environment variables prevent hardcoded table names, make it easy to switch between dev and production, and keep configuration out of source code.

#### Step B4: Deploy and Test

- Click **Deploy**
- Click **Test** → **Create test event** → paste:

```json
{
  "customer_id": "CUST_12345",
  "amount": 50.00,
  "merchant": "Starbucks",
  "category": "DINING"
}
```

- Click **Test**

**Expected:** "Execution result: Succeeded." Response confirms transaction processed, no errors in logs.

> If you see errors: verify the IAM role has DynamoDB permissions and environment variables are set correctly.

---

### Section C: EventBridge Rule Setup

#### Step C1: Create EventBridge Rule

- Services → **EventBridge** → **Create rule**
- Fill in:

| Field | Value |
|-------|-------|
| Name | `route-transactions-to-lambda` |
| Event bus | Default |
| Rule type | Rule |

- Click **Next**

#### Step C2: Define Event Pattern

- Choose **Edit pattern** and paste:

```json
{
  "source": ["custom.transactions"],
  "detail-type": ["transaction_event"],
  "detail": {
    "customer_id": [{"exists": true}],
    "amount":      [{"exists": true}]
  }
}
```

> This matches only events from the `custom.transactions` source that have both `customer_id` and `amount` fields — everything else is ignored.

- Click **Next**

#### Step C3: Add Lambda Target

| Field | Value |
|-------|-------|
| Target type | AWS service |
| Service | Lambda function |
| Function | `ProcessTransaction` |
| Invocation type | Asynchronous (fire-and-forget) |

Click **Create rule**.

---

### Section D: Testing End-to-End

#### Step D1: Publish Test Events

- **EventBridge** → **Send events** (top right)
- Fill in:

| Field | Value |
|-------|-------|
| Source | `custom.transactions` |
| Detail type | `transaction_event` |
| Detail | See JSON below |

```json
{
  "customer_id": "CUST_USER_001",
  "amount": 75.50,
  "merchant": "Amazon",
  "category": "RETAIL"
}
```

- Click **Send** — repeat 5 times with different amounts to create data variety

#### Step D2: Check DynamoDB Tables

- **DynamoDB** → **TransactionEvents** → **Explore table items**
- Confirm transaction rows appear with recent timestamps

#### Step D3: Check Aggregations

- **HourlyAggregations** table → View items
- Confirm hourly summaries show correct totals

#### Step D4: Check Customer Profiles

- **CustomerProfiles** table → View items
- Confirm `CUST_USER_001` has updated spend and count

**Expected:** All 3 tables have data, timestamps are recent, aggregations show correct counts.

---

### Section E: CloudWatch Monitoring

#### Step E1: Create Custom Dashboard

- **CloudWatch** → **Dashboards** → **Create dashboard**
- Name: `Tier6-RealTimeAnalytics`

#### Step E2: Add Metrics

For each metric below: **Add widget** → **Line chart** → **Metrics** → select the metric → Stat: Sum, Period: 1 minute

| Namespace | Metric Name |
|-----------|------------|
| `TransactionProcessing` | `TransactionsProcessed` |
| `TransactionProcessing` | `TransactionAmount` |
| `AWS/Lambda` | `Duration` |
| `AWS/Lambda` | `Invocations` |

#### Step E3: Monitor Logs

- Lambda function → **Monitor** tab → **View logs in CloudWatch**
- Click any log stream to see execution details including processing time, DynamoDB operations, and any errors

---

### Section F: Anomaly Detection with SNS

#### Step F1: Create SNS Topic

- **SNS** → **Create topic** → Name: `fraud-alerts`, Type: Standard → **Create topic**
- Copy the topic ARN

#### Step F2: Add SNS Alerts to Lambda

Add this code inside `lambda_handler`, after `is_anomaly = check_anomaly(...)`:

```python
sns = boto3.client('sns')

if is_anomaly:
    sns.publish(
        TopicArn=os.environ['ALERT_TOPIC_ARN'],
        Subject='Anomalous Transaction Detected',
        Message=f"""
Customer:  {customer_id}
Amount:    ${amount}
Merchant:  {merchant}

This transaction exceeds normal patterns.
Review required.
        """,
        MessageStructure='text'
    )
```

Add a fourth environment variable:

| Key | Value |
|-----|-------|
| `ALERT_TOPIC_ARN` | Your SNS topic ARN (paste it here) |

Click **Deploy**.

#### Step F3: Subscribe to Alerts

- SNS topic → **Create subscription**
- Protocol: Email, Endpoint: your email address
- **Create subscription** → check email → confirm the link

You will now receive email alerts for anomalous transactions.

---

### Section G: Production Patterns

**Concurrency control** — prevent Lambda from overwhelming DynamoDB:

```
Lambda configuration → Reserved concurrency = 100
```

**Dead letter queue** — capture failed events for retry:

```
EventBridge rule → Add dead letter queue (SQS)
Failed events land in SQS for manual review or automatic retry
```

**Batching for cost optimization:**

```
Instead of processing 1 event at a time:
  Batch 100 events together
  Process in a single Lambda invocation
  Result: 100x cheaper per event
```

---

## Part 6: Cost Analysis

### This Lab

| Resource | Usage | Cost |
|----------|-------|------|
| Lambda | 1,000 invocations | $0.20 |
| DynamoDB | 100 KB written | $0.25 |
| CloudWatch | Logs and metrics | $1.00 |
| SNS | 10 notifications | $0.50 |
| **Total** | | **~$2** |

### At Production Scale (1 Million Events/Day)

| Resource | Monthly Cost |
|----------|-------------|
| Lambda (1M invocations) | $20 |
| DynamoDB (10 GB) | $5 |
| CloudWatch | $10 |
| Data export | $5 |
| **Total** | **$40/month** |

| Alternative | Monthly Cost |
|-------------|-------------|
| Redshift | $1,000 minimum |
| Kafka | $2,000 |
| Custom solution | $5,000+ (engineering time) |
| **This architecture** | **$40** |

---

## Part 7: Success Criteria

- [ ] 3 DynamoDB tables created and `ACTIVE`
- [ ] Lambda function deployed
- [ ] Test event succeeded — no errors in logs
- [ ] Data appears in all 3 DynamoDB tables
- [ ] Hourly aggregations calculated correctly
- [ ] Customer profile updated after events
- [ ] CloudWatch dashboard shows metrics
- [ ] SNS alert email received and confirmed
- [ ] Logs visible in CloudWatch
- [ ] Can explain event-driven architecture
- [ ] Can explain Lambda cold starts
- [ ] Can explain DynamoDB write patterns

**If all items are checked, you have completed Lab 6.2!**

---

## Part 8: Teardown

- **DynamoDB** → select all 3 tables → **Delete**
- **Lambda** → select `ProcessTransaction` → **Delete**
- **SNS** → select `fraud-alerts` → **Delete topic**
- **EventBridge** → select `route-transactions-to-lambda` → **Delete rule**

> **Cost after cleanup:** $0. Typical teardown time: 5 minutes.

---

## Part 9: What You Learned

| Area | Skills Gained |
|------|--------------|
| **Event-driven architecture** | Lambda + EventBridge decoupling; async invocation; fire-and-forget patterns |
| **NoSQL design** | Counters with atomic updates; aggregation tables; denormalization for speed |
| **Real-time processing** | Sub-100ms latency; instant event handling vs. batch jobs |
| **DynamoDB optimization** | `UpdateExpression` with `if_not_exists`; on-demand billing; partition key design |
| **Lambda at scale** | Concurrent invocations; cold starts; reserved concurrency |
| **Production monitoring** | CloudWatch dashboards; log streams; custom metric namespaces |
| **Anomaly detection** | Threshold-based patterns; customer baseline comparison; fail-safe design |
| **Cost optimization** | Serverless vs. traditional; batching; 50–100x savings over alternatives |

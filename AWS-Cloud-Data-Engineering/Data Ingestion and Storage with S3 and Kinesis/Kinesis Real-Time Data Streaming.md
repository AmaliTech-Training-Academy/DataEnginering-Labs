# Kinesis Real-Time Data Streaming

---

## PART 0: Understanding Why Real-Time Data Matters

### The Big Picture Story

Imagine you're Netflix and you need to know what users want — right now.

**The Problem (Without Real-Time Data):**

- User clicks "Play" on a show at 8:00 PM
- Data is recorded and processed in a nightly batch at 2:00 AM
- Recommendations update at 6:00 AM the next morning
- User sees updated recommendations 10+ hours later — after they've already left!

**The Cost of Delay:**

- Cost per lost user: $200/year subscriber value
- Loss per user per day due to delay: $0.55
- Netflix with 250M users: $137.5M/day in lost opportunity

**The Solution (With Real-Time Data):**

- User clicks "Play" at 8:00 PM
- Data streams instantly to Kinesis
- Within 100ms: recommendation engine updates
- Within 500ms: next video is queued and showing
- User stays engaged, watches more, revenue increases

### Real-Time Data Use Cases

| Company | What They Stream | Revenue Impact |
|---------|-----------------|----------------|
| Netflix | User clicks, pauses, seeks — 1000s of events/sec | Recommendations worth $1000s/second |
| Amazon | Cart updates, product views, checkout fraud detection | Prevents fraud worth millions/day |
| Uber | Driver locations, ride requests, trip analytics | Dispatch efficiency worth millions |
| Financial Services | Stock ticks, trading signals, risk alerts | Billions in trading profits |

### Batch vs. Real-Time: The Fundamental Difference

| | Batch Processing (Labs 2.1–2.2) | Real-Time Streaming (Lab 2.3) |
|--|--------------------------------|-------------------------------|
| **Flow** | Data collected, processed nightly, available next morning | Data streamed instantly, processed and used immediately |
| **Use case** | Historical analysis, reporting, long-term trends | Live decisions, fraud detection, instant reactions |
| **Example question** | "How many customers signed up last month?" | "User just logged in — is it fraud?" |
| **Latency** | Hours to days | Milliseconds |

**You need BOTH — not one or the other:**

- **Batch:** "What were the top 10 shows last month?" (reporting)
- **Streaming:** "Recommend the next show in 100ms" (engagement)
- **Combined:** Smart recommendations plus historical insights

### Why This Matters for Your Career

- Real-time engineers earn **$150k–$180k** — streaming is the hottest skill in data
- Streaming engineers drive current decisions; batch engineers analyze past data — streaming has higher status
- Powers the fastest-growing products: real-time recommendations, live feeds, instant fraud detection, autonomous driving
- Enables AI/ML at scale — models need fresh data, and streaming delivers it instantly

---

## PART 1: Goals for This Lab

By the end of this lab, you will:

- Understand real-time streaming architectures
- Create a Kinesis Data Stream (high-throughput)
- Build a Python producer (generates events)
- Build a Python consumer (processes events)
- Integrate with S3 via Firehose (durable storage)
- Monitor stream metrics and performance

---

## PART 2: What You'll Create

**Event Producer (Python)**
- Generates simulated user action events
- Sends to Kinesis Data Stream
- Rate: 10–100 events/second

**Kinesis Data Stream**
- 4 shards, each handling 1,000 events/second
- Total capacity: 4,000 events/second

**Event Consumer (Python)**
- Reads from Kinesis
- Processes events (aggregates, filters)
- Outputs to CloudWatch and terminal

**Kinesis Firehose**
- Captures stream data automatically
- Buffers into batches and writes to S3
- Location: `s3://bucket/streaming-data/`

**CloudWatch Dashboard**
- `IncomingRecords` (events/second)
- `GetRecords.IteratorAgeMilliseconds` (latency)
- `ReadProvisionedThroughputExceeded` (scaling alert)
- Custom metrics from consumer

### Data Flow

```
USER ACTIONS (Simulated)
  User clicks Play, pauses, seeks, closes app
         |
         v
KINESIS DATA STREAM (temporary buffer, 24-hour retention)
  Guarantees ordering per shard | 4,000 events/sec capacity
         |                    |
         v                    v
REAL-TIME PROCESSING     FIREHOSE (auto-saves to S3)
  Aggregate metrics         S3 backup
  Detect anomalies          Available within 1 minute
  Trigger alerts            Queryable with Athena
  Update recommendations
         |
         v
INSTANT ACTIONS
  Update UI in 100ms
  Trigger notifications
  Prevent fraud
```

---

## PART 3: The "Why" Behind Design Decisions

### Why Kinesis Over Other Streaming Tools?

| Tool | Pros | Cons | Best For |
|------|------|------|----------|
| **RabbitMQ** | Simple for small scale | Doesn't scale, no AWS integration | Legacy systems, microservices |
| **Kafka** | Highly scalable, battle-tested | Complex to operate, expensive infrastructure | Large enterprises with a DevOps team |
| **Kinesis (AWS)** | Managed, auto-scales, integrates everything | Costs more than DIY, vendor lock-in | Companies wanting simplicity over control |

Netflix, Amazon, and Uber all use Kinesis or Kafka.

### Why Shards?

**Single Shard = Bottleneck**
- Capacity: 1,000 events/second
- If 1,001 events arrive: throttled, some dropped or delayed

**Multiple Shards = Parallel Processing**
- 4 shards = 4,000 events/second total
- Netflix runs 1,000+ shards at ~$100k/month (vs. $500k+/month on Kafka self-hosted)

### Why Firehose for Automatic Backup?

**Without Firehose:** Consumer must write to S3 manually — if consumer fails, the S3 write fails and data is lost.

**With Firehose:** S3 backup happens automatically in the background. Consumer failure does not affect data storage. Cost: $0.03 per GB.

### Why Python Producer/Consumer?

- Easy to understand and modify
- Uses `boto3` (standard AWS SDK)
- Real-world companies use this pattern in production
- Scales easily from testing to production

---

## PART 4: Prerequisites — Check These First

- Completed Labs 1.1–2.2 (IAM, VPC, S3, DataSync)
- Python 3.8+ installed
- `boto3` library (install below)
- Text editor or IDE (VS Code recommended)
- AWS account (same as previous labs)
- IAM access to create Kinesis resources
- 4 hours uninterrupted
- Basic Python knowledge (loops, functions, dicts)

**Don't have Python?**
- Windows: Download from [python.org](https://python.org)
- Mac: `brew install python3`
- Linux: `apt-get install python3`

---

## STEP 0: Login and Prepare

### Step 0.1: Login to AWS

- Open [https://console.aws.amazon.com](https://console.aws.amazon.com)
- Enter credentials and MFA code if prompted
- Verify account name shows in the top right

### Step 0.2: Verify Region

- Top right, confirm region is `us-east-1`
- If different, switch to `us-east-1`

### Step 0.3: Set Up Python Environment

Open your terminal and run:

```bash
mkdir kinesis-lab
cd kinesis-lab
python3 -m venv venv
```

Activate the virtual environment:

```bash
# Mac/Linux:
source venv/bin/activate

# Windows:
venv\Scripts\activate
```

Install dependencies:

```bash
pip install boto3
pip install python-dateutil
```

### Step 0.4: Configure AWS Credentials

**Option A — AWS CLI (Recommended):**

```bash
aws configure
```

Enter your Access Key ID, Secret Access Key, region (`us-east-1`), and output format (`json`).

**Option B — Credentials file (not recommended for production):**

```ini
# ~/.aws/credentials
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
```

### Step 0.5: Prepare Documentation File

Create `Lab_2_3_Kinesis.txt` with this template:

```
=== LAB 2.3: KINESIS REAL-TIME STREAMING ===
Date Created: [TODAY]

KINESIS STREAM:
  Stream Name:  [TO BE FILLED]
  Shard Count:  [TO BE FILLED]
  Throughput:   [TO BE FILLED]

FIREHOSE:
  Delivery Stream Name: [TO BE FILLED]
  Destination:          S3
  Destination Path:     [TO BE FILLED]

PYTHON CODE:
  Producer File:    kinesis_producer.py
  Consumer File:    kinesis_consumer.py
  Events Generated: [TO BE FILLED]
  Events Processed: [TO BE FILLED]
```

---

## PART 1: Create Kinesis Data Stream

### Step 1.1: Navigate to Kinesis

- Click Services, search **kinesis**, click **Kinesis**
- You should see **Data Streams** and **Delivery Streams** tabs

### Steps 1.2 & 1.3: Create Data Stream

- Click **Create data stream**
- Fill in the fields:

| Field | Value |
|-------|-------|
| Stream name | `user-events-stream` |
| Capacity mode | Provisioned (gives us control over shards) |
| Number of shards | 4 (= 4,000 events/second capacity) |

- Click **Create data stream**

**Expected:** Status shows `CREATING`, then `ACTIVE` after 1–2 minutes.

**Save:** Stream Name, Shard Count (4), Throughput (4,000 events/second), Status (ACTIVE)

### Step 1.4: Verify Stream

Click on `user-events-stream` and confirm:

- Status: **ACTIVE**
- Number of Shards: **4**
- Provisioned Throughput: **4,000 events/second**

---

## PART 2: Create Kinesis Firehose (S3 Backup)

### Steps 2.1 & 2.2: Create Delivery Stream

- Click **Delivery Streams**, then **Create delivery stream**
- Fill in the fields:

| Field | Value |
|-------|-------|
| Delivery stream name | `user-events-to-s3` |
| Source | Kinesis Data Stream: `user-events-stream` |
| Destination | Amazon S3 |
| S3 bucket | `data-lake-prod-123456789` |
| S3 prefix | `streaming-data/` |
| Buffer size | 5 MB |
| Buffer time | 300 seconds (5 minutes) |

- Click **Create delivery stream**

**Expected:** Status shows `ACTIVE`.

**Save:** Delivery Stream Name, Source, Destination, Prefix

---

## PART 3: Create Python Producer

### Step 3.1: Create `kinesis_producer.py`

```python
import boto3
import json
import time
from datetime import datetime
import random

# Initialize Kinesis client
kinesis = boto3.client('kinesis', region_name='us-east-1')

STREAM_NAME = 'user-events-stream'

EVENTS = ['play', 'pause', 'resume', 'seek_forward', 'seek_backward',
          'stop', 'like', 'dislike', 'share']

USER_IDS = ['user_001', 'user_002', 'user_003', 'user_004', 'user_005']

CONTENT = ['Stranger Things', 'The Crown', 'Bridgerton',
           'The Witcher', 'Black Mirror']

def generate_event():
    """Generate a random user event"""
    return {
        'timestamp': datetime.now().isoformat(),
        'user_id': random.choice(USER_IDS),
        'event_type': random.choice(EVENTS),
        'content': random.choice(CONTENT),
        'duration_seconds': random.randint(1, 3600),
        'device': random.choice(['web', 'mobile', 'tv']),
        'region': random.choice(['us-east', 'us-west', 'eu-west', 'ap-southeast'])
    }

def send_to_kinesis(event):
    """Send event to Kinesis stream"""
    try:
        response = kinesis.put_record(
            StreamName=STREAM_NAME,
            Data=json.dumps(event),
            PartitionKey=event['user_id']  # Ensures same user goes to same shard
        )
        return True
    except Exception as e:
        print(f"Error sending to Kinesis: {e}")
        return False

def main():
    """Main producer loop"""
    print(f"Starting producer... Sending to stream: {STREAM_NAME}")
    print("Press Ctrl+C to stop\n")

    event_count = 0
    start_time = time.time()

    try:
        while True:
            event = generate_event()
            if send_to_kinesis(event):
                event_count += 1
                if event_count % 10 == 0:
                    elapsed = time.time() - start_time
                    rate = event_count / elapsed
                    print(f"[{event_count}] Sent: {event['event_type']} "
                          f"by {event['user_id']} "
                          f"({rate:.1f} events/sec)")
            time.sleep(0.2)  # 5 events per second

    except KeyboardInterrupt:
        print(f"\n\nStopped. Total events sent: {event_count}")
        elapsed = time.time() - start_time
        print(f"Total time: {elapsed:.1f} seconds")
        print(f"Average rate: {event_count/elapsed:.1f} events/second")

if __name__ == '__main__':
    main()
```

### Step 3.2: Test Producer

```bash
python kinesis_producer.py
```

Expected output:

```
Starting producer... Sending to stream: user-events-stream
Press Ctrl+C to stop

[10] Sent: play by user_001 (5.1 events/sec)
[20] Sent: pause by user_003 (5.2 events/sec)
[30] Sent: resume by user_002 (5.1 events/sec)
```

Let it run for 30 seconds (~150 events), then press `Ctrl+C` to stop.

**Save:** Producer File: `kinesis_producer.py`, Events Generated: ~150, Status: Working

---

## PART 4: Create Python Consumer

### Step 4.1: Create `kinesis_consumer.py`

```python
import boto3
import json
import time
from datetime import datetime

kinesis = boto3.client('kinesis', region_name='us-east-1')
STREAM_NAME = 'user-events-stream'

class KinesisConsumer:
    def __init__(self, stream_name):
        self.stream_name = stream_name
        self.kinesis = boto3.client('kinesis', region_name='us-east-1')
        self.shard_iterators = {}
        self.event_counts = {}
        self.last_event_time = time.time()
        self.initialize_shards()

    def initialize_shards(self):
        """Get shard iterators for all shards"""
        response = self.kinesis.describe_stream(StreamName=self.stream_name)
        shards = response['StreamDescription']['Shards']

        print(f"Found {len(shards)} shards:")
        for shard in shards:
            shard_id = shard['ShardId']
            iterator_response = self.kinesis.get_shard_iterator(
                StreamName=self.stream_name,
                ShardId=shard_id,
                ShardIteratorType='LATEST'
            )
            self.shard_iterators[shard_id] = iterator_response['ShardIterator']
            self.event_counts[shard_id] = 0
            print(f"  - {shard_id}")
        print()

    def process_event(self, event_data):
        """Process a single event"""
        try:
            event = json.loads(event_data)
            user_id = event.get('user_id', 'unknown')
            if user_id not in self.event_counts:
                self.event_counts[user_id] = 0
            self.event_counts[user_id] += 1
            return event
        except json.JSONDecodeError:
            return None

    def consume(self):
        """Consume events from Kinesis"""
        print(f"Starting consumer... Reading from: {self.stream_name}")
        print("Press Ctrl+C to stop\n")

        total_events = 0

        try:
            while True:
                for shard_id, iterator in self.shard_iterators.items():
                    if not iterator:
                        continue
                    try:
                        response = self.kinesis.get_records(
                            ShardIterator=iterator,
                            Limit=100
                        )
                        self.shard_iterators[shard_id] = response['NextShardIterator']

                        for record in response['Records']:
                            event = self.process_event(record['Data'])
                            if event:
                                total_events += 1
                                if total_events % 20 == 0:
                                    print(f"\n[{total_events}] Events Processed")
                                    print(f"Event type: {event['event_type']}")
                                    print(f"User:       {event['user_id']}")
                                    print(f"Content:    {event['content']}")
                                    print(f"Device:     {event['device']}")
                                    print("-" * 50)

                    except Exception as e:
                        print(f"Error reading from {shard_id}: {e}")

                time.sleep(1)

        except KeyboardInterrupt:
            print(f"\n\nStopped. Total events processed: {total_events}")
            print("\nEvents by user:")
            for user_id, count in self.event_counts.items():
                if count > 0:
                    print(f"  {user_id}: {count} events")

def main():
    consumer = KinesisConsumer(STREAM_NAME)
    consumer.consume()

if __name__ == '__main__':
    main()
```

### Step 4.2: Test Consumer (While Producer Runs)

Open a second terminal, activate your virtual environment, then run:

```bash
python kinesis_consumer.py
```

Expected output:

```
Found 4 shards:
  - shardId-000000000000
  - shardId-000000000001
  - shardId-000000000002
  - shardId-000000000003

Starting consumer... Reading from: user-events-stream
Press Ctrl+C to stop

[20] Events Processed
Event type: play
User:       user_001
Content:    Stranger Things
Device:     mobile
--------------------------------------------------
```

**Save:** Consumer File: `kinesis_consumer.py`, Events Processed: [number from output], Status: Working

---

## PART 5: Monitor with CloudWatch

### Steps 5.1 & 5.2: View Stream Metrics

- Go to CloudWatch console, click **Metrics**
- Search **Kinesis**, select **Stream Metrics**, find `user-events-stream`
- Key metrics to watch:

| Metric | What It Shows |
|--------|---------------|
| `IncomingRecords` | Events per second |
| `IncomingBytes` | Data flow rate |
| `GetRecords.IteratorAgeMilliseconds` | Latency |
| `ReadProvisionedThroughputExceeded` | Throttling alerts |

### Step 5.3: Create CloudWatch Dashboard

- Click **Dashboards**, then **Create dashboard**
- Name it `kinesis-monitoring`
- Add widgets for: IncomingRecords (graph), IncomingBytes (graph), IteratorAge (gauge)

---

## PART 6: Verify Data in S3

### Step 6.1: Check S3

- Go to S3, navigate to `data-lake-prod-123456789/streaming-data/`
- You should see folders structured as `YYYY/MM/DD/HH/`
- Files named like `[timestamp]-[random].gz` (compressed JSON)

### Step 6.2: Inspect a File

Download one file, decompress it, and confirm contents look like:

```json
{"timestamp": "2024-01-20T15:30:45.123", "user_id": "user_001", "event_type": "play", ...}
{"timestamp": "2024-01-20T15:30:46.456", "user_id": "user_002", "event_type": "pause", ...}
```

Verify: JSON format, correct timestamps, data integrity preserved.

---

## PART 7: Document Your Streaming Setup

```
=== LAB 2.3: KINESIS REAL-TIME STREAMING COMPLETE ===

KINESIS DATA STREAM:
  Name:           user-events-stream
  Shards:         4
  Throughput:     4,000 events/second
  Status:         ACTIVE
  Data retention: 24 hours

KINESIS FIREHOSE:
  Name:        user-events-to-s3
  Source:      user-events-stream
  Destination: S3 (data-lake-prod-xxx/streaming-data/)
  Buffer size: 5 MB | Buffer time: 300 seconds
  Status:      ACTIVE

PYTHON PRODUCER:
  File:          kinesis_producer.py
  Events sent:   ~150 (30 seconds at 5 events/sec)
  Partition key: user_id (maintains order per user)

PYTHON CONSUMER:
  File:          kinesis_consumer.py
  Events read:   ~150
  Shards read:   4 (parallel)
  Poll interval: 1 second

CLOUDWATCH METRICS:
  IncomingRecords:                    ~5 events/sec
  IncomingBytes:                      ~500 bytes/sec
  IteratorAge:                        <1 second
  ReadProvisionedThroughputExceeded:  0 (no throttling)
  Dashboard:                          kinesis-monitoring

S3 BACKUP (via Firehose):
  Location:  streaming-data/
  Format:    Compressed JSON (gzip)
  Structure: YYYY/MM/DD/HH/[timestamp].gz

COST ESTIMATE:
  Kinesis shards: 4 x $0.36/hr = $1.44/hr
  Firehose:       ~$0.15/hr
  S3 storage:     <$0.01
  Total:          ~$1.60/hr | $38/day | $1,140/month
  NOTE: Delete resources when not testing!
```

---

## Success Criteria — Verify You're Done

- [ ] Kinesis Data Stream created (4 shards)
- [ ] Kinesis Firehose created
- [ ] Firehose connected to S3
- [ ] Python producer script created
- [ ] Producer sends events to Kinesis
- [ ] Python consumer script created
- [ ] Consumer reads events from Kinesis
- [ ] Producer and consumer run simultaneously
- [ ] 150+ events generated
- [ ] 150+ events processed
- [ ] CloudWatch metrics visible
- [ ] CloudWatch dashboard created
- [ ] S3 has streaming data files
- [ ] S3 files contain valid JSON
- [ ] Documentation saved with all details

**If all items are checked, you have completed Lab 2.3!**

---

## Teardown — Save $1,140/Month

Kinesis is expensive — delete resources when done testing.

### Step 8.1: Delete Kinesis Stream

- Go to Kinesis console, click on `user-events-stream`
- Click **Delete stream** and confirm
- Cost saved: ~$346/month

### Step 8.2: Delete Firehose

- Click on `user-events-to-s3`, click **Delete delivery stream** and confirm
- Cost saved: ~$50/month

### Step 8.3: Keep Your Python Scripts

- Keep `kinesis_producer.py` and `kinesis_consumer.py`
- You can rerun them with any new stream anytime

### Step 8.4: Verify in Billing Console

- Go to **Billing**, check **Costs by service**
- Kinesis should show $0 after deletion

---

## What You Learned

| Area | Skills Gained |
|------|--------------|
| **Real-Time Streaming Concepts** | Event-driven architecture; streaming vs. batch; sharding and parallelism |
| **Kinesis Components** | Data Streams (buffering); Firehose (automatic S3 backup); Shards (parallel processing) |
| **Python Integration** | boto3 AWS SDK; producer pattern; consumer pattern |
| **Real-Time Patterns** | High-throughput ingestion; low-latency processing; automatic backups |
| **Monitoring and Operations** | CloudWatch metrics; stream health monitoring; performance optimization |

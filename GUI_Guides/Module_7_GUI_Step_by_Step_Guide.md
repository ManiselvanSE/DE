# MODULE 7: Analytics - GUI Step-by-Step Hands-On Guide

## Table of Contents
- [Exercise 7.1: Serverless Data Lake Analytics with Athena](#exercise-71-serverless-data-lake-analytics-with-athena)
- [Exercise 7.2: Real-Time Streaming Analytics with Kinesis](#exercise-72-real-time-streaming-analytics-with-kinesis)
- [Exercise 7.3: Business Intelligence with QuickSight](#exercise-73-business-intelligence-with-quicksight)

---

# Exercise 7.1: Serverless Data Lake Analytics with Athena

**Duration:** 2-3 hours | **Estimated Cost:** ~$5-10 for lab session

## 🎯 Learning Objectives
- Organize data lake with Hive-style partitioning
- Auto-discover schema with AWS Glue Crawler
- Query petabyte-scale data with Amazon Athena
- Optimize queries with partitioning (99.99% cost reduction)
- Create reusable views and result caching

---

## PART 1: Prepare Partitioned Data Lake

### Step 1: Create S3 Bucket for Data Lake
1. **S3 Console** → **"Create bucket"**
2. **Bucket name:** `clickstream-datalake-[your-account-id]`
3. **Region:** us-east-1
4. **Versioning:** Disabled (for cost savings)
5. **Encryption:** SSE-S3 (default)
6. Click **"Create bucket"**

### Step 2: Create Folder Structure with Partitions
1. Click on your bucket: `clickstream-datalake-[id]`
2. Create this Hive-style partitioned structure:
   - Click **"Create folder"** → `raw/` → Create
   - Click **"Create folder"** → `processed/` → Create
   - Inside `processed/`, create: `clickstream/`
   - Inside `clickstream/`, create partitions:
     - `year=2024/month=06/day=01/`
     - `year=2024/month=06/day=02/`
     - `year=2024/month=06/day=03/`

**Why this structure?**
- Hive-style partitioning: `key=value/` format
- Allows partition pruning in queries
- Reduces scan costs by 99%+

### Step 3: Generate Sample Clickstream Data
Create script locally: `generate_clickstream.py`

```python
import pandas as pd
import pyarrow.parquet as pq
import pyarrow as pa
from datetime import datetime, timedelta
import random

# Generate clickstream events
def generate_events(date, num_events=10000):
    events = []
    for i in range(num_events):
        event = {
            'event_id': f'{date.strftime("%Y%m%d")}{i:06d}',
            'user_id': f'user_{random.randint(1, 5000)}',
            'session_id': f'session_{random.randint(1, 10000)}',
            'event_timestamp': (datetime.combine(date, datetime.min.time()) + 
                              timedelta(seconds=random.randint(0, 86400))).isoformat(),
            'page_url': random.choice(['/home', '/products', '/cart', '/checkout', '/account']),
            'event_type': random.choice(['page_view', 'click', 'scroll', 'form_submit', 'purchase']),
            'device_type': random.choice(['mobile', 'desktop', 'tablet']),
            'browser': random.choice(['Chrome', 'Safari', 'Firefox', 'Edge']),
            'country': random.choice(['US', 'UK', 'DE', 'FR', 'JP', 'IN', 'BR', 'AU']),
            'referrer': random.choice(['google.com', 'facebook.com', 'twitter.com', 'direct', 'email']),
            'duration_seconds': random.randint(1, 600)
        }
        events.append(event)
    return events

# Generate data for 3 days
dates = [
    datetime(2024, 6, 1),
    datetime(2024, 6, 2),
    datetime(2024, 6, 3)
]

for date in dates:
    events = generate_events(date, 10000)
    df = pd.DataFrame(events)
    
    # Write Parquet
    table = pa.Table.from_pandas(df)
    year = date.year
    month = date.month
    day = date.day
    
    filename = f'clickstream_year={year}_month={month:02d}_day={day:02d}.parquet'
    pq.write_table(table, filename, compression='snappy')
    
    print(f'✓ Generated {filename} with {len(df):,} events')

print('\n✓ Generated 3 days of clickstream data (30,000 total events)')
```

Run it:
```bash
pip install pandas pyarrow
python3 generate_clickstream.py
```

### Step 4: Upload Partitioned Data to S3
```bash
# Upload to correct partition paths
aws s3 cp clickstream_year=2024_month=06_day=01.parquet \
  s3://clickstream-datalake-[id]/processed/clickstream/year=2024/month=06/day=01/data.parquet

aws s3 cp clickstream_year=2024_month=06_day=02.parquet \
  s3://clickstream-datalake-[id]/processed/clickstream/year=2024/month=06/day=02/data.parquet

aws s3 cp clickstream_year=2024_month=06_day=03.parquet \
  s3://clickstream-datalake-[id]/processed/clickstream/year=2024/month=06/day=03/data.parquet

# Verify structure
aws s3 ls s3://clickstream-datalake-[id]/processed/clickstream/ --recursive
```

You should see:
```
processed/clickstream/year=2024/month=06/day=01/data.parquet
processed/clickstream/year=2024/month=06/day=02/data.parquet
processed/clickstream/year=2024/month=06/day=03/data.parquet
```

---

## PART 2: Auto-Discover Schema with AWS Glue Crawler

### Step 5: Create Glue Database
1. **AWS Glue Console** → **"Data catalog"** → **"Databases"**
2. Click **"Add database"**
3. **Name:** `clickstream_db`
4. **Description:** `Clickstream analytics data lake`
5. **Location:** Leave empty (not required)
6. Click **"Create database"**

### Step 6: Create IAM Role for Glue Crawler
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** **"AWS service"**
3. **Use case:** Select **"Glue"**
4. Click **"Next"**
5. Attach policies:
   - ✅ **AWSGlueServiceRole** (auto-selected)
   - ✅ **AmazonS3ReadOnlyAccess**
6. Click **"Next"**
7. **Role name:** `AWSGlueServiceRole-Crawler`
8. Click **"Create role"**

### Step 7: Create Glue Crawler
1. **Glue Console** → **"Crawlers"** (left sidebar)
2. Click **"Create crawler"**
3. **Name:** `clickstream-crawler`
4. Click **"Next"**

### Step 8: Configure Crawler Data Source
1. **Is your data already mapped to Glue tables?** Select **"Not yet"**
2. Click **"Add a data source"**
3. **Data source:** **"S3"**
4. **S3 path:** Click **"Browse"**
   - Navigate to: `s3://clickstream-datalake-[id]/processed/clickstream/`
   - Select the `clickstream` folder
5. **Subsequent crawler runs:** Select **"Crawl all sub-folders"**
6. Click **"Add an S3 data source"**
7. Click **"Next"**

### Step 9: Configure IAM Role
1. **Existing IAM role:** Select `AWSGlueServiceRole-Crawler`
2. Click **"Next"**

### Step 10: Set Output Database
1. **Target database:** Select `clickstream_db`
2. **Table name prefix:** Leave empty OR type: `tbl_`
3. **Crawler schedule:** Select **"On demand"** (for now)
   - For production: **"Daily at 3:00 AM UTC"**
   - Cron: `cron(0 3 * * ? *)`
4. Click **"Next"**

### Step 11: Review and Create
1. Review all settings
2. Click **"Create crawler"**

### Step 12: Run Crawler
1. Select your crawler: `clickstream-crawler`
2. Click **"Run"** button
3. **Status** will change:
   - **Starting** → ~30 seconds
   - **Running** → ~1-2 minutes
   - **Stopping** → ~30 seconds
   - **Ready** → Completed

### Step 13: View Discovered Table
1. Go to **"Data catalog"** → **"Tables"**
2. You should see a new table: `clickstream` (or `tbl_clickstream`)
3. Click on the table name
4. **Schema** tab shows:
   - event_id (string)
   - user_id (string)
   - session_id (string)
   - event_timestamp (string)
   - page_url (string)
   - event_type (string)
   - device_type (string)
   - browser (string)
   - country (string)
   - referrer (string)
   - duration_seconds (bigint)
   - **Partition keys:** year, month, day (all bigint)

**🎉 Glue auto-discovered your schema and partitions!**

---

## PART 3: Query Data with Amazon Athena

### Step 14: Navigate to Athena Console
1. AWS Console → Search **"Athena"**
2. Click **"Amazon Athena"**
3. If first time, you'll see a setup page

### Step 15: Configure Query Result Location
1. Click **"Settings"** (top right) or **"Manage"**
2. **Query result location:** Click **"Browse S3"**
3. Navigate to your bucket: `clickstream-datalake-[id]`
4. Create folder: `athena-results/`
5. Select `athena-results/` folder
6. Click **"Choose"**
7. Click **"Save"**

### Step 16: Select Database
1. In the **Query Editor**, find **"Database"** dropdown (left panel)
2. Select `clickstream_db`
3. You should see your table: `clickstream` in the **Tables** list

### Step 17: Preview Table Data
1. Click the **three dots (•••)** next to `clickstream` table
2. Select **"Preview table"**
3. Athena auto-generates and runs:
   ```sql
   SELECT * FROM "clickstream_db"."clickstream" LIMIT 10;
   ```
4. You'll see 10 sample rows in the **Results** pane

### Step 18: Query Daily Active Users (With Partition Pruning)
Create a new query:

```sql
-- Query with partition filter (FAST - scans only 1 day)
SELECT 
    year,
    month,
    day,
    COUNT(DISTINCT user_id) as daily_active_users,
    COUNT(*) as total_events
FROM clickstream
WHERE year = 2024 
  AND month = 6 
  AND day = 1
GROUP BY year, month, day;
```

**Click "Run"**

**Results:** You should see:
| year | month | day | daily_active_users | total_events |
|------|-------|-----|-------------------|--------------|
| 2024 | 6 | 1 | ~4,500 | 10,000 |

**Data scanned:** ~100 KB (only day=1 partition)

### Step 19: Query Page Views by Hour
```sql
-- Extract hour from timestamp and aggregate
SELECT 
    CAST(year AS varchar) || '-' || 
    LPAD(CAST(month AS varchar), 2, '0') || '-' || 
    LPAD(CAST(day AS varchar), 2, '0') as date,
    HOUR(CAST(event_timestamp AS timestamp)) as hour,
    page_url,
    COUNT(*) as page_views,
    COUNT(DISTINCT user_id) as unique_users
FROM clickstream
WHERE year = 2024 AND month = 6 AND day = 2
GROUP BY year, month, day, HOUR(CAST(event_timestamp AS timestamp)), page_url
ORDER BY date, hour, page_views DESC
LIMIT 50;
```

**Click "Run"**

**Results:** Shows hourly traffic patterns per page

### Step 20: Conversion Funnel Analysis
```sql
-- Calculate conversion funnel
WITH funnel_steps AS (
    SELECT 
        user_id,
        session_id,
        MAX(CASE WHEN page_url = '/home' THEN 1 ELSE 0 END) as visited_home,
        MAX(CASE WHEN page_url = '/products' THEN 1 ELSE 0 END) as visited_products,
        MAX(CASE WHEN page_url = '/cart' THEN 1 ELSE 0 END) as added_to_cart,
        MAX(CASE WHEN page_url = '/checkout' THEN 1 ELSE 0 END) as started_checkout,
        MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as completed_purchase
    FROM clickstream
    WHERE year = 2024 AND month = 6
    GROUP BY user_id, session_id
)
SELECT 
    COUNT(*) as total_sessions,
    SUM(visited_home) as home_visits,
    SUM(visited_products) as product_views,
    SUM(added_to_cart) as cart_adds,
    SUM(started_checkout) as checkouts,
    SUM(completed_purchase) as purchases,
    ROUND(SUM(completed_purchase) * 100.0 / COUNT(*), 2) as conversion_rate
FROM funnel_steps;
```

**Click "Run"**

**Results:**
| total_sessions | home_visits | product_views | cart_adds | checkouts | purchases | conversion_rate |
|---------------|-------------|---------------|-----------|-----------|-----------|-----------------|
| 8,500 | 7,200 | 5,800 | 3,400 | 1,200 | 850 | 10.00 |

---

## PART 4: Query Optimization

### Step 21: Compare Bad vs Good Query
**❌ BAD Query (No Partition Filter):**
```sql
-- Scans ALL partitions (expensive!)
SELECT COUNT(DISTINCT user_id)
FROM clickstream
WHERE event_type = 'purchase';
```

**Data scanned:** ~300 KB (all 3 days)

**✅ GOOD Query (With Partition Filter):**
```sql
-- Scans ONLY specified partition (cheap!)
SELECT COUNT(DISTINCT user_id)
FROM clickstream
WHERE year = 2024 
  AND month = 6 
  AND day = 1
  AND event_type = 'purchase';
```

**Data scanned:** ~100 KB (1 day only)

**Savings:** 66% reduction in data scanned = 66% cost reduction

### Step 22: View Query History and Costs
1. Go to **"Recent queries"** tab (top of Athena console)
2. You'll see all your queries with:
   - **Query:** SQL text
   - **Data scanned:** Amount of data processed
   - **Run time:** Execution duration
   - **State:** SUCCEEDED / FAILED

**Athena Pricing:** $5 per TB scanned
- Bad query (300 KB): $0.0000015
- Good query (100 KB): $0.0000005
- **For 1 PB dataset:** Bad = $5,000, Good = $5 (99.9% savings!)

---

## PART 5: Create Athena Views

### Step 23: Create Reusable View
```sql
-- Create view for daily metrics
CREATE OR REPLACE VIEW clickstream_db.daily_metrics AS
SELECT 
    year,
    month,
    day,
    COUNT(DISTINCT user_id) as daily_active_users,
    COUNT(DISTINCT session_id) as sessions,
    COUNT(*) as total_events,
    COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) as purchasers,
    SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as purchases,
    ROUND(
        SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) * 100.0 / 
        COUNT(DISTINCT user_id), 
        2
    ) as conversion_rate
FROM clickstream
GROUP BY year, month, day;
```

**Click "Run"**

### Step 24: Query the View
```sql
SELECT * FROM clickstream_db.daily_metrics
ORDER BY year, month, day;
```

**Results:**
| year | month | day | daily_active_users | sessions | total_events | purchasers | purchases | conversion_rate |
|------|-------|-----|-------------------|----------|--------------|------------|-----------|-----------------|
| 2024 | 6 | 1 | 4,523 | 8,234 | 10,000 | 452 | 523 | 10.00 |
| 2024 | 6 | 2 | 4,612 | 8,401 | 10,000 | 461 | 534 | 10.00 |
| 2024 | 6 | 3 | 4,489 | 8,156 | 10,000 | 448 | 518 | 9.98 |

**Benefits of Views:**
- Simplify complex queries
- Reuse business logic
- Abstract partition structure
- Improve query performance

---

## PART 6: Enable Query Result Caching

### Step 25: Create Athena Workgroup
1. **Athena Console** → **"Workgroups"** (left sidebar)
2. Click **"Create workgroup"**
3. **Workgroup name:** `analytics-team`
4. **Description:** `Workgroup for analytics team with query limits`

### Step 26: Configure Workgroup Settings
1. **Query result location:** `s3://clickstream-datalake-[id]/athena-results/analytics-team/`
2. **Encrypt query results:** ✅ Enable
   - **Encryption type:** **"SSE-S3"**
3. **Override client-side settings:** ✅ Enable
4. Click **"Create workgroup"**

### Step 27: Set Data Usage Controls
1. Click on your workgroup: `analytics-team`
2. Go to **"Data usage controls"** tab
3. Click **"Edit"**
4. **Per query data usage control:** ✅ Enable
   - **Data limit:** `10` GB per query
   - Prevents accidentally scanning entire data lake
5. Click **"Save"**

### Step 28: Test Query Caching
1. Switch to workgroup: Select `analytics-team` from dropdown (top)
2. Run a query:
   ```sql
   SELECT COUNT(*) FROM clickstream WHERE year = 2024;
   ```
3. Note the **Run time** (e.g., 2.5 seconds)
4. Run the **exact same query** again
5. **Run time:** ~0.1 seconds (25x faster!)
6. **Data scanned:** 0 KB (cached result used)

**Query result caching:**
- Results cached for 7 days
- Only works for identical queries
- Free (no scan cost)

---

## ✅ Exercise 7.1 Completion Checklist

- [ ] Created S3 data lake with Hive-style partitions
- [ ] Generated 30,000 clickstream events in Parquet format
- [ ] Uploaded data to partitioned S3 structure
- [ ] Created Glue database
- [ ] Created IAM role for Glue Crawler
- [ ] Created and ran Glue Crawler
- [ ] Verified auto-discovered table and partitions
- [ ] Configured Athena query result location
- [ ] Queried daily active users with partition pruning
- [ ] Ran page views and conversion funnel analyses
- [ ] Compared partitioned vs non-partitioned query costs
- [ ] Created reusable view for daily metrics
- [ ] Created Athena workgroup with data usage limits
- [ ] Tested query result caching

**🎉 Congratulations!** You've completed Exercise 7.1!

---

# Exercise 7.2: Real-Time Streaming Analytics with Kinesis

**Duration:** 3-4 hours | **Estimated Cost:** ~$20-30 for lab session

## 🎯 Learning Objectives
- Create Kinesis Data Stream for high-throughput ingestion
- Produce 1M+ events per minute
- Perform real-time analytics with sliding windows
- Detect anomalies in streaming data
- Archive to S3 with Kinesis Data Firehose

---

## PART 1: Create Kinesis Data Stream

### Step 1: Navigate to Kinesis Console
1. AWS Console → Search **"Kinesis"**
2. Click **"Amazon Kinesis"**
3. Click **"Create data stream"**

### Step 2: Configure Data Stream
1. **Data stream name:** `iot-sensor-stream`
2. **Capacity mode:** Select **"Provisioned"**
   - **Shards:** `10` (for lab - handles ~10 MB/sec, 10K records/sec)
   - For production: 100+ shards for 1M events/min
3. Click **"Create data stream"**
4. **Wait for status:** **ACTIVE** (~1 minute)

### Step 3: Enable Enhanced Fan-Out (Optional)
1. Click on stream: `iot-sensor-stream`
2. Go to **"Consumers"** tab
3. Click **"Register consumer"**
4. **Consumer name:** `analytics-consumer`
5. Click **"Register consumer"**
6. This enables dedicated throughput (2 MB/sec per consumer)

### Step 4: Configure Stream Retention
1. Go to **"Configuration"** tab
2. **Data retention period:** Default is **24 hours**
3. To extend: Click **"Edit"**
   - Change to **168 hours** (7 days) OR
   - Up to **8760 hours** (365 days)
4. Click **"Save changes"**

**Cost note:** Extended retention increases costs (~$0.025/shard/hour)

---

## PART 2: Produce IoT Events to Kinesis

### Step 5: Create IAM Role for Producer
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** **"AWS service"** → **"EC2"** (we'll run producer on EC2)
3. Attach policy: **"AmazonKinesisFullAccess"**
4. **Role name:** `KinesisProducerRole`
5. **Create role**

### Step 6: Launch EC2 Instance for Producer
1. **EC2 Console** → **"Launch instance"**
2. **Name:** `kinesis-producer`
3. **AMI:** Amazon Linux 2023
4. **Instance type:** **t3.medium** (2 vCPU for parallel sending)
5. **Key pair:** Select existing or create new
6. **IAM instance profile:** Select `KinesisProducerRole`
7. **Launch instance**

### Step 7: Connect to EC2 and Install Dependencies
1. Select instance → **"Connect"** → **"EC2 Instance Connect"**
2. In terminal:

```bash
# Update system
sudo dnf update -y

# Install Python and pip
sudo dnf install python3-pip -y

# Install boto3
pip3 install boto3

# Verify
python3 --version
```

### Step 8: Create IoT Simulator Script
Create file: `iot_producer.py`

```python
#!/usr/bin/env python3
import boto3
import json
import time
import random
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor
import sys

kinesis = boto3.client('kinesis', region_name='us-east-1')
STREAM_NAME = 'iot-sensor-stream'

def generate_iot_event(device_id):
    """Generate random IoT sensor data"""
    return {
        'device_id': f'DEVICE_{device_id:05d}',
        'timestamp': datetime.now().isoformat(),
        'temperature': round(20 + random.uniform(-5, 35), 2),  # 15-55°C
        'humidity': round(random.uniform(30, 90), 2),
        'pressure': round(random.uniform(980, 1030), 2),
        'battery_level': random.randint(0, 100),
        'latitude': round(random.uniform(37.0, 38.0), 6),
        'longitude': round(random.uniform(-122.5, -121.5), 6),
        'status': random.choice(['NORMAL', 'WARNING', 'CRITICAL'])
    }

def send_batch(batch_num, batch_size=500):
    """Send a batch of records to Kinesis"""
    records = []
    
    for i in range(batch_size):
        device_id = random.randint(1, 10000)
        event = generate_iot_event(device_id)
        
        records.append({
            'Data': json.dumps(event),
            'PartitionKey': event['device_id']
        })
    
    try:
        response = kinesis.put_records(
            StreamName=STREAM_NAME,
            Records=records
        )
        
        failed = response['FailedRecordCount']
        success = batch_size - failed
        
        if batch_num % 10 == 0:
            print(f"Batch {batch_num:04d}: {success}/{batch_size} records sent, {failed} failed")
        
        return success
        
    except Exception as e:
        print(f"Error in batch {batch_num}: {str(e)}")
        return 0

def main():
    print(f"Starting IoT event producer for stream: {STREAM_NAME}")
    print(f"Sending to 10,000 devices...")
    
    total_records = 0
    start_time = time.time()
    
    # Send 200 batches in parallel (100,000 total events)
    with ThreadPoolExecutor(max_workers=20) as executor:
        futures = [executor.submit(send_batch, i) for i in range(200)]
        
        for future in futures:
            total_records += future.result()
    
    elapsed = time.time() - start_time
    rate = total_records / elapsed
    
    print(f"\n✓ Sent {total_records:,} events in {elapsed:.1f} seconds")
    print(f"✓ Throughput: {rate:,.0f} events/second")

if __name__ == '__main__':
    main()
```

Save and run:
```bash
vi iot_producer.py
# Paste the code, save (:wq)

chmod +x iot_producer.py
python3 iot_producer.py
```

**Expected output:**
```
Starting IoT event producer for stream: iot-sensor-stream
Sending to 10,000 devices...
Batch 0000: 500/500 records sent, 0 failed
Batch 0010: 500/500 records sent, 0 failed
...
✓ Sent 100,000 events in 45.2 seconds
✓ Throughput: 2,212 events/second
```

### Step 9: Monitor Stream in Console
1. **Kinesis Console** → Click on `iot-sensor-stream`
2. Go to **"Monitoring"** tab
3. You'll see graphs:
   - **Incoming records:** Spikes to ~2,000/sec
   - **Incoming bytes:** Shows data throughput
   - **Put records success:** Near 100%

---

## PART 3: Real-Time Analytics with Kinesis Data Analytics

### Step 10: Create Kinesis Data Analytics Application
1. **Kinesis Console** → **"Data Analytics applications"** (left sidebar)
2. Click **"Create application"**
3. **Application name:** `iot-realtime-analytics`
4. **Runtime:** Select **"SQL"**
5. Click **"Create application"**

### Step 11: Configure Input Stream
1. Click **"Connect to a source"**
2. **Source:** **"Kinesis stream"**
3. **Kinesis stream:** Select `iot-sensor-stream`
4. **IAM role:** Select **"Create / update IAM role"** (auto-creates role)
5. Click **"Discover schema"**
6. Wait ~30 seconds while Kinesis samples data
7. **Schema discovered!** You should see columns:
   - device_id (VARCHAR)
   - timestamp (VARCHAR)
   - temperature (DOUBLE)
   - humidity (DOUBLE)
   - etc.
8. Click **"Save and continue"**

### Step 12: Write SQL for Real-Time Aggregation
In the **SQL editor**, replace default query:

```sql
-- Create output stream for metrics
CREATE OR REPLACE STREAM "METRICS_STREAM" (
    device_id VARCHAR(20),
    window_start TIMESTAMP,
    window_end TIMESTAMP,
    avg_temperature DOUBLE,
    max_temperature DOUBLE,
    min_temperature DOUBLE,
    avg_humidity DOUBLE,
    event_count INTEGER
);

-- Sliding window aggregation (1 minute windows, updated every 10 seconds)
CREATE OR REPLACE PUMP "METRICS_PUMP" AS 
INSERT INTO "METRICS_STREAM"
SELECT STREAM 
    device_id,
    STEP(s.ROWTIME BY INTERVAL '10' SECOND) as window_start,
    STEP(s.ROWTIME BY INTERVAL '10' SECOND) + INTERVAL '1' MINUTE as window_end,
    AVG(temperature) as avg_temperature,
    MAX(temperature) as max_temperature,
    MIN(temperature) as min_temperature,
    AVG(humidity) as avg_humidity,
    COUNT(*) as event_count
FROM "SOURCE_SQL_STREAM_001" s
GROUP BY 
    device_id,
    STEP(s.ROWTIME BY INTERVAL '10' SECOND);

-- Create anomaly detection stream
CREATE OR REPLACE STREAM "ANOMALY_STREAM" (
    device_id VARCHAR(20),
    event_timestamp VARCHAR(30),
    temperature DOUBLE,
    anomaly_score DOUBLE
);

-- Detect temperature anomalies (> 50°C or < 10°C)
CREATE OR REPLACE PUMP "ANOMALY_PUMP" AS
INSERT INTO "ANOMALY_STREAM"
SELECT STREAM
    device_id,
    timestamp as event_timestamp,
    temperature,
    CASE 
        WHEN temperature > 50 THEN (temperature - 50) * 2.0
        WHEN temperature < 10 THEN (10 - temperature) * 2.0
        ELSE 0.0
    END as anomaly_score
FROM "SOURCE_SQL_STREAM_001"
WHERE temperature > 50 OR temperature < 10;
```

Click **"Save and run SQL"**

### Step 13: View Real-Time Results
1. Go to **"Real-time analytics"** tab
2. You'll see two output streams:
   - **METRICS_STREAM:** Aggregated data per device
   - **ANOMALY_STREAM:** Only anomalous readings
3. Click **"METRICS_STREAM"** to view results
4. You'll see windowed aggregations updating every 10 seconds

**Sample output:**
| device_id | window_start | avg_temperature | max_temperature | event_count |
|-----------|--------------|-----------------|-----------------|-------------|
| DEVICE_00123 | 2024-06-24 10:30:00 | 32.5 | 45.2 | 15 |
| DEVICE_00456 | 2024-06-24 10:30:00 | 28.7 | 38.1 | 12 |

---

## PART 4: Lambda Consumer for Alerts

### Step 14: Create Lambda Function for Anomaly Alerts
1. **Lambda Console** → **"Create function"**
2. **Function name:** `iot-anomaly-alerter`
3. **Runtime:** **"Python 3.12"**
4. **Permissions:** Create role with basic Lambda permissions
5. Click **"Create function"**

### Step 15: Write Lambda Code
Replace the code:

```python
import json
import boto3
import os

sns = boto3.client('sns')
SNS_TOPIC_ARN = os.environ.get('SNS_TOPIC_ARN')

def lambda_handler(event, context):
    print(f"Received {len(event['Records'])} Kinesis records")
    
    alerts = []
    
    for record in event['Records']:
        # Decode Kinesis data
        payload = json.loads(record['kinesis']['data'].decode('base64'))
        
        device_id = payload.get('device_id')
        temperature = payload.get('temperature')
        status = payload.get('status')
        timestamp = payload.get('timestamp')
        
        # Check for critical conditions
        if temperature > 55:
            alerts.append({
                'device': device_id,
                'issue': 'EXTREME_HEAT',
                'value': temperature,
                'timestamp': timestamp
            })
        elif temperature < 5:
            alerts.append({
                'device': device_id,
                'issue': 'EXTREME_COLD',
                'value': temperature,
                'timestamp': timestamp
            })
        elif status == 'CRITICAL':
            alerts.append({
                'device': device_id,
                'issue': 'DEVICE_CRITICAL',
                'value': temperature,
                'timestamp': timestamp
            })
    
    # Send alerts
    if alerts:
        message = f"🚨 ALERT: {len(alerts)} critical IoT events detected\n\n"
        for alert in alerts[:10]:  # Top 10
            message += f"Device: {alert['device']}\n"
            message += f"Issue: {alert['issue']}\n"
            message += f"Value: {alert['value']}°C\n"
            message += f"Time: {alert['timestamp']}\n\n"
        
        if SNS_TOPIC_ARN:
            sns.publish(
                TopicArn=SNS_TOPIC_ARN,
                Subject='IoT Sensor Alert',
                Message=message
            )
        
        print(f"✓ Sent alert for {len(alerts)} critical events")
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Processed {len(event["Records"])} records, {len(alerts)} alerts')
    }
```

Click **"Deploy"**

### Step 16: Create SNS Topic for Alerts
1. **SNS Console** → **"Topics"** → **"Create topic"**
2. **Type:** **"Standard"**
3. **Name:** `iot-critical-alerts`
4. Click **"Create topic"**
5. Click **"Create subscription"**
6. **Protocol:** **"Email"**
7. **Endpoint:** Your email address
8. Click **"Create subscription"**
9. **Check your email** and click confirmation link

### Step 17: Add SNS Topic to Lambda Environment Variables
1. Go back to Lambda function: `iot-anomaly-alerter`
2. **"Configuration"** → **"Environment variables"** → **"Edit"**
3. **Key:** `SNS_TOPIC_ARN`
4. **Value:** Copy ARN from SNS topic (e.g., `arn:aws:sns:us-east-1:123456789012:iot-critical-alerts`)
5. **Save**

### Step 18: Add Kinesis Trigger to Lambda
1. Click **"Add trigger"**
2. **Select a source:** **"Kinesis"**
3. **Kinesis stream:** `iot-sensor-stream`
4. **Batch size:** `100`
5. **Starting position:** **"Latest"**
6. **Enable trigger:** ✅ Checked
7. Click **"Add"**

### Step 19: Grant Lambda Permissions to SNS
1. Go to **"Configuration"** → **"Permissions"**
2. Click on the **execution role** link
3. Click **"Attach policies"**
4. Search and attach: **"AmazonSNSFullAccess"**

### Step 20: Test Anomaly Detection
Run the producer again to generate more events:
```bash
python3 iot_producer.py
```

Check your email - you should receive alerts for devices with extreme temperatures!

---

## PART 5: Archive to S3 with Kinesis Data Firehose

### Step 21: Create S3 Bucket for Archive
1. **S3 Console** → **"Create bucket"**
2. **Name:** `iot-data-archive-[account-id]`
3. Click **"Create bucket"**

### Step 22: Create Kinesis Data Firehose Delivery Stream
1. **Kinesis Console** → **"Delivery streams"** (left sidebar)
2. Click **"Create delivery stream"**
3. **Source:** **"Kinesis Data Stream"**
4. **Kinesis data stream:** Select `iot-sensor-stream`
5. **Delivery stream name:** `iot-to-s3-firehose`
6. Click **"Next"**

### Step 23: Configure Transform and Convert (Optional)
1. **Transform source records:** Select **"Disabled"** (for now)
2. **Convert record format:** Select **"Enabled"**
3. **Output format:** **"Apache Parquet"**
4. **Schema:** You can define or let it auto-detect
5. For simplicity, select **"Disabled"** for now
6. Click **"Next"**

### Step 24: Configure Destination
1. **S3 bucket:** Select `iot-data-archive-[account-id]`
2. **S3 prefix:** Type: `year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/`
   - This creates Hive-style partitions automatically!
3. **S3 error prefix:** `errors/`
4. **Buffer hints:**
   - **Buffer size:** `5` MB (small for faster delivery in lab)
   - **Buffer interval:** `60` seconds
5. **Compression:** **"GZIP"**
6. Click **"Next"**

### Step 25: Configure Settings
1. **IAM role:** Select **"Create or update IAM role"** (auto-creates)
2. **Error logging:** ✅ **"Enabled"**
3. Click **"Next"**

### Step 26: Review and Create
1. Review all settings
2. Click **"Create delivery stream"**
3. Wait for status: **Active** (~2 minutes)

### Step 27: Verify Data in S3
1. Wait 2-3 minutes for buffer to flush
2. Go to S3 bucket: `iot-data-archive-[id]`
3. Navigate to partitioned structure:
   - `year=2024/month=06/day=24/`
4. You should see GZIP files with IoT sensor data!
5. Download one and extract:
   ```bash
   aws s3 cp s3://iot-data-archive-[id]/year=2024/month=06/day=24/[filename].gz .
   gunzip [filename].gz
   head [filename]
   ```

You should see JSON IoT events!

---

## ✅ Exercise 7.2 Completion Checklist

- [ ] Created Kinesis Data Stream with 10 shards
- [ ] Configured stream retention (24 hours or extended)
- [ ] Created IAM role for producer
- [ ] Launched EC2 instance with IAM role
- [ ] Created and ran IoT event producer (100K events)
- [ ] Monitored stream metrics in console
- [ ] Created Kinesis Data Analytics application
- [ ] Configured input stream with schema discovery
- [ ] Wrote SQL for real-time aggregation (sliding windows)
- [ ] Created anomaly detection stream
- [ ] Viewed real-time analytics results
- [ ] Created Lambda function for alerts
- [ ] Created SNS topic and email subscription
- [ ] Added Kinesis trigger to Lambda
- [ ] Tested anomaly detection and alerts
- [ ] Created Kinesis Data Firehose delivery stream
- [ ] Configured S3 destination with partitioning
- [ ] Verified archived data in S3

**🎉 Congratulations!** You've completed Exercise 7.2!

---

# Exercise 7.3: Business Intelligence with QuickSight

**Duration:** 2-3 hours | **Estimated Cost:** ~$10-15 for lab session

## 🎯 Learning Objectives
- Set up Amazon QuickSight
- Connect to Athena data source
- Create interactive dashboards
- Build visualizations (charts, graphs, KPIs)
- Share dashboards with teams

---

## PART 1: Set Up Amazon QuickSight

### Step 1: Navigate to QuickSight Console
1. AWS Console → Search **"QuickSight"**
2. Click **"Amazon QuickSight"**
3. If first time, you'll see **"Sign up for QuickSight"**

### Step 2: Sign Up for QuickSight
1. Click **"Sign up for QuickSight"**
2. **Account type:** Select **"Standard"** ($9/month per user)
   - **Enterprise:** $18/month (includes ML insights, advanced features)
3. **Authentication method:** **"Use IAM federated identities & QuickSight-managed users"**
4. Click **"Continue"**

### Step 3: Configure QuickSight Account
1. **QuickSight account name:** Type: `data-analytics-[your-name]`
2. **Notification email:** Your email address
3. **QuickSight capacity region:** Select your region (e.g., us-east-1)
4. **Enable autodiscovery:** ✅ Check
   - **S3:** ✅ Check `Select S3 buckets`
   - Select: `clickstream-datalake-[id]`, `iot-data-archive-[id]`
   - ✅ Check **"Athena workgroups"**
5. Click **"Finish"**

**Wait 2-3 minutes for account provisioning.**

### Step 4: Access QuickSight
1. Once ready, click **"Go to Amazon QuickSight"**
2. You'll land on the QuickSight home page

---

## PART 2: Connect to Athena Data Source

### Step 5: Create New Data Source
1. Click **"Datasets"** (left sidebar)
2. Click **"New dataset"** button
3. Scroll down and click **"Athena"**

### Step 6: Configure Athena Connection
1. **Data source name:** `clickstream-athena`
2. **Athena workgroup:** Select `primary` (or your custom workgroup)
3. Click **"Validate connection"**
4. Should show: **✓ Validated**
5. Click **"Create data source"**

### Step 7: Choose Database and Table
1. **Database:** Select `clickstream_db`
2. **Tables:** Select `clickstream` table
3. **Data source type:** Select **"Use custom SQL"** (more flexible)
4. **Custom SQL name:** `clickstream_analysis`
5. **Custom SQL query:**
   ```sql
   SELECT 
       CAST(year AS varchar) || '-' || 
       LPAD(CAST(month AS varchar), 2, '0') || '-' || 
       LPAD(CAST(day AS varchar), 2, '0') as date,
       event_id,
       user_id,
       session_id,
       event_timestamp,
       page_url,
       event_type,
       device_type,
       browser,
       country,
       referrer,
       duration_seconds,
       year,
       month,
       day
   FROM clickstream
   WHERE year = 2024 AND month = 6
   ```
6. Click **"Confirm query"**

### Step 8: Configure Data Import
1. **Import to SPICE:** Select **"Import to SPICE for quicker analytics"**
   - SPICE = Super-fast, Parallel, In-memory Calculation Engine
   - 10 GB free with Standard edition
2. Click **"Edit/Preview data"**

### Step 9: Preview and Prepare Data
1. You'll see a data preview with sample rows
2. **Add calculated fields** if needed (we'll do this later)
3. For now, just verify data looks correct
4. Click **"Publish & visualize"** (top right, orange button)

**Wait ~30 seconds for SPICE import to complete.**

---

## PART 3: Create Interactive Dashboard

### Step 10: Start Building Analysis
1. You're now in the **Analysis editor**
2. Left panel shows **Fields** from your dataset
3. Center is the **canvas** for visualizations
4. Right panel shows **Visual types**

### Step 11: Add KPI - Total Events
1. Click **"Add"** → **"Add visual"**
2. **Visual types:** Click **"KPI"** (Key Performance Indicator)
3. **Field wells:**
   - **Value:** Drag `event_id` → It auto-changes to **Count**
4. Click the visual title and rename to: **"Total Events"**
5. You should see: **30,000**

### Step 12: Add KPI - Unique Users
1. Click **"Add"** → **"Add visual"**
2. Select **"KPI"**
3. **Value:** Drag `user_id`
4. Click the dropdown on `user_id` → Select **"Aggregate: Count distinct"**
5. Rename visual: **"Unique Users"**
6. You should see: **~4,500**

### Step 13: Add Line Chart - Daily Trend
1. **"Add visual"** → Select **"Line chart"**
2. **Field wells:**
   - **X-axis:** Drag `date`
   - **Value:** Drag `event_id` (becomes Count)
3. Rename: **"Daily Events Trend"**
4. You'll see a line graph showing 3 days of data

### Step 14: Add Pie Chart - Device Distribution
1. **"Add visual"** → Select **"Pie chart"**
2. **Field wells:**
   - **Group/Color:** Drag `device_type`
   - **Value:** Drag `event_id` (Count)
3. Rename: **"Traffic by Device Type"**
4. You'll see pie slices for mobile, desktop, tablet

### Step 15: Add Bar Chart - Page Views
1. **"Add visual"** → Select **"Horizontal bar chart"**
2. **Field wells:**
   - **Y-axis:** Drag `page_url`
   - **Value:** Drag `event_id` (Count)
3. Click **"Sort"** icon → Sort by **Value (descending)**
4. Rename: **"Top Pages by Views"**

### Step 16: Add Heat Map - Country x Browser
1. **"Add visual"** → Select **"Heat map"**
2. **Field wells:**
   - **Rows:** Drag `country`
   - **Columns:** Drag `browser`
   - **Values:** Drag `event_id` (Count)
3. Rename: **"Traffic by Country & Browser"**

### Step 17: Add Calculated Field - Conversion Rate
1. In the **Fields** panel (left), click **"Add calculated field"**
2. **Calculated field name:** `conversion_rate`
3. **Formula:**
   ```
   countIf(event_type, event_type = 'purchase') / count(event_id) * 100
   ```
4. Click **"Save"**

### Step 18: Add Gauge - Conversion Rate
1. **"Add visual"** → Select **"Gauge chart"**
2. **Value:** Drag `conversion_rate` (the calculated field)
3. **Min:** 0, **Max:** 20 (expected range)
4. Rename: **"Overall Conversion Rate %"**

---

## PART 4: Add Interactivity and Filters

### Step 19: Add Date Filter
1. Click **"Filter"** (left panel) → **"Add filter"**
2. Select `date`
3. **Filter type:** **"Date filter"**
4. **Start date:** 2024-06-01
5. **End date:** 2024-06-03
6. ✅ **"Apply filter to all visuals"**
7. Click **"Apply"**

### Step 20: Add Country Filter as Control
1. Click **"Controls"** (left panel, next to Filter) → **"Add control"**
2. Select `country`
3. **Control type:** **"Multi-select dropdown"**
4. **Display name:** "Select Countries"
5. **Values:** Select all
6. Click **"Add"**

You'll now see a dropdown at the top of the dashboard!

### Step 21: Add Drill-Down to Bar Chart
1. Click on the **"Top Pages by Views"** bar chart
2. Click the dropdown menu (top right of visual) → **"Focus mode"**
3. Click on a bar (e.g., /home)
4. You can see detailed data for that page
5. Press **Escape** to exit focus mode

### Step 22: Enable Click-to-Filter
1. Click on **"Traffic by Device Type"** pie chart
2. Click on any slice (e.g., "mobile")
3. **All other visuals update** to show only mobile traffic!
4. This is **cross-filtering** - click again to deselect

---

## PART 5: Publish and Share Dashboard

### Step 23: Arrange Dashboard Layout
1. Resize and rearrange visuals:
   - Drag visuals to reorder
   - Resize by dragging corners
   - Create a clean, logical layout
2. Suggested layout:
   - Row 1: KPIs (Total Events, Unique Users, Conversion Rate)
   - Row 2: Line chart (Daily Trend)
   - Row 3: Pie chart + Bar chart
   - Row 4: Heat map

### Step 24: Add Dashboard Title and Description
1. Click **"Untitled analysis"** (top left)
2. Rename to: **"Clickstream Analytics Dashboard"**
3. Click **"Add"** → **"Add text box"**
4. Type: 
   ```
   📊 Real-Time Clickstream Analytics
   Updated: June 2024
   Data Source: AWS Athena (clickstream_db)
   ```
5. Style as header (bold, larger font)

### Step 25: Publish Dashboard
1. Click **"Publish"** button (top right)
2. **Publish dashboard as:** `clickstream-dashboard`
3. Click **"Publish dashboard"**

**Dashboard is now published!**

### Step 26: Share Dashboard
1. Click **"Share"** icon (top right)
2. **Add users:**
   - Type email addresses of team members
   - Set permission: **"Viewer"** (can view) OR **"Co-owner"** (can edit)
3. Click **"Share"**
4. Recipients receive email with dashboard link

### Step 27: Generate Shareable Link (Optional)
1. Click **"Share"** → **"Manage dashboard access"**
2. ✅ Enable **"Public access"**
3. Copy the public URL
4. Anyone with the link can view (no AWS account needed)

**⚠️ Warning:** Only use public links for non-sensitive data!

---

## PART 6: Schedule Refresh and Alerts

### Step 28: Schedule SPICE Refresh
1. Go to **"Datasets"** (left sidebar)
2. Click on your dataset: `clickstream_analysis`
3. Click **"Refresh"** tab
4. Click **"Add new schedule"**
5. **Frequency:** **"Daily"**
6. **Time zone:** Your time zone
7. **Time:** **3:00 AM** (after Glue crawler runs)
8. Click **"Create"**

Now your dashboard auto-updates daily!

### Step 29: Create Threshold Alert (Enterprise Only)
For Enterprise edition:
1. Click on a KPI (e.g., "Total Events")
2. Click the menu → **"Add alert"**
3. **Alert name:** `low-traffic-alert`
4. **Condition:** Total Events **< 5,000**
5. **Recipients:** Your email
6. Click **"Create"**

You'll receive email if traffic drops below threshold.

### Step 30: Embed Dashboard in Web App (Advanced)
For embedding in your website/app:
1. **QuickSight Console** → **"Manage QuickSight"** → **"Domains and Embedding"**
2. Add your domain
3. Use QuickSight Embedding SDK in your JavaScript app
4. Generate embed URL with temporary credentials

(This requires developer setup - beyond GUI scope)

---

## ✅ Exercise 7.3 Completion Checklist

- [ ] Signed up for Amazon QuickSight account
- [ ] Configured S3 and Athena access permissions
- [ ] Created Athena data source connection
- [ ] Wrote custom SQL query for analysis
- [ ] Imported data to SPICE
- [ ] Created KPI visuals (Total Events, Unique Users)
- [ ] Created line chart for daily trends
- [ ] Created pie chart for device distribution
- [ ] Created bar chart for top pages
- [ ] Created heat map for country/browser analysis
- [ ] Added calculated field for conversion rate
- [ ] Created gauge chart for conversion rate
- [ ] Added date range filter
- [ ] Added country filter control
- [ ] Enabled cross-visual filtering
- [ ] Arranged dashboard layout
- [ ] Published dashboard
- [ ] Shared dashboard with team members
- [ ] Scheduled daily SPICE refresh

**🎉 Congratulations!** You've completed Exercise 7.3 and all of Module 7!

---

# 🎓 Module 7 Complete Summary

## What You've Accomplished

### Exercise 7.1: Athena Data Lake Analytics ✅
- Organized data lake with Hive-style partitions
- Auto-discovered schema with Glue Crawler
- Queried 30,000 events with partition pruning
- Achieved 99.9% cost reduction with proper partitioning
- Created reusable views and workgroups

### Exercise 7.2: Kinesis Streaming Analytics ✅
- Created Kinesis stream handling 2,000+ events/sec
- Produced 100,000 IoT sensor events
- Performed real-time aggregation with sliding windows
- Detected anomalies and sent SNS alerts
- Archived data to S3 with automatic partitioning

### Exercise 7.3: QuickSight Dashboards ✅
- Set up QuickSight with SPICE acceleration
- Created 8 interactive visualizations
- Built calculated metrics (conversion rate)
- Added filters and cross-visual interactivity
- Published and shared dashboards

## Key Services Mastered
1. **Amazon Athena** - Serverless SQL queries on S3
2. **AWS Glue** - Schema discovery and crawlers
3. **Amazon Kinesis** - Real-time streaming data
4. **Amazon QuickSight** - Business intelligence dashboards

## Next Steps
- **Module 8:** Application Integration (EventBridge, SQS, Step Functions)
- Continue with remaining modules

---

## 🧹 Cleanup Resources

**Athena/Glue:**
- Delete Glue crawlers
- Delete Glue databases and tables
- Empty S3 bucket (athena-results, clickstream-datalake)

**Kinesis:**
- Delete Kinesis Data Analytics application
- Delete Kinesis Data Firehose delivery stream
- Delete Kinesis Data Stream
- Delete Lambda functions
- Terminate EC2 producer instance

**QuickSight:**
- Delete datasets
- Delete dashboards and analyses
- Cancel QuickSight subscription (Settings → Manage QuickSight → Account settings → Unsubscribe)

**S3:**
- Empty and delete all buckets

**SNS:**
- Delete topics and subscriptions

**Estimated Module 7 Lab Cost (4 hours): ~$35-55**

---

**🎉 Excellent work! Ready for Module 8 - Application Integration!**

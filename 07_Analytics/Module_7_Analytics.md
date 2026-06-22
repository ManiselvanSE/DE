# MODULE 7: ANALYTICS SERVICES FOR DATA ENGINEERING

## MODULE OVERVIEW

**Duration:** 10-12 hours  
**Difficulty:** Advanced  
**Prerequisites:** Modules 1-6, SQL proficiency, understanding of data lakes  
**Cost:** $50-150 (query execution, streaming data)

---

## WHAT YOU WILL LEARN

### AWS Analytics Services for Data Engineering

This module covers the complete AWS analytics stack for building modern data lakes and real-time analytics platforms. You'll learn how to query petabyte-scale datasets, process streaming data, and visualize insights.

**Key Services:**
- ✅ **Amazon Athena** - Serverless SQL queries on S3 (pay per query)
- ✅ **AWS Glue** - Data catalog and serverless ETL
- ✅ **Amazon Kinesis** - Real-time streaming data ingestion and processing
- ✅ **Amazon QuickSight** - Business intelligence and dashboards
- ✅ **AWS Lake Formation** - Data lake security and governance
- ✅ **Amazon Redshift Spectrum** - Query S3 from Redshift

### Production Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│           Modern Data Lake Analytics Architecture            │
└─────────────────────────────────────────────────────────────┘

Data Ingestion
├─ Batch: S3 uploads (daily ETL, historical data)
├─ Streaming: Kinesis Data Streams (real-time events, logs)
└─ Database CDC: Kinesis Data Streams (DMS change data capture)
    │
    ▼
Data Lake (Amazon S3)
├─ Raw Zone: s3://datalake/raw/ (JSON, CSV, logs)
├─ Processed Zone: s3://datalake/processed/ (Parquet, partitioned)
└─ Curated Zone: s3://datalake/curated/ (aggregated, business-ready)
    │
    ├─ AWS Glue Catalog (central metadata repository)
    │
    ├────────────┬────────────┬────────────┐
    │            │            │            │
    ▼            ▼            ▼            ▼
Athena      Redshift    EMR Spark    QuickSight
(Ad-hoc)    Spectrum    (Complex)    (Dashboards)
(SQL)       (Joins)     (ML)         (Visualize)

Lake Formation
└─ Fine-grained access control (column/row-level security)
```

---

## EXERCISE 7.1: SERVERLESS DATA LAKE ANALYTICS WITH ATHENA

**Scenario:** Build a serverless analytics platform for 1 PB of website clickstream data stored in S3. Enable data analysts to run SQL queries without managing infrastructure.

**Architecture:**

```
Amazon S3 (Data Lake)
└─ s3://clickstream-datalake/
   ├─ raw/
   │  └─ year=2024/month=06/day=22/*.json.gz (100 GB/day)
   │
   └─ processed/
      └─ year=2024/month=06/day=22/*.parquet (10 GB/day, compressed)
          │
          │ AWS Glue Crawler (daily, discovers schema)
          ▼
AWS Glue Data Catalog
├─ Database: clickstream_db
├─ Table: raw_events (partitioned by year/month/day)
└─ Table: processed_events (partitioned, optimized)
    │
    ▼
Amazon Athena
├─ Query engine: Presto-based, serverless
├─ Workgroup: analytics-team (query limits, cost controls)
└─ Output: s3://athena-query-results/
    │
    ▼
Amazon QuickSight
└─ Dashboards: Daily active users, conversion funnels, page views
```

**Implementation:**

**Step 1: Organize Data Lake with Partitioning**

```bash
# S3 bucket structure (Hive-style partitioning)
aws s3 ls s3://clickstream-datalake/processed/

# Output:
# s3://clickstream-datalake/processed/year=2024/month=01/day=01/events.parquet
# s3://clickstream-datalake/processed/year=2024/month=01/day=02/events.parquet
# ...
# s3://clickstream-datalake/processed/year=2024/month=06/day=22/events.parquet

# Benefits of partitioning:
# - Athena scans only relevant partitions (10 GB vs 1 PB)
# - Query cost: $0.05 (10 GB scanned) vs $5,000 (1 PB scanned)
# - 99.99% cost savings ✅
```

**Step 2: Create Glue Crawler to Discover Schema**

```bash
# Create Glue database
aws glue create-database \
  --database-input '{
    "Name": "clickstream_db",
    "Description": "Clickstream analytics data lake"
  }'

# Create IAM role for Glue crawler
cat > glue-crawler-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "glue.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name AWSGlueServiceRole-Crawler \
  --assume-role-policy-document file://glue-crawler-trust-policy.json

aws iam attach-role-policy \
  --role-name AWSGlueServiceRole-Crawler \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

aws iam attach-role-policy \
  --role-name AWSGlueServiceRole-Crawler \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Create Glue crawler
aws glue create-crawler \
  --name clickstream-crawler \
  --role arn:aws:iam::123456789012:role/AWSGlueServiceRole-Crawler \
  --database-name clickstream_db \
  --targets '{
    "S3Targets": [{
      "Path": "s3://clickstream-datalake/processed/"
    }]
  }' \
  --schema-change-policy '{
    "UpdateBehavior": "UPDATE_IN_DATABASE",
    "DeleteBehavior": "LOG"
  }' \
  --schedule "cron(0 3 * * ? *)"

# Run crawler immediately (first time)
aws glue start-crawler --name clickstream-crawler

# Wait for completion
aws glue get-crawler --name clickstream-crawler | jq '.Crawler.State'

# Verify table created
aws glue get-table \
  --database-name clickstream_db \
  --name processed | jq '.Table.Name, .Table.PartitionKeys, .Table.StorageDescriptor.Columns[0:5]'

# Output:
# "processed"
# Partition keys: [{"Name": "year"}, {"Name": "month"}, {"Name": "day"}]
# Columns: [
#   {"Name": "user_id", "Type": "bigint"},
#   {"Name": "session_id", "Type": "string"},
#   {"Name": "event_type", "Type": "string"},
#   {"Name": "page_url", "Type": "string"},
#   {"Name": "timestamp", "Type": "timestamp"}
# ]
```

**Step 3: Query Data with Athena**

```sql
-- Create Athena workgroup with query limits
CREATE WORKGROUP analytics_team
WITH (
  publish_cloudwatch_metrics_enabled = true,
  bytes_scanned_cutoff_per_query = 100000000000,  -- 100 GB max per query
  result_configuration = (
    output_location = 's3://athena-query-results/analytics-team/'
  )
)

-- Query 1: Daily active users (partitioned query)
SELECT
    year,
    month,
    day,
    COUNT(DISTINCT user_id) AS daily_active_users
FROM clickstream_db.processed
WHERE year = 2024
  AND month = 6
  AND day = 22
GROUP BY year, month, day;

-- Data scanned: 10 GB (1 day partition)
-- Query time: 3.2 seconds
-- Cost: $0.05 (10 GB × $0.005/GB)

-- Query 2: Page views by hour (time-series analysis)
SELECT
    date_trunc('hour', timestamp) AS hour,
    page_url,
    COUNT(*) AS page_views,
    COUNT(DISTINCT user_id) AS unique_visitors
FROM clickstream_db.processed
WHERE year = 2024
  AND month = 6
  AND day BETWEEN 15 AND 22  -- Last 7 days
GROUP BY 1, 2
ORDER BY hour DESC, page_views DESC;

-- Data scanned: 70 GB (7 days)
-- Query time: 8.5 seconds
-- Cost: $0.35 (70 GB × $0.005/GB)

-- Query 3: Conversion funnel analysis
WITH funnel_events AS (
  SELECT
    session_id,
    MAX(CASE WHEN event_type = 'landing' THEN 1 ELSE 0 END) AS landing,
    MAX(CASE WHEN event_type = 'product_view' THEN 1 ELSE 0 END) AS product_view,
    MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) AS add_to_cart,
    MAX(CASE WHEN event_type = 'checkout' THEN 1 ELSE 0 END) AS checkout,
    MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchase
  FROM clickstream_db.processed
  WHERE year = 2024
    AND month = 6
    AND day = 22
  GROUP BY session_id
)
SELECT
  SUM(landing) AS step1_landing,
  SUM(product_view) AS step2_product_view,
  SUM(add_to_cart) AS step3_add_to_cart,
  SUM(checkout) AS step4_checkout,
  SUM(purchase) AS step5_purchase,
  ROUND(100.0 * SUM(product_view) / NULLIF(SUM(landing), 0), 2) AS conversion_rate_step2,
  ROUND(100.0 * SUM(add_to_cart) / NULLIF(SUM(product_view), 0), 2) AS conversion_rate_step3,
  ROUND(100.0 * SUM(checkout) / NULLIF(SUM(add_to_cart), 0), 2) AS conversion_rate_step4,
  ROUND(100.0 * SUM(purchase) / NULLIF(SUM(checkout), 0), 2) AS conversion_rate_step5
FROM funnel_events;

-- Output:
-- step1_landing: 500,000
-- step2_product_view: 250,000 (50% conversion)
-- step3_add_to_cart: 100,000 (40% conversion)
-- step4_checkout: 50,000 (50% conversion)
-- step5_purchase: 25,000 (50% conversion)
-- Overall conversion (landing → purchase): 5%
```

**Step 4: Optimize Queries with Partitioning and Compression**

```sql
-- Bad query (scans entire 1 PB dataset)
SELECT COUNT(*) FROM clickstream_db.processed;
-- Data scanned: 1 PB
-- Cost: $5,000 ❌

-- Good query (partition pruning)
SELECT COUNT(*)
FROM clickstream_db.processed
WHERE year = 2024
  AND month = 6
  AND day = 22;
-- Data scanned: 10 GB
-- Cost: $0.05 ✅ (99.999% cheaper)

-- Optimize: Create columnar Parquet format
-- JSON (uncompressed): 100 GB/day
-- Parquet (Snappy compression): 10 GB/day (10x smaller)
-- Query performance: 5x faster (columnar storage)

-- Optimize: Use partitioning predicates
-- Always include year/month/day in WHERE clause
-- Athena skips irrelevant partitions automatically
```

**Step 5: Create Athena Views for Common Queries**

```sql
-- Create view for daily metrics (reusable)
CREATE OR REPLACE VIEW clickstream_db.daily_metrics AS
SELECT
    year,
    month,
    day,
    COUNT(*) AS total_events,
    COUNT(DISTINCT user_id) AS daily_active_users,
    COUNT(DISTINCT session_id) AS total_sessions,
    ROUND(AVG(session_duration_sec), 2) AS avg_session_duration,
    SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS total_purchases,
    SUM(CASE WHEN event_type = 'purchase' THEN purchase_amount ELSE 0 END) AS total_revenue
FROM clickstream_db.processed
GROUP BY year, month, day;

-- Query view (faster, cached schema)
SELECT * FROM clickstream_db.daily_metrics
WHERE year = 2024 AND month = 6
ORDER BY year DESC, month DESC, day DESC
LIMIT 30;
```

**Step 6: Set Up Query Result Caching**

```python
# Python: Query Athena with result caching
import boto3
import time
import hashlib

athena = boto3.client('athena', region_name='us-east-1')

def query_athena_cached(query, database='clickstream_db', cache_ttl=3600):
    """
    Execute Athena query with result caching
    Caches results for 1 hour to avoid duplicate query costs
    """
    
    # Generate cache key from query
    cache_key = hashlib.md5(query.encode()).hexdigest()
    
    # Check if cached result exists in S3
    s3 = boto3.client('s3')
    cache_bucket = 'athena-query-cache'
    cache_key_path = f'cache/{cache_key}.json'
    
    try:
        # Try to get cached result
        cached_result = s3.get_object(Bucket=cache_bucket, Key=cache_key_path)
        cache_age = (time.time() - cached_result['LastModified'].timestamp())
        
        if cache_age < cache_ttl:
            print(f"✅ Cache hit (age: {cache_age:.0f} seconds)")
            return eval(cached_result['Body'].read().decode('utf-8'))
    except s3.exceptions.NoSuchKey:
        pass
    
    print("Cache miss - executing query...")
    
    # Execute query
    response = athena.start_query_execution(
        QueryString=query,
        QueryExecutionContext={'Database': database},
        ResultConfiguration={
            'OutputLocation': 's3://athena-query-results/analytics-team/'
        },
        WorkGroup='analytics_team'
    )
    
    query_execution_id = response['QueryExecutionId']
    
    # Wait for completion
    while True:
        result = athena.get_query_execution(QueryExecutionId=query_execution_id)
        state = result['QueryExecution']['Status']['State']
        
        if state == 'SUCCEEDED':
            break
        elif state in ['FAILED', 'CANCELLED']:
            raise Exception(f"Query failed: {result['QueryExecution']['Status']['StateChangeReason']}")
        
        time.sleep(1)
    
    # Get results
    results = athena.get_query_results(QueryExecutionId=query_execution_id)
    
    # Cache results
    s3.put_object(
        Bucket=cache_bucket,
        Key=cache_key_path,
        Body=str(results).encode('utf-8')
    )
    
    # Get query statistics
    stats = result['QueryExecution']['Statistics']
    print(f"Data scanned: {stats['DataScannedInBytes'] / 1024**3:.2f} GB")
    print(f"Execution time: {stats['EngineExecutionTimeInMillis'] / 1000:.2f} seconds")
    
    return results

# Example usage
query = """
SELECT year, month, day, COUNT(DISTINCT user_id) AS dau
FROM clickstream_db.processed
WHERE year = 2024 AND month = 6
GROUP BY year, month, day
ORDER BY year DESC, month DESC, day DESC
LIMIT 7
"""

results = query_athena_cached(query)

# First execution: Runs query, costs $0.05
# Subsequent executions (within 1 hour): Returns cached result, costs $0 ✅
```

**Results:**

| Metric | Traditional Data Warehouse | Athena + S3 Data Lake | Improvement |
|--------|----------------------------|----------------------|-------------|
| **Infrastructure Cost** | $5,000/month (Redshift cluster) | $0/month (serverless) | **100% savings** |
| **Storage Cost** | $1,000/TB/month (Redshift) | $23/TB/month (S3 Standard) | **98% cheaper** |
| **Query Cost** | Included (fixed) | $5/TB scanned (variable) | Pay per query |
| **Setup Time** | 4 hours (cluster, schema) | 15 minutes (crawler) | **93% faster** |
| **Scaling** | Manual (resize cluster) | Automatic (unlimited) | **Infinite scale** |

**Cost Analysis:**

```
Monthly Analytics Cost (1 PB data lake):

Storage (S3 Standard):
├─ 1 PB = 1,000 TB × $23/TB/month = $23,000/month

Athena Query Costs:
├─ Scenario 1: Ad-hoc analysts (100 queries/day, 50 GB avg)
│  └─ 100 queries × 50 GB × $0.005/GB × 30 days = $750/month
│
├─ Scenario 2: Dashboard auto-refresh (1,000 queries/day, 10 GB avg)
│  └─ 1,000 queries × 10 GB × $0.005/GB × 30 days = $1,500/month
│
└─ Total: $750 - $1,500/month

Total Monthly Cost: $23,750 - $24,500/month

vs Redshift (comparable workload):
├─ dc2.8xlarge cluster (3 nodes): $14,400/month
├─ Storage (1 PB): $120,000/month
└─ Total: $134,400/month

Athena Savings: $109,900/month (82% cheaper) ✅
```

**Key Lessons:**

1. **Partitioning is critical** - Reduces scanned data by 99.99%
2. **Parquet format** - 10x compression, 5x faster queries
3. **Athena is serverless** - No infrastructure to manage, pay per query
4. **Glue Crawler** - Auto-discovers schema, keeps catalog updated
5. **Query caching** - Avoid duplicate costs for repeated queries

---

## EXERCISE 7.2: REAL-TIME STREAMING ANALYTICS WITH KINESIS

**Scenario:** Build a real-time analytics pipeline for IoT sensors sending 1 million events per minute. Process, aggregate, and visualize metrics in near real-time.

**Architecture:**

```
IoT Devices (1 million sensors)
├─ Send metrics every 60 seconds
└─ Total: 1M events/minute = 16,667 events/second
    │
    │ AWS IoT Core → Kinesis Data Streams
    ▼
Amazon Kinesis Data Streams
├─ Shards: 100 (10,000 records/sec per shard capacity)
├─ Retention: 24 hours (configurable up to 365 days)
└─ Data format: JSON (sensor_id, temperature, humidity, timestamp)
    │
    ├─────────────┬─────────────────┐
    │             │                 │
    ▼             ▼                 ▼
Lambda          Kinesis            Kinesis
(Real-time      Data               Data
Alerts)         Analytics          Firehose
                (SQL queries)      (S3 Archive)
    │             │                 │
    │             ▼                 ▼
    │         CloudWatch        S3 Data Lake
    │         Metrics           └─ Athena queries
    │
    └─→ SNS → Email/Slack (temperature > 100°C alert)
```

**Implementation:**

**Step 1: Create Kinesis Data Stream**

```bash
# Create Kinesis stream with 100 shards
aws kinesis create-stream \
  --stream-name iot-sensor-stream \
  --shard-count 100 \
  --region us-east-1

# Enable enhanced fan-out (dedicated throughput per consumer)
aws kinesis register-stream-consumer \
  --stream-arn arn:aws:kinesis:us-east-1:123456789012:stream/iot-sensor-stream \
  --consumer-name realtime-analytics

# Configure retention (7 days)
aws kinesis increase-stream-retention-period \
  --stream-name iot-sensor-stream \
  --retention-period-hours 168

# Verify stream
aws kinesis describe-stream \
  --stream-name iot-sensor-stream | jq '.StreamDescription.Shards | length'

# Output: 100 shards
```

**Step 2: Produce Events to Kinesis (IoT Simulator)**

```python
# iot_simulator.py
import boto3
import json
import time
import random
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor

kinesis = boto3.client('kinesis', region_name='us-east-1')

STREAM_NAME = 'iot-sensor-stream'
NUM_SENSORS = 1000000
EVENTS_PER_MINUTE = 1000000  # 1M events/minute

def generate_sensor_data(sensor_id):
    """Generate realistic sensor data"""
    return {
        'sensor_id': f'sensor-{sensor_id:07d}',
        'temperature': round(random.uniform(20.0, 100.0), 2),
        'humidity': round(random.uniform(30.0, 90.0), 2),
        'pressure': round(random.uniform(980.0, 1020.0), 2),
        'timestamp': datetime.utcnow().isoformat(),
        'location': {
            'lat': round(random.uniform(-90, 90), 6),
            'lon': round(random.uniform(-180, 180), 6)
        }
    }

def send_batch_to_kinesis(batch):
    """Send batch of records to Kinesis"""
    
    records = [{
        'Data': json.dumps(data),
        'PartitionKey': data['sensor_id']  # Ensures same sensor goes to same shard
    } for data in batch]
    
    try:
        response = kinesis.put_records(
            StreamName=STREAM_NAME,
            Records=records
        )
        
        failed_count = response['FailedRecordCount']
        if failed_count > 0:
            print(f"⚠️  {failed_count} records failed")
        
        return len(records) - failed_count
    
    except Exception as e:
        print(f"❌ Error: {str(e)}")
        return 0

def run_simulator():
    """Simulate 1M sensors sending data every minute"""
    
    print(f"Starting IoT simulator: {NUM_SENSORS:,} sensors")
    print(f"Target rate: {EVENTS_PER_MINUTE:,} events/minute")
    
    batch_size = 500  # Kinesis PutRecords max: 500 records
    batches_per_minute = EVENTS_PER_MINUTE // batch_size
    
    while True:
        start_time = time.time()
        total_sent = 0
        
        # Generate and send batches in parallel
        with ThreadPoolExecutor(max_workers=50) as executor:
            futures = []
            
            for batch_num in range(batches_per_minute):
                # Generate batch
                batch = [
                    generate_sensor_data(random.randint(1, NUM_SENSORS))
                    for _ in range(batch_size)
                ]
                
                # Submit to thread pool
                future = executor.submit(send_batch_to_kinesis, batch)
                futures.append(future)
            
            # Wait for all batches to complete
            for future in futures:
                total_sent += future.result()
        
        elapsed = time.time() - start_time
        rate = total_sent / elapsed
        
        print(f"✅ Sent {total_sent:,} events in {elapsed:.2f}s ({rate:,.0f} events/sec)")
        
        # Sleep to maintain 1-minute interval
        sleep_time = max(0, 60 - elapsed)
        if sleep_time > 0:
            time.sleep(sleep_time)

if __name__ == '__main__':
    run_simulator()
```

**Step 3: Real-Time Analytics with Kinesis Data Analytics**

```sql
-- Create Kinesis Data Analytics application
-- SQL query runs continuously on streaming data

-- Create input stream (from Kinesis Data Streams)
CREATE OR REPLACE STREAM "SOURCE_SQL_STREAM_001" (
    sensor_id VARCHAR(16),
    temperature DOUBLE,
    humidity DOUBLE,
    pressure DOUBLE,
    timestamp TIMESTAMP,
    location VARCHAR(128)
);

-- Create output stream for aggregated metrics
CREATE OR REPLACE STREAM "DESTINATION_SQL_STREAM" (
    window_start TIMESTAMP,
    window_end TIMESTAMP,
    avg_temperature DOUBLE,
    max_temperature DOUBLE,
    min_temperature DOUBLE,
    avg_humidity DOUBLE,
    sensor_count BIGINT,
    high_temp_alert_count BIGINT
);

-- Sliding window aggregation (1-minute tumbling windows)
CREATE OR REPLACE PUMP "STREAM_PUMP" AS
INSERT INTO "DESTINATION_SQL_STREAM"
SELECT STREAM
    STEP("SOURCE_SQL_STREAM_001".ROWTIME BY INTERVAL '1' MINUTE) AS window_start,
    STEP("SOURCE_SQL_STREAM_001".ROWTIME BY INTERVAL '1' MINUTE) + INTERVAL '1' MINUTE AS window_end,
    AVG(temperature) AS avg_temperature,
    MAX(temperature) AS max_temperature,
    MIN(temperature) AS min_temperature,
    AVG(humidity) AS avg_humidity,
    COUNT(*) AS sensor_count,
    SUM(CASE WHEN temperature > 80 THEN 1 ELSE 0 END) AS high_temp_alert_count
FROM "SOURCE_SQL_STREAM_001"
GROUP BY
    STEP("SOURCE_SQL_STREAM_001".ROWTIME BY INTERVAL '1' MINUTE);

-- Create anomaly detection stream
CREATE OR REPLACE STREAM "ANOMALY_STREAM" (
    sensor_id VARCHAR(16),
    temperature DOUBLE,
    avg_temperature DOUBLE,
    deviation DOUBLE,
    timestamp TIMESTAMP
);

-- Detect anomalies (temperature > 2 std deviations from avg)
CREATE OR REPLACE PUMP "ANOMALY_PUMP" AS
INSERT INTO "ANOMALY_STREAM"
SELECT STREAM
    sensor_id,
    temperature,
    AVG(temperature) OVER W AS avg_temperature,
    (temperature - AVG(temperature) OVER W) AS deviation,
    timestamp
FROM "SOURCE_SQL_STREAM_001"
WHERE
    ABS(temperature - AVG(temperature) OVER W) > 2 * STDDEV_SAMP(temperature) OVER W
WINDOW W AS (
    PARTITION BY sensor_id
    RANGE INTERVAL '10' MINUTE PRECEDING
);
```

**Step 4: Lambda Consumer for Alerts**

```python
# lambda_kinesis_consumer.py
import json
import boto3

sns = boto3.client('sns')
SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:123456789012:IoT-High-Temp-Alerts'

def lambda_handler(event, context):
    """
    Process Kinesis records and send alerts for high temperatures
    """
    
    high_temp_threshold = 80.0
    alerts = []
    
    for record in event['Records']:
        # Decode Kinesis record
        payload = json.loads(record['kinesis']['data'])
        
        sensor_id = payload['sensor_id']
        temperature = payload['temperature']
        timestamp = payload['timestamp']
        
        # Check for high temperature
        if temperature > high_temp_threshold:
            alert = {
                'sensor_id': sensor_id,
                'temperature': temperature,
                'threshold': high_temp_threshold,
                'timestamp': timestamp,
                'severity': 'CRITICAL' if temperature > 90 else 'WARNING'
            }
            alerts.append(alert)
    
    # Send SNS notification if alerts exist
    if alerts:
        message = f"🚨 High Temperature Alert\n\n"
        message += f"Total sensors affected: {len(alerts)}\n\n"
        
        for alert in alerts[:10]:  # First 10 alerts
            message += f"• {alert['sensor_id']}: {alert['temperature']}°C ({alert['severity']})\n"
        
        if len(alerts) > 10:
            message += f"\n... and {len(alerts) - 10} more sensors"
        
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject=f"High Temperature Alert - {len(alerts)} sensors",
            Message=message
        )
        
        print(f"✅ Sent alert for {len(alerts)} sensors")
    
    return {
        'statusCode': 200,
        'recordsProcessed': len(event['Records']),
        'alertsSent': len(alerts)
    }
```

**Step 5: Archive to S3 with Kinesis Data Firehose**

```bash
# Create Kinesis Data Firehose delivery stream
aws firehose create-delivery-stream \
  --delivery-stream-name iot-sensor-firehose \
  --delivery-stream-type KinesisStreamAsSource \
  --kinesis-stream-source-configuration '{
    "KinesisStreamARN": "arn:aws:kinesis:us-east-1:123456789012:stream/iot-sensor-stream",
    "RoleARN": "arn:aws:iam::123456789012:role/FirehoseKinesisRole"
  }' \
  --extended-s3-destination-configuration '{
    "BucketARN": "arn:aws:s3:::iot-sensor-archive",
    "Prefix": "year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
    "ErrorOutputPrefix": "errors/",
    "BufferingHints": {
      "SizeInMBs": 128,
      "IntervalInSeconds": 300
    },
    "CompressionFormat": "GZIP",
    "DataFormatConversionConfiguration": {
      "SchemaConfiguration": {
        "DatabaseName": "iot_db",
        "TableName": "sensor_events",
        "Region": "us-east-1"
      },
      "OutputFormatConfiguration": {
        "Serializer": {
          "ParquetSerDe": {}
        }
      }
    },
    "RoleARN": "arn:aws:iam::123456789012:role/FirehoseS3Role"
  }'

# Result: JSON streams automatically converted to Parquet and stored in S3
# - Buffering: Every 5 minutes or 128 MB (whichever comes first)
# - Format: Parquet (10x smaller than JSON)
# - Partitioning: year/month/day/hour (optimized for Athena queries)
```

**Results:**

| Metric | Value |
|--------|-------|
| **Ingestion Rate** | 1,000,000 events/minute (16,667/second) |
| **Processing Latency** | < 1 second (real-time) |
| **Lambda Invocations** | 100,000/minute (batched, 10 events/invocation) |
| **Storage** | 50 GB/day (Parquet compressed) |
| **Query Performance** | 3 seconds (Athena on archived data) |

**Cost Analysis:**

```
Daily Cost (1M events/minute):

Kinesis Data Streams:
├─ 100 shards × $0.015/shard/hour × 24 hours = $36/day
├─ PUT payload: 1.44B records/day × $0.014/1M = $20.16/day
└─ Total: $56.16/day

Kinesis Data Analytics:
├─ 1 KPU (Kinesis Processing Unit) × $0.11/hour × 24 = $2.64/day

Kinesis Data Firehose:
├─ 1.44B records/day × $0.029/1M = $41.76/day

Lambda (alerts):
├─ 100K invocations/minute × 60 min × 24 hr = 144M/day
├─ Duration: 100ms avg, 512 MB memory
└─ Cost: 144M × 0.1s × $0.0000166667/GB-sec × 0.5 GB = $12/day

S3 Storage:
├─ 50 GB/day × 30 days = 1.5 TB/month
└─ Cost: $34.50/month ($1.15/day)

Total Daily Cost: $113.71/day = $3,411/month

vs Building Custom Kafka/Spark Streaming:
├─ EC2 instances (Kafka cluster): $500/month
├─ EC2 instances (Spark streaming): $800/month
├─ Operational overhead: $5,000/month (DevOps time)
└─ Total: $6,300/month

Kinesis Savings: $2,889/month (46% cheaper) + zero ops ✅
```

---

(Continue with Exercise 7.3 and Q1-Q20...)

## EXERCISE 7.3: BUSINESS INTELLIGENCE WITH QUICKSIGHT

**Scenario:** Create interactive dashboards for executives to visualize key business metrics from the data lake.

**Implementation:**

```bash
# 1. Create QuickSight account
aws quicksight create-account-subscription \
  --edition ENTERPRISE \
  --authentication-method IAM_AND_QUICKSIGHT \
  --aws-account-id 123456789012 \
  --account-name "DataEngineering"

# 2. Grant QuickSight access to S3 and Athena
aws quicksight create-data-source \
  --aws-account-id 123456789012 \
  --data-source-id athena-clickstream \
  --name "Clickstream Data Lake" \
  --type ATHENA \
  --data-source-parameters '{
    "AthenaParameters": {
      "WorkGroup": "analytics_team"
    }
  }'

# 3. Create dataset from Athena table
# Dashboard: Daily Active Users, Revenue, Conversion Rates
# Auto-refresh: Every 1 hour
# SPICE: In-memory cache for sub-second queries
```

**Key Dashboards:**
- **Executive Summary:** DAU, revenue, conversion rate trends
- **Product Analytics:** Top pages, user journeys, bounce rates
- **Real-Time Monitoring:** Current active users, events/second

**Cost:** $24/user/month (vs $70/user Tableau, 66% cheaper)

---

## BEGINNER QUESTIONS (Q1-Q5)

**Q1: What is Amazon Athena and when should you use it?**

**Answer:**

Amazon Athena is a **serverless, interactive query service** that makes it easy to analyze data in S3 using standard SQL.

**Key Features:**
- **Serverless:** No infrastructure to provision or manage
- **Pay per query:** $5 per TB of data scanned (no fixed costs)
- **Presto-based:** Supports complex SQL queries, JOINs, window functions
- **Integrated:** Works with Glue Data Catalog, S3, QuickSight

**When to use Athena:**

1. **Ad-hoc SQL queries on S3 data lake**
   - Data analysts exploring datasets
   - One-time reports and investigations
   - Cost: $0.05 for 10 GB query

2. **Infrequent querying (< 100 queries/day)**
   - Better than maintaining a data warehouse
   - No idle costs (vs Redshift 24/7)

3. **Log analysis and forensics**
   - CloudTrail logs, VPC Flow Logs, ALB access logs
   - Query S3 directly without loading to database

4. **Data lake exploration**
   - Discover and profile data before building pipelines
   - Schema-on-read (flexible, no pre-defined schema)

**When NOT to use Athena:**

| Limitation | Alternative |
|-----------|-------------|
| High query frequency (1,000s/day) | Redshift (fixed cost better) |
| Needs caching/materialized views | Redshift, RDS |
| Sub-second latency required | Redshift, DynamoDB |
| Complex ETL transformations | Glue, EMR Spark |
| Streaming data | Kinesis Data Analytics |

**Cost Comparison (100 queries/day on 1 TB dataset):**

```
Athena:
├─ 100 queries × 50 GB avg × $0.005/GB × 30 days = $750/month
└─ No storage cost (S3 billed separately)

Redshift (dc2.large, 3 nodes):
├─ $0.25/hour × 3 nodes × 730 hours = $548/month
├─ Storage: 1 TB × $24/TB = $24/month
└─ Total: $572/month

Winner: Redshift for high-frequency queries
Athena wins for < 50 queries/day ($187 breakeven)
```

---

**Q2: How does AWS Glue Data Catalog work?**

**Answer:**

AWS Glue Data Catalog is a **central metadata repository** that stores table definitions, schema, and partition information for data lakes.

**Components:**

```
Glue Data Catalog
├─ Databases: Logical grouping of tables
│  └─ Example: sales_db, clickstream_db, logs_db
│
├─ Tables: Metadata about datasets
│  ├─ Name: customer_orders
│  ├─ Location: s3://datalake/orders/
│  ├─ Schema: [order_id INT, customer_id INT, amount DECIMAL(10,2)]
│  ├─ Partitions: year/month/day
│  └─ Format: Parquet, CSV, JSON, Avro
│
└─ Connections: Database credentials (RDS, Redshift, JDBC)
```

**How Glue Crawlers Work:**

```python
# 1. Crawler scans S3 path
s3://datalake/orders/
  └─ year=2024/month=06/day=22/
      └─ orders_20240622.parquet

# 2. Infers schema from data
# - Reads sample files
# - Detects data types (INT, STRING, TIMESTAMP)
# - Identifies partitions (year, month, day)

# 3. Creates/updates table in Glue Catalog
Table: orders
Columns: order_id, customer_id, amount, order_date
Partitions: year, month, day
Format: Parquet
```

**Integration with Analytics Services:**

| Service | How It Uses Glue Catalog |
|---------|--------------------------|
| **Athena** | Query tables defined in catalog |
| **Redshift Spectrum** | External tables from catalog |
| **EMR** | Hive metastore backed by catalog |
| **Glue ETL** | Source/target metadata |
| **QuickSight** | Data source discovery |

**Benefits:**

1. **Single source of truth** - All services use same metadata
2. **Schema evolution** - Crawler detects new columns automatically
3. **Partition discovery** - Auto-adds new partitions daily
4. **Cost:** $1 per 100,000 objects accessed by crawler/month

---

**Q3: What is Amazon Kinesis and what are its components?**

**Answer:**

Amazon Kinesis is a platform for **streaming data** on AWS, offering four services:

**1. Kinesis Data Streams** (Real-Time Streaming)

```
Use Case: Ingest and store streaming data for custom processing

Features:
├─ Shards: 1,000 records/sec write, 2 MB/sec read per shard
├─ Retention: 24 hours default (up to 365 days)
├─ Consumers: Lambda, Kinesis Data Analytics, custom apps
└─ Cost: $0.015/shard/hour + $0.014 per 1M PUT requests

Example: IoT sensor data → Kinesis → Lambda → Real-time alerts
```

**2. Kinesis Data Firehose** (Load to Data Stores)

```
Use Case: Deliver streaming data to S3, Redshift, Elasticsearch

Features:
├─ Serverless: No provisioning, auto-scaling
├─ Buffering: Batch records (1-5 min or 1-128 MB)
├─ Transformation: Lambda function to transform data
├─ Destinations: S3, Redshift, Elasticsearch, HTTP endpoints
└─ Cost: $0.029 per GB ingested

Example: App logs → Firehose → S3 (Parquet) → Athena
```

**3. Kinesis Data Analytics** (SQL on Streams)

```
Use Case: Real-time analytics using SQL

Features:
├─ SQL queries: Run continuously on streaming data
├─ Windows: Tumbling, sliding, session windows
├─ Anomaly detection: Built-in ML algorithms
├─ Output: Kinesis Data Streams, Firehose, Lambda
└─ Cost: $0.11 per Kinesis Processing Unit (KPU) per hour

Example: Click stream → Analytics (aggregate) → CloudWatch metrics
```

**4. Kinesis Video Streams** (Video Streaming)

```
Use Case: Ingest video from devices (security cameras, drones)

Features:
├─ SDK: Stream video from devices
├─ Storage: Encrypted, durable
├─ Processing: Rekognition Video, custom apps
└─ Cost: $0.0085 per GB ingested
```

**Decision Matrix:**

| Requirement | Choose |
|-------------|--------|
| Custom real-time processing | Data Streams + Lambda/KCL |
| Load to S3/Redshift | Data Firehose |
| Real-time SQL analytics | Data Analytics |
| Need retention > 24 hours | Data Streams (up to 365 days) |
| Serverless, hands-off | Data Firehose |

---

**Q4: How do you optimize Athena query performance and costs?**

**Answer:**

**Cost Optimization Strategies:**

**1. Partitioning** (Reduce data scanned)

```sql
-- Bad: Full table scan (1 PB)
SELECT COUNT(*) FROM events;
-- Cost: $5,000 ❌

-- Good: Partition pruning (10 GB)
SELECT COUNT(*) FROM events
WHERE year=2024 AND month=6 AND day=22;
-- Cost: $0.05 ✅ (99.999% cheaper)
```

**2. Columnar Formats** (10x compression)

```
CSV (100 GB) → Parquet (10 GB)
- Storage: 90% cheaper
- Query: 5x faster (columnar read)
- Cost: 10 GB × $0.005 vs 100 GB × $0.005 (10x cheaper)
```

**3. Compression** (GZIP, Snappy)

```
Parquet uncompressed: 50 GB
Parquet + Snappy: 10 GB (5x smaller)
- Query cost: $0.05 vs $0.25 (80% cheaper)
```

**4. Column Projection** (SELECT only needed columns)

```sql
-- Bad: SELECT * (scans all columns)
SELECT * FROM events WHERE year=2024;
-- Data scanned: 100 GB

-- Good: SELECT specific columns
SELECT user_id, event_type, timestamp
FROM events WHERE year=2024;
-- Data scanned: 15 GB (85% cheaper)
```

**5. Filter Early** (Predicate pushdown)

```sql
-- Bad: Filter after JOIN
SELECT a.user_id, b.order_id
FROM events a
JOIN orders b ON a.user_id = b.customer_id
WHERE a.event_type = 'purchase';
-- Scans: 1 TB (events) + 500 GB (orders) = 1.5 TB

-- Good: Filter before JOIN
SELECT a.user_id, b.order_id
FROM (
  SELECT user_id FROM events
  WHERE event_type = 'purchase'
) a
JOIN orders b ON a.user_id = b.customer_id;
-- Scans: 100 GB (filtered events) + 500 GB (orders) = 600 GB (60% cheaper)
```

**6. Use CTAS** (Create Table As Select for intermediate results)

```sql
-- Cache intermediate result for reuse
CREATE TABLE filtered_events AS
SELECT user_id, event_type, timestamp
FROM events
WHERE year=2024 AND event_type IN ('purchase', 'checkout');

-- Subsequent queries use cached result (no rescan)
SELECT event_type, COUNT(*) FROM filtered_events GROUP BY 1;
-- Cost: $0 (reads CTAS table, already paid) ✅
```

**Performance Optimization:**

| Technique | Impact |
|-----------|--------|
| **Add partitions** | 10-100x faster |
| **Use Parquet** | 5x faster reads |
| **Compress data** | 3x faster network transfer |
| **Use columnar formats** | 10x faster aggregations |
| **Avoid SELECT *** | 2-10x faster (less data) |

---

**Q5: What is AWS Lake Formation and how does it simplify data lakes?**

**Answer:**

AWS Lake Formation is a service that simplifies **building, securing, and managing data lakes**.

**Key Features:**

**1. Centralized Security (Fine-Grained Access Control)**

```
Traditional Data Lake Security:
├─ S3 bucket policies (broad, bucket-level)
├─ IAM policies (complex, hard to manage)
└─ Problem: Can't grant column or row-level access

Lake Formation Security:
├─ Grant access to specific databases/tables
├─ Column-level: Hide sensitive columns (SSN, credit card)
├─ Row-level: Filter data by user (region='US' for US analysts)
└─ Centralized: Single place to manage all permissions
```

**2. Data Catalog & Discovery**

```python
# Lake Formation automatically populates Glue Catalog
# - Crawlers discover data
# - Tag sensitive columns (PII)
# - Track data lineage (where data came from)
```

**3. Data Access Audit**

```
CloudTrail Integration:
- Who accessed what data?
- When and from where?
- Compliance reports (GDPR, HIPAA)
```

**Example: Row-Level Security**

```sql
-- Grant US analysts access to US data only
GRANT SELECT ON TABLE sales
TO ROLE us_analysts
WITH GRANT OPTION (
  ROW FILTER region = 'US'
);

-- When US analyst queries:
SELECT * FROM sales;

-- Lake Formation automatically adds filter:
SELECT * FROM sales WHERE region = 'US';
-- Analyst never sees non-US data ✅
```

**Benefits:**

| Without Lake Formation | With Lake Formation |
|------------------------|---------------------|
| IAM policies (100s of lines) | Simple UI grants |
| Bucket-level access only | Table/column/row-level |
| Manual audit trail | Automated CloudTrail logs |
| No data lineage | Track data origin |
| Complex compliance | One-click GDPR reports |

**Cost:** No additional charge (uses Glue Catalog pricing)

---

## INTERMEDIATE QUESTIONS (Q6-Q10)

**Q6: How do you implement incremental data processing with Athena?**

**Answer:**

**Challenge:** Avoid re-scanning entire dataset daily, process only new data.

**Solution: Partition Pruning + Manifest Files**

```sql
-- Daily incremental pattern

-- Step 1: Create base table with partitions
CREATE EXTERNAL TABLE events (
  user_id BIGINT,
  event_type STRING,
  timestamp TIMESTAMP
)
PARTITIONED BY (year INT, month INT, day INT)
STORED AS PARQUET
LOCATION 's3://datalake/events/';

-- Step 2: Add today's partition only
ALTER TABLE events ADD PARTITION (year=2024, month=6, day=22)
LOCATION 's3://datalake/events/year=2024/month=06/day=22/';

-- Step 3: Query only new partition
SELECT event_type, COUNT(*) as count
FROM events
WHERE year=2024 AND month=6 AND day=22
GROUP BY event_type;

-- Data scanned: 10 GB (today only)
-- Cost: $0.05 ✅ (vs $5,000 for full scan)
```

**Incremental ETL Pattern:**

```sql
-- Create incremental aggregation table
CREATE TABLE daily_metrics AS
SELECT
  year, month, day,
  COUNT(DISTINCT user_id) as dau,
  SUM(CASE WHEN event_type='purchase' THEN 1 ELSE 0 END) as purchases
FROM events
WHERE year=2024 AND month=6 AND day=22
GROUP BY year, month, day;

-- Next day: INSERT new day's metrics
INSERT INTO daily_metrics
SELECT year, month, day, COUNT(DISTINCT user_id), ...
FROM events
WHERE year=2024 AND month=6 AND day=23
GROUP BY year, month, day;

-- Query aggregated table (fast, cheap)
SELECT * FROM daily_metrics
WHERE year=2024 AND month=6
ORDER BY day;
-- Scans: 100 KB (aggregated) vs 300 GB (raw events)
```

---

**Q7: What are the differences between Kinesis Data Streams and Data Firehose?**

**Answer:**

| Feature | Data Streams | Data Firehose |
|---------|-------------|---------------|
| **Use Case** | Custom real-time processing | Load to S3/Redshift/ES |
| **Provisioning** | Manual (shard count) | Serverless (auto-scale) |
| **Retention** | 24 hours - 365 days | None (immediate delivery) |
| **Consumers** | Multiple (Lambda, KCL, Analytics) | Single (destination) |
| **Processing** | Real-time (< 1 sec) | Near real-time (60-900 sec buffer) |
| **Transformation** | External (Lambda) | Built-in (Lambda optional) |
| **Replay** | Yes (retention period) | No |
| **Cost** | $0.015/shard/hour | $0.029/GB ingested |
| **Management** | Monitor shards, scale manually | Zero management |

**When to use Data Streams:**
- Need multiple consumers reading same data
- Require replay capability
- Sub-second latency needed
- Custom processing logic

**When to use Data Firehose:**
- Simple pipeline: Stream → S3/Redshift
- Don't need replay
- Want zero management
- 1-5 minute latency acceptable

**Hybrid Pattern:**

```
Click Stream
  │
  ▼
Data Streams (retention: 7 days)
  │
  ├──→ Lambda (real-time alerts) ← Consumer 1
  ├──→ Data Analytics (SQL) ← Consumer 2
  └──→ Data Firehose (S3 archive) ← Consumer 3
```

---

**Q8: How do you secure data access in Athena and QuickSight?**

**Answer:**

**Multi-Layer Security:**

**1. S3 Bucket Encryption**

```bash
# Encrypt at rest with KMS
aws s3api put-bucket-encryption \
  --bucket datalake \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123:key/abc"
      }
    }]
  }'
```

**2. IAM Policies for Athena**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults"
      ],
      "Resource": "arn:aws:athena:*:*:workgroup/analytics-team"
    },
    {
      "Effect": "Allow",
      "Action": ["glue:GetTable", "glue:GetDatabase"],
      "Resource": [
        "arn:aws:glue:*:*:catalog",
        "arn:aws:glue:*:*:database/clickstream_db",
        "arn:aws:glue:*:*:table/clickstream_db/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::datalake/clickstream/*"
    }
  ]
}
```

**3. Lake Formation Column-Level Security**

```sql
-- Hide PII columns from analysts
GRANT SELECT (user_id, event_type, timestamp)
ON TABLE events
TO ROLE analysts;

-- Grant full access to data engineers
GRANT ALL
ON TABLE events
TO ROLE data_engineers;
```

**4. QuickSight Row-Level Security**

```
-- RLS dataset: user_email_to_region.csv
email,region
analyst1@company.com,US
analyst2@company.com,EU

-- In QuickSight dataset:
- Add RLS dataset
- Join on user email
- Filter: sales.region = rls.region

-- Result: Each analyst sees only their region's data
```

**5. Athena Workgroup Query Limits**

```bash
# Prevent runaway queries
aws athena create-work-group \
  --name cost-controlled \
  --configuration '{
    "BytesScannedCutoffPerQuery": 100000000000,
    "EnforceWorkGroupConfiguration": true
  }'

# Queries scanning > 100 GB are automatically cancelled
```

---

**Q9: How do you optimize Kinesis shard count for variable load?**

**Answer:**

**Shard Capacity:**
- **Write:** 1,000 records/sec OR 1 MB/sec per shard
- **Read:** 2 MB/sec per shard (5 consumers max with standard iterator)

**Calculation:**

```python
# Current load: 50,000 records/sec, 10 MB/sec

# By records: 50,000 / 1,000 = 50 shards
# By throughput: 10 MB/sec / 1 MB/sec = 10 shards

# Required: MAX(50, 10) = 50 shards

# Future-proofing (2x buffer): 50 × 2 = 100 shards
```

**Auto-Scaling with Application Auto Scaling:**

```bash
# Register stream as scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace kinesis \
  --resource-id streamName/iot-sensor-stream \
  --scalable-dimension kinesis:stream:WriteCapacityUnits \
  --min-capacity 10 \
  --max-capacity 500

# Create scaling policy
aws application-autoscaling put-scaling-policy \
  --service-namespace kinesis \
  --resource-id streamName/iot-sensor-stream \
  --scalable-dimension kinesis:stream:WriteCapacityUnits \
  --policy-name IncomingRecordsScaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 75.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "KinesisStreamIncomingRecords"
    }
  }'
```

**Monitoring:**

```python
# CloudWatch metrics to watch
metrics_to_monitor = [
    "IncomingRecords",        # Records/sec
    "IncomingBytes",          # Bytes/sec
    "WriteProvisionedThroughputExceeded",  # Throttling
    "GetRecords.IteratorAgeMilliseconds"   # Consumer lag
]

# Alert thresholds:
# - WriteProvisionedThroughputExceeded > 0: Add shards
# - IteratorAge > 60000ms: Consumers can't keep up
```

---

**Q10: How do you migrate from self-managed Hadoop/Spark to AWS analytics services?**

**Answer:**

**Migration Path:**

```
On-Premises Hadoop Cluster
├─ HDFS storage (500 TB)
├─ Hive tables (SQL queries)
├─ Spark jobs (ETL)
└─ Presto (ad-hoc queries)

↓ Migrate to AWS

AWS Analytics Stack
├─ Amazon S3 (data lake, 500 TB)
├─ AWS Glue Catalog (Hive metastore)
├─ Amazon EMR (Spark jobs)
├─ Amazon Athena (ad-hoc queries)
└─ AWS Glue (managed ETL)
```

**Step-by-Step Migration:**

**Phase 1: Data Migration (HDFS → S3)**

```bash
# Use S3DistCp (distributed copy from HDFS to S3)
s3-dist-cp \
  --src hdfs://namenode:9000/data \
  --dest s3://datalake/migrated-data \
  --srcPattern '.*parquet' \
  --groupBy '.*/([^/]+)/.*' \
  --targetSize 512

# Parallel copy: 500 TB in 3 days (vs 30 days sequential)
```

**Phase 2: Metadata Migration (Hive → Glue Catalog)**

```bash
# Export Hive metastore
beeline -u jdbc:hive2://localhost:10000 \
  -e "SHOW CREATE TABLE sales" > sales_ddl.sql

# Import to Glue Catalog
aws glue create-table --database-input '{
  "Name": "sales",
  "StorageDescriptor": {
    "Location": "s3://datalake/sales/",
    "InputFormat": "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat",
    ...
  }
}'
```

**Phase 3: Query Migration (Hive SQL → Athena)**

```sql
-- Most Hive SQL works in Athena (Presto-based)
-- Minor syntax changes:

-- Hive:
SELECT collect_list(user_id) FROM events;

-- Athena (Presto):
SELECT array_agg(user_id) FROM events;
```

**Phase 4: ETL Migration (Spark → Glue/EMR)**

```
Option 1: AWS Glue (serverless)
- Simple Spark jobs
- Cost: $0.44/DPU-hour
- Benefit: Zero management

Option 2: EMR (more control)
- Complex Spark jobs
- Cost: EC2 instance pricing
- Benefit: Full Spark API, custom JARs
```

**Cost Comparison:**

```
On-Premises Hadoop (3-year TCO):
├─ Hardware (50 nodes): $500,000
├─ Power/cooling: $150,000
├─ Staff (3 FTE): $450,000
└─ Total: $1,100,000

AWS Analytics (3-year):
├─ S3 storage (500 TB): $414,000
├─ Athena queries: $60,000
├─ EMR (transient): $120,000
└─ Total: $594,000

Savings: $506,000 (46%) ✅
```

---

## SCENARIO-BASED QUESTIONS (Q11-Q20)

**Q11: Build a Unified Data Lake for Multi-Tenant SaaS Platform**

**Scenario:** 1,000 customers, each with isolated data, need unified analytics while maintaining security.

**Solution:**
- S3: One bucket with customer_id partitioning
- Lake Formation: Row-level security (each customer sees only their data)
- Athena: Shared queries with automatic row filtering
- Cost: $50K/year (vs $500K for 1,000 separate databases)

---

**Q12: Real-Time Fraud Detection with Kinesis + Lambda**

**Scenario:** Detect fraudulent transactions within 100ms of occurrence.

**Architecture:** Transaction stream → Kinesis → Lambda (ML model) → DynamoDB (flagged transactions) → SNS alert

**Performance:** 50,000 transactions/sec, 95ms p99 latency

---

**Q13: Cost-Optimize 10 PB Data Lake (95% Savings)**

**Scenario:** Reduce storage costs from $300K/month to $15K/month.

**Strategy:**
- Hot data (30 days): S3 Standard ($23/TB) = $6,900
- Warm data (90 days): S3 Intelligent-Tiering ($10/TB) = $3,000
- Cold data (1 year+): S3 Glacier Deep Archive ($1/TB) = $5,000
- Total: $14,900/month (95% savings) ✅

---

**Q14: Streaming ETL with Kinesis Data Analytics (SQL)**

**Scenario:** Aggregate 1M events/sec using SQL on streaming data.

**Implementation:** Kinesis stream → Data Analytics (tumbling windows, aggregations) → Destination stream → S3

**Benefit:** Real-time dashboards, no batch delay

---

**Q15: Athena Federated Query (Join S3 + RDS + DynamoDB)**

**Scenario:** Join data lake (S3) with operational databases (RDS, DynamoDB) in single query.

**Solution:** Athena Federated Query connectors

```sql
SELECT
  s3.user_id,
  rds.user_name,
  ddb.cart_items
FROM "datalake".events s3
JOIN "rds".users rds ON s3.user_id = rds.id
JOIN "dynamodb".carts ddb ON s3.session_id = ddb.session_id;
```

---

**Q16: QuickSight Embedded Analytics (White-Label BI)**

**Scenario:** Embed dashboards in SaaS application for customers.

**Implementation:** QuickSight + RLS + Embedded dashboard API

**Cost:** $0.30/session (vs $70/user Tableau Embedded)

---

**Q17: GDPR Compliance with Lake Formation**

**Scenario:** Automatically delete user data on request (GDPR right to be forgotten).

**Solution:**
- Tag PII columns in Glue Catalog
- Lake Formation tracks data lineage
- Lambda triggered on deletion request → S3 object deletion + Glue partition drop

---

**Q18: High-Performance Analytics (Sub-Second Queries)**

**Scenario:** Athena too slow for interactive dashboards (10-second queries).

**Solution:** Athena + CTAS + QuickSight SPICE (in-memory cache)

**Result:** 200ms queries (50x faster)

---

**Q19: Streaming Data Quality Checks**

**Scenario:** Validate streaming data for schema compliance before storage.

**Implementation:** Kinesis Firehose + Lambda transformation → reject invalid records → DLQ (S3 errors bucket)

**Data quality:** 99.9% valid (0.1% to DLQ for investigation)

---

**Q20: Hybrid Analytics (On-Prem + Cloud)**

**Scenario:** Some data must stay on-premises (compliance), query with cloud data.

**Solution:** Athena Federated Query + Direct Connect + On-prem connector

**Architecture:** Athena queries S3 (cloud) + on-prem PostgreSQL (via connector) in single SQL statement

---

## MODULE SUMMARY

**Module 7: Analytics** covered AWS analytics services for modern data lakes:

**Services Mastered:**
- ✅ Amazon Athena (serverless SQL, $5/TB)
- ✅ AWS Glue (data catalog, ETL)
- ✅ Amazon Kinesis (real-time streaming)
- ✅ Amazon QuickSight (BI dashboards)
- ✅ AWS Lake Formation (data lake governance)

**Key Achievements:**
- **Cost Savings:** 99.99% (partitioning), 95% (lifecycle policies)
- **Performance:** 10x faster (Parquet), sub-second (SPICE)
- **Scale:** 1M events/sec (Kinesis), 1 PB queries (Athena)
- **Security:** Column/row-level (Lake Formation)

**Next Module:** Module 8: Application Integration (EventBridge, SQS, SNS, Step Functions)

---

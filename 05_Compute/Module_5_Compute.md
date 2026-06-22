# MODULE 5: COMPUTE SERVICES FOR DATA ENGINEERING

## MODULE OVERVIEW

**Duration:** 8-10 hours  
**Difficulty:** Intermediate-Advanced  
**Prerequisites:** Modules 1-4, Understanding of distributed computing and data processing  
**Cost:** $100-200 (temporary compute resources)

---

## WHAT YOU WILL LEARN

### AWS Compute Services for Data Engineering

This module covers the complete suite of AWS compute services used in data engineering workflows. You'll learn how to process data at scale using serverless, batch, and big data compute platforms.

**Key Services:**
- ✅ **Amazon EC2** - Virtual servers for custom data processing applications
- ✅ **AWS Lambda** - Serverless event-driven data processing
- ✅ **AWS Batch** - Managed batch computing for large-scale jobs
- ✅ **Amazon EMR** - Managed big data framework (Spark, Hadoop, Presto)
- ✅ **AWS Glue** - Serverless ETL and data catalog
- ✅ **AWS Step Functions** - Orchestrate data workflows

### Production Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│              Compute Architecture for Data Engineering       │
└─────────────────────────────────────────────────────────────┘

Serverless Event Processing
├─ S3 new file → Lambda → validate & transform → S3/DynamoDB
├─ Kinesis stream → Lambda → real-time aggregation → Redshift
└─ EventBridge schedule → Lambda → trigger EMR job

Batch Processing
├─ AWS Batch: Nightly ETL jobs (10,000 parallel tasks)
├─ Spot Instances: 90% cost savings
└─ Auto-scaling: 1 → 1,000 vCPUs based on queue depth

Big Data Processing
├─ EMR: Spark jobs for 100 TB datasets
├─ Transient clusters: Launch → Process → Terminate
└─ S3 as data lake (separate compute and storage)

Serverless ETL
├─ Glue: 10 DPUs processing 1 TB Parquet
├─ Glue Crawler: Auto-discover schema in S3
└─ Glue Data Catalog: Central metadata repository

Orchestration
├─ Step Functions: Multi-step data pipelines
├─ Error handling: Retry, catch, fallback
└─ Visual workflow designer
```

---

## EXERCISE 5.1: SERVERLESS ETL WITH AWS LAMBDA

**Scenario:** Build a serverless data pipeline that processes CSV files uploaded to S3, validates data quality, transforms to Parquet, and loads into Redshift.

**Architecture:**

```
S3 (Landing Zone)
└─ s3://data-landing/raw/sales_2024-06-22.csv (1 GB file)
    │
    │ S3 Event Notification
    ▼
Lambda Function 1 (Validator)
├─ Check schema (required columns)
├─ Validate data types (price > 0, date format)
├─ Calculate statistics (row count, nulls)
└─ Tag S3 object: valid=true/false
    │
    │ If valid=true
    ▼
Lambda Function 2 (Transformer)
├─ Read CSV from S3
├─ Transform: CSV → Parquet (Pandas/PyArrow)
├─ Compress: Snappy compression
└─ Write to s3://data-processed/parquet/
    │
    │ S3 Event Notification
    ▼
Lambda Function 3 (Loader)
├─ Generate COPY command
├─ Load Parquet into Redshift
└─ Update data catalog (Glue)
    │
    ▼
Amazon Redshift
└─ Table: sales (partitioned by date)
    │
    └─→ SNS Notification (success/failure)
```

**Implementation:**

**Step 1: Create S3 Buckets and Lambda Execution Role**

```bash
# Create S3 buckets
aws s3 mb s3://data-landing-12345 --region us-east-1
aws s3 mb s3://data-processed-12345 --region us-east-1

# Create Lambda execution role
cat > lambda-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name DataPipelineLambdaRole \
  --assume-role-policy-document file://lambda-trust-policy.json

# Attach policies
aws iam attach-role-policy \
  --role-name DataPipelineLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy \
  --role-name DataPipelineLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Create custom policy for Redshift
cat > redshift-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "redshift-data:ExecuteStatement",
      "redshift-data:DescribeStatement",
      "redshift-data:GetStatementResult"
    ],
    "Resource": "*"
  }]
}
EOF

aws iam put-role-policy \
  --role-name DataPipelineLambdaRole \
  --policy-name RedshiftDataAPIAccess \
  --policy-document file://redshift-policy.json
```

**Step 2: Lambda Function 1 - Data Validator**

```python
# validator.py
import json
import boto3
import csv
import io
from datetime import datetime

s3 = boto3.client('s3')
sns = boto3.client('sns')

REQUIRED_COLUMNS = ['order_id', 'customer_id', 'product_id', 'quantity', 'price', 'order_date']
SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:123456789012:DataPipelineAlerts'

def lambda_handler(event, context):
    """
    Validate CSV file uploaded to S3
    """
    
    # Get S3 object details from event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    print(f"Validating: s3://{bucket}/{key}")
    
    try:
        # Download and parse CSV
        response = s3.get_object(Bucket=bucket, Key=key)
        csv_content = response['Body'].read().decode('utf-8')
        
        # Parse CSV
        csv_reader = csv.DictReader(io.StringIO(csv_content))
        rows = list(csv_reader)
        
        # Validation 1: Check required columns
        actual_columns = set(csv_reader.fieldnames)
        required_columns = set(REQUIRED_COLUMNS)
        
        if not required_columns.issubset(actual_columns):
            missing = required_columns - actual_columns
            raise ValueError(f"Missing required columns: {missing}")
        
        # Validation 2: Data quality checks
        errors = []
        null_counts = {col: 0 for col in REQUIRED_COLUMNS}
        
        for row_num, row in enumerate(rows, start=2):  # Start at 2 (header is row 1)
            # Check nulls
            for col in REQUIRED_COLUMNS:
                if not row.get(col) or row[col].strip() == '':
                    null_counts[col] += 1
            
            # Check data types
            try:
                quantity = int(row['quantity'])
                price = float(row['price'])
                order_date = datetime.strptime(row['order_date'], '%Y-%m-%d')
                
                # Business rules
                if quantity <= 0:
                    errors.append(f"Row {row_num}: quantity must be > 0 (got {quantity})")
                if price <= 0:
                    errors.append(f"Row {row_num}: price must be > 0 (got {price})")
                
            except (ValueError, TypeError) as e:
                errors.append(f"Row {row_num}: Invalid data type - {str(e)}")
        
        # Calculate statistics
        stats = {
            'total_rows': len(rows),
            'null_counts': null_counts,
            'errors': errors[:10],  # First 10 errors
            'error_count': len(errors)
        }
        
        # Determine if valid
        is_valid = len(errors) == 0 and all(count == 0 for count in null_counts.values())
        
        # Tag S3 object with validation result
        s3.put_object_tagging(
            Bucket=bucket,
            Key=key,
            Tagging={
                'TagSet': [
                    {'Key': 'validation_status', 'Value': 'valid' if is_valid else 'invalid'},
                    {'Key': 'row_count', 'Value': str(len(rows))},
                    {'Key': 'validated_at', 'Value': datetime.utcnow().isoformat()}
                ]
            }
        )
        
        print(f"Validation result: {'VALID' if is_valid else 'INVALID'}")
        print(f"Statistics: {json.dumps(stats, indent=2)}")
        
        if is_valid:
            # Trigger transformer Lambda (via S3 tag or SNS)
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'status': 'valid',
                    'stats': stats,
                    'next_step': 'transform'
                })
            }
        else:
            # Send alert for invalid data
            sns.publish(
                TopicArn=SNS_TOPIC_ARN,
                Subject='Data Validation Failed',
                Message=f"""
File: s3://{bucket}/{key}
Status: INVALID
Total rows: {stats['total_rows']}
Errors: {stats['error_count']}

First 10 errors:
{chr(10).join(stats['errors'])}
                """
            )
            
            raise ValueError(f"Validation failed with {stats['error_count']} errors")
    
    except Exception as e:
        print(f"❌ Validation error: {str(e)}")
        
        # Tag as invalid
        s3.put_object_tagging(
            Bucket=bucket,
            Key=key,
            Tagging={
                'TagSet': [
                    {'Key': 'validation_status', 'Value': 'error'},
                    {'Key': 'error_message', 'Value': str(e)[:256]}  # Max 256 chars
                ]
            }
        )
        
        raise
```

**Step 3: Lambda Function 2 - CSV to Parquet Transformer**

```python
# transformer.py
import json
import boto3
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
import io
from datetime import datetime

s3 = boto3.client('s3')

def lambda_handler(event, context):
    """
    Transform CSV to Parquet with Snappy compression
    """
    
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    print(f"Transforming: s3://{bucket}/{key}")
    
    # Check validation tag
    tags = s3.get_object_tagging(Bucket=bucket, Key=key)
    validation_status = next(
        (tag['Value'] for tag in tags['TagSet'] if tag['Key'] == 'validation_status'),
        None
    )
    
    if validation_status != 'valid':
        print(f"⚠️  Skipping transformation - validation_status={validation_status}")
        return {
            'statusCode': 400,
            'body': 'File not validated or validation failed'
        }
    
    try:
        # Read CSV from S3
        csv_obj = s3.get_object(Bucket=bucket, Key=key)
        df = pd.read_csv(io.BytesIO(csv_obj['Body'].read()))
        
        print(f"Loaded DataFrame: {len(df)} rows, {len(df.columns)} columns")
        
        # Data transformations
        # 1. Convert date column to datetime
        df['order_date'] = pd.to_datetime(df['order_date'])
        
        # 2. Add derived columns
        df['total_amount'] = df['quantity'] * df['price']
        df['processed_at'] = datetime.utcnow()
        
        # 3. Optimize data types (reduce storage)
        df['order_id'] = df['order_id'].astype('int64')
        df['customer_id'] = df['customer_id'].astype('int32')
        df['product_id'] = df['product_id'].astype('int32')
        df['quantity'] = df['quantity'].astype('int16')
        df['price'] = df['price'].astype('float32')
        df['total_amount'] = df['total_amount'].astype('float32')
        
        # 4. Add partition column (year-month for partitioning)
        df['year_month'] = df['order_date'].dt.strftime('%Y-%m')
        
        print(f"DataFrame after transformations:")
        print(df.dtypes)
        print(f"Memory usage: {df.memory_usage(deep=True).sum() / 1024**2:.2f} MB")
        
        # Convert to Parquet with Snappy compression
        table = pa.Table.from_pandas(df)
        
        # Write to buffer
        parquet_buffer = io.BytesIO()
        pq.write_table(
            table,
            parquet_buffer,
            compression='snappy',
            use_dictionary=True,
            version='2.6'
        )
        
        # Generate output key (preserve directory structure, change extension)
        output_key = key.replace('/raw/', '/processed/').replace('.csv', '.parquet')
        output_bucket = 'data-processed-12345'
        
        # Upload Parquet to S3
        parquet_buffer.seek(0)
        s3.put_object(
            Bucket=output_bucket,
            Key=output_key,
            Body=parquet_buffer.getvalue(),
            ContentType='application/octet-stream',
            Metadata={
                'source_file': f's3://{bucket}/{key}',
                'row_count': str(len(df)),
                'compression': 'snappy',
                'transformed_at': datetime.utcnow().isoformat()
            }
        )
        
        # Calculate compression ratio
        csv_size = csv_obj['ContentLength']
        parquet_size = len(parquet_buffer.getvalue())
        compression_ratio = csv_size / parquet_size
        
        print(f"✅ Transformation complete:")
        print(f"   Output: s3://{output_bucket}/{output_key}")
        print(f"   CSV size: {csv_size / 1024**2:.2f} MB")
        print(f"   Parquet size: {parquet_size / 1024**2:.2f} MB")
        print(f"   Compression ratio: {compression_ratio:.2f}x")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'output_bucket': output_bucket,
                'output_key': output_key,
                'row_count': len(df),
                'compression_ratio': compression_ratio,
                'parquet_size_mb': parquet_size / 1024**2
            })
        }
    
    except Exception as e:
        print(f"❌ Transformation error: {str(e)}")
        raise
```

**Step 4: Deploy Lambda Functions**

```bash
# Package dependencies (pandas, pyarrow are not in Lambda runtime)
# Create deployment package with layers

# Create Lambda Layer for pandas/pyarrow
mkdir -p /tmp/lambda-layer/python
pip install pandas pyarrow -t /tmp/lambda-layer/python/
cd /tmp/lambda-layer
zip -r /tmp/pandas-layer.zip python/

# Create layer
aws lambda publish-layer-version \
  --layer-name pandas-pyarrow \
  --zip-file fileb:///tmp/pandas-layer.zip \
  --compatible-runtimes python3.11

# Output: LayerVersionArn

# Deploy validator Lambda
zip /tmp/validator.zip validator.py

aws lambda create-function \
  --function-name data-validator \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/DataPipelineLambdaRole \
  --handler validator.lambda_handler \
  --zip-file fileb:///tmp/validator.zip \
  --timeout 300 \
  --memory-size 512

# Deploy transformer Lambda (with layer)
zip /tmp/transformer.zip transformer.py

aws lambda create-function \
  --function-name data-transformer \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/DataPipelineLambdaRole \
  --handler transformer.lambda_handler \
  --zip-file fileb:///tmp/transformer.zip \
  --timeout 900 \
  --memory-size 3008 \
  --layers arn:aws:lambda:us-east-1:123456789012:layer:pandas-pyarrow:1

# Configure S3 trigger for validator
aws lambda add-permission \
  --function-name data-validator \
  --statement-id s3-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::data-landing-12345

aws s3api put-bucket-notification-configuration \
  --bucket data-landing-12345 \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [{
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:data-validator",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {"Name": "prefix", "Value": "raw/"},
            {"Name": "suffix", "Value": ".csv"}
          ]
        }
      }
    }]
  }'

# Configure S3 trigger for transformer (on validated files)
# Use EventBridge rule to trigger on specific S3 tags
```

**Step 5: Test the Pipeline**

```bash
# Generate sample CSV data
cat > /tmp/sales_2024-06-22.csv <<EOF
order_id,customer_id,product_id,quantity,price,order_date
1001,5001,3001,2,29.99,2024-06-22
1002,5002,3002,1,149.99,2024-06-22
1003,5001,3003,5,9.99,2024-06-22
EOF

# Upload to S3 (triggers validator Lambda)
aws s3 cp /tmp/sales_2024-06-22.csv s3://data-landing-12345/raw/sales_2024-06-22.csv

# Monitor Lambda execution
aws logs tail /aws/lambda/data-validator --follow

# Check validation tags
aws s3api get-object-tagging --bucket data-landing-12345 --key raw/sales_2024-06-22.csv

# Expected output:
# {
#   "TagSet": [
#     {"Key": "validation_status", "Value": "valid"},
#     {"Key": "row_count", "Value": "3"}
#   ]
# }

# Manually trigger transformer (or configure EventBridge)
aws lambda invoke \
  --function-name data-transformer \
  --payload '{
    "Records": [{
      "s3": {
        "bucket": {"name": "data-landing-12345"},
        "object": {"key": "raw/sales_2024-06-22.csv"}
      }
    }]
  }' \
  /tmp/transformer-response.json

# Check output Parquet file
aws s3 ls s3://data-processed-12345/processed/ --recursive --human-readable

# Download and inspect Parquet
aws s3 cp s3://data-processed-12345/processed/sales_2024-06-22.parquet /tmp/

python3 -c "
import pandas as pd
df = pd.read_parquet('/tmp/sales_2024-06-22.parquet')
print(df)
print(f'\nShape: {df.shape}')
print(f'Columns: {list(df.columns)}')
"
```

**Results:**

| Metric | CSV (Input) | Parquet (Output) | Improvement |
|--------|-------------|------------------|-------------|
| **File Size** | 100 MB | 15 MB | **85% reduction** |
| **Processing Time** | N/A | 12 seconds (1M rows) | Lambda cold start: 3s |
| **Cost per 1M rows** | N/A | $0.0024 (Lambda) | Serverless, pay per use |
| **Compression** | None | Snappy | 6.7x compression ratio |

**Key Lessons:**

1. **Lambda Layers** enable reusable dependencies (pandas, pyarrow) across functions
2. **S3 Event Notifications** trigger Lambda automatically on file upload (no polling)
3. **Parquet compression** reduces storage by 85% vs CSV (faster Redshift queries)
4. **Tagging** enables conditional processing (only transform validated files)
5. **Memory tuning:** 3008 MB Lambda processes 100 MB CSV in 12 seconds (vs 60s at 512 MB)

---

## EXERCISE 5.2: BATCH PROCESSING WITH AWS BATCH

**Scenario:** Process 10,000 daily log files (each 10 MB) in parallel using AWS Batch with Spot Instances for 90% cost savings.

**Architecture:**

```
Amazon S3
└─ s3://logs-bucket/2024/06/22/
   ├─ app-log-0001.json.gz (10 MB)
   ├─ app-log-0002.json.gz (10 MB)
   ...
   └─ app-log-10000.json.gz (10 MB)
       │
       │ EventBridge Schedule (daily at 2 AM)
       ▼
AWS Batch Job Queue
├─ Job Definition: log-processor
├─ vCPUs: 1 per job
├─ Memory: 2 GB per job
└─ Array job: 10,000 tasks (parallel)
    │
    ├─ Compute Environment
    │  ├─ Instance types: c5.xlarge, c5.2xlarge (compute-optimized)
    │  ├─ Spot Instances: 90% cost savings
    │  ├─ Min vCPUs: 0
    │  ├─ Max vCPUs: 1,000 (250 c5.xlarge instances)
    │  └─ Auto-scaling: Scale up/down based on queue depth
    │
    └─ Docker Container (log-processor:latest)
       ├─ Read log file from S3
       ├─ Parse JSON, extract metrics
       ├─ Aggregate by hour
       └─ Write results to S3 (Parquet)
           │
           ▼
Amazon S3
└─ s3://processed-logs/2024/06/22/aggregated.parquet
    │
    └─→ AWS Glue Crawler → Athena (queryable)
```

**Implementation:**

**Step 1: Create Docker Container for Log Processing**

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Install dependencies
RUN pip install boto3 pandas pyarrow

# Copy processing script
COPY process_log.py /app/process_log.py

# Set working directory
WORKDIR /app

# Entry point
CMD ["python", "process_log.py"]
```

```python
# process_log.py
import os
import json
import gzip
import boto3
import pandas as pd
from datetime import datetime

s3 = boto3.client('s3')

def process_log_file(bucket, key):
    """
    Process a single log file:
    1. Download from S3
    2. Parse JSON logs
    3. Aggregate metrics by hour
    4. Return DataFrame
    """
    
    print(f"Processing: s3://{bucket}/{key}")
    
    # Download gzipped log file
    response = s3.get_object(Bucket=bucket, Key=key)
    
    # Decompress and parse JSON lines
    logs = []
    with gzip.open(response['Body'], 'rt') as f:
        for line in f:
            try:
                log_entry = json.loads(line)
                logs.append(log_entry)
            except json.JSONDecodeError:
                continue
    
    # Convert to DataFrame
    df = pd.DataFrame(logs)
    
    # Parse timestamp
    df['timestamp'] = pd.to_datetime(df['timestamp'])
    df['hour'] = df['timestamp'].dt.floor('H')
    
    # Aggregate metrics by hour
    hourly_stats = df.groupby('hour').agg({
        'request_id': 'count',  # Request count
        'response_time_ms': ['mean', 'p50', 'p95', 'p99', 'max'],
        'status_code': lambda x: (x == 200).sum(),  # Success count
        'user_id': 'nunique'  # Unique users
    }).reset_index()
    
    # Flatten column names
    hourly_stats.columns = [
        'hour', 'request_count',
        'avg_response_time', 'p50_response_time', 'p95_response_time', 'p99_response_time', 'max_response_time',
        'success_count', 'unique_users'
    ]
    
    # Add source file metadata
    hourly_stats['source_file'] = key
    hourly_stats['processed_at'] = datetime.utcnow()
    
    return hourly_stats

def main():
    """
    Main entry point for AWS Batch job
    Processes one log file per task
    """
    
    # AWS Batch provides array job index
    job_index = int(os.environ.get('AWS_BATCH_JOB_ARRAY_INDEX', '0'))
    
    # S3 bucket and key pattern
    bucket = os.environ['LOG_BUCKET']
    date = os.environ['LOG_DATE']  # Format: 2024-06-22
    
    # Construct key for this job index
    # app-log-0001.json.gz, app-log-0002.json.gz, etc.
    key = f"{date}/app-log-{job_index:04d}.json.gz"
    
    print(f"AWS Batch Job Array Index: {job_index}")
    print(f"Processing file: s3://{bucket}/{key}")
    
    try:
        # Process log file
        hourly_stats = process_log_file(bucket, key)
        
        print(f"Processed {len(hourly_stats)} hourly aggregations")
        print(hourly_stats.head())
        
        # Write results to S3 (Parquet)
        output_bucket = os.environ['OUTPUT_BUCKET']
        output_key = f"processed/{date}/hourly-stats-{job_index:04d}.parquet"
        
        # Convert to Parquet
        hourly_stats.to_parquet(
            f'/tmp/output.parquet',
            compression='snappy',
            index=False
        )
        
        # Upload to S3
        s3.upload_file(
            '/tmp/output.parquet',
            output_bucket,
            output_key
        )
        
        print(f"✅ Results written to: s3://{output_bucket}/{output_key}")
        
        return 0
    
    except Exception as e:
        print(f"❌ Error processing file: {str(e)}")
        return 1

if __name__ == '__main__':
    exit(main())
```

**Step 2: Build and Push Docker Image to ECR**

```bash
# Create ECR repository
aws ecr create-repository --repository-name log-processor --region us-east-1

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build Docker image
docker build -t log-processor:latest .

# Tag for ECR
docker tag log-processor:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-processor:latest

# Push to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-processor:latest
```

**Step 3: Create AWS Batch Compute Environment and Job Queue**

```bash
# Create IAM role for Batch service
cat > batch-service-role-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "batch.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name AWSBatchServiceRole \
  --assume-role-policy-document file://batch-service-role-trust.json

aws iam attach-role-policy \
  --role-name AWSBatchServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBatchServiceRole

# Create IAM role for ECS task execution
cat > ecs-task-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://ecs-task-trust.json

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Create IAM role for Batch job (S3 access)
cat > batch-job-role-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name BatchJobRole \
  --assume-role-policy-document file://batch-job-role-trust.json

aws iam attach-role-policy \
  --role-name BatchJobRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# Create Batch compute environment (Spot Instances)
aws batch create-compute-environment \
  --compute-environment-name log-processing-spot \
  --type MANAGED \
  --state ENABLED \
  --compute-resources '{
    "type": "SPOT",
    "allocationStrategy": "SPOT_CAPACITY_OPTIMIZED",
    "minvCpus": 0,
    "maxvCpus": 1000,
    "desiredvCpus": 0,
    "instanceTypes": ["c5.xlarge", "c5.2xlarge"],
    "subnets": ["subnet-0a1b2c3d", "subnet-1a2b3c4d"],
    "securityGroupIds": ["sg-0a1b2c3d"],
    "instanceRole": "arn:aws:iam::123456789012:instance-profile/ecsInstanceRole",
    "tags": {"Name": "BatchWorker"},
    "bidPercentage": 100,
    "spotIamFleetRole": "arn:aws:iam::123456789012:role/AmazonEC2SpotFleetRole"
  }' \
  --service-role arn:aws:iam::123456789012:role/AWSBatchServiceRole

# Create Batch job queue
aws batch create-job-queue \
  --job-queue-name log-processing-queue \
  --state ENABLED \
  --priority 100 \
  --compute-environment-order '[{
    "order": 1,
    "computeEnvironment": "log-processing-spot"
  }]'

# Create Batch job definition
aws batch register-job-definition \
  --job-definition-name log-processor \
  --type container \
  --container-properties '{
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/log-processor:latest",
    "vcpus": 1,
    "memory": 2048,
    "jobRoleArn": "arn:aws:iam::123456789012:role/BatchJobRole",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
    "environment": [
      {"name": "LOG_BUCKET", "value": "logs-bucket"},
      {"name": "OUTPUT_BUCKET", "value": "processed-logs"},
      {"name": "LOG_DATE", "value": "2024-06-22"}
    ]
  }'
```

**Step 4: Submit Array Job (10,000 Parallel Tasks)**

```bash
# Submit array job
aws batch submit-job \
  --job-name log-processing-2024-06-22 \
  --job-queue log-processing-queue \
  --job-definition log-processor \
  --array-properties size=10000

# Output:
{
  "jobName": "log-processing-2024-06-22",
  "jobId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}

# Monitor job progress
aws batch describe-jobs --jobs a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Check job queue
aws batch describe-job-queues --job-queues log-processing-queue | jq '.jobQueues[0]'

# List running jobs
aws batch list-jobs --job-queue log-processing-queue --job-status RUNNING | jq '.jobSummaryList | length'

# Monitor compute environment scaling
watch -n 10 'aws batch describe-compute-environments --compute-environments log-processing-spot | jq ".computeEnvironments[0].computeResources | {minvCpus, desiredvCpus, maxvCpus}"'
```

**Results:**

| Metric | On-Demand Instances | Spot Instances (AWS Batch) | Improvement |
|--------|---------------------|----------------------------|-------------|
| **Processing Time** | 2.5 hours (sequential) | 18 minutes (10,000 parallel) | **83% faster** |
| **Cost** | $125 (c5.xlarge on-demand) | $12.50 (Spot, 90% savings) | **90% cost reduction** |
| **Scaling** | Manual | Automatic (0 → 1,000 vCPUs) | **Fully automated** |
| **Fault Tolerance** | Manual retry | Automatic retry (3 attempts) | **Built-in resilience** |

**Cost Breakdown:**

```
On-Demand Instances (c5.xlarge):
├─ 10,000 jobs × 6 minutes each = 60,000 minutes = 1,000 hours
├─ Parallel: 1,000 vCPUs × 1 hour = 250 c5.xlarge instances (4 vCPUs each)
├─ Cost: 250 instances × $0.17/hour × 1 hour = $42.50/hour
└─ Total: $42.50 × 0.3 hours = $12.75

Spot Instances (90% discount):
├─ Same compute: 250 instances × 0.3 hours
├─ Cost: $42.50/hour × 90% discount = $4.25/hour
└─ Total: $4.25 × 0.3 hours = $1.28 ✅

Actual AWS Batch cost: $1.28 + $0.10 (Batch service overhead) = $1.38
```

---

(Continue with Exercise 5.3 and Q1-Q20...)## EXERCISE 5.3: BIG DATA PROCESSING WITH AMAZON EMR (SPARK)

**Scenario:** Process 100 TB of website clickstream data using Apache Spark on EMR to generate daily user behavior analytics.

**Architecture:**

```
Amazon S3 (Data Lake)
└─ s3://clickstream-data/raw/
   ├─ 2024/06/22/hour=00/*.parquet (4 TB/hour)
   ├─ 2024/06/22/hour=01/*.parquet
   ...
   └─ 2024/06/22/hour=23/*.parquet
       │
       │ EMR Step (PySpark job)
       ▼
Amazon EMR Cluster
├─ Release: emr-7.0.0 (Spark 3.5.0)
├─ Master: m5.xlarge (1 instance)
├─ Core: r5.4xlarge (10 instances, 160 cores, 1.28 TB RAM)
├─ Task: r5.4xlarge (20 instances, Spot 70% discount)
└─ Auto-scaling: 10-50 core instances based on YARN pending memory
    │
    ├─ Spark Job: Clickstream Analytics
    │  ├─ Read 100 TB Parquet from S3
    │  ├─ Filter: Remove bots, invalid sessions
    │  ├─ Aggregate: User sessions, page views, conversion funnel
    │  └─ Write: Partitioned Parquet to S3
    │
    └─ Output: s3://clickstream-analytics/aggregated/
        └─ date=2024-06-22/
            ├─ user_sessions.parquet (500 GB)
            ├─ page_views.parquet (300 GB)
            └─ conversion_funnel.parquet (100 GB)
                │
                └─→ AWS Glue Crawler → Athena/QuickSight
```

**Implementation:**

**Step 1: Create EMR Cluster with Auto-Scaling**

```bash
# Create EMR cluster
aws emr create-cluster \
  --name "Clickstream Analytics Cluster" \
  --release-label emr-7.0.0 \
  --applications Name=Spark Name=Hadoop Name=Hive \
  --ec2-attributes '{
    "KeyName": "emr-keypair",
    "InstanceProfile": "EMR_EC2_DefaultRole",
    "SubnetId": "subnet-0a1b2c3d",
    "EmrManagedMasterSecurityGroup": "sg-master",
    "EmrManagedSlaveSecurityGroup": "sg-slave"
  }' \
  --instance-groups '[
    {
      "Name": "Master",
      "InstanceGroupType": "MASTER",
      "InstanceType": "m5.xlarge",
      "InstanceCount": 1
    },
    {
      "Name": "Core",
      "InstanceGroupType": "CORE",
      "InstanceType": "r5.4xlarge",
      "InstanceCount": 10,
      "AutoScalingPolicy": {
        "Constraints": {
          "MinCapacity": 10,
          "MaxCapacity": 50
        },
        "Rules": [
          {
            "Name": "ScaleUpOnMemoryPressure",
            "Action": {
              "SimpleScalingPolicyConfiguration": {
                "AdjustmentType": "CHANGE_IN_CAPACITY",
                "ScalingAdjustment": 5,
                "CoolDown": 300
              }
            },
            "Trigger": {
              "CloudWatchAlarmDefinition": {
                "ComparisonOperator": "GREATER_THAN",
                "MetricName": "YARNMemoryAvailablePercentage",
                "Period": 300,
                "Threshold": 75,
                "Unit": "PERCENT"
              }
            }
          },
          {
            "Name": "ScaleDownOnIdleMemory",
            "Action": {
              "SimpleScalingPolicyConfiguration": {
                "AdjustmentType": "CHANGE_IN_CAPACITY",
                "ScalingAdjustment": -2,
                "CoolDown": 300
              }
            },
            "Trigger": {
              "CloudWatchAlarmDefinition": {
                "ComparisonOperator": "LESS_THAN",
                "MetricName": "YARNMemoryAvailablePercentage",
                "Period": 300,
                "Threshold": 25,
                "Unit": "PERCENT"
              }
            }
          }
        ]
      }
    },
    {
      "Name": "Task",
      "InstanceGroupType": "TASK",
      "InstanceType": "r5.4xlarge",
      "InstanceCount": 20,
      "BidPrice": "OnDemandPrice",
      "Market": "SPOT"
    }
  ]' \
  --configurations '[
    {
      "Classification": "spark-defaults",
      "Properties": {
        "spark.executor.memory": "24G",
        "spark.executor.cores": "4",
        "spark.dynamicAllocation.enabled": "true",
        "spark.shuffle.service.enabled": "true",
        "spark.sql.adaptive.enabled": "true",
        "spark.sql.adaptive.coalescePartitions.enabled": "true"
      }
    }
  ]' \
  --service-role EMR_DefaultRole \
  --log-uri s3://emr-logs-bucket/clickstream-analytics/ \
  --enable-debugging \
  --region us-east-1

# Output: ClusterId: j-XXXXXXXXXXXXX
```

**Step 2: PySpark Job for Clickstream Analytics**

```python
# clickstream_analytics.py
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.window import Window
from datetime import datetime

# Initialize Spark session
spark = SparkSession.builder \
    .appName("Clickstream Analytics") \
    .config("spark.sql.sources.partitionOverwriteMode", "dynamic") \
    .getOrCreate()

# S3 paths
INPUT_PATH = "s3://clickstream-data/raw/2024/06/22/"
OUTPUT_PATH = "s3://clickstream-analytics/aggregated/date=2024-06-22/"

print(f"Reading clickstream data from: {INPUT_PATH}")
start_time = datetime.now()

# Read raw clickstream data (100 TB Parquet)
df = spark.read.parquet(INPUT_PATH)

print(f"Loaded {df.count():,} raw events")
print(f"Schema: {df.schema}")

# Data quality: Filter out bots and invalid sessions
df_clean = df.filter(
    (F.col("is_bot") == False) &
    (F.col("session_id").isNotNull()) &
    (F.col("user_id").isNotNull()) &
    (F.col("timestamp").isNotNull())
)

print(f"After filtering: {df_clean.count():,} valid events")

# === Analytics 1: User Sessions ===
# Aggregate clickstream into user sessions
session_window = Window.partitionBy("session_id").orderBy("timestamp")

df_sessions = df_clean \
    .withColumn("page_sequence", F.row_number().over(session_window)) \
    .groupBy("session_id", "user_id", "date") \
    .agg(
        F.min("timestamp").alias("session_start"),
        F.max("timestamp").alias("session_end"),
        F.count("*").alias("page_views"),
        F.countDistinct("page_url").alias("unique_pages"),
        F.sum("time_on_page_sec").alias("total_time_sec"),
        F.max(F.when(F.col("event_type") == "purchase", 1).otherwise(0)).alias("converted"),
        F.sum(F.when(F.col("event_type") == "purchase", F.col("purchase_amount")).otherwise(0)).alias("revenue"),
        F.collect_list("page_url").alias("page_journey")
    )

# Calculate session duration
df_sessions = df_sessions.withColumn(
    "session_duration_sec",
    (F.col("session_end").cast("long") - F.col("session_start").cast("long"))
)

# Write user sessions (partitioned by date)
print("Writing user sessions...")
df_sessions.write \
    .mode("overwrite") \
    .partitionBy("date") \
    .parquet(f"{OUTPUT_PATH}/user_sessions/")

# === Analytics 2: Page Views Summary ===
df_page_views = df_clean \
    .groupBy("page_url", "date") \
    .agg(
        F.count("*").alias("total_views"),
        F.countDistinct("user_id").alias("unique_visitors"),
        F.avg("time_on_page_sec").alias("avg_time_on_page"),
        F.sum(F.when(F.col("event_type") == "purchase", 1).otherwise(0)).alias("conversions"),
        F.sum(F.when(F.col("event_type") == "purchase", F.col("purchase_amount")).otherwise(0)).alias("total_revenue")
    ) \
    .withColumn("conversion_rate", F.col("conversions") / F.col("total_views"))

print("Writing page views summary...")
df_page_views.write \
    .mode("overwrite") \
    .partitionBy("date") \
    .parquet(f"{OUTPUT_PATH}/page_views/")

# === Analytics 3: Conversion Funnel ===
# Define funnel steps
funnel_steps = [
    ("landing", "Landing Page"),
    ("product_view", "Product Page"),
    ("add_to_cart", "Add to Cart"),
    ("checkout", "Checkout"),
    ("purchase", "Purchase Complete")
]

# Calculate funnel metrics
funnel_data = []
for step_name, step_label in funnel_steps:
    count = df_clean.filter(F.col("event_type") == step_name).select("session_id").distinct().count()
    funnel_data.append((step_name, step_label, count))

df_funnel = spark.createDataFrame(funnel_data, ["step_name", "step_label", "session_count"])

# Calculate drop-off rates
df_funnel = df_funnel.withColumn("step_order", F.monotonically_increasing_id())
window_prev = Window.orderBy("step_order").rowsBetween(-1, -1)

df_funnel = df_funnel \
    .withColumn("prev_count", F.lag("session_count").over(window_prev)) \
    .withColumn(
        "conversion_rate",
        F.when(F.col("prev_count").isNotNull(), F.col("session_count") / F.col("prev_count")).otherwise(1.0)
    ) \
    .withColumn(
        "drop_off_rate",
        1 - F.col("conversion_rate")
    )

print("Writing conversion funnel...")
df_funnel.write \
    .mode("overwrite") \
    .parquet(f"{OUTPUT_PATH}/conversion_funnel/")

# Show funnel metrics
print("\nConversion Funnel:")
df_funnel.show(truncate=False)

# Performance metrics
end_time = datetime.now()
duration = (end_time - start_time).total_seconds()

print(f"\n{'='*60}")
print(f"Job completed successfully!")
print(f"Duration: {duration/60:.2f} minutes")
print(f"Input data: 100 TB")
print(f"Output data: ~900 GB (compressed)")
print(f"{'='*60}")

spark.stop()
```

**Step 3: Submit Spark Job to EMR**

```bash
# Upload PySpark script to S3
aws s3 cp clickstream_analytics.py s3://emr-scripts/clickstream_analytics.py

# Submit Spark step to EMR cluster
aws emr add-steps \
  --cluster-id j-XXXXXXXXXXXXX \
  --steps '[
    {
      "Name": "Clickstream Analytics",
      "ActionOnFailure": "CONTINUE",
      "HadoopJarStep": {
        "Jar": "command-runner.jar",
        "Args": [
          "spark-submit",
          "--deploy-mode", "cluster",
          "--master", "yarn",
          "--conf", "spark.executor.instances=40",
          "--conf", "spark.executor.memory=24G",
          "--conf", "spark.executor.cores=4",
          "--conf", "spark.driver.memory=8G",
          "s3://emr-scripts/clickstream_analytics.py"
        ]
      }
    }
  ]'

# Monitor step progress
aws emr describe-step \
  --cluster-id j-XXXXXXXXXXXXX \
  --step-id s-XXXXXXXXXXXXX

# Watch logs
aws emr ssh --cluster-id j-XXXXXXXXXXXXX --key-pair-file emr-keypair.pem
# On master node:
yarn logs -applicationId application_1234567890123_0001
```

**Results:**

| Metric | Traditional Hadoop | EMR with Spark | Improvement |
|--------|-------------------|----------------|-------------|
| **Processing Time** | 12 hours | 45 minutes | **93% faster** |
| **Cost** | $450 (on-demand) | $135 (Spot instances) | **70% cheaper** |
| **Data Processed** | 100 TB | 100 TB | Same |
| **Output Size** | 100 TB (uncompressed) | 900 GB (Parquet) | **99% reduction** |
| **Cluster Size** | Fixed (50 nodes) | Auto-scaling (10-50 nodes) | **Dynamic scaling** |

---

## BEGINNER QUESTIONS (Q1-Q5)

**Q1: What is AWS Lambda and when should you use it for data processing?**

**Answer:**

AWS Lambda is a serverless compute service that runs code in response to events without provisioning or managing servers.

**When to use Lambda for data engineering:**

1. **Event-driven ETL pipelines**
   - S3 file upload → Lambda → transform → S3
   - DynamoDB Stream → Lambda → aggregate → Redshift
   - Kinesis Data Stream → Lambda → process → S3

2. **Lightweight data transformations**
   - File format conversions (CSV → Parquet)
   - Data validation and quality checks
   - Simple aggregations and filtering

3. **Glue job orchestration**
   - Trigger Glue jobs on schedule or events
   - Monitor job status and send alerts
   - Manage dependencies between jobs

4. **Real-time data processing**
   - Process streaming data from Kinesis
   - Enrich data with external APIs
   - Write to databases or data warehouses

**Lambda Limitations for Data Engineering:**

| Limit | Value | Impact |
|-------|-------|--------|
| **Max execution time** | 15 minutes | Cannot process long-running batch jobs |
| **Max memory** | 10 GB | Limited for large datasets |
| **Ephemeral storage** | 10 GB (/tmp) | Cannot process files > 10 GB |
| **Concurrency** | 1,000 (soft limit) | Can overwhelm downstream systems |

**When NOT to use Lambda:**
- Batch jobs > 15 minutes (use AWS Batch or EMR)
- Processing files > 10 GB (use EMR or Glue)
- Complex Spark/Hadoop workloads (use EMR)
- Predictable, continuous workloads (EC2 is cheaper)

**Example Use Case:**

```python
# Lambda for CSV to Parquet conversion
import boto3
import pandas as pd

s3 = boto3.client('s3')

def lambda_handler(event, context):
    bucket = event['Records'][0]['s3']['bucket']['name']
    csv_key = event['Records'][0]['s3']['object']['key']
    
    # Read CSV (works for files < 10 GB)
    csv_obj = s3.get_object(Bucket=bucket, Key=csv_key)
    df = pd.read_csv(csv_obj['Body'])
    
    # Convert to Parquet
    parquet_key = csv_key.replace('.csv', '.parquet')
    df.to_parquet(f'/tmp/output.parquet', compression='snappy')
    
    # Upload to S3
    s3.upload_file('/tmp/output.parquet', bucket, parquet_key)
    
    return {'statusCode': 200, 'converted': parquet_key}
```

---

**Q2: What is AWS Batch and how does it differ from Lambda?**

**Answer:**

AWS Batch is a fully managed service for running batch computing workloads at any scale.

**Key Differences:**

| Feature | AWS Lambda | AWS Batch |
|---------|-----------|-----------|
| **Execution Time** | Max 15 minutes | Unlimited (days) |
| **Compute** | Serverless (no instance management) | EC2 instances (managed) |
| **Scaling** | Automatic (1,000 concurrent) | Auto-scaling (0 → 10,000+ vCPUs) |
| **Cost Model** | Pay per request + duration | Pay for EC2 instances used |
| **Use Case** | Event-driven, short tasks | Long-running batch jobs |
| **Memory** | Max 10 GB | Up to 256 GB (depends on instance) |
| **GPU Support** | No | Yes (p3, g4dn instances) |
| **Spot Instances** | No | Yes (90% cost savings) |

**When to use AWS Batch:**

1. **Long-running batch processing**
   - Video transcoding (2-4 hours per file)
   - Scientific simulations (days)
   - Data migration jobs (hours)

2. **Parallel processing at scale**
   - Process 10,000 log files in parallel
   - Run 1,000 ML model training jobs
   - Batch image processing (resize, watermark)

3. **Resource-intensive workloads**
   - Requires > 10 GB memory
   - Needs GPU for ML inference
   - Complex multi-step processing

4. **Cost optimization with Spot Instances**
   - Fault-tolerant workloads
   - Can handle interruptions (automatic retry)
   - 70-90% cost savings vs on-demand

**Example: AWS Batch for Log Processing**

```json
// Batch job definition
{
  "jobDefinitionName": "log-processor",
  "type": "container",
  "containerProperties": {
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/log-processor:latest",
    "vcpus": 2,
    "memory": 4096,
    "environment": [
      {"name": "LOG_BUCKET", "value": "logs-bucket"},
      {"name": "OUTPUT_BUCKET", "value": "processed-logs"}
    ]
  },
  "retryStrategy": {
    "attempts": 3
  }
}

// Submit array job (10,000 tasks)
{
  "jobName": "process-logs-2024-06-22",
  "jobQueue": "log-processing-queue",
  "jobDefinition": "log-processor",
  "arrayProperties": {
    "size": 10000  // 10,000 parallel tasks
  }
}
```

**Cost Comparison (processing 10,000 files, 6 minutes each):**

```
Lambda:
├─ 10,000 invocations × 6 minutes × 1 GB memory
├─ Duration: 60,000 minutes = $0.0000166667/GB-second
├─ Cost: 60,000 × 60 × 1 × $0.0000166667 = $60
└─ Note: Sequential (10,000 × 6 min = 1,000 hours) ❌

AWS Batch (Spot Instances):
├─ 10,000 jobs × 6 minutes = 60,000 minutes
├─ Parallel: 1,000 vCPUs × 1 hour (250 c5.xlarge instances)
├─ Cost: 250 × $0.017/hour (Spot) × 1 hour = $4.25
└─ Duration: 1 hour (parallel) ✅
```

**Winner:** AWS Batch is 93% cheaper and 1,000x faster for parallel batch workloads.

---

**Q3: What is Amazon EMR and when should you use it?**

**Answer:**

Amazon EMR (Elastic MapReduce) is a managed big data platform for running Apache Spark, Hadoop, Presto, and other frameworks at petabyte scale.

**Core Components:**

```
EMR Cluster Architecture
├─ Master Node (1)
│  └─ Manages cluster, coordinates jobs (YARN ResourceManager)
│
├─ Core Nodes (10-100+)
│  ├─ Run tasks (YARN NodeManager)
│  ├─ Store data on HDFS (when not using S3)
│  └─ Cannot be terminated without data loss (if using HDFS)
│
└─ Task Nodes (0-1000+, optional)
   ├─ Run tasks only (no data storage)
   ├─ Can use Spot Instances (70-90% savings)
   └─ Can be added/removed dynamically
```

**When to use EMR:**

1. **Big data processing (> 1 TB)**
   - Spark jobs for 100 TB clickstream data
   - Hadoop MapReduce for log aggregation
   - Presto for interactive SQL on S3 data lake

2. **Machine learning at scale**
   - Train ML models on large datasets
   - Feature engineering with Spark MLlib
   - Distributed hyperparameter tuning

3. **Complex transformations**
   - Multi-step ETL pipelines
   - Graph analytics (Spark GraphX)
   - Time-series analysis

4. **Cost-effective for large workloads**
   - S3 as data lake (separate compute/storage)
   - Transient clusters (launch → process → terminate)
   - Spot Instances for task nodes (70% savings)

**EMR vs Other Compute Services:**

| Service | Best For | Cost (100 TB processing) |
|---------|----------|--------------------------|
| **Lambda** | < 1 GB files, < 15 min | $20,000+ (not practical) ❌ |
| **Glue** | 1-100 GB files, serverless ETL | $5,000 (10 DPUs × 5 hours) |
| **AWS Batch** | Batch jobs, no big data frameworks | $500 (custom containers) |
| **EMR** | > 1 TB, Spark/Hadoop needed | $150 (Spot instances) ✅ |
| **Athena** | Ad-hoc queries, no transformation | $500 (query cost) |

**Example: EMR Cluster Configuration**

```bash
# Create transient EMR cluster for daily ETL
aws emr create-cluster \
  --name "Daily ETL - 2024-06-22" \
  --release-label emr-7.0.0 \
  --applications Name=Spark \
  --instance-groups '[
    {"InstanceType": "m5.xlarge", "InstanceGroupType": "MASTER", "InstanceCount": 1},
    {"InstanceType": "r5.4xlarge", "InstanceGroupType": "CORE", "InstanceCount": 10},
    {"InstanceType": "r5.4xlarge", "InstanceGroupType": "TASK", "InstanceCount": 20, "Market": "SPOT"}
  ]' \
  --auto-terminate \
  --service-role EMR_DefaultRole \
  --ec2-attributes InstanceProfile=EMR_EC2_DefaultRole

# Cost calculation:
# Master: 1 × m5.xlarge × $0.192/hour × 1 hour = $0.19
# Core: 10 × r5.4xlarge × $1.008/hour × 1 hour = $10.08
# Task (Spot): 20 × r5.4xlarge × $0.30/hour × 1 hour = $6.00 (70% discount)
# Total: $16.27 for processing 100 TB in 1 hour ✅
```

---

**Q4: What is AWS Glue and how does it simplify ETL?**

**Answer:**

AWS Glue is a fully managed serverless ETL service that makes it easy to prepare and transform data for analytics.

**Key Components:**

```
AWS Glue Architecture
├─ Glue Data Catalog
│  ├─ Central metadata repository
│  ├─ Stores table schemas, partition info
│  └─ Integrated with Athena, EMR, Redshift Spectrum
│
├─ Glue Crawlers
│  ├─ Automatically discover schemas in S3, RDS, Redshift
│  ├─ Infer data types and partitions
│  └─ Update Data Catalog tables
│
├─ Glue ETL Jobs
│  ├─ Serverless Spark jobs (no cluster management)
│  ├─ Python or Scala code
│  ├─ Built-in transformations (join, filter, map)
│  └─ Auto-scaling (1-100 DPUs)
│
└─ Glue DataBrew
   ├─ Visual data preparation tool (no code)
   ├─ 250+ pre-built transformations
   └─ Profile data quality
```

**When to use Glue:**

1. **Serverless ETL (don't want to manage clusters)**
   - Transform CSV/JSON → Parquet
   - Join data from multiple sources
   - Deduplicate and clean data

2. **Schema discovery**
   - Crawl S3 buckets to create tables
   - Automatically detect partitions
   - Keep catalog updated

3. **Cost-effective for medium workloads (1-100 GB)**
   - Pay per DPU-hour (Data Processing Unit)
   - No cluster overhead
   - Auto-scaling

**Glue vs EMR:**

| Feature | AWS Glue | Amazon EMR |
|---------|----------|------------|
| **Management** | Fully managed (serverless) | Semi-managed (you config cluster) |
| **Scaling** | Automatic (1-100 DPUs) | Manual or auto-scaling |
| **Frameworks** | Spark only (PySpark/Scala) | Spark, Hadoop, Presto, Hive, Flink |
| **Cost Model** | $0.44/DPU-hour | EC2 instance pricing |
| **Best For** | Simple ETL, < 100 GB | Complex workloads, > 1 TB |
| **Startup Time** | 2-3 minutes | 5-10 minutes (cluster launch) |

**Example: Glue Crawler + ETL Job**

```python
# Create Glue crawler
import boto3

glue = boto3.client('glue')

glue.create_crawler(
    Name='s3-sales-data-crawler',
    Role='AWSGlueServiceRole',
    DatabaseName='sales_db',
    Targets={
        'S3Targets': [{
            'Path': 's3://sales-data/parquet/'
        }]
    },
    Schedule='cron(0 2 * * ? *)'  # Daily at 2 AM
)

# Glue ETL job (PySpark)
from awsglue.context import GlueContext
from pyspark.context import SparkContext

sc = SparkContext()
glueContext = GlueContext(sc)

# Read from Data Catalog
datasource = glueContext.create_dynamic_frame.from_catalog(
    database="sales_db",
    table_name="raw_sales"
)

# Transform: Remove duplicates
deduplicated = datasource.drop_duplicates()

# Transform: Convert data types
transformed = deduplicated.apply_mapping([
    ("order_id", "string", "order_id", "long"),
    ("amount", "string", "amount", "decimal(10,2)"),
    ("order_date", "string", "order_date", "date")
])

# Write to S3 as Parquet
glueContext.write_dynamic_frame.from_options(
    frame=transformed,
    connection_type="s3",
    connection_options={"path": "s3://processed-sales/parquet/"},
    format="parquet"
)
```

**Cost Comparison (Transform 10 GB CSV → Parquet):**

```
AWS Glue:
├─ 2 DPUs × 0.5 hours × $0.44/DPU-hour = $0.44
└─ Total: $0.44 ✅

EMR (minimum cluster):
├─ 1 m5.xlarge master × 1 hour × $0.192 = $0.19
├─ 2 r5.xlarge core × 1 hour × $0.252 = $0.50
└─ Total: $0.69 (more expensive for small jobs)

Lambda (if file < 10 GB):
├─ 1 invocation × 300 sec × 3 GB memory = $0.025
└─ Total: $0.025 (cheapest for small files) ✅
```

**Winner by data size:**
- < 1 GB: Lambda ($0.025)
- 1-100 GB: Glue ($0.44)
- > 100 GB: EMR ($16 for 100 TB)

---

**Q5: What is AWS Step Functions and how does it orchestrate data workflows?**

**Answer:**

AWS Step Functions is a serverless orchestration service that coordinates multiple AWS services into serverless workflows.

**Key Features:**

```
Step Functions Workflow
├─ States
│  ├─ Task: Execute Lambda, Batch, Glue, EMR
│  ├─ Choice: Conditional branching
│  ├─ Parallel: Execute multiple branches simultaneously
│  ├─ Wait: Pause for specified time
│  └─ Map: Iterate over array of items
│
├─ Error Handling
│  ├─ Retry: Automatic retry with exponential backoff
│  ├─ Catch: Handle errors and route to fallback
│  └─ Timeout: Fail if task exceeds duration
│
└─ Integration
   ├─ AWS Lambda (invoke functions)
   ├─ AWS Batch (submit jobs)
   ├─ AWS Glue (start ETL jobs)
   ├─ Amazon EMR (create clusters, submit steps)
   ├─ Amazon SQS (send messages)
   └─ Amazon SNS (send notifications)
```

**When to use Step Functions:**

1. **Multi-step ETL pipelines**
   - Extract → Transform → Load → Validate → Notify
   - Coordinate Lambda, Glue, and Redshift

2. **Error handling and retries**
   - Automatic retry on transient failures
   - Fallback to alternative processing path
   - Dead letter queue for failed items

3. **Parallel processing**
   - Process multiple files simultaneously
   - Run independent transformations in parallel
   - Wait for all to complete before next step

4. **Long-running workflows (up to 1 year)**
   - Wait for external approval
   - Pause until data arrives
   - Scheduled batch processing

**Example: Data Pipeline Workflow**

```json
{
  "Comment": "Daily ETL Pipeline",
  "StartAt": "ValidateInput",
  "States": {
    "ValidateInput": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:validate-data",
      "Next": "CheckValidation",
      "Retry": [{
        "ErrorEquals": ["States.TaskFailed"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2.0
      }],
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "Next": "SendFailureNotification"
      }]
    },
    "CheckValidation": {
      "Type": "Choice",
      "Choices": [{
        "Variable": "$.validation.status",
        "StringEquals": "valid",
        "Next": "ParallelProcessing"
      }],
      "Default": "SendFailureNotification"
    },
    "ParallelProcessing": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "TransformToParquet",
          "States": {
            "TransformToParquet": {
              "Type": "Task",
              "Resource": "arn:aws:states:::glue:startJobRun.sync",
              "Parameters": {
                "JobName": "csv-to-parquet"
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "GenerateStatistics",
          "States": {
            "GenerateStatistics": {
              "Type": "Task",
              "Resource": "arn:aws:lambda:us-east-1:123456789012:function:generate-stats",
              "End": true
            }
          }
        }
      ],
      "Next": "LoadToRedshift"
    },
    "LoadToRedshift": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:load-redshift",
      "Next": "SendSuccessNotification"
    },
    "SendSuccessNotification": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:ETLNotifications",
        "Subject": "ETL Pipeline Success",
        "Message.$": "$.result"
      },
      "End": true
    },
    "SendFailureNotification": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:123456789012:ETLNotifications",
        "Subject": "ETL Pipeline Failed",
        "Message.$": "$.error"
      },
      "End": true
    }
  }
}
```

**Visual Workflow:**

```
Start
  │
  ▼
ValidateInput (Lambda)
  │
  ├─→ [Success] → Parallel:
  │                ├─ Transform to Parquet (Glue)
  │                └─ Generate Statistics (Lambda)
  │                     │
  │                     └─→ LoadToRedshift (Lambda)
  │                          │
  │                          └─→ SendSuccessNotification (SNS)
  │
  └─→ [Failure] → SendFailureNotification (SNS)
```

**Cost:**

```
Step Functions Pricing:
├─ State transitions: $0.025 per 1,000 transitions
├─ Example workflow (8 states): $0.0002 per execution
├─ 1 million executions/month: $200
└─ Note: Lambda/Glue/Batch costs are separate
```

---

(Continue with Q6-Q20...)
## INTERMEDIATE QUESTIONS (Q6-Q10)

**Q6: How do you optimize Lambda performance for data processing workloads?**

**Answer:**

**Lambda Performance Optimization Strategies:**

**1. Memory Allocation (directly affects CPU)**

```python
# Memory impacts CPU allocation and network bandwidth
# Memory:  128 MB → 0.08 vCPU
# Memory:  1,024 MB → 0.6 vCPU
# Memory:  1,792 MB → 1.0 vCPU (full vCPU)
# Memory:  10,240 MB → 6.0 vCPUs

# Test: Process 100 MB CSV file at different memory levels
# 512 MB: 60 seconds, $0.0010
# 1,792 MB: 15 seconds, $0.0008 (faster AND cheaper due to shorter duration)
# 3,008 MB: 10 seconds, $0.0009
# Optimal: 1,792 MB (best price/performance)
```

**2. Cold Start Optimization**

```python
# Bad: Import inside function
def lambda_handler(event, context):
    import pandas as pd  # Cold start: +800ms
    import boto3         # Cold start: +200ms
    # Process data...

# Good: Import at module level (outside handler)
import pandas as pd  # Loaded once, reused across invocations
import boto3

# Initialize clients globally (connection pooling)
s3_client = boto3.client('s3')

def lambda_handler(event, context):
    # Handler executes faster (warm start)
    data = s3_client.get_object(Bucket=bucket, Key=key)
```

**3. Use Lambda Layers for Large Dependencies**

```bash
# Create layer for pandas/numpy (reduces deployment package size)
mkdir -p lambda-layer/python
pip install pandas numpy pyarrow -t lambda-layer/python/
cd lambda-layer && zip -r ../layer.zip python/

aws lambda publish-layer-version \
  --layer-name data-processing-deps \
  --zip-file fileb://layer.zip \
  --compatible-runtimes python3.11

# Lambda function uses layer (deployment package < 50 MB)
aws lambda update-function-configuration \
  --function-name data-processor \
  --layers arn:aws:lambda:us-east-1:123456789012:layer:data-processing-deps:1
```

**4. Parallel Processing with Lambda + S3 Batch Operations**

```python
# Instead of: Sequential Lambda invocations (slow)
for file in files:
    lambda_client.invoke(FunctionName='process-file', Payload=json.dumps({'file': file}))

# Use: S3 Batch Operations (parallel, up to 1,000 concurrent Lambdas)
s3control.create_job(
    Operation={
        'LambdaInvoke': {
            'FunctionArn': 'arn:aws:lambda:us-east-1:123456789012:function:process-file'
        }
    },
    Manifest={
        'Location': {
            'ObjectArn': 'arn:aws:s3:::my-bucket/manifest.csv'
        }
    }
)

# Result: 10,000 files processed in 5 minutes (vs 8 hours sequential)
```

**5. Use /tmp Efficiently (10 GB ephemeral storage)**

```python
# Bad: Load entire file into memory (OOM for large files)
s3_obj = s3.get_object(Bucket=bucket, Key=key)
data = s3_obj['Body'].read()  # OutOfMemoryError if file > Lambda memory

# Good: Stream to /tmp, process in chunks
import shutil

s3.download_file(bucket, key, '/tmp/input.csv')

# Process in chunks (memory-efficient)
for chunk in pd.read_csv('/tmp/input.csv', chunksize=100000):
    process_chunk(chunk)

# Clean up /tmp to reuse across invocations
os.remove('/tmp/input.csv')
```

**6. Enable X-Ray Tracing for Performance Analysis**

```bash
# Enable X-Ray tracing
aws lambda update-function-configuration \
  --function-name data-processor \
  --tracing-config Mode=Active

# X-Ray shows breakdown:
# - S3 GetObject: 200ms
# - Data processing: 5,000ms
# - S3 PutObject: 150ms
# → Optimize processing logic (80% of time)
```

**Performance Comparison:**

| Optimization | Before | After | Improvement |
|--------------|--------|-------|-------------|
| **Memory** | 512 MB (60s) | 1,792 MB (15s) | **75% faster, 20% cheaper** |
| **Cold Start** | 2,500ms | 500ms | **80% reduction** |
| **Deployment Size** | 250 MB | 10 MB (with layers) | **96% smaller** |
| **Parallel Processing** | 8 hours (sequential) | 5 minutes (S3 Batch) | **99% faster** |

---

**Q7: How do you configure auto-scaling for AWS Batch compute environments?**

**Answer:**

**AWS Batch Auto-Scaling Strategies:**

**1. Managed Scaling (Default)**

```bash
# AWS Batch automatically scales based on job queue depth
aws batch create-compute-environment \
  --compute-environment-name auto-scaling-batch \
  --type MANAGED \
  --state ENABLED \
  --compute-resources '{
    "type": "EC2",
    "allocationStrategy": "BEST_FIT_PROGRESSIVE",
    "minvCpus": 0,      # Scale to zero when no jobs
    "maxvCpus": 1000,   # Max capacity
    "desiredvCpus": 0,  # Let Batch manage this
    "instanceTypes": ["optimal"],  # Batch chooses best instances
    "subnets": ["subnet-0a1b2c3d"],
    "securityGroupIds": ["sg-0a1b2c3d"],
    "instanceRole": "arn:aws:iam::123456789012:instance-profile/ecsInstanceRole"
  }'

# Scaling behavior:
# - Jobs in queue: 100 (requiring 400 vCPUs)
# - Current capacity: 0 vCPUs
# - Batch scales up: 0 → 400 vCPUs (launches instances)
# - Jobs complete: 100 → 0
# - Batch scales down: 400 → 0 vCPUs (terminates instances)
```

**2. Spot Instance Allocation Strategy**

```bash
# SPOT_CAPACITY_OPTIMIZED: Choose instance types with least interruption risk
aws batch create-compute-environment \
  --compute-environment-name spot-optimized \
  --compute-resources '{
    "type": "SPOT",
    "allocationStrategy": "SPOT_CAPACITY_OPTIMIZED",
    "bidPercentage": 100,  # Bid up to 100% of On-Demand price
    "instanceTypes": ["c5.large", "c5.xlarge", "c5.2xlarge", "c5.4xlarge"],
    "minvCpus": 0,
    "maxvCpus": 2000,
    "spotIamFleetRole": "arn:aws:iam::123456789012:role/AmazonEC2SpotFleetRole"
  }'

# Result: 70-90% cost savings with minimal interruptions
# Batch automatically replaces interrupted Spot instances
```

**3. Mixed Instance Types for Flexibility**

```json
{
  "instanceTypes": ["optimal"],  // Batch chooses from all instance families
  "ec2Configuration": [{
    "imageType": "ECS_AL2",
    "imageIdOverride": "ami-0c55b159cbfafe1f0"
  }]
}

// Batch may choose:
// - c5.xlarge (compute-optimized) for CPU-intensive jobs
// - r5.xlarge (memory-optimized) for data processing
// - m5.xlarge (general purpose) for balanced workloads
```

**4. Multi-Queue Priority Scaling**

```bash
# High-priority queue (processes first)
aws batch create-job-queue \
  --job-queue-name high-priority \
  --priority 100 \
  --compute-environment-order '[{
    "order": 1,
    "computeEnvironment": "auto-scaling-batch"
  }]'

# Low-priority queue (processes when high-priority is empty)
aws batch create-job-queue \
  --job-queue-name low-priority \
  --priority 10 \
  --compute-environment-order '[{
    "order": 1,
    "computeEnvironment": "auto-scaling-batch"
  }]'

# Scaling priority:
# 1. High-priority jobs trigger immediate scaling
# 2. Low-priority jobs use spare capacity
# 3. Shared compute environment scales to meet total demand
```

**5. Custom Scaling with CloudWatch Alarms (Advanced)**

```python
# Monitor job queue depth and scale proactively
import boto3

cloudwatch = boto3.client('cloudwatch')
batch = boto3.client('batch')

# Create alarm for high queue depth
cloudwatch.put_metric_alarm(
    AlarmName='BatchQueueDepthHigh',
    MetricName='ApproximateNumberOfMessagesVisible',
    Namespace='AWS/Batch',
    Statistic='Average',
    Period=300,  # 5 minutes
    EvaluationPeriods=1,
    Threshold=100,  # > 100 jobs in queue
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=[
        'arn:aws:sns:us-east-1:123456789012:BatchScaling'
    ]
)

# Lambda triggered by SNS can update compute environment:
def lambda_handler(event, context):
    batch.update_compute_environment(
        computeEnvironment='auto-scaling-batch',
        computeResources={'desiredvCpus': 500}  # Scale up
    )
```

**Scaling Performance:**

| Metric | Fixed Capacity | Auto-Scaling | Improvement |
|--------|----------------|--------------|-------------|
| **Min Cost** | $100/day (idle instances) | $0/day (scale to zero) | **100% savings when idle** |
| **Scale-Up Time** | N/A (always ready) | 3-5 minutes (EC2 launch) | Slight delay |
| **Max Capacity** | Fixed (50 instances) | Dynamic (0-1,000 instances) | **20x more capacity** |
| **Spot Savings** | N/A | 70-90% vs On-Demand | **Huge cost reduction** |

---

**Q8: How do you optimize EMR costs with Spot Instances and instance fleets?**

**Answer:**

**EMR Cost Optimization Strategies:**

**1. Use Task Nodes with Spot Instances**

```bash
# Architecture:
# - Master + Core: On-Demand (stable, stores HDFS data)
# - Task: Spot Instances (90% savings, can be interrupted)

aws emr create-cluster \
  --name "Cost-Optimized EMR" \
  --instance-groups '[
    {
      "Name": "Master",
      "InstanceGroupType": "MASTER",
      "InstanceType": "m5.xlarge",
      "InstanceCount": 1,
      "Market": "ON_DEMAND"
    },
    {
      "Name": "Core",
      "InstanceGroupType": "CORE",
      "InstanceType": "r5.2xlarge",
      "InstanceCount": 2,
      "Market": "ON_DEMAND"
    },
    {
      "Name": "Task",
      "InstanceGroupType": "TASK",
      "InstanceType": "r5.2xlarge",
      "InstanceCount": 10,
      "Market": "SPOT",
      "BidPrice": "OnDemandPrice"
    }
  ]'

# Cost:
# - On-Demand (Master + 2 Core): 3 × $0.504/hour = $1.51/hour
# - Spot (10 Task): 10 × $0.05/hour (90% off) = $0.50/hour
# - Total: $2.01/hour (vs $6.55 on-demand for all 13 instances)
# - Savings: 69% ✅
```

**2. Instance Fleets (Mix Multiple Instance Types)**

```json
{
  "Name": "Cost-Optimized Fleet",
  "InstanceFleetType": "TASK",
  "TargetSpotCapacity": 100,  // Target 100 vCPUs
  "LaunchSpecifications": {
    "SpotSpecification": {
      "AllocationStrategy": "capacity-optimized",  // Minimize interruptions
      "TimeoutDurationMinutes": 10,
      "TimeoutAction": "SWITCH_TO_ON_DEMAND"
    }
  },
  "InstanceTypeConfigs": [
    {
      "InstanceType": "r5.xlarge",
      "WeightedCapacity": 4,
      "BidPrice": "OnDemandPrice"
    },
    {
      "InstanceType": "r5.2xlarge",
      "WeightedCapacity": 8,
      "BidPrice": "OnDemandPrice"
    },
    {
      "InstanceType": "r5.4xlarge",
      "WeightedCapacity": 16,
      "BidPrice": "OnDemandPrice"
    },
    {
      "InstanceType": "r5a.xlarge",  // AMD variant (often cheaper)
      "WeightedCapacity": 4,
      "BidPrice": "OnDemandPrice"
    }
  ]
}

// EMR chooses optimal mix:
// - Capacity needed: 100 vCPUs
// - EMR selects: 6 × r5.4xlarge (96 vCPUs) + 1 × r5.xlarge (4 vCPUs)
// - Based on: Lowest Spot price + lowest interruption risk
```

**3. Use S3 as Data Lake (Separate Compute/Storage)**

```python
# Bad: Store data on HDFS (Core nodes can't be Spot)
df = spark.read.parquet("hdfs:///data/clickstream/")
# Core nodes must be On-Demand (expensive)

# Good: Store data on S3 (Core nodes can be minimal)
df = spark.read.parquet("s3://data-lake/clickstream/")
# Core nodes: 2 × m5.xlarge (minimal, just for YARN/Spark coordination)
# Task nodes: 50 × r5.4xlarge Spot (90% savings)

# Cost comparison:
# HDFS: 50 × r5.4xlarge On-Demand = $50.40/hour
# S3: 2 × m5.xlarge On-Demand + 50 × r5.4xlarge Spot = $5.40/hour
# Savings: 89% ✅
```

**4. Transient Clusters (Terminate After Job Completes)**

```bash
# Create cluster with --auto-terminate flag
aws emr create-cluster \
  --auto-terminate \
  --steps '[{
    "Name": "Daily ETL",
    "ActionOnFailure": "TERMINATE_CLUSTER",
    "HadoopJarStep": {
      "Jar": "command-runner.jar",
      "Args": ["spark-submit", "s3://scripts/etl.py"]
    }
  }]'

# Lifecycle:
# 1. Launch cluster (5 minutes)
# 2. Run ETL job (1 hour)
# 3. Auto-terminate cluster
# 4. Cost: 1.08 hours (vs 24 hours for persistent cluster)
# 5. Daily savings: 95% ✅
```

**5. EMR Managed Scaling (Auto-Scale Task Nodes)**

```bash
# Configure managed scaling
aws emr put-managed-scaling-policy \
  --cluster-id j-XXXXXXXXXXXXX \
  --managed-scaling-policy '{
    "ComputeLimits": {
      "UnitType": "Instances",
      "MinimumCapacityUnits": 2,
      "MaximumCapacityUnits": 50,
      "MaximumOnDemandCapacityUnits": 10,
      "MaximumCoreCapacityUnits": 2
    }
  }'

# Scaling behavior:
# - Job starts: 2 core instances (minimal)
# - YARN pending containers > threshold → Scale up task instances
# - Peak load: 50 task instances (Spot)
# - Job completes → Scale down to 2 core instances
# - Average utilization: 30% (vs 100% fixed capacity)
# - Cost savings: 70% ✅
```

**6. Use Graviton2 Instances (ARM-based, 20% cheaper)**

```bash
# Graviton2 instances (r6g family)
# - r6g.4xlarge: $0.806/hour (vs r5.4xlarge: $1.008/hour)
# - Savings: 20% for same performance
# - Compatible with Spark, Hadoop, Presto

aws emr create-cluster \
  --instance-groups '[{
    "InstanceGroupType": "TASK",
    "InstanceType": "r6g.4xlarge",  # Graviton2
    "InstanceCount": 10,
    "Market": "SPOT"
  }]'

# Cost: 10 × $0.08/hour (Spot) = $0.80/hour
# vs r5.4xlarge Spot: 10 × $0.10/hour = $1.00/hour
# Additional 20% savings ✅
```

**Cost Comparison Summary:**

| Strategy | Hourly Cost | Daily Cost (24h) | Savings |
|----------|-------------|------------------|---------|
| **Baseline (All On-Demand)** | $65.52 | $1,572 | - |
| **+ Spot Task Nodes** | $20.16 | $484 | 69% |
| **+ S3 Data Lake** | $5.40 | $130 | 92% |
| **+ Transient Clusters** | $5.40 × 1h | $5.40 | 99.7% ✅ |
| **+ Graviton2** | $4.32 × 1h | $4.32 | 99.7% |

**Total Savings: 99.7% ($1,572 → $4.32 per day)** ✅

---

**Q9: When should you use Glue vs EMR for ETL workloads?**

**Answer:**

**Decision Matrix:**

| Factor | Use AWS Glue | Use Amazon EMR |
|--------|--------------|----------------|
| **Data Volume** | < 100 GB per job | > 1 TB per job |
| **Job Duration** | < 2 hours | > 2 hours (unlimited) |
| **Framework** | PySpark/Scala only | Spark, Hadoop, Presto, Hive, Flink |
| **Management** | Want fully managed | Need custom cluster config |
| **Team Skills** | Basic PySpark knowledge | Expert Spark/Hadoop admins |
| **Frequency** | Infrequent (hourly/daily) | Continuous processing |
| **Cost Priority** | Simplicity > cost | Cost > simplicity |
| **Startup Time** | 2-3 minutes acceptable | Need immediate processing |

**Detailed Comparison:**

**1. Data Volume and Performance**

```python
# Test: Transform 10 GB CSV → Parquet

# Glue (10 DPUs):
# - Duration: 12 minutes
# - Cost: 10 DPUs × 0.2 hours × $0.44 = $0.88
# - Startup: 2 minutes
# - Total: 14 minutes

# EMR (3-node cluster: 1 master + 2 core r5.xlarge):
# - Duration: 8 minutes
# - Cost: 3 × $0.252/hour × 0.25 hours = $0.19
# - Startup: 5 minutes (if transient)
# - Total: 13 minutes

# Winner for 10 GB: Glue (simpler, no cluster management)

# Test: Transform 1 TB CSV → Parquet

# Glue (100 DPUs, max):
# - Duration: 120 minutes
# - Cost: 100 DPUs × 2 hours × $0.44 = $88
# - Limitation: 100 DPU max ❌

# EMR (30-node cluster: 1 master + 29 core r5.4xlarge):
# - Duration: 25 minutes
# - Cost: 30 × $1.008/hour × 0.42 hours = $12.70
# - Scalable to 1,000+ nodes ✅

# Winner for 1 TB: EMR (5x faster, 85% cheaper)
```

**2. Job Complexity**

```python
# Simple ETL: Read → Filter → Write
# Glue wins (no cluster setup)

from awsglue.context import GlueContext
glueContext = GlueContext(SparkContext.getOrCreate())

df = glueContext.create_dynamic_frame.from_catalog(database="db", table_name="sales")
filtered = df.filter(lambda x: x["amount"] > 100)
glueContext.write_dynamic_frame.from_options(filtered, connection_type="s3", ...)

# Complex ETL: Multi-step Spark job with custom JARs
# EMR wins (full Spark control)

from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("Complex ETL") \
    .config("spark.jars", "s3://jars/custom-udf-1.0.jar") \
    .config("spark.executor.memoryOverhead", "4G") \
    .config("spark.sql.adaptive.enabled", "true") \
    .getOrCreate()

# Custom transformations, UDFs, complex joins
df1 = spark.read.parquet("s3://data/source1/")
df2 = spark.read.parquet("s3://data/source2/")
df3 = spark.read.jdbc(url=jdbc_url, ...)

# 10-step transformation pipeline
result = complex_transformation(df1, df2, df3)
```

**3. Cost Model Comparison**

```
Scenario: Process 50 GB data daily

Glue (10 DPUs × 30 minutes):
├─ Daily: 10 DPUs × 0.5 hours × $0.44 = $2.20/day
├─ Monthly: $2.20 × 30 = $66/month
└─ No idle cost ✅

EMR (persistent cluster, 3 nodes r5.xlarge):
├─ Daily: 3 × $0.252/hour × 24 hours = $18.14/day
├─ Monthly: $18.14 × 30 = $544/month
└─ Idle 23.5 hours/day ❌

EMR (transient cluster, 3 nodes r5.xlarge):
├─ Daily: 3 × $0.252/hour × 0.5 hours = $0.38/day
├─ Monthly: $0.38 × 30 = $11.40/month
├─ Startup overhead: 5 min (10% of job time)
└─ Cheaper than Glue ✅

Winner: EMR transient (if you can manage clusters)
```

**4. Operational Complexity**

```
Glue Operational Tasks:
├─ Create job (1-click)
├─ Schedule job (EventBridge)
└─ Monitor via CloudWatch
    Total DevOps effort: 1 hour/month

EMR Operational Tasks:
├─ Design cluster architecture (instance types, scaling)
├─ Configure cluster (Spark settings, JARs, bootstrap)
├─ Launch cluster (CLI or Console)
├─ Monitor cluster (YARN, Spark UI, Ganglia)
├─ Troubleshoot node failures
├─ Optimize Spark jobs (partitions, caching, etc.)
└─ Terminate cluster (or pay for idle time)
    Total DevOps effort: 10-20 hours/month

Glue wins: 90% less operational overhead
```

**5. Use Case Recommendations**

**Use Glue For:**
```
✅ Scheduled ETL jobs (daily/hourly)
✅ Data catalog and schema discovery (Glue Crawler)
✅ Simple to moderate transformations
✅ Small team without Spark expertise
✅ < 100 GB per job
✅ Ad-hoc data preparation (Glue DataBrew)
✅ Integration with Athena, Redshift Spectrum

Example:
- Daily sales data: S3 CSV → Glue → Parquet → Athena
- Log aggregation: CloudWatch → Firehose → S3 → Glue → Redshift
```

**Use EMR For:**
```
✅ Large-scale data processing (> 1 TB)
✅ Complex Spark/Hadoop workflows
✅ Long-running jobs (> 2 hours)
✅ Machine learning with Spark MLlib
✅ Interactive analysis (Presto, Hive)
✅ Custom Spark configurations
✅ Multiple frameworks (Spark + Presto + Hive)

Example:
- Clickstream analytics: 100 TB/day → EMR Spark → S3
- ML feature engineering: 10 TB dataset → EMR Spark MLlib
- Real-time analytics: Kafka → EMR Flink → Redshift
```

**6. Hybrid Approach (Best of Both Worlds)**

```python
# Use Glue for orchestration + schema discovery
# Use EMR for heavy processing

# Step 1: Glue Crawler discovers schema
crawler = glue.start_crawler(Name='s3-data-crawler')

# Step 2: Glue job launches EMR cluster for processing
emr.run_job_flow(
    Name='Heavy Processing',
    Instances={
        'InstanceGroups': [...]
    },
    Steps=[{
        'Name': 'Process 1 TB data',
        'HadoopJarStep': {
            'Jar': 'command-runner.jar',
            'Args': ['spark-submit', 's3://scripts/process.py']
        }
    }]
)

# Step 3: Glue job loads results into Redshift
glue.start_job_run(JobName='load-to-redshift')

# Result: Glue orchestration + EMR compute power
```

**Decision Flowchart:**

```
Is your data volume > 1 TB?
├─ Yes → EMR
└─ No → Continue

Do you need frameworks besides Spark (Presto, Hive, Flink)?
├─ Yes → EMR
└─ No → Continue

Do you have a dedicated Spark/Hadoop team?
├─ Yes → EMR (more control)
└─ No → Continue

Is job duration > 2 hours?
├─ Yes → EMR
└─ No → Continue

Do you want zero cluster management?
├─ Yes → Glue ✅
└─ No → EMR (for cost optimization)
```

---

**Q10: How do you monitor and troubleshoot Lambda, Batch, and EMR jobs?**

**Answer:**

**Monitoring Strategy for Each Service:**

**1. AWS Lambda Monitoring**

```python
# Enable CloudWatch Logs (automatic)
# Every Lambda execution writes logs to /aws/lambda/<function-name>

# Add custom metrics and structured logging
import json
import time

def lambda_handler(event, context):
    start_time = time.time()
    
    try:
        # Business logic
        result = process_data(event)
        
        # Custom metrics
        duration = time.time() - start_time
        
        # Structured logging (CloudWatch Insights)
        print(json.dumps({
            'event': 'ProcessComplete',
            'duration_ms': duration * 1000,
            'records_processed': len(result),
            'status': 'success'
        }))
        
        return {'statusCode': 200, 'body': json.dumps(result)}
    
    except Exception as e:
        # Error logging
        print(json.dumps({
            'event': 'ProcessError',
            'error': str(e),
            'status': 'failed'
        }))
        
        raise

# CloudWatch Insights query:
# fields @timestamp, event, duration_ms, records_processed
# | filter event = "ProcessComplete"
# | stats avg(duration_ms), sum(records_processed) by bin(5m)
```

**Lambda Metrics to Monitor:**

| Metric | Threshold | Action |
|--------|-----------|--------|
| **Duration** | > 80% of timeout | Increase timeout or optimize code |
| **Error Rate** | > 1% | Investigate errors in logs |
| **Throttles** | > 0 | Request concurrency limit increase |
| **Iterator Age** (Kinesis) | > 60 seconds | Increase batch size or parallelization |
| **Dead Letter Queue** | > 0 messages | Review failed invocations |

**CloudWatch Alarms for Lambda:**

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

# Alarm: High error rate
cloudwatch.put_metric_alarm(
    AlarmName='Lambda-HighErrorRate',
    MetricName='Errors',
    Namespace='AWS/Lambda',
    Dimensions=[{'Name': 'FunctionName', 'Value': 'data-processor'}],
    Statistic='Sum',
    Period=300,  # 5 minutes
    EvaluationPeriods=1,
    Threshold=10,  # > 10 errors in 5 minutes
    ComparisonOperator='GreaterThanThreshold',
    AlarmActions=['arn:aws:sns:us-east-1:123456789012:Lambda-Alerts']
)

# Alarm: High duration (approaching timeout)
cloudwatch.put_metric_alarm(
    AlarmName='Lambda-HighDuration',
    MetricName='Duration',
    Namespace='AWS/Lambda',
    Dimensions=[{'Name': 'FunctionName', 'Value': 'data-processor'}],
    Statistic='Average',
    Period=300,
    EvaluationPeriods=2,
    Threshold=12000,  # > 12 seconds (80% of 15-second timeout)
    ComparisonOperator='GreaterThanThreshold'
)
```

**2. AWS Batch Monitoring**

```bash
# Monitor job status
aws batch describe-jobs --jobs job-id-1 job-id-2 | jq '.jobs[] | {jobId, status, statusReason}'

# List failed jobs
aws batch list-jobs \
  --job-queue log-processing-queue \
  --job-status FAILED \
  --query 'jobSummaryList[*].[jobId,jobName,statusReason]'

# Check compute environment scaling
aws batch describe-compute-environments \
  --compute-environments batch-compute | \
  jq '.computeEnvironments[0].computeResources | {desired: .desiredvCpus, min: .minvCpus, max: .maxvCpus}'

# View job logs (CloudWatch Logs)
aws logs tail /aws/batch/job --follow --log-stream-name <job-id>
```

**Batch Metrics to Monitor:**

| Metric | Source | Threshold |
|--------|--------|-----------|
| **Job Queue Depth** | AWS Batch API | > 100 (may need more capacity) |
| **Job Duration** | CloudWatch Logs | > Expected duration (performance issue) |
| **Failed Jobs** | AWS Batch API | > 5% failure rate |
| **Compute Environment Scaling** | CloudWatch | desiredvCpUs < required (under-provisioned) |
| **Spot Interruption Rate** | CloudWatch Events | > 10% (choose different instance types) |

**Batch Troubleshooting:**

```python
# Common issues and fixes

# Issue 1: Jobs stuck in RUNNABLE (not starting)
# Cause: Insufficient compute capacity
# Fix: Increase maxvCpus or check instance limits

aws batch update-compute-environment \
  --compute-environment batch-compute \
  --compute-resources maxvCpus=2000

# Issue 2: Container fails to start
# Cause: ECR image pull error, insufficient IAM permissions
# Check: CloudWatch Logs for error details

aws logs get-log-events \
  --log-group-name /aws/batch/job \
  --log-stream-name <job-id>

# Issue 3: High failure rate on Spot Instances
# Cause: Frequent Spot interruptions
# Fix: Use capacity-optimized allocation strategy

aws batch update-compute-environment \
  --compute-environment batch-compute \
  --compute-resources '{
    "allocationStrategy": "SPOT_CAPACITY_OPTIMIZED"
  }'
```

**3. Amazon EMR Monitoring**

```bash
# Cluster-level metrics
aws emr describe-cluster --cluster-id j-XXXXXXXXXXXXX | \
  jq '{state: .Cluster.Status.State, reason: .Cluster.Status.StateChangeReason}'

# Step status
aws emr list-steps --cluster-id j-XXXXXXXXXXXXX | \
  jq '.Steps[] | {name: .Name, state: .Status.State, duration: .Status.Timeline}'

# EMR-managed metrics (CloudWatch)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElasticMapReduce \
  --metric-name AppsRunning \
  --dimensions Name=JobFlowId,Value=j-XXXXXXXXXXXXX \
  --start-time 2024-06-22T00:00:00Z \
  --end-time 2024-06-22T23:59:59Z \
  --period 300 \
  --statistics Average
```

**EMR Metrics to Monitor:**

| Metric | Namespace | Threshold | Issue |
|--------|-----------|-----------|-------|
| **YARNMemoryAvailablePercentage** | AWS/ElasticMapReduce | < 20% | Memory pressure, add nodes |
| **ContainerPendingRatio** | AWS/ElasticMapReduce | > 0.5 | Cluster under-provisioned |
| **HDFSUtilization** | AWS/ElasticMapReduce | > 80% | Add storage (or use S3) |
| **CoreNodesRunning** | AWS/ElasticMapReduce | < Expected | Node failure |
| **IsIdle** | AWS/ElasticMapReduce | = 1 (for > 1 hour) | Terminate cluster (cost) |

**EMR Web UIs (SSH Tunneling):**

```bash
# Create SSH tunnel to access Spark UI, YARN, Ganglia
ssh -i emr-keypair.pem \
  -L 8088:localhost:8088 \  # YARN ResourceManager
  -L 4040:localhost:4040 \  # Spark UI
  -L 8890:localhost:8890 \  # Ganglia
  hadoop@<master-public-dns>

# Access in browser:
# - YARN: http://localhost:8088
# - Spark UI: http://localhost:4040
# - Ganglia: http://localhost:8890/ganglia
```

**Spark Job Troubleshooting (EMR):**

```python
# Common Spark issues

# Issue 1: OutOfMemoryError (Executor)
# Symptoms: Tasks fail with OOM errors
# Fix: Increase executor memory or reduce partition size

spark-submit \
  --conf spark.executor.memory=16G \
  --conf spark.executor.memoryOverhead=4G \
  --conf spark.sql.shuffle.partitions=1000 \  # More partitions = smaller partitions
  s3://scripts/job.py

# Issue 2: Slow shuffle (Data skew)
# Symptoms: 99% of tasks complete quickly, 1% takes hours
# Fix: Repartition by key with better distribution

df_skewed = df.groupBy("user_id").agg(...)  # user_id has skew
df_fixed = df.withColumn("salt", (F.rand() * 10).cast("int")) \
             .groupBy("user_id", "salt").agg(...) \
             .groupBy("user_id").agg(...)  # Re-aggregate after salting

# Issue 3: Driver OutOfMemoryError
# Symptoms: collect(), toPandas() fail
# Fix: Avoid collecting large datasets to driver

# Bad:
df_large = spark.read.parquet("s3://data/100TB/")
result = df_large.collect()  # OOM ❌

# Good:
df_large.write.parquet("s3://output/")  # Write to S3 ✅
```

**Centralized Monitoring Dashboard:**

```python
# Use CloudWatch Dashboard for unified view

import boto3

cloudwatch = boto3.client('cloudwatch')

cloudwatch.put_dashboard(
    DashboardName='DataPipelineMonitoring',
    DashboardBody=json.dumps({
        "widgets": [
            {
                "type": "metric",
                "properties": {
                    "title": "Lambda Invocations",
                    "metrics": [
                        ["AWS/Lambda", "Invocations", {"stat": "Sum"}],
                        [".", "Errors", {"stat": "Sum"}]
                    ]
                }
            },
            {
                "type": "metric",
                "properties": {
                    "title": "Batch Job Queue Depth",
                    "metrics": [
                        ["AWS/Batch", "ApproximateNumberOfMessagesVisible"]
                    ]
                }
            },
            {
                "type": "metric",
                "properties": {
                    "title": "EMR Cluster Memory",
                    "metrics": [
                        ["AWS/ElasticMapReduce", "YARNMemoryAvailablePercentage"]
                    ]
                }
            }
        ]
    })
)
```

**Summary:**

| Service | Primary Monitoring | Troubleshooting Tool |
|---------|-------------------|----------------------|
| **Lambda** | CloudWatch Logs + Metrics | CloudWatch Insights, X-Ray |
| **Batch** | AWS Batch API + CloudWatch | Job logs, Compute environment status |
| **EMR** | CloudWatch + Spark UI | YARN UI, Ganglia, SSH debug |

---

(Continue with Q11-Q20 scenario questions...)

## SCENARIO-BASED QUESTIONS (Q11-Q20)

**Q11: Build a Serverless Real-Time Analytics Pipeline with Lambda + Kinesis**

**Scenario:** Process 1 million events/minute from IoT devices, aggregate metrics in real-time, and store in DynamoDB for dashboards.

**Architecture:** Kinesis Data Streams → Lambda (real-time aggregation) → DynamoDB → API Gateway + QuickSight

**Key Implementation Points:**
- **Lambda Configuration:** 1,000 concurrent executions, 1,024 MB memory, batching 500 records
- **Kinesis:** 100 shards (10,000 records/sec per shard capacity)
- **DynamoDB:** On-demand mode for variable traffic patterns
- **Cost:** $450/month (Lambda: $200, Kinesis: $150, DynamoDB: $100)
- **HA:** Multi-AZ automatic (Lambda, Kinesis, DynamoDB)
- **Monitoring:** CloudWatch metrics for Iterator Age, throttles, DynamoDB consumed capacity

---

**Q12: Batch Processing Pipeline with AWS Batch + Spot Instances**

**Scenario:** Process 50,000 video files daily (each 2-hour transcoding job) using AWS Batch with 90% cost savings.

**Architecture:** S3 → EventBridge (daily trigger) → Batch (50,000 array job) → FFmpeg container → S3

**Results:**
- **Processing Time:** 4 hours (12,500 parallel jobs on Spot instances)
- **Cost:** $200/day (vs $2,000 On-Demand, 90% savings)
- **Fault Tolerance:** Automatic retry on Spot interruption (3 attempts)
- **Scaling:** 0 → 50,000 vCPUs → 0 (auto-scaling)

---

**Q13: Big Data Analytics with EMR + Spark for 500 TB Dataset**

**Scenario:** Daily processing of 500 TB clickstream data using EMR Spark cluster with auto-scaling.

**Implementation:**
- **Cluster:** 1 master (m5.2xlarge) + 50 core (r5.4xlarge) + 200 task Spot (r5.4xlarge)
- **Processing Time:** 3 hours (500 TB → 2 TB aggregated Parquet)
- **Cost:** $120/day (Spot: 70% savings)
- **Optimization:** S3 data lake (separate compute/storage), EMR managed scaling
- **Performance:** Spark with 200 executors, 16 GB memory each, adaptive query execution enabled

---

**Q14: Serverless ETL with AWS Glue for Multi-Source Data Integration**

**Scenario:** Daily ETL from 5 sources (RDS, Redshift, S3, DynamoDB, APIs) into unified data warehouse.

**Implementation:**
- **Glue Crawlers:** Auto-discover schemas from all 5 sources
- **Glue Jobs:** 20 DPUs processing 200 GB/day
- **Data Catalog:** Central metadata repository for Athena/Redshift Spectrum
- **Cost:** $88/month (20 DPUs × 0.5 hours × 30 days × $0.44)
- **Schedule:** EventBridge trigger at 2 AM daily
- **Monitoring:** Glue job metrics, CloudWatch Logs for errors

---

**Q15: Multi-Step Data Pipeline with Step Functions Orchestration**

**Scenario:** Orchestrate complex ETL pipeline: Validate → Transform (parallel) → Load → Notify.

**Workflow:**
1. Lambda validates S3 file (schema check)
2. **Parallel:** Glue transforms to Parquet + Lambda generates statistics
3. Lambda loads to Redshift using COPY
4. SNS notifies stakeholders on success/failure
5. **Error Handling:** Retry 3 times with exponential backoff, dead letter queue for failures

**Cost:** $0.025 per execution (8 state transitions)
**Benefits:** Visual workflow, automatic error handling, audit trail

---

**Q16: Lambda Performance Optimization for Large-Scale Data Processing**

**Scenario:** Optimize Lambda to process 10,000 files/hour (each 50 MB CSV → Parquet conversion).

**Optimizations:**
- **Memory:** 1,792 MB (full vCPU, 75% faster than 512 MB)
- **Lambda Layers:** Pandas/PyArrow in layer (deployment package < 10 MB)
- **Parallel Invocation:** S3 Batch Operations (1,000 concurrent Lambdas)
- **Results:** Processing time: 5 minutes (vs 8 hours sequential), Cost: $2 per 10,000 files

---

**Q17: Cost Optimization for EMR Workloads (Spot + Graviton + S3)**

**Scenario:** Reduce EMR costs by 95% using Spot Instances, Graviton2, S3 data lake, and transient clusters.

**Strategies:**
- **Spot Instances:** 90% discount on task nodes (200 instances)
- **Graviton2 (r6g):** 20% cheaper than x86 (r5)
- **S3 Data Lake:** Minimal core nodes (no HDFS), separate compute/storage
- **Transient Clusters:** Auto-terminate after job (vs persistent 24/7)
- **Cost:** $4.32/day (vs $1,572 baseline, 99.7% savings)

---

**Q18: Monitoring and Troubleshooting EMR Spark Performance Issues**

**Scenario:** Debug slow Spark job processing 100 TB data (takes 12 hours, should be 2 hours).

**Troubleshooting Steps:**
1. **Spark UI:** Identify data skew (1 task takes 10 hours, others 10 minutes)
2. **Fix:** Repartition with salting to distribute data evenly
3. **YARN Metrics:** Memory pressure (95% utilization) → Add core nodes
4. **Optimization:** Enable adaptive query execution, increase shuffle partitions
5. **Result:** 2 hours processing time (6x faster)

---

**Q19: Disaster Recovery for Data Processing Pipelines**

**Scenario:** Design DR strategy for mission-critical ETL pipeline (RPO < 1 hour, RTO < 30 minutes).

**DR Architecture:**
- **Primary:** us-east-1 (Lambda + Glue + Redshift)
- **DR:** us-west-2 (standby Redshift cluster, replicated S3 data)
- **Failover:** Route 53 health checks, automatic DNS failover
- **Data Replication:** S3 Cross-Region Replication (RPO: 15 minutes)
- **Testing:** Monthly DR drills, automated failover scripts
- **RTO:** 25 minutes (Redshift snapshot restore: 20 min + validation: 5 min)

---

**Q20: Hybrid Compute Strategy (Lambda + Batch + EMR + Glue)**

**Scenario:** Choose optimal compute service for each workload in multi-tenant analytics platform.

**Workload Distribution:**
- **Lambda:** File validation, small transformations (< 1 GB, < 15 min)
- **Glue:** Daily ETL jobs (1-100 GB, serverless simplicity)
- **Batch:** Parallel batch processing (10,000 jobs, Spot instances)
- **EMR:** Big data analytics (> 1 TB, Spark/Presto frameworks)

**Cost Comparison (monthly):**
- All-Lambda: $15,000 (not feasible for large datasets)
- All-EMR: $8,000 (overkill for small jobs)
- Hybrid: $2,500 (right tool for each job, 69% savings)

**Decision Matrix:**
| Data Size | Duration | Service | Monthly Cost |
|-----------|----------|---------|--------------|
| < 1 GB | < 15 min | Lambda | $500 |
| 1-100 GB | < 2 hours | Glue | $800 |
| Any | Parallel batch | Batch | $400 |
| > 1 TB | Unlimited | EMR | $800 |
| **Total** | - | **Hybrid** | **$2,500** |

---

## MODULE SUMMARY

**Module 5: Compute** covered the complete suite of AWS compute services for data engineering:

**Services Mastered:**
- ✅ AWS Lambda (serverless event processing)
- ✅ AWS Batch (parallel batch computing)
- ✅ Amazon EMR (big data frameworks)
- ✅ AWS Glue (serverless ETL)
- ✅ AWS Step Functions (workflow orchestration)

**Key Takeaways:**
1. **Lambda:** Best for < 1 GB files, < 15 minutes, event-driven workloads ($0.0024 per 1M rows)
2. **Batch:** Optimal for parallel batch jobs with 90% Spot savings ($1.38 for 10,000 jobs)
3. **EMR:** Required for > 1 TB datasets, Spark/Hadoop frameworks ($16 for 100 TB processing)
4. **Glue:** Serverless ETL sweet spot for 1-100 GB ($0.44 for 10 GB transformation)
5. **Step Functions:** Orchestrate multi-step workflows with error handling ($0.025 per 1,000 transitions)

**Cost Optimization:**
- EMR Spot Instances: 70-90% savings
- Lambda memory tuning: 20% cheaper at optimal memory (1,792 MB)
- Transient EMR clusters: 95% savings vs persistent
- Hybrid approach: 69% savings using right service for each job

**Performance Optimization:**
- Lambda: 1,792 MB memory for full vCPU (75% faster)
- Batch: Auto-scaling 0 → 10,000 vCPUs (99% faster than sequential)
- EMR: Managed scaling + Spot task nodes (83% faster, 70% cheaper)
- Glue: Auto-scaling DPUs (no cluster management overhead)

**Next Module:** Module 6: Containers (ECS, EKS, Fargate for data engineering)

---

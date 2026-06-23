# MODULE 5: Compute Services - GUI Step-by-Step Hands-On Guide

## Table of Contents
- [Exercise 5.1: Serverless ETL with AWS Lambda](#exercise-51-serverless-etl-with-aws-lambda)
- [Exercise 5.2: Batch Processing with AWS Batch](#exercise-52-batch-processing-with-aws-batch)
- [Exercise 5.3: Big Data Processing with Amazon EMR](#exercise-53-big-data-processing-with-amazon-emr)

---

# Exercise 5.1: Serverless ETL with AWS Lambda

**Duration:** 2-3 hours | **Estimated Cost:** ~$5-10 for lab session

## 🎯 Learning Objectives
- Build serverless ETL pipeline with Lambda
- Validate CSV files and transform to Parquet
- Use Lambda layers for dependencies
- Configure S3 event triggers
- Monitor with CloudWatch Logs

---

## PART 1: Create S3 Buckets and Sample Data

### Step 1: Create Landing Bucket
1. **S3 Console** → **"Create bucket"**
2. **Bucket name:** `data-landing-[your-account-id]`
   - Example: `data-landing-123456789012`
3. **Region:** us-east-1
4. **Versioning:** Enable
5. **Encryption:** SSE-S3
6. Click **"Create bucket"**

### Step 2: Create Processed Bucket
1. Click **"Create bucket"** again
2. **Bucket name:** `data-processed-[your-account-id]`
3. Same settings as landing bucket
4. Click **"Create bucket"**

### Step 3: Upload Sample CSV File
1. Click on `data-landing-[id]` bucket
2. Click **"Upload"** → **"Add files"**
3. Create a sample CSV file locally first:

**On your computer, create: `sales-data-2024-06.csv`**
```csv
order_id,customer_id,product_id,quantity,unit_price,order_date,region
1001,C001,P123,5,29.99,2024-06-01,US-East
1002,C002,P456,2,149.99,2024-06-02,US-West
1003,C003,P789,1,599.99,2024-06-03,EU-West
1004,C001,P123,3,29.99,2024-06-04,US-East
1005,C004,P456,1,149.99,2024-06-05,APAC
1006,C005,P999,10,9.99,2024-06-06,US-Central
1007,C002,P123,2,29.99,2024-06-07,US-West
1008,C006,P789,1,599.99,2024-06-08,EU-Central
```

4. Upload this file to the landing bucket
5. Click **"Upload"**

---

## PART 2: Create IAM Role for Lambda

### Step 4: Create Lambda Execution Role
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** Select **"AWS service"**
3. **Use case:** Select **"Lambda"**
4. Click **"Next"**

### Step 5: Attach Policies
1. Search and select these policies:
   - ✅ **AWSLambdaBasicExecutionRole** (CloudWatch Logs)
   - ✅ **AmazonS3FullAccess** (S3 read/write)
2. Click **"Next"**
3. **Role name:** Type: `DataPipelineLambdaRole`
4. Click **"Create role"**

---

## PART 3: Create Lambda Function - Data Validator

### Step 6: Create Validator Function
1. **Lambda Console** → **"Create function"**
2. **Function name:** `DataValidator`
3. **Runtime:** Select **"Python 3.12"**
4. **Permissions:** Expand **"Change default execution role"**
5. Select **"Use an existing role"**
6. **Existing role:** Select `DataPipelineLambdaRole`
7. Click **"Create function"**

### Step 7: Write Validator Code
Replace the default code with:

```python
import json
import boto3
import urllib.parse
from datetime import datetime

s3 = boto3.client('s3')

def lambda_handler(event, context):
    print(f"Received event: {json.dumps(event)}")
    
    # Parse S3 event
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])
        
        print(f"Validating file: s3://{bucket}/{key}")
        
        # Only process CSV files
        if not key.endswith('.csv'):
            print(f"Skipping non-CSV file")
            continue
        
        # Validate file
        validation_result = validate_csv(bucket, key)
        
        # Tag object with validation status
        s3.put_object_tagging(
            Bucket=bucket,
            Key=key,
            Tagging={
                'TagSet': [
                    {
                        'Key': 'ValidationStatus',
                        'Value': 'VALID' if validation_result['is_valid'] else 'INVALID'
                    },
                    {
                        'Key': 'ValidationTimestamp',
                        'Value': datetime.now().isoformat()
                    },
                    {
                        'Key': 'RowCount',
                        'Value': str(validation_result.get('row_count', 0))
                    }
                ]
            }
        )
        
        if validation_result['is_valid']:
            print(f"✓ File is VALID - {validation_result['row_count']} rows")
        else:
            print(f"✗ File is INVALID - {validation_result['errors']}")
            # In production: send SNS alert
        
        return {
            'statusCode': 200,
            'body': json.dumps(validation_result)
        }

def validate_csv(bucket, key):
    """Validate CSV file structure and data quality"""
    try:
        # Download file
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read().decode('utf-8')
        
        lines = content.strip().split('\n')
        
        if len(lines) < 2:
            return {
                'is_valid': False,
                'errors': 'File has no data rows'
            }
        
        # Check headers
        headers = lines[0].split(',')
        required_headers = ['order_id', 'customer_id', 'product_id', 'quantity', 'unit_price']
        
        missing_headers = [h for h in required_headers if h not in headers]
        if missing_headers:
            return {
                'is_valid': False,
                'errors': f'Missing required headers: {missing_headers}'
            }
        
        # Validate data rows
        row_count = len(lines) - 1  # Exclude header
        
        # Check for empty values
        for i, line in enumerate(lines[1:], start=2):
            values = line.split(',')
            if len(values) != len(headers):
                return {
                    'is_valid': False,
                    'errors': f'Row {i} has {len(values)} columns, expected {len(headers)}'
                }
            
            # Check for empty required fields
            for idx, header in enumerate(headers):
                if header in required_headers and not values[idx].strip():
                    return {
                        'is_valid': False,
                        'errors': f'Row {i} has empty value for required field: {header}'
                    }
        
        # All validations passed
        return {
            'is_valid': True,
            'row_count': row_count,
            'headers': headers
        }
        
    except Exception as e:
        return {
            'is_valid': False,
            'errors': f'Exception during validation: {str(e)}'
        }
```

Click **"Deploy"**

### Step 8: Configure Function Settings
1. Go to **"Configuration"** tab → **"General configuration"**
2. Click **"Edit"**
3. **Timeout:** Change to **60 seconds**
4. **Memory:** **512 MB**
5. Click **"Save"**

### Step 9: Add S3 Trigger
1. Click **"Add trigger"** button
2. **Select a source:** **"S3"**
3. **Bucket:** Select `data-landing-[your-account-id]`
4. **Event type:** **"All object create events"**
5. **Suffix:** `.csv` (only CSV files)
6. ✅ Acknowledge recursive invocation warning
7. Click **"Add"**

### Step 10: Test Validator Function
1. Go back to **S3 Console**
2. Navigate to `data-landing-[id]` bucket
3. Upload the `sales-data-2024-06.csv` file again (or rename and upload)
4. Go to **Lambda Console** → `DataValidator` → **"Monitor"** tab
5. Click **"View CloudWatch logs"**
6. Click on the latest log stream
7. You should see:
   ```
   Validating file: s3://data-landing-123/sales-data-2024-06.csv
   ✓ File is VALID - 8 rows
   ```

### Step 11: Verify S3 Object Tags
1. Go to S3, click on the uploaded file
2. Go to **"Properties"** tab
3. Scroll to **"Tags"**
4. You should see:
   - **ValidationStatus:** VALID
   - **ValidationTimestamp:** 2024-06-24T...
   - **RowCount:** 8

---

## PART 4: Create Lambda Function - CSV to Parquet Transformer

### Step 12: Create Lambda Layer for Pandas/PyArrow
Since pandas and pyarrow are large libraries, we need a Lambda Layer.

**Option A: Use AWS Data Wrangler Layer (Recommended)**
1. **Lambda Console** → **"Layers"** → **"Create layer"**
2. Visit: https://github.com/aws/aws-sdk-pandas/releases
3. Find the latest AWS Data Wrangler layer ARN for your region
4. Example ARN: `arn:aws:lambda:us-east-1:336392948345:layer:AWSSDKPandas-Python312:7`

**Option B: Build Custom Layer (Advanced)**
Skip for now, use Option A.

### Step 13: Create Transformer Function
1. **Lambda Console** → **"Create function"**
2. **Function name:** `CSVToParquetTransformer`
3. **Runtime:** **"Python 3.12"**
4. **Execution role:** **"Use an existing role"** → `DataPipelineLambdaRole`
5. Click **"Create function"**

### Step 14: Add AWS Data Wrangler Layer
1. Scroll down to **"Layers"** section
2. Click **"Add a layer"**
3. **Choose a layer:** Select **"AWS layers"**
4. **AWS layers:** Search for **"AWSSDKPandas-Python312"**
5. Select the latest version
6. Click **"Add"**

**If not available, use Specify an ARN:**
1. Select **"Specify an ARN"**
2. Enter: `arn:aws:lambda:us-east-1:336392948345:layer:AWSSDKPandas-Python312:7`
   - (Replace region if different)
3. Click **"Add"**

### Step 15: Write Transformer Code
Replace the code:

```python
import json
import boto3
import awswrangler as wr
import pandas as pd
import urllib.parse
from datetime import datetime

s3 = boto3.client('s3')

def lambda_handler(event, context):
    print(f"Received event: {json.dumps(event)}")
    
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])
        
        # Only process validated CSV files
        if not key.endswith('.csv'):
            continue
        
        # Check validation status from tags
        try:
            tags_response = s3.get_object_tagging(Bucket=bucket, Key=key)
            tags = {tag['Key']: tag['Value'] for tag in tags_response['TagSet']}
            
            if tags.get('ValidationStatus') != 'VALID':
                print(f"Skipping invalid file: {key}")
                continue
        except Exception as e:
            print(f"No validation tags found, skipping: {str(e)}")
            continue
        
        print(f"Transforming: s3://{bucket}/{key}")
        
        # Transform CSV to Parquet
        transform_to_parquet(bucket, key)
    
    return {
        'statusCode': 200,
        'body': json.dumps('Transformation complete')
    }

def transform_to_parquet(bucket, key):
    """Transform CSV to Parquet with optimizations"""
    try:
        # Read CSV from S3
        s3_path = f"s3://{bucket}/{key}"
        print(f"Reading CSV from: {s3_path}")
        
        df = wr.s3.read_csv(path=s3_path)
        
        print(f"Loaded {len(df)} rows, {len(df.columns)} columns")
        
        # Data transformations
        # 1. Add derived columns
        df['total_amount'] = df['quantity'] * df['unit_price']
        df['processing_timestamp'] = datetime.now()
        
        # 2. Optimize data types
        df['order_id'] = df['order_id'].astype('int32')
        df['quantity'] = df['quantity'].astype('int16')
        df['unit_price'] = df['unit_price'].astype('float32')
        df['total_amount'] = df['total_amount'].astype('float32')
        
        # 3. Parse dates
        df['order_date'] = pd.to_datetime(df['order_date'])
        
        print(f"After transformation: {df.shape}")
        print(f"Memory usage: {df.memory_usage(deep=True).sum() / 1024:.2f} KB")
        
        # Prepare output path
        output_bucket = bucket.replace('landing', 'processed')
        filename = key.split('/')[-1].replace('.csv', '.parquet')
        output_path = f"s3://{output_bucket}/{filename}"
        
        # Write Parquet with compression
        wr.s3.to_parquet(
            df=df,
            path=output_path,
            compression='snappy',
            index=False
        )
        
        print(f"✓ Parquet written to: {output_path}")
        
        # Add metadata tags
        s3.put_object_tagging(
            Bucket=output_bucket,
            Key=filename,
            Tagging={
                'TagSet': [
                    {'Key': 'SourceFile', 'Value': key},
                    {'Key': 'TransformTimestamp', 'Value': datetime.now().isoformat()},
                    {'Key': 'RowCount', 'Value': str(len(df))},
                    {'Key': 'Format', 'Value': 'Parquet'},
                    {'Key': 'Compression', 'Value': 'Snappy'}
                ]
            }
        )
        
        return True
        
    except Exception as e:
        print(f"Error transforming file: {str(e)}")
        import traceback
        traceback.print_exc()
        raise

```

Click **"Deploy"**

### Step 16: Configure Transformer Settings
1. **"Configuration"** → **"General configuration"** → **"Edit"**
2. **Timeout:** **300 seconds** (5 minutes)
3. **Memory:** **3008 MB** (pandas needs more memory)
4. **Ephemeral storage:** **1024 MB**
5. Click **"Save"**

### Step 17: Add S3 Trigger for Transformer
1. Click **"Add trigger"**
2. **Source:** **"S3"**
3. **Bucket:** `data-landing-[your-account-id]`
4. **Event type:** **"All object create events"**
5. **Suffix:** `.csv`
6. Click **"Add"**

---

## PART 5: Test End-to-End Pipeline

### Step 18: Upload New CSV File
Create a larger test file: `sales-data-2024-07.csv`

```csv
order_id,customer_id,product_id,quantity,unit_price,order_date,region
2001,C010,P111,15,19.99,2024-07-01,US-East
2002,C011,P222,8,39.99,2024-07-02,EU-West
2003,C012,P333,3,99.99,2024-07-03,APAC
2004,C013,P444,12,14.99,2024-07-04,US-Central
2005,C014,P555,6,29.99,2024-07-05,US-West
2006,C015,P666,20,9.99,2024-07-06,EU-Central
2007,C010,P111,5,19.99,2024-07-07,US-East
2008,C016,P777,1,499.99,2024-07-08,APAC
2009,C017,P888,10,24.99,2024-07-09,US-East
2010,C018,P999,7,34.99,2024-07-10,EU-West
```

Upload to `data-landing-[id]` bucket

### Step 19: Monitor Lambda Executions
1. **CloudWatch Console** → **"Logs"** → **"Log groups"**
2. Find: `/aws/lambda/DataValidator`
3. Click on latest log stream
4. Verify validation succeeded

5. Go back, find: `/aws/lambda/CSVToParquetTransformer`
6. Click latest log stream
7. You should see:
   ```
   Reading CSV from: s3://data-landing-123/sales-data-2024-07.csv
   Loaded 10 rows, 7 columns
   After transformation: (10, 9)
   ✓ Parquet written to: s3://data-processed-123/sales-data-2024-07.parquet
   ```

### Step 20: Verify Parquet File in S3
1. **S3 Console** → `data-processed-[id]` bucket
2. You should see: `sales-data-2024-07.parquet`
3. Click on it → **"Properties"** → **"Tags"**
4. Verify tags:
   - **SourceFile:** sales-data-2024-07.csv
   - **Format:** Parquet
   - **RowCount:** 10
   - **Compression:** Snappy

### Step 21: Download and Inspect Parquet (Optional)
**Using Python locally:**
```python
import pandas as pd
import pyarrow.parquet as pq

# Download from S3 first
# Then read locally
df = pd.read_parquet('sales-data-2024-07.parquet')
print(df.head())
print(df.dtypes)
print(f"Columns: {list(df.columns)}")

# You should see:
# - Original 7 columns
# - New columns: total_amount, processing_timestamp
```

---

## PART 6: Monitor and Optimize

### Step 22: View Lambda Metrics
1. **Lambda Console** → Select `CSVToParquetTransformer`
2. Go to **"Monitor"** tab
3. View metrics:
   - **Invocations:** Number of times executed
   - **Duration:** Average execution time
   - **Errors:** Should be 0
   - **Throttles:** Should be 0
   - **Concurrent executions:** Peak concurrency

### Step 23: Create CloudWatch Dashboard
1. **CloudWatch Console** → **"Dashboards"** → **"Create dashboard"**
2. **Dashboard name:** `DataPipelineDashboard`
3. Click **"Create dashboard"**

### Step 24: Add Lambda Metrics Widget
1. Click **"Add widget"**
2. Select **"Line"** graph
3. **Metrics** → **"Lambda"** → **"By Function Name"**
4. Select both functions: `DataValidator` and `CSVToParquetTransformer`
5. Select metrics: **"Invocations"**, **"Duration"**, **"Errors"**
6. Click **"Create widget"**

### Step 25: Add S3 Metrics Widget
1. Click **"Add widget"**
2. Select **"Number"**
3. **Metrics** → **"S3"** → **"Storage Metrics"**
4. Select your buckets
5. Metric: **"NumberOfObjects"**
6. Click **"Create widget"**

### Step 26: Save Dashboard
1. Click **"Save dashboard"** (top right)
2. You now have a real-time pipeline monitoring dashboard!

---

## ✅ Exercise 5.1 Completion Checklist

- [ ] Created S3 landing and processed buckets
- [ ] Created IAM role for Lambda with S3 permissions
- [ ] Created DataValidator Lambda function
- [ ] Implemented CSV validation logic
- [ ] Configured S3 trigger for validator
- [ ] Tested validation with sample CSV
- [ ] Verified S3 object tags applied
- [ ] Added AWS Data Wrangler layer for pandas/pyarrow
- [ ] Created CSVToParquetTransformer Lambda function
- [ ] Implemented data transformation and Parquet conversion
- [ ] Configured memory and timeout for transformer
- [ ] Tested end-to-end pipeline
- [ ] Verified Parquet output in processed bucket
- [ ] Created CloudWatch dashboard for monitoring

**🎉 Congratulations!** You've completed Exercise 5.1!

---

# Exercise 5.2: Batch Processing with AWS Batch

**Duration:** 3-4 hours | **Estimated Cost:** ~$15-25 for lab session

## 🎯 Learning Objectives
- Create Docker container for batch processing
- Push container to Amazon ECR
- Configure AWS Batch compute environment
- Submit and monitor batch jobs
- Process large-scale data in parallel

---

## PART 1: Create Docker Container for Log Processing

### Step 1: Install Docker (If Not Installed)
**On macOS:**
```bash
# Download Docker Desktop from docker.com
# Or use Homebrew:
brew install --cask docker
```

**On Windows:**
Download Docker Desktop from docker.com

**On Linux:**
```bash
sudo dnf install docker -y  # Amazon Linux
sudo systemctl start docker
sudo usermod -a -G docker $USER
```

### Step 2: Create Project Directory
```bash
mkdir log-processor
cd log-processor
```

### Step 3: Create Python Processing Script
Create file: `process_logs.py`

```python
#!/usr/bin/env python3
import json
import gzip
import boto3
from datetime import datetime
from collections import defaultdict
import os
import sys

s3 = boto3.client('s3')

def process_log_file(bucket, key):
    """Process a single gzipped JSON log file"""
    print(f"Processing: s3://{bucket}/{key}")
    
    # Download file
    local_file = '/tmp/logfile.gz'
    s3.download_file(bucket, key, local_file)
    
    # Parse logs
    metrics = defaultdict(lambda: {
        'count': 0,
        'total_duration_ms': 0,
        'errors': 0,
        'success': 0
    })
    
    with gzip.open(local_file, 'rt') as f:
        for line_num, line in enumerate(f, 1):
            try:
                log = json.loads(line)
                
                # Extract metrics
                endpoint = log.get('endpoint', 'unknown')
                duration = log.get('duration_ms', 0)
                status = log.get('status_code', 0)
                
                # Aggregate
                metrics[endpoint]['count'] += 1
                metrics[endpoint]['total_duration_ms'] += duration
                
                if status >= 400:
                    metrics[endpoint]['errors'] += 1
                else:
                    metrics[endpoint]['success'] += 1
                    
            except json.JSONDecodeError:
                print(f"Warning: Invalid JSON on line {line_num}")
                continue
    
    print(f"Processed {line_num} log entries")
    
    # Calculate averages
    summary = {}
    for endpoint, data in metrics.items():
        summary[endpoint] = {
            'total_requests': data['count'],
            'avg_duration_ms': data['total_duration_ms'] / data['count'] if data['count'] > 0 else 0,
            'error_rate': data['errors'] / data['count'] if data['count'] > 0 else 0,
            'success_count': data['success'],
            'error_count': data['errors']
        }
    
    # Upload results
    output_key = key.replace('raw/', 'processed/').replace('.gz', '.json')
    s3.put_object(
        Bucket=bucket,
        Key=output_key,
        Body=json.dumps(summary, indent=2)
    )
    
    print(f"✓ Results uploaded to: s3://{bucket}/{output_key}")
    return summary

if __name__ == '__main__':
    # Get parameters from environment
    bucket = os.environ.get('S3_BUCKET')
    key = os.environ.get('LOG_FILE_KEY')
    
    if not bucket or not key:
        print("Error: S3_BUCKET and LOG_FILE_KEY environment variables required")
        sys.exit(1)
    
    result = process_log_file(bucket, key)
    print(f"Summary: Processed {len(result)} endpoints")
```

### Step 4: Create Dockerfile
Create file: `Dockerfile`

```dockerfile
FROM python:3.11-slim

# Install dependencies
RUN pip install --no-cache-dir boto3 pandas pyarrow

# Copy processing script
COPY process_logs.py /app/process_logs.py
RUN chmod +x /app/process_logs.py

# Set working directory
WORKDIR /app

# Run script
ENTRYPOINT ["python", "/app/process_logs.py"]
```

### Step 5: Build Docker Image Locally
```bash
# Build image
docker build -t log-processor:latest .

# Verify image created
docker images | grep log-processor
```

---

## PART 2: Push Container to Amazon ECR

### Step 6: Create ECR Repository
1. **AWS Console** → Search **"ECR"** (Elastic Container Registry)
2. Click **"Amazon Elastic Container Registry"**
3. In left sidebar, click **"Repositories"**
4. Click **"Create repository"** button

### Step 7: Configure Repository
1. **Visibility settings:** Select **"Private"**
2. **Repository name:** Type: `log-processor`
3. **Tag immutability:** Leave as **"Disabled"**
4. **Scan on push:** ✅ Enable (scans for vulnerabilities)
5. **Encryption:** Leave as **"AES-256"**
6. Click **"Create repository"**

### Step 8: Get Push Commands
1. Click on the repository: `log-processor`
2. Click **"View push commands"** button (top right)
3. You'll see 4 commands - copy them

### Step 9: Authenticate Docker to ECR
In your terminal (replace with your region and account ID):

```bash
# Step 1: Authenticate
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com
```

You should see: **"Login Succeeded"**

### Step 10: Tag Docker Image
```bash
# Step 2: Tag your image (replace with your ECR URI)
docker tag log-processor:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-processor:latest
```

### Step 11: Push to ECR
```bash
# Step 3: Push image
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/log-processor:latest
```

This takes 1-2 minutes. You'll see upload progress.

### Step 12: Verify Image in ECR
1. Go back to ECR Console
2. Click on `log-processor` repository
3. You should see your image with tag: **latest**
4. Note the **Image URI** (needed for Batch):
   - Example: `123456789012.dkr.ecr.us-east-1.amazonaws.com/log-processor:latest`

---

## PART 3: Create AWS Batch Environment

### Step 13: Navigate to AWS Batch Console
1. AWS Console → Search **"Batch"**
2. Click **"AWS Batch"**
3. If first time, you'll see **"Get started"** - click it

### Step 14: Create Compute Environment
1. In left sidebar, click **"Compute environments"**
2. Click **"Create"** button

### Step 15: Configure Compute Environment - Step 1
1. **Orchestration type:** Select **"Amazon Elastic Compute Cloud (Amazon EC2)"**
2. **Name:** Type: `log-processing-spot-env`
3. **Service role:** Select **"Create new role"**
   - This creates `AWSBatchServiceRole` automatically

### Step 16: Configure Instance Configuration
1. **Provisioning model:** Select **"Spot"**
   - Much cheaper than On-Demand (70-90% savings)
   - **Maximum % on-demand price:** Leave as **100%**
2. **Allowed instance types:** Select **"optimal"**
   - Batch will choose best instances
   - Or manually select: **c5.xlarge**, **c5.2xlarge**
3. **Minimum vCPUs:** Type: `0` (scale to zero when idle)
4. **Desired vCPUs:** Type: `0` (start with zero)
5. **Maximum vCPUs:** Type: `256` (maximum scale)

### Step 17: Configure Network
1. **VPC:** Select your default VPC
2. **Subnets:** Select all available subnets
3. **Security groups:** Use default security group
4. Click **"Create compute environment"**

Wait for status: **VALID** (takes ~30 seconds)

### Step 18: Create Job Queue
1. In left sidebar, click **"Job queues"**
2. Click **"Create"** button

### Step 19: Configure Job Queue
1. **Orchestration type:** **"Amazon Elastic Compute Cloud (Amazon EC2)"**
2. **Name:** Type: `log-processing-queue`
3. **Priority:** Type: `1` (higher = more priority)
4. **Connected compute environments:**
   - Click **"Select compute environments"**
   - Select `log-processing-spot-env`
   - Click **"Add"**
5. Click **"Create job queue"**

Wait for status: **VALID**

### Step 20: Create IAM Role for Batch Jobs
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** **"AWS service"**
3. **Use case:** Search for **"Elastic Container Service"**
4. Select **"Elastic Container Service Task"**
5. Click **"Next"**

### Step 21: Attach Permissions to Job Role
1. Search and attach:
   - ✅ **AmazonS3FullAccess**
   - ✅ **CloudWatchLogsFullAccess**
2. Click **"Next"**
3. **Role name:** `BatchJobS3AccessRole`
4. Click **"Create role"**

### Step 22: Create Job Definition
1. In Batch console, left sidebar → **"Job definitions"**
2. Click **"Create"** button

### Step 23: Configure Job Definition - Step 1
1. **Orchestration type:** **"Amazon Elastic Compute Cloud (Amazon EC2)"**
2. **Name:** Type: `log-processor-job`
3. **Execution timeout:** `300` seconds (5 minutes)

### Step 24: Configure Container Properties
1. **Image:** Paste ECR image URI from Step 12
   - Example: `123456789012.dkr.ecr.us-east-1.amazonaws.com/log-processor:latest`
2. **Command:** Leave empty (uses ENTRYPOINT from Dockerfile)
3. **Job role:** Select `BatchJobS3AccessRole`
4. **Execution role:** Select **"ecsTaskExecutionRole"** (auto-created) OR create new

### Step 25: Configure Resource Requirements
1. **vCPUs:** Type: `1`
2. **Memory (MiB):** Type: `2048` (2 GB)
3. **Number of GPUs:** Leave empty

### Step 26: Environment Variables (Optional)
We'll pass these when submitting jobs, but you can set defaults:
1. Click **"Add environment variable"**
2. **Key:** `S3_BUCKET` | **Value:** `your-logs-bucket`
3. Click **"Add environment variable"**
4. **Key:** `LOG_FILE_KEY` | **Value:** `raw/sample.gz`

Click **"Create job definition"**

---

## PART 4: Create Sample Log Data and Submit Jobs

### Step 27: Create S3 Bucket for Logs
1. **S3 Console** → **"Create bucket"**
2. **Name:** `batch-logs-processing-[your-account-id]`
3. Click **"Create bucket"**
4. Create folders: `raw/` and `processed/`

### Step 28: Generate Sample Log Data
Create a Python script locally: `generate_logs.py`

```python
import json
import gzip
import random
from datetime import datetime, timedelta

endpoints = ['/api/users', '/api/orders', '/api/products', '/api/search', '/api/checkout']
status_codes = [200, 200, 200, 200, 201, 400, 404, 500]

def generate_log_entry():
    return {
        'timestamp': (datetime.now() - timedelta(seconds=random.randint(0, 86400))).isoformat(),
        'endpoint': random.choice(endpoints),
        'duration_ms': random.randint(10, 2000),
        'status_code': random.choice(status_codes),
        'user_id': f"user_{random.randint(1, 1000)}",
        'request_id': f"req_{random.randint(100000, 999999)}"
    }

# Generate 10,000 log entries
with gzip.open('application-logs-001.gz', 'wt') as f:
    for _ in range(10000):
        f.write(json.dumps(generate_log_entry()) + '\n')

print("Generated application-logs-001.gz with 10,000 entries")

# Generate 5 more files
for i in range(2, 6):
    with gzip.open(f'application-logs-{i:03d}.gz', 'wt') as f:
        for _ in range(10000):
            f.write(json.dumps(generate_log_entry()) + '\n')
    print(f"Generated application-logs-{i:03d}.gz")

print("✓ Generated 5 log files (50,000 total entries)")
```

Run it:
```bash
python3 generate_logs.py
```

### Step 29: Upload Log Files to S3
```bash
# Upload all 5 files
aws s3 cp application-logs-001.gz s3://batch-logs-processing-[id]/raw/
aws s3 cp application-logs-002.gz s3://batch-logs-processing-[id]/raw/
aws s3 cp application-logs-003.gz s3://batch-logs-processing-[id]/raw/
aws s3 cp application-logs-004.gz s3://batch-logs-processing-[id]/raw/
aws s3 cp application-logs-005.gz s3://batch-logs-processing-[id]/raw/

# Verify
aws s3 ls s3://batch-logs-processing-[id]/raw/
```

### Step 30: Submit Batch Job via Console
1. **AWS Batch Console** → **"Jobs"** → **"Submit new job"**

### Step 31: Configure Job Submission
1. **Name:** Type: `process-log-001`
2. **Job definition:** Select `log-processor-job:1` (latest revision)
3. **Job queue:** Select `log-processing-queue`
4. **Job type:** Select **"Single"** (not array for now)

### Step 32: Configure Container Overrides
1. Scroll to **"Container overrides"**
2. **Environment variables:**
   - Click **"Add environment variable"**
   - **Key:** `S3_BUCKET` | **Value:** `batch-logs-processing-[your-id]`
   - **Key:** `LOG_FILE_KEY` | **Value:** `raw/application-logs-001.gz`
3. Click **"Next"**

### Step 33: Review and Submit
1. Review all settings
2. Click **"Create job"**

### Step 34: Monitor Job Execution
1. You'll be on the **"Jobs"** page
2. Your job status will change:
   - **SUBMITTED** → Queued
   - **PENDING** → Waiting for resources
   - **RUNNABLE** → Ready to run
   - **STARTING** → EC2 instance launching
   - **RUNNING** → Container executing
   - **SUCCEEDED** → Completed successfully

This takes 3-5 minutes (includes EC2 startup time)

### Step 35: View Job Logs
1. Click on your job name: `process-log-001`
2. Scroll to **"Logs"** section
3. Click **"View logs in CloudWatch"**
4. You should see:
   ```
   Processing: s3://batch-logs-processing-123/raw/application-logs-001.gz
   Processed 10000 log entries
   ✓ Results uploaded to: s3://batch-logs-processing-123/processed/application-logs-001.json
   Summary: Processed 5 endpoints
   ```

### Step 36: Verify Processed Output
1. **S3 Console** → Your bucket → `processed/` folder
2. You should see: `application-logs-001.json`
3. Download and view:
   ```json
   {
     "/api/users": {
       "total_requests": 2034,
       "avg_duration_ms": 987.34,
       "error_rate": 0.12,
       "success_count": 1789,
       "error_count": 245
     },
     ...
   }
   ```

---

## PART 5: Submit Array Job (Process All Files in Parallel)

### Step 37: Submit Array Job
1. **Batch Console** → **"Jobs"** → **"Submit new job"**
2. **Name:** `process-all-logs`
3. **Job definition:** `log-processor-job:1`
4. **Job queue:** `log-processing-queue`
5. **Job type:** Select **"Array"**
6. **Array size:** Type: `5` (one job per file)

### Step 38: Configure Array Job Environment Variables
We need to pass different file names to each array job:

**Limitation:** Console doesn't support dynamic env vars per array index.

**Solution:** Modify script to auto-detect files OR use CLI.

**Using AWS CLI (Recommended for Array Jobs):**

```bash
# Submit job via CLI
aws batch submit-job \
  --job-name process-all-logs \
  --job-queue log-processing-queue \
  --job-definition log-processor-job \
  --array-properties size=5 \
  --container-overrides '{
    "environment": [
      {"name": "S3_BUCKET", "value": "batch-logs-processing-123456789012"},
      {"name": "LOG_FILE_PATTERN", "value": "raw/application-logs-*.gz"}
    ]
  }'
```

### Step 39: Monitor Array Job
1. In Batch Console → **"Jobs"**
2. You'll see the parent job: `process-all-logs`
3. **Array job index:** Shows 0/5, 1/5, 2/5, etc.
4. All 5 jobs run in **parallel** (if capacity allows)
5. Watch as they progress to **SUCCEEDED**

### Step 40: Verify All Files Processed
1. Go to S3 → `processed/` folder
2. You should see 5 JSON files:
   - `application-logs-001.json`
   - `application-logs-002.json`
   - `application-logs-003.json`
   - `application-logs-004.json`
   - `application-logs-005.json`

**🎉 You just processed 50,000 log entries in parallel with AWS Batch!**

---

## ✅ Exercise 5.2 Completion Checklist

- [ ] Created Docker container with log processing script
- [ ] Built Docker image locally
- [ ] Created Amazon ECR repository
- [ ] Pushed Docker image to ECR
- [ ] Created AWS Batch compute environment (Spot instances)
- [ ] Created job queue
- [ ] Created IAM roles for Batch jobs
- [ ] Created job definition with container configuration
- [ ] Generated sample log data (gzipped JSON)
- [ ] Uploaded log files to S3
- [ ] Submitted single Batch job via console
- [ ] Monitored job execution in CloudWatch Logs
- [ ] Verified processed output in S3
- [ ] Submitted array job to process multiple files in parallel
- [ ] Confirmed all files processed successfully

**🎉 Congratulations!** You've completed Exercise 5.2!

---

# Exercise 5.3: Big Data Processing with Amazon EMR (Spark)

**Duration:** 3-4 hours | **Estimated Cost:** ~$20-35 for lab session

## 🎯 Learning Objectives
- Create EMR cluster with Spark
- Configure auto-scaling for cost optimization
- Submit PySpark jobs for large-scale data processing
- Monitor cluster metrics and application logs
- Process data with partitioning and optimization

---

## PART 1: Prepare Sample Data

### Step 1: Create S3 Bucket for EMR
1. **S3 Console** → **"Create bucket"**
2. **Name:** `emr-data-processing-[your-account-id]`
3. Click **"Create bucket"**
4. Create folders:
   - `input/`
   - `output/`
   - `scripts/`
   - `logs/`

### Step 2: Generate Sample Parquet Data
Create script locally: `generate_sample_data.py`

```python
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
from datetime import datetime, timedelta
import random

# Generate 1 million sample web events
num_records = 1_000_000

data = {
    'user_id': [f'user_{random.randint(1, 10000)}' for _ in range(num_records)],
    'session_id': [f'session_{random.randint(1, 100000)}' for _ in range(num_records)],
    'event_timestamp': [(datetime.now() - timedelta(days=random.randint(0, 30))).isoformat() for _ in range(num_records)],
    'page_url': [random.choice(['/home', '/products', '/cart', '/checkout', '/account']) for _ in range(num_records)],
    'event_type': [random.choice(['page_view', 'click', 'scroll', 'form_submit']) for _ in range(num_records)],
    'duration_seconds': [random.randint(1, 300) for _ in range(num_records)],
    'device_type': [random.choice(['mobile', 'desktop', 'tablet']) for _ in range(num_records)],
    'country': [random.choice(['US', 'UK', 'DE', 'FR', 'JP', 'IN']) for _ in range(num_records)]
}

df = pd.DataFrame(data)
print(f"Generated {len(df):,} records")

# Write partitioned Parquet (by date)
df['date'] = pd.to_datetime(df['event_timestamp']).dt.date
table = pa.Table.from_pandas(df)

pq.write_to_dataset(
    table,
    root_path='web_events_data',
    partition_cols=['date'],
    compression='snappy'
)

print("✓ Parquet files created in web_events_data/ (partitioned by date)")
```

Run it:
```bash
pip install pandas pyarrow
python3 generate_sample_data.py
```

### Step 3: Upload Data to S3
```bash
aws s3 sync web_events_data/ s3://emr-data-processing-[id]/input/web_events/

# Verify
aws s3 ls s3://emr-data-processing-[id]/input/web_events/ --recursive
```

---

## PART 2: Create EMR Cluster

### Step 4: Navigate to EMR Console
1. AWS Console → Search **"EMR"**
2. Click **"Amazon EMR"**

### Step 5: Create Cluster
1. Click **"Create cluster"** button

### Step 6: Configure Cluster - Software
1. **Name:** Type: `spark-analytics-cluster`
2. **Release:** Select latest **"emr-7.0.0"** or higher (includes Spark 3.5+)
3. **Applications:** Click **"Spark"** bundle
   - This includes: Spark, Hadoop, Hive, Livy, JupyterHub

### Step 7: Configure Cluster - Hardware
1. **Instance type - Master:** Select **"m5.xlarge"** (4 vCPU, 16 GB RAM)
2. **Instance count - Master:** `1`
3. **Instance type - Core:** Select **"r5.2xlarge"** (8 vCPU, 64 GB RAM)
   - r5 = memory-optimized (good for Spark)
4. **Instance count - Core:** `3` (for lab; use 10+ for production)

### Step 8: Configure Auto-Scaling (Optional)
1. Expand **"Cluster scaling"**
2. ✅ Check **"Use auto scaling"**
3. **Minimum capacity:** `3` core instances
4. **Maximum capacity:** `10` core instances
5. **Auto scaling rule:**
   - **Rule type:** **"Scale out"** (add instances)
   - **CloudWatch alarm:** **"YARNMemoryAvailablePercentage < 20"**
   - **Add instances:** `2`

### Step 9: Configure Task Instances (Spot)
1. Click **"Add task instance group"**
2. **Instance type:** **"r5.2xlarge"**
3. **Instance count:** `5`
4. **Market:** Select **"Spot"**
   - **Maximum Spot price:** **"Use on-demand price"** (100%)
5. **Auto scaling:** Configure similar to core nodes

### Step 10: Configure Networking
1. **VPC:** Select your default VPC
2. **Subnet:** Select any subnet
3. **EC2 security groups:** Use default managed security groups

### Step 11: Configure Cluster Logs
1. **Logging:** ✅ Enable
2. **S3 folder:** Browse and select: `s3://emr-data-processing-[id]/logs/`

### Step 12: Configure Security
1. **EC2 key pair:** Select existing key pair OR create new
   - You need this to SSH into cluster nodes
2. **IAM roles:**
   - **EMR role:** Select **"EMR_DefaultRole"** (auto-created) OR create new
   - **EC2 instance profile:** Select **"EMR_EC2_DefaultRole"**

### Step 13: Review and Create
1. Review all settings on the right panel
2. **Estimated cost:** Should show ~$2-5/hour
3. Click **"Create cluster"**

### Step 14: Wait for Cluster to Start
1. **Cluster state** will show:
   - **Starting** → ~5-10 minutes
   - **Bootstrapping** → Installing software
   - **Running** → Ready for jobs
   - **Waiting** → Ready and idle
2. **WAIT for "Waiting" status** before submitting jobs

---

## PART 3: Create PySpark Job Script

### Step 15: Write PySpark Script
Create file locally: `web_analytics.py`

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.window import Window
import sys

# Initialize Spark
spark = SparkSession.builder \
    .appName("Web Analytics Processing") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .getOrCreate()

# Get S3 paths from arguments
input_path = sys.argv[1] if len(sys.argv) > 1 else "s3://emr-data-processing-123/input/web_events/"
output_path = sys.argv[2] if len(sys.argv) > 2 else "s3://emr-data-processing-123/output/"

print(f"Reading data from: {input_path}")

# Read Parquet data
df = spark.read.parquet(input_path)

print(f"Loaded {df.count():,} events")
df.printSchema()

# Add derived columns
df = df.withColumn("event_hour", hour(col("event_timestamp"))) \
       .withColumn("is_conversion", when(col("page_url") == "/checkout", 1).otherwise(0))

# Analytics 1: User session aggregation
print("Calculating user sessions...")

user_sessions = df.groupBy("user_id", "session_id") \
    .agg(
        count("*").alias("event_count"),
        sum("duration_seconds").alias("total_duration"),
        countDistinct("page_url").alias("unique_pages"),
        max("is_conversion").alias("converted"),
        first("device_type").alias("device_type"),
        first("country").alias("country")
    )

# Write partitioned by country
user_sessions.write \
    .mode("overwrite") \
    .partitionBy("country") \
    .parquet(f"{output_path}/user_sessions/")

print(f"✓ User sessions written to {output_path}/user_sessions/")

# Analytics 2: Hourly page views
print("Calculating hourly page views...")

hourly_views = df.groupBy("date", "event_hour", "page_url") \
    .agg(
        count("*").alias("view_count"),
        countDistinct("user_id").alias("unique_users"),
        avg("duration_seconds").alias("avg_duration")
    ) \
    .orderBy("date", "event_hour", col("view_count").desc())

hourly_views.write \
    .mode("overwrite") \
    .partitionBy("date") \
    .parquet(f"{output_path}/hourly_page_views/")

print(f"✓ Hourly views written to {output_path}/hourly_page_views/")

# Analytics 3: Conversion funnel
print("Calculating conversion funnel...")

funnel_steps = ['/home', '/products', '/cart', '/checkout']

# Create funnel using window functions
window_spec = Window.partitionBy("user_id", "session_id").orderBy("event_timestamp")

df_with_step = df.filter(col("page_url").isin(funnel_steps)) \
    .withColumn("step_number", row_number().over(window_spec))

funnel_summary = df_with_step.groupBy("user_id", "session_id") \
    .agg(
        max("step_number").alias("max_step_reached"),
        collect_list("page_url").alias("funnel_path"),
        max("is_conversion").alias("completed_purchase")
    )

funnel_aggregates = funnel_summary.groupBy("max_step_reached") \
    .agg(
        count("*").alias("user_count"),
        sum("completed_purchase").alias("conversions"),
        (sum("completed_purchase") / count("*") * 100).alias("conversion_rate")
    ) \
    .orderBy("max_step_reached")

funnel_aggregates.write \
    .mode("overwrite") \
    .parquet(f"{output_path}/conversion_funnel/")

print(f"✓ Funnel analysis written to {output_path}/conversion_funnel/")

# Show sample results
print("\n=== SAMPLE RESULTS ===")
print("\nTop 10 users by session duration:")
user_sessions.orderBy(col("total_duration").desc()).show(10)

print("\nConversion funnel:")
funnel_aggregates.show()

print("\n✓ All analytics completed successfully!")

spark.stop()
```

### Step 16: Upload Script to S3
```bash
aws s3 cp web_analytics.py s3://emr-data-processing-[id]/scripts/
```

---

## PART 4: Submit Spark Job to EMR

### Step 17: Add Step to EMR Cluster
1. In **EMR Console**, click on your cluster: `spark-analytics-cluster`
2. Go to **"Steps"** tab
3. Click **"Add step"** button

### Step 18: Configure Spark Step
1. **Step type:** Select **"Spark application"**
2. **Name:** Type: `web-analytics-processing`
3. **Deploy mode:** Select **"Cluster"** (runs on cluster, not client)
4. **Application location:** 
   - Type: `s3://emr-data-processing-[id]/scripts/web_analytics.py`
5. **Spark-submit options:** (Optional) Type:
   ```
   --conf spark.executor.memory=8g --conf spark.executor.cores=4 --conf spark.dynamicAllocation.enabled=true
   ```
6. **Arguments:** Type (separated by spaces):
   ```
   s3://emr-data-processing-[id]/input/web_events/ s3://emr-data-processing-[id]/output/
   ```
7. **Action on failure:** Select **"Continue"** (for testing)

### Step 19: Submit Step
1. Click **"Add step"**
2. The step appears in the **Steps** list
3. **Status** will change:
   - **Pending** → Queued
   - **Running** → Executing
   - **Completed** → Success
   - **Failed** → Error occurred

This takes 3-5 minutes for 1M records.

### Step 20: Monitor Step Progress
1. Click on the step name: `web-analytics-processing`
2. You'll see:
   - **Start time / End time**
   - **Duration**
   - **Log files** (stdout, stderr, controller)

### Step 21: View Step Logs
1. Scroll to **"Log files"** section
2. Click on **"stdout"** link
3. You should see:
   ```
   Reading data from: s3://emr-data-processing-123/input/web_events/
   Loaded 1,000,000 events
   Calculating user sessions...
   ✓ User sessions written to s3://emr-data-processing-123/output/user_sessions/
   ...
   ✓ All analytics completed successfully!
   ```

### Step 22: View Spark UI
1. In cluster details, click on **"Application user interfaces"** tab
2. Click **"Spark History Server"** link
3. **Enable SSH tunnel** (if accessing from outside AWS):
   ```bash
   ssh -i your-key.pem -N -L 18080:[MASTER-PRIVATE-DNS]:18080 hadoop@[MASTER-PUBLIC-DNS]
   ```
4. Open browser: `http://localhost:18080`
5. You'll see:
   - **Application list**
   - Click on your app → **Stages**, **Executors**, **SQL** tabs
   - View DAG visualization, task timelines

---

## PART 5: Verify Output and Query Results

### Step 23: Check Output in S3
1. **S3 Console** → Your bucket → `output/` folder
2. You should see 3 folders:
   - `user_sessions/` (partitioned by country)
   - `hourly_page_views/` (partitioned by date)
   - `conversion_funnel/`

### Step 24: Explore Partitioned Data
1. Navigate to: `output/user_sessions/`
2. You'll see partition folders:
   - `country=US/`
   - `country=UK/`
   - `country=DE/`
   - etc.
3. Inside each: `.parquet` files

### Step 25: Query Results with Athena (Optional)
1. **Athena Console** → **"Query editor"**
2. Create external table:
   ```sql
   CREATE EXTERNAL TABLE user_sessions (
     user_id string,
     session_id string,
     event_count int,
     total_duration int,
     unique_pages int,
     converted int,
     device_type string
   )
   PARTITIONED BY (country string)
   STORED AS PARQUET
   LOCATION 's3://emr-data-processing-[id]/output/user_sessions/';
   
   -- Load partitions
   MSCK REPAIR TABLE user_sessions;
   
   -- Query
   SELECT country, 
          COUNT(*) as sessions,
          SUM(converted) as conversions,
          ROUND(SUM(converted) * 100.0 / COUNT(*), 2) as conversion_rate
   FROM user_sessions
   GROUP BY country
   ORDER BY conversions DESC;
   ```

---

## PART 6: Cleanup and Cost Optimization

### Step 26: Terminate EMR Cluster
1. In EMR console, select your cluster
2. Click **"Terminate"** button
3. Confirm by typing the cluster ID
4. Click **"Terminate"**

**IMPORTANT:** EMR charges per hour, even if idle. Always terminate when done!

### Step 27: Monitor Cluster Shutdown
1. **Status** changes:
   - **Terminating** → Shutting down
   - **Terminated** → All instances stopped
2. This takes ~5 minutes

---

## ✅ Exercise 5.3 Completion Checklist

- [ ] Generated 1M sample web events as Parquet files
- [ ] Uploaded partitioned data to S3
- [ ] Created EMR cluster with Spark
- [ ] Configured master, core, and task instances
- [ ] Enabled auto-scaling for cost optimization
- [ ] Used Spot instances for task nodes
- [ ] Wrote PySpark script with 3 analytics workflows
- [ ] Uploaded script to S3
- [ ] Submitted Spark application as EMR step
- [ ] Monitored step execution and logs
- [ ] Viewed Spark UI for performance analysis
- [ ] Verified partitioned output in S3
- [ ] (Optional) Queried results with Athena
- [ ] Terminated EMR cluster to stop charges

**🎉 Congratulations!** You've completed Exercise 5.3 and all of Module 5!

---

# 🎓 Module 5 Complete Summary

## What You've Accomplished

### Exercise 5.1: Serverless ETL with Lambda ✅
- Built event-driven ETL pipeline
- Validated CSV files with Lambda
- Transformed CSV to optimized Parquet format
- Used Lambda layers for large dependencies
- Monitored with CloudWatch dashboards

### Exercise 5.2: Batch Processing with AWS Batch ✅
- Created Docker container for log processing
- Pushed image to Amazon ECR
- Configured Spot-based compute environment
- Processed 50,000 log entries in parallel array jobs
- Achieved 70-90% cost savings with Spot instances

### Exercise 5.3: Big Data with EMR ✅
- Created EMR cluster with Spark
- Processed 1 million events with PySpark
- Implemented session analysis and conversion funnel
- Used partitioning for query optimization
- Monitored Spark application performance

## Key Services Mastered
1. **AWS Lambda** - Serverless event-driven processing
2. **AWS Batch** - Scalable batch processing with containers
3. **Amazon EMR** - Managed Hadoop/Spark for big data

## Next Steps
- **Module 6:** Containers (ECS, EKS, Fargate)
- Continue with remaining modules

---

## 🧹 Cleanup Resources

**Lambda:**
- Delete functions (no charge when not invoked)

**S3:**
- Empty and delete test buckets

**ECR:**
- Delete Docker images and repositories

**Batch:**
- Disable compute environments
- Delete job queues and definitions

**EMR:**
- **CRITICAL:** Terminate clusters (charges per hour!)

**Estimated Module 5 Lab Cost (3-4 hours): ~$40-70**

---

**🎉 Excellent work! Ready for Module 6 - Containers!**

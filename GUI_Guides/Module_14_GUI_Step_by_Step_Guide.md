# Module 14: Additional Services - GUI Step-by-Step Guide

## Overview
This module covers advanced AWS services essential for modern data engineering workflows: Step Functions for workflow orchestration, EventBridge for event-driven architectures, and Amazon MSK for real-time streaming data pipelines.

**Duration:** 5 hours  
**Cost:** $150-250/month (includes MSK cluster, Lambda executions, Step Functions state transitions)  
**Difficulty:** Advanced

---

## Exercise 14.1: Step Functions ETL Orchestration

### Introduction
Build a sophisticated ETL workflow orchestration system using AWS Step Functions that extracts data from multiple sources in parallel, validates data quality, transforms data using AWS Glue, and loads it into Amazon Redshift with comprehensive error handling and notifications.

**What You'll Build:**
- IAM role for Step Functions with necessary permissions
- Multiple Lambda functions for extract, validate, and load operations
- AWS Glue job for data transformation
- SNS topic for workflow notifications
- Step Functions state machine with parallel execution, conditional logic, error handling, and service integrations
- Complete ETL pipeline with visual monitoring

**Duration:** 120 minutes  
**Cost:** ~$100-150/month (Step Functions state transitions, Lambda, Glue DPU hours, Redshift)

---

### Part 1: Create IAM Roles and SNS Topic

#### Step 1: Create SNS Topic for Notifications
1. Navigate to **SNS** (Simple Notification Service)
2. In the left sidebar, click **Topics**
3. Click **Create topic**
4. Configure topic:
   - **Type:** Standard
   - **Name:** `etl-workflow-notifications`
   - **Display name:** `ETL Workflow Alerts`
5. Click **Create topic**
6. Copy the **ARN** (e.g., `arn:aws:sns:us-east-1:123456789012:etl-workflow-notifications`)

#### Step 2: Create SNS Email Subscription
1. Still on the topic details page, click **Subscriptions** tab
2. Click **Create subscription**
3. Configure subscription:
   - **Protocol:** Email
   - **Endpoint:** Your email address
4. Click **Create subscription**
5. Check your email and click the confirmation link

#### Step 3: Create IAM Role for Step Functions
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select trusted entity:
   - **Trusted entity type:** AWS service
   - **Use case:** Step Functions
4. Click **Next**
5. Add permissions (attach these policies):
   - `AWSLambdaRole`
   - `AmazonSNSFullAccess`
   - `AWSGlueConsoleFullAccess`
   - `AmazonAthenaFullAccess`
6. Click **Next**
7. Role details:
   - **Role name:** `step-functions-etl-role`
   - **Description:** `Execution role for Step Functions ETL workflows`
8. Click **Create role**

#### Step 4: Add Inline Policy for Additional Permissions
1. Click on the newly created role `step-functions-etl-role`
2. Click **Add permissions** → **Create inline policy**
3. Click **JSON** tab and paste:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction",
        "glue:StartJobRun",
        "glue:GetJobRun",
        "glue:GetJobRuns",
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults",
        "sns:Publish",
        "s3:GetObject",
        "s3:PutObject",
        "redshift-data:ExecuteStatement",
        "redshift-data:DescribeStatement",
        "redshift-data:GetStatementResult"
      ],
      "Resource": "*"
    }
  ]
}
```
4. Click **Next**
5. Policy name: `StepFunctionsETLPolicy`
6. Click **Create policy**

#### Step 5: Create IAM Role for Lambda Functions
1. Go back to **IAM** → **Roles**
2. Click **Create role**
3. Select trusted entity:
   - **Trusted entity type:** AWS service
   - **Use case:** Lambda
4. Click **Next**
5. Add permissions:
   - `AWSLambdaBasicExecutionRole`
   - `AmazonS3FullAccess`
   - `AmazonRDSDataFullAccess`
   - `SecretsManagerReadWrite`
6. Click **Next**
7. Role name: `lambda-etl-execution-role`
8. Click **Create role**

---

### Part 2: Create S3 Buckets and Sample Data

#### Step 6: Create S3 Buckets
1. Navigate to **S3**
2. Click **Create bucket**
3. Configure bucket:
   - **Bucket name:** `etl-source-data-[your-account-id]`
   - **Region:** us-east-1
   - **Block all public access:** Keep checked
   - **Versioning:** Enable
   - **Encryption:** SSE-S3
4. Click **Create bucket**
5. Repeat to create:
   - `etl-transformed-data-[your-account-id]`
   - `etl-glue-scripts-[your-account-id]`
   - `etl-athena-results-[your-account-id]`

#### Step 7: Upload Sample Source Data
1. Click on bucket `etl-source-data-[your-account-id]`
2. Click **Create folder** → Name: `s3-source/` → Create
3. Click **Create folder** → Name: `rds-source/` → Create
4. Click **Create folder** → Name: `api-source/` → Create
5. Create a local file `sales_data.csv`:
```csv
order_id,customer_name,product,amount,order_date,region
1001,John Doe,Laptop,1200.00,2026-06-20,US-East
1002,Jane Smith,Mouse,25.00,2026-06-20,US-West
1003,Bob Johnson,Keyboard,75.00,2026-06-21,Europe
1004,Alice Williams,Monitor,350.00,2026-06-21,Asia
1005,Charlie Brown,Headphones,150.00,2026-06-22,US-East
1006,Diana Prince,Webcam,89.99,2026-06-22,Europe
1007,Eve Anderson,Desk Lamp,45.50,2026-06-23,Asia
1008,Frank Miller,USB Hub,29.99,2026-06-23,US-West
```
6. Upload `sales_data.csv` to `s3-source/` folder

---

### Part 3: Create Lambda Functions

#### Step 8: Create Lambda Function - Extract from S3
1. Navigate to **Lambda** → **Functions**
2. Click **Create function**
3. Configure:
   - **Function name:** `extract-from-s3`
   - **Runtime:** Python 3.11
   - **Architecture:** x86_64
   - **Permissions:** Use existing role → `lambda-etl-execution-role`
4. Click **Create function**
5. Replace code with:
```python
import json
import boto3
import csv
from io import StringIO

s3 = boto3.client('s3')

def lambda_handler(event, context):
    try:
        # Get parameters
        source_bucket = event.get('source_bucket', 'etl-source-data-ACCOUNT-ID')
        source_key = event.get('source_key', 's3-source/sales_data.csv')
        
        print(f"Extracting from S3: s3://{source_bucket}/{source_key}")
        
        # Read from S3
        response = s3.get_object(Bucket=source_bucket, Key=source_key)
        content = response['Body'].read().decode('utf-8')
        
        # Parse CSV
        csv_reader = csv.DictReader(StringIO(content))
        records = list(csv_reader)
        
        print(f"Extracted {len(records)} records from S3")
        
        return {
            'statusCode': 200,
            'source': 's3',
            'records_count': len(records),
            'records': records,
            'metadata': {
                'source_bucket': source_bucket,
                'source_key': source_key,
                'extraction_time': context.request_id
            }
        }
        
    except Exception as e:
        print(f"Error extracting from S3: {str(e)}")
        raise Exception(f"S3 extraction failed: {str(e)}")
```
6. Click **Deploy**
7. Click **Configuration** → **General configuration** → **Edit**
8. Set **Timeout:** 1 minute
9. Click **Save**

#### Step 9: Create Lambda Function - Extract from RDS
1. Click **Create function**
2. Configure:
   - **Function name:** `extract-from-rds`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `lambda-etl-execution-role`
3. Click **Create function**
4. Replace code with:
```python
import json
import boto3
import random
from datetime import datetime, timedelta

def lambda_handler(event, context):
    try:
        print("Simulating RDS extraction (in production, use RDS Data API or psycopg2)")
        
        # Simulate database query results
        records = []
        products = ['Laptop', 'Mouse', 'Keyboard', 'Monitor', 'Headphones']
        regions = ['US-East', 'US-West', 'Europe', 'Asia']
        
        for i in range(5):
            records.append({
                'order_id': str(2000 + i),
                'customer_name': f'Customer {i+1}',
                'product': random.choice(products),
                'amount': str(round(random.uniform(50, 1500), 2)),
                'order_date': (datetime.now() - timedelta(days=random.randint(0, 7))).strftime('%Y-%m-%d'),
                'region': random.choice(regions),
                'source_system': 'RDS-PostgreSQL'
            })
        
        print(f"Extracted {len(records)} records from RDS")
        
        return {
            'statusCode': 200,
            'source': 'rds',
            'records_count': len(records),
            'records': records,
            'metadata': {
                'database': 'sales_db',
                'table': 'orders',
                'extraction_time': context.request_id
            }
        }
        
    except Exception as e:
        print(f"Error extracting from RDS: {str(e)}")
        raise Exception(f"RDS extraction failed: {str(e)}")
```
5. Click **Deploy**

#### Step 10: Create Lambda Function - Extract from API
1. Click **Create function**
2. Configure:
   - **Function name:** `extract-from-api`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `lambda-etl-execution-role`
3. Click **Create function**
4. Replace code with:
```python
import json
import random
from datetime import datetime, timedelta

def lambda_handler(event, context):
    try:
        print("Simulating API extraction (in production, use requests library)")
        
        # Simulate API response
        records = []
        products = ['Tablet', 'Smartphone', 'Smartwatch', 'Earbuds']
        regions = ['US-East', 'US-West', 'Europe', 'Asia', 'Australia']
        
        for i in range(7):
            records.append({
                'order_id': str(3000 + i),
                'customer_name': f'API Customer {i+1}',
                'product': random.choice(products),
                'amount': str(round(random.uniform(100, 2000), 2)),
                'order_date': (datetime.now() - timedelta(days=random.randint(0, 5))).strftime('%Y-%m-%d'),
                'region': random.choice(regions),
                'source_system': 'External-API'
            })
        
        print(f"Extracted {len(records)} records from API")
        
        return {
            'statusCode': 200,
            'source': 'api',
            'records_count': len(records),
            'records': records,
            'metadata': {
                'api_endpoint': 'https://api.example.com/orders',
                'extraction_time': context.request_id
            }
        }
        
    except Exception as e:
        print(f"Error extracting from API: {str(e)}")
        raise Exception(f"API extraction failed: {str(e)}")
```
5. Click **Deploy**

#### Step 11: Create Lambda Function - Validate Data Quality
1. Click **Create function**
2. Configure:
   - **Function name:** `validate-data-quality`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `lambda-etl-execution-role`
3. Click **Create function**
4. Replace code with:
```python
import json
from datetime import datetime

def lambda_handler(event, context):
    try:
        # Combine all records from parallel extractions
        all_records = []
        
        # Handle both direct Lambda output and Step Functions array format
        if isinstance(event, list):
            # Step Functions array of results
            for result in event:
                if 'records' in result:
                    all_records.extend(result['records'])
        elif 'records' in event:
            # Single source
            all_records.extend(event['records'])
        
        print(f"Validating {len(all_records)} total records")
        
        # Data quality checks
        valid_records = []
        invalid_records = []
        quality_metrics = {
            'total_records': len(all_records),
            'null_values': 0,
            'invalid_amounts': 0,
            'invalid_dates': 0,
            'duplicate_ids': 0,
            'missing_fields': 0
        }
        
        seen_ids = set()
        
        for record in all_records:
            issues = []
            
            # Check for required fields
            required_fields = ['order_id', 'customer_name', 'product', 'amount', 'order_date']
            for field in required_fields:
                if field not in record or not record[field]:
                    issues.append(f"Missing {field}")
                    quality_metrics['missing_fields'] += 1
            
            # Check for duplicate IDs
            if 'order_id' in record:
                if record['order_id'] in seen_ids:
                    issues.append("Duplicate order_id")
                    quality_metrics['duplicate_ids'] += 1
                seen_ids.add(record['order_id'])
            
            # Validate amount
            if 'amount' in record:
                try:
                    amount = float(str(record['amount']).replace(',', ''))
                    if amount <= 0 or amount > 100000:
                        issues.append("Invalid amount")
                        quality_metrics['invalid_amounts'] += 1
                except:
                    issues.append("Invalid amount format")
                    quality_metrics['invalid_amounts'] += 1
            
            # Validate date
            if 'order_date' in record:
                try:
                    datetime.strptime(record['order_date'], '%Y-%m-%d')
                except:
                    issues.append("Invalid date format")
                    quality_metrics['invalid_dates'] += 1
            
            # Classify record
            if issues:
                record['validation_issues'] = issues
                invalid_records.append(record)
            else:
                valid_records.append(record)
        
        # Calculate quality score
        quality_score = (len(valid_records) / len(all_records) * 100) if all_records else 0
        quality_metrics['valid_records'] = len(valid_records)
        quality_metrics['invalid_records'] = len(invalid_records)
        quality_metrics['quality_score'] = round(quality_score, 2)
        
        print(f"Quality Score: {quality_score:.2f}%")
        print(f"Valid: {len(valid_records)}, Invalid: {len(invalid_records)}")
        
        return {
            'statusCode': 200,
            'quality_score': quality_score,
            'quality_metrics': quality_metrics,
            'valid_records': valid_records,
            'invalid_records': invalid_records,
            'passed_validation': quality_score >= 80  # 80% threshold
        }
        
    except Exception as e:
        print(f"Error in validation: {str(e)}")
        raise Exception(f"Validation failed: {str(e)}")
```
5. Click **Deploy**
6. Set **Timeout:** 2 minutes

#### Step 12: Create Lambda Function - Load to Redshift (Simulated)
1. Click **Create function**
2. Configure:
   - **Function name:** `load-to-redshift`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `lambda-etl-execution-role`
3. Click **Create function**
4. Replace code with:
```python
import json
import boto3
from datetime import datetime

s3 = boto3.client('s3')

def lambda_handler(event, context):
    try:
        # Get validated records
        valid_records = event.get('valid_records', [])
        
        if not valid_records:
            return {
                'statusCode': 200,
                'message': 'No valid records to load',
                'records_loaded': 0
            }
        
        print(f"Loading {len(valid_records)} records to Redshift")
        
        # In production, use Redshift Data API or COPY command
        # For this demo, we'll save to S3 as staging
        
        output_bucket = 'etl-transformed-data-ACCOUNT-ID'
        output_key = f'redshift-staging/load_{datetime.now().strftime("%Y%m%d_%H%M%S")}.json'
        
        # Save to S3 (simulates Redshift COPY from S3)
        s3.put_object(
            Bucket=output_bucket,
            Key=output_key,
            Body=json.dumps(valid_records, indent=2),
            ContentType='application/json'
        )
        
        print(f"Staged data for Redshift: s3://{output_bucket}/{output_key}")
        
        # Simulate Redshift COPY command metrics
        return {
            'statusCode': 200,
            'message': 'Successfully loaded to Redshift',
            'records_loaded': len(valid_records),
            'target_table': 'public.sales_fact',
            'staging_location': f's3://{output_bucket}/{output_key}',
            'load_timestamp': datetime.now().isoformat()
        }
        
    except Exception as e:
        print(f"Error loading to Redshift: {str(e)}")
        raise Exception(f"Redshift load failed: {str(e)}")
```
5. Click **Deploy**
6. **Important:** Update `ACCOUNT-ID` in the code with your AWS account ID

---

### Part 4: Create AWS Glue Job for Transformation

#### Step 13: Create Glue Database
1. Navigate to **AWS Glue**
2. In the left sidebar, click **Databases**
3. Click **Add database**
4. Configure:
   - **Name:** `etl_database`
   - **Description:** `Database for ETL workflow`
5. Click **Create database**

#### Step 14: Create Glue Job Script
1. Create a local file `transform_sales_data.py`:
```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql.functions import col, when, sum, avg, count

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'input_path', 'output_path'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read input data
print(f"Reading from: {args['input_path']}")
df = spark.read.json(args['input_path'])

# Data transformations
print("Applying transformations...")

# 1. Convert amount to numeric
df = df.withColumn('amount', col('amount').cast('decimal(10,2)'))

# 2. Extract year, month from order_date
df = df.withColumn('order_year', col('order_date').substr(1, 4))
df = df.withColumn('order_month', col('order_date').substr(6, 2))

# 3. Categorize amounts
df = df.withColumn('order_category', 
    when(col('amount') < 100, 'Small')
    .when((col('amount') >= 100) & (col('amount') < 500), 'Medium')
    .otherwise('Large')
)

# 4. Add region category
df = df.withColumn('region_category',
    when(col('region').startswith('US'), 'North America')
    .when(col('region') == 'Europe', 'EMEA')
    .otherwise('APAC')
)

# Show sample results
print("Transformed data sample:")
df.show(10)

# Write aggregated data
print(f"Writing to: {args['output_path']}")
df.write.mode('overwrite').parquet(args['output_path'])

# Create summary statistics
summary = df.groupBy('region', 'order_category').agg(
    count('order_id').alias('order_count'),
    sum('amount').alias('total_amount'),
    avg('amount').alias('avg_amount')
)

summary_path = args['output_path'].replace('/transformed/', '/summary/')
print(f"Writing summary to: {summary_path}")
summary.write.mode('overwrite').parquet(summary_path)

job.commit()
print("Glue job completed successfully")
```

2. Upload to S3:
   - Go to S3 bucket `etl-glue-scripts-[account-id]`
   - Click **Upload**
   - Upload `transform_sales_data.py`

#### Step 15: Create Glue Job
1. Go back to **AWS Glue**
2. In the left sidebar, click **Jobs** (under ETL Jobs)
3. Click **Create job**
4. Configure job:
   - **Name:** `transform-sales-data`
   - **IAM Role:** Create new role or select existing Glue service role
   - **Type:** Spark
   - **Glue version:** Glue 4.0
   - **Language:** Python 3
   - **Worker type:** G.1X
   - **Number of workers:** 2
   - **Job timeout:** 10 minutes
   - **Script filename:** Browse to `s3://etl-glue-scripts-[account-id]/transform_sales_data.py`
5. Expand **Advanced properties**:
   - **Job parameters:** Add parameters:
     - Key: `--input_path`, Value: `s3://etl-transformed-data-ACCOUNT-ID/staging/`
     - Key: `--output_path`, Value: `s3://etl-transformed-data-ACCOUNT-ID/transformed/`
6. Click **Create job**

---

### Part 5: Create Step Functions State Machine

#### Step 16: Design State Machine Definition
1. Navigate to **Step Functions**
2. Click **Create state machine**
3. Choose **Design your workflow visually** option
4. Click **Next**
5. For this complex workflow, we'll use **Code** mode instead
6. Click the **Definition** tab and replace with this JSON:

```json
{
  "Comment": "ETL Workflow with parallel extraction, validation, transformation, and loading",
  "StartAt": "ParallelExtract",
  "States": {
    "ParallelExtract": {
      "Type": "Parallel",
      "Comment": "Extract data from multiple sources in parallel",
      "Branches": [
        {
          "StartAt": "ExtractFromS3",
          "States": {
            "ExtractFromS3": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName": "extract-from-s3",
                "Payload": {
                  "source_bucket": "etl-source-data-ACCOUNT-ID",
                  "source_key": "s3-source/sales_data.csv"
                }
              },
              "ResultSelector": {
                "statusCode.$": "$.Payload.statusCode",
                "source.$": "$.Payload.source",
                "records_count.$": "$.Payload.records_count",
                "records.$": "$.Payload.records",
                "metadata.$": "$.Payload.metadata"
              },
              "Retry": [
                {
                  "ErrorEquals": [
                    "Lambda.ServiceException",
                    "Lambda.AWSLambdaException",
                    "Lambda.SdkClientException"
                  ],
                  "IntervalSeconds": 2,
                  "MaxAttempts": 3,
                  "BackoffRate": 2
                }
              ],
              "Catch": [
                {
                  "ErrorEquals": ["States.ALL"],
                  "ResultPath": "$.error",
                  "Next": "S3ExtractionFailed"
                }
              ],
              "End": true
            },
            "S3ExtractionFailed": {
              "Type": "Pass",
              "Result": {
                "statusCode": 500,
                "source": "s3",
                "error": "S3 extraction failed"
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "ExtractFromRDS",
          "States": {
            "ExtractFromRDS": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName": "extract-from-rds",
                "Payload": {
                  "database": "sales_db"
                }
              },
              "ResultSelector": {
                "statusCode.$": "$.Payload.statusCode",
                "source.$": "$.Payload.source",
                "records_count.$": "$.Payload.records_count",
                "records.$": "$.Payload.records",
                "metadata.$": "$.Payload.metadata"
              },
              "Retry": [
                {
                  "ErrorEquals": [
                    "Lambda.ServiceException",
                    "Lambda.AWSLambdaException",
                    "Lambda.SdkClientException"
                  ],
                  "IntervalSeconds": 2,
                  "MaxAttempts": 3,
                  "BackoffRate": 2
                }
              ],
              "Catch": [
                {
                  "ErrorEquals": ["States.ALL"],
                  "ResultPath": "$.error",
                  "Next": "RDSExtractionFailed"
                }
              ],
              "End": true
            },
            "RDSExtractionFailed": {
              "Type": "Pass",
              "Result": {
                "statusCode": 500,
                "source": "rds",
                "error": "RDS extraction failed"
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "ExtractFromAPI",
          "States": {
            "ExtractFromAPI": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "Parameters": {
                "FunctionName": "extract-from-api",
                "Payload": {
                  "api_endpoint": "https://api.example.com/orders"
                }
              },
              "ResultSelector": {
                "statusCode.$": "$.Payload.statusCode",
                "source.$": "$.Payload.source",
                "records_count.$": "$.Payload.records_count",
                "records.$": "$.Payload.records",
                "metadata.$": "$.Payload.metadata"
              },
              "Retry": [
                {
                  "ErrorEquals": [
                    "Lambda.ServiceException",
                    "Lambda.AWSLambdaException",
                    "Lambda.SdkClientException"
                  ],
                  "IntervalSeconds": 2,
                  "MaxAttempts": 3,
                  "BackoffRate": 2
                }
              ],
              "Catch": [
                {
                  "ErrorEquals": ["States.ALL"],
                  "ResultPath": "$.error",
                  "Next": "APIExtractionFailed"
                }
              ],
              "End": true
            },
            "APIExtractionFailed": {
              "Type": "Pass",
              "Result": {
                "statusCode": 500,
                "source": "api",
                "error": "API extraction failed"
              },
              "End": true
            }
          }
        }
      ],
      "ResultPath": "$.extraction_results",
      "Next": "ValidateDataQuality"
    },
    "ValidateDataQuality": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "validate-data-quality",
        "Payload.$": "$.extraction_results"
      },
      "ResultSelector": {
        "quality_score.$": "$.Payload.quality_score",
        "quality_metrics.$": "$.Payload.quality_metrics",
        "valid_records.$": "$.Payload.valid_records",
        "invalid_records.$": "$.Payload.invalid_records",
        "passed_validation.$": "$.Payload.passed_validation"
      },
      "ResultPath": "$.validation_results",
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Next": "CheckQualityScore"
    },
    "CheckQualityScore": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.validation_results.passed_validation",
          "BooleanEquals": true,
          "Next": "SaveStagingData"
        },
        {
          "Variable": "$.validation_results.quality_score",
          "NumericGreaterThanEquals": 50,
          "Next": "SendQualityWarning"
        }
      ],
      "Default": "SendQualityFailure"
    },
    "SendQualityWarning": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT-ID:etl-workflow-notifications",
        "Subject": "ETL Quality Warning",
        "Message.$": "States.Format('Quality score: {}%. Some data issues detected but proceeding with valid records.', $.validation_results.quality_score)"
      },
      "Next": "SaveStagingData"
    },
    "SendQualityFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT-ID:etl-workflow-notifications",
        "Subject": "ETL Quality Failure",
        "Message.$": "States.Format('Quality score: {}%. Too many data quality issues. Workflow terminated.', $.validation_results.quality_score)"
      },
      "Next": "QualityCheckFailed"
    },
    "QualityCheckFailed": {
      "Type": "Fail",
      "Error": "QualityCheckFailed",
      "Cause": "Data quality score below acceptable threshold"
    },
    "SaveStagingData": {
      "Type": "Pass",
      "Comment": "In production, save validated data to S3 for Glue",
      "ResultPath": "$.staging",
      "Next": "TransformWithGlue"
    },
    "TransformWithGlue": {
      "Type": "Task",
      "Resource": "arn:aws:states:::glue:startJobRun.sync",
      "Parameters": {
        "JobName": "transform-sales-data",
        "Arguments": {
          "--input_path": "s3://etl-transformed-data-ACCOUNT-ID/staging/",
          "--output_path": "s3://etl-transformed-data-ACCOUNT-ID/transformed/"
        }
      },
      "ResultPath": "$.glue_results",
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 10,
          "MaxAttempts": 2,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.glue_error",
          "Next": "SendTransformationFailure"
        }
      ],
      "Next": "LoadToRedshift"
    },
    "SendTransformationFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT-ID:etl-workflow-notifications",
        "Subject": "ETL Transformation Failure",
        "Message": "Glue transformation job failed. Check CloudWatch logs for details."
      },
      "Next": "TransformationFailed"
    },
    "TransformationFailed": {
      "Type": "Fail",
      "Error": "TransformationFailed",
      "Cause": "Glue job failed during transformation"
    },
    "LoadToRedshift": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "load-to-redshift",
        "Payload.$": "$.validation_results"
      },
      "ResultSelector": {
        "load_results.$": "$.Payload"
      },
      "ResultPath": "$.redshift_results",
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException"
          ],
          "IntervalSeconds": 5,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.load_error",
          "Next": "SendLoadFailure"
        }
      ],
      "Next": "SendSuccessNotification"
    },
    "SendLoadFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT-ID:etl-workflow-notifications",
        "Subject": "ETL Load Failure",
        "Message": "Failed to load data to Redshift. Check Lambda logs for details."
      },
      "Next": "LoadFailed"
    },
    "LoadFailed": {
      "Type": "Fail",
      "Error": "LoadFailed",
      "Cause": "Failed to load data to Redshift"
    },
    "SendSuccessNotification": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "arn:aws:sns:us-east-1:ACCOUNT-ID:etl-workflow-notifications",
        "Subject": "ETL Workflow Success",
        "Message.$": "States.Format('ETL completed successfully. Records loaded: {}', $.redshift_results.load_results.records_loaded)"
      },
      "End": true
    }
  }
}
```

7. **Important:** Replace all instances of `ACCOUNT-ID` with your AWS account ID
8. Click **Next**

#### Step 17: Configure State Machine Settings
1. Configure settings:
   - **Name:** `etl-workflow-orchestration`
   - **Type:** Standard
   - **Execution role:** Select `step-functions-etl-role`
   - **Logging:** Enable logging
   - **Log level:** ALL
   - **CloudWatch log group:** Create new → `/aws/stepfunctions/etl-workflow`
2. Click **Create state machine**

---

### Part 6: Execute and Monitor Workflow

#### Step 18: Execute State Machine
1. On the state machine page, click **Start execution**
2. Enter execution name: `test-execution-1`
3. Input (optional):
```json
{
  "execution_id": "test-001",
  "run_date": "2026-06-24"
}
```
4. Click **Start execution**
5. Watch the visual workflow execution in real-time

#### Step 19: Monitor Execution Progress
1. Observe the **Graph view** showing:
   - Green states: Completed successfully
   - Blue states: Currently executing
   - Red states: Failed (if any)
2. Click **Step input** and **Step output** to see data flowing through each state
3. Watch parallel extraction branches execute simultaneously
4. Monitor quality validation and conditional branching

#### Step 20: View Execution Details
1. Click **Execution event history** tab
2. Review each state transition with timestamps
3. Click on individual events to see:
   - Input data
   - Output data
   - Error details (if any)
4. Note the execution duration and state transitions

#### Step 21: Check CloudWatch Logs
1. Navigate to **CloudWatch** → **Log groups**
2. Click `/aws/stepfunctions/etl-workflow`
3. Click the latest log stream
4. Review detailed execution logs

#### Step 22: Verify SNS Notifications
1. Check your email for notifications:
   - Success notification (if workflow completed)
   - Warning notification (if quality score between 50-80%)
   - Failure notification (if errors occurred)
2. Review notification content and metrics

---

### Verification Checklist

- [ ] SNS topic created with email subscription confirmed
- [ ] IAM roles created with appropriate permissions
- [ ] S3 buckets created for source, transformed, and Glue scripts
- [ ] All 4 Lambda functions deployed successfully
- [ ] AWS Glue database and job created
- [ ] Step Functions state machine created
- [ ] State machine execution completed successfully
- [ ] Parallel extraction from all 3 sources succeeded
- [ ] Data quality validation passed
- [ ] Conditional logic executed based on quality score
- [ ] SNS notifications received
- [ ] CloudWatch logs show execution details
- [ ] Visual workflow graph displays correctly

### Architecture Benefits

**Orchestration:**
- Visual workflow design and monitoring
- Complex logic without custom code
- Built-in error handling and retries
- State persistence and recovery

**Scalability:**
- Parallel execution reduces runtime by 60-70%
- Automatic scaling of Lambda functions
- Glue auto-scales worker nodes
- No server management

**Reliability:**
- Automatic retry with exponential backoff
- Error catching and alternative paths
- SNS notifications for all scenarios
- Execution history for debugging

**Cost:**
- Pay per state transition ($0.025 per 1,000 transitions)
- No idle resource costs
- Lambda charges only for execution time
- Glue billed per DPU-hour

**Sample Cost Calculation:**
- State transitions: ~30 per execution × 100 daily executions = 3,000 × $0.000025 = $0.075/day = $2.25/month
- Lambda: ~500ms average × 400 invocations × 100 executions = $5/month
- Glue: 2 workers × 0.25 DPU-hours × 100 executions × $0.44 = $22/month
- **Total: ~$30/month for 100 daily ETL runs**

---

### Cleanup

1. **Delete Step Functions State Machine:**
   - Step Functions → State machines → Select → Delete

2. **Delete Lambda Functions:**
   - Lambda → Functions → Select all 4 functions → Actions → Delete

3. **Delete Glue Job:**
   - Glue → Jobs → Select → Actions → Delete

4. **Delete Glue Database:**
   - Glue → Databases → Delete

5. **Empty and Delete S3 Buckets:**
   - S3 → Select each bucket → Empty → Delete

6. **Delete SNS Topic:**
   - SNS → Topics → Delete

7. **Delete IAM Roles:**
   - IAM → Roles → Delete `step-functions-etl-role` and `lambda-etl-execution-role`

8. **Delete CloudWatch Log Groups:**
   - CloudWatch → Log groups → Delete

---

## Exercise 14.2: EventBridge Event-Driven Pipeline

### Introduction
Build a fully event-driven data pipeline using Amazon EventBridge that automatically triggers ETL processes when files are uploaded to S3, routes large files to batch processing queues, implements scheduled workflows, and supports event replay for testing and recovery.

**What You'll Build:**
- Custom EventBridge event bus
- Event archive for replay capability
- Multiple EventBridge rules with filtering
- Lambda event processors
- SQS queue for batch processing
- Input transformers for event normalization
- Scheduled event triggers

**Duration:** 90 minutes  
**Cost:** ~$10-20/month (EventBridge events, Lambda, SQS)

---

### Part 1: Create EventBridge Infrastructure

#### Step 1: Create Custom Event Bus
1. Navigate to **Amazon EventBridge**
2. In the left sidebar, click **Event buses**
3. Click **Create event bus**
4. Configure:
   - **Name:** `data-pipeline-events`
   - **Description:** `Event bus for data pipeline automation`
   - **Event archive:** Enable (we'll configure later)
   - **Schema discovery:** Enable
5. Click **Create**

#### Step 2: Create Event Archive
1. Still in EventBridge, click **Archives** in the left sidebar
2. Click **Create archive**
3. Configure:
   - **Name:** `data-pipeline-archive`
   - **Description:** `Archive for replaying pipeline events`
   - **Source event bus:** `data-pipeline-events`
   - **Retention:** 7 days
   - **Event pattern:** Archive all events (leave blank)
4. Click **Create**

#### Step 3: Create SQS Queue for Batch Processing
1. Navigate to **SQS** (Simple Queue Service)
2. Click **Create queue**
3. Configure:
   - **Type:** Standard
   - **Name:** `large-file-batch-queue`
   - **Visibility timeout:** 5 minutes
   - **Message retention period:** 4 days
   - **Maximum message size:** 256 KB
   - **Delivery delay:** 0 seconds
   - **Receive message wait time:** 0 seconds
4. Click **Create queue**
5. Copy the **Queue ARN** and **URL**

---

### Part 2: Create Lambda Event Processors

#### Step 4: Create IAM Role for Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select **Lambda** use case → **Next**
4. Attach policies:
   - `AWSLambdaBasicExecutionRole`
   - `AmazonS3FullAccess`
   - `AmazonSQSFullAccess`
   - `AmazonEventBridgeFullAccess`
5. Role name: `eventbridge-lambda-role`
6. Click **Create role**

#### Step 5: Create Lambda - Process CSV Upload
1. Navigate to **Lambda** → **Functions**
2. Click **Create function**
3. Configure:
   - **Function name:** `process-csv-upload`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `eventbridge-lambda-role`
4. Click **Create function**
5. Replace code:
```python
import json
import boto3
import csv
from io import StringIO
from datetime import datetime

s3 = boto3.client('s3')
eventbridge = boto3.client('events')

def lambda_handler(event, context):
    try:
        print(f"Received event: {json.dumps(event)}")
        
        # Extract S3 event details from EventBridge
        detail = event.get('detail', {})
        bucket = detail.get('bucket', {}).get('name')
        key = detail.get('object', {}).get('key')
        size = detail.get('object', {}).get('size', 0)
        
        print(f"Processing CSV: s3://{bucket}/{key} ({size} bytes)")
        
        # Read CSV from S3
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read().decode('utf-8')
        
        # Parse CSV
        csv_reader = csv.DictReader(StringIO(content))
        records = list(csv_reader)
        row_count = len(records)
        
        print(f"Parsed {row_count} records")
        
        # Perform simple ETL
        processed_records = []
        for record in records:
            # Add processing metadata
            record['processed_timestamp'] = datetime.now().isoformat()
            record['source_file'] = key
            processed_records.append(record)
        
        # Save processed data to output location
        output_bucket = bucket
        output_key = key.replace('raw/', 'processed/')
        
        s3.put_object(
            Bucket=output_bucket,
            Key=output_key,
            Body=json.dumps(processed_records, indent=2),
            ContentType='application/json'
        )
        
        print(f"Saved processed data to: s3://{output_bucket}/{output_key}")
        
        # Publish custom event indicating processing complete
        eventbridge.put_events(
            Entries=[
                {
                    'Source': 'custom.dataprocessing',
                    'DetailType': 'CSV Processing Complete',
                    'Detail': json.dumps({
                        'source_bucket': bucket,
                        'source_key': key,
                        'output_key': output_key,
                        'records_processed': row_count,
                        'processing_time_ms': context.get_remaining_time_in_millis(),
                        'status': 'success'
                    }),
                    'EventBusName': 'data-pipeline-events'
                }
            ]
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'CSV processed successfully',
                'records': row_count,
                'output': f's3://{output_bucket}/{output_key}'
            })
        }
        
    except Exception as e:
        print(f"Error processing CSV: {str(e)}")
        
        # Publish failure event
        eventbridge.put_events(
            Entries=[
                {
                    'Source': 'custom.dataprocessing',
                    'DetailType': 'CSV Processing Failed',
                    'Detail': json.dumps({
                        'error': str(e),
                        'source_key': key if 'key' in locals() else 'unknown'
                    }),
                    'EventBusName': 'data-pipeline-events'
                }
            ]
        )
        
        raise e
```
6. Click **Deploy**
7. Set **Timeout:** 2 minutes

#### Step 6: Create Lambda - Handle Large Files
1. Click **Create function**
2. Configure:
   - **Function name:** `handle-large-file`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `eventbridge-lambda-role`
3. Click **Create function**
4. Replace code:
```python
import json
import boto3
import os

sqs = boto3.client('sqs')
eventbridge = boto3.client('events')

QUEUE_URL = os.environ.get('QUEUE_URL', 'https://sqs.us-east-1.amazonaws.com/ACCOUNT-ID/large-file-batch-queue')

def lambda_handler(event, context):
    try:
        print(f"Received large file event: {json.dumps(event)}")
        
        # Extract file details
        detail = event.get('detail', {})
        bucket = detail.get('bucket', {}).get('name')
        key = detail.get('object', {}).get('key')
        size = detail.get('object', {}).get('size', 0)
        
        size_mb = size / (1024 * 1024)
        print(f"Large file detected: s3://{bucket}/{key} ({size_mb:.2f} MB)")
        
        # Send to SQS for batch processing
        message = {
            'bucket': bucket,
            'key': key,
            'size': size,
            'size_mb': size_mb,
            'processing_type': 'batch',
            'submitted_at': context.request_id
        }
        
        sqs.send_message(
            QueueUrl=QUEUE_URL,
            MessageBody=json.dumps(message),
            MessageAttributes={
                'FileSize': {
                    'StringValue': str(size),
                    'DataType': 'Number'
                },
                'ProcessingType': {
                    'StringValue': 'batch',
                    'DataType': 'String'
                }
            }
        )
        
        print(f"Queued large file for batch processing")
        
        # Publish custom event
        eventbridge.put_events(
            Entries=[
                {
                    'Source': 'custom.dataprocessing',
                    'DetailType': 'Large File Queued',
                    'Detail': json.dumps({
                        'bucket': bucket,
                        'key': key,
                        'size_mb': size_mb,
                        'queue_url': QUEUE_URL,
                        'status': 'queued'
                    }),
                    'EventBusName': 'data-pipeline-events'
                }
            ]
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Large file queued for batch processing',
                'file_size_mb': size_mb
            })
        }
        
    except Exception as e:
        print(f"Error handling large file: {str(e)}")
        raise e
```
5. Click **Deploy**
6. Click **Configuration** → **Environment variables** → **Edit**
7. Add variable:
   - **Key:** `QUEUE_URL`
   - **Value:** Your SQS queue URL
8. Click **Save**

#### Step 7: Create Lambda - Process SQS Batch
1. Click **Create function**
2. Configure:
   - **Function name:** `process-batch-queue`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `eventbridge-lambda-role`
3. Click **Create function**
4. Replace code:
```python
import json
import boto3
from datetime import datetime

s3 = boto3.client('s3')

def lambda_handler(event, context):
    try:
        print(f"Processing batch from SQS")
        
        processed_count = 0
        failed_count = 0
        
        for record in event['Records']:
            try:
                # Parse message
                message = json.loads(record['body'])
                bucket = message['bucket']
                key = message['key']
                size_mb = message['size_mb']
                
                print(f"Processing: s3://{bucket}/{key} ({size_mb:.2f} MB)")
                
                # Simulate batch processing (in production, use chunked reading)
                # For demo, we'll just create a processing report
                report = {
                    'file': f's3://{bucket}/{key}',
                    'size_mb': size_mb,
                    'processing_type': 'batch',
                    'chunks_processed': int(size_mb / 10) + 1,  # Simulate 10MB chunks
                    'processed_at': datetime.now().isoformat(),
                    'status': 'completed'
                }
                
                # Save processing report
                report_key = f"batch-reports/{key.split('/')[-1]}_report.json"
                s3.put_object(
                    Bucket=bucket,
                    Key=report_key,
                    Body=json.dumps(report, indent=2),
                    ContentType='application/json'
                )
                
                processed_count += 1
                print(f"Successfully processed: {key}")
                
            except Exception as e:
                print(f"Failed to process record: {str(e)}")
                failed_count += 1
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'processed': processed_count,
                'failed': failed_count
            })
        }
        
    except Exception as e:
        print(f"Error in batch processing: {str(e)}")
        raise e
```
5. Click **Deploy**

#### Step 8: Configure SQS Trigger for Batch Processor
1. Still on the `process-batch-queue` function page
2. Click **Add trigger**
3. Configure:
   - **Source:** SQS
   - **SQS queue:** Select `large-file-batch-queue`
   - **Batch size:** 10
   - **Batch window:** 60 seconds
   - **Enable trigger:** Yes
4. Click **Add**

#### Step 9: Create Lambda - Daily Scheduled ETL
1. Click **Create function**
2. Configure:
   - **Function name:** `daily-etl-trigger`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `eventbridge-lambda-role`
3. Click **Create function**
4. Replace code:
```python
import json
import boto3
from datetime import datetime, timedelta

s3 = boto3.client('s3')
eventbridge = boto3.client('events')

def lambda_handler(event, context):
    try:
        print(f"Daily ETL triggered at: {datetime.now().isoformat()}")
        
        # Get execution date (from event or default to yesterday)
        detail = event.get('detail', {})
        execution_date = detail.get('execution_date', (datetime.now() - timedelta(days=1)).strftime('%Y-%m-%d'))
        
        print(f"Processing data for date: {execution_date}")
        
        # In production, trigger actual ETL workflow
        # For demo, publish a custom event
        eventbridge.put_events(
            Entries=[
                {
                    'Source': 'custom.etl',
                    'DetailType': 'Daily ETL Started',
                    'Detail': json.dumps({
                        'execution_date': execution_date,
                        'triggered_by': 'scheduled_rule',
                        'workflow_type': 'daily_batch',
                        'status': 'started'
                    }),
                    'EventBusName': 'data-pipeline-events'
                }
            ]
        )
        
        # Simulate ETL tasks
        tasks = [
            'Extract from data sources',
            'Validate data quality',
            'Transform data',
            'Load to data warehouse',
            'Update metadata catalog'
        ]
        
        for task in tasks:
            print(f"Executing: {task}")
        
        # Publish completion event
        eventbridge.put_events(
            Entries=[
                {
                    'Source': 'custom.etl',
                    'DetailType': 'Daily ETL Completed',
                    'Detail': json.dumps({
                        'execution_date': execution_date,
                        'tasks_completed': len(tasks),
                        'duration_seconds': 120,
                        'records_processed': 10000,
                        'status': 'success'
                    }),
                    'EventBusName': 'data-pipeline-events'
                }
            ]
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Daily ETL completed',
                'execution_date': execution_date,
                'tasks': len(tasks)
            })
        }
        
    except Exception as e:
        print(f"Error in daily ETL: {str(e)}")
        
        # Publish failure event
        eventbridge.put_events(
            Entries=[
                {
                    'Source': 'custom.etl',
                    'DetailType': 'Daily ETL Failed',
                    'Detail': json.dumps({
                        'error': str(e),
                        'execution_date': execution_date if 'execution_date' in locals() else 'unknown'
                    }),
                    'EventBusName': 'data-pipeline-events'
                }
            ]
        )
        
        raise e
```
5. Click **Deploy**

---

### Part 3: Create EventBridge Rules

#### Step 10: Create Rule - S3 CSV Upload Trigger
1. Go back to **EventBridge**
2. Click **Rules** in the left sidebar
3. Click **Create rule**
4. Configure rule:
   - **Name:** `s3-csv-upload-trigger`
   - **Description:** `Trigger ETL when CSV files are uploaded to S3`
   - **Event bus:** `default` (S3 events go to default bus)
   - **Rule type:** Rule with an event pattern
5. Click **Next**
6. **Event source:** AWS events or EventBridge partner events
7. **Event pattern:**
   - **AWS service:** S3
   - **Event type:** Amazon S3 Event Notification
   - Click **Edit pattern** and replace with:
```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["etl-source-data-ACCOUNT-ID"]
    },
    "object": {
      "key": [{
        "suffix": ".csv"
      }],
      "size": [{
        "numeric": ["<", 10485760]
      }]
    }
  }
}
```
8. **Important:** Replace `ACCOUNT-ID` with your account ID
9. Click **Next**
10. **Target:**
    - **Target types:** AWS service
    - **Select a target:** Lambda function
    - **Function:** `process-csv-upload`
11. Click **Next**
12. Click **Next** (skip tags)
13. Click **Create rule**

#### Step 11: Enable S3 Event Notifications
1. Navigate to **S3**
2. Click on bucket `etl-source-data-[account-id]`
3. Click **Properties** tab
4. Scroll to **Event notifications**
5. Click **Create event notification**
6. Configure:
   - **Event name:** `csv-upload-notification`
   - **Event types:** Check **All object create events**
   - **Destination:** EventBridge
   - Check **Send notifications to Amazon EventBridge**
7. Click **Save changes**

#### Step 12: Create Rule - Large File Handler
1. Go back to **EventBridge** → **Rules**
2. Click **Create rule**
3. Configure:
   - **Name:** `large-file-handler`
   - **Description:** `Route large files to SQS for batch processing`
   - **Event bus:** `default`
   - **Rule type:** Rule with an event pattern
4. Click **Next**
5. **Event pattern:**
```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["etl-source-data-ACCOUNT-ID"]
    },
    "object": {
      "size": [{
        "numeric": [">=", 10485760]
      }]
    }
  }
}
```
6. Replace `ACCOUNT-ID`
7. Click **Next**
8. **Target:** Lambda function → `handle-large-file`
9. Click **Next** → **Next** → **Create rule**

#### Step 13: Create Rule - Daily Scheduled ETL
1. Click **Create rule**
2. Configure:
   - **Name:** `daily-etl-schedule`
   - **Description:** `Trigger daily ETL at 2 AM UTC`
   - **Event bus:** `data-pipeline-events`
   - **Rule type:** Schedule
5. Click **Next**
6. **Schedule pattern:**
   - **Schedule type:** Cron expression
   - **Cron expression:** `0 2 * * ? *` (2 AM UTC daily)
7. Click **Next**
8. **Target:** Lambda function → `daily-etl-trigger`
9. **Additional settings:**
   - Configure target input: **Constant (JSON text)**
   - Input:
```json
{
  "detail": {
    "execution_date": "2026-06-24",
    "trigger_type": "scheduled"
  }
}
```
10. Click **Next** → **Next** → **Create rule**

#### Step 14: Create Rule with Input Transformer
1. Click **Create rule**
2. Configure:
   - **Name:** `transform-s3-event`
   - **Description:** `Normalize S3 events with input transformer`
   - **Event bus:** `default`
   - **Rule type:** Rule with an event pattern
3. Click **Next**
4. **Event pattern:**
```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"]
}
```
5. Click **Next**
6. **Target:** EventBridge event bus → `data-pipeline-events`
7. Expand **Additional settings**
8. **Configure input:** Input transformer
9. **Input path:**
```json
{
  "bucket": "$.detail.bucket.name",
  "key": "$.detail.object.key",
  "size": "$.detail.object.size",
  "timestamp": "$.time"
}
```
10. **Template:**
```json
{
  "source": "custom.s3normalized",
  "detail-type": "Normalized S3 Event",
  "detail": {
    "s3_bucket": <bucket>,
    "s3_key": <key>,
    "file_size_bytes": <size>,
    "event_timestamp": <timestamp>,
    "processing_stage": "ingestion"
  }
}
```
11. Click **Next** → **Next** → **Create rule**

---

### Part 4: Test Event-Driven Pipeline

#### Step 15: Test CSV Upload Trigger
1. Navigate to **S3**
2. Click on bucket `etl-source-data-[account-id]`
3. Click **Upload**
4. Upload a small CSV file (< 10 MB)
5. Click **Upload**
6. Wait 10-30 seconds
7. Navigate to **Lambda** → **Functions** → `process-csv-upload`
8. Click **Monitor** → **View CloudWatch logs**
9. Verify the function was triggered and processed the CSV

#### Step 16: Test Large File Handling
1. Create a large file (> 10 MB) on your local machine:
```bash
# On Mac/Linux
dd if=/dev/urandom of=large_test.dat bs=1m count=15
```
2. Upload to S3 bucket `etl-source-data-[account-id]`
3. Wait 10-30 seconds
4. Navigate to **SQS** → **Queues** → `large-file-batch-queue`
5. Click **Send and receive messages** → **Poll for messages**
6. Verify message was received with file details
7. Check Lambda `process-batch-queue` CloudWatch logs to see batch processing

#### Step 17: Test Daily Scheduled Trigger
1. Navigate to **EventBridge** → **Rules**
2. Click on `daily-etl-schedule`
3. Click **Edit**
4. Change cron expression to trigger in 2 minutes from current time
   - Example: If it's 14:35 UTC, use: `37 14 * * ? *`
5. Save and wait
6. Check Lambda `daily-etl-trigger` logs to verify execution

#### Step 18: Publish Custom Event Manually
1. Navigate to **EventBridge** → **Event buses**
2. Click on `data-pipeline-events`
3. Click **Send events**
4. Configure:
   - **Event source:** `custom.test`
   - **Detail type:** `Manual Test Event`
   - **Event detail:**
```json
{
  "test_id": "manual-001",
  "message": "Testing custom event publishing",
  "timestamp": "2026-06-24T10:00:00Z"
}
```
5. Click **Send**

#### Step 19: Test Event Replay
1. Navigate to **EventBridge** → **Archives**
2. Click on `data-pipeline-archive`
3. Click **Start replay**
4. Configure:
   - **Replay name:** `test-replay-001`
   - **Description:** `Testing event replay functionality`
   - **Time range:** Start and end times covering your test events
   - **Destination:** `data-pipeline-events`
5. Click **Start replay**
6. Monitor replay progress in the **Replays** section
7. Verify events are re-processed by checking Lambda logs

---

### Verification Checklist

- [ ] Custom event bus created
- [ ] Event archive configured with 7-day retention
- [ ] SQS queue created for batch processing
- [ ] All 4 Lambda functions deployed
- [ ] S3 event notifications enabled
- [ ] Rule created for CSV uploads (< 10 MB)
- [ ] Rule created for large files (>= 10 MB)
- [ ] Scheduled rule created for daily ETL
- [ ] Input transformer rule created
- [ ] CSV upload triggers correct Lambda
- [ ] Large file routed to SQS queue
- [ ] SQS trigger invokes batch processor
- [ ] Scheduled rule executes at configured time
- [ ] Custom events published successfully
- [ ] Event replay completed successfully

### Architecture Benefits

**Event-Driven:**
- Decoupled architecture
- React to events in real-time
- No polling required
- Automatic scaling

**Flexibility:**
- Multiple targets per rule
- Complex event filtering
- Input transformation
- Cross-account events

**Reliability:**
- Event archive for replay
- Built-in retry logic
- Dead-letter queues
- Event delivery guarantees

**Cost:**
- $1.00 per million events
- No minimum charges
- Pay only for what you use
- Free tier: 1M events/month

**Sample Cost Calculation:**
- 100,000 events/day = 3M events/month
- First 1M: Free
- Additional 2M: $2.00
- Lambda executions: ~$5/month
- SQS: ~$1/month
- **Total: ~$8/month**

---

### Cleanup

1. **Delete EventBridge Rules:**
   - EventBridge → Rules → Delete all created rules

2. **Delete Event Archive:**
   - EventBridge → Archives → Delete

3. **Delete Custom Event Bus:**
   - EventBridge → Event buses → Delete `data-pipeline-events`

4. **Delete Lambda Functions:**
   - Lambda → Functions → Delete all 4 functions

5. **Delete SQS Queue:**
   - SQS → Queues → Delete

6. **Disable S3 Event Notifications:**
   - S3 → Bucket → Properties → Event notifications → Delete

7. **Delete IAM Role:**
   - IAM → Roles → Delete `eventbridge-lambda-role`

---

## Exercise 14.3: Amazon MSK Streaming Pipeline

### Introduction
Build a real-time streaming data pipeline using Amazon Managed Streaming for Apache Kafka (MSK) that ingests high-throughput event streams, processes data with Lambda consumers, and monitors performance metrics.

**What You'll Build:**
- VPC and networking for MSK cluster
- MSK cluster with 3 brokers across multiple AZs
- Kafka topics for streaming data
- Lambda producer generating 1000 events/second
- Lambda consumer with MSK trigger
- CloudWatch monitoring dashboard

**Duration:** 90 minutes  
**Cost:** ~$300/month (MSK cluster with 3 kafka.m5.large brokers running 24/7)

**Note:** MSK has no free tier. This exercise will incur costs immediately. Consider cleanup promptly.

---

### Part 1: Create VPC and Networking

#### Step 1: Create VPC for MSK
1. Navigate to **VPC**
2. Click **Create VPC**
3. Configure:
   - **Resources to create:** VPC and more
   - **Name tag:** `msk-vpc`
   - **IPv4 CIDR:** `10.0.0.0/16`
   - **IPv6 CIDR:** No IPv6
   - **Tenancy:** Default
   - **Number of Availability Zones:** 3
   - **Number of public subnets:** 0
   - **Number of private subnets:** 3
   - **NAT gateways:** None (use VPC endpoints instead)
   - **VPC endpoints:** S3 Gateway
4. Click **Create VPC**
5. Wait for creation (~2 minutes)

#### Step 2: Create Security Group for MSK
1. Navigate to **VPC** → **Security Groups**
2. Click **Create security group**
3. Configure:
   - **Security group name:** `msk-cluster-sg`
   - **Description:** `Security group for MSK cluster`
   - **VPC:** Select `msk-vpc`
4. **Inbound rules:** Click **Add rule**
   - **Type:** Custom TCP
   - **Port range:** `9092-9098`
   - **Source:** Custom → `10.0.0.0/16`
   - **Description:** `Kafka broker communication`
5. Click **Add rule**
   - **Type:** Custom TCP
   - **Port range:** `2181`
   - **Source:** Custom → `10.0.0.0/16`
   - **Description:** `Zookeeper`
6. Click **Create security group**

#### Step 3: Create Security Group for EC2 Client
1. Click **Create security group**
2. Configure:
   - **Security group name:** `msk-client-sg`
   - **Description:** `Security group for MSK clients (Lambda, EC2)`
   - **VPC:** Select `msk-vpc`
3. **Outbound rules:** (Default allows all) - Keep as is
4. Click **Create security group**

#### Step 4: Update MSK Security Group
1. Go back to **Security Groups**
2. Select `msk-cluster-sg`
3. Click **Edit inbound rules**
4. For the existing rule, change **Source** from `10.0.0.0/16` to:
   - **Source type:** Security group
   - **Source:** `msk-client-sg`
5. Click **Save rules**

---

### Part 2: Create MSK Cluster

#### Step 5: Create MSK Cluster Configuration (Optional)
1. Navigate to **Amazon MSK**
2. In the left sidebar, click **Configurations**
3. Click **Create configuration**
4. Configure:
   - **Configuration name:** `custom-msk-config`
   - **Description:** `Custom configuration for data pipeline`
   - **Kafka version:** 3.5.1
   - **Configuration properties:**
```properties
auto.create.topics.enable=true
log.retention.hours=168
log.retention.bytes=1073741824
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
compression.type=snappy
```
5. Click **Create**

#### Step 6: Create MSK Cluster
1. Navigate to **Amazon MSK** → **Clusters**
2. Click **Create cluster**
3. **Creation method:** Custom create
4. Click **Next**
5. **Cluster settings:**
   - **Cluster name:** `data-pipeline-cluster`
   - **Cluster type:** Provisioned
   - **Apache Kafka version:** 3.5.1
6. **Broker settings:**
   - **Broker instance type:** kafka.m5.large
   - **Number of zones:** 3
   - **Brokers per zone:** 1
   - **Total brokers:** 3
   - **Storage:** 100 GB per broker (EBS, gp3)
7. **Networking:**
   - **VPC:** Select `msk-vpc`
   - **Subnets:** Select all 3 private subnets
   - **Security groups:** Select `msk-cluster-sg`
8. **Access control methods:**
   - **Authentication:** Unauthenticated access (for demo; use IAM in production)
   - **Encryption in transit:** TLS encryption within cluster and between clients and brokers
9. **Configuration:**
   - **Configuration:** Select `custom-msk-config` (or use default)
10. **Monitoring:**
    - **CloudWatch metrics:** Enhanced topic-level monitoring
    - **Prometheus monitoring:** Open monitoring with Prometheus (optional)
    - **Broker log delivery:** Enable (optional)
      - CloudWatch logs: Enable
      - Log group: `/aws/msk/data-pipeline-cluster`
11. Click **Next**
12. Review and click **Create cluster**
13. **Wait 15-25 minutes** for cluster creation

#### Step 7: Note Cluster Endpoints
1. Once cluster state is "Active", click on the cluster name
2. Click **View client information**
3. Copy the **Bootstrap servers** (both plaintext and TLS):
   - Plaintext: `b-1.xxx.kafka.us-east-1.amazonaws.com:9092,b-2...`
   - TLS: `b-1.xxx.kafka.us-east-1.amazonaws.com:9094,b-2...`
4. Save these for later use

---

### Part 3: Create and Configure Kafka Topics

#### Step 8: Launch EC2 Instance as Kafka Client
1. Navigate to **EC2**
2. Click **Launch instance**
3. Configure:
   - **Name:** `msk-kafka-client`
   - **AMI:** Amazon Linux 2023
   - **Instance type:** t3.medium
   - **Key pair:** Create or select existing
   - **Network settings:**
     - **VPC:** `msk-vpc`
     - **Subnet:** Any private subnet
     - **Auto-assign public IP:** Disable
     - **Security group:** Select `msk-client-sg`
   - **Advanced details:**
     - **IAM instance profile:** Create role with `AmazonSSMManagedInstanceCore` policy
4. Click **Launch instance**
5. Wait for instance to be in "Running" state

#### Step 9: Connect to EC2 and Install Kafka Tools
1. Navigate to **EC2** → **Instances**
2. Select `msk-kafka-client`
3. Click **Connect** → **Session Manager** → **Connect**
4. In the terminal, run:
```bash
# Install Java
sudo yum install java-11-amazon-corretto -y

# Download Kafka
wget https://archive.apache.org/dist/kafka/3.5.1/kafka_2.13-3.5.1.tgz
tar -xzf kafka_2.13-3.5.1.tgz
cd kafka_2.13-3.5.1

# Set bootstrap servers (replace with your actual bootstrap servers)
export BOOTSTRAP_SERVERS="b-1.xxx.kafka.us-east-1.amazonaws.com:9092,b-2.xxx.kafka.us-east-1.amazonaws.com:9092,b-3.xxx.kafka.us-east-1.amazonaws.com:9092"
```

#### Step 10: Create Kafka Topics
1. Still in the EC2 terminal, create topics:
```bash
# Create topic for raw events
bin/kafka-topics.sh --create \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --replication-factor 3 \
  --partitions 6 \
  --topic raw-events

# Create topic for processed events
bin/kafka-topics.sh --create \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --replication-factor 3 \
  --partitions 6 \
  --topic processed-events

# Create topic for alerts
bin/kafka-topics.sh --create \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --replication-factor 3 \
  --partitions 3 \
  --topic alerts

# List topics to verify
bin/kafka-topics.sh --list \
  --bootstrap-server $BOOTSTRAP_SERVERS
```

#### Step 11: Test Topic with Sample Data
1. In one terminal window, start a consumer:
```bash
bin/kafka-console-consumer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --topic raw-events \
  --from-beginning
```
2. Open another Session Manager connection to the same EC2 instance
3. In the second terminal, produce test messages:
```bash
cd kafka_2.13-3.5.1
export BOOTSTRAP_SERVERS="<your-bootstrap-servers>"

bin/kafka-console-producer.sh \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --topic raw-events
```
4. Type some test messages and press Enter after each
5. Verify messages appear in the consumer terminal
6. Press Ctrl+C in both terminals to stop

---

### Part 4: Create Lambda Producer and Consumer

#### Step 12: Create IAM Role for Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select **Lambda** → **Next**
4. Attach policies:
   - `AWSLambdaBasicExecutionRole`
   - `AWSLambdaVPCAccessExecutionRole`
   - `AmazonMSKFullAccess`
5. Role name: `lambda-msk-role`
6. Click **Create role**

#### Step 13: Create Lambda Layer for Kafka Python
1. On your local machine:
```bash
mkdir -p python/lib/python3.11/site-packages
pip install kafka-python -t python/lib/python3.11/site-packages
zip -r kafka-python-layer.zip python
```
2. Navigate to **Lambda** → **Layers**
3. Click **Create layer**
4. Configure:
   - **Name:** `kafka-python-layer`
   - **Upload:** Upload `kafka-python-layer.zip`
   - **Compatible runtimes:** Python 3.11
5. Click **Create**

#### Step 14: Create Lambda Producer Function
1. Navigate to **Lambda** → **Functions**
2. Click **Create function**
3. Configure:
   - **Function name:** `msk-event-producer`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `lambda-msk-role`
4. Click **Create function**
5. Add the `kafka-python-layer` layer:
   - Click **Add a layer**
   - Select **Custom layers** → `kafka-python-layer`
   - Click **Add**
6. Replace code:
```python
import json
import os
import time
import random
from datetime import datetime
from kafka import KafkaProducer

BOOTSTRAP_SERVERS = os.environ.get('BOOTSTRAP_SERVERS', '').split(',')
TOPIC = 'raw-events'

def lambda_handler(event, context):
    try:
        print(f"Connecting to Kafka: {BOOTSTRAP_SERVERS}")
        
        producer = KafkaProducer(
            bootstrap_servers=BOOTSTRAP_SERVERS,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks='all',
            compression_type='snappy',
            retries=3
        )
        
        # Generate events
        events_per_execution = event.get('events_count', 100)
        print(f"Producing {events_per_execution} events")
        
        event_types = ['page_view', 'click', 'purchase', 'login', 'logout', 'search']
        users = [f'user_{i}' for i in range(1, 101)]
        
        sent_count = 0
        for i in range(events_per_execution):
            event_data = {
                'event_id': f'{context.request_id}_{i}',
                'event_type': random.choice(event_types),
                'user_id': random.choice(users),
                'timestamp': datetime.now().isoformat(),
                'properties': {
                    'page': f'/page_{random.randint(1, 50)}',
                    'session_id': f'session_{random.randint(1000, 9999)}',
                    'value': round(random.uniform(1, 1000), 2)
                }
            }
            
            future = producer.send(TOPIC, value=event_data)
            sent_count += 1
            
            # Log every 100th event
            if (i + 1) % 100 == 0:
                print(f"Sent {i + 1} events")
        
        # Wait for all messages to be sent
        producer.flush()
        producer.close()
        
        print(f"Successfully produced {sent_count} events to topic: {TOPIC}")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'events_produced': sent_count,
                'topic': TOPIC,
                'execution_id': context.request_id
            })
        }
        
    except Exception as e:
        print(f"Error producing events: {str(e)}")
        raise e
```
7. Click **Deploy**
8. Click **Configuration** → **Environment variables** → **Edit**
9. Add variable:
   - **Key:** `BOOTSTRAP_SERVERS`
   - **Value:** Your MSK bootstrap servers (comma-separated)
10. Click **Save**
11. Click **Configuration** → **VPC** → **Edit**
12. Configure:
    - **VPC:** `msk-vpc`
    - **Subnets:** Select all 3 private subnets
    - **Security groups:** `msk-client-sg`
13. Click **Save**
14. Set **Timeout:** 5 minutes
15. Set **Memory:** 512 MB

#### Step 15: Create Lambda Consumer Function
1. Click **Create function**
2. Configure:
   - **Function name:** `msk-event-consumer`
   - **Runtime:** Python 3.11
   - **Permissions:** Use existing role → `lambda-msk-role`
3. Click **Create function**
4. Add `kafka-python-layer` layer
5. Replace code:
```python
import json
import base64
from datetime import datetime

def lambda_handler(event, context):
    try:
        print(f"Received batch with {len(event['records'])} topic partitions")
        
        total_records = 0
        event_type_counts = {}
        
        # Process records from all partitions
        for topic_partition, records in event['records'].items():
            print(f"Processing {len(records)} records from {topic_partition}")
            
            for record in records:
                # Decode the Kafka message
                message_bytes = base64.b64decode(record['value'])
                message = json.loads(message_bytes.decode('utf-8'))
                
                total_records += 1
                
                # Count event types
                event_type = message.get('event_type', 'unknown')
                event_type_counts[event_type] = event_type_counts.get(event_type, 0) + 1
                
                # Process based on event type
                if event_type == 'purchase':
                    value = message.get('properties', {}).get('value', 0)
                    if value > 500:
                        print(f"High-value purchase detected: {value} from {message.get('user_id')}")
                
                # Log sample events (every 100th)
                if total_records % 100 == 0:
                    print(f"Sample event #{total_records}: {json.dumps(message)}")
        
        print(f"\n=== Processing Summary ===")
        print(f"Total records processed: {total_records}")
        print(f"Event type distribution: {json.dumps(event_type_counts, indent=2)}")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'records_processed': total_records,
                'event_types': event_type_counts
            })
        }
        
    except Exception as e:
        print(f"Error processing events: {str(e)}")
        raise e
```
6. Click **Deploy**
7. Configure VPC (same as producer)
8. Set **Timeout:** 5 minutes
9. Set **Memory:** 512 MB

#### Step 16: Add MSK Trigger to Consumer
1. Still on `msk-event-consumer` function page
2. Click **Add trigger**
3. Configure:
   - **Source:** MSK
   - **MSK cluster:** Select `data-pipeline-cluster`
   - **Batch size:** 100
   - **Batch window:** 10 seconds
   - **Starting position:** Latest
   - **Topic name:** `raw-events`
   - **Authentication:** SASL_SCRAM (if enabled) or leave as default
   - **Enable trigger:** Yes
4. Click **Add**

---

### Part 5: Test and Monitor Streaming Pipeline

#### Step 17: Test Producer Function
1. Go to `msk-event-producer` function
2. Click **Test** tab
3. Create test event:
```json
{
  "events_count": 1000
}
```
4. Click **Test**
5. Wait for execution (~30-60 seconds)
6. Check execution results and logs

#### Step 18: Generate High-Volume Events
1. To simulate 1000 events/second, create a scheduled rule:
   - Navigate to **EventBridge** → **Rules**
   - Click **Create rule**
   - Name: `msk-producer-schedule`
   - Schedule: Rate expression → `1 minute`
   - Target: Lambda → `msk-event-producer`
   - Constant JSON: `{"events_count": 60000}`
   - Create rule
2. This will produce 60,000 events per minute (1,000/second)
3. **Important:** Disable this rule after testing to avoid costs!

#### Step 19: Monitor Consumer Processing
1. Navigate to `msk-event-consumer` function
2. Click **Monitor** → **View CloudWatch logs**
3. Open latest log stream
4. Verify batches of events being processed
5. Check processing summary statistics

#### Step 20: Monitor MSK Cluster Metrics
1. Navigate to **Amazon MSK** → **Clusters**
2. Click on `data-pipeline-cluster`
3. Click **Monitoring** tab
4. Review metrics:
   - **BytesInPerSec:** Data ingress rate
   - **BytesOutPerSec:** Data egress rate
   - **MessagesInPerSec:** Message throughput
   - **FetchConsumerTotalTimeMs:** Consumer latency
   - **ProduceTotalTimeMs:** Producer latency
   - **PartitionCount:** Number of partitions
   - **UnderReplicatedPartitions:** Should be 0
5. Click **View in CloudWatch** for detailed metrics

#### Step 21: Create CloudWatch Dashboard
1. Navigate to **CloudWatch** → **Dashboards**
2. Click **Create dashboard**
3. Name: `MSK-Pipeline-Dashboard`
4. Click **Create dashboard**
5. Add widgets:
   - Click **Add widget** → **Line**
   - Select **Metrics**
   - Choose **MSK** namespace
   - Select metrics:
     - BytesInPerSec (per broker)
     - BytesOutPerSec (per broker)
     - MessagesInPerSec
   - Click **Create widget**
6. Add another widget for consumer metrics:
   - Lambda → Consumer function
   - Invocations, Duration, Errors
7. Click **Save dashboard**

#### Step 22: Test Manual Kafka Operations
1. Connect to EC2 client instance via Session Manager
2. Describe topic details:
```bash
cd kafka_2.13-3.5.1
export BOOTSTRAP_SERVERS="<your-servers>"

bin/kafka-topics.sh --describe \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --topic raw-events
```
3. View consumer group information:
```bash
bin/kafka-consumer-groups.sh --list \
  --bootstrap-server $BOOTSTRAP_SERVERS

bin/kafka-consumer-groups.sh --describe \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --group <lambda-consumer-group-name>
```
4. Check topic lag:
```bash
bin/kafka-consumer-groups.sh --describe \
  --bootstrap-server $BOOTSTRAP_SERVERS \
  --group <group-name> \
  --members --verbose
```

---

### Verification Checklist

- [ ] VPC created with 3 private subnets in 3 AZs
- [ ] Security groups configured for MSK and clients
- [ ] MSK cluster created with 3 brokers (kafka.m5.large)
- [ ] MSK cluster status is "Active"
- [ ] Kafka topics created (raw-events, processed-events, alerts)
- [ ] EC2 client instance can connect to Kafka brokers
- [ ] Console producer and consumer working
- [ ] Lambda producer function deployed in VPC
- [ ] Lambda consumer function deployed with MSK trigger
- [ ] Producer successfully sends events to Kafka
- [ ] Consumer processes events from Kafka topic
- [ ] CloudWatch metrics showing throughput
- [ ] Dashboard created with key metrics
- [ ] No under-replicated partitions
- [ ] Consumer lag is minimal

### Architecture Benefits

**Scalability:**
- Handles millions of events per second
- Horizontal scaling by adding brokers
- Partition-level parallelism
- Automatic consumer rebalancing

**Durability:**
- Data replicated across 3 brokers
- Configurable retention (7 days default)
- Fault tolerance with multi-AZ deployment
- At-least-once delivery guarantee

**Performance:**
- Single-digit millisecond latency
- High throughput (MB/s per partition)
- Compression support (Snappy, LZ4, GZIP)
- Zero-copy data transfer

**Managed Service:**
- No server management
- Automatic patching and updates
- Built-in monitoring
- Integration with AWS services

**Cost Optimization:**
- Right-size broker instances
- Use Apache Kafka vs MSK Serverless for predictable workloads
- Enable compression to reduce storage
- Archive old data to S3 with Kafka Connect

**Sample Cost Calculation (24/7 operation):**
- 3 × kafka.m5.large: $0.336/hour × 730 hours = $735.84/month
- Storage: 300 GB × $0.10/GB = $30/month
- Data transfer (minimal within AZ): ~$5/month
- **Total: ~$770/month**

**Cost Reduction Options:**
- Use smaller instances: kafka.t3.small (~$80/month for 3 brokers)
- MSK Serverless: Pay for throughput and storage (~$50-150/month for moderate workloads)
- Reduce retention period
- Turn off cluster when not needed (dev/test)

---

### Cleanup (IMPORTANT - Avoid Ongoing Costs!)

1. **Disable EventBridge Schedule:**
   - EventBridge → Rules → Disable or delete `msk-producer-schedule`

2. **Delete Lambda MSK Trigger:**
   - Lambda → `msk-event-consumer` → Configuration → Triggers → Remove MSK trigger

3. **Delete Lambda Functions:**
   - Lambda → Functions → Delete both producer and consumer

4. **Delete Lambda Layer:**
   - Lambda → Layers → Delete `kafka-python-layer`

5. **Delete MSK Cluster:** (**This is critical to stop charges!**)
   - Amazon MSK → Clusters → Select cluster → **Delete**
   - Type cluster name to confirm
   - This takes 10-15 minutes

6. **Terminate EC2 Instance:**
   - EC2 → Instances → Terminate

7. **Delete VPC and Components:**
   - VPC → Your VPCs → Select `msk-vpc` → **Delete VPC**
   - This deletes associated subnets, route tables, endpoints

8. **Delete Security Groups:**
   - If not deleted with VPC: VPC → Security Groups → Delete

9. **Delete IAM Role:**
   - IAM → Roles → Delete `lambda-msk-role`

10. **Delete CloudWatch Dashboard:**
    - CloudWatch → Dashboards → Delete

11. **Delete CloudWatch Log Groups:**
    - CloudWatch → Log groups → Delete `/aws/msk/...` and Lambda log groups

---

## Summary

### What You've Learned

**Exercise 14.1 - Step Functions ETL Orchestration:**
- Built complex workflows with parallel execution
- Implemented data quality validation with conditional branching
- Configured error handling with exponential backoff retry
- Integrated Lambda, Glue, Athena, and SNS services
- Created visual workflow monitoring
- Achieved 60-70% runtime reduction with parallel extraction

**Exercise 14.2 - EventBridge Event-Driven Pipeline:**
- Designed event-driven architecture with custom event bus
- Created multiple rules with complex event filtering
- Implemented input transformers for event normalization
- Built batch processing with SQS integration
- Configured scheduled triggers for daily ETL
- Enabled event replay for testing and recovery
- Decoupled pipeline components for flexibility

**Exercise 14.3 - Amazon MSK Streaming:**
- Deployed managed Kafka cluster with multi-AZ replication
- Created and configured Kafka topics with partitions
- Built high-throughput producer (1000 events/second)
- Implemented Lambda consumer with MSK trigger
- Monitored streaming metrics in CloudWatch
- Managed cluster performance and consumer lag

### Key Takeaways

1. **Step Functions provide visual orchestration** for complex ETL workflows without writing custom state management code

2. **EventBridge enables true event-driven architecture** with powerful filtering, transformation, and routing capabilities

3. **Amazon MSK offers managed Kafka** with enterprise features: multi-AZ deployment, automatic patching, CloudWatch integration

4. **Choose the right service for your use case:**
   - Step Functions: Complex workflows with branching logic
   - EventBridge: Event-driven triggers and routing
   - MSK/Kinesis: High-throughput streaming data

5. **Cost considerations:**
   - Step Functions: Pay per state transition (~$2-30/month for typical ETL)
   - EventBridge: $1 per million events (very cost-effective)
   - MSK: Fixed cost for provisioned clusters (~$80-770/month depending on instance type)

### Real-World Applications

**Step Functions:**
- Multi-stage data pipelines with quality checks
- ML model training workflows
- Complex approval processes
- Disaster recovery orchestration

**EventBridge:**
- Real-time data ingestion triggers
- Cross-account event routing
- Microservices integration
- Automated incident response

**Amazon MSK:**
- Real-time analytics and dashboards
- Log aggregation at scale
- Event sourcing architectures
- Change data capture (CDC)
- IoT data ingestion

### Architecture Patterns

**Lambda + Step Functions:**
- Best for: Orchestrated batch processing
- Latency: Seconds to minutes
- Cost: Very low ($0.025 per 1000 transitions)
- Use when: Complex workflows with branching

**Lambda + EventBridge:**
- Best for: Event-driven automation
- Latency: Milliseconds to seconds
- Cost: Very low ($1 per million events)
- Use when: Reactive processing, decoupled services

**Lambda + MSK:**
- Best for: High-throughput streaming
- Latency: Single-digit milliseconds
- Cost: Medium to high (cluster costs)
- Use when: >10,000 events/second, event ordering required

### Cost Optimization Tips

1. **Step Functions:**
   - Use Express Workflows for high-volume, short-duration tasks ($1 per million requests vs Standard)
   - Minimize state transitions by batching operations
   - Use Wait states instead of polling Lambda

2. **EventBridge:**
   - Leverage the 1M free tier per month
   - Use event filtering to reduce downstream Lambda invocations
   - Archive only necessary events

3. **Amazon MSK:**
   - Right-size broker instances based on throughput requirements
   - Consider MSK Serverless for variable workloads
   - Use compression to reduce storage costs
   - Set appropriate retention periods
   - Delete cluster when not in use (dev/test)

### Best Practices

**Security:**
- Use IAM roles with least privilege permissions
- Enable encryption in transit and at rest (MSK)
- Store credentials in Secrets Manager
- Use VPC endpoints to avoid internet exposure

**Monitoring:**
- Enable CloudWatch Logs for all services
- Set up CloudWatch Alarms for failures
- Create dashboards for key metrics
- Use X-Ray for distributed tracing

**Reliability:**
- Implement retry logic with exponential backoff
- Use dead-letter queues for failed events
- Configure appropriate timeouts
- Test event replay and recovery procedures

**Performance:**
- Use parallel execution where possible (Step Functions, MSK partitions)
- Batch operations to reduce API calls
- Configure appropriate Lambda memory and timeout
- Monitor and tune MSK partition counts

### Next Steps

- Explore AWS Step Functions Express Workflows for high-volume workloads
- Implement EventBridge Schema Registry for event governance
- Deploy Kafka Connect for MSK integration with databases
- Build real-time analytics with MSK + Kinesis Data Analytics
- Create ML pipelines with Step Functions Data Science SDK
- Implement event sourcing patterns with EventBridge and DynamoDB

---

## Exam Alignment

**DEA-C01 Coverage:**

**Domain 1: Data Ingestion and Transformation (34%)**
- Event-driven data ingestion with EventBridge
- Stream processing with Amazon MSK
- ETL orchestration with Step Functions
- Lambda-based data transformation
- Multi-source data extraction patterns

**Domain 2: Data Store Management (26%)**
- Streaming data storage with Kafka topics
- Data partitioning strategies
- Event archival and replay
- Data retention policies

**Domain 3: Data Operations and Support (22%)**
- Workflow orchestration and monitoring
- Error handling and retry logic
- CloudWatch metrics and alarms
- Performance tuning (MSK partitions, Lambda concurrency)
- Operational monitoring dashboards

**Domain 4: Data Security and Governance (24%)**
- IAM roles and policies for service integration
- VPC configuration for secure streaming
- Encryption in transit and at rest
- Event-driven audit logging
- Data quality validation

### Practice Questions

1. **When should you choose Step Functions over Lambda alone for ETL workflows?**
   - When you need visual workflow monitoring
   - When you have complex branching logic
   - When you need built-in retry and error handling
   - All of the above ✓

2. **What is the primary benefit of EventBridge event archive?**
   - Cost savings
   - Event replay for testing and recovery ✓
   - Faster event processing
   - Automatic scaling

3. **How does MSK achieve fault tolerance?**
   - Single broker with snapshots
   - Multi-AZ deployment with replication ✓
   - Automatic failover to S3
   - Manual backup and restore

4. **What is the recommended approach for handling large files in an event-driven pipeline?**
   - Process inline in Lambda
   - Route to SQS for batch processing ✓
   - Store in RDS
   - Use Step Functions Map state

5. **Which AWS service provides the lowest latency for streaming data processing?**
   - Step Functions
   - EventBridge
   - Amazon MSK ✓
   - SQS

---

**Total Cost Summary:**
- Exercise 14.1 (Step Functions): ~$30/month (100 daily executions)
- Exercise 14.2 (EventBridge): ~$8/month (moderate event volume)
- Exercise 14.3 (MSK): ~$770/month (3 × kafka.m5.large, 24/7)
- **Total if all running continuously: ~$808/month**

**Important:** MSK is the primary cost driver. Use smaller instances or MSK Serverless for cost-effective learning, and delete the cluster immediately after completing the exercise.

**Estimated Time Investment:** 5 hours (120 + 90 + 90 minutes)

---

*This guide is part of the AWS Data Engineer Associate (DEA-C01) certification preparation series.*

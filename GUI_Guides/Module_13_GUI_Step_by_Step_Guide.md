# Module 13: Developer Tools for Data Engineering - GUI Step-by-Step Guide

## Overview
This module covers AWS Developer Tools essential for building robust, production-ready data engineering pipelines. You'll learn to implement CI/CD pipelines, distributed tracing, and AI-powered code quality analysis for your data workflows.

**Duration:** 5 hours  
**Cost:** $15-20/month (with cost optimization)  
**Difficulty:** Advanced

---

## Exercise 13.1: CI/CD Pipeline for Lambda ETL

### Introduction
Build a complete CI/CD pipeline for serverless ETL functions using AWS native tools. Implement automated testing, canary deployments, and automatic rollback capabilities to ensure zero-downtime deployments.

**What You'll Build:**
- CodeCommit repository with Lambda ETL code
- CodeBuild project with automated testing and coverage
- CodeDeploy with canary deployment strategy
- CodePipeline orchestrating the entire workflow
- CloudWatch alarms for automatic rollback

**Duration:** 120 minutes  
**Cost:** $5-8/month (1,000 builds)

---

### Part 1: Create CodeCommit Repository and Lambda ETL Code

#### Step 1: Create CodeCommit Repository
1. Open the **AWS Console** and navigate to **CodeCommit**
2. Click **Create repository**
3. Configure repository:
   - **Repository name:** `lambda-etl-pipeline`
   - **Description:** `Lambda ETL function with CI/CD pipeline`
   - **Tags:** (Optional) Add tags for organization
4. Click **Create**
5. Note the **Clone URL** (HTTPS or SSH)

#### Step 2: Set Up Git Credentials
1. Navigate to **IAM** → **Users**
2. Select your IAM user
3. Click **Security credentials** tab
4. Scroll to **HTTPS Git credentials for AWS CodeCommit**
5. Click **Generate credentials**
6. **Download credentials** (username and password)
7. Save these securely - you'll need them to push code

#### Step 3: Clone Repository Locally
1. Open your terminal/command prompt
2. Clone the repository:
```bash
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/lambda-etl-pipeline
cd lambda-etl-pipeline
```
3. Enter your CodeCommit credentials when prompted

#### Step 4: Create Lambda ETL Function Code
1. Create `lambda_function.py`:
```python
import json
import boto3
import csv
from datetime import datetime
from io import StringIO

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

def lambda_handler(event, context):
    """
    ETL function to process sales data from S3
    Transform and load into DynamoDB
    """
    try:
        # Get source bucket and key from event
        source_bucket = event.get('source_bucket', 'sales-data-source')
        source_key = event.get('source_key', 'sales/input.csv')
        table_name = event.get('table_name', 'ProcessedSales')
        
        # Extract: Read CSV from S3
        response = s3.get_object(Bucket=source_bucket, Key=source_key)
        csv_content = response['Body'].read().decode('utf-8')
        
        # Transform: Parse and validate data
        csv_reader = csv.DictReader(StringIO(csv_content))
        processed_records = []
        
        for row in csv_reader:
            # Data validation
            if not row.get('order_id') or not row.get('amount'):
                continue
            
            # Data transformation
            transformed = {
                'order_id': row['order_id'],
                'customer_name': row.get('customer_name', 'Unknown'),
                'product': row.get('product', 'Unknown'),
                'amount': float(row.get('amount', 0)),
                'order_date': row.get('order_date', datetime.now().isoformat()),
                'processed_at': datetime.now().isoformat(),
                'status': 'processed'
            }
            
            processed_records.append(transformed)
        
        # Load: Write to DynamoDB
        table = dynamodb.Table(table_name)
        with table.batch_writer() as batch:
            for record in processed_records:
                batch.put_item(Item=record)
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'ETL completed successfully',
                'records_processed': len(processed_records),
                'source': f's3://{source_bucket}/{source_key}',
                'destination': table_name
            })
        }
        
    except Exception as e:
        print(f"Error in ETL: {str(e)}")
        raise

def validate_record(record):
    """Validate individual record"""
    required_fields = ['order_id', 'amount']
    return all(field in record and record[field] for field in required_fields)

def calculate_total(records):
    """Calculate total amount from records"""
    return sum(float(r.get('amount', 0)) for r in records)
```

#### Step 5: Create Unit Tests
1. Create `test_lambda_function.py`:
```python
import unittest
import json
from unittest.mock import patch, MagicMock
from lambda_function import lambda_handler, validate_record, calculate_total

class TestLambdaETL(unittest.TestCase):
    
    @patch('lambda_function.s3')
    @patch('lambda_function.dynamodb')
    def test_successful_etl(self, mock_dynamodb, mock_s3):
        """Test successful ETL execution"""
        # Mock S3 response
        csv_data = "order_id,customer_name,product,amount,order_date\n1,John Doe,Laptop,1200.00,2024-01-01"
        mock_s3.get_object.return_value = {
            'Body': MagicMock(read=lambda: csv_data.encode('utf-8'))
        }
        
        # Mock DynamoDB table
        mock_table = MagicMock()
        mock_dynamodb.Table.return_value = mock_table
        
        # Execute
        event = {
            'source_bucket': 'test-bucket',
            'source_key': 'test.csv',
            'table_name': 'TestTable'
        }
        result = lambda_handler(event, {})
        
        # Assert
        self.assertEqual(result['statusCode'], 200)
        body = json.loads(result['body'])
        self.assertEqual(body['records_processed'], 1)
    
    def test_validate_record_valid(self):
        """Test record validation with valid data"""
        record = {'order_id': '123', 'amount': '100.00'}
        self.assertTrue(validate_record(record))
    
    def test_validate_record_invalid(self):
        """Test record validation with invalid data"""
        record = {'order_id': '', 'amount': '100.00'}
        self.assertFalse(validate_record(record))
    
    def test_calculate_total(self):
        """Test total calculation"""
        records = [
            {'amount': '100.00'},
            {'amount': '200.50'},
            {'amount': '50.25'}
        ]
        total = calculate_total(records)
        self.assertEqual(total, 350.75)
    
    @patch('lambda_function.s3')
    def test_s3_error_handling(self, mock_s3):
        """Test S3 error handling"""
        mock_s3.get_object.side_effect = Exception("S3 Error")
        
        event = {'source_bucket': 'test', 'source_key': 'test.csv'}
        
        with self.assertRaises(Exception):
            lambda_handler(event, {})

if __name__ == '__main__':
    unittest.main()
```

#### Step 6: Create BuildSpec File
1. Create `buildspec.yml`:
```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.11
    commands:
      - echo "Installing dependencies..."
      - pip install --upgrade pip
      - pip install boto3 coverage pytest pytest-cov moto
  
  pre_build:
    commands:
      - echo "Running unit tests with coverage..."
      - python -m pytest test_lambda_function.py -v --cov=lambda_function --cov-report=term --cov-report=html
      - echo "Running code quality checks..."
      - python -m py_compile lambda_function.py
  
  build:
    commands:
      - echo "Building deployment package..."
      - mkdir -p build
      - cp lambda_function.py build/
      - cd build
      - pip install boto3 -t .
      - zip -r ../lambda-deployment.zip .
      - cd ..
      - echo "Build completed on $(date)"
  
  post_build:
    commands:
      - echo "Preparing artifacts..."
      - ls -lh lambda-deployment.zip
      - echo "Deployment package size:"
      - du -h lambda-deployment.zip

artifacts:
  files:
    - lambda-deployment.zip
    - appspec.yml
  discard-paths: no

reports:
  coverage-report:
    files:
      - 'htmlcov/index.html'
    file-format: 'HTML'
  
cache:
  paths:
    - '/root/.cache/pip/**/*'
```

#### Step 7: Create AppSpec File for CodeDeploy
1. Create `appspec.yml`:
```yaml
version: 0.0
Resources:
  - LambdaETLFunction:
      Type: AWS::Lambda::Function
      Properties:
        Name: "sales-etl-function"
        Alias: "live"
        CurrentVersion: "1"
        TargetVersion: "2"
Hooks:
  - BeforeAllowTraffic: "PreTrafficHook"
  - AfterAllowTraffic: "PostTrafficHook"
```

#### Step 8: Create Requirements File
1. Create `requirements.txt`:
```
boto3>=1.26.0
pytest>=7.4.0
pytest-cov>=4.1.0
coverage>=7.2.0
moto>=4.1.0
```

#### Step 9: Commit and Push Code
1. Add all files:
```bash
git add lambda_function.py
git add test_lambda_function.py
git add buildspec.yml
git add appspec.yml
git add requirements.txt
```
2. Commit:
```bash
git commit -m "Initial commit: Lambda ETL with tests and build spec"
```
3. Push to CodeCommit:
```bash
git push origin main
```

---

### Part 2: Create Lambda Function and DynamoDB Table

#### Step 10: Create DynamoDB Table
1. Navigate to **DynamoDB**
2. Click **Create table**
3. Configure table:
   - **Table name:** `ProcessedSales`
   - **Partition key:** `order_id` (String)
   - **Table settings:** Default settings
   - **Read/write capacity:** On-demand
4. Click **Create table**

#### Step 11: Create S3 Buckets
1. Navigate to **S3**
2. Click **Create bucket**
3. Create source bucket:
   - **Bucket name:** `sales-data-source-[account-id]`
   - **Region:** us-east-1
   - **Block all public access:** Keep checked
4. Click **Create bucket**
5. Repeat to create artifact bucket:
   - **Bucket name:** `codepipeline-artifacts-[account-id]`

#### Step 12: Upload Sample Data to S3
1. Create a local file `sales-input.csv`:
```csv
order_id,customer_name,product,amount,order_date
1001,John Doe,Laptop,1200.00,2024-01-15
1002,Jane Smith,Mouse,25.50,2024-01-15
1003,Bob Johnson,Keyboard,75.00,2024-01-16
1004,Alice Williams,Monitor,350.00,2024-01-16
1005,Charlie Brown,Headset,89.99,2024-01-17
```
2. Go to S3 bucket `sales-data-source-[account-id]`
3. Create folder: `sales/`
4. Upload `sales-input.csv` to `sales/` folder

#### Step 13: Create IAM Role for Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select **AWS service** → **Lambda**
4. Click **Next**
5. Attach policies:
   - `AWSLambdaBasicExecutionRole`
   - `AmazonS3ReadOnlyAccess`
   - `AmazonDynamoDBFullAccess`
6. Click **Next**
7. **Role name:** `lambda-etl-execution-role`
8. Click **Create role**

#### Step 14: Create Lambda Function
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `sales-etl-function`
   - **Runtime:** Python 3.11
   - **Architecture:** x86_64
   - **Execution role:** Use existing role → Select `lambda-etl-execution-role`
4. Click **Create function**

#### Step 15: Configure Lambda Alias
1. In the Lambda function, click **Aliases** in left sidebar
2. Click **Create alias**
3. Configure:
   - **Name:** `live`
   - **Version:** $LATEST
   - **Description:** `Production alias for deployment`
4. Click **Save**

---

### Part 3: Set Up CodeBuild Project

#### Step 16: Create IAM Role for CodeBuild
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select **AWS service** → **CodeBuild**
4. Attach policies:
   - `AmazonS3FullAccess`
   - `CloudWatchLogsFullAccess`
5. Click **Next**
6. **Role name:** `codebuild-service-role`
7. Click **Create role**

#### Step 17: Create CodeBuild Project
1. Navigate to **CodeBuild**
2. Click **Create project**
3. **Project configuration:**
   - **Project name:** `lambda-etl-build`
   - **Description:** `Build and test Lambda ETL function`
4. **Source:**
   - **Source provider:** AWS CodeCommit
   - **Repository:** `lambda-etl-pipeline`
   - **Branch:** `main`
   - **Git clone depth:** 1
5. **Environment:**
   - **Environment image:** Managed image
   - **Operating system:** Amazon Linux 2
   - **Runtime(s):** Standard
   - **Image:** aws/codebuild/amazonlinux2-x86_64-standard:5.0
   - **Environment type:** Linux
   - **Service role:** Existing service role → `codebuild-service-role`
   - **Timeout:** 15 minutes
   - **Compute:** 3 GB memory, 2 vCPUs
6. **Buildspec:**
   - **Build specifications:** Use a buildspec file
   - **Buildspec name:** `buildspec.yml`
7. **Artifacts:**
   - **Type:** Amazon S3
   - **Bucket name:** `codepipeline-artifacts-[account-id]`
   - **Name:** `lambda-build-output`
   - **Artifacts packaging:** Zip
8. **Logs:**
   - **CloudWatch logs:** Enabled
   - **Group name:** `/aws/codebuild/lambda-etl-build`
9. Click **Create build project**

#### Step 18: Test CodeBuild
1. On the project page, click **Start build**
2. Leave defaults and click **Start build**
3. Monitor the build phases in real-time:
   - Install
   - Pre-build (tests)
   - Build (package)
   - Post-build
4. Verify all tests pass
5. Check **Build logs** for coverage report
6. Download artifacts from S3 to verify

---

### Part 4: Configure CodeDeploy

#### Step 19: Create IAM Role for CodeDeploy
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select **AWS service** → **CodeDeploy**
4. Under **Use case**, select **CodeDeploy - Lambda**
5. Click **Next** (policy is auto-attached)
6. **Role name:** `codedeploy-lambda-role`
7. Click **Create role**

#### Step 20: Create CodeDeploy Application
1. Navigate to **CodeDeploy**
2. Click **Create application**
3. Configure:
   - **Application name:** `lambda-etl-app`
   - **Compute platform:** AWS Lambda
4. Click **Create application**

#### Step 21: Create Deployment Group
1. Inside the application, click **Create deployment group**
2. Configure:
   - **Deployment group name:** `lambda-etl-deployment-group`
   - **Service role:** `codedeploy-lambda-role`
3. **Deployment type:**
   - Select **Canary**
   - **Canary percentage:** 10% every 5 minutes
4. **Environment configuration:**
   - **Lambda function:** `sales-etl-function`
   - **Alias:** `live`
5. **Deployment settings:**
   - **Deployment configuration:** CodeDeployDefault.LambdaCanary10Percent5Minutes
6. **Advanced (optional):**
   - **Automatic rollback:** Enable
   - **Rollback when:** A deployment fails or CloudWatch alarm threshold is met
7. Click **Create deployment group**

#### Step 22: Create CloudWatch Alarm for Rollback
1. Navigate to **CloudWatch**
2. Click **Alarms** → **Create alarm**
3. Click **Select metric**
4. Choose **Lambda** → **By Function Name**
5. Select `sales-etl-function` → **Errors**
6. Click **Select metric**
7. Configure metric:
   - **Statistic:** Sum
   - **Period:** 1 minute
   - **Threshold type:** Static
   - **Whenever Errors is...:** Greater than 2
8. Click **Next**
9. **Notification:**
   - **Alarm state trigger:** In alarm
   - **Select SNS topic:** Create new topic (optional)
10. **Alarm name:** `lambda-etl-errors-alarm`
11. Click **Next** → **Create alarm**

---

### Part 5: Create CodePipeline

#### Step 23: Create Pipeline
1. Navigate to **CodePipeline**
2. Click **Create pipeline**
3. **Pipeline settings:**
   - **Pipeline name:** `lambda-etl-pipeline`
   - **Service role:** New service role (auto-created)
   - **Role name:** `AWSCodePipelineServiceRole-lambda-etl`
   - **Advanced settings:**
     - **Artifact store:** Default location
     - **Encryption key:** Default AWS Managed Key
4. Click **Next**

#### Step 24: Add Source Stage
1. **Source provider:** AWS CodeCommit
2. **Repository name:** `lambda-etl-pipeline`
3. **Branch name:** `main`
4. **Change detection:** Amazon CloudWatch Events (recommended)
5. **Output artifact format:** CodePipeline default
6. Click **Next**

#### Step 25: Add Build Stage
1. **Build provider:** AWS CodeBuild
2. **Region:** us-east-1
3. **Project name:** `lambda-etl-build`
4. **Build type:** Single build
5. Click **Next**

#### Step 26: Add Deploy Stage
1. **Deploy provider:** AWS CodeDeploy
2. **Region:** us-east-1
3. **Application name:** `lambda-etl-app`
4. **Deployment group:** `lambda-etl-deployment-group`
5. Click **Next**

#### Step 27: Review and Create
1. Review all stages
2. Click **Create pipeline**
3. Pipeline will automatically start execution

#### Step 28: Monitor Pipeline Execution
1. Watch the pipeline progress through stages:
   - Source (CodeCommit) - ~10 seconds
   - Build (CodeBuild) - ~2-3 minutes
   - Deploy (CodeDeploy) - ~10 minutes for canary
2. Click on each stage to see details
3. Monitor the canary deployment:
   - 10% traffic → Wait 5 minutes → Monitor errors
   - If successful → 100% traffic
   - If errors > threshold → Automatic rollback

---

### Part 6: Test Deployment and Rollback

#### Step 29: Test Successful Deployment
1. Go to **Lambda** → Functions → `sales-etl-function`
2. Click **Test** tab
3. Create test event:
```json
{
  "source_bucket": "sales-data-source-YOUR-ACCOUNT-ID",
  "source_key": "sales/sales-input.csv",
  "table_name": "ProcessedSales"
}
```
4. Click **Test**
5. Verify successful execution
6. Go to **DynamoDB** → Tables → `ProcessedSales` → **Explore table items**
7. Verify records were loaded

#### Step 30: Simulate Failure for Rollback
1. In your local repository, modify `lambda_function.py` to introduce an error:
```python
def lambda_handler(event, context):
    # Intentional error for rollback testing
    raise Exception("Simulated deployment failure")
    # ... rest of code
```
2. Commit and push:
```bash
git add lambda_function.py
git commit -m "Test: Introduce error for rollback simulation"
git push origin main
```
3. Watch the pipeline execute
4. During canary deployment, the errors will trigger the alarm
5. CodeDeploy will automatically roll back to the previous version

#### Step 31: Verify Automatic Rollback
1. Go to **CodeDeploy** → Applications → `lambda-etl-app`
2. Click **Deployments**
3. Find the failed deployment
4. Status should show: "Failed" or "Stopped"
5. Click on deployment ID to see details
6. Verify rollback occurred
7. Test Lambda function again - it should work (previous version)

#### Step 32: Fix and Redeploy
1. Remove the error from `lambda_function.py`
2. Commit and push:
```bash
git add lambda_function.py
git commit -m "Fix: Remove simulated error"
git push origin main
```
3. Watch successful deployment

---

### Verification Checklist

- [ ] CodeCommit repository created with ETL code
- [ ] Unit tests passing with >80% coverage
- [ ] CodeBuild project building successfully
- [ ] DynamoDB table created
- [ ] S3 buckets created and configured
- [ ] Lambda function deployed with alias
- [ ] CodeDeploy application and deployment group configured
- [ ] CloudWatch alarm created for errors
- [ ] Pipeline executes all stages successfully
- [ ] Canary deployment completes (10% → 100%)
- [ ] Automatic rollback triggered on errors
- [ ] ETL function processes data correctly
- [ ] Records appear in DynamoDB

### Architecture Benefits

**Automation:**
- Zero-touch deployments after code commit
- Automated testing prevents bad code from reaching production
- Self-healing with automatic rollbacks

**Safety:**
- Canary deployments minimize blast radius
- Automatic rollback based on CloudWatch metrics
- Version control and audit trail

**Speed:**
- Full deployment cycle: 15-20 minutes
- Fast feedback loop for developers
- Parallel test execution

**Cost Efficiency:**
- CodeCommit: Free (up to 5 users)
- CodeBuild: $0.005/build minute = ~$0.15 per build
- CodeDeploy: Free for Lambda
- CodePipeline: $1/pipeline/month
- **Total:** ~$5-8/month for active development

---

### Cleanup

1. **Delete Pipeline:**
   - CodePipeline → Select pipeline → Delete

2. **Delete CodeDeploy:**
   - CodeDeploy → Applications → Delete application

3. **Delete CodeBuild Project:**
   - CodeBuild → Build projects → Delete

4. **Delete Lambda Function:**
   - Lambda → Functions → Delete function

5. **Delete DynamoDB Table:**
   - DynamoDB → Tables → Delete table

6. **Delete S3 Buckets:**
   - S3 → Empty then delete both buckets

7. **Delete CodeCommit Repository:**
   - CodeCommit → Repositories → Delete repository

8. **Delete CloudWatch Alarm:**
   - CloudWatch → Alarms → Delete alarm

9. **Delete IAM Roles:**
   - IAM → Roles → Delete all created roles

---

## Exercise 13.2: X-Ray Distributed Tracing

### Introduction
Implement AWS X-Ray distributed tracing for Lambda-based data pipelines to visualize service dependencies, analyze performance bottlenecks, and debug distributed applications.

**What You'll Build:**
- Lambda functions instrumented with X-Ray SDK
- DynamoDB table for pipeline state tracking
- Step Functions workflow with X-Ray tracing
- Service maps and trace analysis
- Custom annotations and metadata

**Duration:** 90 minutes  
**Cost:** $5/month (100K traces)

---

### Part 1: Create Lambda Functions with X-Ray

#### Step 1: Create IAM Role for X-Ray Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select **AWS service** → **Lambda**
4. Attach policies:
   - `AWSLambdaBasicExecutionRole`
   - `AWSXRayDaemonWriteAccess`
   - `AmazonDynamoDBFullAccess`
   - `AmazonS3ReadOnlyAccess`
5. **Role name:** `lambda-xray-execution-role`
6. Click **Create role**

#### Step 2: Create DynamoDB Table for Pipeline State
1. Navigate to **DynamoDB**
2. Click **Create table**
3. Configure:
   - **Table name:** `PipelineState`
   - **Partition key:** `pipeline_id` (String)
   - **Sort key:** `timestamp` (Number)
   - **Table settings:** Default settings
4. Click **Create table**

#### Step 3: Create Data Extraction Lambda
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `extract-data-xray`
   - **Runtime:** Python 3.11
   - **Execution role:** Use existing → `lambda-xray-execution-role`
4. Click **Create function**
5. Replace code with:
```python
import json
import boto3
import time
from datetime import datetime
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patch AWS SDK clients
patch_all()

s3 = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

@xray_recorder.capture('extract_data')
def lambda_handler(event, context):
    """Extract data from S3 with X-Ray tracing"""
    
    # Add annotation for filtering traces
    xray_recorder.begin_subsegment('initialization')
    pipeline_id = event.get('pipeline_id', 'pipeline-001')
    xray_recorder.put_annotation('pipeline_id', pipeline_id)
    xray_recorder.put_annotation('stage', 'extract')
    xray_recorder.end_subsegment()
    
    # Add metadata for debugging
    xray_recorder.put_metadata('event', event)
    xray_recorder.put_metadata('pipeline_config', {
        'source_type': 's3',
        'format': 'csv'
    })
    
    try:
        # Simulate data extraction with subsegment
        with xray_recorder.capture('s3_read'):
            source_bucket = event.get('source_bucket', 'sales-data-source')
            source_key = event.get('source_key', 'sales/input.csv')
            
            # Add metadata about the source
            xray_recorder.put_metadata('source', {
                'bucket': source_bucket,
                'key': source_key
            })
            
            # Simulate processing time
            time.sleep(0.5)
            
            response = s3.get_object(Bucket=source_bucket, Key=source_key)
            data = response['Body'].read().decode('utf-8')
            
            lines = data.strip().split('\n')
            record_count = len(lines) - 1  # Exclude header
            
            xray_recorder.put_annotation('record_count', record_count)
        
        # Record state in DynamoDB
        with xray_recorder.capture('dynamodb_write_state'):
            table = dynamodb.Table('PipelineState')
            table.put_item(Item={
                'pipeline_id': pipeline_id,
                'timestamp': int(datetime.now().timestamp()),
                'stage': 'extract',
                'status': 'completed',
                'record_count': record_count,
                'metadata': json.dumps({
                    'source_bucket': source_bucket,
                    'source_key': source_key
                })
            })
        
        return {
            'statusCode': 200,
            'pipeline_id': pipeline_id,
            'stage': 'extract',
            'record_count': record_count,
            'data': data,
            'next_stage': 'transform'
        }
        
    except Exception as e:
        xray_recorder.put_annotation('error', True)
        xray_recorder.put_metadata('error_details', {
            'message': str(e),
            'type': type(e).__name__
        })
        raise

```
6. Click **Deploy**

#### Step 4: Enable X-Ray Tracing on Extract Lambda
1. Click **Configuration** tab
2. Click **Monitoring and operations tools**
3. Click **Edit**
4. Under **AWS X-Ray:**
   - Check **Active tracing**
5. Click **Save**

#### Step 5: Create Data Transform Lambda
1. Create another Lambda function:
   - **Function name:** `transform-data-xray`
   - **Runtime:** Python 3.11
   - **Execution role:** `lambda-xray-execution-role`
2. Add code:
```python
import json
import time
from datetime import datetime
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all
import boto3

patch_all()

dynamodb = boto3.resource('dynamodb')

@xray_recorder.capture('transform_data')
def lambda_handler(event, context):
    """Transform data with X-Ray tracing"""
    
    pipeline_id = event.get('pipeline_id')
    xray_recorder.put_annotation('pipeline_id', pipeline_id)
    xray_recorder.put_annotation('stage', 'transform')
    
    try:
        # Get data from previous stage
        data = event.get('data', '')
        record_count = event.get('record_count', 0)
        
        xray_recorder.put_metadata('input_stats', {
            'record_count': record_count,
            'data_size_bytes': len(data)
        })
        
        # Transform data with subsegments
        with xray_recorder.capture('parse_csv'):
            lines = data.strip().split('\n')
            header = lines[0].split(',')
            records = []
            
            for line in lines[1:]:
                values = line.split(',')
                record = dict(zip(header, values))
                records.append(record)
        
        with xray_recorder.capture('validate_and_enrich'):
            # Simulate validation and enrichment
            time.sleep(0.3)
            
            valid_records = 0
            invalid_records = 0
            
            for record in records:
                if record.get('order_id') and record.get('amount'):
                    valid_records += 1
                    # Add enrichment
                    record['processed_date'] = datetime.now().isoformat()
                    record['status'] = 'validated'
                else:
                    invalid_records += 1
            
            xray_recorder.put_annotation('valid_records', valid_records)
            xray_recorder.put_annotation('invalid_records', invalid_records)
        
        # Record state
        with xray_recorder.capture('dynamodb_write_state'):
            table = dynamodb.Table('PipelineState')
            table.put_item(Item={
                'pipeline_id': pipeline_id,
                'timestamp': int(datetime.now().timestamp()),
                'stage': 'transform',
                'status': 'completed',
                'valid_records': valid_records,
                'invalid_records': invalid_records
            })
        
        return {
            'statusCode': 200,
            'pipeline_id': pipeline_id,
            'stage': 'transform',
            'transformed_records': records,
            'valid_count': valid_records,
            'invalid_count': invalid_records,
            'next_stage': 'load'
        }
        
    except Exception as e:
        xray_recorder.put_annotation('error', True)
        xray_recorder.put_metadata('error_details', str(e))
        raise
```
3. Click **Deploy**
4. Enable X-Ray active tracing (Configuration → Monitoring tools)

#### Step 6: Create Data Load Lambda
1. Create another Lambda function:
   - **Function name:** `load-data-xray`
   - **Runtime:** Python 3.11
   - **Execution role:** `lambda-xray-execution-role`
2. Add code:
```python
import json
import time
from datetime import datetime
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all
import boto3

patch_all()

dynamodb = boto3.resource('dynamodb')

@xray_recorder.capture('load_data')
def lambda_handler(event, context):
    """Load data into DynamoDB with X-Ray tracing"""
    
    pipeline_id = event.get('pipeline_id')
    xray_recorder.put_annotation('pipeline_id', pipeline_id)
    xray_recorder.put_annotation('stage', 'load')
    
    try:
        transformed_records = event.get('transformed_records', [])
        
        xray_recorder.put_metadata('load_config', {
            'destination': 'ProcessedSales',
            'record_count': len(transformed_records)
        })
        
        # Load data with batch write
        with xray_recorder.capture('dynamodb_batch_write'):
            table = dynamodb.Table('ProcessedSales')
            
            # Simulate batch writing
            time.sleep(0.4)
            
            with table.batch_writer() as batch:
                for record in transformed_records:
                    if record.get('order_id'):
                        batch.put_item(Item=record)
            
            xray_recorder.put_annotation('records_loaded', len(transformed_records))
        
        # Record final state
        with xray_recorder.capture('dynamodb_write_state'):
            state_table = dynamodb.Table('PipelineState')
            state_table.put_item(Item={
                'pipeline_id': pipeline_id,
                'timestamp': int(datetime.now().timestamp()),
                'stage': 'load',
                'status': 'completed',
                'records_loaded': len(transformed_records)
            })
        
        return {
            'statusCode': 200,
            'pipeline_id': pipeline_id,
            'stage': 'load',
            'records_loaded': len(transformed_records),
            'status': 'pipeline_completed'
        }
        
    except Exception as e:
        xray_recorder.put_annotation('error', True)
        xray_recorder.put_metadata('error_details', str(e))
        raise
```
3. Click **Deploy**
4. Enable X-Ray active tracing

#### Step 7: Add X-Ray SDK Layer to All Functions
1. For each function (`extract-data-xray`, `transform-data-xray`, `load-data-xray`):
2. Scroll to **Layers** section
3. Click **Add a layer**
4. Select **AWS layers**
5. Choose **AWSXRay-PythonSDK**
6. Select latest version
7. Click **Add**

---

### Part 2: Create Step Functions Workflow with X-Ray

#### Step 8: Create IAM Role for Step Functions
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select **AWS service** → **Step Functions**
4. Attach policies:
   - `AWSLambdaRole`
   - `AWSXRayDaemonWriteAccess`
5. **Role name:** `step-functions-xray-role`
6. Click **Create role**

#### Step 9: Create Step Functions State Machine
1. Navigate to **Step Functions**
2. Click **Create state machine**
3. Choose **Write your workflow in code**
4. **Type:** Standard
5. **Definition:** Paste this ASL (Amazon States Language):
```json
{
  "Comment": "ETL Pipeline with X-Ray Tracing",
  "StartAt": "Extract",
  "States": {
    "Extract": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:extract-data-xray",
      "ResultPath": "$.extractResult",
      "Next": "Transform",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "HandleError"
        }
      ]
    },
    "Transform": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:transform-data-xray",
      "InputPath": "$.extractResult",
      "ResultPath": "$.transformResult",
      "Next": "Load",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "HandleError"
        }
      ]
    },
    "Load": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:load-data-xray",
      "InputPath": "$.transformResult",
      "ResultPath": "$.loadResult",
      "Next": "Success"
    },
    "Success": {
      "Type": "Succeed"
    },
    "HandleError": {
      "Type": "Fail",
      "Error": "PipelineError",
      "Cause": "ETL pipeline failed"
    }
  }
}
```
6. **Replace** `ACCOUNT_ID` with your AWS account ID
7. Click **Next**

#### Step 10: Configure State Machine Settings
1. **Name:** `etl-pipeline-xray`
2. **Execution role:** Select `step-functions-xray-role`
3. **Logging:**
   - **Log level:** ALL
   - **Include execution data:** Checked
4. **Tracing:**
   - **Enable X-Ray tracing:** Checked
5. Click **Create state machine**

---

### Part 3: Execute and Analyze Traces

#### Step 11: Execute Step Functions Workflow
1. On the state machine page, click **Start execution**
2. **Execution name:** `test-execution-1`
3. **Input:** Paste:
```json
{
  "pipeline_id": "pipeline-001",
  "source_bucket": "sales-data-source-YOUR-ACCOUNT-ID",
  "source_key": "sales/sales-input.csv"
}
```
4. Click **Start execution**
5. Watch the workflow execute through all stages
6. Verify it completes successfully

#### Step 12: Access X-Ray Service Map
1. Navigate to **CloudWatch** → **X-Ray traces** → **Service map**
2. You should see:
   - Step Functions state machine
   - Three Lambda functions
   - DynamoDB tables
   - S3 bucket
3. Observe the connections and dependencies
4. Click on each node to see statistics:
   - Average latency
   - Request count
   - Error rate

#### Step 13: Analyze Individual Traces
1. Go to **CloudWatch** → **X-Ray traces** → **Traces**
2. You'll see a list of traces
3. **Filter traces:**
   - Add filter: `annotation.pipeline_id = "pipeline-001"`
   - Or: `annotation.stage = "extract"`
4. Click on a trace to see the timeline
5. Examine:
   - Total duration
   - Time spent in each segment
   - Subsegment breakdown
   - Annotations and metadata

#### Step 14: Identify Performance Bottlenecks
1. In the trace details, look for:
   - Longest duration segments (bottlenecks)
   - High latency subsegments
   - DynamoDB or S3 call times
2. Example analysis:
   - If `s3_read` takes 80% of extract time → Consider S3 optimization
   - If `dynamodb_batch_write` is slow → Check provisioned capacity
   - If `validate_and_enrich` is slow → Optimize transformation logic

#### Step 15: Create Custom Trace Queries
1. Go to **X-Ray** → **Traces**
2. Use filter expressions:
```
annotation.stage = "transform" AND annotation.valid_records > 100
```
3. Or find errors:
```
annotation.error = true
```
4. Or find slow requests:
```
responsetime > 3
```
5. Save useful queries for monitoring

#### Step 16: Analyze Service Map Metrics
1. In the Service map, click on Lambda function node
2. View **Response time distribution**:
   - p50 (median)
   - p90
   - p99
3. View **Error rate** over time
4. Identify patterns or anomalies

---

### Part 4: Debug Performance Issues

#### Step 17: Simulate Performance Problem
1. Modify `transform-data-xray` Lambda to add delay:
```python
# Add this in the validate_and_enrich subsegment
with xray_recorder.capture('validate_and_enrich'):
    # Simulate slow validation
    time.sleep(2)  # Increased from 0.3
    # ... rest of code
```
2. Deploy the change

#### Step 18: Execute and Compare Traces
1. Run the Step Functions workflow again
2. Go to X-Ray traces
3. Compare:
   - Old trace (fast) vs new trace (slow)
   - Identify which subsegment increased in duration
4. The `validate_and_enrich` subsegment should show ~2 seconds
5. Use this to pinpoint the exact code causing slowness

#### Step 19: View Trace Timeline
1. Open a slow trace
2. Click on the timeline view
3. See visual representation:
   - Extract: 0-0.5s
   - Transform: 0.5-3s (2.5s total, 2s in validate_and_enrich)
   - Load: 3-3.5s
4. This clearly shows transform is the bottleneck

#### Step 20: Fix and Verify
1. Remove the artificial delay in transform Lambda
2. Deploy
3. Execute workflow again
4. Verify trace shows normal performance

---

### Verification Checklist

- [ ] Three Lambda functions created with X-Ray SDK
- [ ] X-Ray active tracing enabled on all functions
- [ ] DynamoDB PipelineState table created
- [ ] Step Functions state machine with X-Ray tracing
- [ ] Service map shows all components
- [ ] Traces captured for all executions
- [ ] Annotations visible in trace filters
- [ ] Metadata visible in trace details
- [ ] Subsegments showing detailed timing
- [ ] Performance bottleneck identified
- [ ] Custom trace queries working
- [ ] Error traces distinguishable

### Performance Insights

**Trace Analysis Benefits:**
- **Visibility:** See exact time spent in each service
- **Debugging:** Identify which component failed
- **Optimization:** Find bottlenecks with millisecond precision
- **Dependency Mapping:** Understand service relationships

**Typical Timings (Well-Optimized Pipeline):**
- Extract: 200-500ms
- Transform: 300-600ms
- Load: 400-800ms
- Total: 900-1900ms

**Cost:**
- First 100,000 traces/month: Free
- Additional traces: $5 per 1 million
- Typical cost: $5-10/month

---

### Cleanup

1. **Delete Step Functions State Machine:**
   - Step Functions → State machines → Delete

2. **Delete Lambda Functions:**
   - Lambda → Delete all three functions

3. **Delete DynamoDB Tables:**
   - DynamoDB → Delete PipelineState and ProcessedSales

4. **Delete IAM Roles:**
   - IAM → Roles → Delete created roles

---

## Exercise 13.3: CodeGuru Code Quality and Profiling

### Introduction
Use AWS CodeGuru to automatically review code for best practices, security vulnerabilities, and performance issues. Profile Lambda functions to identify CPU and memory bottlenecks.

**What You'll Build:**
- CodeCommit repository with CodeGuru Reviewer enabled
- Pull request with code issues
- CodeGuru Profiler for Lambda functions
- Performance analysis and recommendations

**Duration:** 75 minutes  
**Cost:** $5-10/month

---

### Part 1: Enable CodeGuru Reviewer

#### Step 1: Create New CodeCommit Repository
1. Navigate to **CodeCommit**
2. Click **Create repository**
3. Configure:
   - **Repository name:** `data-pipeline-quality`
   - **Description:** `Data pipeline with CodeGuru analysis`
4. Click **Create**

#### Step 2: Enable CodeGuru Reviewer
1. Navigate to **CodeGuru** → **Reviewer**
2. Click **Associate repository**
3. Select **AWS CodeCommit**
4. **Repository:** Select `data-pipeline-quality`
5. Click **Associate**
6. Wait for status to show "Associated"

#### Step 3: Clone and Set Up Repository
1. Clone the repository locally:
```bash
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/data-pipeline-quality
cd data-pipeline-quality
```

#### Step 4: Create Initial Code with Issues
1. Create `data_processor.py` with intentional issues:
```python
import boto3
import json
import time

# Issue 1: Hardcoded credentials (security vulnerability)
AWS_ACCESS_KEY = "AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_KEY = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# Issue 2: Client created outside handler (resource leak)
s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

def process_data(event, context):
    """Process data from S3 and load to DynamoDB"""
    
    # Issue 3: No error handling
    bucket = event['bucket']
    key = event['key']
    table_name = event['table']
    
    # Issue 4: Synchronous operations in loop (performance)
    data = s3_client.get_object(Bucket=bucket, Key=key)
    content = data['Body'].read().decode('utf-8')
    lines = content.split('\n')
    
    table = dynamodb.Table(table_name)
    
    # Issue 5: No batch operations (inefficient)
    for line in lines:
        if line.strip():
            parts = line.split(',')
            item = {
                'id': parts[0],
                'value': parts[1]
            }
            # Individual put_item calls are slow
            table.put_item(Item=item)
            time.sleep(0.1)  # Issue 6: Unnecessary sleep
    
    # Issue 7: No validation of array indices
    # Issue 8: No logging
    # Issue 9: Magic numbers without constants
    
    return {
        'statusCode': 200,
        'body': 'Success'
    }

# Issue 10: Resource-intensive operation in module scope
def expensive_initialization():
    result = []
    for i in range(1000000):  # Million iterations
        result.append(i * i)
    return result

# This runs on every cold start
cache = expensive_initialization()

# Issue 11: SQL injection vulnerability (if using RDS)
def query_database(user_input):
    import pymysql
    conn = pymysql.connect(host='localhost', user='root', password='password')
    cursor = conn.cursor()
    
    # SQL injection risk
    query = f"SELECT * FROM users WHERE name = '{user_input}'"
    cursor.execute(query)
    
    return cursor.fetchall()

# Issue 12: Inefficient string concatenation
def build_message(items):
    message = ""
    for item in items:
        message = message + str(item) + ","  # Use join() instead
    return message

# Issue 13: Missing connection cleanup
def read_from_db():
    import pymysql
    conn = pymysql.connect(host='localhost')
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM data")
    return cursor.fetchall()
    # Connection never closed!

# Issue 14: Inefficient list operations
def process_large_list(data):
    result = []
    for item in data:
        if item not in result:  # O(n) lookup on list
            result.append(item)
    return result

# Issue 15: Memory leak with global state
global_cache = {}

def cache_data(key, value):
    # Cache grows indefinitely
    global_cache[key] = value
```

2. Commit to main branch:
```bash
git add data_processor.py
git commit -m "Initial commit: Data processor with various issues"
git push origin main
```

---

### Part 2: Create Pull Request for Review

#### Step 5: Create Feature Branch with "Improved" Code
1. Create and checkout new branch:
```bash
git checkout -b feature/improved-processor
```

2. Create improved version `data_processor_v2.py`:
```python
import boto3
import json
import logging
from typing import Dict, List, Any
from botocore.exceptions import ClientError

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

class DataProcessor:
    """Process data from S3 to DynamoDB with best practices"""
    
    def __init__(self):
        # Initialize clients inside class (lazy loading)
        self._s3_client = None
        self._dynamodb = None
    
    @property
    def s3_client(self):
        """Lazy initialization of S3 client"""
        if self._s3_client is None:
            self._s3_client = boto3.client('s3')
        return self._s3_client
    
    @property
    def dynamodb(self):
        """Lazy initialization of DynamoDB resource"""
        if self._dynamodb is None:
            self._dynamodb = boto3.resource('dynamodb')
        return self._dynamodb
    
    def process_data(self, event: Dict[str, Any], context: Any) -> Dict[str, Any]:
        """
        Process data from S3 and load to DynamoDB
        
        Args:
            event: Lambda event containing bucket, key, and table
            context: Lambda context
            
        Returns:
            Response dictionary with status
        """
        try:
            # Validate input
            bucket = event.get('bucket')
            key = event.get('key')
            table_name = event.get('table')
            
            if not all([bucket, key, table_name]):
                raise ValueError("Missing required parameters: bucket, key, or table")
            
            logger.info(f"Processing s3://{bucket}/{key}")
            
            # Read data from S3
            data = self._read_from_s3(bucket, key)
            
            # Parse and validate
            items = self._parse_csv_data(data)
            
            # Batch write to DynamoDB
            records_written = self._batch_write_dynamodb(table_name, items)
            
            logger.info(f"Successfully processed {records_written} records")
            
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'message': 'Success',
                    'records_processed': records_written
                })
            }
            
        except ClientError as e:
            logger.error(f"AWS error: {e.response['Error']['Message']}")
            return {
                'statusCode': 500,
                'body': json.dumps({'error': 'AWS service error'})
            }
        except ValueError as e:
            logger.error(f"Validation error: {str(e)}")
            return {
                'statusCode': 400,
                'body': json.dumps({'error': str(e)})
            }
        except Exception as e:
            logger.error(f"Unexpected error: {str(e)}")
            return {
                'statusCode': 500,
                'body': json.dumps({'error': 'Internal server error'})
            }
    
    def _read_from_s3(self, bucket: str, key: str) -> str:
        """Read object from S3"""
        response = self.s3_client.get_object(Bucket=bucket, Key=key)
        return response['Body'].read().decode('utf-8')
    
    def _parse_csv_data(self, data: str) -> List[Dict[str, str]]:
        """Parse CSV data and validate"""
        items = []
        lines = data.strip().split('\n')
        
        for idx, line in enumerate(lines):
            if not line.strip():
                continue
                
            parts = line.split(',')
            
            # Validate minimum fields
            if len(parts) < 2:
                logger.warning(f"Skipping line {idx}: insufficient fields")
                continue
            
            items.append({
                'id': parts[0].strip(),
                'value': parts[1].strip()
            })
        
        return items
    
    def _batch_write_dynamodb(self, table_name: str, items: List[Dict]) -> int:
        """Batch write items to DynamoDB"""
        if not items:
            return 0
        
        table = self.dynamodb.Table(table_name)
        
        # Use batch writer for efficiency
        with table.batch_writer() as batch:
            for item in items:
                batch.put_item(Item=item)
        
        return len(items)

# Lambda handler
processor = DataProcessor()

def lambda_handler(event, context):
    """Lambda entry point"""
    return processor.process_data(event, context)
```

3. Commit and push:
```bash
git add data_processor_v2.py
git commit -m "Improved data processor with better error handling and batch operations"
git push origin feature/improved-processor
```

#### Step 6: Create Pull Request in CodeCommit
1. Go to **CodeCommit** → **Repositories** → `data-pipeline-quality`
2. Click **Pull requests** → **Create pull request**
3. Configure:
   - **Source:** `feature/improved-processor`
   - **Destination:** `main`
   - **Title:** `Improve data processor with best practices`
   - **Description:** `
     - Add proper error handling
     - Implement batch operations for DynamoDB
     - Add input validation
     - Remove hardcoded credentials
     - Add logging
     - Use lazy initialization for clients
     `
4. Click **Create pull request**

#### Step 7: Wait for CodeGuru Review
1. CodeGuru Reviewer will automatically analyze the pull request
2. Wait 5-10 minutes for analysis to complete
3. Refresh the pull request page
4. You'll see **CodeGuru Reviewer** tab appear

#### Step 8: Review CodeGuru Recommendations
1. Click **CodeGuru Reviewer** tab
2. Review recommendations, which may include:
   - **Security:** Remove hardcoded credentials from old file
   - **Performance:** Batch operations are good improvement
   - **Best Practices:** Error handling improvements
   - **Resource Leaks:** Connection management
   - **Code Quality:** Type hints are good practice
3. Click on each recommendation to see:
   - Severity (Critical, High, Medium, Low, Info)
   - Detailed explanation
   - Code snippet
   - Suggested fix
   - Learn more links

#### Step 9: Address CodeGuru Feedback
1. For each recommendation, either:
   - **Fix:** Make the suggested change
   - **Comment:** Explain why it's not applicable
   - **Acknowledge:** Mark as reviewed
2. Example fixes:
```bash
# Remove the old file with security issues
git rm data_processor.py
git commit -m "Remove old processor with security issues per CodeGuru"
git push origin feature/improved-processor
```

#### Step 10: Merge Pull Request
1. After addressing feedback, click **Merge**
2. Select merge strategy: **Fast forward merge**
3. Click **Merge pull request**
4. Delete the feature branch (optional)

---

### Part 3: Enable CodeGuru Profiler for Lambda

#### Step 11: Create Lambda Function for Profiling
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `data-processor-profiled`
   - **Runtime:** Python 3.11
   - **Execution role:** Create new role with basic Lambda permissions
4. Click **Create function**

#### Step 12: Add Code with Performance Issues
1. Replace Lambda code with:
```python
import json
import time
import random

def lambda_handler(event, context):
    """
    Data processing function with performance issues
    for CodeGuru Profiler demonstration
    """
    
    # Simulate data processing workload
    result = {
        'processed_records': 0,
        'processing_time': 0
    }
    
    start_time = time.time()
    
    # CPU-intensive operation
    result['cpu_work'] = cpu_intensive_work(1000)
    
    # Memory-intensive operation
    result['memory_work'] = memory_intensive_work(10000)
    
    # Inefficient string operations
    result['string_work'] = inefficient_string_ops(500)
    
    # Nested loops (O(n²))
    result['nested_loops'] = nested_loop_operation(100)
    
    result['processing_time'] = time.time() - start_time
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }

def cpu_intensive_work(iterations):
    """Simulate CPU-intensive work"""
    total = 0
    for i in range(iterations):
        for j in range(1000):
            total += (i * j) ** 0.5
    return total

def memory_intensive_work(size):
    """Simulate memory-intensive work"""
    # Create large lists
    data = []
    for i in range(size):
        data.append([random.random() for _ in range(100)])
    
    # Sort them (CPU + memory)
    sorted_data = sorted(data, key=lambda x: sum(x))
    
    return len(sorted_data)

def inefficient_string_ops(count):
    """Inefficient string concatenation"""
    result = ""
    for i in range(count):
        result += f"Record {i}: {random.random()}\n"
    return len(result)

def nested_loop_operation(size):
    """O(n²) nested loop"""
    result = []
    for i in range(size):
        for j in range(size):
            if i != j:
                result.append(i * j)
    return len(result)
```
2. Click **Deploy**

#### Step 13: Create CodeGuru Profiler Group
1. Navigate to **CodeGuru** → **Profiler**
2. Click **Create profiling group**
3. Configure:
   - **Name:** `data-processor-profiling-group`
   - **Compute platform:** AWS Lambda
   - **Lambda functions:** Select `data-processor-profiled`
4. Click **Create**

#### Step 14: Update Lambda IAM Role
1. Go to **IAM** → **Roles**
2. Find the role for `data-processor-profiled`
3. Click **Add permissions** → **Attach policies**
4. Search for and attach: `AmazonCodeGuruProfilerAgentAccess`
5. Click **Add permissions**

#### Step 15: Enable Profiler in Lambda
1. Go back to **Lambda** → `data-processor-profiled`
2. Click **Configuration** → **Monitoring and operations tools**
3. Click **Edit**
4. Under **Code profiling:**
   - Toggle **Code profiling** to ON
   - **Profiling group:** Select `data-processor-profiling-group`
5. Click **Save**

#### Step 16: Add CodeGuru Profiler Layer
1. Scroll to **Layers** section
2. Click **Add a layer**
3. Select **AWS layers**
4. Choose **AWSCodeGuruProfilerPythonAgentLambdaLayer**
5. Select latest version
6. Click **Add**

#### Step 17: Update Lambda Code to Initialize Profiler
1. Update the Lambda function code:
```python
import json
import time
import random

# CodeGuru Profiler initialization
from codeguru_profiler_agent import Profiler

# Initialize profiler
Profiler(profiling_group_name='data-processor-profiling-group').start()

def lambda_handler(event, context):
    """Data processing with CodeGuru Profiler"""
    
    result = {
        'processed_records': 0,
        'processing_time': 0
    }
    
    start_time = time.time()
    
    # CPU-intensive operation
    result['cpu_work'] = cpu_intensive_work(1000)
    
    # Memory-intensive operation  
    result['memory_work'] = memory_intensive_work(10000)
    
    # Inefficient string operations
    result['string_work'] = inefficient_string_ops(500)
    
    # Nested loops
    result['nested_loops'] = nested_loop_operation(100)
    
    result['processing_time'] = time.time() - start_time
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }

# ... rest of functions remain the same ...
```
2. Click **Deploy**

#### Step 18: Generate Profiling Data
1. Click **Test** tab
2. Create test event:
```json
{
  "test": "profiling-run"
}
```
3. Click **Test** multiple times (at least 10-15 invocations)
4. Each invocation generates profiling data
5. Wait 5-10 minutes for data to aggregate

---

### Part 4: Analyze Profiling Results

#### Step 19: View CPU Profile
1. Navigate to **CodeGuru** → **Profiler** → **Profiling groups**
2. Click on `data-processor-profiling-group`
3. Click **Flame graph** tab
4. You'll see a flame graph showing:
   - Width = CPU time spent
   - Height = call stack depth
   - Color-coded by function
5. Identify the widest sections - these are CPU hotspots

#### Step 20: Analyze Hotspots
1. Click **Hotspots** tab
2. You'll see a ranked list of functions by CPU usage:
```
Function                        CPU Time    % of Total
cpu_intensive_work()           2,345 ms    45%
memory_intensive_work()        1,234 ms    24%
nested_loop_operation()        789 ms      15%
inefficient_string_ops()       456 ms      9%
lambda_handler()               176 ms      3%
```
3. Click on `cpu_intensive_work` to see:
   - Time spent
   - Call frequency
   - Callers
   - Recommendation: "Consider optimizing nested loops"

#### Step 21: View Memory Profile
1. Click **Overview** tab
2. Scroll to **Memory usage over time**
3. Observe memory spikes during `memory_intensive_work`
4. View heap summary showing:
   - Peak memory usage
   - Objects allocated
   - Memory per object type

#### Step 22: Review Recommendations
1. Click **Recommendations** tab
2. CodeGuru will provide insights like:
   - **Recommendation 1:** "Function cpu_intensive_work() uses 45% of CPU time. Consider caching results or reducing iterations."
   - **Recommendation 2:** "Function inefficient_string_ops() creates many string objects. Use list append + join() instead."
   - **Recommendation 3:** "nested_loop_operation() has O(n²) complexity. Consider algorithm optimization."
   - **Recommendation 4:** "memory_intensive_work() creates large temporary lists. Consider generators or streaming."

#### Step 23: Implement Optimizations
1. Update Lambda code with optimizations:
```python
import json
import time
import random
from functools import lru_cache

from codeguru_profiler_agent import Profiler
Profiler(profiling_group_name='data-processor-profiling-group').start()

def lambda_handler(event, context):
    """Optimized data processing"""
    
    result = {}
    start_time = time.time()
    
    # Optimized operations
    result['cpu_work'] = cpu_intensive_work_optimized(1000)
    result['memory_work'] = memory_intensive_work_optimized(10000)
    result['string_work'] = efficient_string_ops(500)
    result['nested_loops'] = optimized_operation(100)
    
    result['processing_time'] = time.time() - start_time
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }

# Optimization 1: Add caching
@lru_cache(maxsize=128)
def cpu_intensive_work_optimized(iterations):
    """Cached CPU work"""
    total = 0
    for i in range(iterations):
        for j in range(1000):
            total += (i * j) ** 0.5
    return total

# Optimization 2: Use generators
def memory_intensive_work_optimized(size):
    """Memory-efficient with generators"""
    # Use generator instead of list
    data_gen = ([random.random() for _ in range(100)] for i in range(size))
    
    # Process in chunks instead of all at once
    count = sum(1 for _ in data_gen)
    
    return count

# Optimization 3: Use join() for strings
def efficient_string_ops(count):
    """Efficient string building"""
    parts = [f"Record {i}: {random.random()}" for i in range(count)]
    result = "\n".join(parts)
    return len(result)

# Optimization 4: Reduce complexity
def optimized_operation(size):
    """Optimized from O(n²) to O(n)"""
    # Use list comprehension with single loop
    result = [i * j for i in range(size) for j in range(i)]
    return len(result)
```
2. Click **Deploy**
3. Run 10-15 more tests
4. Wait for new profiling data

#### Step 24: Compare Before/After Performance
1. In CodeGuru Profiler, select time range
2. Use the timeline to compare:
   - **Before optimization:** ~3-5 seconds per invocation
   - **After optimization:** ~0.5-1 seconds per invocation
3. View side-by-side flame graphs
4. Confirm CPU hotspots reduced

---

### Verification Checklist

- [ ] CodeCommit repository created
- [ ] CodeGuru Reviewer enabled
- [ ] Pull request created with code issues
- [ ] CodeGuru recommendations received
- [ ] Security issues identified
- [ ] Performance issues identified  
- [ ] Code quality improvements suggested
- [ ] Profiling group created
- [ ] Lambda function profiled
- [ ] Flame graph generated
- [ ] CPU hotspots identified
- [ ] Memory usage analyzed
- [ ] Recommendations reviewed
- [ ] Optimizations implemented
- [ ] Performance improvement verified

### ROI Analysis

**Before Developer Tools:**
- Manual code reviews: 2 hours per PR
- Production bugs: 10 per month × 4 hours = 40 hours
- Performance issues: 8 hours/month debugging
- **Total:** 50-60 hours/month of developer time

**After Developer Tools:**
- Automated code review: 10 minutes per PR
- Production bugs: 2 per month × 4 hours = 8 hours (80% reduction)
- Performance issues: 1 hour/month (AI-guided optimization)
- **Total:** 9-10 hours/month

**Savings:**
- Developer time saved: 50 hours/month
- At $100/hour: $5,000/month saved
- Tool cost: $15-20/month
- **ROI: 304× (30,400% return)**

---

### Cleanup

1. **Disable CodeGuru Reviewer:**
   - CodeGuru → Reviewer → Disassociate repository

2. **Delete CodeGuru Profiling Group:**
   - CodeGuru → Profiler → Delete profiling group

3. **Delete Lambda Function:**
   - Lambda → Functions → Delete

4. **Delete CodeCommit Repositories:**
   - CodeCommit → Repositories → Delete both repos

5. **Delete IAM Roles:**
   - IAM → Roles → Delete created roles

---

## Summary

### What You've Learned

**Exercise 13.1 - CI/CD Pipeline:**
- Built complete CI/CD pipeline with AWS native tools
- Implemented automated testing with coverage reporting
- Configured canary deployments for zero-downtime releases
- Set up automatic rollback based on CloudWatch alarms
- Achieved 15-minute deployment cycle

**Exercise 13.2 - X-Ray Distributed Tracing:**
- Instrumented Lambda functions with X-Ray SDK
- Visualized service dependencies with service maps
- Analyzed performance with subsegment granularity
- Used annotations and metadata for debugging
- Identified bottlenecks with millisecond precision

**Exercise 13.3 - CodeGuru Analysis:**
- Automated code review for security and quality
- Identified 15+ types of code issues automatically
- Profiled Lambda CPU and memory usage
- Received AI-powered optimization recommendations
- Achieved 5-10× performance improvement

### Key Takeaways

1. **Automation prevents human error** - CI/CD catches issues before production

2. **Observability is essential** - X-Ray makes invisible problems visible

3. **AI-powered analysis scales** - CodeGuru reviews 100% of code, not just samples

4. **Canary deployments reduce risk** - 10% traffic first, automatic rollback on errors

5. **Profiling guides optimization** - Measure first, optimize hotspots

### Real-World Applications

- **Financial Services:** Automated compliance checks in CI/CD, distributed tracing for transaction flows
- **E-commerce:** Canary deployments during peak shopping, performance profiling for checkout
- **Healthcare:** Code quality for HIPAA compliance, tracing for patient data flows
- **Media Streaming:** Performance optimization with profiling, tracing for content delivery

### Cost Optimization Tips

1. Use CodeBuild caching to speed up builds and reduce costs
2. Enable CodeGuru Reviewer only for main branches
3. Profile Lambda in production with sampling (not 100%)
4. Use X-Ray sampling rules to control trace volume
5. Archive old build artifacts to Glacier

### Next Steps

- Integrate with third-party tools (Jenkins, GitHub Actions)
- Implement Infrastructure as Code with AWS CDK
- Set up multi-region deployments
- Add chaos engineering with AWS FIS
- Implement feature flags with AWS AppConfig

---

## Exam Alignment

**DEA-C01 Coverage:**

**Domain 1: Data Ingestion and Transformation (34%)**
- Automated ETL pipeline deployment
- CI/CD for data workflows
- Version control for data processing code

**Domain 3: Data Operations and Support (22%)**
- Monitoring with X-Ray and CloudWatch
- Performance optimization with profiling
- Automated rollback and recovery
- Distributed tracing for debugging

**Domain 4: Data Security and Governance (24%)**
- Automated security scanning (CodeGuru)
- Secret management in CI/CD
- Audit trails with CodePipeline
- Compliance with automated checks

### Practice Questions

**Q1:** You need to deploy Lambda ETL functions with zero downtime. Which deployment strategy should you use?
- A) Blue/Green deployment
- B) Canary deployment with automatic rollback
- C) All-at-once deployment
- D) Rolling deployment

**Answer: B** - Canary deployment gradually shifts traffic while monitoring errors, with automatic rollback if issues occur.

**Q2:** Your Lambda function has high latency but you don't know which part is slow. What should you use?
- A) CloudWatch Logs
- B) CloudWatch Metrics
- C) AWS X-Ray with subsegments
- D) Lambda Insights

**Answer: C** - X-Ray subsegments show time spent in each operation with millisecond precision.

**Q3:** How can you automatically detect security vulnerabilities in your data pipeline code?
- A) Manual code review
- B) AWS CodeGuru Reviewer
- C) AWS Inspector
- D) AWS Macie

**Answer: B** - CodeGuru Reviewer automatically scans code for security vulnerabilities, including hardcoded credentials and SQL injection.

---

**Total Cost Summary:**
- Exercise 13.1: $5-8/month (active development)
- Exercise 13.2: $5/month (100K traces)
- Exercise 13.3: $5-10/month (reviews + profiling)
- **Total: $15-20/month**

**Time Saved:**
- Manual deployments: 2 hours → 15 minutes (automated)
- Debugging issues: 8 hours → 30 minutes (with X-Ray)
- Code review: 2 hours → 10 minutes (with CodeGuru)
- Performance optimization: 8 hours → 1 hour (with Profiler)

**Estimated Time Investment:** 5 hours

**ROI: 304× ($5,000 saved per $15-20 cost)**

---

*This guide is part of the AWS Data Engineer Associate certification preparation series.*

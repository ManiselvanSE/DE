# Module 11: Management & Governance - GUI Step-by-Step Guide

## Overview
This module covers AWS management and governance services essential for data engineering operations. You'll learn to implement comprehensive monitoring with CloudWatch, automate infrastructure with CloudFormation, and establish organizational best practices with AWS Organizations.

**Duration:** 5 hours  
**Cost:** $120-150/month (includes monitoring, multi-account setup, and CloudTrail)  
**Difficulty:** Intermediate to Advanced

---

## Exercise 11.1: CloudWatch Monitoring for Data Pipeline

### Introduction
Build a comprehensive monitoring solution for a serverless data pipeline using CloudWatch. Implement custom metrics, multiple alarm types, dashboards, Logs Insights queries, and Contributor Insights for real-time visibility.

**What You'll Build:**
- Lambda function with custom CloudWatch metrics
- Multiple alarm types (threshold, anomaly detection, composite)
- CloudWatch dashboard with visualizations
- Logs Insights queries for analysis
- Contributor Insights rules for top contributors

**Duration:** 90 minutes  
**Cost:** $15-20/month (dashboards, custom metrics, Logs Insights queries)

---

### Part 1: Create Lambda Function with Custom Metrics

#### Step 1: Create S3 Bucket for Data Processing
1. Open the **AWS Console** and navigate to **S3**
2. Click **Create bucket**
3. Configure bucket:
   - **Bucket name:** `data-pipeline-processing-[your-account-id]`
   - **Region:** us-east-1
   - **Block all public access:** Keep checked
   - **Bucket Versioning:** Enable
   - **Encryption:** Server-side encryption with Amazon S3 managed keys (SSE-S3)
4. Click **Create bucket**

#### Step 2: Create IAM Role for Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select trusted entity:
   - **Trusted entity type:** AWS service
   - **Use case:** Lambda
4. Click **Next**
5. Add permissions (select these policies):
   - `AWSLambdaBasicExecutionRole`
   - `AmazonS3FullAccess`
   - `CloudWatchFullAccess`
6. Click **Next**
7. Role details:
   - **Role name:** `data-pipeline-lambda-role`
   - **Description:** `Role for data pipeline Lambda with CloudWatch metrics`
8. Click **Create role**

#### Step 3: Create Lambda Function
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure function:
   - **Function name:** `data-pipeline-processor`
   - **Runtime:** Python 3.11
   - **Architecture:** x86_64
   - **Permissions:** Use an existing role → Select `data-pipeline-lambda-role`
4. Click **Create function**

#### Step 4: Add Lambda Function Code with Custom Metrics
1. In the Lambda function, replace the code in `lambda_function.py` with:

```python
import boto3
import json
import time
import random
from datetime import datetime

s3 = boto3.client('s3')
cloudwatch = boto3.client('cloudwatch')

# Configuration
BUCKET_NAME = 'data-pipeline-processing-YOUR-ACCOUNT-ID'  # Update this
NAMESPACE = 'DataPipeline/Processing'

def put_custom_metrics(metric_name, value, unit='None', dimensions=None):
    """Send custom metrics to CloudWatch"""
    try:
        metric_data = {
            'MetricName': metric_name,
            'Value': value,
            'Unit': unit,
            'Timestamp': datetime.utcnow(),
            'StorageResolution': 1  # High-resolution metric (1-second)
        }
        
        if dimensions:
            metric_data['Dimensions'] = dimensions
        
        cloudwatch.put_metric_data(
            Namespace=NAMESPACE,
            MetricData=[metric_data]
        )
        print(f"Sent metric: {metric_name} = {value} {unit}")
    except Exception as e:
        print(f"Error sending metric: {str(e)}")

def process_file(bucket, key):
    """Process a single file and return metrics"""
    start_time = time.time()
    
    try:
        # Get file from S3
        response = s3.get_object(Bucket=bucket, Key=key)
        file_size = response['ContentLength']
        content = response['Body'].read().decode('utf-8')
        
        # Parse JSON data
        records = json.loads(content)
        record_count = len(records) if isinstance(records, list) else 1
        
        # Simulate processing (would be actual transformation in production)
        time.sleep(random.uniform(0.1, 0.5))  # Simulate processing time
        
        # Calculate metrics
        processing_time = time.time() - start_time
        records_per_second = record_count / processing_time if processing_time > 0 else 0
        
        # Send custom metrics
        put_custom_metrics('RecordsProcessed', record_count, 'Count')
        put_custom_metrics('FileSizeBytes', file_size, 'Bytes')
        put_custom_metrics('ProcessingTime', processing_time * 1000, 'Milliseconds')
        put_custom_metrics('RecordsPerSecond', records_per_second, 'Count/Second')
        
        # Dimension-based metrics for file type
        file_extension = key.split('.')[-1]
        put_custom_metrics(
            'FilesProcessed',
            1,
            'Count',
            dimensions=[
                {'Name': 'FileType', 'Value': file_extension},
                {'Name': 'Status', 'Value': 'Success'}
            ]
        )
        
        return {
            'status': 'success',
            'records': record_count,
            'file_size': file_size,
            'processing_time': processing_time
        }
        
    except json.JSONDecodeError as e:
        # Invalid JSON - send error metric
        put_custom_metrics(
            'FilesProcessed',
            1,
            'Count',
            dimensions=[
                {'Name': 'FileType', 'Value': 'json'},
                {'Name': 'Status', 'Value': 'ParseError'}
            ]
        )
        raise Exception(f"JSON parsing error: {str(e)}")
    
    except Exception as e:
        # General error
        processing_time = time.time() - start_time
        put_custom_metrics('ProcessingTime', processing_time * 1000, 'Milliseconds')
        put_custom_metrics(
            'FilesProcessed',
            1,
            'Count',
            dimensions=[
                {'Name': 'FileType', 'Value': 'unknown'},
                {'Name': 'Status', 'Value': 'Error'}
            ]
        )
        raise

def lambda_handler(event, context):
    """Main Lambda handler"""
    try:
        # Log the event
        print(f"Event: {json.dumps(event)}")
        
        # Check if triggered by S3 or manual test
        if 'Records' in event:
            # S3 trigger
            results = []
            for record in event['Records']:
                bucket = record['s3']['bucket']['name']
                key = record['s3']['object']['key']
                
                print(f"Processing file: s3://{bucket}/{key}")
                result = process_file(bucket, key)
                results.append(result)
            
            total_records = sum(r['records'] for r in results)
            
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'message': 'Success',
                    'files_processed': len(results),
                    'total_records': total_records
                })
            }
        else:
            # Manual test event
            bucket = event.get('bucket', BUCKET_NAME)
            key = event.get('key', 'test-data.json')
            
            # Create test file if it doesn't exist
            test_data = [
                {'id': 1, 'name': 'Product A', 'price': 29.99, 'quantity': 100},
                {'id': 2, 'name': 'Product B', 'price': 49.99, 'quantity': 75},
                {'id': 3, 'name': 'Product C', 'price': 19.99, 'quantity': 200}
            ]
            
            s3.put_object(
                Bucket=bucket,
                Key=key,
                Body=json.dumps(test_data).encode('utf-8'),
                ContentType='application/json'
            )
            
            result = process_file(bucket, key)
            
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'message': 'Success',
                    'result': result
                })
            }
            
    except Exception as e:
        print(f"Error: {str(e)}")
        
        # Send error metric
        put_custom_metrics('ProcessingErrors', 1, 'Count')
        
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

2. **Important:** Update `BUCKET_NAME` with your actual bucket name
3. Click **Deploy**

#### Step 5: Configure Lambda Timeout and Memory
1. Click **Configuration** tab
2. Click **General configuration**
3. Click **Edit**
4. Configure:
   - **Memory:** 512 MB
   - **Timeout:** 3 minutes
5. Click **Save**

#### Step 6: Configure S3 Trigger
1. Click **Add trigger**
2. Select **S3**
3. Configure trigger:
   - **Bucket:** Select `data-pipeline-processing-[your-account-id]`
   - **Event type:** All object create events
   - **Prefix:** `incoming/` (optional)
   - **Suffix:** `.json`
   - **Acknowledge:** Check the box
4. Click **Add**

#### Step 7: Test Lambda Function
1. Click **Test** tab
2. Create new test event:
   - **Event name:** `TestProcessing`
   - **Event JSON:**
```json
{
  "bucket": "data-pipeline-processing-YOUR-ACCOUNT-ID",
  "key": "test-data.json"
}
```
3. Click **Save**
4. Click **Test**
5. Verify successful execution
6. Run the test 5-10 times to generate metrics data

#### Step 8: Verify Custom Metrics in CloudWatch
1. Navigate to **CloudWatch**
2. In the left sidebar, click **Metrics** → **All metrics**
3. Click on **DataPipeline/Processing** namespace
4. You should see metrics:
   - `RecordsProcessed`
   - `FileSizeBytes`
   - `ProcessingTime`
   - `RecordsPerSecond`
   - `FilesProcessed` (with dimensions)
   - `ProcessingErrors`

---

### Part 2: Configure Multiple Types of Alarms

#### Step 9: Create SNS Topic for Alarm Notifications
1. Navigate to **SNS** (Simple Notification Service)
2. In the left sidebar, click **Topics**
3. Click **Create topic**
4. Configure topic:
   - **Type:** Standard
   - **Name:** `data-pipeline-alarms`
   - **Display name:** `Data Pipeline Alarms`
5. Click **Create topic**
6. Click **Create subscription**
7. Configure subscription:
   - **Protocol:** Email
   - **Endpoint:** your-email@example.com
8. Click **Create subscription**
9. Check your email and click the confirmation link

#### Step 10: Create Threshold Alarm for High Processing Time
1. Go to **CloudWatch** → **Alarms** → **All alarms**
2. Click **Create alarm**
3. Click **Select metric**
4. Browse to **DataPipeline/Processing**
5. Select **Metrics with no dimensions**
6. Check the box next to **ProcessingTime**
7. Click **Select metric**
8. Configure metric and conditions:
   - **Statistic:** Average
   - **Period:** 5 minutes
   - **Threshold type:** Static
   - **Whenever ProcessingTime is:** Greater than `2000` (2 seconds in milliseconds)
9. Click **Next**
10. Configure actions:
    - **Alarm state trigger:** In alarm
    - **Send a notification to:** Select `data-pipeline-alarms`
11. Click **Next**
12. Configure alarm details:
    - **Alarm name:** `HighProcessingTime`
    - **Alarm description:** `Alert when average processing time exceeds 2 seconds`
13. Click **Next**
14. Review and click **Create alarm**

#### Step 11: Create Anomaly Detection Alarm
1. Click **Create alarm** again
2. Click **Select metric**
3. Browse to **DataPipeline/Processing**
4. Select **RecordsProcessed**
5. Click **Select metric**
6. Configure metric:
   - **Statistic:** Sum
   - **Period:** 5 minutes
7. Under **Conditions**:
   - **Threshold type:** Select **Anomaly detection**
   - **Whenever RecordsProcessed is:** Outside of the band
   - **Anomaly detection threshold:** 2 (standard deviations)
8. Click **Next**
9. Configure actions:
   - **Send a notification to:** Select `data-pipeline-alarms`
10. Click **Next**
11. Configure alarm details:
    - **Alarm name:** `AnomalousRecordCount`
    - **Alarm description:** `Alert when record processing shows anomalous patterns`
12. Click **Next**
13. Click **Create alarm**

**Note:** Anomaly detection requires at least 2 weeks of data to establish baseline. It will show "Insufficient data" initially.

#### Step 12: Create Composite Alarm
1. Click **Create alarm** again
2. Click **Select metric**
3. Browse to **Lambda** → **By Function Name**
4. Select metrics for your function:
   - Check **Errors**
   - Check **Throttles**
5. Click **Select metric**
6. Note: For composite alarms, we'll create individual alarms first

**Create Error Rate Alarm:**
1. Create a new alarm (repeat steps above)
2. Metric: **Lambda** → **Errors** for `data-pipeline-processor`
3. Conditions:
   - **Statistic:** Sum
   - **Period:** 5 minutes
   - **Greater than:** 5
4. Skip SNS notification (we'll use composite alarm)
5. Name: `LambdaErrors`

**Create Throttle Alarm:**
1. Create another alarm
2. Metric: **Lambda** → **Throttles** for `data-pipeline-processor`
3. Conditions:
   - **Statistic:** Sum
   - **Period:** 5 minutes
   - **Greater than:** 1
4. Skip SNS notification
5. Name: `LambdaThrottles`

**Create Composite Alarm:**
1. Go to **CloudWatch** → **Alarms** → **All alarms**
2. Click **Create alarm**
3. At the top, click **Create a composite alarm**
4. Configure conditions:
   - **Alarm rule:** Enter the following expression:
   ```
   ALARM(LambdaErrors) OR ALARM(LambdaThrottles) OR ALARM(HighProcessingTime)
   ```
5. Click **Next**
6. Configure actions:
   - **Send a notification to:** Select `data-pipeline-alarms`
7. Click **Next**
8. Configure details:
   - **Alarm name:** `DataPipelineHealthCheck`
   - **Alarm description:** `Composite alarm for overall pipeline health`
9. Click **Next**
10. Click **Create alarm**

#### Step 13: Create Metric Math Alarm for Error Rate
1. Click **Create alarm**
2. Click **Select metric**
3. Select **Lambda** → **By Function Name**
4. Check both:
   - **Invocations** (give it ID: `m1`)
   - **Errors** (give it ID: `m2`)
5. Click **Graphed metrics** tab
6. Under **Add math** → **Common** → **Percentage**
7. Or manually add metric math:
   - **Expression:** `(m2/m1)*100`
   - **Label:** `Error Rate %`
   - **ID:** `e1`
8. Uncheck `m1` and `m2` (only keep `e1` checked)
9. Click **Select metric**
10. Configure conditions:
    - **Threshold:** Greater than `10` (10% error rate)
11. Configure notification to `data-pipeline-alarms`
12. Name: `HighErrorRate`
13. Click **Create alarm**

---

### Part 3: Build CloudWatch Dashboard

#### Step 14: Create Dashboard
1. Navigate to **CloudWatch** → **Dashboards**
2. Click **Create dashboard**
3. Dashboard name: `DataPipelineMonitoring`
4. Click **Create dashboard**

#### Step 15: Add Line Widget for Processing Metrics
1. Click **Add widget**
2. Select **Line**
3. Click **Configure**
4. Under **Data source**:
   - Select **Metrics**
5. Browse to **DataPipeline/Processing**
6. Select these metrics:
   - `ProcessingTime` (Statistic: Average)
   - `RecordsProcessed` (Statistic: Sum)
   - `RecordsPerSecond` (Statistic: Average)
7. Click **Create widget**

#### Step 16: Add Number Widget for Total Records
1. Click **Add widget** (the + icon)
2. Select **Number**
3. Click **Configure**
4. Browse to **DataPipeline/Processing**
5. Select `RecordsProcessed`
6. Under **Graphed metrics** tab:
   - **Statistic:** Sum
   - **Period:** 1 hour
7. Click **Widget properties** (gear icon)
8. Configure:
   - **Widget title:** `Total Records Processed (Last Hour)`
   - **Show sparkline:** Yes
9. Click **Create widget**

#### Step 17: Add Stacked Area Chart for File Types
1. Click **Add widget**
2. Select **Stacked area**
3. Click **Configure**
4. Browse to **DataPipeline/Processing**
5. Click **FilesProcessed**
6. Select all FileType dimensions
7. Under **Graphed metrics**:
   - **Statistic:** Sum
   - **Period:** 5 minutes
8. Click **Create widget**

#### Step 18: Add Lambda Metrics Widget
1. Click **Add widget**
2. Select **Line**
3. Browse to **Lambda** → **By Function Name**
4. Select your function and add:
   - `Invocations`
   - `Errors`
   - `Duration` (Average)
   - `Throttles`
   - `ConcurrentExecutions`
5. Click **Create widget**

#### Step 19: Add Pie Chart for Success vs Errors
1. Click **Add widget**
2. Select **Pie**
3. Browse to **DataPipeline/Processing** → **FilesProcessed**
4. Select metrics by Status dimension:
   - `Success`
   - `Error`
   - `ParseError`
5. Configure:
   - **Statistic:** Sum
   - **Period:** 1 hour
6. Click **Widget properties**
7. Title: `File Processing Status (Last Hour)`
8. Click **Create widget**

#### Step 20: Add Logs Widget
1. Click **Add widget**
2. Select **Logs table**
3. Click **Configure**
4. Under **Log groups**:
   - Select `/aws/lambda/data-pipeline-processor`
5. Under **Query**:
```
fields @timestamp, @message
| filter @message like /Error/
| sort @timestamp desc
| limit 20
```
6. Click **Widget properties**
7. Title: `Recent Errors`
8. Click **Create widget**

#### Step 21: Add Alarm Status Widget
1. Click **Add widget**
2. Select **Alarm status**
3. Click **Configure**
4. Select all your alarms:
   - `HighProcessingTime`
   - `AnomalousRecordCount`
   - `DataPipelineHealthCheck`
   - `HighErrorRate`
   - `LambdaErrors`
   - `LambdaThrottles`
5. Click **Create widget**

#### Step 22: Arrange and Save Dashboard
1. Drag widgets to organize the dashboard layout
2. Resize widgets as needed
3. Click **Save dashboard**
4. Click **Actions** → **Set auto refresh** → **1 minute**

---

### Part 4: Use Logs Insights for Analysis

#### Step 23: Create Logs Insights Query for Processing Statistics
1. Navigate to **CloudWatch** → **Logs Insights**
2. Select log group: `/aws/lambda/data-pipeline-processor`
3. Enter this query:
```
fields @timestamp, @message
| filter @message like /Sent metric: RecordsProcessed/
| parse @message /RecordsProcessed = (?<records>\d+)/
| stats sum(records) as TotalRecords, count(*) as Executions, avg(records) as AvgRecords by bin(5m)
| sort @timestamp desc
```
4. Set time range: **Last 1 hour**
5. Click **Run query**
6. View results in table and visualization
7. Click **Save** to save the query:
   - **Query name:** `ProcessingStatistics`
   - Click **Save**

#### Step 24: Create Query for Error Analysis
1. Create a new query:
```
fields @timestamp, @message, @requestId
| filter @message like /Error/
| stats count() as ErrorCount by @requestId, bin(5m) as Time
| sort ErrorCount desc
```
2. Click **Run query**
3. Save as: `ErrorAnalysis`

#### Step 25: Create Query for Performance Analysis
1. Create a new query:
```
fields @timestamp, @duration, @billedDuration, @memorySize, @maxMemoryUsed
| stats avg(@duration) as AvgDuration, 
        max(@duration) as MaxDuration, 
        avg(@maxMemoryUsed / @memorySize * 100) as MemoryUtilization
        by bin(5m)
| sort @timestamp desc
```
2. Click **Run query**
3. Save as: `PerformanceMetrics`

#### Step 26: Create Query for Cold Start Analysis
1. Create a new query:
```
fields @timestamp, @initDuration, @duration
| filter ispresent(@initDuration)
| stats count() as ColdStarts, 
        avg(@initDuration) as AvgColdStartTime,
        avg(@duration) as AvgTotalDuration
        by bin(1h)
```
2. Click **Run query**
3. Save as: `ColdStartAnalysis`

#### Step 27: Add Logs Insights Query to Dashboard
1. From any of your saved queries, click **Add to dashboard**
2. Select: `DataPipelineMonitoring`
3. Widget title: `Processing Statistics`
4. Click **Add to dashboard**
5. Navigate back to your dashboard to see the widget

---

### Part 5: Set Up Contributor Insights

#### Step 28: Create Contributor Insights Rule for Top Error Producers
1. Navigate to **CloudWatch** → **Contributor Insights**
2. Click **Create rule**
3. Configure rule:
   - **Rule name:** `TopErrorProducers`
   - **Log group:** `/aws/lambda/data-pipeline-processor`
4. Enter rule syntax:
```json
{
  "Schema": {
    "Name": "CloudWatchLogRule",
    "Version": 1
  },
  "LogGroupNames": [
    "/aws/lambda/data-pipeline-processor"
  ],
  "LogFormat": "JSON",
  "Fields": {
    "2": "$.requestId",
    "1": "$.level"
  },
  "Contribution": {
    "Keys": [
      "2"
    ],
    "Filters": [
      {
        "Match": "$.level",
        "EqualTo": "ERROR"
      }
    ]
  },
  "AggregateOn": "Count"
}
```
5. Click **Create**

**Note:** If your logs aren't in JSON format, use this alternative for text logs:

```json
{
  "Schema": {
    "Name": "CloudWatchLogRule",
    "Version": 1
  },
  "LogGroupNames": [
    "/aws/lambda/data-pipeline-processor"
  ],
  "LogFormat": "CLF",
  "Fields": {
    "1": "$requestId"
  },
  "Contribution": {
    "Keys": [
      "1"
    ],
    "Filters": [
      {
        "Match": "$message",
        "In": [
          "Error"
        ]
      }
    ]
  },
  "AggregateOn": "Count"
}
```

#### Step 29: Create Rule for Top File Types Processed
1. Click **Create rule** again
2. Configure:
   - **Rule name:** `TopFileTypes`
   - **Log group:** `/aws/lambda/data-pipeline-processor`
3. Enter rule syntax:
```json
{
  "Schema": {
    "Name": "CloudWatchLogRule",
    "Version": 1
  },
  "LogGroupNames": [
    "/aws/lambda/data-pipeline-processor"
  ],
  "LogFormat": "CLF",
  "Fields": {
    "1": "$fileType"
  },
  "Contribution": {
    "Keys": [
      "1"
    ],
    "Filters": []
  },
  "AggregateOn": "Count"
}
```
4. Click **Create**

#### Step 30: View Contributor Insights Reports
1. Go to **Contributor Insights** → **Rules**
2. Click on `TopErrorProducers`
3. View the report showing:
   - Top contributors (request IDs with most errors)
   - Contribution percentage
   - Time series graph
4. Set time range to **Last 1 hour**
5. Click **Add to dashboard** to add to your dashboard

---

### Verification Checklist

- [ ] Lambda function created with custom metrics
- [ ] Custom CloudWatch metrics visible in console
- [ ] SNS topic created with email subscription
- [ ] Threshold alarm created for processing time
- [ ] Anomaly detection alarm configured
- [ ] Composite alarm combining multiple conditions
- [ ] Metric math alarm for error rate created
- [ ] CloudWatch dashboard created with 7+ widgets
- [ ] Logs Insights queries created and saved
- [ ] Contributor Insights rules created
- [ ] Dashboard auto-refresh enabled
- [ ] Email notifications received from alarms

### Architecture Benefits

**Visibility:**
- Real-time monitoring of all pipeline components
- Custom metrics tailored to business needs
- Centralized dashboard for team visibility

**Proactive Alerting:**
- Multiple alarm types catch different issue patterns
- Composite alarms reduce alert fatigue
- Anomaly detection finds unusual patterns automatically

**Cost Optimization:**
- High-resolution metrics (1-second) cost: $0.30/metric/month
- Standard metrics: $0.10/metric/month
- Dashboard: $3/month
- Logs Insights queries: $0.005/GB scanned
- Contributor Insights: $0.50/rule/month

**Troubleshooting:**
- Logs Insights for rapid log analysis
- Contributor Insights identifies top error sources
- Correlation across metrics, logs, and traces

---

## Exercise 11.2: CloudFormation Infrastructure as Code

### Introduction
Implement Infrastructure as Code (IaC) using CloudFormation to automate deployment of data infrastructure across multiple environments with change management and drift detection.

**What You'll Build:**
- CloudFormation templates for data infrastructure
- Dev and prod stacks with different parameters
- Change sets for safe updates
- Drift detection and remediation
- StackSets for multi-account deployment

**Duration:** 120 minutes  
**Cost:** $50-80/month (includes resources deployed across environments)

---

### Part 1: Create CloudFormation Template for Data Pipeline

#### Step 1: Create Basic Template File
1. On your local machine, create a file: `data-pipeline-template.yaml`
2. Add the basic structure:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Data Pipeline Infrastructure - S3, Lambda, DynamoDB, and CloudWatch'

Parameters:
  EnvironmentName:
    Type: String
    Description: Environment name (dev, staging, prod)
    AllowedValues:
      - dev
      - staging
      - prod
    Default: dev
  
  LambdaMemorySize:
    Type: Number
    Description: Lambda function memory size
    Default: 512
    AllowedValues:
      - 256
      - 512
      - 1024
      - 2048
  
  AlarmEmail:
    Type: String
    Description: Email address for alarm notifications
    AllowedPattern: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
  
  EnableDetailedMonitoring:
    Type: String
    Description: Enable detailed CloudWatch monitoring
    Default: 'false'
    AllowedValues:
      - 'true'
      - 'false'

Conditions:
  IsProduction: !Equals [!Ref EnvironmentName, prod]
  EnableMonitoring: !Equals [!Ref EnableDetailedMonitoring, 'true']

Resources:
  # S3 Bucket for raw data
  RawDataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${EnvironmentName}-raw-data-${AWS::AccountId}'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      VersioningConfiguration:
        Status: !If [IsProduction, Enabled, Suspended]
      LifecycleConfiguration:
        Rules:
          - Id: TransitionToIA
            Status: Enabled
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
          - Id: ExpireOldVersions
            Status: Enabled
            NoncurrentVersionExpirationInDays: 90
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
        - Key: ManagedBy
          Value: CloudFormation
  
  # S3 Bucket for processed data
  ProcessedDataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${EnvironmentName}-processed-data-${AWS::AccountId}'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      VersioningConfiguration:
        Status: !If [IsProduction, Enabled, Suspended]
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
  
  # DynamoDB Table for metadata
  MetadataTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: !Sub '${EnvironmentName}-pipeline-metadata'
      BillingMode: !If [IsProduction, PROVISIONED, PAY_PER_REQUEST]
      AttributeDefinitions:
        - AttributeName: PipelineId
          AttributeType: S
        - AttributeName: Timestamp
          AttributeType: N
      KeySchema:
        - AttributeName: PipelineId
          KeyType: HASH
        - AttributeName: Timestamp
          KeyType: RANGE
      ProvisionedThroughput:
        ReadCapacityUnits: !If [IsProduction, 5, !Ref AWS::NoValue]
        WriteCapacityUnits: !If [IsProduction, 5, !Ref AWS::NoValue]
      StreamSpecification:
        StreamViewType: NEW_AND_OLD_IMAGES
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: !If [IsProduction, true, false]
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
  
  # IAM Role for Lambda
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${EnvironmentName}-pipeline-lambda-role'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: S3Access
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                Resource:
                  - !Sub '${RawDataBucket.Arn}/*'
                  - !Sub '${ProcessedDataBucket.Arn}/*'
        - PolicyName: DynamoDBAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:PutItem
                  - dynamodb:GetItem
                  - dynamodb:UpdateItem
                  - dynamodb:Query
                Resource: !GetAtt MetadataTable.Arn
        - PolicyName: CloudWatchMetrics
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - cloudwatch:PutMetricData
                Resource: '*'
  
  # Lambda Function
  DataProcessorFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub '${EnvironmentName}-data-processor'
      Runtime: python3.11
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      MemorySize: !Ref LambdaMemorySize
      Timeout: 300
      Environment:
        Variables:
          ENVIRONMENT: !Ref EnvironmentName
          PROCESSED_BUCKET: !Ref ProcessedDataBucket
          METADATA_TABLE: !Ref MetadataTable
          ENABLE_MONITORING: !Ref EnableDetailedMonitoring
      Code:
        ZipFile: |
          import boto3
          import json
          import os
          from datetime import datetime
          
          s3 = boto3.client('s3')
          dynamodb = boto3.resource('dynamodb')
          cloudwatch = boto3.client('cloudwatch')
          
          PROCESSED_BUCKET = os.environ['PROCESSED_BUCKET']
          METADATA_TABLE = os.environ['METADATA_TABLE']
          ENVIRONMENT = os.environ['ENVIRONMENT']
          
          def lambda_handler(event, context):
              try:
                  table = dynamodb.Table(METADATA_TABLE)
                  
                  for record in event['Records']:
                      bucket = record['s3']['bucket']['name']
                      key = record['s3']['object']['key']
                      
                      # Get object
                      obj = s3.get_object(Bucket=bucket, Key=key)
                      content = obj['Body'].read().decode('utf-8')
                      
                      # Process data (simple transformation)
                      data = json.loads(content)
                      processed_data = {
                          'original': data,
                          'processed_at': datetime.utcnow().isoformat(),
                          'environment': ENVIRONMENT,
                          'record_count': len(data) if isinstance(data, list) else 1
                      }
                      
                      # Save to processed bucket
                      output_key = f"processed/{key}"
                      s3.put_object(
                          Bucket=PROCESSED_BUCKET,
                          Key=output_key,
                          Body=json.dumps(processed_data).encode('utf-8'),
                          ContentType='application/json'
                      )
                      
                      # Save metadata to DynamoDB
                      table.put_item(
                          Item={
                              'PipelineId': key,
                              'Timestamp': int(datetime.utcnow().timestamp()),
                              'SourceBucket': bucket,
                              'SourceKey': key,
                              'OutputBucket': PROCESSED_BUCKET,
                              'OutputKey': output_key,
                              'Status': 'Completed',
                              'Environment': ENVIRONMENT
                          }
                      )
                      
                      # Send custom metric
                      cloudwatch.put_metric_data(
                          Namespace=f'{ENVIRONMENT}/DataPipeline',
                          MetricData=[
                              {
                                  'MetricName': 'FilesProcessed',
                                  'Value': 1,
                                  'Unit': 'Count'
                              }
                          ]
                      )
                  
                  return {
                      'statusCode': 200,
                      'body': json.dumps('Success')
                  }
              
              except Exception as e:
                  print(f"Error: {str(e)}")
                  return {
                      'statusCode': 500,
                      'body': json.dumps({'error': str(e)})
                  }
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
  
  # S3 Event Notification Permission
  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref DataProcessorFunction
      Action: lambda:InvokeFunction
      Principal: s3.amazonaws.com
      SourceArn: !GetAtt RawDataBucket.Arn
  
  # SNS Topic for Alarms
  AlarmTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${EnvironmentName}-pipeline-alarms'
      DisplayName: !Sub '${EnvironmentName} Pipeline Alarms'
      Subscription:
        - Endpoint: !Ref AlarmEmail
          Protocol: email
  
  # CloudWatch Alarm for Lambda Errors
  LambdaErrorAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-lambda-errors'
      AlarmDescription: Alert when Lambda function errors exceed threshold
      MetricName: Errors
      Namespace: AWS/Lambda
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: !If [IsProduction, 5, 10]
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: FunctionName
          Value: !Ref DataProcessorFunction
      AlarmActions:
        - !Ref AlarmTopic
      TreatMissingData: notBreaching
  
  # CloudWatch Alarm for Lambda Throttles
  LambdaThrottleAlarm:
    Type: AWS::CloudWatch::Alarm
    Condition: IsProduction
    Properties:
      AlarmName: !Sub '${EnvironmentName}-lambda-throttles'
      AlarmDescription: Alert when Lambda function is throttled
      MetricName: Throttles
      Namespace: AWS/Lambda
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 1
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: FunctionName
          Value: !Ref DataProcessorFunction
      AlarmActions:
        - !Ref AlarmTopic

Outputs:
  RawDataBucketName:
    Description: Name of the raw data S3 bucket
    Value: !Ref RawDataBucket
    Export:
      Name: !Sub '${EnvironmentName}-RawDataBucket'
  
  ProcessedDataBucketName:
    Description: Name of the processed data S3 bucket
    Value: !Ref ProcessedDataBucket
    Export:
      Name: !Sub '${EnvironmentName}-ProcessedDataBucket'
  
  MetadataTableName:
    Description: Name of the DynamoDB metadata table
    Value: !Ref MetadataTable
    Export:
      Name: !Sub '${EnvironmentName}-MetadataTable'
  
  LambdaFunctionArn:
    Description: ARN of the Lambda function
    Value: !GetAtt DataProcessorFunction.Arn
    Export:
      Name: !Sub '${EnvironmentName}-ProcessorFunctionArn'
  
  AlarmTopicArn:
    Description: ARN of the SNS topic for alarms
    Value: !Ref AlarmTopic
    Export:
      Name: !Sub '${EnvironmentName}-AlarmTopicArn'
```

3. Save the file

#### Step 2: Create Development Stack
1. Navigate to **CloudFormation** in AWS Console
2. Click **Create stack** → **With new resources (standard)**
3. Under **Prepare template:** Choose **Template is ready**
4. Under **Template source:** Choose **Upload a template file**
5. Click **Choose file** and upload `data-pipeline-template.yaml`
6. Click **Next**
7. Configure stack:
   - **Stack name:** `data-pipeline-dev`
   - **EnvironmentName:** dev
   - **LambdaMemorySize:** 512
   - **AlarmEmail:** your-email@example.com
   - **EnableDetailedMonitoring:** false
8. Click **Next**
9. Configure stack options:
   - **Tags:**
     - Key: `Environment`, Value: `dev`
     - Key: `ManagedBy`, Value: `CloudFormation`
   - **Permissions:** Leave as default
   - **Stack failure options:** Roll back all stack resources
10. Click **Next**
11. Review all settings
12. Check **I acknowledge that AWS CloudFormation might create IAM resources**
13. Click **Submit**
14. Wait 3-5 minutes for stack creation (Status: `CREATE_COMPLETE`)

#### Step 3: Create Production Stack with Different Parameters
1. Click **Create stack** again
2. Upload the same template
3. Click **Next**
4. Configure stack:
   - **Stack name:** `data-pipeline-prod`
   - **EnvironmentName:** prod
   - **LambdaMemorySize:** 1024 (higher for production)
   - **AlarmEmail:** your-email@example.com
   - **EnableDetailedMonitoring:** true
5. Click **Next**
6. Add tags:
   - Key: `Environment`, Value: `prod`
   - Key: `ManagedBy`, Value: `CloudFormation`
7. Click **Next**
8. Review and acknowledge IAM resource creation
9. Click **Submit**
10. Wait for stack creation

#### Step 4: Verify Stack Outputs
1. Click on `data-pipeline-dev` stack
2. Click **Outputs** tab
3. Verify all outputs are present:
   - RawDataBucketName
   - ProcessedDataBucketName
   - MetadataTableName
   - LambdaFunctionArn
   - AlarmTopicArn
4. Copy these values for testing
5. Repeat for `data-pipeline-prod` stack

#### Step 5: Test the Deployed Infrastructure
1. Navigate to **S3**
2. Find the dev raw data bucket: `dev-raw-data-[account-id]`
3. Upload a test JSON file:

```json
[
  {"id": 1, "name": "Test Data", "value": 100},
  {"id": 2, "name": "Sample Record", "value": 200}
]
```

4. Name it: `test-input.json`
5. Navigate to **Lambda** and check the dev function was triggered
6. Check the processed data bucket for output
7. Navigate to **DynamoDB** and verify metadata table has an entry
8. Check your email for SNS subscription confirmation

---

### Part 2: Create and Execute Change Sets

#### Step 6: Modify Template to Add New Resource
1. Edit your `data-pipeline-template.yaml` file
2. Add a new S3 bucket for archived data in the `Resources` section:

```yaml
  # Add this after ProcessedDataBucket
  ArchiveDataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${EnvironmentName}-archive-data-${AWS::AccountId}'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      LifecycleConfiguration:
        Rules:
          - Id: TransitionToGlacier
            Status: Enabled
            Transitions:
              - TransitionInDays: 90
                StorageClass: GLACIER
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName
```

3. Add output for the new bucket in the `Outputs` section:

```yaml
  ArchiveDataBucketName:
    Description: Name of the archive data S3 bucket
    Value: !Ref ArchiveDataBucket
    Export:
      Name: !Sub '${EnvironmentName}-ArchiveDataBucket'
```

4. Save the file

#### Step 7: Create Change Set for Dev Stack
1. Go to **CloudFormation** → **Stacks**
2. Select `data-pipeline-dev` stack
3. Click **Stack actions** → **Create change set for current stack**
4. Under **Prepare template:** Choose **Replace current template**
5. Under **Template source:** Upload the modified template
6. Click **Next**
7. Keep all parameters the same
8. Click **Next**
9. Click **Next** again
10. Change set details:
    - **Change set name:** `add-archive-bucket`
    - **Description:** `Add S3 bucket for archived data`
11. Click **Submit**

#### Step 8: Review Change Set
1. Wait for change set creation (status: `CREATE_COMPLETE`)
2. Click on the change set name `add-archive-bucket`
3. Review **Changes** section:
   - You should see:
     - **Add** - AWS::S3::Bucket (ArchiveDataBucket)
     - **Modify** - Outputs
4. Review **JSON changes** tab to see exact modifications
5. Verify no existing resources will be replaced or deleted

#### Step 9: Execute Change Set
1. Click **Execute change set**
2. Confirm by clicking **Execute change set** again
3. Wait for stack update (Status: `UPDATE_COMPLETE`)
4. Go to **Outputs** tab
5. Verify new output `ArchiveDataBucketName` is present
6. Navigate to **S3** and verify the new bucket exists

#### Step 10: Create and Execute Change Set for Prod
1. Repeat steps 7-9 for `data-pipeline-prod` stack
2. Create change set: `add-archive-bucket-prod`
3. Review changes carefully (production requires more scrutiny)
4. Execute the change set
5. Verify production stack updated successfully

---

### Part 3: Detect and Remediate Drift

#### Step 11: Manually Modify Stack Resources (Simulate Drift)
1. Navigate to **S3**
2. Find the dev processed data bucket: `dev-processed-data-[account-id]`
3. Click on the bucket
4. Click **Properties** tab
5. Scroll to **Tags** section
6. Click **Edit**
7. Add a new tag:
   - Key: `ManualChange`
   - Value: `true`
8. Click **Save changes**

9. Navigate to **DynamoDB**
10. Find the table: `dev-pipeline-metadata`
11. Click on the table
12. Click **Additional settings** tab
13. Under **Read/write capacity**, click **Edit**
14. Change billing mode (if it's PAY_PER_REQUEST, you can't change capacity)
15. Instead, go to **Tags** and add:
    - Key: `Modified`
    - Value: `OutsideCloudFormation`
16. Click **Save changes**

#### Step 12: Detect Drift on Entire Stack
1. Go to **CloudFormation** → **Stacks**
2. Select `data-pipeline-dev` stack
3. Click **Stack actions** → **Detect drift**
4. Click **Detect drift** to confirm
5. Wait for drift detection to complete (30-60 seconds)
6. You'll see a notification: "Drift detection complete"

#### Step 13: View Drift Results
1. Click on the stack `data-pipeline-dev`
2. Click **Stack actions** → **View drift results**
3. Review **Drift status:** Should show `DRIFTED`
4. Review **Resource drift status** section:
   - Find `ProcessedDataBucket`: Status `MODIFIED`
   - Find `MetadataTable`: Status `MODIFIED` (if tags were changed)
   - Other resources: Status `IN_SYNC`
5. Click on `ProcessedDataBucket` to see details
6. Review **Differences** tab showing:
   - Expected value (from template)
   - Current value (actual resource)
   - Difference type (e.g., Tags added)

#### Step 14: Detect Drift on Individual Resource
1. Go back to stack resources view
2. Click **Resources** tab
3. Find `ProcessedDataBucket` in the list
4. Select it and click **Drift status** column
5. Click **Detect drift** (resource-level detection)
6. View detailed drift information

#### Step 15: Remediate Drift by Updating Stack
1. Select the `data-pipeline-dev` stack
2. Click **Update**
3. Choose **Use current template**
4. Click **Next**
5. Click **Next** through all screens
6. On review page, click **Submit**
7. CloudFormation will update the stack and **remove the manual tags**
8. Resources will return to the state defined in template
9. Wait for `UPDATE_COMPLETE`

#### Step 16: Verify Drift Remediation
1. Run drift detection again
2. Verify **Drift status:** `IN_SYNC`
3. Navigate to **S3** and check the bucket
4. Verify the `ManualChange` tag is removed
5. Only CloudFormation-managed tags should remain

---

### Part 4: Use StackSets for Multi-Account Deployment

**Note:** This section requires AWS Organizations to be set up. If you don't have multiple accounts, you can still follow the steps to understand the process.

#### Step 17: Enable Trusted Access for StackSets
1. Navigate to **CloudFormation** → **StackSets**
2. If prompted, click **Enable trusted access with AWS Organizations**
3. This allows StackSets to deploy across organization accounts

#### Step 18: Create StackSet Template
1. Create a new file: `logging-infrastructure-template.yaml`
2. Add the following template:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Centralized Logging Infrastructure for Data Pipelines'

Parameters:
  CentralLogBucketName:
    Type: String
    Description: Name of the central logging bucket
    Default: 'central-pipeline-logs'

Resources:
  # S3 Bucket for centralized logs
  CentralLogBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${CentralLogBucketName}-${AWS::AccountId}-${AWS::Region}'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      LifecycleConfiguration:
        Rules:
          - Id: TransitionToIA
            Status: Enabled
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
              - TransitionInDays: 90
                StorageClass: GLACIER
          - Id: ExpireOldLogs
            Status: Enabled
            ExpirationInDays: 365
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
  
  # CloudWatch Log Group
  CentralLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /aws/data-pipeline/central
      RetentionInDays: 30
  
  # SNS Topic for Alerts
  AlertTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: data-pipeline-alerts
      DisplayName: Data Pipeline Alerts

Outputs:
  LogBucketName:
    Description: Central logging bucket name
    Value: !Ref CentralLogBucket
    Export:
      Name: CentralLogBucket
  
  LogGroupName:
    Description: CloudWatch log group name
    Value: !Ref CentralLogGroup
    Export:
      Name: CentralLogGroup
  
  AlertTopicArn:
    Description: SNS topic ARN for alerts
    Value: !Ref AlertTopic
    Export:
      Name: AlertTopicArn
```

3. Save the file

#### Step 19: Create StackSet
1. Go to **CloudFormation** → **StackSets**
2. Click **Create StackSet**
3. Configure template:
   - **Permissions:** Service-managed permissions
   - **Template source:** Upload a template file
   - Upload `logging-infrastructure-template.yaml`
4. Click **Next**
5. StackSet details:
   - **StackSet name:** `logging-infrastructure`
   - **StackSet description:** `Centralized logging for data pipelines`
6. Parameters:
   - **CentralLogBucketName:** `central-pipeline-logs`
7. Click **Next**

#### Step 20: Configure StackSet Deployment Options
1. Deployment targets:
   - **Deploy to:** Deploy to organization
   - **Organizational units (OUs):** Select your OUs
   - Or **Deploy to accounts:** Enter specific account IDs
2. Specify regions:
   - Select regions: `us-east-1`, `us-west-2`
3. Deployment options:
   - **Maximum concurrent accounts:** 2
   - **Failure tolerance:** 0
   - **Region concurrency:** Sequential
4. Click **Next**

#### Step 21: Review and Create StackSet
1. Review all settings
2. Check **I acknowledge that AWS CloudFormation might create IAM resources**
3. Click **Submit**
4. Wait for StackSet creation

#### Step 22: Monitor StackSet Operations
1. Click on the `logging-infrastructure` StackSet
2. Click **Stack instances** tab
3. View deployment status for each account/region combination
4. Status should show `CURRENT` when complete
5. Click **Operations** tab to see deployment history

#### Step 23: Update StackSet
1. Edit the template to add a parameter or resource
2. Go to **StackSets** → Select `logging-infrastructure`
3. Click **Actions** → **Edit StackSet details**
4. Upload the modified template
5. Click **Next** through all screens
6. On **Set deployment options:**
   - Choose accounts and regions to update
7. Click **Next** and **Submit**
8. Monitor the update operation in **Operations** tab

#### Step 24: Delete Stack Instances (Cleanup)
1. Select the StackSet
2. Click **Actions** → **Delete stacks from StackSet**
3. Configure:
   - **Accounts:** Specify account IDs or deploy to organization
   - **Regions:** Select all regions where deployed
   - **Retain stacks:** Uncheck (will delete resources)
4. Click **Next**
5. Deployment options:
   - **Maximum concurrent accounts:** 5
   - **Failure tolerance:** 0
6. Click **Next**
7. Click **Submit**
8. Wait for deletion to complete

#### Step 25: Delete StackSet
1. Once all stack instances are deleted
2. Select the StackSet
3. Click **Actions** → **Delete StackSet**
4. Confirm deletion
5. Click **Delete StackSet**

---

### Verification Checklist

- [ ] CloudFormation template created with parameters and conditions
- [ ] Dev stack deployed successfully
- [ ] Prod stack deployed with different parameters
- [ ] Stack outputs visible and correct
- [ ] Lambda function triggered by S3 upload
- [ ] DynamoDB metadata table populated
- [ ] Change set created and reviewed
- [ ] Change set executed successfully
- [ ] New resources added via change set
- [ ] Drift detection run on stack
- [ ] Drift identified on modified resources
- [ ] Drift remediated by stack update
- [ ] StackSet created for multi-account deployment
- [ ] Stack instances deployed to target accounts/regions

### Architecture Benefits

**Consistency:**
- Identical infrastructure across environments
- Version-controlled infrastructure code
- Repeatable deployments

**Change Management:**
- Change sets allow preview before updates
- Rollback capability on failures
- Audit trail of all changes

**Drift Detection:**
- Identifies manual changes to resources
- Maintains infrastructure integrity
- Prevents configuration drift

**Multi-Account Management:**
- Deploy to multiple accounts simultaneously
- Centralized management of infrastructure
- Consistent security and compliance posture

**Cost Optimization:**
- Different resource sizes per environment
- Lifecycle policies automated
- Easy cleanup of entire stacks

---

## Exercise 11.3: AWS Organizations for Multi-Account Governance

### Introduction
Establish enterprise-grade multi-account governance using AWS Organizations with organizational units, service control policies, centralized logging, consolidated billing, and budget alerts.

**What You'll Build:**
- Organization with hierarchical OUs
- Service Control Policies (SCPs) for governance
- Centralized CloudTrail logging
- Consolidated billing view
- Budget alerts and cost controls

**Duration:** 90 minutes  
**Cost:** $50-70/month (includes CloudTrail, Config, and resources in member accounts)

---

### Part 1: Create Organization with OUs

#### Step 1: Create AWS Organization
1. Navigate to **AWS Organizations**
2. Click **Create an organization**
3. Choose organization type:
   - **Enable all features** (recommended for SCPs)
4. Click **Create organization**
5. Verify your email (check inbox for verification)
6. Confirm email to complete setup

#### Step 2: Create Organizational Units (OUs)
1. In **AWS Organizations**, go to **AWS accounts**
2. You'll see your root account under **Root**
3. Select **Root** 
4. Click **Actions** → **Create new**

**Create Production OU:**
1. **Organizational unit name:** `Production`
2. Click **Create organizational unit**

**Create Development OU:**
1. Select **Root** again
2. Click **Actions** → **Create new**
3. **Organizational unit name:** `Development`
4. Click **Create organizational unit**

**Create Data Engineering OU:**
1. Select **Root** again
2. Click **Actions** → **Create new**
3. **Organizational unit name:** `DataEngineering`
4. Click **Create organizational unit**

**Create Security OU:**
1. Select **Root** again
2. Click **Actions** → **Create new**
3. **Organizational unit name:** `Security`
4. Click **Create organizational unit**

#### Step 3: Create Member Accounts
1. In **AWS Organizations**, click **AWS accounts**
2. Click **Add an AWS account** → **Create an AWS account**

**Create Dev Account:**
1. **AWS account name:** `DataEngineering-Dev`
2. **Email address:** `aws-dev@yourdomain.com` (must be unique)
3. **IAM role name:** `OrganizationAccountAccessRole`
4. Click **Create AWS account**
5. Wait for account creation (takes a few minutes)

**Create Prod Account:**
1. Click **Add an AWS account** → **Create an AWS account**
2. **AWS account name:** `DataEngineering-Prod`
3. **Email address:** `aws-prod@yourdomain.com`
4. **IAM role name:** `OrganizationAccountAccessRole`
5. Click **Create AWS account**

**Create Security Account:**
1. Click **Add an AWS account** → **Create an AWS account**
2. **AWS account name:** `Security-Logging`
3. **Email address:** `aws-security@yourdomain.com`
4. **IAM role name:** `OrganizationAccountAccessRole`
5. Click **Create AWS account**

#### Step 4: Move Accounts to OUs
1. Go to **AWS accounts**
2. Find `DataEngineering-Dev` account
3. Select it and click **Actions** → **Move**
4. Select **Development** OU
5. Click **Move AWS account**

6. Repeat for `DataEngineering-Prod`:
   - Move to **Production** OU

7. Repeat for `Security-Logging`:
   - Move to **Security** OU

#### Step 5: Verify OU Structure
1. Click on **Root**
2. Expand the tree to see:
   ```
   Root
   ├── Production
   │   └── DataEngineering-Prod
   ├── Development
   │   └── DataEngineering-Dev
   ├── DataEngineering
   └── Security
       └── Security-Logging
   ```

---

### Part 2: Implement Service Control Policies (SCPs)

#### Step 6: Enable SCPs
1. In **AWS Organizations**, go to **Policies**
2. Click **Service control policies**
3. If not enabled, click **Enable service control policies**
4. Confirm enablement

#### Step 7: Create SCP to Restrict Regions
1. Click **Create policy**
2. Policy details:
   - **Policy name:** `RestrictRegions`
   - **Description:** `Restrict access to specific AWS regions`
3. Policy content (replace the default JSON):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllOutsideApprovedRegions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        },
        "ForAllValues:StringNotEquals": {
          "aws:PrincipalServiceNamesList": [
            "cloudfront.amazonaws.com",
            "route53.amazonaws.com",
            "iam.amazonaws.com",
            "organizations.amazonaws.com",
            "support.amazonaws.com"
          ]
        }
      }
    }
  ]
}
```

4. Click **Create policy**

#### Step 8: Create SCP to Prevent Root User Usage
1. Click **Create policy**
2. Policy details:
   - **Policy name:** `PreventRootUserAccess`
   - **Description:** `Prevent usage of root user credentials`
3. Policy content:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserAccess",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

4. Click **Create policy**

#### Step 9: Create SCP to Require Encryption
1. Click **Create policy**
2. Policy details:
   - **Policy name:** `RequireS3Encryption`
   - **Description:** `Require encryption for S3 buckets`
3. Policy content:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": [
            "AES256",
            "aws:kms"
          ]
        }
      }
    },
    {
      "Sid": "DenyUnencryptedBucketCreation",
      "Effect": "Deny",
      "Action": "s3:CreateBucket",
      "Resource": "*",
      "Condition": {
        "Null": {
          "s3:x-amz-server-side-encryption-aws-kms-key-id": "true"
        }
      }
    }
  ]
}
```

4. Click **Create policy**

#### Step 10: Create SCP for Development Environment
1. Click **Create policy**
2. Policy details:
   - **Policy name:** `DevelopmentRestrictions`
   - **Description:** `Restrictions for development accounts`
3. Policy content:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PreventLargeInstanceTypes",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances"
      ],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotLike": {
          "ec2:InstanceType": [
            "t2.*",
            "t3.*",
            "t3a.*"
          ]
        }
      }
    },
    {
      "Sid": "PreventProductionDataAccess",
      "Effect": "Deny",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::*prod*/*"
    },
    {
      "Sid": "PreventExpensiveServices",
      "Effect": "Deny",
      "Action": [
        "redshift:*",
        "sagemaker:CreateTrainingJob",
        "emr:RunJobFlow"
      ],
      "Resource": "*"
    }
  ]
}
```

4. Click **Create policy**

#### Step 11: Attach SCPs to OUs
1. Go to **AWS accounts**
2. Select **Development** OU
3. Click **Policies** tab
4. Click **Attach**
5. Select these policies:
   - `RestrictRegions`
   - `DevelopmentRestrictions`
   - `RequireS3Encryption`
6. Click **Attach policy**

7. Select **Production** OU
8. Click **Policies** tab
9. Click **Attach**
10. Select these policies:
    - `RestrictRegions`
    - `PreventRootUserAccess`
    - `RequireS3Encryption`
11. Click **Attach policy**

#### Step 12: Test SCP Enforcement
1. Switch to the Dev member account:
   - In Organizations, click on **DataEngineering-Dev** account
   - Copy the account ID
   - Click your account name → **Switch Role**
   - **Account:** Enter dev account ID
   - **Role:** `OrganizationAccountAccessRole`
   - Click **Switch Role**

2. Try to launch a large instance:
   - Go to **EC2** → **Launch instance**
   - Try to select `m5.2xlarge`
   - It should be denied by SCP

3. Try to create a bucket in restricted region:
   - Go to **S3** → **Create bucket**
   - Change region to `eu-west-1`
   - Try to create bucket
   - Should be denied

4. Switch back to management account

---

### Part 3: Set Up Centralized CloudTrail

#### Step 13: Create S3 Bucket for CloudTrail Logs
1. Switch to **Security-Logging** account (using Switch Role)
2. Navigate to **S3**
3. Click **Create bucket**
4. Configure bucket:
   - **Bucket name:** `org-cloudtrail-logs-[account-id]`
   - **Region:** us-east-1
   - **Block all public access:** Keep checked
   - **Bucket Versioning:** Enable
   - **Encryption:** SSE-S3
5. Click **Create bucket**

#### Step 14: Add Bucket Policy for CloudTrail
1. Click on the bucket
2. Go to **Permissions** tab
3. Click **Bucket policy** → **Edit**
4. Add this policy (replace `SECURITY-ACCOUNT-ID` and `ORG-ID`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs-SECURITY-ACCOUNT-ID"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::org-cloudtrail-logs-SECURITY-ACCOUNT-ID/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
```

5. Click **Save changes**

#### Step 15: Create Organization Trail
1. Switch back to **Management Account**
2. Navigate to **CloudTrail**
3. Click **Create trail**
4. Configure trail:
   - **Trail name:** `OrganizationTrail`
   - **Enable for all accounts in my organization:** Check this box
   - **Storage location:** Use existing S3 bucket
   - **S3 bucket:** `org-cloudtrail-logs-[security-account-id]`
   - **Log file SSE-KMS encryption:** Disable (or configure KMS)
   - **Log file validation:** Enable
   - **SNS notification delivery:** Disable (or configure)
   - **CloudWatch Logs:** Disable (or configure)
5. Click **Next**
6. Choose log events:
   - **Management events:** Read and Write
   - **Data events:** Configure S3 and Lambda events if desired
   - **Insights events:** Enable
7. Click **Next**
8. Review and click **Create trail**

#### Step 16: Verify CloudTrail Logging
1. Wait 15 minutes for logs to appear
2. Switch to **Security-Logging** account
3. Navigate to **S3** → `org-cloudtrail-logs-[account-id]`
4. Browse folders:
   ```
   AWSLogs/
     [ORG-ID]/
       [ACCOUNT-ID]/
         CloudTrail/
           [REGION]/
             [YEAR]/
               [MONTH]/
                 [DAY]/
   ```
5. Download and view a log file (gzip compressed JSON)
6. Verify logs from multiple accounts are present

---

### Part 4: Configure Consolidated Billing

#### Step 17: View Consolidated Billing Dashboard
1. In **Management Account**, navigate to **Billing and Cost Management**
2. Click **Bills** in left sidebar
3. View consolidated bill showing:
   - Total charges for organization
   - Breakdown by linked account
   - Charges by service

#### Step 18: Set Up Cost Allocation Tags
1. Go to **Billing** → **Cost allocation tags**
2. Click **AWS generated cost allocation tags** tab
3. Activate tags:
   - `aws:createdBy`
   - `aws:cloudformation:stack-name`
4. Click **User-defined cost allocation tags** tab
5. Activate custom tags:
   - `Environment`
   - `Project`
   - `Owner`
   - `CostCenter`
6. Click **Activate**

**Note:** Tags take 24 hours to appear in reports

#### Step 19: Create Cost and Usage Reports
1. Go to **Billing** → **Cost & Usage Reports**
2. Click **Create report**
3. Configure report:
   - **Report name:** `OrganizationCostReport`
   - **Additional report details:** Include resource IDs
   - **Data refresh settings:** Automatically refresh
4. Click **Next**
5. Configure delivery:
   - **S3 bucket:** Create new or select existing
   - **Report path prefix:** `cost-reports/`
   - **Time granularity:** Daily
   - **Report versioning:** Overwrite existing report
   - **Enable report data integration for:** Amazon Athena
   - **Compression type:** GZIP
   - **Format:** text/csv
6. Click **Next**
7. Review and click **Create report**

#### Step 20: Set Up Cost Explorer
1. Go to **Billing** → **Cost Explorer**
2. If not enabled, click **Enable Cost Explorer**
3. Wait a few moments for activation
4. Click **Launch Cost Explorer**
5. Explore different views:
   - **Cost and usage** by service
   - **Cost and usage** by linked account
   - **Monthly costs** by service
   - **Daily costs**

#### Step 21: Create Custom Cost Report
1. In Cost Explorer, click **New report**
2. Configure report:
   - **Report name:** `DataEngineering-Costs`
   - **Time range:** Last 3 months
   - **Granularity:** Monthly
   - **Group by:** Service
3. Add filters:
   - **Dimension:** Linked Account
   - Select dev and prod data engineering accounts
4. Click **Apply**
5. View visualization
6. Click **Save** to save the report

---

### Part 5: Create Budget Alerts

#### Step 22: Create Organization-Wide Budget
1. Go to **Billing** → **Budgets**
2. Click **Create budget**
3. Choose budget type:
   - Select **Cost budget**
4. Click **Next**
5. Set budget details:
   - **Name:** `Organization-Monthly-Budget`
   - **Period:** Monthly
   - **Budget effective dates:** Recurring budget
   - **Start month:** Current month
   - **Budgeting method:** Fixed
   - **Enter your budgeted amount:** $500 (adjust as needed)
6. **Budget scope:**
   - **Filter:** All AWS services
7. Click **Next**

#### Step 23: Configure Budget Alerts
1. Alert #1 - 80% Threshold:
   - **Threshold:** 80% of budgeted amount
   - **Trigger:** Actual costs
   - **Email recipients:** your-email@example.com
2. Click **Add alert threshold**

3. Alert #2 - 100% Threshold:
   - **Threshold:** 100% of budgeted amount
   - **Trigger:** Actual costs
   - **Email recipients:** your-email@example.com
4. Click **Add alert threshold**

5. Alert #3 - Forecast:
   - **Threshold:** 110% of budgeted amount
   - **Trigger:** Forecasted costs
   - **Email recipients:** your-email@example.com
6. Click **Next**

#### Step 24: Review and Create Budget
1. Review all settings
2. Click **Create budget**
3. Confirm SNS subscriptions in your email

#### Step 25: Create Account-Specific Budget
1. Click **Create budget** again
2. Choose **Cost budget**
3. Set budget details:
   - **Name:** `Prod-Account-Budget`
   - **Period:** Monthly
   - **Budgeted amount:** $200
4. **Budget scope:**
   - **Filter:** Linked account
   - **Choose accounts:** Select production account
5. Configure alerts at 75%, 90%, 100%
6. Create budget

#### Step 26: Create Service-Specific Budget
1. Click **Create budget**
2. Choose **Cost budget**
3. Set budget details:
   - **Name:** `S3-Storage-Budget`
   - **Period:** Monthly
   - **Budgeted amount:** $100
4. **Budget scope:**
   - **Filter:** Service
   - **Choose services:** Amazon Simple Storage Service
5. Configure alerts
6. Create budget

#### Step 27: Create Usage Budget for Lambda
1. Click **Create budget**
2. Choose **Usage budget**
3. Set budget details:
   - **Name:** `Lambda-Invocation-Budget`
   - **Period:** Monthly
   - **Usage type:** Lambda - Invocations
   - **Usage unit:** Number of invocations
   - **Budgeted amount:** 1000000 (1 million invocations)
4. Configure alerts at 80% and 100%
5. Create budget

---

### Verification Checklist

- [ ] AWS Organization created
- [ ] Multiple OUs created (Production, Development, Security, etc.)
- [ ] Member accounts created
- [ ] Accounts moved to appropriate OUs
- [ ] Service Control Policies created
- [ ] SCPs attached to OUs
- [ ] SCP restrictions tested and verified
- [ ] Central logging S3 bucket created
- [ ] Organization CloudTrail configured
- [ ] CloudTrail logs appearing in S3
- [ ] Consolidated billing dashboard viewed
- [ ] Cost allocation tags activated
- [ ] Cost and Usage Report created
- [ ] Cost Explorer enabled
- [ ] Organization-wide budget created
- [ ] Budget alerts configured
- [ ] Multiple budget types created (cost, usage)

### Architecture Benefits

**Governance:**
- Centralized policy management with SCPs
- Consistent security controls across accounts
- Automated compliance enforcement

**Security:**
- Centralized audit logging
- Immutable log storage
- Cross-account CloudTrail visibility

**Cost Management:**
- Consolidated billing with volume discounts
- Cost visibility by account and service
- Proactive budget alerts

**Operational Efficiency:**
- Simplified account management
- Standardized account creation
- Cross-account access with AssumeRole

**Compliance:**
- Audit trail of all API calls
- Enforced encryption policies
- Regional restrictions
- Root user access prevention

### Cost Analysis

**Monthly Costs:**
- **CloudTrail:** $2 for first trail (free) + $0.10/100,000 events
- **S3 Storage (logs):** ~$5-10/month (depends on activity)
- **Member Accounts:** No charge for accounts themselves
- **Cost Explorer:** Free (enabled by default)
- **Budgets:** First 2 budgets free, $0.02/day per additional budget
- **Total estimated:** $50-70/month for 3-4 accounts

**Cost Savings:**
- **Consolidated billing:** 5-10% volume discounts
- **Reserved Instance sharing:** Share RIs across accounts
- **Savings Plans:** Organization-wide sharing
- **Data transfer:** Free between accounts in same region

---

### Cleanup (Important!)

#### Exercise 11.1 Cleanup:
1. **Delete Dashboard:**
   - CloudWatch → Dashboards → Delete
2. **Delete Alarms:**
   - CloudWatch → Alarms → Select all → Delete
3. **Delete Contributor Insights Rules:**
   - CloudWatch → Contributor Insights → Delete rules
4. **Delete Lambda Function:**
   - Lambda → Functions → Delete
5. **Delete S3 Bucket:**
   - S3 → Empty → Delete
6. **Delete SNS Topic:**
   - SNS → Topics → Delete
7. **Delete IAM Role:**
   - IAM → Roles → Delete

#### Exercise 11.2 Cleanup:
1. **Delete Stacks:**
   - CloudFormation → Stacks → Delete `data-pipeline-dev`
   - Delete `data-pipeline-prod`
2. **Delete StackSets:**
   - CloudFormation → StackSets → Delete stack instances first, then StackSet
3. **Verify Resources:**
   - Check S3, Lambda, DynamoDB, SNS to ensure all resources deleted

#### Exercise 11.3 Cleanup:
**Warning:** Deleting an organization is a multi-step process

1. **Remove Member Accounts:**
   - Organizations → AWS accounts → Select member account
   - Actions → Remove from organization
   - Repeat for all member accounts
2. **Disable CloudTrail:**
   - CloudTrail → Trails → Stop logging → Delete trail
3. **Delete Budgets:**
   - Billing → Budgets → Delete all budgets
4. **Detach SCPs:**
   - Organizations → Policies → Detach all SCPs from OUs
5. **Delete SCPs:**
   - Organizations → Policies → Delete custom SCPs
6. **Delete OUs:**
   - Organizations → AWS accounts → Delete OUs (empty them first)
7. **Delete Organization:**
   - Organizations → Settings → Delete organization

---

## Summary

### What You've Learned

**Exercise 11.1 - CloudWatch Monitoring:**
- Implemented custom CloudWatch metrics in Lambda
- Created multiple alarm types (threshold, anomaly, composite, metric math)
- Built comprehensive dashboards with 7+ widget types
- Used Logs Insights for log analysis
- Set up Contributor Insights for identifying top contributors
- Configured SNS notifications for alerts

**Exercise 11.2 - CloudFormation IaC:**
- Authored CloudFormation templates with parameters and conditions
- Deployed infrastructure across multiple environments
- Used change sets for safe, reviewable updates
- Detected and remediated configuration drift
- Implemented StackSets for multi-account deployments
- Managed infrastructure lifecycle with IaC

**Exercise 11.3 - AWS Organizations:**
- Created multi-account organization structure
- Implemented hierarchical OUs for governance
- Enforced Service Control Policies (SCPs)
- Set up centralized CloudTrail logging
- Configured consolidated billing
- Created proactive budget alerts

### Key Takeaways

1. **Monitoring is Multi-Layered:**
   - Metrics for quantitative data
   - Logs for troubleshooting
   - Alarms for proactive alerts
   - Dashboards for visibility

2. **Infrastructure as Code Provides:**
   - Consistency across environments
   - Version control for infrastructure
   - Safe change management
   - Automated drift detection

3. **Organizations Enable:**
   - Centralized governance
   - Policy-based controls
   - Cost optimization through consolidation
   - Simplified compliance

4. **SCPs are Powerful:**
   - Guardrails that can't be bypassed
   - Inherited through OU hierarchy
   - Essential for multi-account security

5. **Monitoring Costs Scale:**
   - Standard metrics are cheaper than high-resolution
   - Logs Insights charges by data scanned
   - Dashboards have fixed monthly cost
   - Contributor Insights per rule

### Real-World Applications

**Monitoring:**
- **Financial Services:** Real-time anomaly detection for fraud
- **Healthcare:** Compliance monitoring for HIPAA
- **E-commerce:** Performance monitoring during peak sales
- **Data Engineering:** Pipeline health and SLA tracking

**Infrastructure as Code:**
- **Startups:** Rapid multi-environment deployment
- **Enterprises:** Standardized infrastructure patterns
- **DevOps Teams:** GitOps workflows
- **Disaster Recovery:** Automated infrastructure rebuild

**Organizations:**
- **Multi-Team Companies:** Separate accounts per team
- **Regulated Industries:** Centralized audit logs
- **Cost-Conscious Orgs:** Consolidated billing discounts
- **Global Companies:** Regional account segregation

### Cost Optimization Tips

1. **CloudWatch:**
   - Use standard metrics (1-minute) instead of high-resolution for non-critical metrics
   - Set appropriate log retention periods
   - Use metric filters instead of scanning all logs
   - Consolidate dashboards to stay under limits

2. **CloudFormation:**
   - Delete unused stacks promptly
   - Use parameters to share templates across environments
   - Leverage AWS::NoValue for conditional resources
   - Monitor stack drift to prevent cost surprises

3. **Organizations:**
   - Share Reserved Instances across accounts
   - Use Savings Plans at organization level
   - Enable cost allocation tags early
   - Set budgets with forecasting
   - Review consolidated bill for shared service charges

### Best Practices

**CloudWatch:**
- Use custom metrics sparingly (they cost $0.30/month each)
- Implement composite alarms to reduce noise
- Set up anomaly detection for dynamic workloads
- Use Logs Insights for complex queries instead of exporting
- Create dashboards per team/service for focused visibility

**CloudFormation:**
- Use parameters for environment-specific values
- Implement conditions for optional resources
- Always use change sets for production updates
- Enable termination protection on critical stacks
- Tag resources consistently
- Document template parameters clearly
- Use nested stacks for complex architectures

**Organizations:**
- Design OU structure based on your governance model
- Test SCPs in development accounts first
- Use organization trail for all audit logging
- Enable cost allocation tags before deploying resources
- Set up budgets with forecasting
- Regularly review member account activity
- Implement least-privilege SCPs

### Next Steps

**Advanced Monitoring:**
- Implement AWS X-Ray for distributed tracing
- Set up CloudWatch Application Insights
- Configure CloudWatch Synthetics for endpoint monitoring
- Use CloudWatch RUM for real user monitoring
- Integrate with third-party monitoring tools (DataDog, New Relic)

**Advanced IaC:**
- Explore AWS CDK for programmatic infrastructure
- Implement AWS Service Catalog for self-service
- Use CloudFormation macros for advanced templating
- Set up CI/CD for infrastructure deployments
- Implement automated compliance scanning

**Advanced Governance:**
- Set up AWS Control Tower for automated account factory
- Implement AWS Config for continuous compliance
- Use AWS Security Hub for centralized security findings
- Set up AWS Systems Manager for cross-account management
- Implement AWS Resource Access Manager for resource sharing

---

## Exam Alignment

**DEA-C01 Coverage:**

**Domain 1: Data Ingestion and Transformation (34%)**
- CloudWatch monitoring for data pipelines
- Lambda metrics and logging
- Pipeline health monitoring

**Domain 2: Data Store Management (26%)**
- CloudFormation for database infrastructure
- Automated resource provisioning
- Multi-environment data stores

**Domain 3: Data Operations and Support (22%)**
- CloudWatch Logs Insights for troubleshooting
- Drift detection and remediation
- Centralized logging with CloudTrail
- Performance monitoring and optimization

**Domain 4: Data Security and Governance (24%)**
- Service Control Policies for data access governance
- Centralized audit logging
- Encryption enforcement policies
- Multi-account security boundaries
- Cost allocation and chargeback
- Compliance monitoring

### Practice Scenarios

**Scenario 1: Pipeline Failure Detection**
You have a data pipeline processing files from S3. How would you detect failures within 5 minutes?
- Create CloudWatch alarm on Lambda Errors metric
- Set threshold to 1 error in 5-minute period
- Configure SNS notification to PagerDuty/email
- Implement custom metric for processing success rate

**Scenario 2: Multi-Environment Deployment**
You need to deploy identical infrastructure to dev, staging, and prod with different configurations.
- Use CloudFormation template with parameters
- Create separate stacks per environment
- Use conditions for environment-specific resources
- Implement change sets for controlled updates

**Scenario 3: Cost Overrun Prevention**
A development team's AWS costs are growing unexpectedly.
- Create SCP to restrict expensive instance types
- Set up budget alerts at 75%, 90%, 100%
- Enable detailed billing with cost allocation tags
- Implement automated resource cleanup policies

**Scenario 4: Compliance Requirement**
Auditors require visibility into all API calls across 50 AWS accounts.
- Create AWS Organization
- Configure organization CloudTrail
- Store logs in dedicated security account
- Set up Athena for log querying

---

**Total Cost Summary:**
- Exercise 11.1: $15-20/month (CloudWatch monitoring)
- Exercise 11.2: $50-80/month (resources across environments)
- Exercise 11.3: $50-70/month (CloudTrail, multi-account setup)
- **Total: $120-150/month**

**Estimated Time Investment:** 5 hours

---

*This guide is part of the AWS Data Engineer Associate certification preparation series.*

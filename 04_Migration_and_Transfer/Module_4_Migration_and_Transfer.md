# MODULE 4: MIGRATION AND TRANSFER SERVICES FOR DATA ENGINEERING

## MODULE OVERVIEW

**Duration:** 6-8 hours  
**Difficulty:** Intermediate-Advanced  
**Prerequisites:** Modules 1-3, Understanding of databases and networking  
**Cost:** $50-150 (temporary migration resources)  

---

## WHAT YOU WILL LEARN

### AWS Migration and Transfer Services

This module covers the complete suite of AWS migration and transfer services used in data engineering workflows. You'll learn how to migrate on-premises databases, transfer large datasets, and orchestrate complex migration projects.

**Key Services:**
- ✅ **AWS Database Migration Service (DMS)** - Migrate databases with minimal downtime
- ✅ **AWS DataSync** - Automate data transfer between on-premises and AWS
- ✅ **AWS Transfer Family** - Managed SFTP/FTPS/FTP for S3
- ✅ **AWS Snow Family** - Physical data transfer (Snowcone, Snowball, Snowmobile)
- ✅ **AWS Application Discovery Service** - Discover on-premises resources
- ✅ **AWS Migration Hub** - Track migration progress across services

### Production Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│           Migration & Transfer Architecture                │
└─────────────────────────────────────────────────────────────┘

On-Premises Data Center
├─ Oracle Database (5TB) ──────→ DMS ────→ Aurora PostgreSQL
├─ File Servers (100TB) ───────→ DataSync ──→ S3
├─ SFTP Server (daily files) ──→ Transfer Family ──→ S3
└─ Petabyte Archive ───────────→ Snowball ──→ S3 Glacier

Migration Timeline:
├─ Week 1: Discovery (Application Discovery Service)
├─ Week 2: Planning (Migration Hub)
├─ Week 3-4: Database migration (DMS)
├─ Week 5-6: File transfer (DataSync/Snowball)
└─ Week 7: Cutover and validation
```

---

## EXERCISE 4.1: DATABASE MIGRATION WITH AWS DMS

### Exercise Overview

**Purpose:** Migrate a MySQL database to Aurora MySQL using DMS with minimal downtime  
**Real-World Use Case:** You're migrating an e-commerce database from on-premises MySQL to Aurora for better performance and cost savings  
**Services Used:** DMS, RDS, Aurora  
**Estimated Duration:** 90 minutes  

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              DMS Migration Architecture                     │
└─────────────────────────────────────────────────────────────┘

Source: MySQL 8.0 (On-Premises or EC2)
├─ Database: ecommerce
├─ Size: 50 GB
├─ Tables: customers, orders, products (100K+ rows)
└─ Replication: Binary logs enabled

          ↓ (Full Load + CDC)

DMS Replication Instance
├─ Instance class: dms.t3.medium
├─ Storage: 100 GB
├─ Engine version: 3.4.7
└─ VPC: Private subnet

          ↓

Target: Aurora MySQL 3.0 (AWS)
├─ Cluster: prod-aurora-mysql
├─ Writer instance: db.r6g.large
├─ Read replicas: 2
└─ Replication lag: < 1 second
```

### Step-by-Step Implementation

**Step 1: Create Source MySQL Database (Simulated)**

```bash
# Launch EC2 instance with MySQL
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name my-key \
  --security-group-ids sg-mysql-source \
  --subnet-id subnet-12345 \
  --user-data '#!/bin/bash
    yum update -y
    yum install -y mysql-server
    systemctl start mysqld
    systemctl enable mysqld'

# Connect and create database
mysql -u root -p

CREATE DATABASE ecommerce;
USE ecommerce;

-- Create sample tables
CREATE TABLE customers (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(50),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE products (
    product_id INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(255),
    category VARCHAR(100),
    price DECIMAL(10,2),
    stock_quantity INT
);

-- Insert sample data (100K rows)
DELIMITER $$
CREATE PROCEDURE populate_customers(IN num_rows INT)
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= num_rows DO
        INSERT INTO customers (email, name) 
        VALUES (CONCAT('customer', i, '@example.com'), CONCAT('Customer ', i));
        SET i = i + 1;
    END WHILE;
END$$
DELIMITER ;

CALL populate_customers(100000);

-- Enable binary logging for CDC
-- Add to /etc/my.cnf:
-- log-bin=mysql-bin
-- binlog-format=ROW
-- server-id=1

-- Restart MySQL
systemctl restart mysqld
```

**Step 2: Create Target Aurora MySQL Cluster**

```bash
# Create Aurora MySQL cluster
aws rds create-db-cluster \
  --db-cluster-identifier prod-aurora-mysql \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.02.0 \
  --master-username admin \
  --master-user-password 'YourSecurePassword123!' \
  --database-name ecommerce \
  --db-subnet-group-name private-subnet-group \
  --vpc-security-group-ids sg-aurora-target \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --storage-encrypted \
  --kms-key-id alias/aurora-mysql

# Create writer instance
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-mysql-writer \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --db-cluster-identifier prod-aurora-mysql

# Create read replicas (2)
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-mysql-reader-1 \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --db-cluster-identifier prod-aurora-mysql

aws rds create-db-instance \
  --db-instance-identifier prod-aurora-mysql-reader-2 \
  --db-instance-class db.r6g.large \
  --engine aurora-mysql \
  --db-cluster-identifier prod-aurora-mysql

# Wait for cluster to become available
aws rds wait db-cluster-available \
  --db-cluster-identifier prod-aurora-mysql
```

**Step 3: Create DMS Replication Instance**

```bash
# Create DMS subnet group
aws dms create-replication-subnet-group \
  --replication-subnet-group-identifier dms-subnet-group \
  --replication-subnet-group-description "DMS subnet group for private subnets" \
  --subnet-ids subnet-12345 subnet-67890

# Create DMS replication instance
aws dms create-replication-instance \
  --replication-instance-identifier mysql-aurora-migration \
  --replication-instance-class dms.t3.medium \
  --allocated-storage 100 \
  --vpc-security-group-ids sg-dms-instance \
  --replication-subnet-group-identifier dms-subnet-group \
  --multi-az \
  --engine-version 3.4.7 \
  --publicly-accessible false

# Wait for replication instance to become available
aws dms wait replication-instance-available \
  --filters Name=replication-instance-id,Values=mysql-aurora-migration
```

**Step 4: Create DMS Endpoints**

```bash
# Create source endpoint (MySQL)
aws dms create-endpoint \
  --endpoint-identifier mysql-source-endpoint \
  --endpoint-type source \
  --engine-name mysql \
  --server-name 10.0.1.100 \
  --port 3306 \
  --database-name ecommerce \
  --username dms_user \
  --password 'YourPassword123!' \
  --extra-connection-attributes "cleanSourceMetadataOnMismatch=true"

# Create target endpoint (Aurora MySQL)
aws dms create-endpoint \
  --endpoint-identifier aurora-target-endpoint \
  --endpoint-type target \
  --engine-name aurora \
  --server-name prod-aurora-mysql.cluster-xyz.us-east-1.rds.amazonaws.com \
  --port 3306 \
  --database-name ecommerce \
  --username admin \
  --password 'YourSecurePassword123!'

# Test endpoints
aws dms test-connection \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:mysql-aurora-migration \
  --endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:mysql-source-endpoint

aws dms test-connection \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:mysql-aurora-migration \
  --endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:aurora-target-endpoint
```

**Step 5: Create DMS Replication Task (Full Load + CDC)**

```json
// table-mappings.json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-all-tables",
      "object-locator": {
        "schema-name": "ecommerce",
        "table-name": "%"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "transformation",
      "rule-id": "2",
      "rule-name": "add-prefix",
      "object-locator": {
        "schema-name": "ecommerce",
        "table-name": "%"
      },
      "rule-target": "table",
      "rule-action": "add-prefix",
      "value": "migrated_"
    }
  ]
}
```

```bash
# Create replication task
aws dms create-replication-task \
  --replication-task-identifier mysql-aurora-migration-task \
  --source-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:mysql-source-endpoint \
  --target-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:aurora-target-endpoint \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:mysql-aurora-migration \
  --migration-type full-load-and-cdc \
  --table-mappings file://table-mappings.json \
  --replication-task-settings '{
    "TargetMetadata": {
      "TargetSchema": "ecommerce",
      "SupportLobs": true,
      "FullLobMode": false,
      "LobChunkSize": 64,
      "LimitedSizeLobMode": true,
      "LobMaxSize": 32
    },
    "FullLoadSettings": {
      "TargetTablePrepMode": "DROP_AND_CREATE",
      "MaxFullLoadSubTasks": 8,
      "TransactionConsistencyTimeout": 600,
      "CommitRate": 10000
    },
    "Logging": {
      "EnableLogging": true,
      "LogComponents": [
        {"Id": "SOURCE_CAPTURE", "Severity": "LOGGER_SEVERITY_INFO"},
        {"Id": "TARGET_APPLY", "Severity": "LOGGER_SEVERITY_INFO"},
        {"Id": "TASK_MANAGER", "Severity": "LOGGER_SEVERITY_INFO"}
      ]
    },
    "ChangeProcessingDdlHandlingPolicy": {
      "HandleSourceTableDropped": true,
      "HandleSourceTableTruncated": true,
      "HandleSourceTableAltered": true
    }
  }'

# Start replication task
aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:us-east-1:123456789012:task:mysql-aurora-migration-task \
  --start-replication-task-type start-replication
```

**Step 6: Monitor Migration Progress**

```bash
# Monitor task status
watch -n 10 'aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=arn:aws:dms:...:task:mysql-aurora-migration-task \
  --query "ReplicationTasks[0].[Status,ReplicationTaskStats]"'

# Check table statistics
aws dms describe-table-statistics \
  --replication-task-arn arn:aws:dms:us-east-1:123456789012:task:mysql-aurora-migration-task \
  --query 'TableStatistics[*].[TableName,Inserts,Deletes,Updates,FullLoadRows,ValidationState]' \
  --output table

# Monitor CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/DMS \
  --metric-name CDCLatencySource \
  --dimensions Name=ReplicationTaskIdentifier,Value=mysql-aurora-migration-task \
  --start-time 2024-06-22T00:00:00Z \
  --end-time 2024-06-22T23:59:59Z \
  --period 300 \
  --statistics Average,Maximum
```

**Step 7: Validation**

```bash
# Connect to Aurora and verify data
mysql -h prod-aurora-mysql.cluster-xyz.us-east-1.rds.amazonaws.com \
      -u admin -p ecommerce

-- Check row counts
SELECT 'customers' AS table_name, COUNT(*) AS row_count FROM customers
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'products', COUNT(*) FROM products;

-- Compare checksums (source vs target)
-- Source MySQL:
SELECT table_name, 
       MD5(GROUP_CONCAT(customer_id, email, name ORDER BY customer_id)) AS checksum
FROM customers;

-- Target Aurora:
SELECT table_name,
       MD5(GROUP_CONCAT(customer_id, email, name ORDER BY customer_id)) AS checksum
FROM customers;

-- Verify CDC is working (make change on source)
-- Source:
INSERT INTO customers (email, name) VALUES ('test@example.com', 'Test User');

-- Wait 5 seconds, then check target
SELECT * FROM customers WHERE email = 'test@example.com';
-- Should appear on target (CDC replication)
```

**Step 8: Cutover**

```python
# cutover_script.py
import boto3
import pymysql
import time

dms = boto3.client('dms')

def execute_cutover():
    """
    Cutover from MySQL source to Aurora target
    Downtime: 2-5 minutes
    """
    
    print("[1/6] Stopping application writes to source MySQL...")
    # Stop application servers or set database to read-only
    
    print("[2/6] Waiting for DMS to sync final changes...")
    # Monitor CDC latency
    while True:
        response = dms.describe-replication-tasks(
            Filters=[{'Name': 'replication-task-arn', 'Values': ['arn:aws:dms:...']}]
        )
        
        cdc_latency = response['ReplicationTasks'][0]['ReplicationTaskStats'].get('CDCLatencySource', 0)
        
        if cdc_latency < 1:  # < 1 second lag
            print(f"[2/6] CDC latency: {cdc_latency}s - Ready for cutover")
            break
        
        print(f"[2/6] CDC latency: {cdc_latency}s - Waiting...")
        time.sleep(5)
    
    print("[3/6] Stopping DMS replication task...")
    dms.stop_replication_task(
        ReplicationTaskArn='arn:aws:dms:us-east-1:123456789012:task:mysql-aurora-migration-task'
    )
    
    print("[4/6] Final validation...")
    # Compare row counts, checksums
    
    print("[5/6] Updating application connection strings to Aurora...")
    # Update environment variables, config files to point to Aurora endpoint
    
    print("[6/6] Starting applications with Aurora backend...")
    # Start application servers
    
    print("✅ Cutover complete! Application now using Aurora MySQL.")

if __name__ == "__main__":
    execute_cutover()
```

### Results

| Metric | Value |
|--------|-------|
| **Database Size** | 50 GB (100K customers, 500K orders, 10K products) |
| **Full Load Time** | 45 minutes |
| **CDC Lag** | < 1 second |
| **Cutover Downtime** | 3 minutes |
| **Migration Success** | 100% (all rows migrated) |
| **Cost** | $25 (DMS instance for 2 days) |

---

## EXERCISE 4.2: FILE TRANSFER WITH AWS DATASYNC

### Exercise Overview

**Purpose:** Automate ongoing file transfer from on-premises NFS server to S3  
**Real-World Use Case:** Sync 10 TB of daily log files from on-premises data center to S3 for analysis  
**Services Used:** DataSync, S3, CloudWatch  
**Estimated Duration:** 60 minutes  

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                 DataSync Architecture                       │
└─────────────────────────────────────────────────────────────┘

On-Premises NFS Server
├─ Location: 192.168.1.100:/exports/logs
├─ Size: 10 TB (daily logs)
├─ Files: 50,000 log files
└─ Network: 1 Gbps link to AWS

          ↓ (via DataSync Agent)

DataSync Agent (On-Premises VM)
├─ Hypervisor: VMware ESXi
├─ CPU: 4 vCPUs
├─ Memory: 32 GB RAM
└─ Network: Connected to both on-prem and AWS VPC

          ↓ (over AWS PrivateLink or Internet)

AWS DataSync Service
├─ Task: nfs-to-s3-daily-logs
├─ Schedule: Daily at 2 AM
├─ Transfer rate: 100 MB/s
└─ Verification: Full file checksum

          ↓

S3 Bucket: company-logs-archive
├─ Storage class: S3 Standard
├─ Lifecycle: → S3-IA after 30 days
└─ Encryption: SSE-KMS
```

### Step-by-Step Implementation

**Step 1: Deploy DataSync Agent (On-Premises)**

```bash
# Download DataSync agent OVA
# From AWS Console: DataSync → Agents → Deploy agent
# https://s3.amazonaws.com/aws-datasync-agent/latest/datasync-agent.ova

# Deploy to VMware ESXi
# VM Configuration:
# - 4 vCPUs
# - 32 GB RAM
# - 80 GB disk
# - Network: Connected to on-prem network with route to AWS

# After boot, get activation key from agent console
# Access: https://<agent-ip>
# Copy activation key

# Activate agent
aws datasync create-agent \
  --agent-name on-prem-datasync-agent \
  --activation-key ABCDE-12345-FGHIJ-67890 \
  --vpc-endpoint-id vpce-12345 \
  --subnet-arns arn:aws:ec2:us-east-1:123456789012:subnet/subnet-12345 \
  --security-group-arns arn:aws:ec2:us-east-1:123456789012:security-group/sg-datasync
```

**Step 2: Create Source Location (NFS)**

```bash
# Create NFS source location
aws datasync create-location-nfs \
  --server-hostname 192.168.1.100 \
  --subdirectory /exports/logs \
  --on-prem-config '{
    "AgentArns": ["arn:aws:datasync:us-east-1:123456789012:agent/agent-12345"]
  }' \
  --mount-options '{
    "Version": "NFS3"
  }'
```

**Step 3: Create Destination Location (S3)**

```bash
# Create S3 bucket
aws s3 mb s3://company-logs-archive

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket company-logs-archive \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket company-logs-archive \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abc-123"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Create S3 location
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::company-logs-archive \
  --s3-storage-class STANDARD \
  --s3-config '{
    "BucketAccessRoleArn": "arn:aws:iam::123456789012:role/DataSyncS3AccessRole"
  }' \
  --subdirectory /logs/$(date +%Y-%m-%d)
```

**Step 4: Create DataSync Task**

```bash
# Create DataSync task
aws datasync create-task \
  --source-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-source-nfs \
  --destination-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-dest-s3 \
  --name nfs-to-s3-daily-logs \
  --options '{
    "VerifyMode": "POINT_IN_TIME_CONSISTENT",
    "OverwriteMode": "NEVER",
    "Atime": "BEST_EFFORT",
    "Mtime": "PRESERVE",
    "Uid": "NONE",
    "Gid": "NONE",
    "PreserveDeletedFiles": "PRESERVE",
    "PreserveDevices": "NONE",
    "PosixPermissions": "NONE",
    "BytesPerSecond": 104857600,
    "TaskQueueing": "ENABLED",
    "LogLevel": "TRANSFER",
    "TransferMode": "CHANGED"
  }' \
  --excludes '[
    {"FilterType": "SIMPLE_PATTERN", "Value": "*.tmp"},
    {"FilterType": "SIMPLE_PATTERN", "Value": ".snapshot"}
  ]' \
  --schedule '{
    "ScheduleExpression": "cron(0 2 * * ? *)"
  }' \
  --cloud-watch-log-group-arn arn:aws:logs:us-east-1:123456789012:log-group:/aws/datasync
```

**Step 5: Start Task Execution**

```bash
# Start task manually (first run)
TASK_EXECUTION_ARN=$(aws datasync start-task-execution \
  --task-arn arn:aws:datasync:us-east-1:123456789012:task/task-12345 \
  --query 'TaskExecutionArn' \
  --output text)

echo "Task execution started: $TASK_EXECUTION_ARN"

# Monitor task execution
watch -n 10 'aws datasync describe-task-execution \
  --task-execution-arn '$TASK_EXECUTION_ARN' \
  --query "[Status,BytesTransferred,FilesTransferred]" \
  --output table'
```

**Step 6: Monitor with CloudWatch**

```python
# monitor_datasync.py
import boto3
from datetime import datetime, timedelta

datasync = boto3.client('datasync')
cloudwatch = boto3.client('cloudwatch')

def get_task_metrics(task_arn):
    """
    Get DataSync task metrics from CloudWatch
    """
    
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=24)
    
    # Files transferred
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/DataSync',
        MetricName='FilesTransferred',
        Dimensions=[{'Name': 'TaskId', 'Value': task_arn.split('/')[-1]}],
        StartTime=start_time,
        EndTime=end_time,
        Period=3600,
        Statistics=['Sum']
    )
    
    total_files = sum([dp['Sum'] for dp in response['Datapoints']])
    print(f"Files transferred (24h): {total_files:,.0f}")
    
    # Bytes transferred
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/DataSync',
        MetricName='BytesTransferred',
        Dimensions=[{'Name': 'TaskId', 'Value': task_arn.split('/')[-1]}],
        StartTime=start_time,
        EndTime=end_time,
        Period=3600,
        Statistics=['Sum']
    )
    
    total_bytes = sum([dp['Sum'] for dp in response['Datapoints']])
    print(f"Data transferred (24h): {total_bytes / 1024**4:.2f} TB")
    
    # Transfer rate
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/DataSync',
        MetricName='BytesPerSecond',
        Dimensions=[{'Name': 'TaskId', 'Value': task_arn.split('/')[-1]}],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average', 'Maximum']
    )
    
    if response['Datapoints']:
        avg_rate = sum([dp['Average'] for dp in response['Datapoints']]) / len(response['Datapoints'])
        max_rate = max([dp['Maximum'] for dp in response['Datapoints']])
        
        print(f"Average transfer rate: {avg_rate / 1024**2:.2f} MB/s")
        print(f"Peak transfer rate: {max_rate / 1024**2:.2f} MB/s")

if __name__ == "__main__":
    task_arn = "arn:aws:datasync:us-east-1:123456789012:task/task-12345"
    get_task_metrics(task_arn)
```

### Results

| Metric | Value |
|--------|-------|
| **Data Transferred** | 10 TB (50,000 files) |
| **Transfer Time** | 24 hours (scheduled daily) |
| **Average Speed** | 100 MB/s (limited by network) |
| **Files Verified** | 100% (checksum validation) |
| **Cost** | $200/month (data transfer + DataSync charges) |
| **Automation** | Daily at 2 AM (cron schedule) |

---

## EXERCISE 4.3: SFTP FILE INGESTION WITH AWS TRANSFER FAMILY

### Exercise Overview

**Purpose:** Set up managed SFTP server for partners to upload files to S3  
**Real-World Use Case:** Trading partners upload daily CSV files via SFTP, automatically processed by Lambda  
**Services Used:** AWS Transfer Family, S3, Lambda, EventBridge  
**Estimated Duration:** 45 minutes  

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│           AWS Transfer Family Architecture                  │
└─────────────────────────────────────────────────────────────┘

Trading Partners (External)
├─ Partner A: SFTP client
├─ Partner B: SFTP client
└─ Partner C: SFTP client
         ↓
    SFTP Connection (sftp://s-1234567890.server.transfer.us-east-1.amazonaws.com)
         ↓
AWS Transfer for SFTP
├─ Endpoint: Public (VPC with EIP)
├─ Protocol: SFTP (SSH File Transfer Protocol)
├─ Identity: Service-managed users
└─ Home directory: /partner-uploads/${transfer:UserName}
         ↓
S3 Bucket: partner-file-uploads
├─ Folder structure:
│   ├─ partner-a/incoming/
│   ├─ partner-b/incoming/
│   └─ partner-c/incoming/
└─ Event notification: S3 → EventBridge
         ↓
EventBridge Rule
├─ Filter: *.csv files
└─ Target: Lambda function
         ↓
Lambda: process-partner-files
├─ Validate CSV format
├─ Move to /processed/ folder
└─ Trigger downstream ETL (Glue)
```

### Step-by-Step Implementation

**Step 1: Create S3 Bucket**

```bash
# Create bucket for partner uploads
aws s3 mb s3://partner-file-uploads

# Create folder structure
aws s3api put-object --bucket partner-file-uploads --key partner-a/incoming/
aws s3api put-object --bucket partner-file-uploads --key partner-a/processed/
aws s3api put-object --bucket partner-file-uploads --key partner-b/incoming/
aws s3api put-object --bucket partner-file-uploads --key partner-b/processed/
aws s3api put-object --bucket partner-file-uploads --key partner-c/incoming/
aws s3api put-object --bucket partner-file-uploads --key partner-c/processed/

# Enable EventBridge notifications
aws s3api put-bucket-notification-configuration \
  --bucket partner-file-uploads \
  --notification-configuration '{
    "EventBridgeConfiguration": {}
  }'
```

**Step 2: Create IAM Role for Transfer Family**

```json
// transfer-family-role-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "transfer.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

```json
// transfer-family-s3-access-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListingOfUserFolder",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::partner-file-uploads",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["${transfer:UserName}/*", "${transfer:UserName}"]
        }
      }
    },
    {
      "Sid": "HomeDirObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::partner-file-uploads/${transfer:UserName}/*"
    }
  ]
}
```

```bash
# Create IAM role
aws iam create-role \
  --role-name TransferFamilySFTPRole \
  --assume-role-policy-document file://transfer-family-role-trust-policy.json

aws iam put-role-policy \
  --role-name TransferFamilySFTPRole \
  --policy-name S3AccessPolicy \
  --policy-document file://transfer-family-s3-access-policy.json
```

**Step 3: Create Transfer Family Server**

```bash
# Create SFTP server
aws transfer create-server \
  --endpoint-type PUBLIC \
  --protocols SFTP \
  --identity-provider-type SERVICE_MANAGED \
  --logging-role arn:aws:iam::123456789012:role/TransferFamilyLoggingRole \
  --tags Key=Name,Value=partner-sftp-server Key=Environment,Value=production

# Get server ID
SERVER_ID=$(aws transfer list-servers \
  --query 'Servers[0].ServerId' \
  --output text)

echo "Server ID: $SERVER_ID"
echo "Endpoint: s-$SERVER_ID.server.transfer.us-east-1.amazonaws.com"
```

**Step 4: Create SFTP Users for Partners**

```bash
# Generate SSH key for partner-a
ssh-keygen -t rsa -b 4096 -f partner-a-key -N ""

# Create user for Partner A
aws transfer create-user \
  --server-id $SERVER_ID \
  --user-name partner-a \
  --role arn:aws:iam::123456789012:role/TransferFamilySFTPRole \
  --home-directory /partner-file-uploads/partner-a \
  --ssh-public-key-body "$(cat partner-a-key.pub)"

# Create user for Partner B
ssh-keygen -t rsa -b 4096 -f partner-b-key -N ""

aws transfer create-user \
  --server-id $SERVER_ID \
  --user-name partner-b \
  --role arn:aws:iam::123456789012:role/TransferFamilySFTPRole \
  --home-directory /partner-file-uploads/partner-b \
  --ssh-public-key-body "$(cat partner-b-key.pub)"

# Create user for Partner C
ssh-keygen -t rsa -b 4096 -f partner-c-key -N ""

aws transfer create-user \
  --server-id $SERVER_ID \
  --user-name partner-c \
  --role arn:aws:iam::123456789012:role/TransferFamilySFTPRole \
  --home-directory /partner-file-uploads/partner-c \
  --ssh-public-key-body "$(cat partner-c-key.pub)"
```

**Step 5: Test SFTP Upload**

```bash
# Test connection as Partner A
sftp -i partner-a-key partner-a@s-$SERVER_ID.server.transfer.us-east-1.amazonaws.com

# SFTP commands
sftp> ls
incoming  processed

sftp> cd incoming
sftp> put test-file.csv
Uploading test-file.csv to /partner-a/incoming/test-file.csv
test-file.csv                  100%   1024     1.0KB/s   00:01

sftp> ls incoming
test-file.csv

sftp> exit
```

**Step 6: Create Lambda Function to Process Files**

```python
# lambda_function.py
import boto3
import csv
import io

s3 = boto3.client('s3')
glue = boto3.client('glue')

def lambda_handler(event, context):
    """
    Process CSV files uploaded by partners via SFTP
    """
    
    # Extract S3 details from EventBridge event
    bucket = event['detail']['bucket']['name']
    key = event['detail']['object']['key']
    
    print(f"Processing: s3://{bucket}/{key}")
    
    # Determine partner from folder structure
    partner = key.split('/')[0]  # e.g., "partner-a"
    
    try:
        # Download CSV file
        response = s3.get_object(Bucket=bucket, Key=key)
        csv_content = response['Body'].read().decode('utf-8')
        
        # Validate CSV
        csv_reader = csv.DictReader(io.StringIO(csv_content))
        rows = list(csv_reader)
        
        # Data quality checks
        if len(rows) == 0:
            raise ValueError("Empty CSV file")
        
        required_columns = ['transaction_id', 'amount', 'date']
        if not all(col in csv_reader.fieldnames for col in required_columns):
            raise ValueError(f"Missing columns. Expected: {required_columns}")
        
        # Move to processed folder
        new_key = key.replace('/incoming/', '/processed/')
        s3.copy_object(
            Bucket=bucket,
            CopySource={'Bucket': bucket, 'Key': key},
            Key=new_key
        )
        
        # Delete from incoming
        s3.delete_object(Bucket=bucket, Key=key)
        
        # Trigger Glue ETL job
        glue.start_job_run(
            JobName='process-partner-data',
            Arguments={
                '--PARTNER': partner,
                '--S3_PATH': f's3://{bucket}/{new_key}'
            }
        )
        
        return {
            'statusCode': 200,
            'body': f'Successfully processed {len(rows)} rows from {partner}'
        }
        
    except Exception as e:
        # Move to DLQ
        dlq_key = key.replace('/incoming/', '/dlq/')
        s3.copy_object(
            Bucket=bucket,
            CopySource={'Bucket': bucket, 'Key': key},
            Key=dlq_key
        )
        
        print(f"Error: {str(e)}")
        raise
```

```bash
# Deploy Lambda function
zip lambda_function.zip lambda_function.py

aws lambda create-function \
  --function-name process-partner-files \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://lambda_function.zip \
  --timeout 300 \
  --memory-size 512
```

**Step 7: Create EventBridge Rule**

```bash
# Create EventBridge rule
aws events put-rule \
  --name partner-csv-upload-rule \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"],
    "detail": {
      "bucket": {"name": ["partner-file-uploads"]},
      "object": {
        "key": [{"suffix": ".csv"}]
      }
    }
  }'

# Add Lambda as target
aws events put-targets \
  --rule partner-csv-upload-rule \
  --targets "Id=1,Arn=arn:aws:lambda:us-east-1:123456789012:function:process-partner-files"

# Grant EventBridge permission to invoke Lambda
aws lambda add-permission \
  --function-name process-partner-files \
  --statement-id EventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:us-east-1:123456789012:rule/partner-csv-upload-rule
```

### Results

| Metric | Value |
|--------|-------|
| **Partners Onboarded** | 3 |
| **Files Processed/Day** | 500 |
| **Average File Size** | 10 MB |
| **Processing Latency** | 30 seconds (upload → Lambda complete) |
| **Success Rate** | 99.8% |
| **Cost** | $220/month (Transfer Family + data transfer) |

---

## INTERVIEW PREPARATION

### Beginner Questions (5)

**Q1: What is AWS Database Migration Service (DMS) and when do you use it?**

**Answer:**

AWS DMS is a managed service that helps you migrate databases to AWS quickly and securely. The source database remains fully operational during the migration, minimizing downtime.

**Use Cases:**
- **Database migration**: On-premises → AWS Cloud
- **Database consolidation**: Multiple databases → Single database
- **Database replication**: Continuous data replication for DR
- **Database upgrade**: Oracle 11g → Oracle 19c (homogeneous)
- **Database engine change**: Oracle → Aurora PostgreSQL (heterogeneous)

**Migration Types:**
1. **Full Load**: One-time migration (requires downtime)
2. **Full Load + CDC**: Minimal downtime migration (recommended)
3. **CDC Only**: Ongoing replication (for DR or analytics)

**Oracle DBA Analogy:**
- DMS is like Oracle Data Pump + GoldenGate combined
- Full Load = Data Pump export/import
- CDC = GoldenGate real-time replication

**Example:**
```bash
# Migrate 1TB Oracle database to Aurora
# Downtime: 5-10 minutes (only during cutover)
# Timeline: 24 hours full load + ongoing CDC
```

---

**Q2: What is AWS DataSync and how is it different from S3 Transfer Acceleration?**

**Answer:**

| Feature | AWS DataSync | S3 Transfer Acceleration |
|---------|-------------|-------------------------|
| **Purpose** | Automated data transfer (on-prem ↔ AWS) | Fast S3 uploads over long distances |
| **Source** | NFS, SMB, HDFS, S3, EFS, FSx | Any (application uploads to S3) |
| **Destination** | S3, EFS, FSx | S3 only |
| **Automation** | Scheduled tasks, incremental sync | Manual uploads (via SDK/CLI) |
| **Verification** | Built-in checksum validation | No automatic validation |
| **Network** | Uses dedicated DataSync agent | Uses CloudFront edge locations |
| **Use Case** | Migrate file servers, ongoing sync | Speed up single file uploads |

**When to use DataSync:**
- Migrate 10 TB of files from on-premises NFS → S3
- Schedule daily sync of log files
- One-time data center migration

**When to use Transfer Acceleration:**
- Upload large video files (5 GB) from Asia to US S3 bucket
- Speed up individual PUT requests
- No need for scheduling or automation

**Example:**
```bash
# DataSync: Automated daily sync
aws datasync start-task-execution --task-arn arn:aws:datasync:...:task/task-12345

# Transfer Acceleration: Manual upload with speed boost
aws s3 cp large-file.mp4 s3://my-bucket/ \
  --endpoint-url https://my-bucket.s3-accelerate.amazonaws.com
```

---

**Q3: What are the AWS Snow Family devices and when do you use each?**

**Answer:**

| Device | Capacity | Use Case | Cost |
|--------|----------|----------|------|
| **Snowcone** | 8 TB | Edge computing, small migrations | $60 + shipping |
| **Snowball Edge Storage Optimized** | 80 TB | Medium migrations (10-80 TB) | $300 + shipping |
| **Snowball Edge Compute Optimized** | 42 TB + 52 vCPUs | Edge processing + migration | $300 + shipping |
| **Snowmobile** | 100 PB | Massive data center migrations | Custom pricing |

**When to use Snow Family:**
- **Network too slow**: 10 TB over 100 Mbps = 9 days (use Snowball instead)
- **Limited bandwidth**: Don't want to saturate internet connection
- **One-time migration**: Data center closure, cloud migration
- **Offline locations**: Oil rigs, ships, military bases (Snowcone with offline mode)

**Calculation Example:**
```
Data to transfer: 50 TB
Internet speed: 100 Mbps

Option 1: Internet transfer
Time = 50 TB * 8 / 100 Mbps = 44 days
Cost = $0 (data transfer out costs apply)

Option 2: Snowball Edge
Time = 1 day (load) + 2 days (shipping) + 1 day (import) = 4 days
Cost = $300 + shipping

✅ Snowball is 11x faster and worth the cost
```

**Oracle DBA Analogy:**
- Snowball = Shipping hard drives between data centers
- But AWS handles encryption, tracking, and secure wipe

---

**Q4: What is AWS Transfer Family and what protocols does it support?**

**Answer:**

AWS Transfer Family is a managed file transfer service that provides SFTP, FTPS, and FTP access to S3 or EFS.

**Supported Protocols:**
1. **SFTP** (SSH File Transfer Protocol) - Most secure, recommended
2. **FTPS** (FTP over SSL/TLS) - Legacy systems that need FTP + encryption
3. **FTP** (File Transfer Protocol) - Insecure, only for internal VPC use

**Use Cases:**
- **Trading partners**: External partners upload files via SFTP
- **Legacy systems**: Mainframe systems using FTP/FTPS
- **Regulatory compliance**: SFTP required by partners/regulators
- **No code changes**: Replace existing SFTP server with AWS-managed service

**Architecture:**
```
Partner SFTP Client
    ↓
AWS Transfer for SFTP (managed endpoint)
    ↓
S3 Bucket (files stored)
    ↓
Lambda (automated processing)
```

**Benefits:**
- ✅ **Fully managed**: No servers to patch/maintain
- ✅ **Highly available**: Multi-AZ, auto-scaling
- ✅ **Secure**: SSH keys, service-managed users, IAM integration
- ✅ **Integrated**: Direct S3/EFS storage, EventBridge notifications

**Cost:**
- $0.30/hour (server uptime) = $216/month
- $0.04/GB transferred
- Example: $216 + (1TB * $0.04) = $256/month for 1TB transfer

---

**Q5: What is AWS Application Discovery Service and why use it before migration?**

**Answer:**

AWS Application Discovery Service automatically discovers on-premises servers, applications, and dependencies to help plan AWS migrations.

**Discovery Methods:**

1. **Agentless Discovery (VMware)**
   - Deploys virtual appliance in VMware environment
   - Collects: CPU, memory, disk, network
   - No agent installation needed
   - Limited to VMware VMs

2. **Agent-Based Discovery**
   - Install agent on each server (Windows/Linux)
   - Collects: System performance, running processes, network connections
   - Works on physical servers, any hypervisor
   - More detailed data

**Data Collected:**
- **Server inventory**: Hostname, IP, OS, CPU, RAM
- **Performance metrics**: CPU utilization, memory usage, disk I/O
- **Network dependencies**: Which servers talk to which (TCP connections)
- **Running processes**: Applications and services

**Why Use Before Migration:**
1. **Understand dependencies**: App server → database → storage
2. **Right-sizing**: Choose correct EC2 instance types based on utilization
3. **Group servers**: Migrate related servers together (app + DB)
4. **TCO analysis**: Compare on-prem costs vs AWS costs

**Integration with Migration Hub:**
```
Application Discovery Service (discover servers)
    ↓
Migration Hub (plan migration)
    ↓
DMS, DataSync, CloudEndure (execute migration)
```

**Oracle DBA Analogy:**
- Like running AWR/ADDM reports before migration
- Identifies resource usage and dependencies
- But automated across entire data center

**Example Output:**
```
Server: app-server-01
├─ CPU: 8 cores, 40% avg utilization → Recommend t3.xlarge
├─ RAM: 32 GB, 20 GB used → Recommend 32 GB instance
├─ Network connections:
│   ├─ db-server-01:3306 (MySQL)
│   ├─ cache-server-01:6379 (Redis)
│   └─ storage-server-01:2049 (NFS)
└─ Recommendation: Migrate together as a group
```

---

### Intermediate Questions (5)

**Q6: Explain DMS task settings: Full Load, CDC, and Full Load + CDC. When to use each?**

**Answer:**

| Mode | How It Works | Downtime | Use Case |
|------|--------------|----------|----------|
| **Full Load** | Copy all data once, then stop | Hours to days | Dev/test, data warehousing, historical data |
| **CDC (Change Data Capture)** | Replicate ongoing changes only | None (assumes target already has data) | Ongoing replication, DR, real-time analytics |
| **Full Load + CDC** | Copy all data, then replicate changes | Minutes (cutover only) | **Production migration (recommended)** |

**Full Load:**
```
Source (Oracle 1TB) → DMS copies all data → Target (Aurora 1TB) → STOP
Timeline: 10 hours
Downtime: 10 hours (application must be offline)
```

**CDC Only:**
```
Source (changes) → DMS replicates changes → Target (must already have baseline)
Timeline: Ongoing
Downtime: 0 (but target must be pre-populated)
Use: After full load completes, switch to CDC for ongoing sync
```

**Full Load + CDC (Recommended):**
```
Phase 1: Full Load (app still running on source)
Source (Oracle 1TB) → DMS → Target (Aurora)
Timeline: 10 hours (app writes continue to source)

Phase 2: CDC (catch up to real-time)
Source (new writes) → DMS → Target
CDC lag: 10 hours → 1 hour → 5 min → < 1 sec

Phase 3: Cutover (5 min downtime)
1. Stop app writes
2. Wait for CDC lag = 0
3. Point app to Aurora
4. Start app
```

**Configuration:**

```bash
# Full Load only
aws dms create-replication-task \
  --migration-type full-load

# CDC only
aws dms create-replication-task \
  --migration-type cdc

# Full Load + CDC (production)
aws dms create-replication-task \
  --migration-type full-load-and-cdc \
  --cdc-start-position "checkpoint:..."
```

**Best Practices:**
- ✅ **Production**: Full Load + CDC (minimal downtime)
- ✅ **Dev/Test**: Full Load (simpler, downtime acceptable)
- ✅ **DR/Analytics**: CDC only (after initial baseline)

---

**Q7: How do you troubleshoot slow DMS migration performance?**

**Answer:**

**Common Performance Issues:**

1. **Slow Full Load (< 100 MB/s)**

**Diagnosis:**
```bash
# Check table statistics
aws dms describe-table-statistics \
  --replication-task-arn arn:aws:dms:...:task/my-task \
  --query 'TableStatistics[*].[TableName,FullLoadRows,FullLoadCondtnlChkFailedRows]'

# Check CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/DMS \
  --metric-name FullLoadThroughputRowsSource \
  --dimensions Name=ReplicationTaskIdentifier,Value=my-task \
  --start-time 2024-06-22T00:00:00Z \
  --end-time 2024-06-22T23:59:59Z \
  --period 300 \
  --statistics Average
```

**Solutions:**
```json
// Increase parallelism
{
  "FullLoadSettings": {
    "MaxFullLoadSubTasks": 8,  // Default: 8, Max: 49
    "CommitRate": 10000        // Rows per transaction
  }
}
```

2. **High CDC Latency (> 10 seconds)**

**Diagnosis:**
```bash
# Check CDC latency
aws cloudwatch get-metric-statistics \
  --namespace AWS/DMS \
  --metric-name CDCLatencySource \
  --dimensions Name=ReplicationTaskIdentifier,Value=my-task
```

**Causes:**
- Source database has high write volume
- DMS instance undersized
- Large transactions (affects commit lag)
- Network bandwidth limitations

**Solutions:**
```bash
# 1. Increase DMS instance size
aws dms modify-replication-instance \
  --replication-instance-arn arn:aws:dms:...:rep:my-instance \
  --replication-instance-class dms.c5.4xlarge  # More CPU

# 2. Add Multi-AZ for better I/O
aws dms modify-replication-instance \
  --replication-instance-arn arn:aws:dms:...:rep:my-instance \
  --multi-az

# 3. Adjust batch settings
{
  "ChangeProcessingTuning": {
    "BatchApplyTimeoutMin": 1,
    "BatchApplyTimeoutMax": 30,
    "BatchApplyMemoryLimit": 500,
    "BatchSplitSize": 0,
    "MinTransactionSize": 1000,
    "CommitTimeout": 1
  }
}
```

3. **Table Mapping Issues**

```json
// Exclude large BLOB columns
{
  "rules": [{
    "rule-type": "transformation",
    "rule-id": "1",
    "rule-name": "exclude-lobs",
    "object-locator": {
      "schema-name": "mydb",
      "table-name": "large_table"
    },
    "rule-action": "remove-column",
    "column-name": "blob_column"
  }]
}
```

4. **Network Bandwidth**

```bash
# Check network throughput
aws cloudwatch get-metric-statistics \
  --namespace AWS/DMS \
  --metric-name NetworkTransmitThroughput \
  --dimensions Name=ReplicationInstanceIdentifier,Value=my-instance
```

**Solution:** Use VPN or Direct Connect for higher bandwidth

**Performance Checklist:**

| Item | Target | How to Check |
|------|--------|--------------|
| Full load throughput | > 100 MB/s | CloudWatch: FullLoadThroughputBytesSource |
| CDC latency | < 10 seconds | CloudWatch: CDCLatencySource |
| CPU utilization (DMS) | < 80% | CloudWatch: CPUUtilization |
| Network bandwidth | < 80% of max | CloudWatch: NetworkTransmitThroughput |
| Table errors | 0 | describe-table-statistics |

---

**Q8: What is AWS DataSync task scheduling and how do you optimize costs?**

**Answer:**

**Task Scheduling Options:**

1. **On-Demand (Manual)**
```bash
# Start task manually
aws datasync start-task-execution --task-arn arn:aws:datasync:...:task/task-12345
```

2. **Scheduled (Cron)**
```bash
# Daily at 2 AM
aws datasync create-task \
  --schedule '{
    "ScheduleExpression": "cron(0 2 * * ? *)"
  }'

# Every 4 hours
--schedule '{"ScheduleExpression": "cron(0 */4 * * ? *)"}'

# Weekdays at 6 PM
--schedule '{"ScheduleExpression": "cron(0 18 ? * MON-FRI *)"}'
```

3. **EventBridge (Event-Driven)**
```bash
# Trigger DataSync when S3 object created
aws events put-rule \
  --name trigger-datasync-on-s3-upload \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"]
  }'

aws events put-targets \
  --rule trigger-datasync-on-s3-upload \
  --targets "Id=1,Arn=arn:aws:datasync:...:task/task-12345,RoleArn=arn:aws:iam::123456789012:role/EventBridgeRole"
```

**Cost Optimization Strategies:**

1. **Transfer Mode: CHANGED vs ALL**
```bash
# Only transfer modified files (incremental)
aws datasync create-task \
  --options '{
    "TransferMode": "CHANGED",  # Only changed files
    "OverwriteMode": "NEVER"    # Skip existing files
  }'

# Saves bandwidth and costs
# Example: 10 TB initial transfer, then 100 GB daily changes
# Cost: $3,000 (initial) + $30/day (incremental)
```

2. **Bandwidth Throttling**
```bash
# Limit to 100 MB/s (don't saturate link)
aws datasync create-task \
  --options '{
    "BytesPerSecond": 104857600  # 100 MB/s = 100 * 1024 * 1024
  }'

# Prevents network congestion
# Extends transfer time but avoids affecting production traffic
```

3. **Schedule During Off-Hours**
```bash
# Transfer at night (cheaper data transfer, no user impact)
--schedule '{"ScheduleExpression": "cron(0 2 * * ? *)"}'  # 2 AM daily

# Avoid peak hours:
# - Don't run during business hours (9 AM - 5 PM)
# - Don't run during backup windows
# - Coordinate with other scheduled tasks
```

4. **Use S3 Lifecycle for Destination**
```bash
# DataSync writes to S3 Standard
# Lifecycle moves old data to cheaper tiers

aws s3api put-bucket-lifecycle-configuration \
  --bucket datasync-destination \
  --lifecycle-configuration '{
    "Rules": [{
      "Id": "archive-old-files",
      "Status": "Enabled",
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"}
      ]
    }]
  }'

# Saves 84% on storage costs after 90 days
```

5. **Verify Mode Optimization**
```bash
# POINT_IN_TIME_CONSISTENT: Verifies during transfer (faster)
# ONLY_FILES_TRANSFERRED: Verifies after transfer (thorough)
# NONE: No verification (fastest, but risky)

aws datasync create-task \
  --options '{
    "VerifyMode": "POINT_IN_TIME_CONSISTENT"  # Balance speed & accuracy
  }'
```

**Cost Breakdown Example:**

```
Scenario: Sync 10 TB daily from on-prem NFS to S3

Initial transfer (10 TB):
├─ DataSync data copied: 10 TB * $0.0125/GB = $128
├─ Data transfer out (on-prem): 10 TB * $0.09/GB = $922
└─ S3 storage: 10 TB * $0.023/GB = $236/month
    Total initial: $1,286

Daily incremental (100 GB):
├─ DataSync: 100 GB * $0.0125/GB = $1.28/day = $38/month
├─ Data transfer: 100 GB * $0.09/GB = $9/day = $270/month
└─ S3 storage: +100 GB/day = +$70/month
    Total monthly: $308 + growing storage

With optimization (CHANGED mode, off-hours, lifecycle):
├─ DataSync: $38/month (same, only changed files)
├─ Data transfer: $270/month (same, but off-hours avoids congestion)
├─ S3 storage: 3 TB active ($69), 7 TB Glacier IA ($35) = $104/month
└─ Savings: $302/month → $207/month (32% reduction)
```

---

**Q9: How does AWS Transfer Family integrate with Lambda for file processing?**

**Answer:**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│     Transfer Family + Lambda Integration                   │
└─────────────────────────────────────────────────────────────┘

1. SFTP Upload
   Partner uploads file.csv
        ↓
   AWS Transfer for SFTP
        ↓
   S3 Bucket: /incoming/file.csv

2. S3 Event Notification
   S3 → EventBridge
        ↓
   EventBridge Rule (filter: *.csv)
        ↓
   Lambda: validate-and-process

3. Lambda Processing
   ├─ Validate CSV schema
   ├─ Virus scan (ClamAV)
   ├─ Move to /processed/ or /quarantine/
   └─ Trigger Glue ETL

4. Downstream Processing
   Glue Crawler → Athena (queryable)
```

**Implementation:**

**Option 1: EventBridge (Recommended)**

```bash
# Enable EventBridge on S3 bucket
aws s3api put-bucket-notification-configuration \
  --bucket sftp-uploads \
  --notification-configuration '{
    "EventBridgeConfiguration": {}
  }'

# Create EventBridge rule
aws events put-rule \
  --name sftp-file-uploaded \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"],
    "detail": {
      "bucket": {"name": ["sftp-uploads"]},
      "object": {"key": [{"prefix": "incoming/"}]}
    }
  }'

# Target Lambda
aws events put-targets \
  --rule sftp-file-uploaded \
  --targets "Id=1,Arn=arn:aws:lambda:...:function:process-sftp-file"
```

**Option 2: S3 Event Notification (Direct)**

```bash
# Add S3 event notification (alternative to EventBridge)
aws s3api put-bucket-notification-configuration \
  --bucket sftp-uploads \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [{
      "Id": "sftp-upload-notification",
      "LambdaFunctionArn": "arn:aws:lambda:...:function:process-sftp-file",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {"Name": "prefix", "Value": "incoming/"},
            {"Name": "suffix", "Value": ".csv"}
          ]
        }
      }
    }]
  }'
```

**Lambda Function:**

```python
# lambda_function.py
import boto3
import csv
import io
import hashlib

s3 = boto3.client('s3')
sns = boto3.client('sns')

def lambda_handler(event, context):
    """
    Process file uploaded via SFTP
    """
    
    # EventBridge event structure
    bucket = event['detail']['bucket']['name']
    key = event['detail']['object']['key']
    size = event['detail']['object']['size']
    
    print(f"Processing: s3://{bucket}/{key} ({size} bytes)")
    
    try:
        # 1. Download file
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read()
        
        # 2. Virus scan (integrate with ClamAV)
        if not virus_scan(content):
            quarantine_file(bucket, key, "Virus detected")
            return {'statusCode': 400, 'body': 'File quarantined: virus'}
        
        # 3. Validate CSV format
        csv_reader = csv.DictReader(io.StringIO(content.decode('utf-8')))
        rows = list(csv_reader)
        
        required_columns = ['transaction_id', 'amount', 'date', 'customer_id']
        if not all(col in csv_reader.fieldnames for col in required_columns):
            quarantine_file(bucket, key, f"Missing columns: {required_columns}")
            return {'statusCode': 400, 'body': 'Invalid schema'}
        
        # 4. Data quality checks
        for row in rows:
            if not row['transaction_id'] or not row['amount']:
                quarantine_file(bucket, key, "Missing required fields")
                return {'statusCode': 400, 'body': 'Data quality failed'}
        
        # 5. Calculate checksum
        checksum = hashlib.md5(content).hexdigest()
        
        # 6. Move to processed folder
        new_key = key.replace('/incoming/', '/processed/')
        s3.copy_object(
            Bucket=bucket,
            CopySource={'Bucket': bucket, 'Key': key},
            Key=new_key,
            Metadata={'checksum': checksum, 'row_count': str(len(rows))},
            MetadataDirective='REPLACE'
        )
        
        # Delete from incoming
        s3.delete_object(Bucket=bucket, Key=key)
        
        # 7. Notify success
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:file-processing-alerts',
            Subject='✅ File Processed Successfully',
            Message=f'File: {key}\nRows: {len(rows)}\nChecksum: {checksum}'
        )
        
        # 8. Trigger downstream processing (Glue)
        glue = boto3.client('glue')
        glue.start_crawler(Name='sftp-files-crawler')
        
        return {
            'statusCode': 200,
            'body': f'Successfully processed {len(rows)} rows'
        }
        
    except Exception as e:
        quarantine_file(bucket, key, str(e))
        raise

def virus_scan(content):
    """
    Scan file for viruses using ClamAV
    (Simplified - real implementation uses pyclamd or Lambda Layer)
    """
    # Integrate with ClamAV or S3 Virus Scanning solution
    return True  # Simplified for demo

def quarantine_file(bucket, key, reason):
    """
    Move file to quarantine folder
    """
    quarantine_key = key.replace('/incoming/', '/quarantine/')
    
    s3.copy_object(
        Bucket=bucket,
        CopySource={'Bucket': bucket, 'Key': key},
        Key=quarantine_key,
        Metadata={'quarantine_reason': reason},
        MetadataDirective='REPLACE'
    )
    
    s3.delete_object(Bucket=bucket, Key=key)
    
    # Alert ops team
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:file-processing-alerts',
        Subject='⚠️ File Quarantined',
        Message=f'File: {key}\nReason: {reason}'
    )
```

**Benefits:**
- ✅ **Automated**: No manual intervention needed
- ✅ **Validated**: Catches invalid files before processing
- ✅ **Secure**: Virus scanning, quarantine folder
- ✅ **Auditable**: CloudWatch Logs, SNS notifications
- ✅ **Scalable**: Lambda handles 1000s of files/hour

---

**Q10: Compare Snowball, DataSync, and Direct Connect for large data transfers.**

**Answer:**

| Factor | Snowball Edge | DataSync | Direct Connect |
|--------|---------------|----------|----------------|
| **Capacity** | 80 TB per device | Unlimited (network-limited) | Unlimited (network-limited) |
| **Speed** | ~1 week (ship + load) | 100-1000 Mbps (network) | 1-100 Gbps (dedicated) |
| **Cost** | $300 + shipping | $0.0125/GB + data transfer | $0.30/hr/port + data transfer |
| **Setup Time** | 2-3 days (shipping) | 1 hour (agent install) | 2-4 weeks (circuit install) |
| **Ongoing** | No (one-time) | Yes (scheduled tasks) | Yes (always-on connection) |
| **Security** | Encrypted device | Encrypted transfer (TLS) | Private connection (not internet) |
| **Best For** | 10-80 TB one-time migration | Ongoing sync < 10 TB/day | Hybrid cloud, > 1 Gbps ongoing |

**Decision Tree:**

```
How much data to transfer?

< 10 TB
├─ One-time migration? → Internet + aws s3 sync
└─ Ongoing sync? → DataSync

10-80 TB
├─ One-time migration? → Snowball Edge
└─ Ongoing sync? → Direct Connect + DataSync

> 80 TB
├─ One-time migration? → Multiple Snowballs or Snowmobile
└─ Ongoing sync? → Direct Connect (10 Gbps+)

Need < 1 ms latency? → Direct Connect
Need offline transfer? → Snowball
Budget-conscious? → DataSync over internet
```

**Cost Comparison (100 TB migration):**

**Option 1: Internet + DataSync**
```
Data transfer: 100 TB * $0.09/GB = $9,000 (egress from on-prem)
DataSync: 100 TB * $0.0125/GB = $1,280
Time: 100 TB / 100 Mbps = 92 days
Total: $10,280, 92 days
```

**Option 2: Snowball Edge (2 devices)**
```
Device rental: 2 * $300 = $600
Shipping: 2 * $50 = $100
Data transfer to AWS: $0 (included)
Time: 7 days (load) + 3 days (ship) + 2 days (import) = 12 days
Total: $700, 12 days ✅ 93% cheaper, 87% faster
```

**Option 3: Direct Connect (10 Gbps)**
```
Setup: $0.30/hour * 24 * 30 = $216/month (port fee)
Data transfer: 100 TB * $0.02/GB = $2,048 (reduced rate via DX)
Time: 100 TB / 10 Gbps = 22 hours
Total: $2,264, 1 day ✅ Fastest
```

**Recommendation:**
- **< 10 TB**: DataSync over internet
- **10-80 TB one-time**: Snowball Edge
- **> 80 TB one-time**: Multiple Snowballs or Snowmobile
- **Ongoing, > 1 Gbps**: Direct Connect + DataSync
- **Latency-sensitive**: Direct Connect only

---

### Scenario-Based Questions (10)

**Q11: Migrate 500 TB Oracle Data Warehouse to Redshift with Zero Downtime**

**Scenario**: Your company runs a 500 TB Oracle Exadata data warehouse with 24/7 reporting requirements. You need to migrate to Amazon Redshift to reduce costs by 70% while maintaining zero downtime for analysts querying the data. The warehouse has:
- **500 TB data** (10 billion rows across 200 tables)
- **Daily ETL**: 2 TB new data loaded every night
- **Query workload**: 500 concurrent analysts running reports
- **Downtime tolerance**: 0 (must maintain read access)
- **Timeline**: 60 days to complete migration

How do you design and execute this migration?

**Solution:**

**Migration Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│      Zero-Downtime Data Warehouse Migration               │
└─────────────────────────────────────────────────────────────┘

Phase 1: Parallel Operation (Weeks 1-8)
┌────────────────────────────────────────────────────────────┐
│ Oracle Exadata (Source)          Redshift (Target)         │
│ ├─ Analysts query here           ├─ Loading data           │
│ ├─ Daily ETL continues           ├─ Testing queries        │
│ └─ Production workload           └─ Shadow environment     │
└────────────────────────────────────────────────────────────┘
         │                                  ↑
         │     AWS DMS (Full Load + CDC)    │
         └──────────────────────────────────┘

Phase 2: Dual Operation (Week 9)
┌────────────────────────────────────────────────────────────┐
│ Oracle (Primary)                 Redshift (Shadow)         │
│ ├─ 80% analysts                  ├─ 20% analysts (testing)│
│ ├─ Critical reports              ├─ Non-critical reports  │
│ └─ DMS replicating → → → → → → │ └─ In sync (< 5 min lag)│
└────────────────────────────────────────────────────────────┘

Phase 3: Cutover (Week 10)
┌────────────────────────────────────────────────────────────┐
│ Oracle (Standby)                 Redshift (Primary)        │
│ ├─ Read-only                     ├─ 100% analysts          │
│ ├─ Keep for 30 days              ├─ All reports migrated  │
│ └─ DMS stopped                   └─ Daily ETL on Redshift │
└────────────────────────────────────────────────────────────┘
```

**Step-by-Step Implementation:**

**Week 1-2: Assessment and Planning**

```bash
# 1. Run AWS Schema Conversion Tool (SCT)
# Download: https://aws.amazon.com/dms/schema-conversion-tool/

# SCT Analysis:
# - Converts Oracle SQL to Redshift SQL
# - Identifies incompatibilities
# - Recommends distribution and sort keys

# Example SCT output:
Total objects: 1,200
├─ Tables: 200
├─ Views: 500
├─ Stored Procedures: 300
├─ Functions: 200
└─ Auto-conversion rate: 75%

Incompatibilities:
├─ Oracle-specific functions: 50 (need manual rewrite)
├─ Materialized views: 100 (convert to Redshift MVs)
└─ Partitioning: Use Redshift sort keys instead

# 2. Right-size Redshift cluster
# Based on 500 TB data + query workload

# Oracle performance:
# - 500 concurrent queries
# - Avg query time: 30 seconds
# - Peak CPU: 70%

# Redshift sizing:
aws redshift create-cluster \
  --cluster-identifier prod-dw-cluster \
  --node-type ra3.4xlarge \
  --number-of-nodes 16 \  # 16 nodes * 32 TB = 512 TB capacity
  --master-username admin \
  --master-user-password 'SecurePassword123!' \
  --cluster-subnet-group-name private-subnet-group \
  --vpc-security-group-ids sg-redshift \
  --encrypted \
  --kms-key-id alias/redshift-prod

# Cost comparison:
# Oracle Exadata: $1.2M/year
# Redshift RA3.4xlarge (16 nodes): $350K/year (70% savings) ✅
```

**Week 3-4: Schema Migration**

```sql
-- Create Redshift schema with optimized distribution/sort keys

-- Oracle table (source):
CREATE TABLE fact_sales (
    sale_id NUMBER PRIMARY KEY,
    customer_id NUMBER,
    product_id NUMBER,
    sale_date DATE,
    amount NUMBER(10,2),
    quantity NUMBER
) PARTITION BY RANGE (sale_date);

-- Redshift table (target) - optimized:
CREATE TABLE fact_sales (
    sale_id BIGINT,
    customer_id BIGINT DISTKEY,  -- Distribute by customer (frequent JOIN key)
    product_id BIGINT,
    sale_date DATE SORTKEY,       -- Sort by date (WHERE clause filter)
    amount DECIMAL(10,2),
    quantity INT
)
DISTSTYLE KEY;

-- Small dimension tables: use DISTSTYLE ALL
CREATE TABLE dim_customers (
    customer_id BIGINT PRIMARY KEY,
    customer_name VARCHAR(255),
    region VARCHAR(50)
)
DISTSTYLE ALL;  -- Replicate to all nodes (< 1 GB table)
```

**Week 5-7: Data Migration (DMS Full Load + CDC)**

```bash
# 1. Create DMS replication instance (large for 500 TB)
aws dms create-replication-instance \
  --replication-instance-identifier oracle-redshift-migration \
  --replication-instance-class dms.c5.24xlarge \  # 96 vCPU, 192 GB RAM
  --allocated-storage 10000 \  # 10 TB for staging
  --multi-az

# 2. Create endpoints
# Source: Oracle Exadata
aws dms create-endpoint \
  --endpoint-identifier oracle-exadata-source \
  --endpoint-type source \
  --engine-name oracle \
  --server-name exadata.company.local \
  --port 1521 \
  --database-name PRODDB \
  --username dms_user \
  --password 'OraclePassword123!'

# Target: Redshift
aws dms create-endpoint \
  --endpoint-identifier redshift-target \
  --endpoint-type target \
  --engine-name redshift \
  --server-name prod-dw-cluster.xyz.us-east-1.redshift.amazonaws.com \
  --port 5439 \
  --database-name analytics \
  --username admin \
  --password 'SecurePassword123!'

# 3. Create migration task (Full Load + CDC)
aws dms create-replication-task \
  --replication-task-identifier oracle-redshift-full-cdc \
  --source-endpoint-arn arn:aws:dms:...:endpoint:oracle-exadata-source \
  --target-endpoint-arn arn:aws:dms:...:endpoint:redshift-target \
  --replication-instance-arn arn:aws:dms:...:rep:oracle-redshift-migration \
  --migration-type full-load-and-cdc \
  --table-mappings file://table-mappings.json \
  --replication-task-settings '{
    "TargetMetadata": {
      "SupportLobs": true,
      "LobMaxSize": 32
    },
    "FullLoadSettings": {
      "TargetTablePrepMode": "DROP_AND_CREATE",
      "MaxFullLoadSubTasks": 49,  # Parallel loading
      "TransactionConsistencyTimeout": 600
    }
  }'

# 4. Monitor migration progress
# Full load: 500 TB / 1 GB/s = 138 hours = 6 days
# Actual: 21 days (slower due to network, source load)

# Timeline:
# Day 1-21: Full load (500 TB copied)
# Day 22+: CDC (replication lag < 5 minutes)
```

**Week 8: Testing and Validation**

```sql
-- 1. Compare row counts
-- Oracle:
SELECT table_name, num_rows 
FROM user_tables 
WHERE table_name LIKE 'FACT_%' OR table_name LIKE 'DIM_%'
ORDER BY num_rows DESC;

-- Redshift:
SELECT "table", tbl_rows 
FROM svv_table_info 
WHERE "table" LIKE 'fact_%' OR "table" LIKE 'dim_%'
ORDER BY tbl_rows DESC;

-- 2. Validate data integrity (checksums)
-- Oracle:
SELECT COUNT(*), SUM(amount), MAX(sale_date), MIN(sale_date)
FROM fact_sales;

-- Redshift:
SELECT COUNT(*), SUM(amount), MAX(sale_date), MIN(sale_date)
FROM fact_sales;

-- 3. Performance testing (run same queries on both)
-- Oracle query:
SELECT 
    customer_region,
    product_category,
    SUM(sale_amount) AS total_revenue
FROM fact_sales s
JOIN dim_customers c ON s.customer_id = c.customer_id
JOIN dim_products p ON s.product_id = p.product_id
WHERE sale_date BETWEEN '2024-01-01' AND '2024-06-30'
GROUP BY customer_region, product_category
ORDER BY total_revenue DESC
LIMIT 100;

-- Oracle: 45 seconds
-- Redshift: 8 seconds (5.6x faster) ✅

-- 4. Migrate 20% of users to Redshift (shadow testing)
-- Create separate Redshift endpoint for test users
-- Monitor for errors, performance issues
```

**Week 9: Gradual Cutover**

```
User Migration Plan:
├─ Week 9.1: 20% users → Redshift (non-critical reports)
├─ Week 9.2: 40% users → Redshift (more complex queries)
├─ Week 9.3: 60% users → Redshift (most users)
├─ Week 9.4: 80% users → Redshift (leave critical reports on Oracle)
└─ Week 10: 100% users → Redshift (final cutover)
```

```python
# Automated cutover script
import boto3

route53 = boto3.client('route53')

def gradual_cutover(percent):
    """
    Gradually shift users from Oracle to Redshift
    Using Route 53 weighted routing
    """
    
    # Update DNS weighted routing
    route53.change_resource_record_sets(
        HostedZoneId='Z1234567890',
        ChangeBatch={
            'Changes': [
                {
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': 'datawarehouse.company.com',
                        'Type': 'CNAME',
                        'SetIdentifier': 'Oracle',
                        'Weight': 100 - percent,  # Decreasing weight
                        'TTL': 60,
                        'ResourceRecords': [{'Value': 'oracle-exadata.company.local'}]
                    }
                },
                {
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': 'datawarehouse.company.com',
                        'Type': 'CNAME',
                        'SetIdentifier': 'Redshift',
                        'Weight': percent,  # Increasing weight
                        'TTL': 60,
                        'ResourceRecords': [{'Value': 'prod-dw-cluster.xyz.us-east-1.redshift.amazonaws.com'}]
                    }
                }
            ]
        }
    )
    
    print(f"Cutover: {percent}% traffic → Redshift, {100-percent}% → Oracle")

# Execute gradual cutover
gradual_cutover(20)  # Week 9.1
# Monitor for 2 days...
gradual_cutover(40)  # Week 9.2
# Monitor for 2 days...
gradual_cutover(100)  # Week 10 (final)
```

**Week 10: Final Cutover and Decommission**

```bash
# 1. Stop DMS replication
aws dms stop-replication-task \
  --replication-task-arn arn:aws:dms:...:task:oracle-redshift-full-cdc

# 2. Set Oracle to read-only (keep for 30 days backup)
# Oracle:
ALTER SYSTEM SET READ_ONLY = TRUE;

# 3. Migrate daily ETL to Redshift
# Old ETL (Oracle):
# - TRUNCATE fact_sales_daily;
# - INSERT INTO fact_sales_daily SELECT...;

# New ETL (Redshift):
# - Use COPY from S3 (10x faster than INSERT)
COPY fact_sales_daily
FROM 's3://etl-staging/sales/2024-06-22/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
FORMAT AS PARQUET;

# 4. Decommission Oracle (after 30 days)
# Final backup, then shutdown
```

**Results:**

| Metric | Oracle Exadata | Amazon Redshift | Improvement |
|--------|----------------|-----------------|-------------|
| **Capacity** | 500 TB | 512 TB (16 nodes) | +2% |
| **Query Performance** | 45 sec (avg) | 8 sec (avg) | **5.6x faster** |
| **Concurrent Users** | 500 | 500 | Same |
| **Cost** | $1.2M/year | $350K/year | **70% reduction** |
| **Migration Downtime** | N/A | 0 seconds | **Zero downtime** |
| **Migration Duration** | N/A | 60 days | On schedule |

**Cost Savings:**
- Annual savings: $1.2M - $350K = **$850K/year**
- 3-year TCO savings: **$2.55M**
- ROI: 24x (savings / migration cost)

**Lessons Learned:**
1. **DMS Full Load + CDC is critical** for zero-downtime migration
2. **Gradual cutover** reduces risk (20% → 100% over 2 weeks)
3. **DISTKEY/SORTKEY optimization** provides 5x query speedup
4. **RA3 instances** provide flexible storage scaling (up to 8 PB)

---

(Continue to next question...)
**Q12: Migrate 2 PB of Historical Data from On-Premises to S3 Glacier Using AWS Snowball Edge**

**Scenario:**

You're a data engineer at a biotech company with 2 PB (petabytes) of genomics research data stored on-premises in a tape library. The data includes:
- 1.5 PB: Sequencing data (FASTQ files, 100 GB - 500 GB each)
- 300 TB: Analysis results (BAM/VCF files)
- 200 TB: Metadata and reports

**Current State:**
- On-premises storage: Dell EMC tape library with LTO-8 tapes
- Network: 1 Gbps internet connection
- Compliance: HIPAA, data must be encrypted in transit and at rest
- Access pattern: Cold storage, accessed 2-3 times per year

**Requirements:**
1. Migrate all 2 PB to AWS S3 Glacier Deep Archive
2. Maintain compliance with HIPAA encryption requirements
3. Complete migration within 3 months
4. Minimize network bandwidth usage (cannot saturate 1 Gbps link)
5. Preserve file metadata and directory structure

**Challenge:** Should you use network transfer (DataSync) or physical transfer (Snowball Edge)?

---

### Answer:

**Decision: Use AWS Snowball Edge for Physical Transfer**

**Why Snowball Edge vs DataSync:**

```
Network Transfer Analysis (DataSync):
- 2 PB = 2,000,000 GB
- 1 Gbps = 125 MB/s theoretical max
- Assume 50% utilization = 62.5 MB/s actual
- Time = 2,000,000 GB / (62.5 MB/s × 3600 s/hr × 24 hr/day)
-      = 2,000,000 / 5,400 GB/day
-      = 370 days ❌ (exceeds 3-month requirement)

Physical Transfer Analysis (Snowball Edge):
- 10× Snowball Edge Storage Optimized devices
- Each device: 210 TB usable capacity
- Total capacity: 2,100 TB (covers 2 PB requirement) ✅
- Timeline:
  - Week 1-2: Device delivery (10 devices)
  - Week 3-8: Data copy (6 weeks for 2 PB @ 350 TB/week)
  - Week 9-10: Ship devices back to AWS
  - Week 11-12: AWS ingests data into S3
- Total: 12 weeks = 3 months ✅
```

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│           Snowball Edge Migration Architecture              │
└─────────────────────────────────────────────────────────────┘

On-Premises Tape Library
│
├─ Restore tapes to staging disk array (Dell EMC Unity)
│  └─ 300 TB staging capacity (reuse for batches)
│
├─ 10× Snowball Edge Devices (parallel copy)
│  ├─ Device 1: Sequencing data batch 1 (210 TB)
│  ├─ Device 2: Sequencing data batch 2 (210 TB)
│  ├─ Device 3: Sequencing data batch 3 (210 TB)
│  ├─ Device 4: Sequencing data batch 4 (210 TB)
│  ├─ Device 5: Sequencing data batch 5 (210 TB)
│  ├─ Device 6: Sequencing data batch 6 (210 TB)
│  ├─ Device 7: Sequencing data batch 7 (210 TB)
│  ├─ Device 8: Analysis results (300 TB, 2 devices)
│  └─ Device 10: Metadata (200 TB)
│
└─ AWS receives and ingests into S3 Glacier Deep Archive
   └─ s3://biotech-genomics-archive/

Security:
├─ 256-bit encryption (Snowball Edge built-in)
├─ Trusted Platform Module (TPM) chip
├─ AWS KMS key management
└─ Tamper-resistant shipping container
```

**Implementation:**

**Step 1: Order Snowball Edge Devices**

```bash
# Create Snowball job using AWS CLI
aws snowball create-job \
  --job-type IMPORT \
  --resources '{
    "S3Resources": [
      {
        "BucketArn": "arn:aws:s3:::biotech-genomics-archive",
        "KeyRange": {
          "BeginMarker": "",
          "EndMarker": ""
        }
      }
    ]
  }' \
  --description "Genomics data migration - 2 PB" \
  --address-id "ADID123456789" \
  --kms-key-arn "arn:aws:kms:us-east-1:123456789012:key/abc123" \
  --shipping-option SECOND_DAY \
  --snowball-capacity-preference T100 \  # 100 TB devices (10 devices)
  --snowball-type EDGE_S

# Output:
{
  "JobId": "JID123e4567-e89b-12d3-a456-426655440000"
}

# Track shipment
aws snowball describe-job --job-id JID123e4567-e89b-12d3-a456-426655440000
```

**Step 2: Set Up On-Premises Staging Environment**

```bash
# Staging disk array configuration
# Dell EMC Unity 300 TB capacity

# Create staging directories (batch processing)
mkdir -p /staging/batch1_sequencing
mkdir -p /staging/batch2_sequencing
mkdir -p /staging/batch3_sequencing
mkdir -p /staging/analysis_results
mkdir -p /staging/metadata

# Restore data from tape library to staging
# Using Dell EMC Avamar for tape restore
# Batch 1: Restore 200 TB of sequencing data
avamar-restore --tape-library LTO8 \
  --source "/tape/sequencing/batch1" \
  --destination "/staging/batch1_sequencing" \
  --threads 8

# Monitoring: Tape restore speed ~5 TB/hour
# 200 TB batch = 40 hours restore time
```

**Step 3: Copy Data to Snowball Edge Devices**

```bash
# When Snowball Edge device arrives:
# 1. Power on and connect to local network
# 2. Unlock device using manifest and unlock code

# Get device credentials
aws snowball get-snowball-unlock-code \
  --job-id JID123e4567-e89b-12d3-a456-426655440000

# Configure Snowball Edge client
snowballEdge configure \
  --manifest-file /path/to/job-manifest.bin \
  --unlock-code 01234-abcde-01234-ABCDE-01234

# Get device IP
snowballEdge describe-device

# Copy data using S3 adapter (built-in on Snowball Edge)
aws s3 cp /staging/batch1_sequencing/ s3://biotech-genomics-archive/sequencing/batch1/ \
  --recursive \
  --endpoint http://192.168.1.100:8080 \  # Snowball Edge local endpoint
  --profile snowball \
  --metadata "project=genomics,data-type=fastq,batch=1"

# Performance tuning for large files (100-500 GB FASTQ files)
aws configure set default.s3.max_concurrent_requests 20
aws configure set default.s3.multipart_threshold 512MB
aws configure set default.s3.multipart_chunksize 128MB

# Typical performance: 250 MB/s (210 TB in ~10 days per device)
```

**Parallel Copy Script (10 Devices Simultaneously):**

```python
#!/usr/bin/env python3
"""
Parallel Snowball Edge data copy orchestration
Manages 10 devices copying data simultaneously
"""

import subprocess
import concurrent.futures
import logging
from datetime import datetime

# Device configuration
DEVICES = [
    {'id': 'device-01', 'ip': '192.168.1.101', 'source': '/staging/batch1_sequencing', 'dest': 's3://biotech-genomics-archive/sequencing/batch1/'},
    {'id': 'device-02', 'ip': '192.168.1.102', 'source': '/staging/batch2_sequencing', 'dest': 's3://biotech-genomics-archive/sequencing/batch2/'},
    {'id': 'device-03', 'ip': '192.168.1.103', 'source': '/staging/batch3_sequencing', 'dest': 's3://biotech-genomics-archive/sequencing/batch3/'},
    {'id': 'device-04', 'ip': '192.168.1.104', 'source': '/staging/batch4_sequencing', 'dest': 's3://biotech-genomics-archive/sequencing/batch4/'},
    {'id': 'device-05', 'ip': '192.168.1.105', 'source': '/staging/batch5_sequencing', 'dest': 's3://biotech-genomics-archive/sequencing/batch5/'},
    {'id': 'device-06', 'ip': '192.168.1.106', 'source': '/staging/batch6_sequencing', 'dest': 's3://biotech-genomics-archive/sequencing/batch6/'},
    {'id': 'device-07', 'ip': '192.168.1.107', 'source': '/staging/batch7_sequencing', 'dest': 's3://biotech-genomics-archive/sequencing/batch7/'},
    {'id': 'device-08', 'ip': '192.168.1.108', 'source': '/staging/analysis_bam', 'dest': 's3://biotech-genomics-archive/analysis/bam/'},
    {'id': 'device-09', 'ip': '192.168.1.109', 'source': '/staging/analysis_vcf', 'dest': 's3://biotech-genomics-archive/analysis/vcf/'},
    {'id': 'device-10', 'ip': '192.168.1.110', 'source': '/staging/metadata', 'dest': 's3://biotech-genomics-archive/metadata/'},
]

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')

def copy_to_device(device):
    """Copy data to a single Snowball Edge device"""
    device_id = device['id']
    endpoint = f"http://{device['ip']}:8080"
    source = device['source']
    dest = device['dest']
    
    logging.info(f"[{device_id}] Starting copy from {source} to {dest}")
    start_time = datetime.now()
    
    try:
        # AWS S3 copy command to Snowball Edge endpoint
        cmd = [
            'aws', 's3', 'cp', source, dest,
            '--recursive',
            '--endpoint', endpoint,
            '--profile', 'snowball',
            '--metadata', f'device-id={device_id},migration-date={datetime.now().isoformat()}'
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True, check=True)
        
        end_time = datetime.now()
        duration = (end_time - start_time).total_seconds() / 3600  # hours
        
        logging.info(f"[{device_id}] ✅ Copy completed in {duration:.2f} hours")
        logging.info(f"[{device_id}] Output: {result.stdout}")
        
        return {'device_id': device_id, 'status': 'success', 'duration_hours': duration}
        
    except subprocess.CalledProcessError as e:
        logging.error(f"[{device_id}] ❌ Copy failed: {e.stderr}")
        return {'device_id': device_id, 'status': 'failed', 'error': str(e)}

def main():
    """Run parallel copy jobs for all 10 devices"""
    
    logging.info("Starting parallel Snowball Edge copy for 10 devices")
    
    # Use ThreadPoolExecutor for parallel execution
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        # Submit all copy jobs
        futures = [executor.submit(copy_to_device, device) for device in DEVICES]
        
        # Wait for all to complete
        results = []
        for future in concurrent.futures.as_completed(futures):
            result = future.result()
            results.append(result)
            
            if result['status'] == 'success':
                logging.info(f"✅ {result['device_id']}: {result['duration_hours']:.2f} hours")
            else:
                logging.error(f"❌ {result['device_id']}: {result['error']}")
    
    # Summary
    successful = sum(1 for r in results if r['status'] == 'success')
    failed = sum(1 for r in results if r['status'] == 'failed')
    total_hours = sum(r.get('duration_hours', 0) for r in results)
    
    logging.info(f"\n{'='*60}")
    logging.info(f"Migration Summary:")
    logging.info(f"  Successful devices: {successful}/10")
    logging.info(f"  Failed devices: {failed}/10")
    logging.info(f"  Total copy time: {total_hours:.2f} hours (wall-clock)")
    logging.info(f"  Average time per device: {total_hours/10:.2f} hours")
    logging.info(f"{'='*60}\n")

if __name__ == '__main__':
    main()
```

**Step 4: Ship Devices Back to AWS**

```bash
# When copy is complete, verify data integrity
snowballEdge describe-device

# Request device pickup
# AWS automatically schedules UPS pickup within 1 business day

# Track shipment status
aws snowball describe-job --job-id JID123e4567-e89b-12d3-a456-426655440000 | jq '.JobMetadata.JobState'

# Possible states:
# - "New" → job created
# - "PreparingAppliance" → device being prepared
# - "PreparingShipment" → device ready to ship to you
# - "InTransitToCustomer" → device shipped to you
# - "WithCustomer" → device at your facility
# - "InTransitToAWS" → device shipped back to AWS ✅
# - "WithAWS" → AWS received device
# - "InProgress" → AWS copying data to S3
# - "Complete" → data in S3 ✅
```

**Step 5: AWS Ingests Data to S3 Glacier Deep Archive**

```bash
# After AWS receives devices, data is copied to S3
# Timeline: 7-10 days for 2 PB ingestion

# Verify data in S3 (while in Standard storage class initially)
aws s3 ls s3://biotech-genomics-archive/sequencing/ --recursive --human-readable --summarize

# Output:
# Total Objects: 15,234
# Total Size: 1.5 PB

# Apply lifecycle policy to transition to Glacier Deep Archive
cat > lifecycle-policy.json <<EOF
{
  "Rules": [
    {
      "Id": "TransitionToGlacierDeepArchive",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 0,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ]
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket biotech-genomics-archive \
  --lifecycle-configuration file://lifecycle-policy.json

# Data transitions to Glacier Deep Archive immediately
# Transition time: 12 hours for metadata update
```

**Step 6: Validation and Compliance**

```python
#!/usr/bin/env python3
"""
Validate Snowball migration completion and compliance
"""

import boto3
import hashlib
import json

s3 = boto3.client('s3')
cloudtrail = boto3.client('cloudtrail')

def validate_migration():
    """Validate all 2 PB migrated successfully"""
    
    bucket = 'biotech-genomics-archive'
    
    # Count objects and total size
    paginator = s3.get_paginator('list_objects_v2')
    total_objects = 0
    total_size = 0
    
    for page in paginator.paginate(Bucket=bucket):
        if 'Contents' in page:
            total_objects += len(page['Contents'])
            total_size += sum(obj['Size'] for obj in page['Contents'])
    
    total_size_tb = total_size / (1024**4)  # Convert to TB
    
    print(f"Migration Validation:")
    print(f"  Total objects: {total_objects:,}")
    print(f"  Total size: {total_size_tb:.2f} TB")
    print(f"  Expected: 2,000 TB")
    
    if total_size_tb >= 1950:  # Allow 2.5% margin
        print("  ✅ Migration complete")
    else:
        print(f"  ❌ Migration incomplete ({total_size_tb/2000*100:.1f}%)")
    
    return total_size_tb >= 1950

def check_encryption_compliance():
    """Verify HIPAA encryption compliance"""
    
    bucket = 'biotech-genomics-archive'
    
    # Check bucket encryption
    try:
        encryption = s3.get_bucket_encryption(Bucket=bucket)
        kms_key = encryption['ServerSideEncryptionConfiguration']['Rules'][0]['ApplyServerSideEncryptionByDefault']['KMSMasterKeyID']
        
        print(f"\nEncryption Compliance:")
        print(f"  Bucket encryption: Enabled (KMS)")
        print(f"  KMS Key: {kms_key}")
        print(f"  ✅ HIPAA compliant")
        
        return True
        
    except:
        print(f"  ❌ Bucket encryption not enabled!")
        return False

def audit_trail():
    """Generate CloudTrail audit for compliance"""
    
    # Query CloudTrail for Snowball job events
    response = cloudtrail.lookup_events(
        LookupAttributes=[
            {'AttributeKey': 'EventName', 'AttributeValue': 'CreateJob'},
        ],
        MaxResults=50
    )
    
    print(f"\nAudit Trail:")
    for event in response['Events']:
        event_time = event['EventTime']
        username = event.get('Username', 'Unknown')
        event_name = event['EventName']
        
        print(f"  {event_time}: {username} performed {event_name}")
    
    print(f"  ✅ Audit trail available for compliance review")

if __name__ == '__main__':
    migration_valid = validate_migration()
    encryption_compliant = check_encryption_compliance()
    audit_trail()
    
    if migration_valid and encryption_compliant:
        print(f"\n{'='*60}")
        print(f"✅ Migration completed successfully and is HIPAA compliant")
        print(f"{'='*60}")
    else:
        print(f"\n❌ Migration validation failed - review errors above")
```

**Results:**

| Metric | On-Premises Tape | AWS Snowball Edge | Improvement |
|--------|------------------|-------------------|-------------|
| **Data Volume** | 2 PB | 2 PB | Same |
| **Migration Duration** | N/A | 12 weeks | Within 3-month SLA ✅ |
| **Network Bandwidth Used** | 0 | ~10 GB (metadata only) | **99.9% bandwidth saved** |
| **Storage Cost** | $250K/year (tape + maintenance) | $18K/year (Glacier Deep Archive) | **93% reduction** |
| **Retrieval Time** | 24-48 hours (tape restore) | 12 hours (Glacier restore) | 50% faster |
| **Encryption** | Manual (tape encryption) | Automatic (256-bit AES) | HIPAA compliant ✅ |

**Cost Analysis:**

```
Snowball Edge Migration Cost:
├─ 10× Snowball Edge devices: $3,000 ($300/device)
├─ Shipping (round-trip): $1,500
├─ On-premises staging storage (3 months): $5,000
├─ Labor (150 hours @ $150/hr): $22,500
└─ TOTAL MIGRATION COST: $32,000

Annual Storage Cost (Glacier Deep Archive):
├─ 2 PB = 2,000 TB
├─ $0.00099/GB/month = $0.99/TB/month
├─ 2,000 TB × $0.99/TB × 12 months = $23,760/year
└─ vs Tape storage: $250K/year
    SAVINGS: $226,240/year (90% reduction) ✅

3-Year TCO:
├─ Migration cost: $32,000 (one-time)
├─ 3-year storage: $71,280 ($23,760 × 3)
└─ TOTAL: $103,280
    vs On-premises tape: $750,000 (3 years)
    SAVINGS: $646,720 (86% reduction) ✅
```

**Key Lessons:**

1. **Snowball Edge is cost-effective for petabyte-scale migrations** when network transfer would take >6 months
2. **Parallel device usage** (10 devices simultaneously) completes migration in 12 weeks vs 60 weeks (single device)
3. **Glacier Deep Archive** provides 90% cost savings vs tape for cold storage
4. **HIPAA compliance** achieved with KMS encryption and CloudTrail auditing
5. **Staging disk array** (300 TB reusable capacity) enables batch processing from tape library

---

**Q13: Optimize AWS DataSync Performance for Multi-Region Disaster Recovery**

**Scenario:**

You're a data engineer at a financial services company managing a data lake with the following requirements:

**Current State:**
- Primary data lake: S3 bucket in us-east-1 (500 TB, 50 million objects)
- File types: Parquet files (10-500 MB), JSON logs (1-10 MB), CSV reports (100 KB - 1 MB)
- Daily change rate: 5 TB new data, 2 TB modified data, 500 GB deletions
- Compliance: SEC requires disaster recovery with RPO < 1 hour

**Requirements:**
1. Replicate data to secondary region (us-west-2) for disaster recovery
2. Initial replication: 500 TB within 7 days
3. Daily sync: Complete within 8-hour batch window (12 AM - 8 AM EST)
4. Minimize cost (avoid expensive network transfer)
5. Maintain file metadata, permissions, and directory structure

**Challenge:** Should you use S3 Replication or AWS DataSync? How do you optimize for performance and cost?

---

### Answer:

**Decision: Use AWS DataSync for Initial Sync, then S3 Replication for Ongoing Changes**

**Why DataSync + S3 Replication Hybrid:**

```
Option 1: S3 Replication Only
├─ Pros: Simple, automatic, no setup
├─ Cons: 
│  ├─ Initial 500 TB replication takes 10-14 days (slower than DataSync)
│  ├─ No control over replication schedule (runs immediately)
│  └─ Cannot filter objects by prefix during initial sync
└─ Cost: $0.02/GB × 500,000 GB = $10,000 initial + $140/day ongoing

Option 2: DataSync Only
├─ Pros: Fast initial sync (500 TB in 5-7 days), scheduled tasks
├─ Cons:
│  ├─ Ongoing sync requires manual task scheduling
│  └─ Less efficient for small, frequent changes
└─ Cost: $0.0125/GB × 500,000 GB = $6,250 initial + $87.50/day ongoing

Option 3: DataSync (Initial) + S3 Replication (Ongoing) ✅
├─ Pros:
│  ├─ Fast initial sync (DataSync: 500 TB in 5 days)
│  ├─ Efficient ongoing replication (S3 Replication: automatic, real-time)
│  └─ Best of both worlds
└─ Cost: $6,250 initial + $140/day ongoing = OPTIMAL ✅
```

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│      Multi-Region DR with DataSync + S3 Replication         │
└─────────────────────────────────────────────────────────────┘

Phase 1: Initial 500 TB Sync (DataSync)
─────────────────────────────────────────
Primary Region (us-east-1)
├─ S3: s3://finserv-datalake-primary/
│  └─ 500 TB, 50M objects
│
├─ VPC Endpoint (S3)
│  └─ DataSync Agent (in VPC, m5.4xlarge)
│      └─ 16 vCPU, 64 GB RAM, 10 Gbps network
│
└─ DataSync Task ──────→ us-west-2
                         │
                         └─ S3: s3://finserv-datalake-dr/
                            └─ 500 TB replicated

Phase 2: Ongoing Replication (S3 Replication)
──────────────────────────────────────────────
s3://finserv-datalake-primary/ (us-east-1)
│
├─ S3 Replication Rule: Replicate all new/modified objects
│  ├─ Filter: Entire bucket
│  ├─ Priority: High
│  └─ Replication Time Control (RTC): Enabled (15-min SLA)
│
└─────→ s3://finserv-datalake-dr/ (us-west-2)
        └─ RPO: 15 minutes (RTC) ✅
        └─ Compliance: SEC requirement met

Monitoring:
├─ CloudWatch Alarms: DataSync task failures, S3 Replication lag
├─ S3 Storage Lens: Replication metrics
└─ EventBridge: Trigger Lambda on replication failures
```

**Implementation:**

**Phase 1: Initial 500 TB Sync with DataSync**

```bash
# Step 1: Create S3 locations for DataSync
# Source location (us-east-1)
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::finserv-datalake-primary \
  --s3-config '{
    "BucketAccessRoleArn": "arn:aws:iam::123456789012:role/DataSyncS3Role"
  }' \
  --region us-east-1

# Output:
{
  "LocationArn": "arn:aws:datasync:us-east-1:123456789012:location/loc-0a1b2c3d4e5f67890"
}

# Destination location (us-west-2)
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::finserv-datalake-dr \
  --s3-config '{
    "BucketAccessRoleArn": "arn:aws:iam::123456789012:role/DataSyncS3Role"
  }' \
  --region us-east-1  # Create in same region as task

# Output:
{
  "LocationArn": "arn:aws:datasync:us-east-1:123456789012:location/loc-1a2b3c4d5e6f78901"
}

# Step 2: Create DataSync task (optimized for performance)
aws datasync create-task \
  --source-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-0a1b2c3d4e5f67890 \
  --destination-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-1a2b3c4d5e6f78901 \
  --name "finserv-datalake-initial-sync" \
  --options '{
    "VerifyMode": "POINT_IN_TIME_CONSISTENT",
    "OverwriteMode": "ALWAYS",
    "Atime": "BEST_EFFORT",
    "Mtime": "PRESERVE",
    "PreserveDeletedFiles": "REMOVE",
    "PreserveDevices": "NONE",
    "PosixPermissions": "PRESERVE",
    "BytesPerSecond": -1,
    "TaskQueueing": "ENABLED",
    "TransferMode": "CHANGED"
  }' \
  --region us-east-1

# Output:
{
  "TaskArn": "arn:aws:datasync:us-east-1:123456789012:task/task-0a1b2c3d4e5f67890"
}

# Step 3: Start DataSync task
aws datasync start-task-execution \
  --task-arn arn:aws:datasync:us-east-1:123456789012:task/task-0a1b2c3d4e5f67890 \
  --region us-east-1

# Output:
{
  "TaskExecutionArn": "arn:aws:datasync:us-east-1:123456789012:task/task-0a1b2c3d4e5f67890/execution/exec-0a1b2c3d4e5f67890"
}

# Step 4: Monitor progress (500 TB = 5-7 days)
aws datasync describe-task-execution \
  --task-execution-arn arn:aws:datasync:us-east-1:123456789012:task/task-0a1b2c3d4e5f67890/execution/exec-0a1b2c3d4e5f67890 \
  --region us-east-1

# Output (after 3 days):
{
  "TaskExecutionArn": "...",
  "Status": "EXECUTING",
  "BytesTransferred": 214748364800000,  # 214 TB transferred
  "BytesWritten": 214748364800000,
  "FilesTransferred": 21474836,
  "Result": {
    "PrepareDuration": 120000,  # 2 minutes
    "PrepareStatus": "SUCCESS",
    "TransferDuration": 259200000,  # 72 hours (3 days)
    "TransferStatus": "IN_PROGRESS"
  }
}

# Performance: ~100 GB/hour (2.4 TB/day per task)
# For 500 TB: Create 5 parallel tasks (each handles 100 TB)
# Total time: 500 TB / (5 tasks × 2.4 TB/day) = 41.7 days / 5 = 8.3 days
```

**Performance Optimization: Parallel DataSync Tasks**

```python
#!/usr/bin/env python3
"""
Create parallel DataSync tasks for faster 500 TB migration
Split by S3 prefix to avoid conflicts
"""

import boto3
import time

datasync = boto3.client('datasync', region_name='us-east-1')

# Define prefixes to split data (each ~100 TB)
PREFIXES = [
    'raw/year=2020/',      # 100 TB
    'raw/year=2021/',      # 100 TB
    'raw/year=2022/',      # 100 TB
    'processed/year=2021/', # 100 TB
    'processed/year=2022/', # 100 TB
]

SOURCE_LOCATION = 'arn:aws:datasync:us-east-1:123456789012:location/loc-0a1b2c3d4e5f67890'
DEST_LOCATION = 'arn:aws:datasync:us-east-1:123456789012:location/loc-1a2b3c4d5e6f78901'

def create_parallel_tasks():
    """Create 5 parallel DataSync tasks (one per prefix)"""
    
    task_arns = []
    
    for prefix in PREFIXES:
        # Create task with prefix filter
        response = datasync.create_task(
            SourceLocationArn=SOURCE_LOCATION,
            DestinationLocationArn=DEST_LOCATION,
            Name=f'finserv-sync-{prefix.replace("/", "-")}',
            Options={
                'VerifyMode': 'POINT_IN_TIME_CONSISTENT',
                'OverwriteMode': 'ALWAYS',
                'TransferMode': 'CHANGED',
                'BytesPerSecond': -1,  # No throttling
            },
            Includes=[
                {'FilterType': 'SIMPLE_PATTERN', 'Value': f'/{prefix}*'}
            ]
        )
        
        task_arn = response['TaskArn']
        task_arns.append({'prefix': prefix, 'task_arn': task_arn})
        print(f"✅ Created task for prefix: {prefix}")
        print(f"   Task ARN: {task_arn}")
    
    return task_arns

def start_all_tasks(task_arns):
    """Start all 5 tasks in parallel"""
    
    execution_arns = []
    
    for task in task_arns:
        response = datasync.start_task_execution(
            TaskArn=task['task_arn']
        )
        
        exec_arn = response['TaskExecutionArn']
        execution_arns.append({
            'prefix': task['prefix'],
            'exec_arn': exec_arn
        })
        print(f"✅ Started task for {task['prefix']}")
    
    return execution_arns

def monitor_progress(execution_arns):
    """Monitor all 5 parallel tasks"""
    
    while True:
        all_complete = True
        total_transferred = 0
        
        print(f"\n{'='*80}")
        print(f"DataSync Progress Report - {time.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"{'='*80}")
        
        for exec in execution_arns:
            response = datasync.describe_task_execution(
                TaskExecutionArn=exec['exec_arn']
            )
            
            status = response.get('Status')
            bytes_transferred = response.get('BytesTransferred', 0)
            bytes_written = response.get('BytesWritten', 0)
            files_transferred = response.get('FilesTransferred', 0)
            
            transferred_gb = bytes_transferred / (1024**3)
            total_transferred += bytes_transferred
            
            print(f"Prefix: {exec['prefix']}")
            print(f"  Status: {status}")
            print(f"  Data transferred: {transferred_gb:,.2f} GB")
            print(f"  Files transferred: {files_transferred:,}")
            
            if status != 'SUCCESS':
                all_complete = False
        
        total_tb = total_transferred / (1024**4)
        print(f"\nTotal transferred: {total_tb:.2f} TB / 500 TB ({total_tb/500*100:.1f}%)")
        print(f"{'='*80}\n")
        
        if all_complete:
            print("✅ All DataSync tasks completed successfully!")
            break
        
        # Check every 30 minutes
        time.sleep(1800)

if __name__ == '__main__':
    print("Creating 5 parallel DataSync tasks...")
    task_arns = create_parallel_tasks()
    
    print("\nStarting all tasks in parallel...")
    execution_arns = start_all_tasks(task_arns)
    
    print("\nMonitoring progress (updates every 30 minutes)...")
    monitor_progress(execution_arns)
```

**Phase 2: Enable S3 Replication for Ongoing Changes**

```bash
# After DataSync completes initial 500 TB sync:

# Step 1: Enable versioning (required for replication)
aws s3api put-bucket-versioning \
  --bucket finserv-datalake-primary \
  --versioning-configuration Status=Enabled \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket finserv-datalake-dr \
  --versioning-configuration Status=Enabled \
  --region us-west-2

# Step 2: Create replication configuration
cat > replication-config.json <<EOF
{
  "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
  "Rules": [
    {
      "ID": "ReplicateAllObjects",
      "Priority": 1,
      "Filter": {},
      "Status": "Enabled",
      "Destination": {
        "Bucket": "arn:aws:s3:::finserv-datalake-dr",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {
            "Minutes": 15
          }
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": {
            "Minutes": 15
          }
        },
        "StorageClass": "STANDARD"
      },
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      }
    }
  ]
}
EOF

aws s3api put-bucket-replication \
  --bucket finserv-datalake-primary \
  --replication-configuration file://replication-config.json \
  --region us-east-1

# Step 3: Monitor replication metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name ReplicationLatency \
  --dimensions Name=SourceBucket,Value=finserv-datalake-primary Name=DestinationBucket,Value=finserv-datalake-dr \
  --start-time 2024-06-22T00:00:00Z \
  --end-time 2024-06-22T23:59:59Z \
  --period 3600 \
  --statistics Average,Maximum \
  --region us-east-1

# Typical replication latency: 5-15 minutes (with RTC)
```

**Cost Optimization:**

```bash
# Use S3 Intelligent-Tiering for DR bucket to reduce storage costs
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket finserv-datalake-dr \
  --id "AutoArchive" \
  --intelligent-tiering-configuration '{
    "Id": "AutoArchive",
    "Status": "Enabled",
    "Tierings": [
      {
        "Days": 90,
        "AccessTier": "ARCHIVE_ACCESS"
      },
      {
        "Days": 180,
        "AccessTier": "DEEP_ARCHIVE_ACCESS"
      }
    ]
  }' \
  --region us-west-2

# Cost savings:
# - Files not accessed for 90 days → Glacier (50% savings)
# - Files not accessed for 180 days → Glacier Deep Archive (75% savings)
```

**Monitoring and Alerting:**

```python
#!/usr/bin/env python3
"""
CloudWatch alarms for S3 replication monitoring
"""

import boto3

cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')

# Alarm 1: Replication lag > 30 minutes
cloudwatch.put_metric_alarm(
    AlarmName='S3-Replication-Lag-High',
    ComparisonOperator='GreaterThanThreshold',
    EvaluationPeriods=2,
    MetricName='ReplicationLatency',
    Namespace='AWS/S3',
    Period=300,  # 5 minutes
    Statistic='Maximum',
    Threshold=1800000,  # 30 minutes in milliseconds
    ActionsEnabled=True,
    AlarmActions=[
        'arn:aws:sns:us-east-1:123456789012:DataEngineering-Alerts'
    ],
    AlarmDescription='Alert when S3 replication lag exceeds 30 minutes (RPO breach)',
    Dimensions=[
        {'Name': 'SourceBucket', 'Value': 'finserv-datalake-primary'},
        {'Name': 'DestinationBucket', 'Value': 'finserv-datalake-dr'}
    ]
)

# Alarm 2: Replication failure rate > 1%
cloudwatch.put_metric_alarm(
    AlarmName='S3-Replication-FailureRate-High',
    ComparisonOperator='GreaterThanThreshold',
    EvaluationPeriods=1,
    MetricName='OperationsFailedReplication',
    Namespace='AWS/S3',
    Period=3600,  # 1 hour
    Statistic='Sum',
    Threshold=100,  # More than 100 failures per hour
    ActionsEnabled=True,
    AlarmActions=[
        'arn:aws:sns:us-east-1:123456789012:DataEngineering-Alerts'
    ],
    AlarmDescription='Alert when S3 replication failures exceed threshold',
    Dimensions=[
        {'Name': 'SourceBucket', 'Value': 'finserv-datalake-primary'}
    ]
)

print("✅ CloudWatch alarms created for S3 replication monitoring")
```

**Results:**

| Metric | Before (No DR) | After (DataSync + S3 Replication) | Improvement |
|--------|----------------|-----------------------------------|-------------|
| **Initial Sync Time** | N/A | 5 days (500 TB) | Within 7-day SLA ✅ |
| **RPO (Recovery Point Objective)** | N/A | 15 minutes (RTC) | Meets SEC < 1 hour ✅ |
| **Daily Replication** | N/A | 7 TB replicated in 2-4 hours | Within 8-hour window ✅ |
| **Cost (Initial Sync)** | N/A | $6,250 (DataSync) | Cheaper than S3 Replication |
| **Cost (Daily Ongoing)** | N/A | $140/day (S3 Replication) | Automatic, no manual intervention |
| **Storage Cost (DR)** | N/A | $11,500/month (Intelligent-Tiering) | 30% savings vs Standard |

**Cost Analysis:**

```
Initial 500 TB Sync (One-Time):
├─ DataSync transfer: 500 TB × $0.0125/GB = $6,400
├─ DataSync data scanned: 500 TB × $0.0025/GB (first scan) = $1,280
└─ TOTAL: $7,680

Daily Ongoing Replication (S3 Replication):
├─ New data: 5 TB × $0.02/GB = $102.40/day
├─ Modified data: 2 TB × $0.02/GB = $40.96/day
├─ Replication Time Control (RTC): $0.01/GB × 7 TB = $71.68/day
└─ TOTAL: $215.04/day = $6,451/month

Monthly Total:
├─ DataSync (initial, one-time in Month 1): $7,680
├─ S3 Replication (ongoing): $6,451/month
├─ DR storage (500 TB with Intelligent-Tiering): $11,500/month
└─ Month 1: $25,631 | Ongoing: $17,951/month

Annual Cost: $7,680 + ($17,951 × 12) = $223,092/year

vs On-Premises DR (tape-based):
├─ Tape library at secondary datacenter: $500,000/year
└─ SAVINGS: $276,908/year (55% reduction) ✅
```

**Key Lessons:**

1. **Hybrid approach** (DataSync for initial + S3 Replication for ongoing) provides best performance and cost
2. **Parallel DataSync tasks** (5 tasks × 100 TB each) reduce migration time from 40 days to 5 days
3. **S3 Replication Time Control (RTC)** guarantees 15-minute RPO, meeting SEC compliance
4. **S3 Intelligent-Tiering** on DR bucket provides 30% storage cost savings
5. **CloudWatch alarms** ensure RPO breaches are detected within 5 minutes

---

**Q14: Hybrid Cloud Data Synchronization with AWS Direct Connect and DataSync**

**Scenario:**

You're a data engineer at a media company with hybrid cloud architecture:

**Current State:**
- On-premises: 200 TB video rendering farm (produces 2 TB/day of rendered videos)
- Local NFS storage: Dell EMC Isilon (10 Gbps network)
- AWS: S3 bucket for video archive and CDN distribution
- Network: 10 Gbps AWS Direct Connect connection (us-east-1)
- Workflow: Render videos on-premises → sync to S3 → CloudFront distribution

**Requirements:**
1. Sync 2 TB of new rendered videos daily to S3 (complete within 4-hour window)
2. Preserve file metadata (creation time, permissions, custom attributes)
3. Only sync completed videos (ignore temp files, in-progress renders)
4. Minimize internet bandwidth costs (use Direct Connect, not public internet)
5. Automated scheduling (sync at 2 AM daily when rendering is complete)

**Challenge:** How do you optimize DataSync to use Direct Connect and meet the 4-hour SLA?

---

### Answer:

**Solution: DataSync with VPC Endpoint over AWS Direct Connect**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│    Hybrid Cloud DataSync with Direct Connect Architecture   │
└─────────────────────────────────────────────────────────────┘

On-Premises Data Center
├─ Video Rendering Farm (100 nodes)
│  └─ Render output: /mnt/nfs/renders/completed/
│
├─ Dell EMC Isilon NFS (200 TB)
│  ├─ /mnt/nfs/renders/completed/     ← Final videos (sync)
│  ├─ /mnt/nfs/renders/in-progress/   ← Ignore
│  └─ /mnt/nfs/renders/temp/          ← Ignore
│
└─ DataSync Agent (EC2-like appliance)
   ├─ Deployed in on-prem VMware cluster
   ├─ 4 vCPU, 32 GB RAM, 80 GB disk
   └─ Connected to 10 Gbps Direct Connect

      │
      │ 10 Gbps AWS Direct Connect
      │ (private VIF to VPC)
      ▼

AWS Cloud (us-east-1)
├─ VPC (10.0.0.0/16)
│  ├─ Private Subnet (10.0.1.0/24)
│  │  └─ VPC Endpoint for S3 (gateway endpoint)
│  │
│  └─ VPC Endpoint for DataSync (interface endpoint)
│     └─ PrivateLink to DataSync service
│
└─ S3: s3://media-company-video-archive/
   ├─ renders/2024/06/22/video001.mp4
   ├─ renders/2024/06/22/video002.mp4
   └─ ... (2 TB/day)
       │
       └─→ CloudFront CDN (global distribution)

Data Flow:
1. On-prem agent scans /mnt/nfs/renders/completed/
2. Agent connects to DataSync via VPC Endpoint (PrivateLink)
3. Data flows over Direct Connect (10 Gbps, private network)
4. DataSync writes to S3 via VPC Gateway Endpoint (no internet)
5. S3 triggers Lambda to update CDN cache
```

**Implementation:**

**Step 1: Deploy DataSync Agent On-Premises**

```bash
# Download DataSync agent OVA for VMware
# From AWS Console: DataSync → Agents → Create agent → Download OVA

# Deploy in VMware vSphere:
# 1. Deploy OVF template
# 2. Configure network (10 Gbps uplink to Direct Connect)
# 3. Power on agent VM
# 4. Access agent web interface: http://<agent-ip>/

# Activate agent (connects to DataSync service via VPC endpoint)
aws datasync create-agent \
  --agent-name "media-onprem-agent" \
  --vpc-endpoint-id vpce-0a1b2c3d4e5f67890 \  # DataSync VPC endpoint
  --subnet-arns arn:aws:ec2:us-east-1:123456789012:subnet/subnet-0a1b2c3d \
  --security-group-arns arn:aws:ec2:us-east-1:123456789012:security-group/sg-0a1b2c3d \
  --activation-key "ABCDE-12345-FGHIJ-67890-KLMNO" \  # From agent web UI
  --region us-east-1

# Output:
{
  "AgentArn": "arn:aws:datasync:us-east-1:123456789012:agent/agent-0a1b2c3d4e5f67890"
}
```

**Step 2: Create VPC Endpoints (S3 and DataSync)**

```bash
# Create VPC Gateway Endpoint for S3 (no charge)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0a1b2c3d \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids rtb-0a1b2c3d \
  --region us-east-1

# Output:
{
  "VpcEndpoint": {
    "VpcEndpointId": "vpce-s3-0a1b2c3d4e5f67890",
    "ServiceName": "com.amazonaws.us-east-1.s3",
    "State": "available"
  }
}

# Create VPC Interface Endpoint for DataSync ($0.01/hour)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0a1b2c3d \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.datasync \
  --subnet-ids subnet-0a1b2c3d subnet-1a2b3c4d \
  --security-group-ids sg-0a1b2c3d \
  --region us-east-1

# Output:
{
  "VpcEndpoint": {
    "VpcEndpointId": "vpce-0a1b2c3d4e5f67890",
    "ServiceName": "com.amazonaws.us-east-1.datasync",
    "State": "available",
    "DnsEntries": [
      {
        "DnsName": "vpce-0a1b2c3d-abcd1234.datasync.us-east-1.vpce.amazonaws.com"
      }
    ]
  }
}
```

**Step 3: Create DataSync Locations and Task**

```bash
# Create source location (on-prem NFS)
aws datasync create-location-nfs \
  --server-hostname isilon.media.local \  # On-prem NFS server
  --subdirectory /ifs/renders/completed \  # Only sync completed videos
  --on-prem-config '{
    "AgentArns": ["arn:aws:datasync:us-east-1:123456789012:agent/agent-0a1b2c3d4e5f67890"]
  }' \
  --region us-east-1

# Output:
{
  "LocationArn": "arn:aws:datasync:us-east-1:123456789012:location/loc-nfs-0a1b2c3d"
}

# Create destination location (S3)
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::media-company-video-archive \
  --s3-config '{
    "BucketAccessRoleArn": "arn:aws:iam::123456789012:role/DataSyncS3Role"
  }' \
  --region us-east-1

# Output:
{
  "LocationArn": "arn:aws:datasync:us-east-1:123456789012:location/loc-s3-1a2b3c4d"
}

# Create DataSync task with filtering and scheduling
aws datasync create-task \
  --source-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-nfs-0a1b2c3d \
  --destination-location-arn arn:aws:datasync:us-east-1:123456789012:location/loc-s3-1a2b3c4d \
  --name "media-daily-video-sync" \
  --options '{
    "VerifyMode": "ONLY_FILES_TRANSFERRED",
    "OverwriteMode": "NEVER",
    "Atime": "BEST_EFFORT",
    "Mtime": "PRESERVE",
    "Uid": "NONE",
    "Gid": "NONE",
    "PreserveDeletedFiles": "REMOVE",
    "PreserveDevices": "NONE",
    "PosixPermissions": "NONE",
    "BytesPerSecond": -1,
    "TaskQueueing": "ENABLED",
    "TransferMode": "CHANGED"
  }' \
  --excludes '[
    {"FilterType": "SIMPLE_PATTERN", "Value": "*.tmp"},
    {"FilterType": "SIMPLE_PATTERN", "Value": "*.part"},
    {"FilterType": "SIMPLE_PATTERN", "Value": "*_incomplete*"}
  ]' \
  --schedule '{
    "ScheduleExpression": "cron(0 2 * * ? *)"
  }' \
  --region us-east-1

# Output:
{
  "TaskArn": "arn:aws:datasync:us-east-1:123456789012:task/task-0a1b2c3d4e5f67890"
}

# Schedule explanation:
# cron(0 2 * * ? *) = Every day at 2:00 AM UTC
# Runs automatically after rendering completes (1 AM cutoff)
```

**Step 4: Performance Optimization**

```bash
# Calculate bandwidth requirements:
# - Daily data: 2 TB = 2,048 GB
# - Time window: 4 hours = 14,400 seconds
# - Required throughput: 2,048 GB / 14,400 s = 0.142 GB/s = 142 MB/s
# - Direct Connect: 10 Gbps = 1,250 MB/s
# - Utilization: 142 MB/s / 1,250 MB/s = 11.4% ✅

# DataSync performance tuning
aws datasync update-task \
  --task-arn arn:aws:datasync:us-east-1:123456789012:task/task-0a1b2c3d4e5f67890 \
  --options '{
    "BytesPerSecond": 200000000,
    "TaskQueueing": "ENABLED"
  }' \
  --region us-east-1

# BytesPerSecond = 200 MB/s (headroom for network variability)
# Actual: 2 TB in ~2.8 hours (well within 4-hour SLA) ✅
```

**Step 5: Monitoring and Alerting**

```python
#!/usr/bin/env python3
"""
Monitor DataSync task execution and alert on SLA breach
"""

import boto3
import time
from datetime import datetime, timedelta

datasync = boto3.client('datasync', region_name='us-east-1')
cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')
sns = boto3.client('sns', region_name='us-east-1')

TASK_ARN = 'arn:aws:datasync:us-east-1:123456789012:task/task-0a1b2c3d4e5f67890'
SNS_TOPIC = 'arn:aws:sns:us-east-1:123456789012:DataSync-Alerts'
SLA_HOURS = 4

def get_latest_execution():
    """Get the most recent task execution"""
    
    response = datasync.list_task_executions(
        TaskArn=TASK_ARN,
        MaxResults=1
    )
    
    if not response.get('TaskExecutions'):
        return None
    
    exec_arn = response['TaskExecutions'][0]['TaskExecutionArn']
    
    exec_details = datasync.describe_task_execution(
        TaskExecutionArn=exec_arn
    )
    
    return exec_details

def check_sla_breach():
    """Check if task execution exceeds 4-hour SLA"""
    
    execution = get_latest_execution()
    
    if not execution:
        print("No recent task executions found")
        return
    
    status = execution.get('Status')
    start_time = execution.get('StartTime')
    
    if status == 'SUCCESS':
        transfer_duration_ms = execution['Result'].get('TransferDuration', 0)
        transfer_duration_hours = transfer_duration_ms / (1000 * 3600)
        
        bytes_transferred = execution.get('BytesTransferred', 0)
        bytes_transferred_gb = bytes_transferred / (1024**3)
        
        print(f"✅ Task completed successfully")
        print(f"   Data transferred: {bytes_transferred_gb:.2f} GB")
        print(f"   Duration: {transfer_duration_hours:.2f} hours")
        
        if transfer_duration_hours > SLA_HOURS:
            # SLA breach!
            message = f"""
            ⚠️ DataSync SLA Breach Alert
            
            Task: media-daily-video-sync
            Duration: {transfer_duration_hours:.2f} hours
            SLA: {SLA_HOURS} hours
            Data: {bytes_transferred_gb:.2f} GB
            
            Action required: Investigate performance degradation
            """
            
            sns.publish(
                TopicArn=SNS_TOPIC,
                Subject='DataSync SLA Breach',
                Message=message
            )
            
            print(f"❌ SLA BREACH! Duration {transfer_duration_hours:.2f}h exceeds {SLA_HOURS}h")
        
    elif status == 'EXECUTING':
        # Task is running, check if it's been running too long
        elapsed = datetime.now(start_time.tzinfo) - start_time
        elapsed_hours = elapsed.total_seconds() / 3600
        
        print(f"⏳ Task is running ({elapsed_hours:.2f} hours elapsed)")
        
        if elapsed_hours > SLA_HOURS:
            message = f"""
            ⚠️ DataSync Long-Running Task Alert
            
            Task: media-daily-video-sync
            Elapsed: {elapsed_hours:.2f} hours (still running)
            SLA: {SLA_HOURS} hours
            
            Task may miss SLA deadline
            """
            
            sns.publish(
                TopicArn=SNS_TOPIC,
                Subject='DataSync Task Running Long',
                Message=message
            )
            
            print(f"⚠️ WARNING: Task running for {elapsed_hours:.2f}h (SLA: {SLA_HOURS}h)")
    
    elif status == 'ERROR':
        print(f"❌ Task failed with error")
        # Send alert (implementation omitted)

def publish_metrics():
    """Publish custom CloudWatch metrics"""
    
    execution = get_latest_execution()
    
    if not execution or execution.get('Status') != 'SUCCESS':
        return
    
    transfer_duration_ms = execution['Result'].get('TransferDuration', 0)
    transfer_duration_hours = transfer_duration_ms / (1000 * 3600)
    bytes_transferred = execution.get('BytesTransferred', 0)
    bytes_transferred_gb = bytes_transferred / (1024**3)
    
    # Metric 1: Transfer duration
    cloudwatch.put_metric_data(
        Namespace='MediaCompany/DataSync',
        MetricData=[
            {
                'MetricName': 'TransferDuration',
                'Value': transfer_duration_hours,
                'Unit': 'None',
                'Timestamp': datetime.now()
            }
        ]
    )
    
    # Metric 2: Data volume
    cloudwatch.put_metric_data(
        Namespace='MediaCompany/DataSync',
        MetricData=[
            {
                'MetricName': 'DataTransferred',
                'Value': bytes_transferred_gb,
                'Unit': 'Gigabytes',
                'Timestamp': datetime.now()
            }
        ]
    )
    
    # Metric 3: Throughput
    throughput_mbps = (bytes_transferred_gb * 1024) / (transfer_duration_ms / 1000)
    cloudwatch.put_metric_data(
        Namespace='MediaCompany/DataSync',
        MetricData=[
            {
                'MetricName': 'Throughput',
                'Value': throughput_mbps,
                'Unit': 'Megabytes/Second',
                'Timestamp': datetime.now()
            }
        ]
    )
    
    print(f"📊 Published metrics to CloudWatch")

if __name__ == '__main__':
    print("Checking DataSync SLA compliance...")
    check_sla_breach()
    publish_metrics()
```

**Step 6: Cost Optimization**

```bash
# Direct Connect pricing vs Internet transfer

# Option 1: Internet Transfer (baseline)
# - Data transfer OUT: 2 TB/day × 30 days = 60 TB/month
# - Cost: 60,000 GB × $0.09/GB = $5,400/month
# - DataSync: 60,000 GB × $0.0125/GB = $750/month
# - TOTAL: $6,150/month

# Option 2: Direct Connect (10 Gbps)
# - Port fee: $2,250/month (dedicated 10 Gbps)
# - Data transfer OUT: 60,000 GB × $0.02/GB = $1,200/month (80% cheaper)
# - DataSync: 60,000 GB × $0.0125/GB = $750/month
# - VPC Endpoint (interface): $0.01/hour × 730 hours = $7.30/month
# - TOTAL: $4,207.30/month
# - SAVINGS: $1,942.70/month (32% cheaper) ✅

# Direct Connect also provides:
# - Lower latency (10-15ms vs 40-60ms public internet)
# - Consistent performance (dedicated bandwidth)
# - Security (private connection, no public internet)
```

**Results:**

| Metric | Before (Manual Sync) | After (DataSync + Direct Connect) | Improvement |
|--------|----------------------|-----------------------------------|-------------|
| **Daily Sync Time** | 8-10 hours (rsync) | 2.8 hours (DataSync) | **65% faster** |
| **SLA Compliance** | 60% (manual delays) | 100% (automated) | **40% improvement** |
| **Bandwidth Cost** | $5,400/month (internet) | $1,200/month (Direct Connect) | **78% reduction** |
| **Operational Overhead** | 2 hours/day (manual) | 0 hours (automated) | **100% reduction** |
| **Error Rate** | 5% (manual errors) | 0.1% (automated validation) | **98% reduction** |

**Cost Breakdown:**

```
Monthly Cost (DataSync + Direct Connect):
├─ AWS Direct Connect (10 Gbps port): $2,250/month
├─ Direct Connect data transfer OUT: $1,200/month (60 TB × $0.02/GB)
├─ DataSync data transfer: $750/month (60 TB × $0.0125/GB)
├─ DataSync VPC endpoint: $7.30/month
├─ S3 storage (60 TB new/month): $1,380/month (Standard class)
└─ TOTAL: $5,587.30/month

Annual Cost: $67,048/year

vs On-Premises NAS Expansion:
├─ Additional Dell EMC Isilon (for AWS mirror): $150,000 CapEx
├─ Annual maintenance: $30,000/year
└─ TOTAL (3-year TCO): $240,000

AWS Solution (3-year TCO): $201,144
SAVINGS: $38,856 (16% reduction) + operational efficiency ✅
```

**Key Lessons:**

1. **Direct Connect + VPC Endpoints** avoid internet data transfer costs (78% savings)
2. **DataSync scheduling** eliminates manual sync operations (2 hours/day saved)
3. **File filtering** (exclude .tmp, .part files) reduces unnecessary transfer by 15%
4. **10 Gbps Direct Connect** provides 11% utilization for 2 TB/day, allowing growth to 18 TB/day
5. **CloudWatch custom metrics** enable proactive SLA monitoring and alerting

---
**Q15: Secure SFTP File Transfer with AWS Transfer Family for Financial Compliance**

**Scenario:**

You're a data engineer at a hedge fund managing third-party data feeds:

**Current State:**
- 50 data vendors send daily files via SFTP (financial market data, SEC filings, news feeds)
- File sizes: 100 MB - 5 GB per vendor
- Total daily volume: 150 GB
- Current setup: On-premises SFTP server (aging hardware, compliance concerns)
- Compliance requirements: SOC 2, PCI DSS, FINRA regulations

**Requirements:**
1. Migrate SFTP server to AWS (eliminate on-premises hardware)
2. Maintain vendor SFTP endpoints (vendors cannot change their workflows)
3. Automatically process files upon arrival (validate, parse, load into Redshift)
4. Audit trail for all file transfers (who, when, what, IP address)
5. Encrypt all data at rest and in transit (compliance mandate)
6. High availability (99.9% uptime SLA for market hours)

**Challenge:** Design a secure, compliant SFTP solution using AWS Transfer Family

---

### Answer:

**Solution: AWS Transfer Family (SFTP) + S3 + Lambda + Redshift**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│       Secure SFTP Pipeline with Transfer Family             │
└─────────────────────────────────────────────────────────────┘

Data Vendors (50 vendors)
├─ Vendor 1: reuters.sftp.hedgefund.com (market data)
├─ Vendor 2: bloomberg.sftp.hedgefund.com (prices)
├─ Vendor 3: sec.sftp.hedgefund.com (SEC filings)
└─ ... (50 vendors total)
    │
    │ SFTP over TLS (port 22)
    │ Public internet + IP whitelisting
    ▼
AWS Transfer Family (SFTP Server)
├─ Endpoint: sftp.hedgefund.com (Elastic IP)
├─ Multi-AZ deployment (us-east-1a, us-east-1b)
├─ Authentication: AWS Secrets Manager (SSH keys per vendor)
├─ Authorization: IAM roles (scoped S3 access per vendor)
└─ Logging: CloudWatch Logs + S3 access logs
    │
    ▼
Amazon S3 (Landing Zone)
├─ s3://hedgefund-vendor-data/
│  ├─ reuters/2024/06/22/market_data_20240622.csv
│  ├─ bloomberg/2024/06/22/prices_20240622.csv
│  └─ sec/2024/06/22/filings_20240622.xml
│
└─ Encryption: SSE-KMS (customer-managed key)
    │
    │ S3 Event Notification
    ▼
AWS Lambda (File Processor)
├─ Triggered on S3 PutObject event
├─ Validate file format (schema, checksums)
├─ Parse and transform data
├─ Load into Redshift (COPY command)
└─ Send SNS notification on success/failure
    │
    ▼
Amazon Redshift (Analytics)
├─ Schema: vendor_data (market_data, prices, filings tables)
├─ Access: QuickSight dashboards for analysts
└─ Retention: 7 years (FINRA compliance)

Security & Compliance:
├─ CloudTrail: API audit trail
├─ CloudWatch Logs: SFTP session logs
├─ S3 Access Logs: File-level access audit
├─ GuardDuty: Threat detection
└─ Config: Compliance monitoring (SOC 2, PCI DSS)
```

**Implementation:**

**Step 1: Create AWS Transfer Family SFTP Server**

```bash
# Create S3 bucket for vendor data
aws s3api create-bucket \
  --bucket hedgefund-vendor-data \
  --region us-east-1

# Enable versioning (compliance requirement)
aws s3api put-bucket-versioning \
  --bucket hedgefund-vendor-data \
  --versioning-configuration Status=Enabled

# Enable encryption with KMS (customer-managed key)
aws s3api put-bucket-encryption \
  --bucket hedgefund-vendor-data \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abc-123-def-456"
      }
    }]
  }'

# Create Transfer Family SFTP server
aws transfer create-server \
  --endpoint-type PUBLIC \
  --protocols SFTP \
  --identity-provider-type SERVICE_MANAGED \
  --logging-role arn:aws:iam::123456789012:role/TransferLoggingRole \
  --tags Key=Environment,Value=Production Key=Compliance,Value=SOC2

# Output:
{
  "ServerId": "s-1234567890abcdef0"
}

# Get server endpoint
aws transfer describe-server --server-id s-1234567890abcdef0 | jq '.Server.EndpointDetails'

# Output:
{
  "AddressAllocationIds": [],
  "SubnetIds": null,
  "VpcEndpointId": null,
  "VpcId": null,
  "SecurityGroupIds": null
}

# Note: Server endpoint will be: s-1234567890abcdef0.server.transfer.us-east-1.amazonaws.com
# Create DNS CNAME: sftp.hedgefund.com → s-1234567890abcdef0.server.transfer.us-east-1.amazonaws.com
```

**Step 2: Configure Per-Vendor User Accounts with IAM**

```python
#!/usr/bin/env python3
"""
Create Transfer Family users for 50 vendors
Each vendor gets isolated S3 prefix access
"""

import boto3
import json

transfer = boto3.client('transfer', region_name='us-east-1')
iam = boto3.client('iam')
secretsmanager = boto3.client('secretsmanager')

SERVER_ID = 's-1234567890abcdef0'
BUCKET = 'hedgefund-vendor-data'

VENDORS = [
    {'name': 'reuters', 'ssh_public_key': 'ssh-rsa AAAAB3NzaC1yc2E...'},
    {'name': 'bloomberg', 'ssh_public_key': 'ssh-rsa AAAAB3NzaC1yc2E...'},
    {'name': 'sec', 'ssh_public_key': 'ssh-rsa AAAAB3NzaC1yc2E...'},
    # ... (50 vendors total)
]

def create_vendor_role(vendor_name):
    """Create IAM role with scoped S3 access for vendor"""
    
    # IAM policy: vendor can only write to their prefix
    policy_document = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowListBucket",
                "Effect": "Allow",
                "Action": "s3:ListBucket",
                "Resource": f"arn:aws:s3:::{BUCKET}",
                "Condition": {
                    "StringLike": {
                        "s3:prefix": [f"{vendor_name}/*", f"{vendor_name}"]
                    }
                }
            },
            {
                "Sid": "AllowWriteToVendorPrefix",
                "Effect": "Allow",
                "Action": [
                    "s3:PutObject",
                    "s3:PutObjectAcl",
                    "s3:GetObject",
                    "s3:GetObjectVersion",
                    "s3:DeleteObject",
                    "s3:DeleteObjectVersion"
                ],
                "Resource": f"arn:aws:s3:::{BUCKET}/{vendor_name}/*"
            }
        ]
    }
    
    # Create IAM role
    role_name = f'TransferFamily-{vendor_name}'
    
    assume_role_policy = {
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"Service": "transfer.amazonaws.com"},
            "Action": "sts:AssumeRole"
        }]
    }
    
    try:
        role = iam.create_role(
            RoleName=role_name,
            AssumeRolePolicyDocument=json.dumps(assume_role_policy),
            Description=f'Transfer Family role for vendor {vendor_name}'
        )
        
        # Attach inline policy
        iam.put_role_policy(
            RoleName=role_name,
            PolicyName=f'{vendor_name}-S3Access',
            PolicyDocument=json.dumps(policy_document)
        )
        
        print(f"✅ Created role: {role_name}")
        return role['Role']['Arn']
        
    except iam.exceptions.EntityAlreadyExistsException:
        print(f"⚠️  Role {role_name} already exists")
        role = iam.get_role(RoleName=role_name)
        return role['Role']['Arn']

def create_transfer_user(vendor_name, ssh_public_key, role_arn):
    """Create Transfer Family user for vendor"""
    
    # Home directory mapping (user sees / but it's actually /vendor-name/)
    home_directory_mappings = [
        {
            'Entry': '/',
            'Target': f'/{BUCKET}/{vendor_name}'
        }
    ]
    
    try:
        response = transfer.create_user(
            ServerId=SERVER_ID,
            UserName=vendor_name,
            HomeDirectoryType='LOGICAL',
            HomeDirectoryMappings=home_directory_mappings,
            Role=role_arn,
            SshPublicKeyBody=ssh_public_key,
            Tags=[
                {'Key': 'Vendor', 'Value': vendor_name},
                {'Key': 'Compliance', 'Value': 'SOC2'}
            ]
        )
        
        print(f"✅ Created Transfer user: {vendor_name}")
        print(f"   Endpoint: sftp.hedgefund.com (user: {vendor_name})")
        print(f"   Home: /{BUCKET}/{vendor_name}")
        
    except transfer.exceptions.ResourceExistsException:
        print(f"⚠️  User {vendor_name} already exists")

def main():
    """Set up all 50 vendor accounts"""
    
    for vendor in VENDORS:
        vendor_name = vendor['name']
        ssh_key = vendor['ssh_public_key']
        
        print(f"\nSetting up vendor: {vendor_name}")
        
        # Create IAM role
        role_arn = create_vendor_role(vendor_name)
        
        # Create Transfer Family user
        create_transfer_user(vendor_name, ssh_key, role_arn)
    
    print(f"\n{'='*60}")
    print(f"✅ All 50 vendor accounts created successfully")
    print(f"   SFTP Endpoint: sftp.hedgefund.com")
    print(f"   Authentication: SSH public key")
    print(f"   Logging: CloudWatch Logs")
    print(f"{'='*60}")

if __name__ == '__main__':
    main()
```

**Step 3: Automated File Processing with Lambda**

```python
#!/usr/bin/env python3
"""
Lambda function: Process vendor files on S3 upload
Triggered by S3 event notification
"""

import boto3
import json
import csv
import io
from datetime import datetime

s3 = boto3.client('s3')
redshift_data = boto3.client('redshift-data')
sns = boto3.client('sns')

REDSHIFT_CLUSTER = 'hedgefund-analytics'
REDSHIFT_DATABASE = 'vendor_data'
REDSHIFT_USER = 'admin'
SNS_TOPIC = 'arn:aws:sns:us-east-1:123456789012:VendorDataProcessing'

def lambda_handler(event, context):
    """
    Process vendor file uploaded via SFTP
    """
    
    # Get S3 object details from event
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        print(f"Processing file: s3://{bucket}/{key}")
        
        # Extract vendor name from S3 prefix
        # Example: reuters/2024/06/22/market_data_20240622.csv
        vendor_name = key.split('/')[0]
        
        try:
            # Step 1: Validate file
            validation_result = validate_file(bucket, key, vendor_name)
            
            if not validation_result['valid']:
                raise Exception(f"Validation failed: {validation_result['error']}")
            
            # Step 2: Load into Redshift
            load_to_redshift(bucket, key, vendor_name)
            
            # Step 3: Send success notification
            send_notification(
                subject=f"✅ {vendor_name} file processed successfully",
                message=f"File: {key}\nRows: {validation_result['row_count']}\nStatus: SUCCESS"
            )
            
            print(f"✅ Successfully processed {key}")
            
        except Exception as e:
            print(f"❌ Error processing {key}: {str(e)}")
            
            # Send failure notification
            send_notification(
                subject=f"❌ {vendor_name} file processing failed",
                message=f"File: {key}\nError: {str(e)}\nStatus: FAILED"
            )
            
            raise

def validate_file(bucket, key, vendor_name):
    """
    Validate vendor file format and content
    """
    
    # Download file from S3
    response = s3.get_object(Bucket=bucket, Key=key)
    file_content = response['Body'].read().decode('utf-8')
    
    # Parse CSV
    csv_reader = csv.DictReader(io.StringIO(file_content))
    rows = list(csv_reader)
    
    # Vendor-specific validation rules
    if vendor_name == 'reuters':
        # Reuters market data: required columns
        required_columns = ['symbol', 'price', 'volume', 'timestamp']
        
        if not all(col in csv_reader.fieldnames for col in required_columns):
            return {
                'valid': False,
                'error': f'Missing required columns: {required_columns}'
            }
        
        # Validate data types
        for row in rows:
            try:
                float(row['price'])
                int(row['volume'])
                datetime.fromisoformat(row['timestamp'])
            except ValueError as e:
                return {
                    'valid': False,
                    'error': f'Invalid data type: {str(e)}'
                }
    
    elif vendor_name == 'bloomberg':
        # Bloomberg prices: different schema
        required_columns = ['ticker', 'bid', 'ask', 'date']
        
        if not all(col in csv_reader.fieldnames for col in required_columns):
            return {
                'valid': False,
                'error': f'Missing required columns: {required_columns}'
            }
    
    # Validation passed
    return {
        'valid': True,
        'row_count': len(rows),
        'file_size': len(file_content)
    }

def load_to_redshift(bucket, key, vendor_name):
    """
    Load CSV file into Redshift using COPY command
    """
    
    # Determine target table based on vendor
    table_mapping = {
        'reuters': 'market_data',
        'bloomberg': 'prices',
        'sec': 'filings'
    }
    
    target_table = table_mapping.get(vendor_name, 'unknown')
    
    # Redshift COPY command (serverless)
    copy_sql = f"""
    COPY {target_table}
    FROM 's3://{bucket}/{key}'
    IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
    CSV
    IGNOREHEADER 1
    DATEFORMAT 'auto'
    TIMEFORMAT 'auto';
    """
    
    # Execute COPY command using Redshift Data API
    response = redshift_data.execute_statement(
        ClusterIdentifier=REDSHIFT_CLUSTER,
        Database=REDSHIFT_DATABASE,
        DbUser=REDSHIFT_USER,
        Sql=copy_sql
    )
    
    statement_id = response['Id']
    
    # Wait for completion (synchronous for Lambda)
    import time
    while True:
        status_response = redshift_data.describe_statement(Id=statement_id)
        status = status_response['Status']
        
        if status == 'FINISHED':
            print(f"✅ COPY command completed: {statement_id}")
            break
        elif status in ['FAILED', 'ABORTED']:
            error = status_response.get('Error', 'Unknown error')
            raise Exception(f"COPY command failed: {error}")
        
        time.sleep(1)  # Poll every second

def send_notification(subject, message):
    """Send SNS notification to data engineering team"""
    
    sns.publish(
        TopicArn=SNS_TOPIC,
        Subject=subject,
        Message=message
    )
```

**Step 4: Compliance and Auditing**

```bash
# Enable CloudTrail logging for Transfer Family API calls
aws cloudtrail put-event-selectors \
  --trail-name hedgefund-audit-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::hedgefund-vendor-data/*"]
    }]
  }]'

# Enable S3 access logging (file-level audit)
aws s3api put-bucket-logging \
  --bucket hedgefund-vendor-data \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "hedgefund-audit-logs",
      "TargetPrefix": "s3-access-logs/"
    }
  }'

# Create CloudWatch Logs Insights query for SFTP sessions
cat > sftp-audit-query.txt <<'EOF'
fields @timestamp, username, client_ip, file_path, bytes_transferred, status
| filter event_type = "PutObject"
| stats count() as uploads, sum(bytes_transferred) as total_bytes by username
| sort total_bytes desc
EOF

# Run query (last 7 days)
aws logs start-query \
  --log-group-name /aws/transfer/s-1234567890abcdef0 \
  --start-time $(date -u -d '7 days ago' +%s) \
  --end-time $(date -u +%s) \
  --query-string "$(cat sftp-audit-query.txt)"
```

**Step 5: High Availability and Disaster Recovery**

```python
#!/usr/bin/env python3
"""
Multi-AZ Transfer Family with automated failover
"""

import boto3

transfer = boto3.client('transfer', region_name='us-east-1')
route53 = boto3.client('route53')

def create_multi_az_sftp():
    """
    Create Transfer Family server in VPC (Multi-AZ)
    """
    
    # Create server with VPC endpoint (Multi-AZ)
    response = transfer.create_server(
        EndpointType='VPC',
        EndpointDetails={
            'SubnetIds': [
                'subnet-0a1b2c3d',  # us-east-1a
                'subnet-1a2b3c4d'   # us-east-1b
            ],
            'VpcId': 'vpc-0a1b2c3d',
            'SecurityGroupIds': ['sg-0a1b2c3d'],
            'AddressAllocationIds': [
                'eipalloc-0a1b2c3d',  # Elastic IP 1
                'eipalloc-1a2b3c4d'   # Elastic IP 2
            ]
        },
        Protocols=['SFTP'],
        IdentityProviderType='SERVICE_MANAGED'
    )
    
    server_id = response['ServerId']
    print(f"✅ Created Multi-AZ SFTP server: {server_id}")
    
    # Get Elastic IPs
    server_details = transfer.describe_server(ServerId=server_id)
    endpoint_ips = server_details['Server']['EndpointDetails']['AddressAllocationIds']
    
    # Create Route 53 health checks for both IPs
    for idx, eip_alloc_id in enumerate(endpoint_ips):
        # Get Elastic IP address
        ec2 = boto3.client('ec2', region_name='us-east-1')
        eip_response = ec2.describe_addresses(AllocationIds=[eip_alloc_id])
        ip_address = eip_response['Addresses'][0]['PublicIp']
        
        # Create health check
        health_check = route53.create_health_check(
            HealthCheckConfig={
                'Type': 'TCP',
                'IPAddress': ip_address,
                'Port': 22,  # SFTP port
                'RequestInterval': 30,
                'FailureThreshold': 3
            }
        )
        
        health_check_id = health_check['HealthCheck']['Id']
        print(f"✅ Created health check for {ip_address}: {health_check_id}")
    
    print(f"\n{'='*60}")
    print(f"Multi-AZ SFTP Server Configuration:")
    print(f"  Server ID: {server_id}")
    print(f"  Availability Zones: us-east-1a, us-east-1b")
    print(f"  Elastic IPs: {endpoint_ips}")
    print(f"  Uptime SLA: 99.99% (Multi-AZ)")
    print(f"{'='*60}")

if __name__ == '__main__':
    create_multi_az_sftp()
```

**Results:**

| Metric | On-Premises SFTP | AWS Transfer Family | Improvement |
|--------|------------------|---------------------|-------------|
| **Uptime** | 98.5% (hardware failures) | 99.99% (Multi-AZ) | **1.5% improvement** |
| **Operational Cost** | $120K/year (hardware, staff) | $45K/year (Transfer Family) | **62% reduction** |
| **Compliance** | Manual audits (quarterly) | Automated (CloudTrail, S3 logs) | **100% coverage** |
| **File Processing Time** | 2-4 hours (manual) | 5 minutes (automated Lambda) | **96% faster** |
| **Security Incidents** | 2/year (unauthorized access) | 0/year (IAM scoped access) | **100% reduction** |

**Cost Analysis:**

```
Monthly Cost (AWS Transfer Family):
├─ Transfer Family SFTP server: $0.30/hour × 730 hours = $219/month
├─ Data upload: 150 GB/day × 30 days × $0.04/GB = $180/month
├─ Data download: 10 GB/day × 30 days × $0.04/GB = $12/month
├─ S3 storage: 4.5 TB (150 GB/day × 30 days) × $0.023/GB = $103.50/month
├─ Lambda invocations: 50 vendors × 30 days × $0.20/1M = $0.03/month
├─ Redshift Serverless: 50 RPUs × $0.36/hour × 8 hours/day = $540/month
├─ CloudWatch Logs: $5/month
├─ CloudTrail: $2/month
└─ TOTAL: $1,061.53/month = $12,738/year

vs On-Premises SFTP:
├─ SFTP server hardware: $30,000 CapEx (amortized over 3 years = $10,000/year)
├─ Network infrastructure: $15,000/year
├─ IT staff (0.5 FTE): $75,000/year
├─ Compliance audits: $20,000/year
└─ TOTAL: $120,000/year

AWS Solution Annual Cost: $12,738/year
SAVINGS: $107,262/year (89% reduction) ✅
```

**Key Lessons:**

1. **AWS Transfer Family eliminates hardware management** while maintaining SFTP compatibility
2. **Scoped IAM roles** provide vendor isolation (each vendor can only access their S3 prefix)
3. **Automated processing with Lambda** reduces manual work from hours to minutes
4. **CloudTrail + S3 access logs** provide comprehensive audit trail for SOC 2 compliance
5. **Multi-AZ deployment** achieves 99.99% uptime (vs 98.5% on-premises)

---

**Q16: Heterogeneous Multi-Database Migration (SQL Server + MySQL → Aurora)**

**Scenario:**

You're a data engineer at a SaaS company consolidating databases:

**Current State:**
- 3 SQL Server databases (customer CRM, billing, support tickets) - 8 TB total
- 5 MySQL databases (product catalog, inventory, order processing, user accounts, analytics) - 12 TB total
- Total: 8 databases, 20 TB data
- On-premises datacenter with EOL (end-of-life) in 6 months
- Applications: 25 microservices depend on these databases
- Downtime tolerance: < 1 hour for migration cutover

**Requirements:**
1. Migrate all 8 databases to AWS Aurora PostgreSQL (standardize on single engine)
2. Schema conversion: SQL Server + MySQL → PostgreSQL
3. Zero data loss (no missing transactions during cutover)
4. Minimize application changes (compatibility layer needed)
5. Complete migration within 3 months

**Challenge:** How do you orchestrate a heterogeneous migration from SQL Server + MySQL to Aurora PostgreSQL with minimal application impact?

---

### Answer:

Use AWS DMS + AWS Schema Conversion Tool (SCT) for heterogeneous migration with zero data loss.

**Results:**

| Metric | On-Premises | Aurora PostgreSQL | Improvement |
|--------|-------------|-------------------|-------------|
| **Migration Time** | N/A | 10 weeks | Within 3-month SLA ✅ |
| **Downtime** | N/A | 45 minutes | Within 1-hour SLA ✅ |
| **Cost Savings** | $300K/year | $63K/year | **79% reduction** |

**Key Implementation:**
- AWS SCT: Convert SQL Server/MySQL schemas → PostgreSQL (95% automated)
- DMS Full Load + CDC: 20 TB migrated in 7 days with zero data loss  
- Gradual cutover: 0% → 100% traffic over 6 weeks (reduces risk)
- Database consolidation: 8 databases → 1 Aurora cluster (8 schemas)

---

**Q17: AWS Migration Hub Orchestration for Complex Application Migration**

**Scenario:**

You're a data engineer at an e-commerce company migrating a complex 3-tier application:

**Current State:**
- 15 application servers (Java Spring Boot microservices)
- 3 databases (PostgreSQL primary, 2 read replicas) - 5 TB total
- 5 TB file storage (NFS for product images, PDFs)
- 50 TB object storage (S3-compatible on-prem Minio)
- 12 batch ETL jobs (Python scripts, Airflow orchestration)
- On-premises datacenter (retiring in 4 months)

**Requirements:**
1. Track migration progress across all components
2. Discover dependencies between servers and databases
3. Minimize downtime (< 2 hours for cutover)
4. Ensure data consistency across migration waves
5. Generate compliance reports for stakeholders

**Challenge:** How do you use AWS Migration Hub to orchestrate and track this complex migration?

---

### Answer:

**Solution: AWS Migration Hub + Application Discovery Service + AWS DMS**

**Architecture:**

```
AWS Migration Hub (Central Tracking)
├─ Discovery Phase
│  └─ AWS Application Discovery Service
│     ├─ Deploy agents on 15 app servers
│     ├─ Discover dependencies (app ↔ database connections)
│     └─ Generate migration recommendations
│
├─ Migration Phase (4 Waves)
│  ├─ Wave 1: File storage (DataSync: 5 TB NFS → S3)
│  ├─ Wave 2: Object storage (S3 Sync: 50 TB Minio → S3)
│  ├─ Wave 3: Databases (DMS: PostgreSQL → Aurora)
│  └─ Wave 4: Application servers (Replatform: EC2/ECS)
│
└─ Tracking & Reporting
   ├─ Real-time migration status
   ├─ Dependency visualization
   └─ Compliance reports (weekly)
```

**Implementation:**

```bash
# Step 1: Enable Application Discovery Service
aws discovery start-data-collection-by-agent-ids \
  --agent-ids agentId-1 agentId-2 ... agentId-15

# Step 2: Create migration project in Migration Hub
aws migrationhub create-progress-update-stream \
  --progress-update-stream-name ecommerce-migration

# Step 3: Track components
aws migrationhub associate-discovered-resource \
  --progress-update-stream-name ecommerce-migration \
  --migration-task-name wave-1-file-storage \
  --discovered-resource '{
    "ConfigurationId": "d-server-0123456789abcdef",
    "Description": "NFS file server"
  }'

# Step 4: Update migration status
aws migrationhub put-resource-attributes \
  --progress-update-stream-name ecommerce-migration \
  --migration-task-name wave-3-database \
  --resource-attribute-list '[{
    "Type": "STRING",
    "Key": "STATUS",
    "Value": "IN_PROGRESS"
  }]'
```

**Results:**

| Metric | Before Migration Hub | With Migration Hub | Improvement |
|--------|----------------------|--------------------|-------------|
| **Migration Visibility** | Manual spreadsheets | Real-time dashboard | **100% automated** |
| **Dependency Discovery** | Manual (2 weeks) | Automated (24 hours) | **93% faster** |
| **Migration Duration** | 6 months (estimated) | 12 weeks (actual) | **50% faster** |
| **Downtime** | 8 hours (estimated) | 1.5 hours (actual) | **81% reduction** |

**Key Lessons:**
1. **Application Discovery Service** automatically maps dependencies (saves 2 weeks vs manual discovery)
2. **Wave-based migration** (4 waves) allows testing between waves (reduces risk)
3. **Migration Hub dashboard** provides single pane of glass for executive reporting
4. **DMS + DataSync integration** with Migration Hub enables automated status tracking

---

**Q18: Database Migration Validation and Testing Strategies**

**Scenario:**

You're a data engineer migrating a critical financial database from Oracle to Aurora PostgreSQL:

**Current State:**
- Oracle database: 10 TB, 500 tables, 1.2 billion rows
- Complex business logic: 2,000 stored procedures, 500 views
- Query workload: 10,000 queries/hour (mix of OLTP and analytics)
- Zero-tolerance for data discrepancies (financial regulations)

**Requirements:**
1. Validate 100% data accuracy (row counts, checksums, referential integrity)
2. Performance testing (ensure Aurora matches or exceeds Oracle query performance)
3. Functional testing (validate stored procedure conversions)
4. Load testing (10,000 queries/hour with < 100ms p99 latency)
5. Generate validation reports for auditors

**Challenge:** Design a comprehensive testing strategy to ensure migration success

---

### Answer:

**Validation Strategy:**

```python
#!/usr/bin/env python3
"""
Comprehensive database migration validation
Oracle → Aurora PostgreSQL
"""

import cx_Oracle
import psycopg2
import hashlib

class MigrationValidator:
    def __init__(self):
        self.oracle_conn = cx_Oracle.connect('user/pass@oracle-host:1521/ORCL')
        self.aurora_conn = psycopg2.connect(
            host='aurora-cluster.xyz.us-east-1.rds.amazonaws.com',
            database='migrated_db',
            user='admin',
            password='password'
        )
    
    def validate_row_counts(self):
        """Test 1: Compare row counts for all tables"""
        
        oracle_cur = self.oracle_conn.cursor()
        aurora_cur = self.aurora_conn.cursor()
        
        # Get Oracle table counts
        oracle_cur.execute("""
            SELECT table_name, num_rows 
            FROM user_tables 
            ORDER BY table_name
        """)
        oracle_counts = {row[0]: row[1] for row in oracle_cur.fetchall()}
        
        # Get Aurora table counts
        aurora_cur.execute("""
            SELECT table_name, 
                   (xpath('//row/c/text()', 
                          query_to_xml(
                              format('SELECT COUNT(*) AS c FROM %I.%I', 
                                     table_schema, table_name), 
                              false, true, '')))[1]::text::int AS row_count
            FROM information_schema.tables
            WHERE table_schema = 'public'
            ORDER BY table_name
        """)
        aurora_counts = {row[0]: row[1] for row in aurora_cur.fetchall()}
        
        # Compare
        discrepancies = []
        for table in oracle_counts:
            oracle_count = oracle_counts[table]
            aurora_count = aurora_counts.get(table, 0)
            
            if oracle_count != aurora_count:
                discrepancies.append({
                    'table': table,
                    'oracle': oracle_count,
                    'aurora': aurora_count,
                    'diff': abs(oracle_count - aurora_count)
                })
        
        if discrepancies:
            print(f"❌ Row count discrepancies found:")
            for d in discrepancies:
                print(f"   {d['table']}: Oracle={d['oracle']}, Aurora={d['aurora']} (diff={d['diff']})")
            return False
        else:
            print(f"✅ All table row counts match ({len(oracle_counts)} tables)")
            return True
    
    def validate_checksums(self):
        """Test 2: Compare data checksums"""
        
        # Example: checksum for transactions table
        oracle_cur = self.oracle_conn.cursor()
        aurora_cur = self.aurora_conn.cursor()
        
        # Oracle checksum
        oracle_cur.execute("""
            SELECT SUM(ORA_HASH(
                transaction_id || amount || transaction_date
            )) AS checksum
            FROM transactions
        """)
        oracle_checksum = oracle_cur.fetchone()[0]
        
        # Aurora checksum (equivalent using MD5)
        aurora_cur.execute("""
            SELECT SUM(('x' || LEFT(MD5(
                transaction_id::text || amount::text || transaction_date::text
            ), 8))::bit(32)::bigint) AS checksum
            FROM transactions
        """)
        aurora_checksum = aurora_cur.fetchone()[0]
        
        if oracle_checksum == aurora_checksum:
            print(f"✅ Data checksums match (checksum={oracle_checksum})")
            return True
        else:
            print(f"❌ Checksum mismatch: Oracle={oracle_checksum}, Aurora={aurora_checksum}")
            return False
    
    def performance_comparison(self):
        """Test 3: Compare query performance"""
        
        import time
        
        test_queries = [
            "SELECT COUNT(*) FROM transactions WHERE amount > 1000",
            "SELECT customer_id, SUM(amount) FROM transactions GROUP BY customer_id",
            "SELECT * FROM transactions ORDER BY transaction_date DESC LIMIT 100"
        ]
        
        results = []
        
        for query in test_queries:
            # Oracle timing
            oracle_cur = self.oracle_conn.cursor()
            oracle_start = time.time()
            oracle_cur.execute(query)
            oracle_cur.fetchall()
            oracle_duration = time.time() - oracle_start
            
            # Aurora timing
            aurora_cur = self.aurora_conn.cursor()
            aurora_start = time.time()
            aurora_cur.execute(query)
            aurora_cur.fetchall()
            aurora_duration = time.time() - aurora_start
            
            speedup = oracle_duration / aurora_duration
            results.append({
                'query': query[:50] + '...',
                'oracle_ms': oracle_duration * 1000,
                'aurora_ms': aurora_duration * 1000,
                'speedup': speedup
            })
            
            print(f"Query: {query[:50]}...")
            print(f"  Oracle: {oracle_duration*1000:.2f}ms")
            print(f"  Aurora: {aurora_duration*1000:.2f}ms")
            print(f"  Speedup: {speedup:.2f}x")
        
        avg_speedup = sum(r['speedup'] for r in results) / len(results)
        
        if avg_speedup >= 1.0:
            print(f"✅ Aurora performance meets/exceeds Oracle ({avg_speedup:.2f}x average)")
            return True
        else:
            print(f"⚠️  Aurora slower than Oracle ({avg_speedup:.2f}x average)")
            return False

validator = MigrationValidator()
validator.validate_row_counts()
validator.validate_checksums()
validator.performance_comparison()
```

**Results:**

| Test Category | Oracle | Aurora PostgreSQL | Pass/Fail |
|---------------|--------|-------------------|-----------|
| **Row Counts** | 1.2B rows | 1.2B rows | ✅ Pass |
| **Data Checksums** | Match | Match | ✅ Pass |
| **Query Performance** | 145ms avg | 95ms avg (1.5x faster) | ✅ Pass |
| **Stored Procedures** | 2,000 | 2,000 converted | ✅ Pass (98% automated) |
| **Referential Integrity** | 500 foreign keys | 500 foreign keys | ✅ Pass |

---

**Q19: Cost Optimization for Large-Scale Migrations**

**Scenario:**

You need to migrate 1 PB of data from on-premises to AWS S3:

**Current State:**
- 1 PB data on Dell EMC Isilon NFS storage
- 10 Gbps Direct Connect to AWS
- Network transfer cost: $0.02/GB over Direct Connect
- Timeline: 3 months

**Challenge:** Minimize migration cost while meeting 3-month timeline

---

### Answer:

**Cost Optimization Strategy:**

```
Option 1: DataSync over Direct Connect
├─ Data transfer: 1 PB × $0.02/GB = $20,480
├─ DataSync: 1 PB × $0.0125/GB = $12,800
├─ Direct Connect (10 Gbps): $2,250/month × 3 = $6,750
└─ TOTAL: $40,030

Option 2: AWS Snowball Edge (10 devices)
├─ Snowball devices: 10 × $300 = $3,000
├─ Shipping: $1,500
├─ No network transfer costs
└─ TOTAL: $4,500 ✅ (89% cheaper)

Winner: Snowball Edge
```

**Cost Breakdown:**

| Cost Component | DataSync | Snowball Edge | Savings |
|----------------|----------|---------------|---------|
| **Data Transfer** | $20,480 | $0 | $20,480 |
| **Service Fee** | $12,800 | $3,000 | $9,800 |
| **Network** | $6,750 | $0 | $6,750 |
| **TOTAL** | $40,030 | $4,500 | **$35,530 (89%)** |

**Key Lessons:**
- **Snowball Edge is 89% cheaper** than network transfer for > 100 TB migrations
- **Break-even point**: ~80 TB (above this, Snowball is cheaper)
- **Timeline consideration**: Snowball takes 2-3 weeks longer than DataSync

---

**Q20: Real-Time Data Replication with DMS + Kinesis for Analytics**

**Scenario:**

You're a data engineer building a real-time analytics platform:

**Current State:**
- Production MySQL database (100 GB, 50 tables)
- Need real-time changes in data lake (S3) for analytics
- Latency requirement: < 5 minutes (near real-time)
- Analytics tools: Athena, QuickSight, SageMaker

**Challenge:** Stream database changes to S3 in real-time for analytics

---

### Answer:

**Architecture: DMS CDC → Kinesis Data Streams → Kinesis Firehose → S3**

```
MySQL (Production)
│
├─ DMS Replication Instance (CDC mode)
│  └─ Captures INSERT, UPDATE, DELETE events
│
└─→ Amazon Kinesis Data Streams
    └─ Real-time event stream (1000 events/sec)
        │
        └─→ Kinesis Data Firehose
            ├─ Batching: 5 minutes or 5 MB
            ├─ Transformation: Lambda (JSON→Parquet)
            └─ Compression: Snappy
                │
                └─→ Amazon S3 (Data Lake)
                    └─ s3://analytics-lake/mysql-cdc/
                        ├─ year=2024/month=06/day=22/hour=14/
                        └─ customers_20240622_14.parquet
                            │
                            └─→ AWS Glue Crawler (schema discovery)
                                └─→ Amazon Athena (SQL queries)
                                    └─→ QuickSight (dashboards)
```

**Implementation:**

```bash
# Create Kinesis Data Stream
aws kinesis create-stream \
  --stream-name mysql-cdc-stream \
  --shard-count 4 \
  --region us-east-1

# Create DMS target endpoint (Kinesis)
aws dms create-endpoint \
  --endpoint-identifier kinesis-target \
  --endpoint-type target \
  --engine-name kinesis \
  --kinesis-settings '{
    "StreamArn": "arn:aws:kinesis:us-east-1:123456789012:stream/mysql-cdc-stream",
    "ServiceAccessRoleArn": "arn:aws:iam::123456789012:role/DMSKinesisRole",
    "MessageFormat": "json"
  }'

# Create Firehose delivery stream
aws firehose create-delivery-stream \
  --delivery-stream-name mysql-to-s3 \
  --delivery-stream-type KinesisStreamAsSource \
  --kinesis-stream-source-configuration '{
    "KinesisStreamARN": "arn:aws:kinesis:us-east-1:123456789012:stream/mysql-cdc-stream",
    "RoleARN": "arn:aws:iam::123456789012:role/FirehoseRole"
  }' \
  --extended-s3-destination-configuration '{
    "BucketARN": "arn:aws:s3:::analytics-lake",
    "Prefix": "mysql-cdc/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
    "BufferingHints": {
      "SizeInMBs": 5,
      "IntervalInSeconds": 300
    },
    "CompressionFormat": "SNAPPY",
    "DataFormatConversionConfiguration": {
      "SchemaConfiguration": {
        "DatabaseName": "analytics_db",
        "TableName": "mysql_cdc",
        "Region": "us-east-1"
      },
      "OutputFormatConfiguration": {
        "Serializer": {
          "ParquetSerDe": {}
        }
      }
    }
  }'
```

**Results:**

| Metric | Batch ETL (Daily) | Real-Time CDC | Improvement |
|--------|-------------------|---------------|-------------|
| **Data Latency** | 24 hours | 3 minutes | **99.8% faster** |
| **Dashboard Freshness** | Yesterday's data | Real-time | **Immediate insights** |
| **Cost** | $500/month (EMR batch) | $250/month (Kinesis) | **50% cheaper** |

**Key Lessons:**
1. **DMS CDC mode** captures real-time database changes with < 1 second latency
2. **Kinesis Firehose** automatically converts JSON → Parquet (90% storage savings)
3. **5-minute batching** balances latency vs cost (vs 1-minute batching: 5x more expensive)
4. **Real-time analytics** enables business to react 99.8% faster than daily batch ETL

---
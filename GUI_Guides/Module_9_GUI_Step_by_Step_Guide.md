# MODULE 9: Security, Identity & Compliance - GUI Step-by-Step Hands-On Guide

## Table of Contents
- [Exercise 9.1: Secure Data Pipeline with IAM, KMS, and Secrets Manager](#exercise-91-secure-data-pipeline-with-iam-kms-and-secrets-manager)
- [Exercise 9.2: Data Lake Governance with AWS Lake Formation](#exercise-92-data-lake-governance-with-aws-lake-formation)
- [Exercise 9.3: Threat Detection with GuardDuty and Macie](#exercise-93-threat-detection-with-guardduty-and-macie)

---

# Exercise 9.1: Secure Data Pipeline with IAM, KMS, and Secrets Manager

**Duration:** 2-3 hours | **Estimated Cost:** ~$5-10 for lab session

## 🎯 Learning Objectives
- Encrypt data at rest with AWS KMS
- Store database credentials securely with Secrets Manager
- Create least-privilege IAM policies
- Enable audit logging with CloudTrail
- Monitor compliance with AWS Config
- Implement end-to-end encryption

---

## PART 1: Create KMS Encryption Keys

### Step 1: Navigate to KMS Console
1. AWS Console → Search **"KMS"** (Key Management Service)
2. Click **"AWS Key Management Service"**
3. In left sidebar, click **"Customer managed keys"**

### Step 2: Create Customer Managed Key
1. Click **"Create key"**
2. **Key type:** Select **"Symmetric"**
3. **Key usage:** **"Encrypt and decrypt"**
4. Click **"Next"**

### Step 3: Add Labels and Tags
1. **Alias:** Type: `data-lake-encryption`
2. **Description:** `Master key for data lake encryption`
3. **Tags (optional):**
   - Key: `Environment` | Value: `Production`
   - Key: `Purpose` | Value: `DataEncryption`
4. Click **"Next"**

### Step 4: Define Key Administrative Permissions
1. **Key administrators:** Select your IAM user/role
2. ✅ Check **"Allow key administrators to delete this key"**
3. Click **"Next"**

### Step 5: Define Key Usage Permissions
1. **Key users:** Select:
   - IAM roles that will use this key (e.g., Lambda execution roles, Glue roles)
   - Your user (for testing)
2. **Other AWS accounts:** Leave empty (unless multi-account)
3. Review the **Key policy** (auto-generated JSON)
4. Click **"Next"**

### Step 6: Review and Create
1. Review all settings
2. Click **"Finish"**
3. **Key created!** Note the **Key ID** (e.g., `abcd1234-5678-90ab-cdef-1234567890ab`)

### Step 7: Enable Automatic Key Rotation
1. Click on your key: `data-lake-encryption`
2. Go to **"Key rotation"** tab
3. ✅ Check **"Automatically rotate this KMS key every year"**
4. Click **"Save"**

**Why rotate?** Reduces impact if key is compromised.

---

## PART 2: Store Secrets in AWS Secrets Manager

### Step 8: Navigate to Secrets Manager
1. AWS Console → Search **"Secrets Manager"**
2. Click **"AWS Secrets Manager"**
3. Click **"Store a new secret"**

### Step 9: Choose Secret Type for RDS
1. **Secret type:** Select **"Credentials for Amazon RDS database"**
2. **User name:** Type: `admin`
3. **Password:** Type a strong password OR click **"Generate random password"**
4. **Encryption key:** Select `data-lake-encryption` (the KMS key we created)
5. **Database:** Select your RDS instance (or leave empty if not created yet)
6. Click **"Next"**

### Step 10: Configure Secret Name and Rotation
1. **Secret name:** Type: `prod/rds/credentials`
2. **Description:** `RDS master credentials for production ETL pipeline`
3. **Automatic rotation:** ✅ **"Enable automatic rotation"**
4. **Rotation schedule:** **"30 days"**
5. **Select rotation function:** **"Create a new Lambda function"**
   - This creates a Lambda that rotates the RDS password automatically
6. Click **"Next"**

### Step 11: Review and Store
1. Review secret configuration
2. Click **"Store"**
3. **IMPORTANT:** Click **"Retrieve secret value"** and save the secret ARN
   - Example: `arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/rds/credentials-AbCdEf`

### Step 12: Create Secret for Redshift
1. Click **"Store a new secret"** again
2. **Secret type:** Select **"Other type of secret"**
3. **Key/value pairs:**
   - `host`: Your Redshift endpoint
   - `port`: `5439`
   - `database`: `analytics`
   - `username`: `admin`
   - `password`: Strong password
4. **Encryption key:** Select `data-lake-encryption`
5. Click **"Next"**
6. **Secret name:** `prod/redshift/credentials`
7. **Rotation:** Enable 30-day rotation (if Redshift cluster exists)
8. **Store**

---

## PART 3: Create Least-Privilege IAM Policies

### Step 13: Create Custom IAM Policy for Lambda
1. **IAM Console** → **"Policies"** → **"Create policy"**
2. Click **"JSON"** tab
3. Paste this policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:us-east-1:*:secret:prod/rds/credentials-*",
        "arn:aws:secretsmanager:us-east-1:*:secret:prod/redshift/credentials-*"
      ]
    },
    {
      "Sid": "DecryptSecrets",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:*:key/*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "WriteEncryptedS3",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::data-lake-*/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "EncryptS3Objects",
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:*:alias/data-lake-encryption"
    }
  ]
}
```

4. Click **"Next"**
5. **Policy name:** `DataPipelineLambdaSecurePolicy`
6. **Description:** `Secure access to secrets, KMS, and S3 for data pipeline`
7. Click **"Create policy"**

### Step 14: Create IAM Role for Lambda with Secure Policy
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** **"Lambda"**
3. Click **"Next"**
4. Attach policies:
   - ✅ **AWSLambdaBasicExecutionRole** (CloudWatch Logs)
   - ✅ **DataPipelineLambdaSecurePolicy** (our custom policy)
5. Click **"Next"**
6. **Role name:** `SecureDataPipelineLambdaRole`
7. Click **"Create role"**

---

## PART 4: Create Secure Lambda Function

### Step 15: Create Lambda Function
1. **Lambda Console** → **"Create function"**
2. **Function name:** `secure-rds-extract`
3. **Runtime:** **"Python 3.12"**
4. **Existing role:** Select `SecureDataPipelineLambdaRole`
5. **Create function**

### Step 16: Write Secure Code
Replace code:

```python
import json
import boto3
import psycopg2
import os

secrets_client = boto3.client('secretsmanager')
s3_client = boto3.client('s3')

SECRET_ARN = os.environ.get('RDS_SECRET_ARN')
DEST_BUCKET = os.environ.get('DEST_BUCKET')
KMS_KEY_ID = os.environ.get('KMS_KEY_ID')

def lambda_handler(event, context):
    print("Retrieving RDS credentials from Secrets Manager")
    
    # Get credentials securely
    credentials = get_db_credentials()
    
    # Connect to RDS with SSL
    conn = psycopg2.connect(
        host=credentials['host'],
        port=credentials['port'],
        database=credentials['database'],
        user=credentials['username'],
        password=credentials['password'],
        sslmode='require'  # Force SSL
    )
    
    cursor = conn.cursor()
    
    # Extract data
    cursor.execute("SELECT * FROM sales_data WHERE order_date = %s", (event.get('date'),))
    rows = cursor.fetchall()
    
    print(f"Extracted {len(rows)} rows")
    
    # Convert to CSV
    csv_data = "\\n".join([",".join(map(str, row)) for row in rows])
    
    # Upload to S3 with encryption
    s3_key = f"extracts/{event.get('date')}/sales.csv"
    
    s3_client.put_object(
        Bucket=DEST_BUCKET,
        Key=s3_key,
        Body=csv_data.encode('utf-8'),
        ServerSideEncryption='aws:kms',
        SSEKMSKeyId=KMS_KEY_ID
    )
    
    print(f"✓ Uploaded encrypted data to s3://{DEST_BUCKET}/{s3_key}")
    
    cursor.close()
    conn.close()
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'rows_extracted': len(rows),
            's3_location': f's3://{DEST_BUCKET}/{s3_key}'
        })
    }

def get_db_credentials():
    """Retrieve database credentials from Secrets Manager"""
    try:
        response = secrets_client.get_secret_value(SecretId=SECRET_ARN)
        secret = json.loads(response['SecretString'])
        return secret
    except Exception as e:
        print(f"Error retrieving secret: {str(e)}")
        raise
```

Click **"Deploy"**

### Step 17: Add Environment Variables
1. **"Configuration"** → **"Environment variables"** → **"Edit"**
2. Add variables:
   - **RDS_SECRET_ARN:** Your secret ARN from Step 11
   - **DEST_BUCKET:** `data-lake-encrypted-[account-id]`
   - **KMS_KEY_ID:** Your KMS key ID from Step 6
3. **Save**

### Step 18: Create Encrypted S3 Bucket
1. **S3 Console** → **"Create bucket"**
2. **Name:** `data-lake-encrypted-[account-id]`
3. **Default encryption:** Select **"Enable"**
4. **Encryption type:** **"AWS Key Management Service key (SSE-KMS)"**
5. **AWS KMS key:** Select `data-lake-encryption`
6. **Bucket Key:** ✅ **"Enable"** (reduces KMS costs)
7. **Create bucket**

---

## PART 5: Enable CloudTrail for Audit Logging

### Step 19: Create S3 Bucket for CloudTrail Logs
1. **S3 Console** → **"Create bucket"**
2. **Name:** `cloudtrail-logs-[account-id]`
3. **Encryption:** **SSE-S3** (default)
4. **Create bucket**

### Step 20: Configure Bucket Policy for CloudTrail
1. Click on bucket → **"Permissions"** → **"Bucket policy"** → **"Edit"**
2. Paste this policy (replace `ACCOUNT-ID`):

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
      "Resource": "arn:aws:s3:::cloudtrail-logs-ACCOUNT-ID"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::cloudtrail-logs-ACCOUNT-ID/AWSLogs/ACCOUNT-ID/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
```

3. **Save changes**

### Step 21: Create CloudTrail Trail
1. **CloudTrail Console** → **"Trails"** → **"Create trail"**
2. **Trail name:** `data-security-audit-trail`
3. **Storage location:** Select existing bucket: `cloudtrail-logs-[account-id]`
4. **Log file SSE-KMS encryption:** ✅ **"Enabled"**
   - **New:** Select `data-lake-encryption` key
5. **Log file validation:** ✅ **"Enabled"** (detects tampering)
6. Click **"Next"**

### Step 22: Choose Events to Log
1. **Event type:**
   - ✅ **"Management events"** (API calls)
   - ✅ **"Data events"** (S3 object operations)
2. **Management events:**
   - **API activity:** **"Read"** and **"Write"**
3. **Data events:**
   - **S3 buckets:** Click **"Add"**
   - Select: `data-lake-encrypted-[account-id]`
   - ✅ **"Read"** and ✅ **"Write"**
4. Click **"Next"**

### Step 23: Review and Create Trail
1. Review all settings
2. Click **"Create trail"**
3. **Logging starts immediately**

### Step 24: View CloudTrail Events
1. **CloudTrail Console** → **"Event history"**
2. You'll see recent API calls:
   - **Event name:** PutObject, GetObject, CreateBucket, etc.
   - **User name:** IAM user/role that made the call
   - **Event time:** When it happened
3. Click on any event to see full details (JSON)

---

## PART 6: Monitor Compliance with AWS Config

### Step 25: Set Up AWS Config
1. **AWS Config Console** → **"Get started"** (if first time)
2. Click **"1-click setup"** OR **"Manual setup"**

### Step 26: Configure Recording
1. **Resource types to record:** Select **"All resources"**
2. ✅ **"Include global resources"** (IAM, CloudFront)
3. **AWS Config role:** **"Create AWS Config service-linked role"** (auto)
4. Click **"Next"**

### Step 27: Configure Delivery Method
1. **S3 bucket:** **"Create a bucket"** OR select existing
2. **Bucket name:** `config-snapshots-[account-id]`
3. **SNS topic:** Optional (for notifications)
4. Click **"Next"**

### Step 28: Add Compliance Rules
1. Click **"Add rule"**
2. **Rule name:** Search and select:
   - **s3-bucket-server-side-encryption-enabled**
   - **rds-encryption-enabled**
   - **cloudtrail-enabled**
   - **iam-user-mfa-enabled**
3. For each rule:
   - Click **"Add rule"**
   - Accept default settings
   - Click **"Save"**

### Step 29: View Compliance Dashboard
1. **AWS Config Console** → **"Dashboard"**
2. You'll see:
   - **Compliant resources:** Green checkmark
   - **Non-compliant resources:** Red X
   - **Rules:** Number of active compliance rules
3. Click on any non-compliant resource to see details

### Step 30: Query Config with SQL
1. **AWS Config** → **"Advanced queries"**
2. Example query - Find unencrypted S3 buckets:
   ```sql
   SELECT 
     resourceId,
     resourceType,
     configuration.serverSideEncryptionConfiguration
   WHERE 
     resourceType = 'AWS::S3::Bucket'
     AND configuration.serverSideEncryptionConfiguration IS NULL
   ```
3. Click **"Run query"**
4. Shows buckets without encryption

---

## ✅ Exercise 9.1 Completion Checklist

- [ ] Created customer-managed KMS key
- [ ] Enabled automatic key rotation
- [ ] Stored RDS credentials in Secrets Manager
- [ ] Stored Redshift credentials in Secrets Manager
- [ ] Configured automatic secret rotation (30 days)
- [ ] Created least-privilege IAM policy
- [ ] Created IAM role for Lambda with secure policies
- [ ] Created Lambda function that retrieves secrets
- [ ] Implemented SSL/TLS for database connections
- [ ] Created S3 bucket with KMS encryption
- [ ] Uploaded encrypted data to S3
- [ ] Created S3 bucket for CloudTrail logs
- [ ] Configured bucket policy for CloudTrail access
- [ ] Created CloudTrail trail with KMS encryption
- [ ] Enabled log file validation
- [ ] Configured data events for S3 buckets
- [ ] Viewed CloudTrail event history
- [ ] Set up AWS Config resource recording
- [ ] Added compliance rules (S3, RDS, CloudTrail, IAM)
- [ ] Viewed compliance dashboard
- [ ] Ran SQL queries against Config data

**🎉 Congratulations!** You've completed Exercise 9.1!

---

# Exercise 9.2: Data Lake Governance with AWS Lake Formation

**Duration:** 2-3 hours | **Estimated Cost:** ~$5-10 for lab session

## 🎯 Learning Objectives
- Set up AWS Lake Formation for centralized governance
- Implement row-level security (data filters)
- Configure column-level security (column masking)
- Use tag-based access control (TBAC)
- Grant fine-grained permissions
- Audit data access

---

## PART 1: Set Up Lake Formation

### Step 1: Navigate to Lake Formation Console
1. AWS Console → Search **"Lake Formation"**
2. Click **"AWS Lake Formation"**
3. If first time, click **"Get started"**

### Step 2: Add Data Lake Administrators
1. **"Settings"** → **"Data lake administrators"**
2. Click **"Add"**
3. Select your IAM user or role
4. Click **"Save"**

**Lake Formation admin can grant all permissions.**

### Step 3: Register S3 Data Lake Location
1. **Lake Formation** → **"Data lake locations"** (left sidebar)
2. Click **"Register location"**
3. **Amazon S3 path:** Browse and select your data bucket
   - Example: `s3://data-lake-encrypted-[account-id]/`
4. **IAM role:** **"Create new role"** → `LakeFormationServiceRole`
5. Click **"Register location"**

**Lake Formation can now manage permissions for this S3 location.**

---

## PART 2: Create Glue Database and Table

### Step 4: Create Database
1. **Lake Formation** → **"Databases"** → **"Create database"**
2. **Name:** `sales_analytics`
3. **Description:** `Sales data lake for analytics`
4. **Location:** `s3://data-lake-encrypted-[id]/sales/`
5. Click **"Create database"**

### Step 5: Create Table with Sample Data
First, upload sample data to S3:

Create file: `customer_orders.csv`
```csv
order_id,customer_name,customer_ssn,email,department,product,amount,order_date
1001,John Doe,123-45-6789,john@example.com,Marketing,Laptop,1200.00,2024-06-01
1002,Jane Smith,987-65-4321,jane@example.com,Sales,Monitor,350.00,2024-06-02
1003,Bob Johnson,555-55-5555,bob@example.com,Engineering,Keyboard,89.99,2024-06-03
1004,Alice Williams,111-11-1111,alice@example.com,Marketing,Mouse,29.99,2024-06-04
1005,Charlie Brown,222-22-2222,charlie@example.com,Sales,Laptop,1200.00,2024-06-05
```

Upload to: `s3://data-lake-encrypted-[id]/sales/customer_orders/`

### Step 6: Create Glue Table
1. **Glue Console** → **"Tables"** → **"Add table"**
2. **Name:** `customer_orders`
3. **Database:** Select `sales_analytics`
4. **Data location:** `s3://data-lake-encrypted-[id]/sales/customer_orders/`
5. **Data format:** **"CSV"**
6. **Define schema:**
   - order_id: int
   - customer_name: string
   - customer_ssn: string
   - email: string
   - department: string
   - product: string
   - amount: decimal(10,2)
   - order_date: date
7. **Create table**

---

## PART 3: Implement Row-Level Security

### Step 7: Create Data Filter for Marketing Department
1. **Lake Formation** → **"Data filters"** → **"Create new filter"**
2. **Data filter name:** `marketing-only-filter`
3. **Target database:** `sales_analytics`
4. **Target table:** `customer_orders`
5. **Column-level access:** **"All columns"**
6. **Row filter expression:**
   ```sql
   department = 'Marketing'
   ```
7. Click **"Create filter"**

### Step 8: Create Data Filter for Sales Department
1. **"Create new filter"** again
2. **Data filter name:** `sales-only-filter`
3. **Target database:** `sales_analytics`
4. **Target table:** `customer_orders`
5. **Row filter expression:**
   ```sql
   department = 'Sales'
   ```
6. **Create filter**

### Step 9: Create IAM Roles for Analysts
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** **"AWS service"** → **"Glue"** (or Lambda if querying via Athena)
3. **Role name:** `MarketingAnalystRole`
4. **Create role**
5. Repeat for `SalesAnalystRole`

### Step 10: Grant Permissions with Data Filter
1. **Lake Formation** → **"Permissions"** → **"Grant"**
2. **Principals:** Select `MarketingAnalystRole`
3. **LF-Tags or catalog resources:** **"Named data catalog resources"**
4. **Databases:** Select `sales_analytics`
5. **Tables:** Select `customer_orders`
6. **Data filters:** Select `marketing-only-filter`
7. **Table permissions:** ✅ **"Select"**
8. Click **"Grant"**

Repeat for `SalesAnalystRole` with `sales-only-filter`.

### Step 11: Test Row-Level Security
Assume `MarketingAnalystRole` and query via Athena:
```sql
SELECT * FROM sales_analytics.customer_orders;
```

**Result:** Only sees rows where `department = 'Marketing'`!

---

## PART 4: Implement Column-Level Security

### Step 12: Create Data Filter Excluding Sensitive Columns
1. **Lake Formation** → **"Data filters"** → **"Create new filter"**
2. **Data filter name:** `no-ssn-filter`
3. **Target database:** `sales_analytics`
4. **Target table:** `customer_orders`
5. **Column-level access:** Select **"Include columns"**
6. **Select columns:** Choose all EXCEPT `customer_ssn`
   - order_id
   - customer_name
   - email
   - department
   - product
   - amount
   - order_date
7. **Row filter expression:** Leave empty (no row filtering)
8. **Create filter**

### Step 13: Grant Permissions to Junior Analysts
1. Create role: `JuniorAnalystRole`
2. **Lake Formation** → **"Permissions"** → **"Grant"**
3. **Principals:** `JuniorAnalystRole`
4. **Database/Table:** `sales_analytics.customer_orders`
5. **Data filters:** Select `no-ssn-filter`
6. **Permissions:** **"Select"**
7. **Grant**

### Step 14: Grant Full Access to Senior Analysts
1. Create role: `SeniorAnalystRole`
2. **Lake Formation** → **"Permissions"** → **"Grant"**
3. **Principals:** `SeniorAnalystRole`
4. **Database/Table:** `sales_analytics.customer_orders`
5. **Data filters:** **"Do not use data filter"** (full access)
6. **Permissions:** **"Select"**, **"Describe"**
7. **Grant**

### Step 15: Test Column-Level Security
**As JuniorAnalystRole:**
```sql
SELECT * FROM sales_analytics.customer_orders;
```
**Result:** `customer_ssn` column is NOT visible!

**As SeniorAnalystRole:**
```sql
SELECT * FROM sales_analytics.customer_orders;
```
**Result:** ALL columns visible, including `customer_ssn`!

---

## PART 5: Implement Tag-Based Access Control (TBAC)

### Step 16: Create LF-Tags
1. **Lake Formation** → **"LF-Tags"** → **"Create LF-Tag"**
2. **Key:** `Confidentiality`
3. **Values:** `Public`, `Internal`, `Confidential`, `Restricted`
4. **Create**

Repeat to create:
- **Key:** `Department` | **Values:** `Marketing`, `Sales`, `Engineering`, `Finance`
- **Key:** `Compliance` | **Values:** `PCI`, `HIPAA`, `GDPR`, `None`

### Step 17: Assign Tags to Resources
1. **Lake Formation** → **"Tables"** → Select `customer_orders`
2. **Actions** → **"Edit LF-Tags"**
3. **Assign LF-Tags:**
   - `Confidentiality`: `Confidential`
   - `Compliance`: `PCI` (has credit card data)
4. **Save**

### Step 18: Tag Individual Columns
1. Click on table: `customer_orders`
2. Go to **"Schema"** tab
3. Select column: `customer_ssn`
4. **"Edit LF-Tags"**
5. **Assign:**
   - `Confidentiality`: `Restricted`
6. **Save**

### Step 19: Grant Permissions via LF-Tags
1. **Lake Formation** → **"Permissions"** → **"Grant"**
2. **Principals:** `DataAnalystRole`
3. **LF-Tags or catalog resources:** Select **"Resources matched by LF-Tags"**
4. **LF-Tag key-value pairs:**
   - **Key:** `Confidentiality`
   - **Values:** `Public`, `Internal`, `Confidential`
   - (Excludes `Restricted`)
5. **Permissions:** **"Describe"**, **"Select"**
6. **Grant**

**Result:** Analysts can access tables/columns tagged as Public/Internal/Confidential, but NOT Restricted (SSN column)!

---

## PART 6: Audit Data Access

### Step 20: View Lake Formation Audit Logs in CloudTrail
1. **CloudTrail Console** → **"Event history"**
2. **Filter:**
   - **Event source:** `lakeformation.amazonaws.com`
3. You'll see events:
   - **GrantPermissions:** When permissions were granted
   - **GetDataAccess:** When data was accessed
   - **RevokePermissions:** When permissions were revoked

### Step 21: Create CloudWatch Alarm for Unauthorized Access
1. **CloudWatch Console** → **"Logs"** → **"Log groups"**
2. Find: `/aws/lakeformation/`
3. Click **"Create metric filter"**
4. **Filter pattern:**
   ```
   { ($.errorCode = "AccessDeniedException") }
   ```
5. **Metric name:** `UnauthorizedLakeFormationAccess`
6. **Create filter**

### Step 22: Create Alarm
1. **CloudWatch** → **"Alarms"** → **"Create alarm"**
2. **Metric:** Select `UnauthorizedLakeFormationAccess`
3. **Threshold:** **Greater than 5** (in 5 minutes)
4. **Notification:** SNS topic with email
5. **Create alarm**

**You'll get alerted if anyone tries unauthorized data access!**

---

## ✅ Exercise 9.2 Completion Checklist

- [ ] Set up AWS Lake Formation
- [ ] Added data lake administrators
- [ ] Registered S3 location as data lake
- [ ] Created Glue database in Lake Formation
- [ ] Created table with sample customer order data
- [ ] Created data filter for row-level security (Marketing dept)
- [ ] Created data filter for Sales department
- [ ] Created IAM roles for different analyst types
- [ ] Granted permissions with data filters
- [ ] Tested row-level security (departments see only their data)
- [ ] Created data filter excluding sensitive columns (SSN)
- [ ] Granted column-filtered permissions to junior analysts
- [ ] Granted full access to senior analysts
- [ ] Tested column-level security
- [ ] Created LF-Tags (Confidentiality, Department, Compliance)
- [ ] Assigned tags to tables and columns
- [ ] Granted permissions via tag-based policies
- [ ] Verified tag-based access control
- [ ] Viewed Lake Formation audit logs in CloudTrail
- [ ] Created CloudWatch alarm for unauthorized access

**🎉 Congratulations!** You've completed Exercise 9.2!

---

# Exercise 9.3: Threat Detection with GuardDuty and Macie

**Duration:** 1-2 hours | **Estimated Cost:** ~$5-10 for lab session

## 🎯 Learning Objectives
- Enable Amazon GuardDuty for threat detection
- Detect malicious activity automatically
- Enable Amazon Macie for sensitive data discovery
- Scan S3 buckets for PII (personally identifiable information)
- Automate remediation with Lambda
- Integrate with AWS Security Hub

---

## PART 1: Enable Amazon GuardDuty

### Step 1: Navigate to GuardDuty Console
1. AWS Console → Search **"GuardDuty"**
2. Click **"Amazon GuardDuty"**
3. Click **"Get started"**

### Step 2: Enable GuardDuty
1. **Choose account type:**
   - **"Standalone account"** (for single account)
   - OR **"Administrator account"** (for multi-account)
2. Click **"Enable GuardDuty"**
3. **GuardDuty is now active!** (no agent installation needed)

### Step 3: Configure Finding Settings
1. Go to **"Settings"** (left sidebar)
2. **Finding export frequency:** Select **"Update findings every 15 minutes"**
3. **S3 bucket for findings:** Optional - select bucket for exports
4. **Save**

### Step 4: Enable S3 Protection
1. Still in **"Settings"**
2. Scroll to **"S3 Protection"**
3. ✅ **"Enable"** (detects suspicious S3 API calls)
4. **Save**

### Step 5: View Sample Findings
1. Go to **"Findings"** (left sidebar)
2. Click **"Settings"** → **"Sample findings"**
3. Click **"Generate sample findings"**
4. You'll see sample threats:
   - **Recon:EC2/PortProbeUnprotectedPort** (port scanning)
   - **CryptoCurrency:EC2/BitcoinTool.B** (bitcoin mining)
   - **Trojan:EC2/DNSDataExfiltration** (data theft)
   - **UnauthorizedAccess:S3/MaliciousIPCaller** (S3 access from bad IP)

---

## PART 2: Configure GuardDuty Alerts

### Step 6: Create SNS Topic for Security Alerts
1. **SNS Console** → **"Create topic"**
2. **Name:** `security-alerts`
3. **Display name:** `Security Alerts`
4. **Create topic**
5. **Create subscription:**
   - **Protocol:** Email
   - **Endpoint:** Your email
   - **Create subscription** → Confirm via email

### Step 7: Create EventBridge Rule for GuardDuty
1. **EventBridge Console** → **"Rules"** → **"Create rule"**
2. **Name:** `guardduty-high-severity-alert`
3. **Event pattern:**
   ```json
   {
     "source": ["aws.guardduty"],
     "detail-type": ["GuardDuty Finding"],
     "detail": {
       "severity": [7, 7.5, 8, 8.5, 9, 9.5, 10]
     }
   }
   ```
4. **Target:** SNS topic: `security-alerts`
5. **Create rule**

**You'll now get emailed for high-severity threats!**

### Step 8: Create Lambda for Automated Response
1. **Lambda Console** → **"Create function"**
2. **Function name:** `guardduty-auto-response`
3. **Runtime:** Python 3.12
4. **Create function**
5. Code:

```python
import json
import boto3

iam = boto3.client('iam')
ec2 = boto3.client('ec2')
s3 = boto3.client('s3')

def lambda_handler(event, context):
    print(f"GuardDuty finding: {json.dumps(event)}")
    
    finding = event['detail']
    finding_type = finding['type']
    severity = finding['severity']
    
    print(f"Type: {finding_type}, Severity: {severity}")
    
    # Automated responses based on finding type
    if 'UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B' in finding_type:
        # Compromised credentials - deactivate access keys
        user_name = finding['resource']['accessKeyDetails']['userName']
        access_key = finding['resource']['accessKeyDetails']['accessKeyId']
        
        print(f"Deactivating compromised access key: {access_key}")
        iam.update_access_key(
            UserName=user_name,
            AccessKeyId=access_key,
            Status='Inactive'
        )
        
    elif 'UnauthorizedAccess:S3' in finding_type:
        # Suspicious S3 access - block public access
        bucket = finding['resource']['s3BucketDetails'][0]['name']
        
        print(f"Blocking public access on bucket: {bucket}")
        s3.put_public_access_block(
            Bucket=bucket,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
    
    return {
        'statusCode': 200,
        'body': json.dumps('Automated response executed')
    }
```

6. **Deploy**
7. Add as EventBridge target for GuardDuty findings

---

## PART 3: Enable Amazon Macie

### Step 9: Navigate to Macie Console
1. AWS Console → Search **"Macie"**
2. Click **"Amazon Macie"**
3. Click **"Get started"**

### Step 10: Enable Macie
1. **Service-linked role:** Auto-created
2. **Finding publishing frequency:** **"15 minutes"**
3. Click **"Enable Macie"**
4. **Macie is now active!**

### Step 11: View S3 Bucket Inventory
1. **Macie Console** → **"S3 buckets"** (left sidebar)
2. Macie automatically discovered all your S3 buckets!
3. You'll see columns:
   - **Bucket name**
   - **Public access:** Yes/No
   - **Encryption:** Enabled/Disabled
   - **Versioning:** Enabled/Disabled
   - **Objects:** Count
   - **Sensitive data:** Pending (until you scan)

---

## PART 4: Scan for Sensitive Data

### Step 12: Create Sample File with PII
Create file: `customer_data.csv`
```csv
first_name,last_name,ssn,credit_card,email,phone
John,Doe,123-45-6789,4532-1234-5678-9010,john.doe@example.com,555-123-4567
Jane,Smith,987-65-4321,5425-2345-6789-0123,jane.smith@example.com,555-234-5678
Bob,Johnson,555-55-5555,3782-456789-01234,bob.j@example.com,555-345-6789
```

Upload to: `s3://data-lake-encrypted-[id]/customer-pii/`

### Step 13: Create Sensitive Data Discovery Job
1. **Macie Console** → **"Jobs"** → **"Create job"**
2. **Job type:** **"One-time job"** (scan once)
3. **S3 buckets:** Select `data-lake-encrypted-[id]`
4. **Scope:** **"Include"** → Folder: `customer-pii/`
5. Click **"Next"**

### Step 14: Configure Managed Data Identifiers
1. **Managed data identifiers:** ✅ **"Use all"**
   - This detects: SSN, credit cards, passport numbers, driver licenses, bank accounts, medical records, etc.
2. Click **"Next"**

### Step 15: Create Custom Data Identifier (Optional)
1. **Custom data identifiers:** Click **"Create"**
2. **Name:** `employee-id-pattern`
3. **Regular expression:** `EMP-\d{5}` (e.g., EMP-12345)
4. **Keywords:** `employee`, `staff`
5. **Create**
6. Select it for this job
7. Click **"Next"**

### Step 16: Review and Create Job
1. **Job name:** `customer-pii-scan`
2. **Description:** `Scan for PII in customer data`
3. Review settings
4. Click **"Submit"**

### Step 17: Monitor Job Progress
1. **Jobs** → Click on `customer-pii-scan`
2. **Status:**
   - **Running** → 1-5 minutes
   - **Complete** → Scan finished
3. Wait for completion

### Step 18: View Findings
1. **Macie Console** → **"Findings"** (left sidebar)
2. You should see findings:
   - **Sensitive data type:** Social Security Number
   - **Severity:** High
   - **Bucket:** data-lake-encrypted-[id]
   - **Object:** customer-pii/customer_data.csv
   - **Occurrences:** 3 (3 SSNs found)

Click on a finding to see details:
- **Line numbers** where PII was found
- **Column names** containing PII
- **Data identifiers matched** (e.g., USA_SSN, CREDIT_CARD_NUMBER)

---

## PART 5: Automate PII Remediation

### Step 19: Create EventBridge Rule for Macie Findings
1. **EventBridge Console** → **"Rules"** → **"Create rule"**
2. **Name:** `macie-pii-remediation`
3. **Event pattern:**
   ```json
   {
     "source": ["aws.macie"],
     "detail-type": ["Macie Finding"],
     "detail": {
       "severity": {
         "description": ["High"]
       }
     }
   }
   ```
4. **Target:** Lambda function (create next)
5. **Create rule**

### Step 20: Create Remediation Lambda
1. **Lambda Console** → **"Create function"**
2. **Function name:** `macie-pii-remediation`
3. **Runtime:** Python 3.12
4. Code:

```python
import json
import boto3

s3 = boto3.client('s3')
sns = boto3.client('sns')

SNS_TOPIC_ARN = 'arn:aws:sns:us-east-1:123456789012:security-alerts'

def lambda_handler(event, context):
    finding = event['detail']
    
    bucket = finding['resourcesAffected']['s3Bucket']['name']
    object_key = finding['resourcesAffected']['s3Object']['key']
    severity = finding['severity']['description']
    
    print(f"PII detected: s3://{bucket}/{object_key}")
    print(f"Severity: {severity}")
    
    # Remediation actions
    # 1. Copy to quarantine bucket
    quarantine_bucket = f"{bucket}-quarantine"
    
    try:
        s3.copy_object(
            CopySource={'Bucket': bucket, 'Key': object_key},
            Bucket=quarantine_bucket,
            Key=f"pii-detected/{object_key}"
        )
        print(f"✓ Copied to quarantine: s3://{quarantine_bucket}/pii-detected/{object_key}")
    except:
        print(f"Quarantine bucket doesn't exist: {quarantine_bucket}")
    
    # 2. Tag original object
    s3.put_object_tagging(
        Bucket=bucket,
        Key=object_key,
        Tagging={
            'TagSet': [
                {'Key': 'PII-Detected', 'Value': 'True'},
                {'Key': 'Severity', 'Value': severity},
                {'Key': 'ReviewRequired', 'Value': 'True'}
            ]
        }
    )
    print(f"✓ Tagged object with PII-Detected flag")
    
    # 3. Send notification to data governance team
    message = f"""
    🚨 SENSITIVE DATA DETECTED
    
    Bucket: {bucket}
    Object: {object_key}
    Severity: {severity}
    
    Action Taken:
    - Object tagged for review
    - Copied to quarantine bucket
    
    Please review and take appropriate action.
    """
    
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject='🚨 PII Detected in Data Lake',
        Message=message
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps('Remediation completed')
    }
```

5. **Deploy**
6. Update SNS_TOPIC_ARN in code

### Step 21: Test Remediation
Upload another file with PII → Macie scans → Lambda triggers → Object tagged + copied to quarantine + notification sent!

---

## PART 6: Integrate with AWS Security Hub

### Step 22: Enable Security Hub
1. **Security Hub Console** → **"Get started"**
2. **Standards:**
   - ✅ **AWS Foundational Security Best Practices**
   - ✅ **CIS AWS Foundations Benchmark**
3. Click **"Enable Security Hub"**

### Step 23: View Aggregated Findings
1. **Security Hub** → **"Findings"**
2. You'll see consolidated findings from:
   - Amazon GuardDuty
   - Amazon Macie
   - AWS Config
   - IAM Access Analyzer
   - Amazon Inspector
3. **Filter by severity:** Critical, High, Medium, Low

### Step 24: Create Insight
1. **Security Hub** → **"Insights"** → **"Create insight"**
2. **Name:** `Top PII Violations`
3. **Group by:** Resource ID
4. **Filters:**
   - **Product name:** Macie
   - **Severity:** High
5. **Create insight**

Shows which S3 buckets/objects have most PII violations!

### Step 25: Export Findings to Athena
1. **Security Hub** → **"Settings"** → **"Integrations"**
2. Find **"Amazon Athena"**
3. Click **"Configure"**
4. Creates Athena table with all Security Hub findings
5. Query with SQL:

```sql
SELECT 
    ProductName,
    SeverityLabel,
    COUNT(*) as finding_count
FROM securityhub_findings
WHERE year = '2024' AND month = '06'
GROUP BY ProductName, SeverityLabel
ORDER BY finding_count DESC;
```

---

## ✅ Exercise 9.3 Completion Checklist

- [ ] Enabled Amazon GuardDuty
- [ ] Configured finding frequency (15 minutes)
- [ ] Enabled S3 protection
- [ ] Generated sample findings
- [ ] Created SNS topic for security alerts
- [ ] Created EventBridge rule for high-severity findings
- [ ] Created Lambda for automated incident response
- [ ] Enabled Amazon Macie
- [ ] Viewed S3 bucket inventory
- [ ] Created sample file with PII (SSN, credit cards)
- [ ] Created sensitive data discovery job
- [ ] Configured managed data identifiers
- [ ] Created custom data identifier (employee ID pattern)
- [ ] Ran scan and viewed PII findings
- [ ] Created EventBridge rule for Macie findings
- [ ] Created Lambda for PII remediation
- [ ] Tested automated quarantine and tagging
- [ ] Enabled AWS Security Hub
- [ ] Enabled security standards (Foundational + CIS)
- [ ] Viewed aggregated findings from all services
- [ ] Created custom insight for PII violations
- [ ] Exported findings to Athena for SQL analysis

**🎉 Congratulations!** You've completed Exercise 9.3 and all of Module 9!

---

# 🎓 Module 9 Complete Summary

## What You've Accomplished

### Exercise 9.1: Secure Pipeline ✅
- Created KMS customer-managed keys
- Stored secrets in Secrets Manager with rotation
- Implemented least-privilege IAM policies
- Encrypted data at rest with KMS
- Enabled CloudTrail for audit logging
- Monitored compliance with AWS Config

### Exercise 9.2: Lake Formation Governance ✅
- Set up centralized data lake governance
- Implemented row-level security (data filters)
- Configured column-level security (masked SSN)
- Used tag-based access control
- Granted fine-grained permissions
- Audited data access via CloudTrail

### Exercise 9.3: Threat Detection ✅
- Enabled GuardDuty for threat detection
- Automated incident response with Lambda
- Enabled Macie for PII discovery
- Scanned S3 buckets for sensitive data
- Automated PII remediation and quarantine
- Integrated findings with Security Hub

## Key Services Mastered
1. **AWS KMS** - Encryption key management
2. **AWS Secrets Manager** - Credential storage and rotation
3. **AWS Lake Formation** - Data lake governance
4. **Amazon GuardDuty** - Threat detection
5. **Amazon Macie** - Sensitive data discovery
6. **AWS Security Hub** - Security findings aggregation

## Next Steps
- Apply security best practices to all modules
- Continue with additional modules
- Prepare for certification exam

---

## 🧹 Cleanup Resources

**KMS:**
- Schedule key deletion (7-30 day waiting period)

**Secrets Manager:**
- Delete secrets

**Lake Formation:**
- Revoke all permissions
- Deregister S3 locations

**GuardDuty:**
- Disable GuardDuty ($0.0042/event after free tier)

**Macie:**
- Disable Macie ($0.10/GB scanned)

**Security Hub:**
- Disable Security Hub

**CloudTrail/Config:**
- Delete trails
- Stop Config recording

**S3:**
- Empty and delete all buckets

**Estimated Module 9 Lab Cost (3-4 hours): ~$15-30**

---

**🎉 Excellent work! You've mastered AWS Security for Data Engineering!**

**All 9 modules now complete! You're ready for the certification exam!** 🎓

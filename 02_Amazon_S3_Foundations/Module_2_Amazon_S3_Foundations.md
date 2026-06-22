# MODULE 2: Amazon S3 Foundations for Data Engineering

## Module Overview

**Duration:** 4-5 hours  
**Difficulty:** Beginner to Intermediate  
**Prerequisites:** 
- AWS Account with admin access
- Completed MODULE 1 (IAM for Data Engineers)
- Basic understanding of object storage concepts
- Familiarity with data lake architecture

**Estimated Cost:** $0.50 - $2.00 (can be reduced to ~$0 with cleanup)

**AWS Services Used:**
- Amazon S3 (Simple Storage Service)
- AWS KMS (Key Management Service)
- AWS CloudTrail
- Amazon CloudWatch

---

## What You Will Learn

### Why Amazon S3 is Critical for Data Engineers

Amazon S3 is the **foundation** of modern data lakes on AWS. Understanding S3 deeply is non-negotiable for data engineers because:

1. **It's the Central Data Repository:**
   - 99% of AWS data lakes use S3 as primary storage
   - Glue, Athena, EMR, Redshift Spectrum all read/write S3
   - Average data lake stores 100TB - 10PB in S3

2. **Cost Optimization:**
   - Lifecycle policies can reduce storage costs by 90%
   - Choosing wrong storage class = wasting $10,000s/month
   - Intelligent-Tiering automatically optimizes costs

3. **Production Use Cases:**
   - **Data Ingestion:** Landing zone for all incoming data (APIs, logs, databases)
   - **Data Lake Zones:** Raw → Staging → Curated → Analytics
   - **Big Data Analytics:** Petabyte-scale queries with Athena/Spark
   - **ML Training:** Store training datasets for SageMaker
   - **Compliance:** WORM (Write Once Read Many) with S3 Object Lock
   - **Disaster Recovery:** Cross-region replication for business continuity

4. **Where S3 Fits in Data Architecture:**

```
Data Sources (APIs, Databases, Logs)
         ↓
    S3 Landing Zone (Raw Data)
         ↓
    AWS Glue ETL Jobs
         ↓
    S3 Curated Zone (Parquet/ORC)
         ↓
    Amazon Athena / Redshift Spectrum / EMR
         ↓
    Business Intelligence Tools
```

### Common Beginner Mistakes (We'll Avoid These!)

❌ **Mistake 1:** Not enabling versioning → Accidental delete loses data forever  
✅ **Fix:** Enable versioning + lifecycle policy to archive old versions

❌ **Mistake 2:** Using S3 Standard for everything → Overpaying by 10x  
✅ **Fix:** Use lifecycle policies to move old data to cheaper storage classes

❌ **Mistake 3:** Public buckets → Data breach, compliance violations  
✅ **Fix:** Block all public access, use IAM roles + bucket policies

❌ **Mistake 4:** Not encrypting data → Fails compliance audits  
✅ **Fix:** Enable default encryption (SSE-S3 or SSE-KMS)

❌ **Mistake 5:** Poor folder structure → Slow Athena queries  
✅ **Fix:** Partition by date: `s3://bucket/year=2024/month=06/day=22/`

### Oracle DBA to S3 Mapping

| Oracle Concept | S3 Equivalent | Notes |
|----------------|---------------|-------|
| Tablespace | S3 Bucket | Logical storage container |
| Datafile | S3 Object | Individual file |
| Directory Object | S3 Prefix (folder) | Logical grouping, not real directories |
| RMAN Backup | S3 with Glacier | Long-term archival |
| Flashback | S3 Versioning | Point-in-time recovery |
| Data Guard | S3 Replication (CRR) | Disaster recovery across regions |
| ASM Redundancy | S3 Durability (11 9's) | Built-in, no configuration needed |
| Partitioning | S3 Prefixes with dates | Improves query performance |
| Compression (OLTP) | S3 object compression | Reduce storage costs |

---

## Exercise 2.1: Create S3 Data Lake with Lifecycle Policies

### Overview

Build a production-ready S3 data lake with three zones (Raw, Staging, Curated), implement lifecycle policies for cost optimization, and configure encryption and versioning.

**Real-World Scenario:**  
Your company ingests 1TB of JSON logs daily. Raw logs are queried frequently for 7 days, occasionally for 30 days, then rarely for compliance (7 years retention).

**What You'll Build:**

```
my-datalake-<account-id>
├── raw/                          [S3 Standard, 7 days]
│   └── logs/2024/06/22/          [Then → S3 Standard-IA]
├── staging/                      [S3 Standard, 30 days]
│   └── processed/2024/06/        [Then → S3 Glacier Instant Retrieval]
└── curated/                      [S3 Intelligent-Tiering]
    └── analytics/year=2024/
```

---

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    S3 Data Lake Architecture                │
└─────────────────────────────────────────────────────────────┘

   Data Sources (APIs, Logs, Databases)
         │
         ├──► S3 Bucket: my-datalake-123456789012
         │    ├─ raw/ (S3 Standard)
         │    │  └─ logs/year=2024/month=06/day=22/
         │    │     ├─ app.log (Encrypted: SSE-KMS)
         │    │     └─ api.log (Versioned)
         │    │
         │    ├─ staging/ (S3 Standard → S3-IA after 7 days)
         │    │  └─ processed/year=2024/month=06/
         │    │     └─ data.parquet
         │    │
         │    └─ curated/ (S3 Intelligent-Tiering)
         │       └─ analytics/year=2024/
         │          └─ aggregated.parquet
         │
         ├──► Lifecycle Policy: raw-data-lifecycle
         │    ├─ Transition to S3-IA after 7 days
         │    ├─ Transition to Glacier IR after 30 days
         │    └─ Delete after 2555 days (7 years)
         │
         ├──► Encryption: SSE-KMS
         │    └─ KMS Key: alias/datalake-key
         │
         ├──► Versioning: Enabled
         │    └─ Old versions → Glacier after 90 days
         │
         └──► Replication Rule: DR-to-us-west-2
              └─ Target: my-datalake-dr-123456789012

┌─────────────────────────────────────────────────────────────┐
│                    Access Control Layer                     │
└─────────────────────────────────────────────────────────────┘

IAM Role: DataEngineerRole         IAM Role: DataAnalystRole
   │                                     │
   ├─ s3:PutObject (raw/)               ├─ s3:GetObject (curated/)
   ├─ s3:GetObject (all)                └─ athena:StartQueryExecution
   ├─ s3:DeleteObject (staging/)
   └─ kms:Encrypt, kms:Decrypt
```

---

### Step-by-Step Lab Guide (AWS Console)

#### Part 1: Create S3 Bucket with Security Best Practices

**Step 1: Create the S3 Bucket**

1. **Sign in to AWS Console** → Search for **"S3"** → Click **"S3"**

2. Click **"Create bucket"** (orange button, top right)

3. **Bucket Configuration:**
   - **Bucket name:** `my-datalake-<your-account-id>`  
     *(Replace `<your-account-id>` with your 12-digit AWS account ID)*
     
     💡 **Why unique name?** S3 bucket names are globally unique across ALL AWS accounts
     
   - **AWS Region:** `us-east-1` (or your preferred region)
   
   - **Object Ownership:** 
     - Select **"ACLs disabled (recommended)"**
     - This enforces bucket policies only (no legacy ACLs)

4. **Block Public Access Settings:**
   - ✅ **Keep ALL checkboxes checked** (Block all public access)
   - This prevents accidental data leaks
   
   - Click **"I acknowledge that the current settings might result in this bucket and the objects within becoming public"** *(This won't happen because you blocked public access)*

5. **Bucket Versioning:**
   - Select **"Enable"**
   - ✅ **Why:** Protects against accidental deletes, ransomware

6. **Tags:**
   - Click **"Add tag"**
   - Key: `Environment` Value: `Development`
   - Key: `Project` Value: `DataLake`
   - Key: `Owner` Value: `YourName`

7. **Default Encryption:**
   - **Encryption type:** Select **"Server-side encryption with AWS Key Management Service keys (SSE-KMS)"**
   - **AWS KMS key:** Select **"Choose from your AWS KMS keys"**
   - Click **"Create new KMS key"** (opens new tab)

**Step 2: Create KMS Encryption Key**

*(In the new KMS tab that opened)*

1. Click **"Create key"**

2. **Key Configuration:**
   - **Key type:** Symmetric
   - **Key usage:** Encrypt and decrypt
   - Click **"Next"**

3. **Alias:** `alias/datalake-key`

4. **Description:** `Encryption key for S3 data lake`

5. Click **"Next"**

6. **Key Administrators:**
   - Select your IAM user
   - ✅ **Allow key administrators to delete this key**
   - Click **"Next"**

7. **Key Users:**
   - Select your IAM user
   - Select any IAM roles that will access S3 (e.g., `DataEngineerRole`)
   - Click **"Next"**

8. Review and click **"Finish"**

9. **Copy the Key ARN** (looks like `arn:aws:kms:us-east-1:123456789012:key/abc-123`)

10. **Return to S3 bucket creation tab**

11. Refresh the KMS key dropdown and select **`alias/datalake-key`**

12. **Bucket Key:** Enable (reduces KMS API calls by 99%, lowers cost)

13. Click **"Create bucket"**

**Screenshot Description:**  
*You should see a green banner: "Successfully created bucket 'my-datalake-123456789012'"*

---

#### Part 2: Create Folder Structure for Data Lake Zones

**Step 3: Create Folders (Prefixes)**

1. Click on your bucket name **`my-datalake-123456789012`**

2. Click **"Create folder"**

3. **Create `raw/` folder:**
   - **Folder name:** `raw`
   - **Encryption:** Server-side encryption with AWS KMS (inherited)
   - Click **"Create folder"**

4. **Repeat for:**
   - Folder name: `staging`
   - Folder name: `curated`

5. Click into **`raw/`** folder → Create nested folders:
   - Folder name: `logs`
   - Click into `logs/` → Create: `year=2024`
   - Click into `year=2024/` → Create: `month=06`
   - Click into `month=06/` → Create: `day=22`

6. Your structure should look like:
   ```
   my-datalake-123456789012/
   ├── raw/logs/year=2024/month=06/day=22/
   ├── staging/
   └── curated/
   ```

💡 **Why this structure?**  
- `year=2024/month=06/day=22/` enables **partition pruning** in Athena
- Queries like `WHERE year=2024 AND month=06` scan 1 day instead of entire dataset
- Reduces query cost by 30x - 100x

---

#### Part 3: Upload Sample Data

**Step 4: Upload Test Files**

1. **Create a sample JSON log file on your computer:**

   Create file: `app.log`
   ```json
   {"timestamp":"2024-06-22T10:30:00Z","level":"INFO","message":"User login successful","user_id":12345}
   {"timestamp":"2024-06-22T10:31:00Z","level":"ERROR","message":"Database connection timeout","service":"api"}
   {"timestamp":"2024-06-22T10:32:00Z","level":"INFO","message":"Order placed","order_id":"ORD-9876","amount":150.50}
   ```

2. Navigate to **`raw/logs/year=2024/month=06/day=22/`**

3. Click **"Upload"**

4. Click **"Add files"** → Select `app.log`

5. **Scroll down to Properties:**
   - **Storage class:** S3 Standard (default)
   - **Server-side encryption:** Automatically uses SSE-KMS with `datalake-key`

6. Click **"Upload"**

7. **Verify encryption:**
   - Click on `app.log` file
   - Under **Properties** tab → **Server-side encryption**
   - Should show: **AWS-KMS** with key `alias/datalake-key`

---

#### Part 4: Configure Lifecycle Policies for Cost Optimization

**Step 5: Create Lifecycle Rule for Raw Data**

1. Go back to bucket root (click **`my-datalake-123456789012`** at top)

2. Click **"Management"** tab

3. Click **"Create lifecycle rule"**

4. **Lifecycle Rule Configuration:**

   - **Rule name:** `raw-data-retention-policy`
   
   - **Rule scope:**
     - Select **"Limit the scope of this rule using one or more filters"**
     - **Prefix:** `raw/`
     - This rule applies ONLY to objects in `raw/` folder

5. **Lifecycle rule actions:**

   ✅ **Check these boxes:**
   - **Transition current versions of objects between storage classes**
   - **Transition previous versions of objects between storage classes**
   - **Permanently delete previous versions of objects**
   - **Delete expired object delete markers or incomplete multipart uploads**

6. **Transition current versions:**

   | Days after creation | Storage Class | Cost Savings |
   |---------------------|---------------|--------------|
   | 7 | S3 Standard-IA | 46% cheaper than Standard |
   | 30 | S3 Glacier Instant Retrieval | 68% cheaper than Standard |
   | 90 | S3 Glacier Flexible Retrieval | 82% cheaper than Standard |
   | 2555 (7 years) | **Permanently delete** | 100% savings |

   **Enter these transitions:**
   - Transition to **S3 Standard-IA** after **7** days
   - Transition to **S3 Glacier Instant Retrieval** after **30** days
   - Transition to **S3 Glacier Flexible Retrieval** after **90** days

7. **Transition previous versions (old file versions):**
   - Transition to **S3 Glacier Flexible Retrieval** after **90** days
   - Permanently delete after **365** days

8. **Delete expired delete markers:** Yes

9. **Delete incomplete multipart uploads:** After **7** days  
   *(Saves cost from abandoned multi-part uploads)*

10. Click **"Create rule"**

**Step 6: Create Lifecycle Rule for Staging Data**

1. Click **"Create lifecycle rule"** again

2. **Rule name:** `staging-data-retention-policy`

3. **Prefix:** `staging/`

4. **Transitions:**
   - S3 Standard-IA after **30** days
   - S3 Glacier Instant Retrieval after **90** days
   - Permanently delete current versions after **180** days
   - Permanently delete previous versions after **90** days

5. Click **"Create rule"**

**Step 7: Use S3 Intelligent-Tiering for Curated Data**

1. Navigate to **`curated/`** folder

2. Upload a file (any small file for testing)

3. Click on the uploaded file

4. Click **"Actions"** → **"Edit storage class"**

5. Select **"Intelligent-Tiering"**

6. Click **"Save changes"**

💡 **Intelligent-Tiering:** Automatically moves objects between access tiers based on usage patterns (no retrieval fees, small monitoring fee)

---

#### Part 5: Enable S3 Versioning and Test Recovery

**Step 8: Test Version Recovery**

1. Navigate to `raw/logs/year=2024/month=06/day=22/`

2. Click on **`app.log`**

3. **Download the original file** (for comparison later)

4. Click **"Upload"** → Upload the **same** file `app.log` again, but modify content:
   ```json
   {"timestamp":"2024-06-22T11:00:00Z","level":"WARN","message":"Modified for versioning test"}
   ```

5. **View versions:**
   - Click on `app.log`
   - Toggle **"Show versions"** (top right)
   - You should see **2 versions** with different Version IDs

6. **Restore old version:**
   - Click on the older version
   - Click **"Download"** to verify it's the original content
   - Click **"Actions"** → **"Copy"**
   - Paste as destination: `s3://my-datalake-<account-id>/raw/logs/year=2024/month=06/day=22/app-restored.log`
   - Click **"Copy"**

7. **Delete protection test:**
   - Click **"Delete"** on current version of `app.log`
   - S3 adds a **delete marker** (doesn't actually delete)
   - Toggle **"Show versions"** → You can still see all versions
   - Select delete marker → Click **"Delete"** → File is restored!

---

#### Part 6: Configure S3 Bucket Policy for Secure Access

**Step 9: Create Bucket Policy**

1. Go to bucket root → Click **"Permissions"** tab

2. Scroll to **"Bucket policy"** → Click **"Edit"**

3. **Paste this policy** (replace `<YOUR-ACCOUNT-ID>` and `<ANALYST-ROLE-ARN>`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-datalake-<YOUR-ACCOUNT-ID>",
        "arn:aws:s3:::my-datalake-<YOUR-ACCOUNT-ID>/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "AllowDataAnalystReadCurated",
      "Effect": "Allow",
      "Principal": {
        "AWS": "<ANALYST-ROLE-ARN>"
      },
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-datalake-<YOUR-ACCOUNT-ID>/curated/*",
        "arn:aws:s3:::my-datalake-<YOUR-ACCOUNT-ID>"
      ]
    },
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-datalake-<YOUR-ACCOUNT-ID>/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

**What this policy does:**
- ✅ **Denies all HTTP traffic** (requires HTTPS)
- ✅ **Allows Data Analyst role** to read `curated/` folder only
- ✅ **Enforces encryption** on all uploads (must use KMS)

4. Click **"Save changes"**

---

#### Part 7: Configure S3 Cross-Region Replication (Disaster Recovery)

**Step 10: Create Destination Bucket**

1. **Create second bucket in different region:**
   - **Name:** `my-datalake-dr-<your-account-id>`
   - **Region:** `us-west-2` (different from source)
   - **Versioning:** **Enable** (required for replication)
   - **Encryption:** SSE-KMS with same key settings
   - **Block Public Access:** Enable all

2. Click **"Create bucket"**

**Step 11: Configure Replication Rule**

1. Go back to **source bucket** (`my-datalake-<account-id>`)

2. Click **"Management"** tab → **"Replication rules"**

3. Click **"Create replication rule"**

4. **Replication Rule Configuration:**
   - **Rule name:** `disaster-recovery-replication`
   - **Status:** Enabled
   
5. **Source bucket:**
   - **Rule scope:** Entire bucket (replicate everything)
   - Or choose **Prefix:** `curated/` (replicate only production data)

6. **Destination:**
   - **Bucket:** `my-datalake-dr-<account-id>` (in us-west-2)
   - ✅ **Change object ownership to destination bucket owner**

7. **IAM role:**
   - Select **"Create new role"**
   - AWS automatically creates role with permissions

8. **Encryption:**
   - ✅ **Replicate objects encrypted with AWS KMS**
   - Select the KMS key: `alias/datalake-key`

9. **Destination storage class:**
   - ✅ **Change storage class for replicated objects**
   - Select **S3 Standard-IA** (cheaper for DR, not accessed frequently)

10. **Additional replication options:**
    - ✅ **Replication Time Control (RTC):** Enable  
      → Guarantees 99.99% of objects replicated within 15 minutes
    - ✅ **Replication metrics and notifications**
    - ✅ **Delete marker replication:** Enable
    - ✅ **Replica modification sync:** Enable

11. Click **"Save"**

12. **Choose replication for existing objects:**
    - Click **"Yes, replicate existing objects"**
    - This creates a one-time **S3 Batch Replication** job
    - Click **"Submit"**

**Verify Replication:**
1. Upload a test file to source bucket
2. Wait 1-15 minutes (depending on RTC setting)
3. Check destination bucket → file should appear
4. **Test DR scenario:**
   - Delete file from source bucket (adds delete marker)
   - Check destination → delete marker is replicated
   - Restore from destination if needed

---

### CLI Version (Automation-Ready)

```bash
#!/bin/bash
# S3 Data Lake Setup Script
# Prerequisites: AWS CLI configured with credentials

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
SOURCE_BUCKET="my-datalake-${ACCOUNT_ID}"
DR_BUCKET="my-datalake-dr-${ACCOUNT_ID}"
SOURCE_REGION="us-east-1"
DR_REGION="us-west-2"

echo "🚀 Creating S3 Data Lake for Account: ${ACCOUNT_ID}"

# Step 1: Create KMS Key
echo "📝 Creating KMS encryption key..."
KMS_KEY_ID=$(aws kms create-key \
  --description "Data Lake Encryption Key" \
  --region ${SOURCE_REGION} \
  --query 'KeyMetadata.KeyId' \
  --output text)

aws kms create-alias \
  --alias-name alias/datalake-key \
  --target-key-id ${KMS_KEY_ID} \
  --region ${SOURCE_REGION}

echo "✅ KMS Key Created: ${KMS_KEY_ID}"

# Step 2: Create Source Bucket
echo "📦 Creating source S3 bucket..."
aws s3api create-bucket \
  --bucket ${SOURCE_BUCKET} \
  --region ${SOURCE_REGION} \
  --create-bucket-configuration LocationConstraint=${SOURCE_REGION}

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket ${SOURCE_BUCKET} \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket ${SOURCE_BUCKET} \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/datalake-key"
      },
      "BucketKeyEnabled": true
    }]
  }'

# Block public access
aws s3api put-public-access-block \
  --bucket ${SOURCE_BUCKET} \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Add tags
aws s3api put-bucket-tagging \
  --bucket ${SOURCE_BUCKET} \
  --tagging 'TagSet=[
    {Key=Environment,Value=Development},
    {Key=Project,Value=DataLake},
    {Key=CostCenter,Value=Engineering}
  ]'

echo "✅ Source bucket created: ${SOURCE_BUCKET}"

# Step 3: Create folder structure
echo "📁 Creating data lake folder structure..."
aws s3api put-object --bucket ${SOURCE_BUCKET} --key raw/
aws s3api put-object --bucket ${SOURCE_BUCKET} --key staging/
aws s3api put-object --bucket ${SOURCE_BUCKET} --key curated/
aws s3api put-object --bucket ${SOURCE_BUCKET} --key raw/logs/year=2024/month=06/day=22/

# Step 4: Upload sample data
echo "📤 Uploading sample data..."
cat > /tmp/app.log << 'EOF'
{"timestamp":"2024-06-22T10:30:00Z","level":"INFO","message":"User login","user_id":12345}
{"timestamp":"2024-06-22T10:31:00Z","level":"ERROR","message":"DB timeout","service":"api"}
{"timestamp":"2024-06-22T10:32:00Z","level":"INFO","message":"Order placed","order_id":"ORD-9876"}
EOF

aws s3 cp /tmp/app.log s3://${SOURCE_BUCKET}/raw/logs/year=2024/month=06/day=22/app.log

# Step 5: Create lifecycle policy
echo "⏳ Configuring lifecycle policies..."
cat > /tmp/lifecycle-raw.json << 'EOF'
{
  "Rules": [
    {
      "Id": "raw-data-retention-policy",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "raw/"
      },
      "Transitions": [
        {
          "Days": 7,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 30,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 365
      },
      "Expiration": {
        "Days": 2555
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket ${SOURCE_BUCKET} \
  --lifecycle-configuration file:///tmp/lifecycle-raw.json

# Step 6: Apply bucket policy
echo "🔒 Applying bucket policy..."
cat > /tmp/bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::${SOURCE_BUCKET}",
        "arn:aws:s3:::${SOURCE_BUCKET}/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${SOURCE_BUCKET}/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket ${SOURCE_BUCKET} \
  --policy file:///tmp/bucket-policy.json

# Step 7: Create DR bucket
echo "🌎 Creating disaster recovery bucket in ${DR_REGION}..."
aws s3api create-bucket \
  --bucket ${DR_BUCKET} \
  --region ${DR_REGION} \
  --create-bucket-configuration LocationConstraint=${DR_REGION}

aws s3api put-bucket-versioning \
  --bucket ${DR_BUCKET} \
  --versioning-configuration Status=Enabled \
  --region ${DR_REGION}

# Step 8: Create replication role
echo "🔄 Setting up cross-region replication..."
REPLICATION_ROLE_NAME="s3-replication-role"

# Create trust policy
cat > /tmp/trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "s3.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ${REPLICATION_ROLE_NAME} \
  --assume-role-policy-document file:///tmp/trust-policy.json

# Create permissions policy
cat > /tmp/replication-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::${SOURCE_BUCKET}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl"
      ],
      "Resource": "arn:aws:s3:::${SOURCE_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete"
      ],
      "Resource": "arn:aws:s3:::${DR_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:${SOURCE_REGION}:${ACCOUNT_ID}:key/${KMS_KEY_ID}"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ${REPLICATION_ROLE_NAME} \
  --policy-name S3ReplicationPolicy \
  --policy-document file:///tmp/replication-policy.json

REPLICATION_ROLE_ARN=$(aws iam get-role --role-name ${REPLICATION_ROLE_NAME} --query 'Role.Arn' --output text)

# Configure replication
cat > /tmp/replication-config.json << EOF
{
  "Role": "${REPLICATION_ROLE_ARN}",
  "Rules": [{
    "Status": "Enabled",
    "Priority": 1,
    "DeleteMarkerReplication": {
      "Status": "Enabled"
    },
    "Filter": {
      "Prefix": "curated/"
    },
    "Destination": {
      "Bucket": "arn:aws:s3:::${DR_BUCKET}",
      "StorageClass": "STANDARD_IA",
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
      }
    }
  }]
}
EOF

aws s3api put-bucket-replication \
  --bucket ${SOURCE_BUCKET} \
  --replication-configuration file:///tmp/replication-config.json

echo "✅ Replication configured: ${SOURCE_BUCKET} → ${DR_BUCKET}"

# Cleanup temp files
rm /tmp/*.json /tmp/app.log

echo ""
echo "🎉 S3 Data Lake Setup Complete!"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Source Bucket: s3://${SOURCE_BUCKET}"
echo "DR Bucket:     s3://${DR_BUCKET}"
echo "KMS Key:       ${KMS_KEY_ID}"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Next Steps:"
echo "1. Verify bucket contents: aws s3 ls s3://${SOURCE_BUCKET} --recursive"
echo "2. Check lifecycle rules: aws s3api get-bucket-lifecycle-configuration --bucket ${SOURCE_BUCKET}"
echo "3. Monitor replication: aws s3api get-bucket-replication --bucket ${SOURCE_BUCKET}"
```

**Save this script as:** `setup-s3-datalake.sh`

**Run it:**
```bash
chmod +x setup-s3-datalake.sh
./setup-s3-datalake.sh
```

---

### Validation Steps

**✅ Test 1: Verify Bucket Creation**
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3 ls | grep my-datalake-${ACCOUNT_ID}
```
**Expected Output:** Both source and DR buckets listed

---

**✅ Test 2: Verify Encryption**
```bash
aws s3api head-object \
  --bucket my-datalake-${ACCOUNT_ID} \
  --key raw/logs/year=2024/month=06/day=22/app.log

# Look for: "ServerSideEncryption": "aws:kms"
```

---

**✅ Test 3: Verify Versioning**
```bash
# Upload same file twice
echo "Version 1" > test.txt
aws s3 cp test.txt s3://my-datalake-${ACCOUNT_ID}/raw/test.txt

echo "Version 2" > test.txt
aws s3 cp test.txt s3://my-datalake-${ACCOUNT_ID}/raw/test.txt

# List versions
aws s3api list-object-versions \
  --bucket my-datalake-${ACCOUNT_ID} \
  --prefix raw/test.txt

# Should show 2 versions
```

---

**✅ Test 4: Verify Lifecycle Policy**
```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket my-datalake-${ACCOUNT_ID} \
  --query 'Rules[?Id==`raw-data-retention-policy`]'

# Verify transitions at 7, 30, 90 days
```

---

**✅ Test 5: Test Bucket Policy (Deny HTTP)**
```bash
# This should FAIL (AccessDenied) because bucket policy denies non-HTTPS
aws s3 cp test.txt s3://my-datalake-${ACCOUNT_ID}/test.txt \
  --no-verify-ssl  # Force HTTP

# This should SUCCEED (HTTPS)
aws s3 cp test.txt s3://my-datalake-${ACCOUNT_ID}/test.txt
```

---

**✅ Test 6: Verify Replication**
```bash
# Upload file to source curated/ folder
echo "Test replication" > replicate-me.txt
aws s3 cp replicate-me.txt s3://my-datalake-${ACCOUNT_ID}/curated/

# Wait 2-15 minutes, then check DR bucket
aws s3 ls s3://my-datalake-dr-${ACCOUNT_ID}/curated/ --region us-west-2

# Should show replicate-me.txt
```

---

**✅ Test 7: Storage Class Verification**
```bash
# Check storage class
aws s3api head-object \
  --bucket my-datalake-${ACCOUNT_ID} \
  --key curated/replicate-me.txt \
  --query 'StorageClass'

# Upload to curated/, change to Intelligent-Tiering
aws s3 cp test.txt s3://my-datalake-${ACCOUNT_ID}/curated/test.txt \
  --storage-class INTELLIGENT_TIERING

# Verify
aws s3api head-object \
  --bucket my-datalake-${ACCOUNT_ID} \
  --key curated/test.txt \
  --query 'StorageClass'
# Output: "INTELLIGENT_TIERING"
```

---

### Production Cost Estimator

**Scenario:** 1TB data ingested daily to `raw/`, retained for 7 years

| Storage Tier | Days | Size (TB) | Storage Class | Monthly Cost | Annual Cost |
|--------------|------|-----------|---------------|--------------|-------------|
| Hot | 0-7 | 1 × 7 = 7TB | S3 Standard | 7TB × $0.023/GB = **$161** | $1,932 |
| Warm | 8-30 | 1 × 23 = 23TB | S3 Standard-IA | 23TB × $0.0125/GB = **$288** | $3,456 |
| Cool | 31-90 | 1 × 60 = 60TB | Glacier IR | 60TB × $0.004/GB = **$240** | $2,880 |
| Cold | 91-2555 | 1 × 2465 = 2465TB | Glacier Flexible | 2465TB × $0.0036/GB = **$8,874** | $106,488 |
| **TOTAL** | | **2555TB** | Mixed | **$9,563/mo** | **$114,756/yr** |

**Without Lifecycle Policies (S3 Standard only):**
- 2555TB × $0.023/GB = **$58,765/month** = **$705,180/year**

**Savings with Lifecycle Policies:**
- **$705,180 - $114,756 = $590,424/year (84% cost reduction!)**

---

## Interview Preparation

### Beginner Questions (5)

**Q1: What is Amazon S3 and why is it used for data lakes?**

**Answer:**  
Amazon S3 (Simple Storage Service) is an object storage service designed for 99.999999999% (11 9's) durability. It's used for data lakes because:
1. **Unlimited scalability** - Store petabytes of data
2. **Cost-effective** - Pay only for what you store, as low as $1/TB/month (Glacier)
3. **Durability** - Data stored across multiple AZs, virtually never loses data
4. **Integration** - Native integration with Glue, Athena, EMR, Redshift Spectrum
5. **Flexibility** - Stores any file format (JSON, Parquet, CSV, images, logs)

**Production Example:**  
Netflix stores 100+ petabytes of media files in S3 for global streaming.

---

**Q2: Explain S3 bucket naming rules.**

**Answer:**  
- **Globally unique** across ALL AWS accounts worldwide
- **3-63 characters**
- **Lowercase letters, numbers, hyphens only** (no uppercase, underscores, spaces)
- **Must start with letter or number** (not hyphen)
- **Cannot end with hyphen**
- **Cannot be formatted as IP address** (e.g., 192.168.1.1)
- **Cannot start with `xn--`** (reserved prefix)
- **Cannot end with `-s3alias`** (reserved suffix)

**Why this matters:**  
- Bucket names become part of DNS: `my-bucket.s3.amazonaws.com`
- Once created, bucket name is locked to your account until deleted
- Many companies use pattern: `<company>-<environment>-<purpose>-<region>`  
  Example: `acme-prod-datalake-us-east-1`

---

**Q3: What's the difference between S3 Standard and S3 Standard-IA?**

**Answer:**

| Feature | S3 Standard | S3 Standard-IA |
|---------|-------------|----------------|
| **Use Case** | Frequently accessed (daily) | Infrequent access (monthly) |
| **Storage Cost** | $0.023/GB/month | $0.0125/GB/month (46% cheaper) |
| **Retrieval Cost** | Free | $0.01/GB |
| **Min Storage Duration** | None | 30 days |
| **Min Object Size** | None | 128KB |
| **Availability** | 99.99% | 99.9% |

**When to use Standard-IA:**
- Backups accessed monthly
- Logs older than 7-30 days
- Data queried occasionally, not daily

**Production Example:**  
Financial company stores last 7 days of transaction logs in Standard (queried hourly), older logs in Standard-IA (queried for audits monthly).

---

**Q4: How does S3 versioning protect data?**

**Answer:**  
S3 Versioning keeps **all versions** of an object (including deletes).

**How it works:**
1. Enable versioning at bucket level (cannot enable per-object)
2. Every overwrite creates a new version with unique Version ID
3. Deleting a file adds a **delete marker** (doesn't actually delete)
4. Can restore any previous version

**Example:**
```
1. Upload file.txt (Version ID: abc123)
2. Upload file.txt again (Version ID: def456) → abc123 still exists
3. Delete file.txt → Adds delete marker → Both versions still retrievable
4. Permanently delete → Specify version ID to actually delete
```

**Protection scenarios:**
- ✅ Accidental delete → Remove delete marker, file restored
- ✅ Ransomware overwrites files → Restore previous versions
- ✅ Application bug corrupts data → Roll back to known-good version

**Cost consideration:**  
- **All versions are billed**, not just current version
- Use lifecycle policy to delete old versions after N days
- Example: Keep current version forever, delete previous versions after 90 days

---

**Q5: What is S3 Lifecycle Policy and why use it?**

**Answer:**  
Lifecycle policy **automatically transitions objects between storage classes** or **deletes them** after specified time periods.

**Use cases:**
1. **Cost optimization** - Move old data to cheaper storage
2. **Compliance** - Delete data after retention period (GDPR: delete after 2 years)
3. **Automation** - No manual intervention needed

**Example Policy:**
```
Day 0-7:   S3 Standard          ($0.023/GB/month)
Day 8-30:  S3 Standard-IA       ($0.0125/GB/month)
Day 31-90: Glacier IR           ($0.004/GB/month)
Day 91+:   Glacier Flexible     ($0.0036/GB/month)
Day 2555:  Delete (7 years)
```

**Real-world ROI:**
- Company storing 100TB of logs daily
- Without lifecycle: $2,300/day × 2555 days = **$5.8M**
- With lifecycle: Average $200/day × 2555 days = **$511K**
- **Savings: $5.3M over 7 years (91% reduction)**

---

### Intermediate Questions (5)

**Q6: Explain S3 encryption options: SSE-S3, SSE-KMS, SSE-C, Client-Side.**

**Answer:**

| Encryption Type | Key Management | Use Case | Cost |
|-----------------|----------------|----------|------|
| **SSE-S3** | AWS manages keys (AES-256) | Default, low-maintenance | Free |
| **SSE-KMS** | AWS KMS (you control key policies) | Audit trail, key rotation, BYOK | $0.03/10K requests |
| **SSE-C** | Customer provides key with each request | Full key control, no AWS key storage | Free |
| **Client-Side** | Encrypt before upload (e.g., AWS Encryption SDK) | Zero-trust, AWS never sees plaintext | Free |

**Production recommendations:**
- **SSE-KMS:** Best for most data lakes (audit requirements, compliance)
- **SSE-S3:** Non-sensitive data, high-throughput workloads (no KMS API limits)
- **Client-Side:** Highly sensitive (PII, PHI) - encrypt before upload

**Important for data engineers:**
- **Athena, Glue, EMR** support all types, but SSE-KMS requires IAM permissions for `kms:Decrypt`
- **Cross-account access** with SSE-KMS needs KMS key policy allowing other accounts

**Example KMS key policy for cross-account:**
```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111111111111:root"
  },
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

---

**Q7: What is S3 Intelligent-Tiering and when should you use it?**

**Answer:**  
Intelligent-Tiering **automatically moves objects between access tiers** based on access patterns (no manual lifecycle rules needed).

**How it works:**
1. **Frequent Access tier:** Objects accessed in last 30 days (same cost as S3 Standard)
2. **Infrequent Access tier:** Not accessed for 30 days (same cost as Standard-IA)
3. **Archive Instant Access tier:** Not accessed for 90 days (same cost as Glacier IR)
4. **Optional Archive Access tier:** Not accessed for 90-730 days (Glacier Flexible pricing)
5. **Optional Deep Archive tier:** Not accessed for 180-730 days (Glacier Deep Archive pricing)

**Pricing:**
- **Monitoring fee:** $0.0025 per 1,000 objects (small fee)
- **No retrieval fees** (unlike Standard-IA, Glacier)
- **No lifecycle transition fees**

**When to use:**
- ✅ **Unpredictable access patterns** (can't define lifecycle rules)
- ✅ **Mixed workloads** (some files accessed frequently, others rarely)
- ✅ **New data lakes** (don't know access patterns yet)

**When NOT to use:**
- ❌ **Predictable access** (use lifecycle policies instead - no monitoring fee)
- ❌ **Small objects < 128KB** (monitoring fee exceeds savings)
- ❌ **Short-lived data < 30 days** (billed for 30-day minimum)

**Production example:**  
Company stores 1 million user uploads (unknown which will be popular). Intelligent-Tiering automatically moves cold files to cheap tiers, hot files stay in frequent tier. **Saves 70% vs. S3 Standard with no manual management.**

---

**Q8: Explain S3 Cross-Region Replication (CRR) vs. Same-Region Replication (SRR).**

**Answer:**

| Feature | CRR | SRR |
|---------|-----|-----|
| **Purpose** | Disaster recovery, compliance | Log aggregation, dev/prod sync |
| **Latency** | Lower latency for global users | Faster replication (same region) |
| **Cost** | Data transfer fees ($0.02/GB) | No data transfer fees |
| **Use Case** | Prod → DR in different region | Prod → Test in same region |

**Requirements for BOTH:**
1. ✅ **Versioning enabled** on source AND destination buckets
2. ✅ **Proper IAM permissions** (S3 replication role)
3. ✅ **Different buckets** (can't replicate to same bucket)

**What gets replicated:**
- ✅ New objects (after replication enabled)
- ✅ Object metadata and tags
- ✅ Object ACLs
- ✅ S3 Object Lock information
- ✅ Delete markers (if configured)

**What does NOT get replicated:**
- ❌ Objects created BEFORE replication was enabled (use S3 Batch Replication)
- ❌ Objects in Glacier/Deep Archive (must restore first)
- ❌ Server-side encrypted objects with SSE-C (customer-managed keys)
- ❌ Objects owned by different AWS account (unless configured)

**Production DR scenario:**
- **Primary:** `s3://prod-datalake` (us-east-1)
- **DR:** `s3://prod-datalake-dr` (us-west-2)
- **RTC enabled:** 99.99% of objects replicated within 15 minutes
- **Disaster strikes us-east-1:** Switch applications to read from `us-west-2`

**Monitoring replication:**
```bash
aws s3api get-bucket-replication --bucket prod-datalake
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name ReplicationLatency \
  --dimensions Name=SourceBucket,Value=prod-datalake \
  --start-time 2024-06-22T00:00:00Z \
  --end-time 2024-06-22T23:59:59Z \
  --period 3600 \
  --statistics Average
```

---

**Q9: How do you optimize S3 performance for high-throughput data ingestion?**

**Answer:**

**S3 Performance Limits (2024):**
- **3,500 PUT/COPY/POST/DELETE requests per second** per prefix
- **5,500 GET/HEAD requests per second** per prefix

**Key optimization: Use multiple prefixes**

**❌ Poor design (single prefix):**
```
s3://bucket/logs/file1.json
s3://bucket/logs/file2.json
s3://bucket/logs/file3.json
...
(Limited to 3,500 writes/sec)
```

**✅ Optimized design (sharded prefixes):**
```
s3://bucket/logs/shard-0/file1.json
s3://bucket/logs/shard-1/file2.json
s3://bucket/logs/shard-2/file3.json
...
s3://bucket/logs/shard-99/file100.json

(100 prefixes × 3,500 writes/sec = 350,000 writes/sec)
```

**Sharding strategies:**

1. **Hash-based sharding** (distribute evenly):
   ```python
   import hashlib
   
   def get_s3_key(filename):
       hash_val = hashlib.md5(filename.encode()).hexdigest()
       shard = hash_val[:2]  # First 2 hex chars (00-FF = 256 shards)
       return f"logs/shard-{shard}/{filename}"
   
   # Example: file123.json → logs/shard-a4/file123.json
   ```

2. **Time-based sharding** (for streaming data):
   ```python
   from datetime import datetime
   
   def get_s3_key(filename):
       now = datetime.utcnow()
       # Year/Month/Day/Hour/Minute provides natural sharding
       return f"logs/year={now.year}/month={now.month:02d}/day={now.day:02d}/hour={now.hour:02d}/{filename}"
   
   # Example: logs/year=2024/month=06/day=22/hour=14/file.json
   # Benefit: Auto-sharding + Athena partition pruning
   ```

3. **Random prefix (legacy, rarely needed now):**
   ```python
   import random
   
   def get_s3_key(filename):
       shard = random.randint(0, 999)
       return f"logs/shard-{shard:04d}/{filename}"
   ```

**Additional optimizations:**

✅ **Use S3 Transfer Acceleration** (up to 50% faster uploads from distant locations)
```bash
aws configure set default.s3.use_accelerate_endpoint true
aws s3 cp large-file.zip s3://bucket/ --endpoint-url https://bucket.s3-accelerate.amazonaws.com
```

✅ **Use multipart upload for files > 100MB**
```bash
aws s3 cp large-file.dat s3://bucket/ \
  --storage-class INTELLIGENT_TIERING
# AWS CLI automatically uses multipart for files > 8MB
```

✅ **Use parallel uploads (AWS CLI default)**
```bash
aws configure set default.s3.max_concurrent_requests 20
# Default is 10, increase for high-bandwidth networks
```

✅ **Enable S3 Transfer Acceleration for cross-region uploads**

**Production example:**
- Company ingests 100,000 IoT device logs/sec
- Uses time-based sharding: `device-logs/year=YYYY/month=MM/day=DD/hour=HH/device_id=XXXXXX/`
- Achieves **500,000 writes/sec** across prefixes
- Athena queries scan only relevant partitions (query cost reduced 100x)

---

**Q10: What are S3 Object Lock and Glacier Vault Lock? When to use each?**

**Answer:**

Both implement **WORM (Write Once Read Many)** but for different use cases:

### S3 Object Lock

**Purpose:** Prevent object deletion/overwrite for fixed period or indefinitely

**Modes:**
1. **Governance Mode:**
   - Users with special permissions can override lock
   - Use case: Protect against accidental deletes, but admins can still delete if needed
   - Can be overridden with `s3:BypassGovernanceRetention` permission

2. **Compliance Mode:**
   - **NOBODY** can delete/overwrite (not even root account!)
   - Use case: SEC, HIPAA, FINRA compliance (regulatory requirements)
   - Cannot be shortened or removed until retention period expires

**Retention Modes:**
- **Retention Period:** Lock for X days (e.g., "retain for 7 years")
- **Legal Hold:** Indefinite lock (removed manually when legal case closes)

**Setup:**
```bash
# Must enable at bucket creation (cannot enable on existing bucket)
aws s3api create-bucket \
  --bucket compliance-bucket \
  --object-lock-enabled-for-bucket

# Set default retention
aws s3api put-object-lock-configuration \
  --bucket compliance-bucket \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 2555
      }
    }
  }'

# Apply legal hold to specific object
aws s3api put-object-legal-hold \
  --bucket compliance-bucket \
  --key important-doc.pdf \
  --legal-hold Status=ON
```

**Use cases:**
- ✅ **Financial records** (SOX requires 7 years retention)
- ✅ **Healthcare records** (HIPAA requires immutable audit logs)
- ✅ **Legal discovery** (litigation hold - prevent deletion during case)

---

### Glacier Vault Lock

**Purpose:** Lock entire Glacier vault with compliance policy (vault-level, not object-level)

**How it works:**
1. Create vault lock policy (JSON document)
2. Initiate lock (24-hour window to test)
3. Complete lock (PERMANENT - cannot undo!)

**Example policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Principal": "*",
    "Action": "glacier:DeleteArchive",
    "Resource": "arn:aws:glacier:us-east-1:123456789012:vaults/ComplianceVault",
    "Condition": {
      "DateLessThan": {
        "aws:CurrentTime": "2031-01-01T00:00:00Z"
      }
    }
  }]
}
```
This denies ALL deletes until 2031 (7-year retention).

**Setup:**
```bash
# Initiate vault lock (24-hour testing period)
LOCK_ID=$(aws glacier initiate-vault-lock \
  --account-id - \
  --vault-name ComplianceVault \
  --policy file://vault-lock-policy.json \
  --query 'lockId' \
  --output text)

# Test the lock for 24 hours...
# If satisfied, complete the lock (PERMANENT!)
aws glacier complete-vault-lock \
  --account-id - \
  --vault-name ComplianceVault \
  --lock-id ${LOCK_ID}
```

**Comparison:**

| Feature | S3 Object Lock | Glacier Vault Lock |
|---------|----------------|---------------------|
| **Granularity** | Per object | Per vault (all objects) |
| **Storage Class** | Any S3 class | Glacier only |
| **Reversible** | Governance mode = Yes | No (compliance mode) |
| **Use Case** | Active data lake | Long-term archives |
| **Cost** | S3 Standard pricing + Object Lock | Glacier pricing + Vault Lock |

**Production example (Financial Services):**
- **S3 Object Lock (Compliance):** Trade confirmations (retain 7 years, SEC Rule 17a-4)
- **Glacier Vault Lock:** Backup tapes (retain 10 years, cannot delete even if wanted to)

**Oracle DBA equivalent:**
- Oracle Flashback Archive = S3 Versioning + Object Lock
- Oracle RMAN backup retention = Glacier Vault Lock

---

## Scenario-Based Interview Questions (10)

### **Q11: Multi-Tenant SaaS S3 Bucket Design**

**Scenario**: Design S3 bucket structure for a multi-tenant SaaS platform

**Question:**  
You're building a SaaS analytics platform where customers upload CSV files for analysis. Each customer's data must be isolated (Customer A cannot see Customer B's data). You expect 10,000 customers, each uploading 1-100GB monthly. Design the S3 bucket structure, IAM policies, and cost optimization strategy.

**Answer:**

**S3 Bucket Structure:**

**Option 1: Single Bucket with Customer Prefixes (Recommended)**
```
s3://saas-analytics-platform/
├── customer-00001/
│   ├── raw/
│   ├── processed/
│   └── results/
├── customer-00002/
│   ├── raw/
│   ├── processed/
│   └── results/
...
├── customer-10000/
    ├── raw/
    ├── processed/
    └── results/
```

**Pros:**
- ✅ Easy to manage (1 bucket vs. 10,000 buckets)
- ✅ Centralized lifecycle policies
- ✅ Centralized encryption with 1 KMS key
- ✅ No risk of hitting account limits (1,000 buckets per account)

**Cons:**
- ❌ Risk of misconfigured IAM policy exposing data
- ❌ All customers share same bucket performance limits (mitigated by prefixes)

**Option 2: Separate Bucket per Customer**
```
s3://customer-00001-data/
s3://customer-00002-data/
...
s3://customer-10000-data/
```

**Pros:**
- ✅ True isolation (no risk of cross-customer access)
- ✅ Easier to implement customer-specific settings (encryption, region)
- ✅ Easier to delete all customer data (compliance: delete on customer request)

**Cons:**
- ❌ Management overhead (10,000 buckets)
- ❌ Hitting account limits (need to request quota increase)
- ❌ Lifecycle policies must be applied 10,000 times

**Recommendation:**  
**Option 1 (single bucket)** unless compliance requires physical isolation.

---

**IAM Policy for Customer Isolation:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAccessToOwnCustomerPrefix",
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:ListBucket"
    ],
    "Resource": [
      "arn:aws:s3:::saas-analytics-platform/${aws:PrincipalTag/CustomerID}/*",
      "arn:aws:s3:::saas-analytics-platform"
    ],
    "Condition": {
      "StringLike": {
        "s3:prefix": ["${aws:PrincipalTag/CustomerID}/*"]
      }
    }
  }]
}
```

**How it works:**
1. Each customer's IAM role has tag: `CustomerID=00001`
2. Policy grants access ONLY to `s3://bucket/${aws:PrincipalTag/CustomerID}/*`
3. Customer 00001 can access `customer-00001/*`, but NOT `customer-00002/*`

**Application code (assuming IAM role):**
```python
import boto3

s3 = boto3.client('s3')

# This works (own prefix)
s3.put_object(
    Bucket='saas-analytics-platform',
    Key='customer-00001/raw/data.csv',
    Body=file_data
)

# This fails with AccessDenied (different prefix)
s3.put_object(
    Bucket='saas-analytics-platform',
    Key='customer-00002/raw/data.csv',  # ❌ Denied
    Body=file_data
)
```

---

**Cost Optimization Strategy:**

| Data Zone | Days | Storage Class | Monthly Cost (100GB) | Annual Cost |
|-----------|------|---------------|----------------------|-------------|
| raw/ (uploaded files) | 0-7 | S3 Standard | $2.30 | $27.60 |
| raw/ (old uploads) | 8-30 | S3 Standard-IA | $1.25 | $15.00 |
| processed/ | 0-90 | S3 Standard | $2.30 | $27.60 |
| results/ | 91+ | S3 Glacier IR | $0.40 | $4.80 |
| **Total per customer** | | | **~$3-6/mo** | **$36-72/yr** |
| **10,000 customers** | | | **$30K-60K/mo** | **$360K-720K/yr** |

**Lifecycle policy:**
```json
{
  "Rules": [
    {
      "Id": "raw-data-lifecycle",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "customer-"
      },
      "Transitions": [
        {"Days": 7, "StorageClass": "STANDARD_IA"},
        {"Days": 30, "StorageClass": "GLACIER_IR"}
      ],
      "NoncurrentVersionExpiration": {"NoncurrentDays": 90}
    }
  ]
}
```

**Additional cost optimizations:**
1. **Compress files before upload** (Gzip reduces size by 70-90%)
   - Customer uploads `data.csv.gz` instead of `data.csv`
   - Athena can query Gzip files directly
   - Savings: 70% storage cost reduction

2. **Use S3 Select for partial reads** (query inside S3, don't download entire file)
   ```python
   response = s3.select_object_content(
       Bucket='saas-analytics-platform',
       Key='customer-00001/raw/data.csv',
       Expression="SELECT * FROM s3object s WHERE s.\"amount\" > 1000",
       ExpressionType='SQL',
       InputSerialization={'CSV': {'FileHeaderInfo': 'Use'}},
       OutputSerialization={'JSON': {}}
   )
   # Only rows with amount > 1000 are transferred (reduces data transfer cost)
   ```

3. **Delete incomplete multipart uploads**
   - Lifecycle rule: Delete after 7 days
   - Prevents orphaned chunks from abandoned uploads

**Security considerations:**
- ✅ **Encryption:** SSE-KMS with customer-specific key or single key (customer choice)
- ✅ **Bucket policy:** Deny access from outside VPC (prevent data exfiltration)
- ✅ **CloudTrail:** Log all S3 API calls (who accessed what, when)
- ✅ **S3 Access Analyzer:** Detect accidentally public objects

---

### **Q12: S3 Data Consistency During ETL Failures**

**Scenario**: S3 data consistency during ETL pipeline failures

**Question:**  
Your Glue ETL job reads from `s3://raw/`, processes data, and writes to `s3://curated/`. The job occasionally fails mid-processing. How do you ensure:
1. No duplicate data in curated zone
2. Failed jobs can be safely retried
3. No data loss if job crashes
4. Consumers always read consistent data

**Answer:**

**Problem:**  
ETL job crashes after writing 50% of output files. If retried, it might create duplicates or partial data.

**Solution: Atomic Writes with S3 Prefix Staging**

**Pattern 1: Write to Temporary Prefix, Then Rename (Atomic Commit)**

```python
import boto3
from datetime import datetime

s3 = boto3.client('s3')
BUCKET = 'my-datalake'

def etl_job():
    job_id = datetime.utcnow().strftime('%Y%m%d_%H%M%S')
    staging_prefix = f'curated/_temp/{job_id}/'
    final_prefix = f'curated/year=2024/month=06/'
    
    try:
        # Step 1: Read from raw/
        raw_data = read_from_s3('raw/input.json')
        
        # Step 2: Process data
        processed_data = transform(raw_data)
        
        # Step 3: Write to STAGING prefix (not final location)
        for i, record in enumerate(processed_data):
            s3.put_object(
                Bucket=BUCKET,
                Key=f'{staging_prefix}part-{i:05d}.parquet',
                Body=record
            )
        
        # Step 4: ATOMIC COMMIT - Copy all files to final location
        # (This is fast because it's a metadata operation, not data copy)
        staging_objects = s3.list_objects_v2(
            Bucket=BUCKET,
            Prefix=staging_prefix
        )['Contents']
        
        for obj in staging_objects:
            source_key = obj['Key']
            final_key = source_key.replace(staging_prefix, final_prefix)
            
            # Copy to final location
            s3.copy_object(
                Bucket=BUCKET,
                CopySource={'Bucket': BUCKET, 'Key': source_key},
                Key=final_key
            )
        
        # Step 5: Delete staging files
        for obj in staging_objects:
            s3.delete_object(Bucket=BUCKET, Key=obj['Key'])
        
        print(f"✅ ETL job {job_id} completed successfully")
        
    except Exception as e:
        print(f"❌ ETL job {job_id} failed: {e}")
        # Staging files remain in _temp/ for debugging
        # Cleanup job deletes _temp/ files older than 7 days
        raise

# Cleanup job (runs daily)
def cleanup_failed_jobs():
    # Delete staging files older than 7 days
    pass
```

**Why this works:**
- ✅ **Atomicity:** Final data appears all-at-once (consumers never see partial data)
- ✅ **Idempotency:** Retry creates new `job_id`, doesn't conflict with previous attempt
- ✅ **No duplicates:** Final location only updated after ALL files written
- ✅ **Debugging:** Failed job leaves staging files for inspection

---

**Pattern 2: Use S3 Object Tagging to Mark Complete Jobs**

```python
import boto3

s3 = boto3.client('s3')

def etl_job_with_completion_marker():
    output_prefix = 'curated/year=2024/month=06/'
    
    try:
        # Write output files
        for i in range(100):
            s3.put_object(
                Bucket='my-datalake',
                Key=f'{output_prefix}part-{i:05d}.parquet',
                Body=data
            )
        
        # Write SUCCESS marker (empty file signaling job completion)
        s3.put_object(
            Bucket='my-datalake',
            Key=f'{output_prefix}_SUCCESS',
            Body=b''
        )
        
    except Exception as e:
        # No _SUCCESS marker = job failed, consumers should ignore
        raise

# Consumer code (Athena, Spark, etc.)
def read_only_complete_partitions():
    partitions = ['year=2024/month=06/', 'year=2024/month=05/']
    
    for partition in partitions:
        # Check for _SUCCESS marker
        try:
            s3.head_object(
                Bucket='my-datalake',
                Key=f'curated/{partition}_SUCCESS'
            )
            # Marker exists, safe to read
            read_partition(partition)
        except s3.exceptions.NoSuchKey:
            # Marker missing, partition incomplete, skip
            print(f'⚠️ Partition {partition} incomplete, skipping')
```

**Why this works:**
- ✅ **Consumers know which data is complete** (_SUCCESS marker present)
- ✅ **Failed jobs don't pollute output** (partial files exist but no marker)
- ✅ **Retry overwrites previous failed attempt** (idempotent)

---

**Pattern 3: Use S3 Versioning + Lifecycle for Cleanup**

Enable versioning on `curated/` bucket:
- ✅ **Failed job writes garbage:** Delete current version, previous version restored
- ✅ **Accidental overwrite:** Restore previous version
- ✅ **Lifecycle policy:** Delete noncurrent versions after 7 days (cleanup)

```bash
# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-datalake \
  --versioning-configuration Status=Enabled

# Lifecycle policy to cleanup old versions
cat > lifecycle.json << 'EOF'
{
  "Rules": [{
    "Id": "cleanup-old-versions",
    "Status": "Enabled",
    "NoncurrentVersionExpiration": {
      "NoncurrentDays": 7
    }
  }]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-datalake \
  --lifecycle-configuration file://lifecycle.json
```

---

**Complete Production-Grade ETL Pattern:**

```python
import boto3
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

class S3AtomicETL:
    def __init__(self, bucket):
        self.s3 = boto3.client('s3')
        self.bucket = bucket
    
    def run_etl(self, raw_prefix, curated_prefix):
        job_id = datetime.utcnow().strftime('%Y%m%d_%H%M%S')
        staging_prefix = f'{curated_prefix}_staging/{job_id}/'
        
        try:
            # Step 1: Read input (with retry logic)
            raw_objects = self.list_objects(raw_prefix)
            logger.info(f'Found {len(raw_objects)} input files')
            
            # Step 2: Process and write to staging
            output_files = []
            for raw_obj in raw_objects:
                data = self.read_object(raw_obj['Key'])
                processed = self.transform(data)
                output_key = self.write_to_staging(processed, staging_prefix)
                output_files.append(output_key)
            
            logger.info(f'Wrote {len(output_files)} files to staging')
            
            # Step 3: Atomic commit (all-or-nothing)
            self.atomic_commit(staging_prefix, curated_prefix)
            
            # Step 4: Write _SUCCESS marker
            self.s3.put_object(
                Bucket=self.bucket,
                Key=f'{curated_prefix}_SUCCESS',
                Body=f'Job {job_id} completed at {datetime.utcnow()}'.encode()
            )
            
            # Step 5: Cleanup staging
            self.cleanup_staging(staging_prefix)
            
            logger.info(f'✅ ETL job {job_id} completed successfully')
            return {'status': 'success', 'job_id': job_id, 'files': len(output_files)}
            
        except Exception as e:
            logger.error(f'❌ ETL job {job_id} failed: {e}')
            # Staging files remain for debugging
            return {'status': 'failed', 'job_id': job_id, 'error': str(e)}
    
    def atomic_commit(self, staging_prefix, final_prefix):
        """Move all files from staging to final location (atomic operation)"""
        staging_files = self.list_objects(staging_prefix)
        
        for file_obj in staging_files:
            source_key = file_obj['Key']
            final_key = source_key.replace(staging_prefix, final_prefix)
            
            # S3 copy is a metadata operation (fast, atomic)
            self.s3.copy_object(
                Bucket=self.bucket,
                CopySource={'Bucket': self.bucket, 'Key': source_key},
                Key=final_key
            )
        
        logger.info(f'Committed {len(staging_files)} files to {final_prefix}')
    
    def cleanup_staging(self, staging_prefix):
        """Delete staging files after successful commit"""
        staging_files = self.list_objects(staging_prefix)
        
        if staging_files:
            self.s3.delete_objects(
                Bucket=self.bucket,
                Delete={
                    'Objects': [{'Key': obj['Key']} for obj in staging_files]
                }
            )
            logger.info(f'Deleted {len(staging_files)} staging files')

# Usage
etl = S3AtomicETL(bucket='my-datalake')
result = etl.run_etl(
    raw_prefix='raw/2024/06/22/',
    curated_prefix='curated/year=2024/month=06/day=22/'
)
print(result)
```

**Benefits:**
- ✅ **Atomic:** Consumers see complete data or nothing
- ✅ **Idempotent:** Retries don't create duplicates
- ✅ **Debuggable:** Failed jobs leave staging files
- ✅ **Consistent:** _SUCCESS marker indicates complete partitions

---

### **Q13: S3 Performance Optimization for High-Throughput Ingestion**

**Scenario**: S3 performance optimization for high-throughput ingestion

**Question:**  
You're ingesting 1 million IoT device messages per second (each message = 1KB JSON). Each device uploads to S3 via API Gateway → Lambda → S3. Current implementation hits rate limits and experiences slow uploads. Optimize this architecture.

**Answer:**

**Current Architecture (BEFORE):**
```
1M devices → API Gateway → Lambda → S3
              └─ Single prefix: s3://bucket/logs/
              └─ 3,500 writes/sec limit = BOTTLENECK
```

**Problem:**
- S3 limit: 3,500 PUT/sec per prefix
- Ingestion rate: 1,000,000 writes/sec
- **Result: 99.65% of writes are throttled!**

---

**Optimized Architecture (AFTER):**

```
1M devices
    ↓
API Gateway (throttle: 10K req/sec per endpoint)
    ↓
Lambda (Fan-out to Kinesis)
    ↓
Kinesis Data Streams (1,000 shards = 1M records/sec)
    ↓
Kinesis Firehose (batches records)
    ↓
S3 (sharded prefixes + Parquet compression)
    ↓
s3://bucket/logs/year=2024/month=06/day=22/hour=14/shard-0000/
s3://bucket/logs/year=2024/month=06/day=22/hour=14/shard-0001/
...
s3://bucket/logs/year=2024/month=06/day=22/hour=14/shard-0999/
```

**Step-by-step optimization:**

---

**Optimization 1: Use Kinesis Data Streams for Buffering**

```python
import boto3
import json
import hashlib

kinesis = boto3.client('kinesis')

def lambda_handler(event, context):
    """
    Lambda function receives IoT device message from API Gateway
    Writes to Kinesis Data Streams (not directly to S3)
    """
    device_message = json.loads(event['body'])
    device_id = device_message['device_id']
    
    # Partition key determines which shard (distribute load)
    partition_key = hashlib.md5(device_id.encode()).hexdigest()
    
    kinesis.put_record(
        StreamName='iot-ingestion-stream',
        Data=json.dumps(device_message).encode(),
        PartitionKey=partition_key
    )
    
    return {'statusCode': 200, 'body': 'Accepted'}

# Kinesis capacity: 1,000 shards × 1,000 records/sec = 1M records/sec ✅
```

**Why this works:**
- ✅ **Kinesis buffers data** (absorbs spikes)
- ✅ **Batching:** Firehose aggregates 1M small records into larger files
- ✅ **Automatic sharding:** Kinesis distributes across shards

---

**Optimization 2: Kinesis Firehose with Dynamic Partitioning**

```bash
# Create Firehose delivery stream with dynamic partitioning
aws firehose create-delivery-stream \
  --delivery-stream-name iot-to-s3 \
  --delivery-stream-type KinesisStreamAsSource \
  --kinesis-stream-source-configuration '{
    "KinesisStreamARN": "arn:aws:kinesis:us-east-1:123456789012:stream/iot-ingestion-stream",
    "RoleARN": "arn:aws:iam::123456789012:role/FirehoseRole"
  }' \
  --extended-s3-destination-configuration '{
    "BucketARN": "arn:aws:s3:::iot-datalake",
    "Prefix": "logs/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
    "ErrorOutputPrefix": "errors/",
    "BufferingHints": {
      "SizeInMBs": 128,
      "IntervalInSeconds": 300
    },
    "CompressionFormat": "GZIP",
    "DataFormatConversionConfiguration": {
      "SchemaConfiguration": {
        "DatabaseName": "iot_db",
        "TableName": "device_logs",
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

**Firehose benefits:**
- ✅ **Batching:** Aggregates 1M small JSON files into 128MB Parquet files
- ✅ **Compression:** Gzip reduces size by 80%
- ✅ **Format conversion:** JSON → Parquet (optimized for Athena)
- ✅ **Partitioning:** Auto-creates S3 prefixes with year/month/day/hour

**Result:**
- **Before:** 1M files × 1KB = 1GB/sec, 86.4 billion objects/day
- **After:** ~300 files × 128MB = 38.4GB/sec, 259,200 objects/day (99.9% reduction!)

---

**Optimization 3: S3 Prefix Sharding for Athena Queries**

Even with Firehose batching, you need prefix sharding for query performance:

```
s3://iot-datalake/logs/
  ├── year=2024/month=06/day=22/hour=14/region=us-east/
  ├── year=2024/month=06/day=22/hour=14/region=us-west/
  ├── year=2024/month=06/day=22/hour=14/region=eu-west/
  ...
```

**Athena query:**
```sql
SELECT COUNT(*) 
FROM device_logs 
WHERE year = 2024 
  AND month = 6 
  AND day = 22 
  AND hour = 14
  AND region = 'us-east';

-- Scans ONLY 1 partition instead of entire dataset
-- Query cost: $0.01 instead of $100 (10,000x reduction!)
```

---

**Optimization 4: Use S3 Transfer Acceleration for Global Devices**

```python
import boto3

# Enable Transfer Acceleration on bucket
s3 = boto3.client('s3')
s3.put_bucket_accelerate_configuration(
    Bucket='iot-datalake',
    AccelerateConfiguration={'Status': 'Enabled'}
)

# Upload using accelerated endpoint
s3_accelerate = boto3.client(
    's3',
    endpoint_url='https://s3-accelerate.amazonaws.com'
)

s3_accelerate.put_object(
    Bucket='iot-datalake',
    Key='logs/device-12345.json',
    Body=data
)

# 50% faster uploads from Asia/Europe to us-east-1 bucket
```

---

**Final Optimized Architecture:**

```
┌────────────────────────────────────────────────────────────────┐
│                    IoT Ingestion Pipeline                      │
└────────────────────────────────────────────────────────────────┘

1M IoT Devices (Global)
    │
    ├──► API Gateway (Regional endpoints)
    │    ├─ us-east-1: 300K devices
    │    ├─ eu-west-1: 400K devices
    │    └─ ap-southeast-1: 300K devices
    │
    ├──► Lambda (Fan-out)
    │    └─ Writes to Kinesis Data Streams
    │
    ├──► Kinesis Data Streams
    │    ├─ 1,000 shards
    │    ├─ 1M records/sec capacity
    │    └─ 24-hour retention
    │
    ├──► Kinesis Firehose
    │    ├─ Batching: 128MB or 5 minutes
    │    ├─ Compression: Gzip
    │    ├─ Format: JSON → Parquet
    │    └─ Dynamic partitioning
    │
    └──► S3 (Multi-prefix)
         └─ logs/year=YYYY/month=MM/day=DD/hour=HH/region=XX/
            ├─ file-001.parquet.gz (128MB)
            ├─ file-002.parquet.gz (128MB)
            ...

┌────────────────────────────────────────────────────────────────┐
│                     Query Layer (Athena)                       │
└────────────────────────────────────────────────────────────────┘

Athena (Partition Projection for fast queries)
    ├─ Scans only relevant partitions
    ├─ Columnar format (Parquet) = 10x faster
    └─ Query cost: $5/TB scanned (instead of $5/day)
```

**Performance Results:**

| Metric | Before (Direct S3) | After (Kinesis + Firehose) | Improvement |
|--------|-------------------|----------------------------|-------------|
| **Write Throughput** | 3,500/sec (throttled) | 1,000,000/sec | 286x |
| **Objects Created/Day** | 86.4 billion | 259,200 | 333,333x fewer |
| **Storage Cost/Day** | $2,000 | $160 (compression) | 12.5x cheaper |
| **Query Performance** | 10 minutes | 5 seconds | 120x faster |
| **S3 API Costs** | $432,000/day | $1.30/day | 332,307x cheaper |

**Total Monthly Savings: ~$13M** 🎉

---

**Key Takeaways:**

1. **Never upload millions of tiny files directly to S3**
   - Use Kinesis/Firehose to batch into larger files

2. **Shard prefixes for both writes AND reads**
   - Writes: Avoid throttling
   - Reads: Enable partition pruning

3. **Choose right file format**
   - JSON: Human-readable, but slow queries
   - Parquet: 10x faster queries, 5x smaller size

4. **Use managed services**
   - Kinesis handles scaling automatically
   - Firehose handles batching, compression, partitioning

5. **Monitor with CloudWatch**
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/Kinesis \
     --metric-name IncomingRecords \
     --dimensions Name=StreamName,Value=iot-ingestion-stream \
     --start-time 2024-06-22T00:00:00Z \
     --end-time 2024-06-22T23:59:59Z \
     --period 60 \
     --statistics Sum
   ```

---

This concludes the detailed scenario-based questions. Would you like me to continue with the remaining sections (Certification Tips, Production Best Practices, Hands-On Challenges, Resume Project, Solution Engineer Perspective, and Cleanup)?

---



### **Q14: Cost Optimization - Reducing S3 Storage Costs by 84%**

**Scenario**: Your data lake stores 2.5 PB (2,555 TB) of data with the following access patterns:
- **Raw data** (1,200 TB): Accessed daily for 7 days, then rarely accessed
- **Processed data** (1,000 TB): Accessed weekly for 30 days, then archived
- **Curated data** (355 TB): Accessed monthly for 90 days, compliance retention 7 years

Current cost: $705,180/year (all S3 Standard). Design a cost-optimized storage strategy.

**Solution:**

**Storage Class Selection:**

```
┌─────────────────────────────────────────────────────────────┐
│           Cost-Optimized Data Lake Architecture            │
└─────────────────────────────────────────────────────────────┘

Raw Zone (1,200 TB):
├─ Days 0-7: S3 Standard (hot access)          $27,600/year
├─ Days 8-90: S3 Standard-IA (warm)            $18,000/year  
└─ Days 91+: S3 Glacier Instant Retrieval      $5,760/year
   Total raw zone: $51,360/year

Processed Zone (1,000 TB):
├─ Days 0-30: S3 Standard                      $23,000/year
├─ Days 31-365: S3 Standard-IA                 $15,000/year
└─ Year 2+: S3 Glacier Flexible Retrieval      $4,320/year
   Total processed zone: $42,320/year

Curated Zone (355 TB):
├─ Days 0-90: S3 Standard                      $8,165/year
└─ Days 91-7 years: S3 Glacier Deep Archive    $421/year
   Total curated zone: $8,586/year

TOTAL OPTIMIZED COST: $102,266/year
SAVINGS: $602,914/year (85% reduction)
```

**Lifecycle Policy Implementation:**

```json
{
  "Rules": [
    {
      "Id": "raw-data-lifecycle",
      "Prefix": "raw/",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 7,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        }
      ]
    },
    {
      "Id": "processed-data-lifecycle",
      "Prefix": "processed/",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER"
        }
      ]
    },
    {
      "Id": "curated-data-lifecycle",
      "Prefix": "curated/",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    },
    {
      "Id": "delete-incomplete-multipart",
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

**Apply lifecycle policy:**

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-datalake \
  --lifecycle-configuration file://lifecycle-policy.json
```

**Results:**

| Zone | Size | Before | After | Savings |
|------|------|--------|-------|---------|
| Raw | 1,200 TB | $276K/yr | $51K/yr | 81% |
| Processed | 1,000 TB | $230K/yr | $42K/yr | 82% |
| Curated | 355 TB | $82K/yr | $9K/yr | 89% |
| **TOTAL** | **2,555 TB** | **$705K/yr** | **$102K/yr** | **85%** |

**Additional Optimizations:**

1. **S3 Intelligent-Tiering** for unknown access patterns (auto-optimizes):
   ```bash
   aws s3api put-bucket-intelligent-tiering-configuration \
     --bucket my-datalake \
     --id unknown-access-pattern \
     --intelligent-tiering-configuration '{
       "Id": "unknown-access-pattern",
       "Status": "Enabled",
       "Tierings": [
         {"Days": 90, "AccessTier": "ARCHIVE_ACCESS"},
         {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
       ]
     }'
   ```

2. **Delete incomplete multipart uploads** (saves storage costs):
   - Configured in lifecycle policy above (7 days)

3. **S3 Storage Lens** for visibility:
   ```bash
   aws s3control create-storage-lens-configuration \
     --account-id 123456789012 \
     --config-id cost-optimization-lens \
     --storage-lens-configuration file://lens-config.json
   ```

---

### **Q15: S3 Security Architecture - 7-Layer Defense**

**Scenario**: Design a comprehensive S3 security architecture for a healthcare data lake storing HIPAA-compliant patient records. Requirements:
- Encryption at rest and in transit
- Access logging and monitoring
- Data cannot be deleted for 7 years (compliance)
- Only authorized users can access specific patient data
- Audit trail for all access

**Solution:**

**7-Layer Security Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│              HIPAA-Compliant S3 Security Layers            │
└─────────────────────────────────────────────────────────────┘

Layer 1: Network Isolation
├─ VPC Endpoints (private access, no internet)
├─ S3 bucket in private subnet
└─ Security groups restrict access

Layer 2: Encryption at Rest
├─ SSE-KMS (AWS managed key)
├─ Customer Master Key (CMK) per department
└─ Automatic encryption enforcement

Layer 3: Encryption in Transit
├─ Enforce HTTPS (deny HTTP)
├─ TLS 1.2+ required
└─ Bucket policy denies non-SSL

Layer 4: Access Control
├─ IAM policies (least privilege)
├─ Bucket policies (resource-based)
├─ S3 Access Points (per-application)
└─ Pre-signed URLs (time-limited)

Layer 5: Immutability
├─ S3 Object Lock (Compliance mode)
├─ 7-year retention period
└─ MFA Delete enabled

Layer 6: Logging & Monitoring
├─ S3 Server Access Logs
├─ CloudTrail (data events)
├─ S3 Event Notifications
└─ CloudWatch alarms

Layer 7: Data Classification
├─ S3 Object Tags (PHI, PII)
├─ Macie (auto-detect sensitive data)
└─ Lake Formation (column-level access)
```

**Implementation:**

**Layer 1 & 2: Encryption**

```bash
# Create KMS key for encryption
aws kms create-key \
  --description "HIPAA patient data encryption key" \
  --key-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "Enable IAM policies",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
      "Action": "kms:*",
      "Resource": "*"
    }]
  }'

# Enable default encryption
aws s3api put-bucket-encryption \
  --bucket hipaa-patient-data \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abc-123"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

**Layer 3: Enforce HTTPS**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyInsecureTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::hipaa-patient-data",
      "arn:aws:s3:::hipaa-patient-data/*"
    ],
    "Condition": {
      "Bool": {
        "aws:SecureTransport": "false"
      }
    }
  }]
}
```

**Layer 4: Least Privilege IAM**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:ListBucket"
    ],
    "Resource": [
      "arn:aws:s3:::hipaa-patient-data",
      "arn:aws:s3:::hipaa-patient-data/cardiology/*"
    ],
    "Condition": {
      "StringEquals": {
        "s3:ExistingObjectTag/Department": "Cardiology"
      }
    }
  }]
}
```

**Layer 5: Object Lock (7-year retention)**

```bash
# Enable Object Lock at bucket creation
aws s3api create-bucket \
  --bucket hipaa-patient-data \
  --object-lock-enabled-for-bucket

# Set default retention (7 years = 2555 days)
aws s3api put-object-lock-configuration \
  --bucket hipaa-patient-data \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 2555
      }
    }
  }'

# Enable MFA Delete
aws s3api put-bucket-versioning \
  --bucket hipaa-patient-data \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa 123456"
```

**Layer 6: Logging & Monitoring**

```bash
# Enable S3 access logging
aws s3api put-bucket-logging \
  --bucket hipaa-patient-data \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "hipaa-access-logs",
      "TargetPrefix": "patient-data-logs/"
    }
  }'

# Enable CloudTrail data events
aws cloudtrail put-event-selectors \
  --trail-name hipaa-trail \
  --event-selectors '[{
    "ReadWriteType": "All",
    "IncludeManagementEvents": true,
    "DataResources": [{
      "Type": "AWS::S3::Object",
      "Values": ["arn:aws:s3:::hipaa-patient-data/*"]
    }]
  }]'

# CloudWatch alarm for unauthorized access
aws cloudwatch put-metric-alarm \
  --alarm-name s3-unauthorized-access \
  --alarm-description "Alert on S3 access denied events" \
  --metric-name AccessDenied \
  --namespace AWS/S3 \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:security-alerts
```

**Results:**

| Security Layer | Implementation | Compliance |
|----------------|----------------|------------|
| Network Isolation | VPC Endpoint | ✅ HIPAA |
| Encryption at Rest | KMS CMK | ✅ HIPAA |
| Encryption in Transit | TLS 1.2+ | ✅ HIPAA |
| Access Control | IAM + Bucket Policy | ✅ HIPAA |
| Immutability | Object Lock Compliance | ✅ HIPAA |
| Logging | CloudTrail + S3 Logs | ✅ HIPAA |
| Monitoring | CloudWatch Alarms | ✅ HIPAA |

---

### **Q16: Event-Driven Data Pipeline - S3 → EventBridge → Lambda → Glue**

**Scenario**: Build an event-driven ETL pipeline that automatically processes CSV files uploaded to S3, validates data quality, and loads into Glue Data Catalog for Athena queries.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│           Event-Driven ETL Pipeline                        │
└─────────────────────────────────────────────────────────────┘

User uploads CSV → S3 (raw/)
         ↓
    S3 Event Notification
         ↓
    EventBridge Rule (filter: *.csv)
         ↓
    Lambda (validate-data)
         ├─ Valid: Trigger Glue Crawler
         └─ Invalid: Send to DLQ (Dead Letter Queue)
              ↓
         Glue Crawler
              ├─ Infer schema
              ├─ Update Glue Data Catalog
              └─ Trigger SNS notification
                   ↓
              Athena Query Available
```

**Implementation:**

**Step 1: S3 Bucket with EventBridge**

```bash
# Create S3 bucket
aws s3api create-bucket --bucket event-driven-datalake

# Enable EventBridge notifications
aws s3api put-bucket-notification-configuration \
  --bucket event-driven-datalake \
  --notification-configuration '{
    "EventBridgeConfiguration": {}
  }'
```

**Step 2: EventBridge Rule**

```bash
# Create EventBridge rule (filter for CSV files)
aws events put-rule \
  --name s3-csv-upload-rule \
  --event-pattern '{
    "source": ["aws.s3"],
    "detail-type": ["Object Created"],
    "detail": {
      "bucket": {"name": ["event-driven-datalake"]},
      "object": {
        "key": [{"suffix": ".csv"}]
      }
    }
  }'

# Add Lambda as target
aws events put-targets \
  --rule s3-csv-upload-rule \
  --targets "Id=1,Arn=arn:aws:lambda:us-east-1:123456789012:function:validate-data"
```

**Step 3: Lambda Validation Function**

```python
# lambda_function.py
import boto3
import csv
import io

s3 = boto3.client('s3')
glue = boto3.client('glue')
sns = boto3.client('sns')

def lambda_handler(event, context):
    """
    Validate CSV file and trigger Glue Crawler
    """
    
    # Extract S3 details from EventBridge event
    bucket = event['detail']['bucket']['name']
    key = event['detail']['object']['key']
    
    print(f"Processing: s3://{bucket}/{key}")
    
    try:
        # Download CSV file
        response = s3.get_object(Bucket=bucket, Key=key)
        csv_content = response['Body'].read().decode('utf-8')
        
        # Validate CSV structure
        csv_reader = csv.DictReader(io.StringIO(csv_content))
        rows = list(csv_reader)
        
        # Data quality checks
        if len(rows) == 0:
            raise ValueError("Empty CSV file")
        
        required_columns = ['user_id', 'timestamp', 'event_type']
        if not all(col in csv_reader.fieldnames for col in required_columns):
            raise ValueError(f"Missing required columns: {required_columns}")
        
        # Validation passed - trigger Glue Crawler
        glue.start_crawler(Name='csv-data-crawler')
        
        # Send success notification
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:data-pipeline-alerts',
            Subject='✅ CSV File Processed Successfully',
            Message=f'File: s3://{bucket}/{key}\nRows: {len(rows)}\nStatus: VALID'
        )
        
        return {
            'statusCode': 200,
            'body': f'Successfully validated {len(rows)} rows'
        }
        
    except Exception as e:
        # Validation failed - move to DLQ
        dlq_key = key.replace('raw/', 'dlq/')
        s3.copy_object(
            Bucket=bucket,
            CopySource={'Bucket': bucket, 'Key': key},
            Key=dlq_key
        )
        
        # Send failure notification
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:data-pipeline-alerts',
            Subject='❌ CSV Validation Failed',
            Message=f'File: s3://{bucket}/{key}\nError: {str(e)}\nMoved to DLQ: {dlq_key}'
        )
        
        raise
```

**Step 4: Glue Crawler**

```bash
# Create Glue Crawler
aws glue create-crawler \
  --name csv-data-crawler \
  --role GlueServiceRole \
  --database-name analytics_db \
  --targets '{
    "S3Targets": [{
      "Path": "s3://event-driven-datalake/raw/"
    }]
  }' \
  --schema-change-policy '{
    "UpdateBehavior": "UPDATE_IN_DATABASE",
    "DeleteBehavior": "DEPRECATE_IN_DATABASE"
  }'
```

**Step 5: Query with Athena**

```sql
-- Data is now queryable via Athena
SELECT 
    user_id,
    event_type,
    COUNT(*) AS event_count
FROM analytics_db.raw
WHERE DATE(timestamp) = CURRENT_DATE
GROUP BY user_id, event_type
ORDER BY event_count DESC
LIMIT 10;
```

**Results:**

| Metric | Value |
|--------|-------|
| **Latency** | 30-60 seconds (upload → queryable) |
| **Cost** | $0.20 per 1,000 files processed |
| **Automation** | 100% (no manual intervention) |
| **Data Quality** | Validated before processing |
| **Monitoring** | SNS notifications for success/failure |

---

### **Q17: Cross-Region Disaster Recovery with S3 Replication**

**Scenario**: Design a disaster recovery strategy for a critical data lake using S3 Cross-Region Replication (CRR). Requirements: RTO < 1 hour, RPO < 15 minutes, automatic failover.

**Solution:**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│          S3 Cross-Region Disaster Recovery                 │
└─────────────────────────────────────────────────────────────┘

Primary Region: us-east-1
┌──────────────────────────────┐
│ S3: prod-datalake-us-east-1  │
│ ├─ Versioning: Enabled       │
│ ├─ Encryption: KMS           │
│ ├─ Size: 500 TB              │
│ └─ Replication: → us-west-2  │
└──────────────────────────────┘
         │ S3 CRR (99.99% SLA)
         │ Replication lag: < 15 min
         ↓
Secondary Region: us-west-2 (DR)
┌──────────────────────────────┐
│ S3: prod-datalake-us-west-2  │
│ ├─ Versioning: Enabled       │
│ ├─ Encryption: KMS           │
│ ├─ Size: 500 TB (replica)    │
│ └─ Status: Standby           │
└──────────────────────────────┘
         │
         │ Route 53 Failover
         ↓
    Application (reads from DR bucket on failure)
```

**Implementation:**

**Step 1: Enable Versioning (Required for CRR)**

```bash
# Primary bucket
aws s3api put-bucket-versioning \
  --bucket prod-datalake-us-east-1 \
  --versioning-configuration Status=Enabled

# DR bucket
aws s3api put-bucket-versioning \
  --bucket prod-datalake-us-west-2 \
  --versioning-configuration Status=Enabled
```

**Step 2: Create IAM Role for Replication**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "s3.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::prod-datalake-us-east-1"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl"
      ],
      "Resource": "arn:aws:s3:::prod-datalake-us-east-1/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete"
      ],
      "Resource": "arn:aws:s3:::prod-datalake-us-west-2/*"
    }
  ]
}
```

**Step 3: Configure S3 Replication**

```bash
aws s3api put-bucket-replication \
  --bucket prod-datalake-us-east-1 \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
    "Rules": [{
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::prod-datalake-us-west-2",
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
        }
      },
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      }
    }]
  }'
```

**Step 4: Monitor Replication**

```bash
# Check replication status
aws s3api get-bucket-replication \
  --bucket prod-datalake-us-east-1

# CloudWatch metrics for replication lag
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name ReplicationLatency \
  --dimensions Name=SourceBucket,Value=prod-datalake-us-east-1 \
  --start-time 2024-06-22T00:00:00Z \
  --end-time 2024-06-22T23:59:59Z \
  --period 3600 \
  --statistics Average,Maximum

# Alarm for replication lag > 15 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name s3-replication-lag-high \
  --metric-name ReplicationLatency \
  --namespace AWS/S3 \
  --statistic Maximum \
  --period 900 \
  --threshold 900000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:dr-alerts
```

**Step 5: Failover to DR Region**

```python
# dr_failover.py
import boto3

route53 = boto3.client('route53')
sns = boto3.client('sns')

def failover_to_dr():
    """
    Failover to DR region (us-west-2)
    """
    
    # Update Route 53 to point to DR bucket
    route53.change_resource_record_sets(
        HostedZoneId='Z1234567890',
        ChangeBatch={
            'Changes': [{
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': 'datalake.company.com',
                    'Type': 'CNAME',
                    'TTL': 60,
                    'ResourceRecords': [{
                        'Value': 'prod-datalake-us-west-2.s3.us-west-2.amazonaws.com'
                    }]
                }
            }]
        }
    )
    
    # Notify ops team
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:dr-alerts',
        Subject='🚨 DR FAILOVER INITIATED',
        Message=f"""
        Failover to DR region completed.
        
        Primary: us-east-1 (FAILED)
        DR: us-west-2 (ACTIVE)
        
        DNS: datalake.company.com → us-west-2
        RTO: < 1 hour
        RPO: < 15 minutes
        
        Next steps:
        1. Verify application connectivity
        2. Investigate us-east-1 failure
        3. Plan failback when primary restored
        """
    )
    
    return {'statusCode': 200, 'body': 'Failover complete'}
```

**Results:**

| Metric | Target | Actual |
|--------|--------|--------|
| **RTO** | < 1 hour | 30 minutes |
| **RPO** | < 15 minutes | 5-10 minutes |
| **Replication Lag** | < 15 min | 5-10 min (avg) |
| **Cost** | N/A | $1,000/month (500TB replication) |
| **Availability** | 99.99% | 99.99% |

---

### **Q18: Data Lake Governance with AWS Lake Formation**

**Scenario**: Implement fine-grained access control for a data lake with 50 tables across 5 departments. Requirements:
- Row-level security (sales reps see only their region's data)
- Column-level security (mask PII from analysts)
- Grant permissions via Lake Formation (not IAM)

**Solution:**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│          Lake Formation Access Control                     │
└─────────────────────────────────────────────────────────────┘

Glue Data Catalog:
├─ Database: sales_db
│   ├─ Table: customers (10 columns, 5M rows)
│   │   ├─ Columns: customer_id, name, email, region, revenue
│   │   └─ Sensitive: email (PII)
│   └─ Table: orders (8 columns, 50M rows)
│       └─ Partition: region (US-WEST, US-EAST, EU)

Lake Formation Permissions:
├─ SalesManager (all regions, all columns)
├─ SalesRep_USWest (region=US-WEST, email masked)
├─ Analyst (all regions, email + SSN masked)
└─ Compliance (all data, read-only)
```

**Implementation:**

**Step 1: Register S3 Location with Lake Formation**

```bash
# Register S3 path
aws lakeformation register-resource \
  --resource-arn arn:aws:s3:::sales-datalake \
  --use-service-linked-role

# Grant Lake Formation permissions (revoke IAM permissions)
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/LakeFormationAdmin \
  --permissions ALL \
  --resource '{
    "Database": {
      "Name": "sales_db"
    }
  }'
```

**Step 2: Create Row-Level Security (Data Filter)**

```bash
# Row filter for US-WEST sales rep
aws lakeformation create-data-cells-filter \
  --table-data '{
    "DatabaseName": "sales_db",
    "TableName": "orders",
    "Name": "uswest_filter",
    "RowFilter": {
      "FilterExpression": "region = '\''US-WEST'\''"
    }
  }'
```

**Step 3: Create Column-Level Security (Mask PII)**

```bash
# Grant permissions with column exclusions
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/AnalystRole \
  --permissions SELECT \
  --resource '{
    "TableWithColumns": {
      "DatabaseName": "sales_db",
      "TableName": "customers",
      "ColumnWildcard": {
        "ExcludedColumnNames": ["email", "ssn", "phone"]
      }
    }
  }'
```

**Step 4: Grant Row + Column Permissions**

```bash
# Sales rep (US-WEST only, email masked)
aws lakeformation grant-permissions \
  --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/SalesRepUSWest \
  --permissions SELECT \
  --resource '{
    "DataCellsFilter": {
      "DatabaseName": "sales_db",
      "TableName": "orders",
      "Name": "uswest_filter"
    }
  }'
```

**Step 5: Query with Athena**

```sql
-- Sales manager sees all rows
SELECT region, COUNT(*) AS order_count, SUM(revenue) AS total_revenue
FROM sales_db.orders
GROUP BY region;

-- Result (sales manager):
-- region    | order_count | total_revenue
-- US-WEST   | 15M         | $450M
-- US-EAST   | 20M         | $600M
-- EU        | 15M         | $400M

-- Sales rep (US-WEST) sees only their region
SELECT region, COUNT(*) AS order_count
FROM sales_db.orders
GROUP BY region;

-- Result (sales rep):
-- region    | order_count
-- US-WEST   | 15M
-- (other regions filtered out by Lake Formation)

-- Analyst cannot see PII columns
SELECT customer_id, email, revenue
FROM sales_db.customers
LIMIT 10;

-- Result (analyst):
-- customer_id | email        | revenue
-- C001        | [MASKED]     | $5,000
-- (email column masked by Lake Formation)
```

**Results:**

| User Role | Row Access | Column Access | Implementation |
|-----------|------------|---------------|----------------|
| SalesManager | All regions | All columns | Lake Formation grant |
| SalesRep_USWest | US-WEST only | Email masked | Data cells filter |
| Analyst | All regions | PII masked | Column exclusion |
| Compliance | All regions | All columns (read-only) | Read-only grant |

**Benefits:**
- ✅ Centralized access control (no IAM policies per user)
- ✅ Fine-grained security (row + column level)
- ✅ Audit trail (CloudTrail logs all access)
- ✅ Scales to 1000s of users

---

### **Q19: S3 Intelligent-Tiering vs Manual Lifecycle Policies**

**Scenario**: You have a 1 PB data lake with unpredictable access patterns. Some data is accessed frequently for weeks, then rarely for months. Compare S3 Intelligent-Tiering vs manual lifecycle policies for cost optimization.

**Comparison:**

**Option 1: S3 Intelligent-Tiering (Recommended)**

```
How it works:
├─ Monitors access patterns automatically
├─ Moves objects between tiers based on access
├─ No retrieval fees for Frequent/Infrequent tiers
└─ No lifecycle policies needed

Tiers:
├─ Frequent Access (< 30 days since last access)
├─ Infrequent Access (30-90 days)
├─ Archive Instant Access (90-180 days)
├─ Archive Access (180-365 days)
└─ Deep Archive Access (365+ days)
```

**Setup:**

```bash
# Enable Intelligent-Tiering on entire bucket
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-datalake \
  --id intelligent-tiering-config \
  --intelligent-tiering-configuration '{
    "Id": "intelligent-tiering-config",
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
  }'
```

**Cost (1 PB with 20% frequent, 50% infrequent, 30% archive):**

| Tier | Size | Cost/GB/Month | Total/Month |
|------|------|---------------|-------------|
| Frequent Access (20%) | 200 TB | $0.023 | $4,600 |
| Infrequent Access (50%) | 500 TB | $0.0125 | $6,250 |
| Archive Access (30%) | 300 TB | $0.004 | $1,200 |
| Monitoring fee | 1 PB | $0.0025 | $2,500 |
| **TOTAL** | **1 PB** | - | **$14,550/month** |

---

**Option 2: Manual Lifecycle Policies**

```json
{
  "Rules": [{
    "Id": "transition-to-ia",
    "Status": "Enabled",
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER_IR"},
      {"Days": 365, "StorageClass": "GLACIER"}
    ]
  }]
}
```

**Cost (same 1 PB, but FIXED transition schedule):**

| Tier | Size | Cost/GB/Month | Total/Month |
|------|------|---------------|-------------|
| Standard (0-30 days) | 100 TB | $0.023 | $2,300 |
| Standard-IA (30-90 days) | 200 TB | $0.0125 | $2,500 |
| Glacier IR (90-365 days) | 400 TB | $0.004 | $1,600 |
| Glacier (365+ days) | 300 TB | $0.0036 | $1,080 |
| **TOTAL** | **1 PB** | - | **$7,480/month** |

**BUT: Retrieval costs for unpredictable access:**
- Standard-IA: $0.01/GB retrieval
- Glacier IR: $0.03/GB retrieval
- If you retrieve 100 TB/month: +$1,000 to $3,000

**Total with retrievals: $10,480/month**

---

**Comparison:**

| Metric | Intelligent-Tiering | Manual Lifecycle |
|--------|---------------------|------------------|
| **Monthly Cost** | $14,550 | $7,480 + retrieval |
| **Retrieval Fees** | $0 (Frequent/Infrequent) | $1,000-$3,000/month |
| **Complexity** | Low (automatic) | High (tune policies) |
| **Flexibility** | Adapts to changes | Fixed schedule |
| **Best For** | Unknown patterns | Predictable patterns |

**Recommendation:**
- **Use Intelligent-Tiering** if access patterns are unpredictable
- **Use Manual Lifecycle** if access patterns are well-known and stable

---

### **Q20: Optimizing Large File Uploads - Multipart Upload & Transfer Acceleration**

**Scenario**: You need to upload 5 TB of daily logs (10,000 files, 500 MB each) from on-premises to S3. Upload via standard PUT is slow (12 hours) and frequently fails. Optimize for speed and reliability.

**Solution:**

**Option 1: Multipart Upload (Recommended for Large Files)**

**How it works:**
```
Large file (500 MB) split into parts (5 MB each):
├─ Part 1 (5 MB) → Upload in parallel
├─ Part 2 (5 MB) → Upload in parallel
├─ ...
└─ Part 100 (5 MB) → Upload in parallel
      ↓
AWS assembles parts into single object
```

**Benefits:**
- ✅ **Parallel uploads** (10x faster)
- ✅ **Resume failed uploads** (retry only failed parts)
- ✅ **Improved throughput** (utilize full network bandwidth)

**Implementation:**

```python
import boto3
from boto3.s3.transfer import TransferConfig

s3 = boto3.client('s3')

# Configure multipart upload (5 MB parts)
config = TransferConfig(
    multipart_threshold=5 * 1024 * 1024,  # 5 MB
    max_concurrency=10,                    # 10 parallel uploads
    multipart_chunksize=5 * 1024 * 1024,   # 5 MB parts
    use_threads=True
)

# Upload large file
s3.upload_file(
    Filename='/data/logs/app-2024-06-22.log',
    Bucket='my-datalake',
    Key='raw/logs/app-2024-06-22.log',
    Config=config
)
```

**AWS CLI:**

```bash
# Multipart upload (automatic for files > 8 MB)
aws s3 cp /data/logs/app-2024-06-22.log s3://my-datalake/raw/logs/ \
  --storage-class STANDARD \
  --metadata "source=on-prem,date=2024-06-22"

# Monitor multipart uploads
aws s3api list-multipart-uploads --bucket my-datalake

# Cleanup incomplete multipart uploads (saves storage)
aws s3api abort-multipart-upload \
  --bucket my-datalake \
  --key raw/logs/app-2024-06-22.log \
  --upload-id <upload-id>
```

---

**Option 2: S3 Transfer Acceleration (For Geographic Distance)**

**How it works:**
```
On-premises (Asia) → CloudFront Edge (Asia) → AWS Backbone → S3 (us-east-1)
                     (Fast upload)              (Fast transfer)
```

**Benefits:**
- ✅ **50-500% faster** for long-distance uploads
- ✅ **Uses AWS global network** (bypasses public internet)
- ✅ **No client-side changes** (just change endpoint)

**Setup:**

```bash
# Enable Transfer Acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket my-datalake \
  --accelerate-configuration Status=Enabled

# Upload using accelerated endpoint
aws s3 cp /data/logs/app-2024-06-22.log \
  s3://my-datalake/raw/logs/ \
  --endpoint-url https://my-datalake.s3-accelerate.amazonaws.com
```

**Speed Test:**

```bash
# Test standard upload vs accelerated
aws s3 cp test-file.dat s3://my-datalake/ --region us-east-1
# Standard: 2 MB/s (slow from Asia)

aws s3 cp test-file.dat s3://my-datalake/ \
  --endpoint-url https://my-datalake.s3-accelerate.amazonaws.com
# Accelerated: 15 MB/s (7.5x faster!)
```

---

**Combined Solution: Multipart + Transfer Acceleration**

```python
import boto3
from boto3.s3.transfer import TransferConfig

# Use Transfer Acceleration endpoint
s3 = boto3.client(
    's3',
    endpoint_url='https://my-datalake.s3-accelerate.amazonaws.com'
)

# Multipart upload with acceleration
config = TransferConfig(
    multipart_threshold=5 * 1024 * 1024,
    max_concurrency=10,
    multipart_chunksize=5 * 1024 * 1024
)

# Upload
s3.upload_file(
    Filename='/data/logs/large-file.log',
    Bucket='my-datalake',
    Key='raw/logs/large-file.log',
    Config=config
)
```

**Results:**

| Method | Upload Time (5 TB) | Cost | Reliability |
|--------|-------------------|------|-------------|
| Standard PUT | 12 hours | $0 | Low (fails often) |
| Multipart Upload | 2 hours | $0 | High (resume on failure) |
| Transfer Acceleration | 4 hours | $200 (data transfer) | Medium |
| **Multipart + Acceleration** | **1.5 hours** | **$200** | **High** |

**Recommendation:**
- **Local upload**: Multipart Upload (no extra cost)
- **Cross-continent upload**: Multipart + Transfer Acceleration (6x faster)

---

# AWS CERTIFIED DATA ENGINEER ASSOCIATE (DEA-C01)
## COMPLETE HANDS-ON RUNBOOK - MODULES 1-4
### Professional Training Manual for Certification & Interview Preparation

---

**Author Profile:** Senior AWS Data Engineer, AWS Instructor, Interview Coach  
**Target Audience:** Beginner to Intermediate learners with 14+ years Oracle DBA experience transitioning to Data Engineering  
**Certification:** AWS Certified Data Engineer – Associate (DEA-C01)  
**Date:** June 2026  

---

# TABLE OF CONTENTS

## MODULE 1: IAM FOR DATA ENGINEERS
## MODULE 2: AMAZON S3 FOUNDATIONS  


---

# MODULE 1: IAM FOR DATA ENGINEERS

## MODULE OVERVIEW

**Duration:** 3-4 hours  
**Difficulty:** Beginner  
**Prerequisites:** Active AWS Account, Basic understanding of security concepts  
**Cost:** $0 (Free Tier)  

---

## WHAT YOU WILL LEARN

### Why IAM Exists
AWS Identity and Access Management (IAM) is the foundation of AWS security. Think of it as Oracle's role-based access control (RBAC) but designed for cloud services. Every AWS service interaction requires proper IAM permissions.

**Production Use Cases:**
- Data engineers need read access to S3 but not deletion rights
- Glue jobs need permissions to read S3, write to databases, and access Secrets Manager
- Athena users need specific database/table permissions
- Cross-account data sharing in multi-tenant environments

### How IAM Fits Into Data Engineering Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    DATA ENGINEERING PIPELINE                 │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  User/Application  →  IAM Authentication  →  IAM Authorization│
│         ↓                                                     │
│    IAM Role/Policy  →  S3 Bucket  →  Glue Job  →  Athena    │
│                           ↓             ↓          ↓         │
│                      Encrypted      Catalog    Query         │
│                      at Rest       Metadata    Data          │
└─────────────────────────────────────────────────────────────┘
```

### Common Beginner Mistakes
1. **Using Root Account** - Never use root for daily operations
2. **Overly Permissive Policies** - Granting `*:*` permissions (admin access to everything)
3. **Hardcoding Credentials** - Storing access keys in code instead of using roles
4. **Not Enabling MFA** - Multi-factor authentication should always be enabled
5. **Inline Policies** - Using inline policies instead of managed policies (hard to maintain)

---

## EXERCISE 1.1: CREATE IAM USERS FOR DATA ENGINEERING TEAM

### Exercise Overview

**Purpose:** Create IAM users with least privilege access for data engineering tasks  
**Real-World Use Case:** You're building a data engineering team. You need a Data Engineer who can access S3 and run Glue jobs, and a Data Analyst who can only query data via Athena.  
**Services Used:** IAM  
**Estimated Duration:** 30 minutes  

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      AWS ACCOUNT                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                  IAM USERS                              │  │
│  │                                                          │  │
│  │  ┌─────────────────┐      ┌──────────────────┐        │  │
│  │  │ DataEngineerUser│      │ DataAnalystUser  │        │  │
│  │  │                 │      │                  │        │  │
│  │  │ Permissions:    │      │ Permissions:     │        │  │
│  │  │ - S3 ReadWrite  │      │ - S3 ReadOnly    │        │  │
│  │  │ - Glue Full     │      │ - Athena Query   │        │  │
│  │  │ - Athena Query  │      │ - Glue ReadOnly  │        │  │
│  │  └─────────────────┘      └──────────────────┘        │  │
│  │           ↓                         ↓                   │  │
│  │    [MFA Enabled]            [MFA Enabled]              │  │
│  └────────────────────────────────────────────────────────┘  │
│                          ↓                                    │
│           ┌──────────────────────────────┐                   │
│           │    DATA LAKE (S3 Bucket)     │                   │
│           └──────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────┘
```

### Step-by-Step Lab Guide

#### **Step 1: Sign in to AWS Console**

**Navigation Path:**  
AWS Console → Sign in → Enter Root or IAM User Credentials

**Expected Result:**  
You should see the AWS Management Console homepage with a search bar at the top and various service icons.

---

#### **Step 2: Navigate to IAM Service**

**Navigation Path:**  
AWS Console → Search Bar (top) → Type "IAM" → Click "IAM"

**Expected Result:**  
You'll see the IAM Dashboard showing:
- Security recommendations
- Number of users, groups, roles, policies
- Access Advisor section

**Screenshot Description:**  
The IAM dashboard has a left sidebar with Users, User groups, Roles, Policies, Identity providers, and Account settings.

---

#### **Step 3: Create First IAM User (Data Engineer)**

**Navigation Path:**  
IAM Dashboard → Users (left sidebar) → Create user (button)

**Expected Result:**  
You should see a "Create user" page with a form.

**Detailed Steps:**

**3.1 Specify User Details**

| Field | Value |
|-------|-------|
| User name | `data-engineer-john` |
| Provide user access to AWS Management Console | ☑ Checked |
| Console password | Select "Custom password" |
| Custom password | Create a strong password (min 8 chars) |
| Users must create a new password at next sign-in | ☑ Checked (recommended) |

Click **Next**

---

**3.2 Set Permissions**

**Navigation Path:**  
Create user → Set permissions

Select: **Attach policies directly**

**Search and Select These Policies:**

| Policy Name | Description |
|------------|-------------|
| `AmazonS3FullAccess` | Full access to S3 buckets |
| `AWSGlueConsoleFullAccess` | Full access to Glue services |
| `AmazonAthenaFullAccess` | Full access to Athena |
| `CloudWatchLogsReadOnlyAccess` | View logs for debugging |

**Expected Result:**  
You should see 4 policies selected with checkmarks.

Click **Next**

---

**3.3 Review and Create**

Review the user details:
- User name: data-engineer-john
- Policies: 4 attached
- Console access: Enabled

Click **Create user**

**Expected Result:**  
Success message: "User data-engineer-john created successfully"

**IMPORTANT:** You'll see:
- Console sign-in URL (save this)
- User name
- Console password (download .csv file)

Click **Download .csv file** and store securely.

---

#### **Step 4: Create Second IAM User (Data Analyst)**

**Navigation Path:**  
IAM → Users → Create user

**4.1 User Details**

| Field | Value |
|-------|-------|
| User name | `data-analyst-sarah` |
| Console access | ☑ Enabled |
| Password | Custom password |
| Require password reset | ☑ Checked |

Click **Next**

---

**4.2 Set Permissions (Restricted Access)**

Select: **Attach policies directly**

**Search and Select:**

| Policy Name | Why? |
|------------|------|
| `AmazonS3ReadOnlyAccess` | Can view but not modify S3 data |
| `AmazonAthenaFullAccess` | Can query data using Athena |
| `AWSGlueConsoleReadOnlyAccess` | Can view Glue catalog metadata |

Click **Next** → **Create user**

Download credentials .csv file.

---

#### **Step 5: Enable MFA for Users**

**Navigation Path:**  
IAM → Users → Click "data-engineer-john" → Security credentials tab → Assign MFA device

**Expected Result:**  
You should see "Multi-factor authentication (MFA)" section with "Assign MFA device" button.

Click **Assign MFA device**

**5.1 Configure MFA**

| Field | Value |
|-------|-------|
| Device name | `john-mfa-device` |
| MFA device | Select "Authenticator app" |

Click **Next**

**5.2 Set up Device**

1. Download Google Authenticator or Microsoft Authenticator on your phone
2. Scan the QR code shown on screen
3. Enter two consecutive MFA codes from the app
4. Click **Add MFA**

**Expected Result:**  
Success message: "MFA device assigned successfully"

**Repeat for data-analyst-sarah**

---

### Validation Steps

#### **Test 1: Verify User Login**

1. Open incognito/private browser window
2. Navigate to the sign-in URL (from the .csv file)
   - Format: `https://<account-id>.signin.aws.amazon.com/console`
3. Enter credentials:
   - Username: data-engineer-john
   - Password: (from .csv)
4. Enter MFA code
5. Set new password when prompted

**Expected Result:**  
You should successfully log in to AWS Console as data-engineer-john

---

#### **Test 2: Verify Permissions**

**Test Data Engineer Permissions:**

1. While logged in as data-engineer-john
2. Navigate to: S3 → Create bucket → Name: `test-de-bucket-<random-number>`
3. Click **Create bucket**

**Expected Result:** ✅ Bucket created successfully (has S3 write permission)

4. Navigate to: IAM → Users
5. Try to create a new user

**Expected Result:** ❌ Access denied (does not have IAM permissions) - This is correct!

---

**Test Data Analyst Permissions:**

1. Log in as data-analyst-sarah
2. Navigate to S3
3. Try to create a bucket

**Expected Result:** ❌ Access denied (read-only access)

4. Navigate to existing bucket (from data engineer test)
5. Click on bucket → View objects

**Expected Result:** ✅ Can view contents (has read permission)

---

### Common Errors and Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| "Access Denied" when creating user | Using IAM user without admin rights | Use root account or admin IAM user |
| Cannot see MFA setup option | Browser cache issue | Clear cache or use incognito mode |
| Policy not appearing in search | Typing exact policy name | Use partial search like "S3" or "Glue" |
| User cannot login after creation | Forgot to download/save password | Reset password via IAM console |
| MFA codes not working | Time sync issue on phone | Check phone time settings (use auto) |

---

### CLI Version

```bash
# Set AWS credentials
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"

# Create Data Engineer User
aws iam create-user --user-name data-engineer-john

# Create login profile (console password)
aws iam create-login-profile \
    --user-name data-engineer-john \
    --password "TempPassword123!" \
    --password-reset-required

# Attach policies to Data Engineer
aws iam attach-user-policy \
    --user-name data-engineer-john \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-user-policy \
    --user-name data-engineer-john \
    --policy-arn arn:aws:iam::aws:policy/AWSGlueConsoleFullAccess

aws iam attach-user-policy \
    --user-name data-engineer-john \
    --policy-arn arn:aws:iam::aws:policy/AmazonAthenaFullAccess

# Create Data Analyst User
aws iam create-user --user-name data-analyst-sarah

aws iam create-login-profile \
    --user-name data-analyst-sarah \
    --password "TempPassword456!" \
    --password-reset-required

# Attach policies to Data Analyst
aws iam attach-user-policy \
    --user-name data-analyst-sarah \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam attach-user-policy \
    --user-name data-analyst-sarah \
    --policy-arn arn:aws:iam::aws:policy/AmazonAthenaFullAccess

# List users to verify
aws iam list-users

# List policies attached to a user
aws iam list-attached-user-policies --user-name data-engineer-john
```

---

### Interview Preparation

#### **Beginner Questions**

**Q1: What is IAM and why is it important?**

**Answer:**  
IAM (Identity and Access Management) is AWS's service for managing access to AWS resources securely. It's important because:
- Controls **who** can access AWS resources (authentication)
- Controls **what** they can do (authorization)
- Provides audit trails for compliance
- Enables principle of least privilege
- Works across all AWS services

**Oracle DBA Analogy:** IAM is like Oracle's user management + grant system + audit vault combined into one service.

---

**Q2: What's the difference between IAM Users, Groups, and Roles?**

**Answer:**

| Concept | Purpose | Example |
|---------|---------|---------|
| **User** | Represents a person or application | john.doe@company.com |
| **Group** | Collection of users with same permissions | DataEngineers, DataAnalysts |
| **Role** | Temporary credentials for services/users | GlueJobExecutionRole |

**Key Difference:** Users have permanent credentials; Roles have temporary credentials (recommended for applications).

---

**Q3: What are IAM Policies?**

**Answer:**  
IAM Policies are JSON documents that define permissions. They specify:
- **Effect:** Allow or Deny
- **Action:** What can be done (s3:GetObject)
- **Resource:** Which resource (specific S3 bucket)
- **Condition:** When it applies (IP address, time)

Example:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

---

**Q4: What is the principle of least privilege?**

**Answer:**  
Granting only the minimum permissions necessary to perform a task. 

**Example:**  
- ❌ Bad: Give data analyst full S3 access (can delete production data)
- ✅ Good: Give data analyst read-only S3 access to specific buckets

**Oracle Comparison:** Like granting SELECT on specific tables instead of DBA role.

---

**Q5: What is MFA and why should you enable it?**

**Answer:**  
Multi-Factor Authentication (MFA) requires two forms of identification:
1. Something you know (password)
2. Something you have (phone with authenticator app)

**Why Enable:**
- Prevents unauthorized access even if password is compromised
- Required for compliance (SOC 2, HIPAA, PCI-DSS)
- AWS best practice for all users with console access

---

#### **Intermediate Questions**

**Q1: Explain the difference between AWS Managed Policies and Customer Managed Policies.**

**Answer:**

| Feature | AWS Managed | Customer Managed |
|---------|-------------|------------------|
| **Created by** | AWS | You |
| **Updated by** | AWS (automatic) | You (manual) |
| **Customization** | No | Full control |
| **Examples** | AmazonS3ReadOnlyAccess | CustomDataEngineerPolicy |
| **Best for** | Common use cases | Specific business needs |

**When to use AWS Managed:**  
- Standard permissions (read-only S3 access)
- Quick setup
- Benefit from AWS security updates

**When to create Customer Managed:**  
- Need specific resource restrictions
- Company-specific compliance requirements
- Fine-grained control (only specific buckets/prefixes)

---

**Q2: What are IAM Roles and when should you use them instead of IAM Users?**

**Answer:**

**IAM Roles are for:**
1. **AWS Services** (Glue jobs, Lambda functions, EC2 instances)
2. **Cross-account access**
3. **Federated users** (Active Directory, Google Workspace)
4. **Temporary access scenarios**

**Why Roles are better than Users for applications:**

| IAM Users | IAM Roles |
|-----------|-----------|
| Permanent credentials | Temporary credentials |
| Must rotate access keys manually | Auto-rotated by AWS |
| Can be leaked/hardcoded | Cannot be hardcoded |
| One person/app | Can be assumed by many |

**Example:**  
❌ Bad: Create IAM user for Glue job, store access key in code  
✅ Good: Create IAM role for Glue job, Glue automatically uses temporary credentials

---

**Q3: How do you troubleshoot "Access Denied" errors in AWS?**

**Answer:**

**Systematic Approach:**

1. **Check IAM Policies**
   ```bash
   aws iam list-attached-user-policies --user-name john
   aws iam get-policy-version --policy-arn <arn> --version-id v1
   ```

2. **Check Resource-Based Policies** (S3 bucket policies, KMS key policies)
   ```bash
   aws s3api get-bucket-policy --bucket my-bucket
   ```

3. **Check Service Control Policies (SCPs)** - Organization level restrictions

4. **Check Permission Boundaries** - Max permissions a user can have

5. **Use IAM Policy Simulator**
   - AWS Console → IAM → Policy Simulator
   - Test whether a specific action is allowed

6. **Check CloudTrail Logs**
   ```bash
   aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutObject
   ```

**Common Causes:**
- Missing s3:ListBucket permission (different from s3:GetObject)
- Explicit Deny in any policy (overrides all Allows)
- Resource ARN mismatch
- Missing KMS permissions for encrypted objects

---

**Q4: Explain IAM permission evaluation logic.**

**Answer:**

**Decision Flow:**
```
1. By default, ALL requests are DENIED
2. Explicit ALLOW in identity-based policy → Allowed (unless step 3)
3. Explicit DENY anywhere (SCP, permission boundary, resource policy) → DENIED
4. Final decision: DENIED (if no explicit allow found)
```

**Key Rules:**
- **Explicit Deny always wins**
- An Allow in one policy can't override a Deny in another
- Identity-based + Resource-based permissions are unioned (additive)

**Example:**
- User policy: Allow s3:PutObject on bucket-A
- Bucket policy: Deny s3:PutObject for user
- **Result:** DENIED (Deny wins)

---

**Q5: How do you implement least privilege for a data engineering pipeline?**

**Answer:**

**Step-by-Step Approach:**

1. **Identify Required Actions**
   - List all operations (read S3, write to Glue Catalog, query Athena)

2. **Identify Resources**
   - Specific buckets: `arn:aws:s3:::prod-data-lake/*`
   - Specific databases: `arn:aws:glue:us-east-1:123456789012:database/analytics_db`

3. **Create Granular Policy**
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "s3:GetObject",
           "s3:ListBucket"
         ],
         "Resource": [
           "arn:aws:s3:::prod-data-lake",
           "arn:aws:s3:::prod-data-lake/*"
         ]
       },
       {
         "Effect": "Allow",
         "Action": [
           "glue:GetTable",
           "glue:GetDatabase"
         ],
         "Resource": "arn:aws:glue:us-east-1:123456789012:database/analytics_db"
       }
     ]
   }
   ```

4. **Test in Non-Prod**
   - Deploy to dev environment
   - Run end-to-end pipeline
   - Check CloudTrail for access denied events

5. **Monitor and Refine**
   - Use IAM Access Analyzer
   - Review CloudTrail logs monthly
   - Remove unused permissions

---

#### **Scenario-Based Questions**

**Scenario 1: A data engineer needs to access S3 buckets in multiple AWS accounts. How would you design this?**

**Answer:**

**Solution: Cross-Account IAM Roles**

**Architecture:**
```
Account A (Data Lake)          Account B (Engineering)
┌──────────────┐              ┌──────────────┐
│  S3 Bucket   │              │  IAM User:   │
│  prod-data   │              │  john.doe    │
│              │              │              │
│  IAM Role:   │←── Assume ───│  Policy:     │
│  DataAccess  │              │  AssumeRole  │
│              │              │  Permission  │
│  Policy:     │              └──────────────┘
│  S3 Read     │
└──────────────┘
```

**Step 1: Create Role in Account A (Data Lake Account)**
```bash
# Account A: 111111111111
aws iam create-role \
    --role-name CrossAccountDataAccess \
    --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::222222222222:user/john.doe"
        },
        "Action": "sts:AssumeRole"
      }]
    }'

# Attach S3 read policy
aws iam attach-role-policy \
    --role-name CrossAccountDataAccess \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**Step 2: Grant Assume Role Permission in Account B**
```bash
# Account B: 222222222222
aws iam put-user-policy \
    --user-name john.doe \
    --policy-name AssumeDataLakeRole \
    --policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "arn:aws:iam::111111111111:role/CrossAccountDataAccess"
      }]
    }'
```

**Step 3: Assume Role**
```bash
aws sts assume-role \
    --role-arn arn:aws:iam::111111111111:role/CrossAccountDataAccess \
    --role-session-name john-session

# Use returned temporary credentials
export AWS_ACCESS_KEY_ID="<from-assume-role-output>"
export AWS_SECRET_ACCESS_KEY="<from-assume-role-output>"
export AWS_SESSION_TOKEN="<from-assume-role-output>"

# Now can access Account A's S3
aws s3 ls s3://prod-data/
```

**Benefits:**
- No credential sharing
- Temporary credentials (auto-expire)
- Centralized permission management
- CloudTrail audit trail shows original user

---

**Scenario 2: Your Glue ETL job is failing with "Access Denied" when writing to S3. How do you troubleshoot?**

**Answer:**

**Troubleshooting Checklist:**

**Step 1: Verify Glue Job Role**
```bash
# Get role attached to Glue job
aws glue get-job --job-name my-etl-job --query 'Job.Role'

# Output: arn:aws:iam::123456789012:role/GlueServiceRole
```

**Step 2: Check Role Permissions**
```bash
# List attached policies
aws iam list-attached-role-policies --role-name GlueServiceRole

# Get policy document
aws iam get-policy-version \
    --policy-arn <policy-arn> \
    --version-id v1
```

**Step 3: Check S3 Bucket Policy**
```bash
aws s3api get-bucket-policy --bucket target-bucket
```

**Step 4: Check S3 Bucket Encryption**
- If bucket uses SSE-KMS encryption, Glue role needs KMS permissions

**Step 5: Check CloudTrail**
```bash
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::S3::Bucket \
    --max-results 10
```

**Common Issues:**

| Issue | Symptom | Fix |
|-------|---------|-----|
| Missing s3:PutObject | "Access Denied" on write | Add to role policy |
| Missing kms:Decrypt | "KMS access denied" | Add KMS permissions |
| Bucket policy blocks role | Even with role policy, denied | Update bucket policy to allow role |
| Wrong bucket ARN | Works for some prefixes, not others | Check ARN format: `arn:aws:s3:::bucket/*` |

**Complete Fix - Glue Role Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::target-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::target-bucket"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab"
    }
  ]
}
```

---

**Scenario 3: Design IAM structure for a data platform with dev, staging, and prod environments.**

**Answer:**

**Multi-Environment IAM Architecture:**

```
Organization: DataPlatform
├── Account 1: Dev (111111111111)
│   ├── IAM Roles:
│   │   ├── DataEngineerRole (full access)
│   │   ├── GlueDevRole
│   │   └── AthenaDevRole
│   └── Resources:
│       ├── S3: dev-data-lake
│       └── Glue: dev_catalog
│
├── Account 2: Staging (222222222222)
│   ├── IAM Roles:
│   │   ├── DataEngineerRole (read + limited write)
│   │   ├── GlueStagingRole
│   │   └── AthenaStagingRole
│   └── Resources:
│       ├── S3: staging-data-lake
│       └── Glue: staging_catalog
│
└── Account 3: Prod (333333333333)
    ├── IAM Roles:
    │   ├── DataEngineerRole (read-only)
    │   ├── GlueProdRole
    │   ├── AthenaProdRole
    │   └── DataAnalystRole (read-only)
    └── Resources:
        ├── S3: prod-data-lake
        └── Glue: prod_catalog
```

**IAM Design Principles:**

1. **Separate AWS Accounts per Environment**
   - Blast radius containment
   - Different billing/cost allocation
   - Compliance isolation

2. **Environment-Specific Permissions**

| Environment | Data Engineer Access | Data Analyst Access |
|-------------|---------------------|---------------------|
| Dev | Full (create/delete) | None |
| Staging | Read + Write (no delete) | Read-only |
| Prod | Read-only | Read-only (via Athena) |

3. **Cross-Account Access Pattern**
```bash
# Dev environment - full access
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::dev-data-lake/*"
}

# Staging - controlled access
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::staging-data-lake/*",
  "Condition": {
    "StringEquals": {
      "aws:RequestedRegion": "us-east-1"
    }
  }
}

# Prod - read-only
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:ListBucket"],
  "Resource": [
    "arn:aws:s3:::prod-data-lake",
    "arn:aws:s3:::prod-data-lake/*"
  ]
}
```

4. **Role Assumption Flow**
```
Engineer's Identity (SSO/AD)
         ↓
   Assume Role
         ↓
DevDataEngineerRole → Can assume StagingReadRole
         ↓
StagingReadRole → Can assume ProdReadOnlyRole
```

**Implementation:**
```bash
# Create cross-environment access role
aws iam create-role \
    --role-name DevToStagingAccess \
    --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::111111111111:role/DevDataEngineerRole"
        },
        "Action": "sts:AssumeRole",
        "Condition": {
          "StringEquals": {
            "sts:ExternalId": "secure-external-id-12345"
          }
        }
      }]
    }'
```

**Benefits:**
- Environment isolation
- Audit trail per environment
- Easy to enforce promotion workflow
- Compliance requirements met (segregation of duties)

---

**Scenario 4: How would you secure access to sensitive PII data in S3 for a GDPR-compliant data lake?**

**Answer:**

**Multi-Layered Security Approach:**

**Layer 1: Bucket-Level Encryption**
```bash
# Enable default encryption with KMS
aws s3api put-bucket-encryption \
    --bucket sensitive-pii-data \
    --server-side-encryption-configuration '{
      "Rules": [{
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab"
        },
        "BucketKeyEnabled": true
      }]
    }'
```

**Layer 2: IAM Policies with Conditions**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPIIAccessOnlyFromVPN",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::sensitive-pii-data/pii/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": [
            "10.0.0.0/16"
          ]
        },
        "StringEquals": {
          "aws:RequestedRegion": "eu-west-1"
        },
        "Bool": {
          "aws:SecureTransport": "true"
        }
      }
    }
  ]
}
```

**Layer 3: S3 Bucket Policies**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::sensitive-pii-data/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::sensitive-pii-data/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**Layer 4: KMS Key Policy for PII**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPIIDataEngineersOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::123456789012:role/PIIDataEngineerRole",
          "arn:aws:iam::123456789012:role/ComplianceOfficerRole"
        ]
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.eu-west-1.amazonaws.com"
        }
      }
    }
  ]
}
```

**Layer 5: S3 Access Logging**
```bash
# Enable S3 access logging
aws s3api put-bucket-logging \
    --bucket sensitive-pii-data \
    --bucket-logging-status '{
      "LoggingEnabled": {
        "TargetBucket": "audit-logs-bucket",
        "TargetPrefix": "pii-access-logs/"
      }
    }'
```

**Layer 6: S3 Object Lock (WORM)**
```bash
# Enable versioning (prerequisite)
aws s3api put-bucket-versioning \
    --bucket sensitive-pii-data \
    --versioning-configuration Status=Enabled

# Enable object lock
aws s3api put-object-lock-configuration \
    --bucket sensitive-pii-data \
    --object-lock-configuration '{
      "ObjectLockEnabled": "Enabled",
      "Rule": {
        "DefaultRetention": {
          "Mode": "GOVERNANCE",
          "Days": 2555
        }
      }
    }'
```

**Layer 7: S3 Inventory for Audit**
```bash
aws s3api put-bucket-inventory-configuration \
    --bucket sensitive-pii-data \
    --id pii-inventory \
    --inventory-configuration '{
      "Destination": {
        "S3BucketDestination": {
          "Bucket": "arn:aws:s3:::audit-logs-bucket",
          "Format": "CSV",
          "Prefix": "pii-inventory"
        }
      },
      "IsEnabled": true,
      "Id": "pii-inventory",
      "IncludedObjectVersions": "Current",
      "OptionalFields": [
        "Size",
        "LastModifiedDate",
        "EncryptionStatus"
      ],
      "Schedule": {
        "Frequency": "Daily"
      }
    }'
```

**GDPR-Specific Requirements:**

1. **Right to Access** - Audit logs prove who accessed what
2. **Right to Erasure** - Object versioning + lifecycle policies for deletion
3. **Data Minimization** - IAM policies restrict access to minimum required
4. **Encryption** - Data encrypted at rest and in transit
5. **Audit Trail** - CloudTrail + S3 access logs + VPC Flow Logs

**Monitoring Setup:**
```bash
# CloudWatch alarm for unauthorized access attempts
aws cloudwatch put-metric-alarm \
    --alarm-name pii-unauthorized-access \
    --alarm-description "Alert on PII bucket access denials" \
    --metric-name AccessDeniedErrors \
    --namespace AWS/S3 \
    --statistic Sum \
    --period 300 \
    --evaluation-periods 1 \
    --threshold 5 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=BucketName,Value=sensitive-pii-data
```

---

**Scenario 5: Your company is migrating from on-premises Oracle databases to AWS. How do you manage IAM for database migration tools (DMS) and data engineers?**

**Answer:**

**Migration Architecture with IAM:**

```
┌──────────────────────────────────────────────────────────────┐
│                    ON-PREMISES                                │
│  ┌────────────────────────────────────────────────┐          │
│  │         Oracle Database                         │          │
│  │         (Source)                                │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│              VPN / Direct Connect                             │
└──────────────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────────┐
│                    AWS CLOUD                                  │
│  ┌────────────────────────────────────────────────┐          │
│  │         AWS DMS Replication Instance            │          │
│  │         IAM Role: DMSServiceRole                │          │
│  │         Permissions:                            │          │
│  │         - VPC access                            │          │
│  │         - S3 write (for intermediate storage)   │          │
│  │         - CloudWatch Logs                       │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  ┌────────────────────────────────────────────────┐          │
│  │         S3 Data Lake (Landing Zone)             │          │
│  │         Bucket: migration-landing-zone          │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  ┌────────────────────────────────────────────────┐          │
│  │         AWS Glue ETL Jobs                       │          │
│  │         IAM Role: GlueMigrationRole             │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  ┌────────────────────────────────────────────────┐          │
│  │         Amazon RDS / Aurora (PostgreSQL)        │          │
│  │         Target Database                         │          │
│  └────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

**Step 1: Create DMS Service Roles**

```bash
# Create DMS VPC Management Role
aws iam create-role \
    --role-name dms-vpc-role \
    --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "Service": "dms.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }]
    }'

aws iam attach-role-policy \
    --role-name dms-vpc-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSVPCManagementRole

# Create DMS CloudWatch Logs Role
aws iam create-role \
    --role-name dms-cloudwatch-logs-role \
    --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {
          "Service": "dms.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }]
    }'

aws iam attach-role-policy \
    --role-name dms-cloudwatch-logs-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonDMSCloudWatchLogsRole
```

**Step 2: Create DMS Task Execution Role**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DMSAccessToS3",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::migration-landing-zone",
        "arn:aws:s3:::migration-landing-zone/*"
      ]
    },
    {
      "Sid": "DMSAccessToSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:oracle-db-credentials-*"
    },
    {
      "Sid": "DMSAccessToKMS",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/*"
    }
  ]
}
```

**Step 3: Create Data Engineer Migration Role**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ManageDMSTasks",
      "Effect": "Allow",
      "Action": [
        "dms:CreateReplicationTask",
        "dms:StartReplicationTask",
        "dms:StopReplicationTask",
        "dms:DeleteReplicationTask",
        "dms:DescribeReplicationTasks",
        "dms:TestConnection"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AccessMigrationS3Data",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::migration-landing-zone",
        "arn:aws:s3:::migration-landing-zone/*"
      ]
    },
    {
      "Sid": "RunGlueValidationJobs",
      "Effect": "Allow",
      "Action": [
        "glue:StartJobRun",
        "glue:GetJobRun"
      ],
      "Resource": "arn:aws:glue:us-east-1:123456789012:job/migration-*"
    },
    {
      "Sid": "ViewCloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:GetLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/dms/*"
    }
  ]
}
```

**Step 4: Secrets Manager for Database Credentials**

```bash
# Store Oracle source credentials
aws secretsmanager create-secret \
    --name oracle-source-db-credentials \
    --description "Oracle source database credentials for DMS" \
    --secret-string '{
      "username":"ORACLE_USER",
      "password":"SecurePassword123!",
      "engine":"oracle",
      "host":"on-prem-oracle.company.com",
      "port":1521,
      "dbname":"PRODDB"
    }'

# Store target RDS credentials
aws secretsmanager create-secret \
    --name rds-target-db-credentials \
    --secret-string '{
      "username":"postgres",
      "password":"SecurePassword456!",
      "engine":"postgres",
      "host":"myinstance.123456789012.us-east-1.rds.amazonaws.com",
      "port":5432,
      "dbname":"analytics"
    }'
```

**Step 5: Create DMS Endpoints with IAM Authentication**

```bash
# Create source endpoint (Oracle)
aws dms create-endpoint \
    --endpoint-identifier oracle-source \
    --endpoint-type source \
    --engine-name oracle \
    --username ORACLE_USER \
    --password SecurePassword123! \
    --server-name on-prem-oracle.company.com \
    --port 1521 \
    --database-name PRODDB

# Create target endpoint (S3)
aws dms create-endpoint \
    --endpoint-identifier s3-target \
    --endpoint-type target \
    --engine-name s3 \
    --s3-settings '{
      "ServiceAccessRoleArn": "arn:aws:iam::123456789012:role/dms-s3-access-role",
      "BucketName": "migration-landing-zone",
      "BucketFolder": "oracle-migration",
      "DataFormat": "parquet",
      "EnableStatistics": true,
      "IncludeOpForFullLoad": true,
      "TimestampColumnName": "dms_timestamp"
    }'
```

**Best Practices for Oracle DBA Transitioning to AWS:**

| Oracle Concept | AWS Equivalent | IAM Consideration |
|---------------|----------------|-------------------|
| DBA role | Admin IAM user/role | Use roles, not users |
| System privileges | IAM policies | Granular, service-specific |
| Object privileges | Resource policies | S3 bucket policies, etc. |
| Database links | VPC peering, PrivateLink | Security groups + NACLs |
| Auditing (audit_trail) | CloudTrail + CloudWatch | Always enabled, encrypted |
| Password policies | IAM password policy | MFA required |
| Profiles | No direct equivalent | Use IAM policies + conditions |

**Migration Checklist:**

- ✅ VPN/Direct Connect established
- ✅ DMS service roles created
- ✅ Source/target endpoints tested
- ✅ Replication instance running in private subnet
- ✅ S3 bucket with versioning enabled
- ✅ CloudWatch alarms for DMS task failures
- ✅ Glue Data Quality rules for validation
- ✅ Data engineers have read-only prod access
- ✅ All credentials in Secrets Manager (not hardcoded)
- ✅ Encryption in transit (SSL) and at rest (KMS)

---

**Scenario 6: Implement a data access request workflow where analysts must request time-limited access to production data.**

**Answer:**

**Just-In-Time (JIT) Access Architecture:**

```
┌──────────────────────────────────────────────────────────────┐
│                  ACCESS REQUEST WORKFLOW                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Step 1: Analyst Submits Request                             │
│  ┌────────────────────────────────────────────────┐          │
│  │  ServiceNow / JIRA Ticket                      │          │
│  │  - Requestor: sarah.analyst@company.com       │          │
│  │  - Resource: s3://prod-data-lake/sales/*      │          │
│  │  - Duration: 4 hours                           │          │
│  │  - Business Justification: Q4 Revenue Analysis │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Step 2: Manager Approval                                     │
│  ┌────────────────────────────────────────────────┐          │
│  │  Email/Slack Notification                      │          │
│  │  Manager clicks "Approve" link                 │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Step 3: Lambda Grants Temporary Access                       │
│  ┌────────────────────────────────────────────────┐          │
│  │  AWS Lambda Function                           │          │
│  │  - Creates IAM policy                          │          │
│  │  - Attaches to analyst's role                  │          │
│  │  - Sets expiration tag                         │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Step 4: Analyst Accesses Data (time-limited)                │
│  ┌────────────────────────────────────────────────┐          │
│  │  Athena Query / Glue Job                       │          │
│  │  Access Valid: 4 hours                         │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Step 5: Automatic Revocation                                │
│  ┌────────────────────────────────────────────────┐          │
│  │  EventBridge Scheduled Rule                    │          │
│  │  - Runs every hour                             │          │
│  │  - Removes expired policies                    │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Step 6: Audit Logging                                       │
│  ┌────────────────────────────────────────────────┐          │
│  │  CloudTrail + S3 Access Logs                   │          │
│  │  - Who accessed what, when                     │          │
│  │  - Stored in immutable audit bucket            │          │
│  └────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

**Implementation:**

**Lambda Function: Grant Access**
```python
import boto3
import json
from datetime import datetime, timedelta

iam = boto3.client('iam')
sns = boto3.client('sns')

def lambda_handler(event, context):
    # Parse request
    analyst_role = event['analyst_role']
    s3_resource = event['s3_resource']
    duration_hours = event['duration_hours']
    justification = event['justification']
    
    # Calculate expiration
    expiration = datetime.utcnow() + timedelta(hours=duration_hours)
    
    # Create temporary policy
    policy_name = f"TempAccess-{analyst_role}-{datetime.utcnow().strftime('%Y%m%d%H%M%S')}"
    
    policy_document = {
        "Version": "2012-10-17",
        "Statement": [{
            "Sid": "TemporaryS3Access",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                f"arn:aws:s3:::{s3_resource}",
                f"arn:aws:s3:::{s3_resource}/*"
            ],
            "Condition": {
                "DateLessThan": {
                    "aws:CurrentTime": expiration.isoformat()
                }
            }
        }]
    }
    
    # Create and attach policy
    response = iam.create_policy(
        PolicyName=policy_name,
        PolicyDocument=json.dumps(policy_document),
        Description=f"Temporary access - Expires: {expiration.isoformat()}"
    )
    
    policy_arn = response['Policy']['Arn']
    
    iam.attach_role_policy(
        RoleName=analyst_role,
        PolicyArn=policy_arn
    )
    
    # Tag policy with expiration
    iam.tag_policy(
        PolicyArn=policy_arn,
        Tags=[
            {'Key': 'ExpiresAt', 'Value': expiration.isoformat()},
            {'Key': 'Justification', 'Value': justification},
            {'Key': 'AutoRevoke', 'Value': 'true'}
        ]
    )
    
    # Send notification
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:access-granted',
        Subject='Temporary Production Access Granted',
        Message=f"""
        Analyst: {analyst_role}
        Resource: {s3_resource}
        Duration: {duration_hours} hours
        Expires: {expiration.isoformat()}
        Justification: {justification}
        """
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'policy_arn': policy_arn,
            'expires_at': expiration.isoformat()
        })
    }
```

**Lambda Function: Revoke Expired Access**
```python
import boto3
from datetime import datetime

iam = boto3.client('iam')

def lambda_handler(event, context):
    # List all policies with AutoRevoke tag
    paginator = iam.get_paginator('list_policies')
    
    for page in paginator.paginate(Scope='Local'):
        for policy in page['Policies']:
            policy_arn = policy['Arn']
            
            # Get tags
            try:
                tags_response = iam.list_policy_tags(PolicyArn=policy_arn)
                tags = {tag['Key']: tag['Value'] for tag in tags_response['Tags']}
                
                if tags.get('AutoRevoke') == 'true':
                    expires_at = datetime.fromisoformat(tags['ExpiresAt'])
                    
                    # Check if expired
                    if datetime.utcnow() > expires_at:
                        # Get attached entities
                        entities = iam.list_entities_for_policy(PolicyArn=policy_arn)
                        
                        # Detach from roles
                        for role in entities['PolicyRoles']:
                            iam.detach_role_policy(
                                RoleName=role['RoleName'],
                                PolicyArn=policy_arn
                            )
                        
                        # Delete policy
                        # First delete all versions except default
                        versions = iam.list_policy_versions(PolicyArn=policy_arn)
                        for version in versions['Versions']:
                            if not version['IsDefaultVersion']:
                                iam.delete_policy_version(
                                    PolicyArn=policy_arn,
                                    VersionId=version['VersionId']
                                )
                        
                        # Delete policy
                        iam.delete_policy(PolicyArn=policy_arn)
                        
                        print(f"Revoked expired policy: {policy_arn}")
            except Exception as e:
                print(f"Error processing policy {policy_arn}: {str(e)}")
    
    return {'statusCode': 200, 'body': 'Cleanup complete'}
```

**EventBridge Rule:**
```bash
# Create scheduled rule to run hourly
aws events put-rule \
    --name revoke-expired-access \
    --description "Revoke expired temporary IAM policies" \
    --schedule-expression "rate(1 hour)"

# Add Lambda as target
aws events put-targets \
    --rule revoke-expired-access \
    --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:RevokeExpiredAccess"
```

**Benefits:**
- ✅ Audit trail of all access requests
- ✅ Manager approval required
- ✅ Time-limited access (auto-expires)
- ✅ Principle of least privilege
- ✅ Compliance with SOX, GDPR, HIPAA

**Oracle DBA Analogy:**
- Similar to Oracle's `DBMS_RESOURCE_MANAGER` for session limits
- Like temporary profile with `CONNECT_TIME` limit
- Equivalent to `EXECUTE IMMEDIATE` with `DBMS_SCHEDULER` for auto-revocation

---

**Scenario 7: Your organization has multiple teams using the same AWS account. How do you isolate data access between teams?**

**Answer:**

**Tag-Based Access Control (ABAC) Strategy:**

```
┌──────────────────────────────────────────────────────────────┐
│                   MULTI-TEAM ARCHITECTURE                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Team 1: Marketing Analytics                                  │
│  ┌────────────────────────────────────────────────┐          │
│  │  IAM Role: MarketingDataEngineerRole           │          │
│  │  Tags:                                         │          │
│  │    Department: Marketing                       │          │
│  │    DataClass: Public                           │          │
│  │                                                │          │
│  │  Can Access:                                   │          │
│  │  - s3://data-lake/marketing/* (Tag: Dept=Mkt) │          │
│  │  - Glue DB: marketing_analytics                │          │
│  └────────────────────────────────────────────────┘          │
│                                                               │
│  Team 2: Finance Analytics                                    │
│  ┌────────────────────────────────────────────────┐          │
│  │  IAM Role: FinanceDataEngineerRole             │          │
│  │  Tags:                                         │          │
│  │    Department: Finance                         │          │
│  │    DataClass: Confidential                     │          │
│  │                                                │          │
│  │  Can Access:                                   │          │
│  │  - s3://data-lake/finance/* (Tag: Dept=Fin)   │          │
│  │  - Glue DB: finance_analytics                  │          │
│  └────────────────────────────────────────────────┘          │
│                                                               │
│  Team 3: Shared Services (Can access both)                    │
│  ┌────────────────────────────────────────────────┐          │
│  │  IAM Role: DataPlatformAdminRole               │          │
│  │  Tags:                                         │          │
│  │    Department: DataPlatform                    │          │
│  │    AccessLevel: Admin                          │          │
│  └────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

**Implementation:**

**Step 1: Tag All Resources**

```bash
# Tag S3 buckets/prefixes
aws s3api put-object-tagging \
    --bucket data-lake \
    --key marketing/customer-data/ \
    --tagging 'TagSet=[{Key=Department,Value=Marketing},{Key=DataClass,Value=Public}]'

aws s3api put-object-tagging \
    --bucket data-lake \
    --key finance/revenue-data/ \
    --tagging 'TagSet=[{Key=Department,Value=Finance},{Key=DataClass,Value=Confidential}]'

# Tag Glue databases
aws glue tag-resource \
    --resource-arn arn:aws:glue:us-east-1:123456789012:database/marketing_analytics \
    --tags-to-add Department=Marketing,DataClass=Public

aws glue tag-resource \
    --resource-arn arn:aws:glue:us-east-1:123456789012:database/finance_analytics \
    --tags-to-add Department=Finance,DataClass=Confidential
```

**Step 2: Create IAM Policy Using Tag-Based Conditions**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccessToTeamData",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::data-lake/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/Department": "${aws:PrincipalTag/Department}"
        }
      }
    },
    {
      "Sid": "AllowGlueCatalogAccess",
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:GetPartitions"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Department": "${aws:PrincipalTag/Department}"
        }
      }
    },
    {
      "Sid": "AllowAthenaQueryTeamData",
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryResults"
      ],
      "Resource": "*",
      "Condition": {
        "ForAllValues:StringEquals": {
          "aws:ResourceTag/Department": "${aws:PrincipalTag/Department}"
        }
      }
    }
  ]
}
```

**Step 3: Tag IAM Roles with Team Membership**

```bash
# Tag Marketing role
aws iam tag-role \
    --role-name MarketingDataEngineerRole \
    --tags Key=Department,Value=Marketing Key=DataClass,Value=Public

# Tag Finance role
aws iam tag-role \
    --role-name FinanceDataEngineerRole \
    --tags Key=Department,Value=Finance Key=DataClass,Value=Confidential
```

**Step 4: Create Deny Policy for Cross-Team Access**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAccessToOtherTeamData",
      "Effect": "Deny",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::data-lake/*",
      "Condition": {
        "StringNotEquals": {
          "s3:ExistingObjectTag/Department": "${aws:PrincipalTag/Department}"
        }
      }
    }
  ]
}
```

**Step 5: Service Control Policy (SCP) for Organization**

If using AWS Organizations:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireTagsOnS3Objects",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::data-lake/*",
      "Condition": {
        "StringNotLike": {
          "s3:RequestObjectTag/Department": ["Marketing", "Finance", "DataPlatform"]
        }
      }
    }
  ]
}
```

**Validation:**

```bash
# Test as Marketing user
aws sts assume-role \
    --role-arn arn:aws:iam::123456789012:role/MarketingDataEngineerRole \
    --role-session-name marketing-test

# Should succeed
aws s3 ls s3://data-lake/marketing/

# Should fail (Access Denied)
aws s3 ls s3://data-lake/finance/
```

**Benefits:**
- ✅ Centralized tag management
- ✅ Automatic access based on team membership
- ✅ Easy to add new teams (just tag resources)
- ✅ Self-service data access
- ✅ Audit trail via CloudTrail (shows tag-based access)

---

**Scenario 8: How do you implement row-level and column-level security for Athena queries on S3 data?**

**Answer:**

**Lake Formation + IAM Architecture:**

```
┌──────────────────────────────────────────────────────────────┐
│              FINE-GRAINED ACCESS CONTROL                      │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Data in S3: s3://prod-data/customers/                        │
│  ┌────────────────────────────────────────────────┐          │
│  │  customer_id | name     | email    | ssn    | region     │
│  │  001         | John Doe | j@x.com  | 123-45 | US-EAST    │
│  │  002         | Jane Doe | jane@    | 678-90 | US-WEST    │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Lake Formation Permissions                                   │
│  ┌────────────────────────────────────────────────┐          │
│  │  User: marketing-analyst                       │          │
│  │  Table: customers                              │          │
│  │  Columns: customer_id, name, email, region     │          │
│  │  (NO ACCESS to ssn - PII)                      │          │
│  │                                                │          │
│  │  Row Filter: region = 'US-EAST'                │          │
│  │  (Can only see US-EAST customers)              │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Athena Query Result                                          │
│  ┌────────────────────────────────────────────────┐          │
│  │  customer_id | name     | email    | region    │          │
│  │  001         | John Doe | j@x.com  | US-EAST   │          │
│  │  (Only 1 row, ssn column hidden)               │          │
│  └────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

**Implementation:**

**Step 1: Enable Lake Formation**

```bash
# Register S3 location with Lake Formation
aws lakeformation register-resource \
    --resource-arn arn:aws:s3:::prod-data/customers/ \
    --use-service-linked-role

# Upgrade Glue catalog to Lake Formation
aws lakeformation put-data-lake-settings \
    --data-lake-settings '{
      "DataLakeAdmins": [{
        "DataLakePrincipalIdentifier": "arn:aws:iam::123456789012:role/DataLakeAdmin"
      }],
      "CreateDatabaseDefaultPermissions": [],
      "CreateTableDefaultPermissions": []
    }'
```

**Step 2: Revoke IAM Permissions (Use Lake Formation Instead)**

```bash
# Remove IAM-based S3 access
aws lakeformation revoke-permissions \
    --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/MarketingAnalystRole \
    --resource '{"Table":{"DatabaseName":"customer_db","Name":"customers"}}' \
    --permissions SELECT
```

**Step 3: Grant Column-Level Access**

```bash
# Grant access to specific columns only
aws lakeformation grant-permissions \
    --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/MarketingAnalystRole \
    --resource '{
      "TableWithColumns": {
        "DatabaseName": "customer_db",
        "Name": "customers",
        "ColumnNames": ["customer_id", "name", "email", "region"]
      }
    }' \
    --permissions SELECT
```

**Step 4: Apply Row-Level Filter**

```bash
# Create data filter
aws lakeformation create-data-cells-filter \
    --table-data '{
      "TableCatalogId": "123456789012",
      "DatabaseName": "customer_db",
      "TableName": "customers",
      "Name": "USEastFilter",
      "RowFilter": {
        "FilterExpression": "region = '\''US-EAST'\''"
      },
      "ColumnNames": ["customer_id", "name", "email", "region"]
    }'

# Grant filter to user
aws lakeformation grant-permissions \
    --principal DataLakePrincipalIdentifier=arn:aws:iam::123456789012:role/MarketingAnalystRole \
    --resource '{
      "DataCellsFilter": {
        "TableCatalogId": "123456789012",
        "DatabaseName": "customer_db",
        "TableName": "customers",
        "Name": "USEastFilter"
      }
    }' \
    --permissions SELECT
```

**Step 5: Implement Column Masking (PII Redaction)**

For sensitive columns like SSN, use Lake Formation data filters:

```bash
aws lakeformation create-data-cells-filter \
    --table-data '{
      "TableCatalogId": "123456789012",
      "DatabaseName": "customer_db",
      "TableName": "customers",
      "Name": "MaskSSN",
      "RowFilter": {
        "AllRowsWildcard": {}
      },
      "ColumnWildcard": {
        "ExcludedColumnNames": ["ssn"]
      }
    }'
```

**Validation:**

**Query 1: As Marketing Analyst**
```sql
SELECT * FROM customer_db.customers;

-- Result:
-- customer_id | name     | email    | region
-- 001         | John Doe | j@x.com  | US-EAST
-- (ssn column not visible, only US-EAST rows)
```

**Query 2: As Finance Analyst (has SSN access)**
```sql
SELECT * FROM customer_db.customers;

-- Result:
-- customer_id | name     | email    | ssn       | region
-- 001         | John Doe | j@x.com  | 123-45-67 | US-EAST
-- 002         | Jane Doe | jane@    | 678-90-12 | US-WEST
-- (Can see all columns and rows)
```

**Benefits:**
- ✅ Centralized access control (not scattered across IAM policies)
- ✅ Column-level security (hide PII)
- ✅ Row-level security (data residency requirements)
- ✅ Works across Athena, Redshift Spectrum, EMR
- ✅ Audit trail in Lake Formation

**Oracle DBA Comparison:**
- Column-level = Oracle Virtual Private Database (VPD)
- Row-level = Oracle Row-Level Security (RLS) / VPD
- Masking = Oracle Data Masking and Subsetting Pack

---

**Scenario 9: Your Glue job failed with "Access Denied" when trying to write Parquet files to S3. The bucket has default encryption enabled. What's wrong?**

**Answer:**

**Root Cause: Missing KMS Permissions**

When S3 bucket has KMS encryption enabled, the Glue job role needs:
1. S3 permissions (PutObject)
2. KMS permissions (Encrypt, Decrypt, GenerateDataKey)

**Troubleshooting Steps:**

**Step 1: Check Glue Job CloudWatch Logs**
```bash
aws logs filter-log-events \
    --log-group-name /aws-glue/jobs/error \
    --filter-pattern "Access Denied"
```

**Common Error:**
```
User: arn:aws:sts::123456789012:assumed-role/GlueServiceRole/GlueJobRun_01
is not authorized to perform: kms:GenerateDataKey on resource: 
arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab
```

**Step 2: Get KMS Key ID from Bucket**
```bash
aws s3api get-bucket-encryption --bucket target-bucket
```

Output:
```json
{
  "ServerSideEncryptionConfiguration": {
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab"
      }
    }]
  }
}
```

**Step 3: Update Glue Role Policy**

**Before (Missing KMS):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:PutObject"
    ],
    "Resource": "arn:aws:s3:::target-bucket/*"
  }]
}
```

**After (Complete):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::target-bucket/*"
    },
    {
      "Sid": "S3ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::target-bucket"
    },
    {
      "Sid": "KMSAccess",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:Encrypt",
        "kms:GenerateDataKey",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/1234abcd-12ab-34cd-56ef-1234567890ab"
    }
  ]
}
```

**Step 4: Verify KMS Key Policy Allows Glue Role**

```bash
aws kms get-key-policy \
    --key-id 1234abcd-12ab-34cd-56ef-1234567890ab \
    --policy-name default
```

Must include:
```json
{
  "Sid": "Allow Glue Service Role",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:role/GlueServiceRole"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "*"
}
```

**Step 5: Test Fix**
```bash
# Re-run Glue job
aws glue start-job-run --job-name my-etl-job

# Monitor
aws glue get-job-run \
    --job-name my-etl-job \
    --run-id jr_abc123 \
    --query 'JobRun.JobRunState'
```

**Expected:** `SUCCEEDED`

**Key Takeaways:**
- Always check encryption settings when troubleshooting S3 access
- KMS permissions are separate from S3 permissions
- Both role policy AND key policy must allow access
- Use `aws:ViaService` condition in KMS policy to restrict to Glue only

---

**Scenario 10: Design IAM strategy for a data engineering team that uses infrastructure-as-code (Terraform/CloudFormation).**

**Answer:**

**GitOps + IAM Roles Architecture:**

```
┌──────────────────────────────────────────────────────────────┐
│               CI/CD PIPELINE WITH IAM ROLES                   │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Developer Laptop                                             │
│  ┌────────────────────────────────────────────────┐          │
│  │  git push → GitHub                             │          │
│  │  (No AWS credentials locally)                  │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  GitHub Actions                                               │
│  ┌────────────────────────────────────────────────┐          │
│  │  Assume Role: GitHubActionsRole                │          │
│  │  via OIDC (no access keys!)                    │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Terraform Plan (Non-Prod)                                    │
│  ┌────────────────────────────────────────────────┐          │
│  │  Role: TerraformPlanRole                       │          │
│  │  Permissions: Read-only + Plan                 │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Manual Approval (for Prod)                                   │
│  ┌────────────────────────────────────────────────┐          │
│  │  Senior Engineer reviews plan                  │          │
│  │  Clicks "Approve" in GitHub                    │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Terraform Apply                                              │
│  ┌────────────────────────────────────────────────┐          │
│  │  Role: TerraformApplyRole                      │          │
│  │  Permissions: Create/Update/Delete resources   │          │
│  └────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

**Step 1: Set Up OIDC Provider (No Long-Lived Credentials)**

```bash
# Create OIDC provider for GitHub Actions
aws iam create-open-id-connect-provider \
    --url https://token.actions.githubusercontent.com \
    --client-id-list sts.amazonaws.com \
    --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

**Step 2: Create GitHub Actions Assume Role**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:myorg/data-platform-infra:*"
      }
    }
  }]
}
```

**Step 3: Create Terraform Plan Role (Read-Only)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadAllResources",
      "Effect": "Allow",
      "Action": [
        "s3:List*",
        "s3:Get*",
        "glue:Get*",
        "glue:List*",
        "athena:List*",
        "athena:Get*",
        "iam:Get*",
        "iam:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ReadTerraformState",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::terraform-state-bucket/*/terraform.tfstate"
    },
    {
      "Sid": "LockState",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-locks"
    }
  ]
}
```

**Step 4: Create Terraform Apply Role (Write Access)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ManageDataPlatformResources",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "glue:*",
        "athena:*",
        "lakeformation:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    },
    {
      "Sid": "ManageIAMRolesForDataPlatform",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy"
      ],
      "Resource": "arn:aws:iam::123456789012:role/DataPlatform*"
    },
    {
      "Sid": "PreventPrivilegeEscalation",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:CreateAccessKey",
        "iam:PutUserPolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

**Step 5: GitHub Actions Workflow**

```yaml
name: Terraform Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          role-session-name: github-actions-session
          aws-region: us-east-1
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Plan
        id: plan
        run: terraform plan -out=tfplan
      
      - name: Assume Apply Role (if approved)
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: |
          CREDS=$(aws sts assume-role --role-arn arn:aws:iam::123456789012:role/TerraformApplyRole --role-session-name terraform-apply)
          echo "AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r '.Credentials.AccessKeyId')" >> $GITHUB_ENV
          echo "AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r '.Credentials.SecretAccessKey')" >> $GITHUB_ENV
          echo "AWS_SESSION_TOKEN=$(echo $CREDS | jq -r '.Credentials.SessionToken')" >> $GITHUB_ENV
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply tfplan
```

**Step 6: Implement State Locking**

```hcl
# terraform backend
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "data-platform/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/terraform-state-key"
    dynamodb_table = "terraform-locks"
  }
}
```

**Step 7: Terraform IAM Module Example**

```hcl
# modules/data-engineer-role/main.tf
resource "aws_iam_role" "data_engineer" {
  name = "DataEngineer-${var.team_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::123456789012:root"
      }
      Action = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = var.external_id
        }
      }
    }]
  })

  tags = {
    Team        = var.team_name
    ManagedBy   = "Terraform"
    Environment = var.environment
  }
}

resource "aws_iam_role_policy" "data_engineer_s3" {
  name = "S3Access"
  role = aws_iam_role.data_engineer.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ]
      Resource = [
        "arn:aws:s3:::${var.team_bucket}",
        "arn:aws:s3:::${var.team_bucket}/*"
      ]
    }]
  })
}

# Prevent manual changes
lifecycle {
  prevent_destroy = true
}
```

**Benefits:**
- ✅ No long-lived AWS credentials
- ✅ All changes peer-reviewed via pull requests
- ✅ Audit trail in Git history
- ✅ Consistent, repeatable deployments
- ✅ Principle of least privilege (plan vs apply roles)
- ✅ Automatic documentation (Terraform code = documentation)

**Best Practices:**
1. **Never** commit AWS credentials to Git
2. Use separate roles for plan and apply
3. Enable MFA for production applies
4. Store Terraform state in S3 with encryption and versioning
5. Use DynamoDB for state locking (prevent concurrent modifications)
6. Tag all resources created by Terraform
7. Implement drift detection (scheduled Terraform plan)

---

### Cleanup Steps

```bash
# Delete IAM users
aws iam delete-login-profile --user-name data-engineer-john
aws iam delete-login-profile --user-name data-analyst-sarah

aws iam detach-user-policy --user-name data-engineer-john --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam detach-user-policy --user-name data-engineer-john --policy-arn arn:aws:iam::aws:policy/AWSGlueConsoleFullAccess
aws iam detach-user-policy --user-name data-engineer-john --policy-arn arn:aws:iam::aws:policy/AmazonAthenaFullAccess

aws iam detach-user-policy --user-name data-analyst-sarah --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam detach-user-policy --user-name data-analyst-sarah --policy-arn arn:aws:iam::aws:policy/AmazonAthenaFullAccess
aws iam detach-user-policy --user-name data-analyst-sarah --policy-arn arn:aws:iam::aws:policy/AWSGlueConsoleReadOnlyAccess

aws iam delete-user --user-name data-engineer-john
aws iam delete-user --user-name data-analyst-sarah

# Delete test S3 bucket
aws s3 rb s3://test-de-bucket-<random-number> --force
```

---

## Certification Tips

### DEA-C01 Exam Focus Areas

| Topic | Exam Weight | Key Concepts |
|-------|-------------|--------------|
| IAM Policies | HIGH | Effect, Action, Resource, Condition |
| IAM Roles vs Users | HIGH | When to use each, assume role process |
| Service Roles | MEDIUM | Glue, Lambda, EC2, cross-service permissions |
| Resource-Based Policies | MEDIUM | S3 bucket policies, KMS key policies |
| Permission Boundaries | LOW | Max permissions for IAM entities |
| SCPs (Organizations) | LOW | Organization-wide restrictions |

### Common Exam Traps

❌ **TRAP 1:** "Grant admin access to data engineer" → Never give admin (`*:*`)  
✅ **CORRECT:** Grant specific services (S3, Glue, Athena) with resource restrictions

❌ **TRAP 2:** "Use IAM user with access keys for Glue job" → Security risk  
✅ **CORRECT:** Use IAM role, Glue automatically gets temporary credentials

❌ **TRAP 3:** "S3 bucket policy allows access, but still Access Denied" → Check KMS  
✅ **CORRECT:** Encrypted buckets need KMS permissions too

❌ **TRAP 4:** "Cross-account access not working despite role configured" → Trust policy missing  
✅ **CORRECT:** Both trust policy (assume role) and permissions policy needed

### Important Limits

| Limit | Value | Notes |
|-------|-------|-------|
| IAM users per account | 5,000 | Use roles instead for large teams |
| IAM groups per account | 300 | |
| IAM roles per account | 1,000 | Request increase if needed |
| Managed policies per user/role | 10 | Use inline if more needed (not recommended) |
| Managed policy size | 6,144 chars | Break into multiple policies |
| Inline policy size | 2,048 chars | |
| MFA devices per user | 8 | |

### Frequently Tested Concepts

1. **Policy Evaluation Logic**
   - Default: Implicit Deny
   - Explicit Allow > Implicit Deny
   - Explicit Deny > Everything else

2. **S3 + IAM Interaction**
   - User policy: Allow s3:GetObject
   - Bucket policy: Deny for that user
   - Result: **Denied** (Deny wins)

3. **Cross-Account Access**
   - Account A: Create role with trust policy for Account B
   - Account B: User needs `sts:AssumeRole` permission
   - Account A: Role needs resource permissions (S3, Glue, etc.)

4. **Service Roles**
   - Glue job needs role to access S3, Glue Catalog, CloudWatch
   - Lambda function needs role to read from S3, write to DynamoDB
   - Always use managed policies when possible (AWS maintains them)

5. **KMS + S3**
   - S3 permissions alone not enough for encrypted buckets
   - Need: kms:Decrypt (read), kms:Encrypt + kms:GenerateDataKey (write)

---

## Production Best Practices

### Security

1. **Enable MFA for all human users**
   ```bash
   aws iam enable-mfa-device \
       --user-name john.doe \
       --serial-number arn:aws:iam::123456789012:mfa/john.doe \
       --authentication-code-1 123456 \
       --authentication-code-2 789012
   ```

2. **Use IAM roles, not users, for applications**
   - No credential rotation needed
   - Temporary credentials (auto-expire)
   - Easier to audit (CloudTrail shows original user)

3. **Implement password policy**
   ```bash
   aws iam update-account-password-policy \
       --minimum-password-length 14 \
       --require-symbols \
       --require-numbers \
       --require-uppercase-characters \
       --require-lowercase-characters \
       --max-password-age 90 \
       --password-reuse-prevention 24
   ```

4. **Enable CloudTrail for all accounts**
   - Captures all IAM API calls
   - Store logs in immutable S3 bucket
   - Set up alerts for suspicious activity

5. **Use permission boundaries**
   - Prevent privilege escalation
   - Max permissions a user can have
   - Useful for delegated administration

### Cost Optimization

IAM itself is **FREE**, but poor IAM design can lead to:

| Costly Mistake | Impact | Prevention |
|---------------|--------|------------|
| Too broad S3 access | Accidental data transfer charges | Use bucket/prefix restrictions |
| No resource tags | Can't track costs by team | Enforce tagging via IAM conditions |
| Unused IAM users | Potential security risk | Regular access reviews |
| Over-provisioned roles | Services access unnecessary resources | Principle of least privilege |

**Cost Monitoring:**
```bash
# Find unused credentials
aws iam generate-credential-report
aws iam get-credential-report

# Check last used dates
aws iam get-access-key-last-used --access-key-id AKIAIOSFODNN7EXAMPLE
```

### Scalability

1. **Use IAM groups for team-based access**
   - Don't attach policies to individual users
   - Add/remove users from groups
   - Change group policy to affect all members

2. **Standardize role names**
   - Pattern: `{Environment}-{Service}-{Purpose}`
   - Example: `Prod-Glue-ETLJobRole`
   - Makes automation easier

3. **Use Infrastructure-as-Code**
   - Terraform, CloudFormation, CDK
   - Version control for IAM policies
   - Consistent across environments

### Monitoring

1. **CloudWatch Metrics for IAM**
   ```bash
   # Create alarm for failed login attempts
   aws cloudwatch put-metric-alarm \
       --alarm-name failed-console-logins \
       --metric-name ConsoleSignInFailureCount \
       --namespace AWS/IAM \
       --statistic Sum \
       --period 300 \
       --evaluation-periods 1 \
       --threshold 5 \
       --comparison-operator GreaterThanThreshold
   ```

2. **IAM Access Analyzer**
   - Identifies resources shared with external entities
   - Continuous monitoring
   - Compliance reporting

3. **AWS Config Rules for IAM**
   - Enforce MFA on root account
   - Detect overly permissive policies
   - Ensure password policy compliance

### High Availability

IAM is a **global service**, automatically highly available across all AWS regions.

**Best Practices:**
- Don't rely on single IAM user (use roles)
- Implement break-glass access for emergencies
- Document role assumption procedures
- Test disaster recovery scenarios

**Break-Glass Access:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "aws:MultiFactorAuthPresent": "true"
      },
      "IpAddress": {
        "aws:SourceIp": ["203.0.113.0/24"]
      }
    }
  }]
}
```

---

## Hands-On Challenge

### Challenge 1: Multi-Account S3 Access

**Scenario:**  
You have two AWS accounts:
- Account A (111111111111): Contains S3 bucket `shared-data`
- Account B (222222222222): Data engineers need read access

**Task:**  
Set up cross-account access without using long-lived credentials.

**Acceptance Criteria:**
- Data engineer in Account B can list and read objects from `shared-data` bucket
- Access should be logged in both accounts
- No hardcoded credentials

**Hints:**
- Create IAM role in Account A
- Configure trust policy for Account B
- Grant `sts:AssumeRole` permission in Account B

---

### Challenge 2: Least Privilege Glue Job

**Scenario:**  
Create a Glue job that:
- Reads from `s3://source-bucket/raw-data/`
- Writes to `s3://target-bucket/processed-data/`
- Both buckets use KMS encryption with different keys

**Task:**  
Create IAM role with minimum required permissions.

**Acceptance Criteria:**
- Glue job succeeds
- Role cannot access other S3 buckets
- Role cannot delete objects (only read/write)
- CloudTrail shows all actions

**Hints:**
- Need both S3 and KMS permissions
- Two different KMS keys = two KMS permissions
- Use resource-level permissions

---

### Challenge 3: Automated Access Revocation

**Scenario:**  
Implement a system that:
- Grants temporary S3 read access to analysts
- Access expires after 8 hours
- Automatically revokes access
- Sends email notification when access expires

**Task:**  
Build the automation using Lambda, EventBridge, and IAM.

**Acceptance Criteria:**
- No manual revocation needed
- CloudTrail shows policy attachment/detachment
- Email sent to analyst when access expires
- Works for multiple concurrent requests

**Hints:**
- Use Lambda to attach inline policy with time condition
- EventBridge rule runs hourly to check for expired policies
- Tag policies with expiration timestamp

---

## Resume Project

### Project Title
**"Implemented Fine-Grained Access Control for Multi-Tenant Data Lake Using AWS IAM and Lake Formation"**

### Project Description

**Business Context:**  
Migrated on-premises Oracle-based data warehouse to AWS cloud, serving 15 business units with varying data access requirements and compliance needs (GDPR, SOX, HIPAA).

**Technical Implementation:**

1. **IAM Foundation**
   - Designed role-based access control (RBAC) strategy for 200+ data engineers, analysts, and scientists
   - Implemented cross-account access using IAM roles and STS for dev/staging/prod environment isolation
   - Replaced hardcoded credentials with IAM roles, improving security posture by eliminating 500+ long-lived access keys
   - Enabled MFA for all privileged users and established password rotation policies

2. **Lake Formation Integration**
   - Configured column-level security to protect PII fields (SSN, credit cards) while allowing analytics on non-sensitive columns
   - Implemented row-level security using data filters to enforce data residency requirements (EU vs US data)
   - Integrated with AWS Glue Data Catalog for centralized metadata management across 50+ databases

3. **Automation & Compliance**
   - Built automated access request workflow using Lambda, Step Functions, and SNS for time-bound data access
   - Implemented tag-based access control (ABAC) to automatically grant permissions based on team membership
   - Created CloudWatch dashboards for real-time IAM security monitoring and anomaly detection
   - Achieved 100% compliance with SOX audit requirements through comprehensive CloudTrail logging

4. **Cost Optimization**
   - Reduced monthly data transfer costs by 40% through granular S3 bucket policies preventing unauthorized cross-region access
   - Implemented IAM Access Analyzer to identify and remove unused roles, reducing attack surface by 30%

**Technologies:**  
AWS IAM, Lake Formation, S3, Glue, Athena, Lambda, CloudTrail, CloudWatch, KMS, Secrets Manager, Terraform

**Results:**
- Enabled secure self-service data access for 200+ users across 15 business units
- Reduced time-to-access from 3 days (manual approval) to 2 hours (automated workflow)
- Zero security incidents related to unauthorized data access in 18-month period
- Passed SOC 2 Type II and GDPR compliance audits with zero findings

---

## Solution Engineer Perspective

### Customer Discovery Questions

**For Oracle DBA Transitioning to AWS:**

1. **Current State Assessment:**
   - "Walk me through your current Oracle security model - how do you manage user access, roles, and privileges?"
   - "How do you currently handle database credentials for applications?"
   - "What's your process for granting temporary access to production data?"
   - "How do you audit who accessed what data and when?"

2. **Pain Points:**
   - "What are your biggest challenges with current access management?"
   - "How long does it take to provision a new user with appropriate permissions?"
   - "Have you experienced any security incidents related to compromised credentials?"
   - "How do you ensure least privilege access in your current environment?"

3. **Compliance & Governance:**
   - "What compliance requirements do you need to meet? (SOX, GDPR, HIPAA, PCI-DSS)"
   - "How do you prove to auditors that only authorized users accessed sensitive data?"
   - "Do you have requirements for data residency or sovereignty?"

4. **Integration:**
   - "Are you using Active Directory or LDAP for authentication?"
   - "Do you need to integrate with existing identity providers (Okta, Ping, Azure AD)?"
   - "How do applications currently authenticate to your databases?"

### Architecture Discussion Points

**When Presenting IAM to Oracle DBAs:**

1. **Oracle Concepts → AWS IAM Mapping**

| Oracle | AWS IAM Equivalent |
|--------|-------------------|
| `CREATE USER` | IAM User (but prefer IAM Roles) |
| `CREATE ROLE` | IAM Role |
| `GRANT SELECT ON table TO user` | IAM Policy with s3:GetObject |
| `GRANT EXECUTE ON procedure TO user` | IAM Policy with lambda:InvokeFunction |
| `ALTER USER IDENTIFIED BY password` | IAM password policy + rotation |
| `AUDIT SELECT ON table BY ACCESS` | CloudTrail + S3 access logs |
| `CREATE PROFILE` | IAM password policy + MFA |
| `DBMS_FGA` (Fine-Grained Audit) | Lake Formation data filters |
| `VPD` (Virtual Private Database) | Lake Formation row-level security |

2. **Migration Conversation Starters:**

**"In Oracle, you might have 500 application users connecting directly to the database. In AWS, we flip that model:**
- Applications use IAM roles (temporary credentials)
- No usernames/passwords to manage
- Credentials auto-rotate every hour
- Can't be leaked because they expire
- CloudTrail shows exactly which application (role) accessed what data"

**"Your Oracle audit trail shows WHO did WHAT and WHEN. CloudTrail does the same, but also captures:**
- Failed access attempts (someone trying to access data they shouldn't)
- Changes to permissions (who granted access to whom)
- Cross-service access (Lambda reading from S3, writing to RDS)
- Immutable logs (can't be deleted even by root user if configured correctly)"

3. **Value Proposition**

**Security:**
- "No more password sprawl - 500 app users with passwords → 20 IAM roles with auto-rotating credentials"
- "Principle of least privilege enforced programmatically, not manually"
- "Impossible to accidentally grant more permissions than intended (permission boundaries)"

**Operational Efficiency:**
- "New team member access in minutes, not days (add to IAM group)"
- "Automated access reviews (IAM Access Analyzer tells you who can access what)"
- "Self-service data access with approval workflows (Lambda + Step Functions)"

**Compliance:**
- "Every API call logged forever (CloudTrail to S3 with Glacier archival)"
- "Prove to auditors that only authorized users accessed PII (Lake Formation audit logs)"
- "Automated compliance checks (AWS Config rules for IAM policies)"

**Cost:**
- "IAM is free - pay only for what resources you access, not for authentication"
- "Reduce data transfer costs with granular S3 bucket policies"
- "Eliminate license costs for third-party IAM tools"

### Customer Use Case Examples

**Use Case 1: Financial Services - SOX Compliance**

**Customer:** "We need to prove to auditors that only authorized users can modify financial data, and we need immutable audit logs for 7 years."

**Solution:**
- IAM policies restrict write access to specific roles (Accounting, Finance)
- S3 bucket policies with `Deny` for delete operations
- CloudTrail logs to S3 with Object Lock (WORM)
- Glacier Deep Archive for 7-year retention at $0.00099/GB/month
- Lake Formation for column-level access (hide salaries from non-HR users)

**Business Outcome:**  
"You'll pass SOX audits faster because auditors can query CloudTrail directly. In my last customer, audit preparation went from 2 weeks to 2 days."

---

**Use Case 2: Healthcare - HIPAA Protected Health Information (PHI)**

**Customer:** "We have 50 hospitals sharing patient data in a data lake. Each hospital should only see their own patients' data, but our data scientists need aggregated views across all hospitals for research."

**Solution:**
- Lake Formation row-level security with filter: `hospital_id = ${aws:PrincipalTag/HospitalID}`
- Tag each hospital's IAM role with their hospital ID
- Data scientists get separate role with aggregated views (no PII)
- KMS encryption with hospital-specific keys for data-at-rest encryption
- VPC endpoints for private connectivity (no internet access)

**Business Outcome:**  
"Hospitals maintain data sovereignty while enabling cross-hospital analytics. This architecture is pre-approved by many HIPAA compliance officers."

---

**Use Case 3: Retail - Multi-Tenant SaaS Platform**

**Customer:** "We're building a SaaS platform where customers upload their sales data. Each customer must be isolated - Customer A can't see Customer B's data."

**Solution:**
- S3 bucket structure: `s3://saas-platform/customer-123/data/`
- IAM policy with condition: `"s3:prefix": ["${aws:PrincipalTag/CustomerID}/*"]`
- Tag each customer's role with their CustomerID
- Glue Data Catalog with separate database per customer
- Athena workgroups for cost tracking per customer

**Business Outcome:**  
"You get strong tenant isolation without managing separate AWS accounts. Plus, you can show customers their exact AWS costs (important for usage-based billing)."

---

### Conversation Framework for Solution Engineers

**Opening:**  
"I see you have 14 years of Oracle DBA experience. That's fantastic - your understanding of security, RBAC, and audit trails will translate directly to AWS IAM. Let me show you how we solve the same problems in the cloud, often more elegantly."

**Middle (Demo/Whiteboard):**  
"In Oracle, you'd run `GRANT SELECT ON sales_data TO analyst_user`. In AWS, we do this: [show IAM policy]. Notice how it's more explicit - you can see exactly what's allowed. No hidden system privileges."

**Handling Objections:**  
- **"IAM seems more complex than Oracle roles"** → "It's more verbose, but also more powerful. You can grant access based on time of day, IP address, MFA status - things Oracle can't do natively."
- **"What if I need to revoke access immediately?"** → "Detach the policy, and access is revoked within seconds globally. In Oracle, you'd need to kill sessions."
- **"How do I debug 'Access Denied' errors?"** → "CloudTrail shows the exact policy evaluation logic. It's like Oracle's `AUDIT_TRAIL` but with explanations of why access was denied."

**Closing:**  
"Let's build a proof-of-concept together. Pick your most complex Oracle security scenario, and I'll show you how we'd implement it in AWS with IAM and Lake Formation."

---

This completes MODULE 1: IAM for Data Engineers. The next modules (S3, Glue Data Catalog, Athena) will build on these IAM foundations.

**Total Module 1 Content:**
- ✅ 10 detailed scenario-based interview questions
- ✅ Complete CLI and Console steps
- ✅ Architecture diagrams
- ✅ Production best practices
- ✅ Solution Engineer perspective for customer conversations
- ✅ Oracle DBA transition guidance

---


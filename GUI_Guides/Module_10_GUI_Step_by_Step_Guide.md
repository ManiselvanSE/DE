# Module 10: Networking & Content Delivery - GUI Step-by-Step Guide

## Overview
This module covers AWS networking and content delivery services essential for data engineering. You'll learn to build secure VPC architectures, implement global content delivery with CloudFront, and establish hybrid connectivity with Direct Connect.

**Duration:** 4-5 hours  
**Cost:** $10-40 (depending on Direct Connect usage)  
**Difficulty:** Intermediate to Advanced

---

## Exercise 10.1: Secure VPC Architecture for Data Lake

### Introduction
Build a secure VPC environment for a data lake with private subnets, VPC endpoints, RDS database, and Lambda function - all without internet access or NAT Gateway costs.

**What You'll Build:**
- VPC with private subnets across 2 availability zones
- S3 and Secrets Manager VPC endpoints
- RDS PostgreSQL in private subnet
- Lambda function to extract data from RDS to S3

**Duration:** 90 minutes  
**Cost:** Free (VPC and gateway endpoints are free)

---

### Part 1: Create VPC and Subnets

#### Step 1: Create VPC
1. Open the **AWS Console** and navigate to **VPC**
2. In the left sidebar, click **Your VPCs**
3. Click the orange **Create VPC** button
4. Configure VPC:
   - **Resources to create:** Select **VPC only**
   - **Name tag:** `data-lake-vpc`
   - **IPv4 CIDR block:** `10.0.0.0/16`
   - **IPv6 CIDR block:** No IPv6 CIDR block
   - **Tenancy:** Default
5. Click **Create VPC**

#### Step 2: Enable DNS Hostnames
1. Select your newly created VPC (`data-lake-vpc`)
2. Click **Actions** → **Edit VPC settings**
3. Under **DNS settings**:
   - Check **Enable DNS hostnames**
   - Verify **Enable DNS resolution** is checked
4. Click **Save**

#### Step 3: Create Private Subnet 1
1. In the left sidebar, click **Subnets**
2. Click **Create subnet**
3. Configure subnet:
   - **VPC ID:** Select `data-lake-vpc`
   - **Subnet name:** `private-subnet-1a`
   - **Availability Zone:** Select `us-east-1a`
   - **IPv4 CIDR block:** `10.0.1.0/24`
4. Click **Create subnet**

#### Step 4: Create Private Subnet 2
1. Click **Create subnet** again
2. Configure subnet:
   - **VPC ID:** Select `data-lake-vpc`
   - **Subnet name:** `private-subnet-1b`
   - **Availability Zone:** Select `us-east-1b`
   - **IPv4 CIDR block:** `10.0.2.0/24`
3. Click **Create subnet**

---

### Part 2: Create VPC Endpoints

#### Step 5: Create S3 Gateway Endpoint
1. In the left sidebar, click **Endpoints**
2. Click **Create endpoint**
3. Configure endpoint:
   - **Name tag:** `s3-gateway-endpoint`
   - **Service category:** Select **AWS services**
   - **Services:** Search for `s3` and select **com.amazonaws.us-east-1.s3** (Type: Gateway)
   - **VPC:** Select `data-lake-vpc`
   - **Route tables:** Check the default route table for `data-lake-vpc`
   - **Policy:** Full access (default)
4. Click **Create endpoint**

#### Step 6: Create Security Group for VPC Endpoints
1. In the left sidebar, click **Security Groups**
2. Click **Create security group**
3. Configure security group:
   - **Security group name:** `vpc-endpoint-sg`
   - **Description:** `Security group for VPC endpoints`
   - **VPC:** Select `data-lake-vpc`
4. Under **Inbound rules**, click **Add rule**:
   - **Type:** HTTPS
   - **Protocol:** TCP
   - **Port range:** 443
   - **Source:** Custom - `10.0.0.0/16`
   - **Description:** Allow HTTPS from VPC
5. Click **Create security group**

#### Step 7: Create Secrets Manager Interface Endpoint
1. Go back to **Endpoints** and click **Create endpoint**
2. Configure endpoint:
   - **Name tag:** `secretsmanager-endpoint`
   - **Service category:** Select **AWS services**
   - **Services:** Search for `secretsmanager` and select **com.amazonaws.us-east-1.secretsmanager** (Type: Interface)
   - **VPC:** Select `data-lake-vpc`
   - **Subnets:** 
     - Check `private-subnet-1a` (us-east-1a)
     - Check `private-subnet-1b` (us-east-1b)
   - **Security groups:** Select `vpc-endpoint-sg`
   - **Enable DNS name:** Check this box
   - **Policy:** Full access (default)
3. Click **Create endpoint**

---

### Part 3: Create RDS Database in Private Subnet

#### Step 8: Create RDS Security Group
1. Go to **VPC** → **Security Groups**
2. Click **Create security group**
3. Configure security group:
   - **Security group name:** `rds-sg`
   - **Description:** `Security group for RDS PostgreSQL`
   - **VPC:** Select `data-lake-vpc`
4. Leave **Inbound rules** empty for now (will add Lambda access later)
5. Click **Create security group**

#### Step 9: Create DB Subnet Group
1. Navigate to **RDS** service
2. In the left sidebar, click **Subnet groups**
3. Click **Create DB subnet group**
4. Configure subnet group:
   - **Name:** `data-lake-db-subnet`
   - **Description:** `Subnet group for RDS in private subnets`
   - **VPC:** Select `data-lake-vpc`
   - **Availability Zones:** Select `us-east-1a` and `us-east-1b`
   - **Subnets:** 
     - Select `10.0.1.0/24` (private-subnet-1a)
     - Select `10.0.2.0/24` (private-subnet-1b)
5. Click **Create**

#### Step 10: Store Database Credentials in Secrets Manager
1. Navigate to **Secrets Manager**
2. Click **Store a new secret**
3. Configure secret:
   - **Secret type:** Select **Credentials for Amazon RDS database**
   - **User name:** `postgres`
   - **Password:** `MySecurePassword123!`
   - **Encryption key:** Use default AWS managed key
   - **Database:** We'll select this later (after RDS creation)
4. Click **Next**
5. Secret name and description:
   - **Secret name:** `prod/rds/master`
   - **Description:** `Master credentials for data lake RDS`
6. Click **Next**
7. Automatic rotation: Disable for now
8. Click **Next**, then **Store**

#### Step 11: Create RDS PostgreSQL Instance
1. Go to **RDS** → **Databases**
2. Click **Create database**
3. Configure database:
   - **Engine type:** PostgreSQL
   - **Version:** PostgreSQL 15.4 (or latest)
   - **Templates:** Free tier (or Dev/Test)
   - **DB instance identifier:** `data-lake-rds`
   - **Master username:** `postgres`
   - **Master password:** `MySecurePassword123!`
   - **Confirm password:** `MySecurePassword123!`
   - **DB instance class:** db.t3.micro
   - **Storage type:** General Purpose SSD (gp3)
   - **Allocated storage:** 20 GiB
   - **Storage autoscaling:** Disable
4. Connectivity:
   - **Virtual private cloud (VPC):** Select `data-lake-vpc`
   - **DB subnet group:** Select `data-lake-db-subnet`
   - **Public access:** No
   - **VPC security group:** Choose existing → Select `rds-sg`
   - **Availability Zone:** us-east-1a
5. Additional configuration:
   - **Initial database name:** `salesdb`
   - **Enable automated backups:** Yes (7 days retention)
   - **Encryption:** Enable encryption
   - **Monitoring:** Enable Enhanced Monitoring (optional)
6. Click **Create database**
7. Wait 5-10 minutes for database creation

#### Step 12: Update Secret with RDS Endpoint
1. Once RDS is available, copy the **Endpoint** (e.g., `data-lake-rds.xxxxxx.us-east-1.rds.amazonaws.com`)
2. Go to **Secrets Manager** → **Secrets**
3. Click on `prod/rds/master`
4. Click **Retrieve secret value**
5. Click **Edit**
6. Update the JSON to include endpoint:
```json
{
  "username": "postgres",
  "password": "MySecurePassword123!",
  "engine": "postgres",
  "host": "data-lake-rds.xxxxxx.us-east-1.rds.amazonaws.com",
  "port": 5432,
  "dbname": "salesdb"
}
```
7. Click **Save**

---

### Part 4: Create Lambda Function in VPC

#### Step 13: Create S3 Bucket for Lambda Output
1. Navigate to **S3**
2. Click **Create bucket**
3. Configure bucket:
   - **Bucket name:** `data-lake-output-[your-account-id]` (must be globally unique)
   - **Region:** us-east-1
   - **Block all public access:** Keep checked
   - **Bucket Versioning:** Disable
   - **Encryption:** Server-side encryption with Amazon S3 managed keys (SSE-S3)
4. Click **Create bucket**

#### Step 14: Create Lambda Security Group
1. Go to **VPC** → **Security Groups**
2. Click **Create security group**
3. Configure security group:
   - **Security group name:** `lambda-sg`
   - **Description:** `Security group for Lambda functions`
   - **VPC:** Select `data-lake-vpc`
4. **Outbound rules:**
   - Click **Add rule**:
     - **Type:** HTTPS
     - **Protocol:** TCP
     - **Port:** 443
     - **Destination:** Custom - `10.0.0.0/16`
     - **Description:** HTTPS to VPC endpoints
   - Click **Add rule** again:
     - **Type:** PostgreSQL
     - **Protocol:** TCP
     - **Port:** 5432
     - **Destination:** Select security group `rds-sg`
     - **Description:** Database access
5. Click **Create security group**

#### Step 15: Update RDS Security Group (Allow Lambda Access)
1. Still in **Security Groups**, select `rds-sg`
2. Click **Edit inbound rules**
3. Click **Add rule**:
   - **Type:** PostgreSQL
   - **Protocol:** TCP
   - **Port:** 5432
   - **Source:** Select security group `lambda-sg`
   - **Description:** Allow Lambda access
4. Click **Save rules**

#### Step 16: Create IAM Role for Lambda
1. Navigate to **IAM** → **Roles**
2. Click **Create role**
3. Select trusted entity:
   - **Trusted entity type:** AWS service
   - **Use case:** Lambda
4. Click **Next**
5. Add permissions (select these policies):
   - `AWSLambdaVPCAccessExecutionRole`
   - `SecretsManagerReadWrite`
   - `AmazonS3FullAccess` (or create custom policy for specific bucket)
6. Click **Next**
7. Role name: `lambda-vpc-execution-role`
8. Click **Create role**

#### Step 17: Create Lambda Layer for psycopg2
1. On your local machine, create a directory and install psycopg2:
```bash
mkdir -p python/lib/python3.11/site-packages
pip install psycopg2-binary -t python/lib/python3.11/site-packages
zip -r psycopg2-layer.zip python
```
2. Go to **Lambda** → **Layers**
3. Click **Create layer**
4. Configure layer:
   - **Name:** `psycopg2-layer`
   - **Upload:** Upload the `psycopg2-layer.zip` file
   - **Compatible runtimes:** Python 3.11
5. Click **Create**

#### Step 18: Create Lambda Function
1. Go to **Lambda** → **Functions**
2. Click **Create function**
3. Configure function:
   - **Function name:** `rds-extractor-vpc`
   - **Runtime:** Python 3.11
   - **Architecture:** x86_64
   - **Permissions:** Use an existing role → Select `lambda-vpc-execution-role`
4. Click **Create function**

#### Step 19: Add Lambda Layer
1. Scroll down to **Layers** section
2. Click **Add a layer**
3. Select **Custom layers**
4. Choose `psycopg2-layer` (version 1)
5. Click **Add**

#### Step 20: Configure Lambda VPC Settings
1. Click **Configuration** tab
2. Click **VPC** in the left menu
3. Click **Edit**
4. Configure VPC:
   - **VPC:** Select `data-lake-vpc`
   - **Subnets:** Select both `private-subnet-1a` and `private-subnet-1b`
   - **Security groups:** Select `lambda-sg`
5. Click **Save**

#### Step 21: Add Lambda Function Code
1. Click **Code** tab
2. Replace the code in `lambda_function.py` with:
```python
import boto3
import psycopg2
import json
import csv
from io import StringIO

secretsmanager = boto3.client('secretsmanager')
s3 = boto3.client('s3')

def lambda_handler(event, context):
    try:
        # Get database credentials from Secrets Manager
        secret = secretsmanager.get_secret_value(SecretId='prod/rds/master')
        creds = json.loads(secret['SecretString'])
        
        # Connect to RDS PostgreSQL
        conn = psycopg2.connect(
            host=creds['host'],
            port=creds['port'],
            database=creds['dbname'],
            user=creds['username'],
            password=creds['password']
        )
        
        # Create sales table if it doesn't exist
        cursor = conn.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS sales (
                order_id SERIAL PRIMARY KEY,
                customer_name VARCHAR(100),
                product VARCHAR(100),
                amount DECIMAL(10,2),
                order_date DATE DEFAULT CURRENT_DATE
            )
        """)
        
        # Insert sample data if table is empty
        cursor.execute("SELECT COUNT(*) FROM sales")
        if cursor.fetchone()[0] == 0:
            cursor.execute("""
                INSERT INTO sales (customer_name, product, amount, order_date)
                VALUES 
                    ('John Doe', 'Laptop', 1200.00, CURRENT_DATE),
                    ('Jane Smith', 'Mouse', 25.00, CURRENT_DATE),
                    ('Bob Johnson', 'Keyboard', 75.00, CURRENT_DATE - INTERVAL '1 day'),
                    ('Alice Williams', 'Monitor', 350.00, CURRENT_DATE - INTERVAL '1 day')
            """)
            conn.commit()
        
        # Query yesterday's sales
        cursor.execute("""
            SELECT order_id, customer_name, product, amount, order_date 
            FROM sales 
            WHERE order_date >= CURRENT_DATE - INTERVAL '1 day'
        """)
        rows = cursor.fetchall()
        columns = [desc[0] for desc in cursor.description]
        
        # Convert to CSV
        csv_buffer = StringIO()
        writer = csv.writer(csv_buffer)
        writer.writerow(columns)  # Header
        writer.writerows(rows)
        
        # Upload to S3
        bucket_name = event.get('bucket', 'data-lake-output-REPLACE-WITH-YOUR-ACCOUNT-ID')
        s3.put_object(
            Bucket=bucket_name,
            Key='rds-extracts/sales.csv',
            Body=csv_buffer.getvalue().encode('utf-8'),
            ContentType='text/csv'
        )
        
        conn.close()
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Success',
                'rows_extracted': len(rows),
                'file': f's3://{bucket_name}/rds-extracts/sales.csv'
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```
3. **Important:** Replace `REPLACE-WITH-YOUR-ACCOUNT-ID` with your actual AWS account ID
4. Click **Deploy**

#### Step 22: Configure Lambda Timeout
1. Click **Configuration** tab
2. Click **General configuration**
3. Click **Edit**
4. Set **Timeout:** 5 minutes
5. Click **Save**

#### Step 23: Test Lambda Function
1. Click **Test** tab
2. Create new test event:
   - **Event name:** `TestExtract`
   - **Event JSON:**
```json
{
  "bucket": "data-lake-output-YOUR-ACCOUNT-ID"
}
```
3. Click **Save**
4. Click **Test**
5. Wait for execution (may take 30-60 seconds on first run)
6. Verify successful execution in the response

#### Step 24: Verify Data in S3
1. Navigate to **S3**
2. Click on your bucket (`data-lake-output-[account-id]`)
3. Navigate to `rds-extracts/` folder
4. You should see `sales.csv` file
5. Click on the file and download to verify contents

---

### Verification Checklist

- [ ] VPC created with CIDR 10.0.0.0/16
- [ ] Two private subnets created in different AZs
- [ ] S3 Gateway endpoint created
- [ ] Secrets Manager interface endpoint created
- [ ] RDS PostgreSQL created in private subnet
- [ ] Database credentials stored in Secrets Manager
- [ ] Lambda function deployed in VPC
- [ ] Lambda successfully connects to RDS via VPC
- [ ] Lambda successfully accesses Secrets Manager via endpoint
- [ ] Lambda successfully writes to S3 via gateway endpoint
- [ ] CSV file with sales data appears in S3

### Architecture Benefits

**Cost Savings:**
- **Without VPC Endpoints:** Would need NAT Gateway ($32.40/month + $0.045/GB)
- **With VPC Endpoints:** Free for gateway endpoints, $7.20/month for interface endpoints
- **Monthly Savings:** ~78% ($54.60 vs $15.60 for moderate usage)

**Security:**
- Zero internet exposure
- All traffic stays within AWS network
- Encrypted secrets management
- No public IPs required

**Performance:**
- Low latency (< 5 ms to VPC endpoints)
- High throughput (10 Gbps+)
- No internet bottlenecks

---

### Cleanup (Important!)

1. **Delete Lambda Function:**
   - Lambda → Functions → Select `rds-extractor-vpc` → Actions → Delete

2. **Delete RDS Instance:**
   - RDS → Databases → Select `data-lake-rds` → Actions → Delete
   - Uncheck "Create final snapshot"
   - Check "I acknowledge..."
   - Type `delete me`
   - Click Delete

3. **Delete DB Subnet Group:**
   - RDS → Subnet groups → Select `data-lake-db-subnet` → Delete

4. **Delete VPC Endpoints:**
   - VPC → Endpoints → Select both endpoints → Actions → Delete

5. **Delete Security Groups:**
   - VPC → Security Groups → Select `lambda-sg`, `rds-sg`, `vpc-endpoint-sg` → Actions → Delete

6. **Delete Subnets:**
   - VPC → Subnets → Select both subnets → Actions → Delete

7. **Delete VPC:**
   - VPC → Your VPCs → Select `data-lake-vpc` → Actions → Delete

8. **Delete S3 Bucket:**
   - S3 → Select bucket → Empty → Delete

9. **Delete Secrets:**
   - Secrets Manager → Select `prod/rds/master` → Actions → Delete secret → 7 days recovery

10. **Delete IAM Role:**
    - IAM → Roles → Select `lambda-vpc-execution-role` → Delete

---

## Exercise 10.2: CloudFront Distribution for Analytics Dashboard

### Introduction
Deploy a global content delivery network (CDN) using CloudFront to serve an analytics dashboard with low latency worldwide, secure S3 access, and WAF protection.

**What You'll Build:**
- S3 bucket hosting static dashboard assets
- CloudFront distribution with Origin Access Identity
- Signed URLs for authenticated access
- AWS WAF rules for rate limiting and security

**Duration:** 75 minutes  
**Cost:** ~$10/month for moderate traffic (first 10 TB free tier)

---

### Part 1: Create and Configure S3 Bucket

#### Step 1: Create S3 Bucket
1. Navigate to **S3**
2. Click **Create bucket**
3. Configure bucket:
   - **Bucket name:** `analytics-dashboard-assets-[unique-id]`
   - **Region:** us-east-1
   - **Block all public access:** Keep checked (CloudFront will access privately)
   - **Versioning:** Enable
   - **Encryption:** Server-side encryption with Amazon S3 managed keys (SSE-S3)
4. Click **Create bucket**

#### Step 2: Upload Sample Dashboard Files
1. Click on your bucket
2. Click **Upload**
3. Create a simple `index.html` file locally:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Analytics Dashboard</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <h1>Sales Analytics Dashboard</h1>
    <div id="dashboard">
        <div class="metric">
            <h2>Total Sales</h2>
            <p class="value">$1,234,567</p>
        </div>
        <div class="metric">
            <h2>Orders Today</h2>
            <p class="value">342</p>
        </div>
        <div class="metric">
            <h2>Active Users</h2>
            <p class="value">1,847</p>
        </div>
    </div>
    <script src="app.js"></script>
</body>
</html>
```
4. Create `styles.css`:
```css
body {
    font-family: Arial, sans-serif;
    margin: 0;
    padding: 20px;
    background: #f5f5f5;
}
h1 {
    color: #333;
}
#dashboard {
    display: flex;
    gap: 20px;
}
.metric {
    background: white;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    flex: 1;
}
.value {
    font-size: 36px;
    font-weight: bold;
    color: #0066cc;
}
```
5. Create `app.js`:
```javascript
console.log('Dashboard loaded successfully');
document.addEventListener('DOMContentLoaded', function() {
    console.log('Analytics dashboard initialized');
});
```
6. Upload all three files to the bucket
7. Click **Upload**

#### Step 3: Create Error Page
1. Create `error.html` locally:
```html
<!DOCTYPE html>
<html>
<head>
    <title>Error - Analytics Dashboard</title>
</head>
<body>
    <h1>Oops! Something went wrong</h1>
    <p>The page you're looking for cannot be found.</p>
</body>
</html>
```
2. Upload to the bucket

---

### Part 2: Configure CloudFront Distribution

#### Step 4: Create Origin Access Identity (OAI)
1. Navigate to **CloudFront**
2. In the left sidebar, click **Origin access** (under Security)
3. Click **Create origin access identity**
4. Configure OAI:
   - **Name:** `analytics-dashboard-oai`
   - **Comment:** `OAI for analytics dashboard`
5. Click **Create**
6. **Copy the OAI ID** (e.g., `E1234ABCDEF567`) - you'll need this

#### Step 5: Update S3 Bucket Policy
1. Go back to **S3** → Select your bucket
2. Click **Permissions** tab
3. Scroll to **Bucket policy** → Click **Edit**
4. Add this policy (replace `YOUR-BUCKET-NAME` and `OAI-ID`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAI",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity OAI-ID"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```
5. Click **Save changes**

#### Step 6: Create CloudFront Distribution
1. Go to **CloudFront** → **Distributions**
2. Click **Create distribution**
3. **Origin settings:**
   - **Origin domain:** Select your S3 bucket from dropdown
   - **Origin path:** Leave blank
   - **Name:** Leave default (auto-filled)
   - **Origin access:** Select **Legacy access identities**
   - **Origin access identity:** Select `analytics-dashboard-oai`
   - **Bucket policy:** Click **Yes, update the bucket policy** (if shown)
4. **Default cache behavior:**
   - **Viewer protocol policy:** Redirect HTTP to HTTPS
   - **Allowed HTTP methods:** GET, HEAD
   - **Cache key and origin requests:** Select **Cache policy and origin request policy**
   - **Cache policy:** Managed-CachingOptimized
   - **Origin request policy:** None
   - **Response headers policy:** None
   - **Compress objects automatically:** Yes
5. **Function associations:** Leave blank for now
6. **Settings:**
   - **Price class:** Use all edge locations (best performance)
   - **AWS WAF web ACL:** None (will add later)
   - **Alternate domain name (CNAME):** Leave blank (or add your custom domain)
   - **Custom SSL certificate:** Default CloudFront Certificate
   - **Supported HTTP versions:** HTTP/2, HTTP/3
   - **Default root object:** `index.html`
   - **Standard logging:** Off (or enable if you want access logs)
   - **IPv6:** On
   - **Description:** `Analytics dashboard distribution`
7. Click **Create distribution**
8. Wait 5-15 minutes for deployment (Status will change from "Deploying" to "Enabled")
9. **Copy the Distribution domain name** (e.g., `d1234abcdef567.cloudfront.net`)

#### Step 7: Configure Custom Error Response
1. Select your distribution → Click **Error pages** tab
2. Click **Create custom error response**
3. Configure error response:
   - **HTTP error code:** 404: Not Found
   - **Customize error response:** Yes
   - **Response page path:** `/error.html`
   - **HTTP response code:** 404: Not Found
   - **Error caching minimum TTL:** 300
4. Click **Create custom error response**
5. Repeat for 403 error if desired

#### Step 8: Test CloudFront Distribution
1. Copy the CloudFront domain name
2. Open a browser and navigate to: `https://d1234abcdef567.cloudfront.net`
3. You should see your analytics dashboard
4. Test CSS and JS loading by opening browser DevTools
5. Test error page: `https://d1234abcdef567.cloudfront.net/nonexistent.html`

---

### Part 3: Implement Signed URLs for Authentication

#### Step 9: Create CloudFront Key Pair
1. Log in to AWS Console as **root user** (required for key pair creation)
2. Click on account name → **Security Credentials**
3. Scroll to **CloudFront key pairs** section
4. Click **Create key pair**
5. Download both:
   - Private key file (`.pem`)
   - Public key file (`.xml`)
6. **Note the Key Pair ID** (e.g., `APKAXXXXXXXX`)
7. Store the private key securely

#### Step 10: Add Key Pair as Trusted Signer
1. Go to **CloudFront** → **Distributions**
2. Select your distribution → **Behaviors** tab
3. Select the default behavior → Click **Edit**
4. Scroll to **Restrict viewer access**:
   - Select **Yes**
   - **Trusted authorization type:** Select **Trusted key groups (recommended)**
5. Click **Create key group**
6. In the new tab:
   - Click **Public keys** in left sidebar
   - Click **Add public key**
   - **Key name:** `analytics-dashboard-key`
   - Paste the content from your public key `.xml` file (the long base64 string)
   - Click **Add**
7. Go to **Key groups** → **Create key group**
   - **Key group name:** `analytics-dashboard-key-group`
   - **Public keys:** Select the key you just created
   - Click **Create key group**
8. Go back to your distribution behavior edit page
9. Under **Trusted key groups:** Select your key group
10. Click **Save changes**
11. Wait for distribution to deploy (~5-10 minutes)

#### Step 11: Create Lambda Function for Signed URL Generation
1. Navigate to **Lambda**
2. Click **Create function**
3. Configure:
   - **Function name:** `generate-signed-url`
   - **Runtime:** Python 3.11
   - **Architecture:** x86_64
4. Click **Create function**
5. Replace code with:
```python
import json
import datetime
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding
from botocore.signers import CloudFrontSigner
import base64

PRIVATE_KEY_PEM = """
-----BEGIN RSA PRIVATE KEY-----
[PASTE YOUR PRIVATE KEY CONTENT HERE]
-----END RSA PRIVATE KEY-----
"""

KEY_PAIR_ID = "APKAXXXXXXXX"  # Replace with your key pair ID
CLOUDFRONT_DOMAIN = "d1234abcdef567.cloudfront.net"  # Replace with your domain

def lambda_handler(event, context):
    # Get parameters from event or use defaults
    resource_path = event.get('path', '/index.html')
    expiration_hours = int(event.get('expiration_hours', 1))
    
    # Calculate expiration time
    expire_time = datetime.datetime.now() + datetime.timedelta(hours=expiration_hours)
    
    # Load private key
    private_key = serialization.load_pem_private_key(
        PRIVATE_KEY_PEM.encode('utf-8'),
        password=None
    )
    
    # Create RSA signer function
    def rsa_signer(message):
        return private_key.sign(message, padding.PKCS1v15(), hashes.SHA1())
    
    # Create CloudFront signer
    cloudfront_signer = CloudFrontSigner(KEY_PAIR_ID, rsa_signer)
    
    # Generate URL
    url = f"https://{CLOUDFRONT_DOMAIN}{resource_path}"
    
    # Generate signed URL
    signed_url = cloudfront_signer.generate_presigned_url(
        url,
        date_less_than=expire_time
    )
    
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps({
            'signed_url': signed_url,
            'expires_at': expire_time.isoformat(),
            'resource': resource_path
        })
    }
```
6. Update the placeholders with your actual values
7. Click **Deploy**

#### Step 12: Add Cryptography Layer to Lambda
1. On your local machine, create and package the cryptography library:
```bash
mkdir -p python/lib/python3.11/site-packages
pip install cryptography -t python/lib/python3.11/site-packages
zip -r cryptography-layer.zip python
```
2. Go to **Lambda** → **Layers** → **Create layer**
3. Configure:
   - **Name:** `cryptography-layer`
   - **Upload:** Upload `cryptography-layer.zip`
   - **Compatible runtimes:** Python 3.11
4. Click **Create**
5. Go back to your function → **Layers** → **Add a layer**
6. Select your `cryptography-layer` and **Add**

#### Step 13: Test Signed URL Generation
1. Click **Test** tab
2. Create test event:
```json
{
  "path": "/index.html",
  "expiration_hours": 2
}
```
3. Click **Test**
4. Copy the `signed_url` from the response
5. Try accessing CloudFront without signed URL (should fail)
6. Access with the signed URL (should work)

---

### Part 4: Add AWS WAF Protection

#### Step 14: Create WAF Web ACL
1. Navigate to **AWS WAF**
2. Click **Create web ACL**
3. **Describe web ACL:**
   - **Resource type:** CloudFront distributions
   - **Name:** `analytics-dashboard-waf`
   - **Description:** `WAF for analytics dashboard`
   - **CloudWatch metric name:** `analyticsDashboardWAF`
   - **Region:** Global (CloudFront)
4. Click **Next**

#### Step 15: Add Rate Limiting Rule
1. Click **Add rules** → **Add my own rules and rule groups**
2. **Rule type:** Rate-based rule
3. Configure:
   - **Name:** `RateLimitRule`
   - **Rate limit:** 2000 requests per 5 minutes
   - **IP address to use:** Source IP address
   - **Scope:** Consider all requests
   - **Action:** Block
4. Click **Add rule**

#### Step 16: Add Geo-Blocking Rule (Optional)
1. Click **Add rule** again
2. **Rule type:** Regular rule
3. Configure:
   - **Name:** `GeoBlockRule`
   - **Type:** Regular rule
   - **If a request:** matches the statement
4. **Statement:**
   - **Inspect:** Originates from a country in
   - **Country codes:** Select countries to block (e.g., XX, YY)
   - **Action:** Block
5. Click **Add rule**

#### Step 17: Add AWS Managed Rules
1. Click **Add rules** → **Add managed rule groups**
2. Expand **AWS managed rule groups** (free)
3. Select:
   - **Core rule set** (protects against common threats)
   - **Known bad inputs** (blocks malformed requests)
4. Click **Add rules**

#### Step 18: Set Default Action
1. Scroll to **Default web ACL action for requests that don't match any rules**
2. Select **Allow**
3. Click **Next**

#### Step 19: Configure CloudWatch Metrics
1. **Request sampling:** Enable
2. **CloudWatch metrics:** Enable
3. Click **Next**

#### Step 20: Review and Create
1. Review all rules
2. Click **Create web ACL**

#### Step 21: Associate WAF with CloudFront
1. Still in WAF, click **Associated AWS resources** tab
2. Click **Add AWS resources**
3. **Resource type:** CloudFront distribution
4. Select your analytics dashboard distribution
5. Click **Add**

---

### Verification Checklist

- [ ] S3 bucket created with dashboard files
- [ ] CloudFront distribution deployed and accessible
- [ ] Origin Access Identity configured
- [ ] Dashboard loads via CloudFront URL
- [ ] CSS and JavaScript files load correctly
- [ ] Custom error page displays for 404 errors
- [ ] Signed URLs generated successfully
- [ ] Unsigned URLs blocked after enabling restriction
- [ ] WAF Web ACL created with rate limiting
- [ ] WAF associated with CloudFront distribution
- [ ] Rate limiting tested (optional)

### Performance Metrics

**Global Latency:**
- US East: 10-20 ms
- Europe: 50-80 ms
- Asia: 100-150 ms
- Australia: 150-200 ms

**Without CloudFront (Direct S3):**
- All regions from us-east-1: 200-400 ms average

**Cache Hit Ratio:** Typically 85-95% after warming up

---

### Cleanup

1. **Remove WAF Association:**
   - WAF → Web ACLs → Select ACL → Associated AWS resources → Remove

2. **Delete WAF Web ACL:**
   - WAF → Web ACLs → Delete

3. **Delete CloudFront Distribution:**
   - CloudFront → Distributions → Disable → Wait → Delete

4. **Delete CloudFront Key Group:**
   - CloudFront → Key groups → Delete

5. **Delete Public Key:**
   - CloudFront → Public keys → Delete

6. **Delete Lambda Function:**
   - Lambda → Functions → Delete

7. **Delete Lambda Layer:**
   - Lambda → Layers → Delete

8. **Empty and Delete S3 Bucket:**
   - S3 → Empty bucket → Delete bucket

9. **Delete Origin Access Identity:**
   - CloudFront → Origin access → Delete

---

## Exercise 10.3: Hybrid Data Lake with Direct Connect

### Introduction
Establish a dedicated network connection between on-premises infrastructure and AWS for high-speed, consistent data transfer to your data lake.

**What You'll Build:**
- Virtual Private Gateway for VPN connectivity
- Direct Connect Gateway (simulated)
- VPN backup connection
- Performance testing for large data transfers

**Duration:** 120 minutes  
**Cost:** Direct Connect port: $0.30/hour (~$216/month) + $0.02/GB data transfer  
**Note:** This exercise simulates Direct Connect setup. Actual Direct Connect requires physical connection provisioning through AWS Direct Connect partners.

---

### Part 1: Set Up Virtual Private Gateway

#### Step 1: Create Virtual Private Gateway
1. Navigate to **VPC**
2. In the left sidebar, click **Virtual private gateways**
3. Click **Create virtual private gateway**
4. Configure:
   - **Name tag:** `data-lake-vgw`
   - **Autonomous System Number (ASN):** Amazon default ASN
5. Click **Create virtual private gateway**

#### Step 2: Attach VGW to VPC
1. Select the newly created VGW
2. Click **Actions** → **Attach to VPC**
3. Select your VPC (either `data-lake-vpc` or create new one)
4. Click **Attach to VPC**
5. Wait for state to change to "Attached"

#### Step 3: Enable Route Propagation
1. In the left sidebar, click **Route tables**
2. Select your private subnet route table
3. Click **Route propagation** tab
4. Click **Edit route propagation**
5. Check the box next to your VGW
6. Click **Save**

---

### Part 2: Create Customer Gateway (Simulates On-Premises Router)

#### Step 4: Create Customer Gateway
1. In VPC left sidebar, click **Customer gateways**
2. Click **Create customer gateway**
3. Configure:
   - **Name:** `onprem-customer-gateway`
   - **Routing:** Dynamic
   - **BGP ASN:** 65000 (standard private ASN)
   - **IP address:** Your public IP (or use `203.0.113.1` for testing)
   - **Certificate ARN:** Leave blank
   - **Device:** Optional
4. Click **Create customer gateway**

---

### Part 3: Create VPN Connection (Backup for Direct Connect)

#### Step 5: Create Site-to-Site VPN
1. In VPC left sidebar, click **Site-to-Site VPN connections**
2. Click **Create VPN connection**
3. Configure:
   - **Name tag:** `data-lake-backup-vpn`
   - **Target gateway type:** Virtual private gateway
   - **Virtual private gateway:** Select `data-lake-vgw`
   - **Customer gateway:** Existing
   - **Customer gateway ID:** Select `onprem-customer-gateway`
   - **Routing options:** Dynamic (requires BGP)
   - **Enable acceleration:** No (costs extra)
   - **Tunnel inside IP version:** IPv4
4. **Tunnel options:** Leave as default or customize:
   - Inside IPv4 CIDR for Tunnel 1: `169.254.10.0/30`
   - Pre-shared key: Auto-generate
   - Inside IPv4 CIDR for Tunnel 2: `169.254.11.0/30`
   - Pre-shared key: Auto-generate
5. Click **Create VPN connection**
6. Wait 5-10 minutes for creation

#### Step 6: Download VPN Configuration
1. Select your VPN connection
2. Click **Download configuration**
3. **Vendor:** Generic
4. Click **Download**
5. Save the configuration file (contains tunnel IPs, pre-shared keys, BGP info)

---

### Part 4: Simulate Direct Connect Gateway Setup

**Note:** Actual Direct Connect requires:
- Selecting a Direct Connect location
- Ordering a connection through AWS or a partner
- Physical cable installation (1-4 weeks)
- LOA-CFA (Letter of Authorization) document

For this exercise, we'll configure the AWS side that would be used with Direct Connect.

#### Step 7: Create Direct Connect Gateway
1. Navigate to **Direct Connect**
2. In the left sidebar, click **Direct Connect gateways**
3. Click **Create Direct Connect gateway**
4. Configure:
   - **Name:** `data-lake-dxgw`
   - **Amazon side ASN:** 64512 (private ASN)
5. Click **Create Direct Connect gateway**

#### Step 8: Associate Direct Connect Gateway with VGW
1. Select your newly created Direct Connect Gateway
2. Click **Gateway associations** tab
3. Click **Associate gateway**
4. Configure:
   - **Gateways:** Virtual private gateway
   - **Virtual private gateway:** Select `data-lake-vgw`
   - **Allowed prefixes:** Click **Add prefix**
     - Enter: `10.0.0.0/16` (your VPC CIDR)
5. Click **Associate gateway**
6. Wait for association state to become "Associated"

#### Step 9: Create Virtual Interface (Simulated)
Since we don't have an actual Direct Connect connection, we'll document the configuration:

**What you would configure:**
1. **Direct Connect** → **Virtual interfaces** → **Create virtual interface**
2. **Type:** Private
3. **Virtual interface name:** `data-lake-vif`
4. **Connection:** Select your Direct Connect connection ID
5. **Virtual interface owner:** My AWS account
6. **Direct Connect gateway:** `data-lake-dxgw`
7. **VLAN:** 100 (provided by Direct Connect partner)
8. **BGP ASN:** 65000 (your on-premises ASN)
9. **Router peer IP:** 169.254.1.1/30 (AWS side)
10. **Your router peer IP:** 169.254.1.2/30 (on-prem side)
11. **BGP authentication key:** Auto-generate

---

### Part 5: Test Data Transfer Performance

Since we can't actually test Direct Connect without physical setup, we'll test VPN performance and document expected Direct Connect performance.

#### Step 10: Create S3 Bucket for Data Transfer Testing
1. Navigate to **S3**
2. Click **Create bucket**
3. Configure:
   - **Bucket name:** `hybrid-datalake-transfer-test-[unique-id]`
   - **Region:** Same as your VPC
   - **Block all public access:** Keep checked
4. Click **Create bucket**

#### Step 11: Create EC2 Instance for Transfer Testing
1. Navigate to **EC2**
2. Click **Launch instance**
3. Configure:
   - **Name:** `transfer-test-instance`
   - **AMI:** Amazon Linux 2023
   - **Instance type:** t3.xlarge (for better network performance)
   - **Key pair:** Create or select existing
   - **Network settings:**
     - **VPC:** Select your VPC
     - **Subnet:** Select a private subnet
     - **Auto-assign public IP:** Disable
4. **Advanced details:**
   - **IAM instance profile:** Create role with S3FullAccess
5. Click **Launch instance**

#### Step 12: Create Test Data Transfer Script
1. Connect to your EC2 instance (via Session Manager or VPN)
2. Create a test script `test_transfer.py`:
```python
#!/usr/bin/env python3
import boto3
import time
import os
from datetime import datetime

s3 = boto3.client('s3')
BUCKET_NAME = 'hybrid-datalake-transfer-test-YOUR-ID'

def create_test_file(size_gb):
    """Create a test file of specified size in GB"""
    filename = f'test_{size_gb}gb.dat'
    size_bytes = size_gb * 1024 * 1024 * 1024
    
    print(f"Creating {size_gb} GB test file...")
    with open(filename, 'wb') as f:
        # Write in 100 MB chunks
        chunk_size = 100 * 1024 * 1024
        for i in range(size_bytes // chunk_size):
            f.write(os.urandom(chunk_size))
    
    return filename

def upload_with_metrics(filename):
    """Upload file to S3 and measure performance"""
    file_size = os.path.getsize(filename)
    
    print(f"\nStarting upload of {filename} ({file_size / 1e9:.2f} GB)...")
    start_time = time.time()
    
    # Upload with multipart
    config = boto3.s3.transfer.TransferConfig(
        multipart_threshold=100 * 1024 * 1024,  # 100 MB
        max_concurrency=10,
        multipart_chunksize=100 * 1024 * 1024
    )
    
    s3.upload_file(
        filename, 
        BUCKET_NAME, 
        f'transfers/{filename}',
        Config=config
    )
    
    duration = time.time() - start_time
    speed_mbps = (file_size * 8 / duration) / 1_000_000
    
    print(f"\n{'='*60}")
    print(f"Upload Complete!")
    print(f"File size: {file_size / 1e9:.2f} GB")
    print(f"Duration: {duration:.1f} seconds ({duration/60:.1f} minutes)")
    print(f"Average speed: {speed_mbps:.0f} Mbps ({speed_mbps/1000:.2f} Gbps)")
    print(f"Data transferred: {file_size / 1e9:.2f} GB")
    print(f"{'='*60}\n")
    
    return {
        'file_size_gb': file_size / 1e9,
        'duration_seconds': duration,
        'speed_mbps': speed_mbps
    }

if __name__ == '__main__':
    # Test with different file sizes
    results = []
    
    for size_gb in [1, 5, 10]:  # Test 1GB, 5GB, 10GB
        filename = create_test_file(size_gb)
        result = upload_with_metrics(filename)
        results.append(result)
        
        # Cleanup local file
        os.remove(filename)
        
        print(f"Waiting 10 seconds before next test...")
        time.sleep(10)
    
    # Summary
    print("\n" + "="*60)
    print("SUMMARY OF ALL TRANSFERS")
    print("="*60)
    for r in results:
        print(f"{r['file_size_gb']:.0f} GB: {r['speed_mbps']:.0f} Mbps in {r['duration_seconds']:.0f}s")
```
3. Update `BUCKET_NAME` with your actual bucket name
4. Install boto3: `pip3 install boto3`
5. Run: `python3 test_transfer.py`

#### Step 13: Analyze Results

**Expected Performance Comparison:**

| Connection Type | Bandwidth | 100 GB Transfer Time | Monthly Cost |
|----------------|-----------|---------------------|--------------|
| Internet (100 Mbps) | 100 Mbps | ~2.2 hours | $0 (data out charges apply) |
| VPN over Internet (1 Gbps) | ~500-800 Mbps | ~20-30 min | $36.50/month |
| Direct Connect (1 Gbps) | 1 Gbps | ~13 minutes | $216/month port + $2/TB |
| Direct Connect (10 Gbps) | 10 Gbps | ~1.3 minutes | $2,250/month port + $2/TB |

**Break-even Analysis:**
- Direct Connect 1 Gbps breaks even at: ~30-40 TB/month
- Direct Connect 10 Gbps breaks even at: ~300+ TB/month

---

### Verification Checklist

- [ ] Virtual Private Gateway created and attached
- [ ] Customer Gateway configured
- [ ] Site-to-Site VPN connection established
- [ ] VPN configuration downloaded
- [ ] Direct Connect Gateway created (for future use)
- [ ] DX Gateway associated with VGW
- [ ] Route propagation enabled
- [ ] S3 bucket created for testing
- [ ] EC2 instance launched for transfer testing
- [ ] Transfer performance script executed
- [ ] Performance metrics collected

### Architecture Benefits

**Consistency:**
- Dedicated bandwidth (not shared with internet traffic)
- Predictable latency and throughput
- No jitter or packet loss

**Security:**
- Private connectivity (not over public internet)
- Data doesn't traverse ISP networks
- Can combine with encryption for compliance

**Cost (for large transfers):**
- More economical than VPN for >30 TB/month
- No NAT Gateway costs
- Reduced data transfer costs

**Use Cases:**
- Daily ETL jobs transferring TBs
- Hybrid cloud architectures
- Disaster recovery with large datasets
- Real-time replication of on-premises databases

---

### Cleanup

1. **Delete EC2 Instance:**
   - EC2 → Instances → Terminate

2. **Empty and Delete S3 Bucket:**
   - S3 → Empty → Delete

3. **Delete VPN Connection:**
   - VPC → Site-to-Site VPN connections → Delete

4. **Delete Direct Connect Gateway Association:**
   - Direct Connect → Gateways → Gateway associations → Disassociate

5. **Delete Direct Connect Gateway:**
   - Direct Connect → Gateways → Delete

6. **Delete Customer Gateway:**
   - VPC → Customer gateways → Delete

7. **Detach and Delete Virtual Private Gateway:**
   - VPC → Virtual private gateways → Actions → Detach from VPC
   - Wait, then Delete

---

## Summary

### What You've Learned

**Exercise 10.1 - VPC Architecture:**
- Designed secure private networking for data lakes
- Implemented VPC endpoints for cost-effective AWS service access
- Configured Lambda in VPC with RDS and S3 access
- Saved ~78% vs NAT Gateway approach

**Exercise 10.2 - CloudFront CDN:**
- Deployed global content delivery network
- Secured S3 content with Origin Access Identity
- Implemented signed URLs for authentication
- Added WAF protection for security
- Achieved 10-200ms global latency

**Exercise 10.3 - Hybrid Connectivity:**
- Configured Virtual Private Gateway
- Set up VPN for secure connectivity
- Planned Direct Connect architecture
- Measured transfer performance
- Identified break-even points for Direct Connect

### Key Takeaways

1. **VPC Endpoints save significant costs** for services that support them (S3, DynamoDB, Secrets Manager, etc.)

2. **CloudFront dramatically improves global performance** while maintaining security with OAI and signed URLs

3. **Direct Connect is cost-effective** for large-scale regular data transfers (>30 TB/month)

4. **Security can be layered** - private networks, encryption, authentication, WAF protection

5. **Choose the right connectivity option:**
   - Internet: Ad-hoc, small transfers
   - VPN: Regular transfers <10 TB/month
   - Direct Connect: Consistent, large-scale transfers

### Real-World Applications

- **Financial Services:** Private, high-speed data ingestion from trading platforms
- **Healthcare:** HIPAA-compliant data transfer without internet exposure  
- **Media & Entertainment:** Global content delivery with low latency
- **E-commerce:** Real-time analytics dashboards for worldwide teams

### Cost Optimization Tips

1. Use S3 Transfer Acceleration for sporadic large transfers instead of Direct Connect
2. Leverage CloudFront regional edge caches for frequently accessed data
3. Implement VPC endpoints for all supported services
4. Use Direct Connect for predictable, high-volume workloads
5. Enable CloudFront compression to reduce data transfer costs

### Next Steps

- Explore AWS Transit Gateway for multi-VPC connectivity
- Implement AWS PrivateLink for SaaS integration
- Set up Route 53 with health checks and failover
- Configure CloudFront Lambda@Edge for dynamic content
- Design multi-region architectures with Route 53 routing policies

---

## Exam Alignment

**DEA-C01 Coverage:**

**Domain 1: Data Ingestion and Transformation (34%)**
- VPC configuration for secure data ingestion
- Hybrid connectivity patterns
- Network optimization for data transfer

**Domain 3: Data Operations and Support (22%)**
- VPC endpoint configuration
- Network troubleshooting
- Performance monitoring and optimization

**Domain 4: Data Security and Governance (24%)**
- Private network architecture
- Encryption in transit
- Access control with signed URLs
- WAF security rules

### Practice Questions

Test your knowledge with these scenario-based questions in the next module!

---

**Total Cost Summary:**
- Exercise 10.1: $0 (Free tier eligible)
- Exercise 10.2: ~$10/month (moderate traffic)
- Exercise 10.3: $0 for setup (Direct Connect ~$216/month if implemented)

**Estimated Time Investment:** 4-5 hours

---

*This guide is part of the AWS Data Engineer Associate certification preparation series.*

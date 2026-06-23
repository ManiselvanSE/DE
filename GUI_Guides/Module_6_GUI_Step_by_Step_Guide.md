# MODULE 6: Containers - GUI Step-by-Step Hands-On Guide

## Table of Contents
- [Exercise 6.1: Containerized ETL with ECS Fargate](#exercise-61-containerized-etl-with-ecs-fargate)
- [Exercise 6.2: Kubernetes Pipeline with Amazon EKS](#exercise-62-kubernetes-pipeline-with-amazon-eks)
- [Exercise 6.3: Cost-Optimized Batch with ECS Spot](#exercise-63-cost-optimized-batch-with-ecs-spot)

---

# Exercise 6.1: Containerized ETL Pipeline with ECS Fargate

**Duration:** 2-3 hours | **Estimated Cost:** ~$10-15 for lab session

## 🎯 Learning Objectives
- Build containerized ETL pipeline with Docker
- Deploy to ECS Fargate (serverless containers)
- Process CSV data and load to Redshift
- Schedule tasks with EventBridge
- Monitor container execution

---

## PART 1: Create ETL Container

### Step 1: Create Project Directory
```bash
mkdir sales-etl
cd sales-etl
```

### Step 2: Create ETL Python Script
Create file: `etl_processor.py`

```python
#!/usr/bin/env python3
import boto3
import pandas as pd
import psycopg2
import os
import sys
from datetime import datetime

s3 = boto3.client('s3')
secrets = boto3.client('secretsmanager')

def get_db_credentials():
    """Retrieve Redshift credentials from Secrets Manager"""
    secret_name = os.environ.get('DB_SECRET_NAME', 'redshift/credentials')
    
    try:
        response = secrets.get_secret_value(SecretId=secret_name)
        import json
        creds = json.loads(response['SecretString'])
        return creds
    except Exception as e:
        print(f"Error retrieving secrets: {e}")
        # Fallback to environment variables
        return {
            'host': os.environ.get('REDSHIFT_HOST'),
            'port': os.environ.get('REDSHIFT_PORT', '5439'),
            'database': os.environ.get('REDSHIFT_DB', 'dev'),
            'username': os.environ.get('REDSHIFT_USER', 'admin'),
            'password': os.environ.get('REDSHIFT_PASSWORD')
        }

def download_from_s3(bucket, key):
    """Download CSV file from S3"""
    local_file = '/tmp/sales_data.csv'
    print(f"Downloading s3://{bucket}/{key}...")
    s3.download_file(bucket, key, local_file)
    return local_file

def transform_data(csv_file):
    """Transform CSV data with pandas"""
    print("Reading CSV...")
    df = pd.read_csv(csv_file)
    
    print(f"Loaded {len(df)} rows, {len(df.columns)} columns")
    
    # Data transformations
    # 1. Calculate total amount
    df['total_amount'] = df['quantity'] * df['unit_price']
    
    # 2. Add processing timestamp
    df['processed_at'] = datetime.now()
    
    # 3. Parse dates
    df['order_date'] = pd.to_datetime(df['order_date'])
    
    # 4. Data quality checks
    initial_count = len(df)
    df = df.dropna(subset=['order_id', 'customer_id'])  # Remove null IDs
    df = df[df['quantity'] > 0]  # Remove invalid quantities
    df = df[df['unit_price'] > 0]  # Remove invalid prices
    
    print(f"After quality checks: {len(df)} rows ({initial_count - len(df)} removed)")
    
    # 5. Optimize data types
    df['order_id'] = df['order_id'].astype('int32')
    df['quantity'] = df['quantity'].astype('int16')
    
    return df

def load_to_redshift(df, creds):
    """Load data to Redshift"""
    print("Connecting to Redshift...")
    
    conn = psycopg2.connect(
        host=creds['host'],
        port=creds['port'],
        database=creds['database'],
        user=creds['username'],
        password=creds['password']
    )
    cursor = conn.cursor()
    
    # Create table if not exists
    create_table_sql = """
    CREATE TABLE IF NOT EXISTS sales_transactions (
        order_id INT,
        customer_id VARCHAR(50),
        product_id VARCHAR(50),
        quantity INT,
        unit_price DECIMAL(10,2),
        order_date DATE,
        region VARCHAR(50),
        total_amount DECIMAL(12,2),
        processed_at TIMESTAMP
    )
    SORTKEY(order_date)
    DISTKEY(customer_id);
    """
    
    cursor.execute(create_table_sql)
    conn.commit()
    print("✓ Table created/verified")
    
    # Insert data (batch insert)
    print(f"Inserting {len(df)} rows...")
    
    insert_sql = """
    INSERT INTO sales_transactions 
    (order_id, customer_id, product_id, quantity, unit_price, order_date, region, total_amount, processed_at)
    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
    """
    
    # Convert dataframe to list of tuples
    data_tuples = list(df.itertuples(index=False, name=None))
    
    # Batch insert
    cursor.executemany(insert_sql, data_tuples)
    conn.commit()
    
    print(f"✓ Inserted {cursor.rowcount} rows")
    
    # Verify
    cursor.execute("SELECT COUNT(*) FROM sales_transactions")
    total_rows = cursor.fetchone()[0]
    print(f"✓ Total rows in table: {total_rows}")
    
    cursor.close()
    conn.close()

def main():
    print("=== Sales ETL Process Started ===")
    print(f"Timestamp: {datetime.now()}")
    
    # Get parameters from environment
    bucket = os.environ.get('S3_BUCKET')
    key = os.environ.get('S3_KEY', 'input/sales-data.csv')
    
    if not bucket:
        print("ERROR: S3_BUCKET environment variable required")
        sys.exit(1)
    
    try:
        # Step 1: Extract
        csv_file = download_from_s3(bucket, key)
        
        # Step 2: Transform
        df = transform_data(csv_file)
        
        # Step 3: Load
        creds = get_db_credentials()
        load_to_redshift(df, creds)
        
        print("\n=== ETL Process Completed Successfully ===")
        
    except Exception as e:
        print(f"\n!!! ETL Process Failed !!!")
        print(f"Error: {str(e)}")
        import traceback
        traceback.print_exc()
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### Step 3: Create Dockerfile
Create file: `Dockerfile`

```dockerfile
FROM python:3.11-slim

# Install PostgreSQL client
RUN apt-get update && apt-get install -y \
    postgresql-client \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install Python packages
RUN pip install --no-cache-dir \
    boto3 \
    pandas \
    pyarrow \
    psycopg2-binary \
    requests

# Copy ETL script
COPY etl_processor.py /app/etl_processor.py
RUN chmod +x /app/etl_processor.py

# Set working directory
WORKDIR /app

# Run ETL script
CMD ["python", "/app/etl_processor.py"]
```

### Step 4: Build Docker Image
```bash
docker build -t sales-etl:latest .

# Verify
docker images | grep sales-etl
```

---

## PART 2: Push to Amazon ECR

### Step 5: Create ECR Repository
1. **ECR Console** → **"Repositories"** → **"Create repository"**
2. **Repository name:** `sales-etl`
3. **Scan on push:** ✅ Enable
4. Click **"Create repository"**

### Step 6: Push Image to ECR
1. Click on repository: `sales-etl`
2. Click **"View push commands"**
3. Run the commands in your terminal:

```bash
# Authenticate
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin [ACCOUNT-ID].dkr.ecr.us-east-1.amazonaws.com

# Tag image
docker tag sales-etl:latest [ACCOUNT-ID].dkr.ecr.us-east-1.amazonaws.com/sales-etl:latest

# Push
docker push [ACCOUNT-ID].dkr.ecr.us-east-1.amazonaws.com/sales-etl:latest
```

### Step 7: Note ECR Image URI
Copy the image URI: `[ACCOUNT-ID].dkr.ecr.us-east-1.amazonaws.com/sales-etl:latest`

---

## PART 3: Create Redshift Cluster (Target Database)

### Step 8: Create Redshift Cluster
1. **Redshift Console** → **"Create cluster"**
2. **Cluster identifier:** `sales-etl-target`
3. **Node type:** **dc2.large**
4. **Nodes:** **2**
5. **Database:** `salesdb`
6. **Master username:** `admin`
7. **Password:** Create strong password
8. **VPC:** Default VPC
9. **Publicly accessible:** **Yes** (for testing)
10. Click **"Create cluster"**
11. **Wait for "Available" status** (~10 minutes)

### Step 9: Configure Security Group
1. Click on cluster → **"Properties"** → Security group link
2. **"Inbound rules"** → **"Edit"**
3. **Add rule:** Type: **Redshift**, Source: **Anywhere-IPv4** (for lab only!)
4. **Save rules**

### Step 10: Get Redshift Endpoint
1. Cluster details → **"General information"**
2. Copy **"Endpoint"** (without port)
   - Example: `sales-etl-target.xxxxx.us-east-1.redshift.amazonaws.com`

### Step 11: Store Credentials in Secrets Manager
1. **Secrets Manager Console** → **"Store a new secret"**
2. **Secret type:** **"Other type of secret"**
3. **Key/value pairs:**
   - `host`: [Redshift endpoint]
   - `port`: `5439`
   - `database`: `salesdb`
   - `username`: `admin`
   - `password`: [your password]
4. **Secret name:** `redshift/sales-etl-credentials`
5. Click **"Next"** → **"Next"** → **"Store"**

---

## PART 4: Create ECS Cluster and Task Definition

### Step 12: Navigate to ECS Console
1. AWS Console → **"ECS"** (Elastic Container Service)
2. Click **"Get started"** (if first time) OR **"Clusters"**

### Step 13: Create ECS Cluster
1. Click **"Create cluster"**
2. **Cluster name:** `sales-etl-cluster`
3. **Infrastructure:** Leave **"AWS Fargate (serverless)"** selected
4. Click **"Create"**

### Step 14: Create IAM Execution Role
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** **"AWS service"**
3. **Use case:** **"Elastic Container Service"** → **"Elastic Container Service Task"**
4. Click **"Next"**
5. Attach policies:
   - ✅ **AmazonECSTaskExecutionRolePolicy**
6. **Role name:** `ECSTaskExecutionRole`
7. **Create role**

### Step 15: Create IAM Task Role (for S3/Redshift access)
1. **Create role** again
2. **Trusted entity:** **"AWS service"** → **"Elastic Container Service Task"**
3. Attach policies:
   - ✅ **AmazonS3ReadOnlyAccess**
   - ✅ **SecretsManagerReadWrite**
4. **Role name:** `SalesETLTaskRole`
5. **Create role**

### Step 16: Create Task Definition
1. **ECS Console** → **"Task definitions"** → **"Create new task definition"**
2. **Task definition family:** `sales-etl-task`

### Step 17: Configure Task Definition - Infrastructure
1. **Launch type:** **"AWS Fargate"**
2. **Operating system:** **"Linux/X86_64"**
3. **CPU:** **"1 vCPU"** (or **"2 vCPU"** for larger files)
4. **Memory:** **"4 GB"**
5. **Task role:** Select `SalesETLTaskRole`
6. **Task execution role:** Select `ECSTaskExecutionRole`

### Step 18: Configure Container
1. **Container name:** `sales-etl-container`
2. **Image URI:** Paste ECR image URI from Step 7
3. **Essential container:** ✅ Yes

### Step 19: Configure Environment Variables
Scroll to **"Environment variables"**:

1. **Key:** `S3_BUCKET` | **Value:** `your-bucket-name`
2. **Key:** `S3_KEY` | **Value:** `input/sales-data-2024-06.csv`
3. **Key:** `DB_SECRET_NAME` | **Value:** `redshift/sales-etl-credentials`

### Step 20: Configure Logging
1. Scroll to **"Logging"**
2. ✅ Check **"Use log collection"**
3. **Log driver:** **"awslogs"**
4. **Log group name:** `/ecs/sales-etl`
5. **Auto-configure CloudWatch Logs:** ✅ Enable

### Step 21: Create Task Definition
1. Scroll to bottom → Click **"Create"**

---

## PART 5: Prepare Sample Data and Run Task

### Step 22: Create S3 Bucket and Upload Data
1. **S3 Console** → **"Create bucket"**
2. **Name:** `ecs-sales-data-[account-id]`
3. Create folders: `input/` and `processed/`

### Step 23: Create Sample CSV
Create file locally: `sales-data-2024-06.csv`

```csv
order_id,customer_id,product_id,quantity,unit_price,order_date,region
10001,CUST001,PROD123,5,29.99,2024-06-01,US-East
10002,CUST002,PROD456,2,149.99,2024-06-02,US-West
10003,CUST003,PROD789,1,599.99,2024-06-03,EU-West
10004,CUST001,PROD123,3,29.99,2024-06-04,US-East
10005,CUST004,PROD456,1,149.99,2024-06-05,APAC
10006,CUST005,PROD999,10,9.99,2024-06-06,US-Central
10007,CUST002,PROD123,2,29.99,2024-06-07,US-West
10008,CUST006,PROD789,1,599.99,2024-06-08,EU-Central
10009,CUST007,PROD111,7,39.99,2024-06-09,US-East
10010,CUST008,PROD222,4,79.99,2024-06-10,APAC
```

Upload to: `s3://ecs-sales-data-[id]/input/sales-data-2024-06.csv`

### Step 24: Run ECS Task
1. **ECS Console** → **"Clusters"** → `sales-etl-cluster`
2. Click **"Run new task"** button

### Step 25: Configure Task Run
1. **Compute options:** **"Launch type"**
2. **Launch type:** **"FARGATE"**
3. **Platform version:** **"LATEST"**
4. **Task definition:**
   - **Family:** `sales-etl-task`
   - **Revision:** Latest (1)
5. **Desired tasks:** `1`

### Step 26: Configure Networking
1. **VPC:** Select default VPC
2. **Subnets:** Select 2-3 subnets
3. **Security group:** Create new OR use default
   - **Outbound:** Allow all (needs internet for Redshift/S3)
4. **Public IP:** **"Enabled"** (Fargate needs this for internet access)

### Step 27: Launch Task
1. Click **"Create"** button
2. You'll see the task in **"Tasks"** tab

### Step 28: Monitor Task Execution
1. **Status** changes:
   - **PROVISIONING** → Allocating resources
   - **PENDING** → Pulling container image
   - **RUNNING** → Container executing
   - **STOPPED** → Completed (success or failure)

This takes 2-5 minutes.

### Step 29: View Task Logs
1. Click on the task (Task ID)
2. Go to **"Logs"** tab
3. You should see:
   ```
   === Sales ETL Process Started ===
   Downloading s3://ecs-sales-data-123/input/sales-data-2024-06.csv...
   Reading CSV...
   Loaded 10 rows, 7 columns
   After quality checks: 10 rows (0 removed)
   Connecting to Redshift...
   ✓ Table created/verified
   Inserting 10 rows...
   ✓ Inserted 10 rows
   ✓ Total rows in table: 10
   === ETL Process Completed Successfully ===
   ```

### Step 30: Verify Data in Redshift
Connect to Redshift:
```bash
psql -h sales-etl-target.xxxxx.us-east-1.redshift.amazonaws.com -U admin -d salesdb -p 5439
```

Query the data:
```sql
SELECT * FROM sales_transactions ORDER BY order_id LIMIT 10;

SELECT 
    region,
    COUNT(*) as orders,
    SUM(total_amount) as revenue
FROM sales_transactions
GROUP BY region
ORDER BY revenue DESC;
```

---

## PART 6: Schedule with EventBridge

### Step 31: Create EventBridge Rule
1. **EventBridge Console** → **"Rules"** → **"Create rule"**
2. **Name:** `sales-etl-daily-schedule`
3. **Event bus:** `default`
4. **Rule type:** **"Schedule"**
5. Click **"Next"**

### Step 32: Configure Schedule
1. **Schedule pattern:** Select **"A schedule that runs at a regular rate"**
2. **Rate expression:** `rate(1 day)` OR
3. **Cron expression:** `cron(0 2 * * ? *)` (2 AM daily)
4. Click **"Next"**

### Step 33: Select Target
1. **Target types:** **"AWS service"**
2. **Select a target:** Choose **"ECS task"**
3. **Cluster:** Select `sales-etl-cluster`
4. **Task definition family:** `sales-etl-task`
5. **Revision:** **LATEST**

### Step 34: Configure ECS Task Launch
1. **Launch type:** **"FARGATE"**
2. **Platform version:** **LATEST**
3. **Subnets:** Select same as before
4. **Security groups:** Select same
5. **Auto-assign public IP:** **ENABLED**

### Step 35: Create IAM Role for EventBridge
1. **Execution role:** Select **"Create a new role for this specific resource"**
2. This auto-creates `Amazon_EventBridge_Invoke_ECS_*` role

### Step 36: Finalize Rule
1. Click **"Next"** → **"Next"**
2. Review settings
3. Click **"Create rule"**

**Your ETL now runs automatically every day at 2 AM!**

---

## ✅ Exercise 6.1 Completion Checklist

- [ ] Created Docker container with Python ETL script
- [ ] Built and tested Docker image locally
- [ ] Created ECR repository and pushed image
- [ ] Created Redshift cluster as target database
- [ ] Stored database credentials in Secrets Manager
- [ ] Created ECS Fargate cluster
- [ ] Created IAM execution and task roles
- [ ] Registered ECS task definition with container config
- [ ] Prepared sample CSV data in S3
- [ ] Ran ECS task manually
- [ ] Monitored task execution via CloudWatch Logs
- [ ] Verified data loaded to Redshift
- [ ] Created EventBridge schedule rule for daily execution
- [ ] Configured automated task launches

**🎉 Congratulations!** You've completed Exercise 6.1!

---

# Exercise 6.2: Kubernetes-Based Pipeline with Amazon EKS

**Duration:** 4-5 hours | **Estimated Cost:** ~$30-50 for lab session

## 🎯 Learning Objectives
- Create Amazon EKS cluster
- Deploy Apache Airflow on Kubernetes
- Create DAG for data pipeline orchestration
- Use Kubernetes auto-scaling

---

## PART 1: Create EKS Cluster

### Step 1: Install kubectl (Kubernetes CLI)
**On macOS:**
```bash
brew install kubectl
```

**On Windows:**
Download from: https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/

**On Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Verify:
```bash
kubectl version --client
```

### Step 2: Install eksctl (EKS CLI)
**On macOS:**
```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

**On Windows/Linux:**
Follow: https://eksctl.io/installation/

Verify:
```bash
eksctl version
```

### Step 3: Create EKS Cluster via Console
1. **EKS Console** → **"Clusters"** → **"Create cluster"**
2. **Name:** `data-pipeline-cluster`
3. **Kubernetes version:** Select latest (e.g., 1.28)
4. **Cluster service role:** Click **"Create role"** OR select existing

### Step 4: Create Cluster Service Role
(If creating new role):
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** **"AWS service"** → **"EKS - Cluster"**
3. Attach policy: **AmazonEKSClusterPolicy** (auto-selected)
4. **Role name:** `EKSClusterRole`
5. **Create role**

Go back to EKS cluster creation, select `EKSClusterRole`.

### Step 5: Configure Networking
1. **VPC:** Select default VPC OR create new
2. **Subnets:** Select at least 2 subnets in different AZs
3. **Security groups:** Select default
4. **Cluster endpoint access:** **"Public and private"** (for testing)
5. Click **"Next"**

### Step 6: Configure Logging (Optional)
1. Enable: **API server**, **Audit**, **Authenticator**
2. Click **"Next"**

### Step 7: Review and Create
1. Review settings
2. Click **"Create"**
3. **Wait 10-15 minutes** for cluster to become **ACTIVE**

### Step 8: Configure kubectl
Once cluster is **ACTIVE**:

```bash
# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name data-pipeline-cluster

# Verify connection
kubectl get svc
```

You should see: `kubernetes ClusterIP ...`

### Step 9: Create Node Group
1. In EKS console, click on cluster: `data-pipeline-cluster`
2. Go to **"Compute"** tab
3. Click **"Add node group"**

### Step 10: Configure Node Group
1. **Name:** `airflow-workers`
2. **Node IAM role:** Click **"Create role"** OR select existing

### Step 11: Create Node IAM Role
(If creating new):
1. **IAM** → **"Roles"** → **"Create role"**
2. **Use case:** **"EC2"**
3. Attach policies:
   - ✅ **AmazonEKSWorkerNodePolicy**
   - ✅ **AmazonEC2ContainerRegistryReadOnly**
   - ✅ **AmazonEKS_CNI_Policy**
4. **Role name:** `EKSNodeRole`
5. **Create role**

Go back, select `EKSNodeRole`, click **"Next"**.

### Step 12: Configure Compute and Scaling
1. **AMI type:** **Amazon Linux 2**
2. **Instance types:** **t3.medium** (for lab) OR **r5.large** (production)
3. **Disk size:** **20 GB**
4. **Minimum size:** `2` nodes
5. **Maximum size:** `5` nodes
6. **Desired size:** `2` nodes
7. Click **"Next"**

### Step 13: Configure Networking for Node Group
1. **Subnets:** Select same as cluster
2. **SSH access:** Optional (select key pair if needed)
3. Click **"Next"** → **"Create"**

Wait 5-10 minutes for nodes to be **Active**.

### Step 14: Verify Nodes
```bash
kubectl get nodes
```

You should see 2 nodes in **Ready** state.

---

## PART 2: Deploy Apache Airflow with Helm

### Step 15: Install Helm
**On macOS:**
```bash
brew install helm
```

**On Windows/Linux:**
Follow: https://helm.sh/docs/intro/install/

Verify:
```bash
helm version
```

### Step 16: Add Airflow Helm Repository
```bash
helm repo add apache-airflow https://airflow.apache.org
helm repo update
```

### Step 17: Create Namespace for Airflow
```bash
kubectl create namespace airflow
```

### Step 18: Create Airflow Values File
Create file: `airflow-values.yaml`

```yaml
# Airflow Helm values
executor: "CeleryExecutor"  # Distributed task execution

# Web server
webserver:
  replicas: 1
  service:
    type: LoadBalancer  # Expose web UI publicly

# Scheduler
scheduler:
  replicas: 1

# Workers
workers:
  replicas: 3  # Start with 3 workers
  persistence:
    enabled: false
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "1000m"
      memory: "2Gi"
  # Auto-scaling
  keda:
    enabled: true
    minReplicaCount: 2
    maxReplicaCount: 10

# PostgreSQL (metadata DB)
postgresql:
  enabled: true
  postgresqlPassword: "airflow123"
  persistence:
    enabled: true
    size: 10Gi

# Redis (message broker for Celery)
redis:
  enabled: true
  password: "redis123"

# DAGs
dags:
  gitSync:
    enabled: false  # We'll mount DAGs directly for now

# Enable RBAC
rbac:
  create: true

# Default Airflow config
config:
  AIRFLOW__CORE__LOAD_EXAMPLES: 'False'  # Don't load example DAGs
  AIRFLOW__WEBSERVER__EXPOSE_CONFIG: 'True'
```

### Step 19: Install Airflow
```bash
helm install airflow apache-airflow/airflow \
  --namespace airflow \
  --values airflow-values.yaml \
  --timeout 10m
```

This takes 5-10 minutes to deploy all components.

### Step 20: Monitor Deployment
```bash
kubectl get pods -n airflow --watch
```

Wait for all pods to show **Running** status:
- `airflow-webserver-*`
- `airflow-scheduler-*`
- `airflow-worker-*` (3 replicas)
- `airflow-postgresql-*`
- `airflow-redis-*`

Press `Ctrl+C` to stop watching.

### Step 21: Get Airflow Web UI URL
```bash
kubectl get svc -n airflow airflow-webserver
```

Look for **EXTERNAL-IP** under `airflow-webserver` (LoadBalancer type).

**Example:** `a1234567890abcdef-1234567890.us-east-1.elb.amazonaws.com`

### Step 22: Access Airflow Web UI
1. Open browser
2. Go to: `http://[EXTERNAL-IP]:8080`
3. **Username:** `admin`
4. **Password:** Get it with:
   ```bash
   kubectl get secret --namespace airflow airflow-webserver-secret -o jsonpath="{.data.webserver-secret-key}" | base64 --decode
   ```
   Default is usually: `admin`

---

## PART 3: Create Airflow DAG

### Step 23: Create DAG File
Create file: `daily_sales_pipeline.py`

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.operators.emr import (
    EmrCreateJobFlowOperator,
    EmrAddStepsOperator,
    EmrTerminateJobFlowOperator
)
from airflow.providers.amazon.aws.sensors.emr import EmrStepSensor
from airflow.providers.amazon.aws.operators.s3 import S3CreateObjectOperator
from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
from datetime import datetime, timedelta
import boto3

default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2024, 6, 1),
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'daily_sales_pipeline',
    default_args=default_args,
    description='Daily sales data processing pipeline',
    schedule_interval='0 2 * * *',  # 2 AM daily
    catchup=False,
    tags=['sales', 'etl', 'production']
)

def validate_s3_data(**context):
    """Check if source data exists in S3"""
    s3 = boto3.client('s3')
    bucket = 'your-data-bucket'
    prefix = 'raw/sales/{{ ds }}/'  # ds = execution date
    
    response = s3.list_objects_v2(Bucket=bucket, Prefix=prefix)
    
    if 'Contents' not in response:
        raise ValueError(f"No files found in s3://{bucket}/{prefix}")
    
    file_count = len(response['Contents'])
    print(f"✓ Found {file_count} files to process")
    return file_count

validate_task = PythonOperator(
    task_id='validate_s3_data',
    python_callable=validate_s3_data,
    dag=dag
)

# EMR cluster configuration
emr_cluster_config = {
    'Name': 'sales-processing-{{ ds }}',
    'ReleaseLabel': 'emr-7.0.0',
    'Applications': [{'Name': 'Spark'}, {'Name': 'Hadoop'}],
    'Instances': {
        'InstanceGroups': [
            {
                'Name': 'Master',
                'Market': 'ON_DEMAND',
                'InstanceRole': 'MASTER',
                'InstanceType': 'm5.xlarge',
                'InstanceCount': 1,
            },
            {
                'Name': 'Core',
                'Market': 'ON_DEMAND',
                'InstanceRole': 'CORE',
                'InstanceType': 'r5.2xlarge',
                'InstanceCount': 3,
            }
        ],
        'Ec2SubnetId': 'subnet-xxxxx',  # Replace with your subnet
        'KeepJobFlowAliveWhenNoSteps': True,
        'TerminationProtected': False,
    },
    'JobFlowRole': 'EMR_EC2_DefaultRole',
    'ServiceRole': 'EMR_DefaultRole',
    'LogUri': 's3://your-bucket/emr-logs/'
}

create_emr_cluster = EmrCreateJobFlowOperator(
    task_id='create_emr_cluster',
    job_flow_overrides=emr_cluster_config,
    dag=dag
)

# Spark processing step
spark_step = [
    {
        'Name': 'Process Sales Data',
        'ActionOnFailure': 'CONTINUE',
        'HadoopJarStep': {
            'Jar': 'command-runner.jar',
            'Args': [
                'spark-submit',
                '--deploy-mode', 'cluster',
                '--conf', 'spark.sql.adaptive.enabled=true',
                's3://your-bucket/scripts/process_sales.py',
                '--input', 's3://your-bucket/raw/sales/{{ ds }}/',
                '--output', 's3://your-bucket/processed/sales/{{ ds }}/'
            ]
        }
    }
]

add_spark_step = EmrAddStepsOperator(
    task_id='add_spark_step',
    job_flow_id="{{ task_instance.xcom_pull(task_ids='create_emr_cluster', key='return_value') }}",
    steps=spark_step,
    dag=dag
)

wait_for_spark = EmrStepSensor(
    task_id='wait_for_spark_step',
    job_flow_id="{{ task_instance.xcom_pull(task_ids='create_emr_cluster', key='return_value') }}",
    step_id="{{ task_instance.xcom_pull(task_ids='add_spark_step')[0] }}",
    timeout=3600,  # 1 hour
    poke_interval=60,  # Check every minute
    dag=dag
)

load_to_redshift = S3ToRedshiftOperator(
    task_id='load_to_redshift',
    s3_bucket='your-bucket',
    s3_key='processed/sales/{{ ds }}/',
    schema='public',
    table='sales_transactions',
    copy_options=['PARQUET', 'TRUNCATECOLUMNS'],
    redshift_conn_id='redshift_default',
    dag=dag
)

def data_quality_check(**context):
    """Validate loaded data in Redshift"""
    import psycopg2
    
    conn = psycopg2.connect(
        host='your-redshift-endpoint',
        port=5439,
        database='salesdb',
        user='admin',
        password='password'
    )
    
    cursor = conn.cursor()
    cursor.execute("""
        SELECT 
            COUNT(*) as row_count,
            COUNT(DISTINCT customer_id) as unique_customers,
            SUM(total_amount) as total_revenue
        FROM sales_transactions
        WHERE order_date = '{{ ds }}'
    """)
    
    row_count, unique_customers, total_revenue = cursor.fetchone()
    
    print(f"✓ Quality Check Results:")
    print(f"  Rows: {row_count:,}")
    print(f"  Unique Customers: {unique_customers:,}")
    print(f"  Total Revenue: ${total_revenue:,.2f}")
    
    # Validation rules
    if row_count == 0:
        raise ValueError("No data loaded for this date")
    if total_revenue < 0:
        raise ValueError("Negative revenue detected")
    
    cursor.close()
    conn.close()

quality_check = PythonOperator(
    task_id='data_quality_check',
    python_callable=data_quality_check,
    dag=dag
)

terminate_emr = EmrTerminateJobFlowOperator(
    task_id='terminate_emr_cluster',
    job_flow_id="{{ task_instance.xcom_pull(task_ids='create_emr_cluster', key='return_value') }}",
    trigger_rule='all_done',  # Always terminate, even if upstream fails
    dag=dag
)

# Define task dependencies
validate_task >> create_emr_cluster >> add_spark_step >> wait_for_spark >> load_to_redshift >> quality_check >> terminate_emr
```

### Step 24: Deploy DAG to Airflow
**Option A: Copy to Airflow DAGs folder (if using persistent volume)**

```bash
# Find the scheduler pod
kubectl get pods -n airflow | grep scheduler

# Copy DAG file
kubectl cp daily_sales_pipeline.py airflow/airflow-scheduler-xxxxx:/opt/airflow/dags/

# Verify
kubectl exec -it -n airflow airflow-scheduler-xxxxx -- ls /opt/airflow/dags/
```

**Option B: Use ConfigMap (simpler for testing)**

```bash
# Create ConfigMap with DAG
kubectl create configmap airflow-dags \
  --from-file=daily_sales_pipeline.py \
  --namespace airflow

# Mount ConfigMap to Airflow pods (requires helm upgrade with custom values)
```

**Option C: Use Git-Sync (production approach)**
Configure in `airflow-values.yaml`:
```yaml
dags:
  gitSync:
    enabled: true
    repo: https://github.com/your-org/airflow-dags.git
    branch: main
    subPath: dags/
```

### Step 25: Verify DAG in Airflow UI
1. Go to Airflow web UI: `http://[EXTERNAL-IP]:8080`
2. You should see **daily_sales_pipeline** in the DAG list
3. DAG is **paused** by default (toggle ON to activate)
4. Click on the DAG name to view details

### Step 26: Trigger DAG Manually
1. Click the **▶ Play** button next to the DAG
2. Select **"Trigger DAG"**
3. Optionally add execution date
4. Click **"Trigger"**

### Step 27: Monitor DAG Execution
1. Click on the DAG → **"Graph"** view
2. Watch tasks change colors:
   - **Gray:** Not started
   - **Light green:** Queued
   - **Green:** Running
   - **Dark green:** Success
   - **Red:** Failed
3. Click on any task → **"Log"** to view details

---

## ✅ Exercise 6.2 Completion Checklist

- [ ] Installed kubectl and eksctl
- [ ] Created Amazon EKS cluster via console
- [ ] Configured kubectl to connect to cluster
- [ ] Created node group with auto-scaling (2-5 nodes)
- [ ] Verified nodes are Ready
- [ ] Installed Helm package manager
- [ ] Added Apache Airflow Helm repository
- [ ] Created Airflow values configuration file
- [ ] Deployed Airflow with CeleryExecutor
- [ ] Verified all Airflow pods running
- [ ] Accessed Airflow web UI via LoadBalancer
- [ ] Created complex DAG with 7 tasks
- [ ] Deployed DAG to Airflow
- [ ] Triggered DAG execution
- [ ] Monitored pipeline in Graph view

**🎉 Congratulations!** You've completed Exercise 6.2!

---

# Exercise 6.3: Cost-Optimized Batch with ECS Spot

**Duration:** 2-3 hours | **Estimated Cost:** ~$5-10 for lab session

## 🎯 Learning Objectives
- Process images in batch using ECS
- Leverage Spot Instances for 90% cost savings
- Configure auto-scaling based on SQS queue depth
- Build resilient batch processing system

---

## PART 1: Create Image Processing Container

### Step 1: Create Processing Script
Create file: `image_processor.py`

```python
#!/usr/bin/env python3
import boto3
import os
import json
import time
from PIL import Image
from io import BytesIO

s3 = boto3.client('s3')
sqs = boto3.client('sqs')

QUEUE_URL = os.environ.get('SQS_QUEUE_URL')
SOURCE_BUCKET = os.environ.get('SOURCE_BUCKET')
DEST_BUCKET = os.environ.get('DEST_BUCKET')

def process_image(bucket, key):
    """Download, resize, and upload image"""
    print(f"Processing: s3://{bucket}/{key}")
    
    # Download image
    response = s3.get_object(Bucket=bucket, Key=key)
    img_data = response['Body'].read()
    
    # Open with PIL
    img = Image.open(BytesIO(img_data))
    print(f"Original size: {img.size}")
    
    # Resize to thumbnail (300x300)
    img.thumbnail((300, 300), Image.LANCZOS)
    print(f"Resized to: {img.size}")
    
    # Add watermark
    from PIL import ImageDraw, ImageFont
    draw = ImageDraw.Draw(img)
    watermark = "© 2024 Company"
    draw.text((10, 10), watermark, fill=(255, 255, 255))
    
    # Convert to bytes
    buffer = BytesIO()
    img.save(buffer, format='JPEG', quality=85, optimize=True)
    buffer.seek(0)
    
    # Upload to destination
    dest_key = f"processed/{key}"
    s3.put_object(
        Bucket=DEST_BUCKET,
        Key=dest_key,
        Body=buffer,
        ContentType='image/jpeg'
    )
    
    print(f"✓ Uploaded to: s3://{DEST_BUCKET}/{dest_key}")

def main():
    print("Image processor started")
    print(f"Queue URL: {QUEUE_URL}")
    
    while True:
        # Poll SQS for messages
        response = sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=1,
            WaitTimeSeconds=20  # Long polling
        )
        
        if 'Messages' not in response:
            print("No messages, waiting...")
            continue
        
        for message in response['Messages']:
            try:
                # Parse S3 event
                body = json.loads(message['Body'])
                if 'Records' in body:
                    for record in body['Records']:
                        bucket = record['s3']['bucket']['name']
                        key = record['s3']['object']['key']
                        
                        # Process image
                        process_image(bucket, key)
                
                # Delete message from queue
                sqs.delete_message(
                    QueueUrl=QUEUE_URL,
                    ReceiptHandle=message['ReceiptHandle']
                )
                print("✓ Message processed and deleted")
                
            except Exception as e:
                print(f"Error processing message: {e}")
                # Message will return to queue after visibility timeout

if __name__ == '__main__':
    main()
```

### Step 2: Create Dockerfile
```dockerfile
FROM python:3.11-slim

# Install Pillow dependencies
RUN apt-get update && apt-get install -y \
    libjpeg-dev \
    zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python packages
RUN pip install --no-cache-dir \
    boto3 \
    Pillow

# Copy script
COPY image_processor.py /app/image_processor.py
RUN chmod +x /app/image_processor.py

WORKDIR /app

CMD ["python", "/app/image_processor.py"]
```

### Step 3: Build and Push to ECR
```bash
# Build
docker build -t image-processor:latest .

# Create ECR repo
aws ecr create-repository --repository-name image-processor

# Authenticate, tag, and push (replace with your account ID)
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin [ACCOUNT-ID].dkr.ecr.us-east-1.amazonaws.com

docker tag image-processor:latest [ACCOUNT-ID].dkr.ecr.us-east-1.amazonaws.com/image-processor:latest

docker push [ACCOUNT-ID].dkr.ecr.us-east-1.amazonaws.com/image-processor:latest
```

---

## PART 2: Create S3 Buckets and SQS Queue

### Step 4: Create S3 Buckets
1. **S3 Console** → Create 2 buckets:
   - `image-uploads-[account-id]` (source)
   - `image-processed-[account-id]` (destination)

### Step 5: Create SQS Queue
1. **SQS Console** → **"Create queue"**
2. **Type:** **"Standard"**
3. **Name:** `image-processing-queue`
4. **Visibility timeout:** `300` seconds (5 minutes)
5. **Message retention:** `4` days
6. Click **"Create queue"**
7. **Copy Queue URL** (needed for ECS task)

### Step 6: Configure S3 Event Notification
1. Go to S3 bucket: `image-uploads-[id]`
2. **Properties** → **"Event notifications"** → **"Create event notification"**
3. **Name:** `new-image-uploaded`
4. **Event types:** ✅ **"All object create events"**
5. **Prefix:** `uploads/`
6. **Suffix:** `.jpg` (or leave empty for all)
7. **Destination:** **"SQS queue"**
8. **SQS queue:** Select `image-processing-queue`
9. **Save changes**

---

## PART 3: Create ECS Service with Spot Instances

### Step 7: Create ECS Cluster
1. **ECS Console** → **"Clusters"** → **"Create cluster"**
2. **Cluster name:** `image-processing-cluster`
3. **Infrastructure:** Select **"Amazon EC2 instances"** (not Fargate, we need EC2 for Spot)
4. Click **"Create"**

### Step 8: Create Capacity Provider
1. Click on cluster → **"Capacity providers"** tab
2. Click **"Create capacity provider"**
3. **Name:** `spot-capacity-provider`
4. **Auto Scaling group:** Click **"Create new ASG"**

### Step 9: Configure Auto Scaling Group
1. **AMI:** Select **"Amazon Linux 2 (ECS optimized)"**
2. **Instance type:** **c5.xlarge** (compute optimized for image processing)
3. **Minimum:** `0`
4. **Maximum:** `20`
5. **Desired:** `2`
6. **Purchasing options:** ✅ **"Combine purchase options"**
7. **On-Demand base capacity:** `2` (minimum always-on)
8. **On-Demand percentage:** `20%` (80% Spot)
9. **Spot allocation strategy:** **"Capacity optimized"**
10. **VPC/Subnets:** Select default VPC and subnets
11. Click **"Create"**

### Step 10: Enable Managed Scaling
1. After capacity provider created
2. **Managed scaling:** ✅ **"Enabled"**
3. **Target capacity:** `100%` (maintain utilization)
4. **Minimum scaling step:** `1`
5. **Maximum scaling step:** `10`
6. **Save**

### Step 11: Create Task Definition
1. **ECS** → **"Task definitions"** → **"Create"**
2. **Family:** `image-processor-task`
3. **Launch type:** **"EC2"** (not Fargate)
4. **Network mode:** **"bridge"**
5. **Task execution role:** `ecsTaskExecutionRole`
6. **Task role:** Create role with S3 + SQS permissions

### Step 12: Add Container
1. **Container name:** `image-processor`
2. **Image:** ECR URI from Step 3
3. **Memory:** **1024 MiB**
4. **CPU:** **512**
5. **Environment variables:**
   - `SQS_QUEUE_URL`: [Your queue URL]
   - `SOURCE_BUCKET`: `image-uploads-[id]`
   - `DEST_BUCKET`: `image-processed-[id]`
6. **Logging:** Enable **awslogs**
7. **Create**

### Step 13: Create ECS Service
1. Cluster → **"Services"** → **"Create"**
2. **Compute options:** **"Capacity provider strategy"**
3. **Capacity provider:** Select `spot-capacity-provider`
4. **Weight:** `1`
5. **Base:** `0`
6. **Task definition:** `image-processor-task`
7. **Service name:** `image-processor-service`
8. **Desired tasks:** `2` (start small)

### Step 14: Configure Auto-Scaling for Service
1. **Service auto-scaling:** ✅ **"Enable"**
2. **Minimum tasks:** `2`
3. **Maximum tasks:** `50`
4. **Scaling policy:** **"Target tracking"**
5. **Metric:** **"SQS - ApproximateNumberOfMessagesVisible"**
6. **Target value:** `100` (maintain < 100 messages per task)
7. **Scale-out cooldown:** `60` seconds
8. **Scale-in cooldown:** `300` seconds
9. **Create service**

---

## PART 4: Test Batch Processing

### Step 15: Generate Test Images
Create script: `generate_test_images.py`

```python
from PIL import Image, ImageDraw
import random

for i in range(100):
    # Create random colored image
    img = Image.new('RGB', (1920, 1080), 
                    color=(random.randint(0,255), random.randint(0,255), random.randint(0,255)))
    
    draw = ImageDraw.Draw(img)
    draw.text((100, 100), f"Test Image {i+1}", fill=(255,255,255))
    
    img.save(f"test-image-{i+1:03d}.jpg", quality=95)
    print(f"Generated test-image-{i+1:03d}.jpg")

print("✓ Generated 100 test images")
```

Run:
```bash
python3 generate_test_images.py
```

### Step 16: Upload Test Images to S3
```bash
# Upload all images
for file in test-image-*.jpg; do
  aws s3 cp $file s3://image-uploads-[id]/uploads/
done

# Verify
aws s3 ls s3://image-uploads-[id]/uploads/ | wc -l
# Should show 100
```

### Step 17: Monitor SQS Queue
1. **SQS Console** → `image-processing-queue`
2. **Monitoring** tab
3. Watch **"Messages Available"** metric spike to ~100

### Step 18: Watch Auto-Scaling in Action
1. **ECS Console** → Cluster → Service
2. **Metrics** tab
3. Watch **"Desired count"** increase as queue depth grows
4. **Tasks** tab shows new tasks launching

### Step 19: Monitor Processing
1. **CloudWatch Logs** → `/ecs/image-processor-task`
2. Watch logs from multiple tasks processing images concurrently

### Step 20: Verify Processed Images
1. **S3 Console** → `image-processed-[id]` bucket
2. Navigate to `processed/uploads/` folder
3. Download a few images and verify:
   - Resized to 300x300
   - Watermark applied
   - JPEG compression

### Step 21: Check Cost Savings
1. **EC2 Console** → **"Spot Requests"**
2. View Spot instance usage
3. Compare: 
   - On-Demand c5.xlarge: **$0.17/hour**
   - Spot c5.xlarge: **~$0.05-0.07/hour** (70% savings!)

---

## ✅ Exercise 6.3 Completion Checklist

- [ ] Created Docker container for image processing
- [ ] Pushed image to ECR
- [ ] Created S3 source and destination buckets
- [ ] Created SQS queue for job coordination
- [ ] Configured S3 event notifications to SQS
- [ ] Created ECS cluster with EC2 launch type
- [ ] Configured Spot capacity provider with ASG
- [ ] Enabled managed scaling for capacity provider
- [ ] Created task definition with container config
- [ ] Created ECS service with Spot instances
- [ ] Configured auto-scaling based on SQS queue depth
- [ ] Generated 100 test images
- [ ] Uploaded images and triggered processing
- [ ] Monitored auto-scaling in action
- [ ] Verified processed images in destination bucket
- [ ] Confirmed 70%+ cost savings with Spot

**🎉 Congratulations!** You've completed Exercise 6.3 and all of Module 6!

---

# 🎓 Module 6 Complete Summary

## What You've Accomplished

### Exercise 6.1: ECS Fargate ETL ✅
- Built containerized ETL pipeline
- Deployed serverless containers with Fargate
- Loaded data to Redshift from S3
- Scheduled daily execution with EventBridge

### Exercise 6.2: EKS with Airflow ✅
- Created production EKS cluster
- Deployed Apache Airflow on Kubernetes
- Built complex DAG orchestrating EMR/Redshift
- Leveraged Kubernetes auto-scaling

### Exercise 6.3: ECS Spot Batch Processing ✅
- Processed 100 images in parallel
- Achieved 70% cost savings with Spot instances
- Implemented SQS-based auto-scaling
- Built resilient, cost-optimized batch system

## Key Services Mastered
1. **Amazon ECS** - Container orchestration
2. **AWS Fargate** - Serverless containers
3. **Amazon EKS** - Managed Kubernetes
4. **Spot Instances** - Cost-optimized compute

## Next Steps
- Practice combining skills across modules
- Build end-to-end data pipelines
- Prepare for certification exam

---

## 🧹 Cleanup Resources

**ECS:**
- Delete services
- Deregister task definitions
- Delete clusters
- Terminate EC2 instances in ASG

**EKS:**
- Delete Airflow Helm deployment
- Delete node groups
- Delete EKS cluster

**S3/SQS:**
- Empty and delete buckets
- Delete SQS queues

**ECR:**
- Delete images and repositories

**Redshift:**
- Delete clusters

**Estimated Module 6 Lab Cost (4-5 hours): ~$55-95**

---

**🎉 Fantastic work! You've mastered AWS container services!**

---

# 🎓 Modules 3-6 Complete!

You now have comprehensive GUI-based guides for:
- ✅ **Module 3:** Database Services (Aurora, Redshift, DynamoDB)
- ✅ **Module 4:** Migration & Transfer (DMS, DataSync, Transfer Family)
- ✅ **Module 5:** Compute (Lambda, Batch, EMR)
- ✅ **Module 6:** Containers (ECS, EKS, Fargate)

**Total hands-on exercises completed: 12**

**Next:** Continue with remaining modules or start practicing for the certification exam!

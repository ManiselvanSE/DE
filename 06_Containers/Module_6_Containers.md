# MODULE 6: CONTAINERS FOR DATA ENGINEERING

## MODULE OVERVIEW

**Duration:** 8-10 hours  
**Difficulty:** Advanced  
**Prerequisites:** Modules 1-5, Docker fundamentals, Kubernetes basics  
**Cost:** $100-250 (temporary container resources)

---

## WHAT YOU WILL LEARN

### AWS Container Services for Data Engineering

This module covers AWS container orchestration services for running scalable, portable data engineering workloads. You'll learn when to use ECS vs EKS vs Fargate and how to containerize data pipelines.

**Key Services:**
- ✅ **Amazon ECS (Elastic Container Service)** - AWS-native container orchestration
- ✅ **Amazon EKS (Elastic Kubernetes Service)** - Managed Kubernetes for complex workloads
- ✅ **AWS Fargate** - Serverless compute for containers (no EC2 management)
- ✅ **Amazon ECR (Elastic Container Registry)** - Private Docker image repository
- ✅ **ECS with Spot Instances** - 70-90% cost savings for fault-tolerant workloads

### Production Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│         Container Architecture for Data Engineering          │
└─────────────────────────────────────────────────────────────┘

Containerized ETL Pipelines
├─ ECS Fargate: Serverless Spark jobs (no cluster management)
├─ EKS: Complex workflows with Airflow on Kubernetes
├─ ECS with Spot: Cost-optimized batch processing (90% savings)
└─ ECR: Private image registry for custom data processing containers

Real-Time Streaming
├─ ECS: Kafka consumers processing 1M events/second
├─ Fargate: Auto-scaling stream processors
└─ EKS: Apache Flink for complex event processing

ML Training & Inference
├─ ECS: Distributed training jobs with GPU instances
├─ EKS: Kubeflow for ML pipelines
└─ Fargate: Serverless model inference endpoints

Data Lake Processing
├─ ECS Tasks: PySpark containers reading from S3
├─ EKS Pods: Presto query engine for ad-hoc analytics
└─ Fargate: Glue-compatible Spark containers
```

---

## EXERCISE 6.1: CONTAINERIZED ETL PIPELINE WITH ECS FARGATE

**Scenario:** Build a containerized ETL pipeline that processes daily sales data from S3, transforms it using custom Python logic, and loads into Redshift—all running on serverless Fargate.

**Architecture:**

```
Amazon S3 (Landing Zone)
└─ s3://sales-data/raw/2024-06-22/sales.csv (5 GB)
    │
    │ EventBridge Schedule (daily at 2 AM)
    ▼
Amazon ECS (Fargate Launch Type)
├─ Task Definition: sales-etl-task
│  ├─ CPU: 4 vCPU (4,096 CPU units)
│  ├─ Memory: 16 GB (16,384 MB)
│  ├─ Container: sales-processor:latest (from ECR)
│  └─ Task Role: Access to S3, Redshift, Secrets Manager
│
├─ Container Execution:
│  ├─ Read CSV from S3 (streaming, memory-efficient)
│  ├─ Transform: Deduplicate, validate, enrich with APIs
│  ├─ Write to /tmp/output.parquet (10 GB ephemeral storage)
│  └─ COPY to Redshift from S3
│
└─ Auto-scaling: 1-10 tasks based on queue depth
    │
    ▼
Amazon Redshift
└─ sales.fact_sales table (partitioned by date)
    │
    └─→ QuickSight dashboards
```

**Implementation:**

**Step 1: Create Docker Container for ETL**

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Install dependencies
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python packages
COPY requirements.txt /app/
RUN pip install --no-cache-dir -r /app/requirements.txt

# Copy ETL script
COPY etl_processor.py /app/
WORKDIR /app

# Entry point
CMD ["python", "etl_processor.py"]
```

```python
# requirements.txt
boto3==1.34.100
pandas==2.2.0
pyarrow==15.0.0
psycopg2-binary==2.9.9
requests==2.31.0
```

```python
# etl_processor.py
import os
import boto3
import pandas as pd
import psycopg2
from datetime import datetime
import logging

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

# AWS clients
s3 = boto3.client('s3')
secrets = boto3.client('secretsmanager', region_name='us-east-1')

def get_redshift_credentials():
    """Retrieve Redshift credentials from Secrets Manager"""
    secret = secrets.get_secret_value(SecretId='redshift-sales-db')
    creds = eval(secret['SecretString'])
    return creds

def download_from_s3(bucket, key, local_path):
    """Download file from S3 with progress tracking"""
    logger.info(f"Downloading s3://{bucket}/{key}")
    
    file_size = s3.head_object(Bucket=bucket, Key=key)['ContentLength']
    logger.info(f"File size: {file_size / 1024**3:.2f} GB")
    
    s3.download_file(bucket, key, local_path)
    logger.info(f"Downloaded to {local_path}")

def transform_data(input_path, output_path):
    """Transform CSV to Parquet with data quality checks"""
    logger.info(f"Reading CSV: {input_path}")
    
    # Read in chunks (memory-efficient for large files)
    chunks = []
    chunk_size = 100000
    
    for chunk in pd.read_csv(input_path, chunksize=chunk_size):
        # Data quality: Remove duplicates
        chunk = chunk.drop_duplicates(subset=['order_id'])
        
        # Data quality: Validate dates
        chunk['order_date'] = pd.to_datetime(chunk['order_date'], errors='coerce')
        chunk = chunk.dropna(subset=['order_date'])
        
        # Data quality: Validate amounts
        chunk = chunk[chunk['amount'] > 0]
        
        # Add processing timestamp
        chunk['processed_at'] = datetime.utcnow()
        
        chunks.append(chunk)
    
    # Combine chunks
    df = pd.concat(chunks, ignore_index=True)
    logger.info(f"Rows after transformation: {len(df):,}")
    
    # Optimize data types (reduce storage)
    df['order_id'] = df['order_id'].astype('int64')
    df['customer_id'] = df['customer_id'].astype('int32')
    df['amount'] = df['amount'].astype('float32')
    
    # Write to Parquet
    df.to_parquet(
        output_path,
        compression='snappy',
        index=False
    )
    
    logger.info(f"Written Parquet: {output_path}")
    return len(df)

def upload_to_s3(local_path, bucket, key):
    """Upload file to S3"""
    logger.info(f"Uploading to s3://{bucket}/{key}")
    s3.upload_file(local_path, bucket, key)
    logger.info(f"Upload complete")

def load_to_redshift(s3_path, table_name):
    """Load Parquet from S3 into Redshift using COPY"""
    logger.info(f"Loading {s3_path} into Redshift table: {table_name}")
    
    # Get credentials
    creds = get_redshift_credentials()
    
    # Connect to Redshift
    conn = psycopg2.connect(
        host=creds['host'],
        port=creds['port'],
        database=creds['database'],
        user=creds['username'],
        password=creds['password']
    )
    
    cursor = conn.cursor()
    
    # COPY command (Parquet format)
    copy_sql = f"""
    COPY {table_name}
    FROM '{s3_path}'
    IAM_ROLE '{creds['iam_role']}'
    FORMAT AS PARQUET;
    """
    
    logger.info("Executing COPY command...")
    cursor.execute(copy_sql)
    conn.commit()
    
    # Get row count
    cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
    row_count = cursor.fetchone()[0]
    
    logger.info(f"Loaded {row_count:,} rows into {table_name}")
    
    cursor.close()
    conn.close()

def main():
    """Main ETL workflow"""
    logger.info("Starting ETL job")
    start_time = datetime.now()
    
    # Environment variables (set in ECS task definition)
    input_bucket = os.environ['INPUT_BUCKET']
    input_key = os.environ['INPUT_KEY']
    output_bucket = os.environ['OUTPUT_BUCKET']
    output_key = os.environ['OUTPUT_KEY']
    redshift_table = os.environ['REDSHIFT_TABLE']
    
    # Local paths (use /tmp for ephemeral storage)
    local_input = '/tmp/input.csv'
    local_output = '/tmp/output.parquet'
    
    try:
        # Step 1: Download from S3
        download_from_s3(input_bucket, input_key, local_input)
        
        # Step 2: Transform
        row_count = transform_data(local_input, local_output)
        
        # Step 3: Upload to S3
        upload_to_s3(local_output, output_bucket, output_key)
        
        # Step 4: Load to Redshift
        s3_path = f"s3://{output_bucket}/{output_key}"
        load_to_redshift(s3_path, redshift_table)
        
        # Cleanup
        os.remove(local_input)
        os.remove(local_output)
        
        duration = (datetime.now() - start_time).total_seconds()
        logger.info(f"✅ ETL job completed successfully in {duration:.2f} seconds")
        logger.info(f"   Rows processed: {row_count:,}")
        
        return 0
    
    except Exception as e:
        logger.error(f"❌ ETL job failed: {str(e)}")
        raise

if __name__ == '__main__':
    exit(main())
```

**Step 2: Build and Push Docker Image to ECR**

```bash
# Create ECR repository
aws ecr create-repository \
  --repository-name sales-etl \
  --region us-east-1

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build Docker image
docker build -t sales-etl:latest .

# Tag for ECR
docker tag sales-etl:latest \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/sales-etl:latest

# Push to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/sales-etl:latest

# Verify
aws ecr describe-images \
  --repository-name sales-etl \
  --region us-east-1
```

**Step 3: Create ECS Task Definition (Fargate)**

```bash
# Create IAM role for ECS task execution
cat > task-execution-role-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://task-execution-role-trust.json

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Create IAM role for task (S3, Redshift, Secrets Manager access)
cat > task-role-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ecs-tasks.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name salesEtlTaskRole \
  --assume-role-policy-document file://task-role-trust.json

# Attach policies
aws iam attach-role-policy \
  --role-name salesEtlTaskRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-role-policy \
  --role-name salesEtlTaskRole \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite

# Create ECS task definition
cat > task-definition.json <<EOF
{
  "family": "sales-etl-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "4096",
  "memory": "16384",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/salesEtlTaskRole",
  "containerDefinitions": [{
    "name": "sales-processor",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/sales-etl:latest",
    "essential": true,
    "environment": [
      {"name": "INPUT_BUCKET", "value": "sales-data"},
      {"name": "INPUT_KEY", "value": "raw/2024-06-22/sales.csv"},
      {"name": "OUTPUT_BUCKET", "value": "sales-processed"},
      {"name": "OUTPUT_KEY", "value": "parquet/2024-06-22/sales.parquet"},
      {"name": "REDSHIFT_TABLE", "value": "sales.fact_sales"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/sales-etl",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
EOF

# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json
```

**Step 4: Create ECS Cluster and Service**

```bash
# Create ECS cluster (Fargate)
aws ecs create-cluster \
  --cluster-name sales-etl-cluster \
  --region us-east-1

# Create CloudWatch Log Group
aws logs create-log-group \
  --log-group-name /ecs/sales-etl \
  --region us-east-1

# Run task manually (test)
aws ecs run-task \
  --cluster sales-etl-cluster \
  --launch-type FARGATE \
  --task-definition sales-etl-task \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-0a1b2c3d", "subnet-1a2b3c4d"],
      "securityGroups": ["sg-0a1b2c3d"],
      "assignPublicIp": "ENABLED"
    }
  }' \
  --region us-east-1

# Monitor task
aws ecs describe-tasks \
  --cluster sales-etl-cluster \
  --tasks <task-arn> \
  --region us-east-1

# View logs
aws logs tail /ecs/sales-etl --follow
```

**Step 5: Schedule with EventBridge**

```bash
# Create EventBridge rule (daily at 2 AM)
aws events put-rule \
  --name sales-etl-daily \
  --schedule-expression "cron(0 2 * * ? *)" \
  --state ENABLED \
  --region us-east-1

# Add ECS task as target
aws events put-targets \
  --rule sales-etl-daily \
  --targets '[{
    "Id": "1",
    "Arn": "arn:aws:ecs:us-east-1:123456789012:cluster/sales-etl-cluster",
    "RoleArn": "arn:aws:iam::123456789012:role/ecsEventsRole",
    "EcsParameters": {
      "TaskDefinitionArn": "arn:aws:ecs:us-east-1:123456789012:task-definition/sales-etl-task:1",
      "TaskCount": 1,
      "LaunchType": "FARGATE",
      "NetworkConfiguration": {
        "awsvpcConfiguration": {
          "Subnets": ["subnet-0a1b2c3d", "subnet-1a2b3c4d"],
          "SecurityGroups": ["sg-0a1b2c3d"],
          "AssignPublicIp": "ENABLED"
        }
      }
    }
  }]' \
  --region us-east-1
```

**Results:**

| Metric | Traditional EC2 | ECS Fargate | Improvement |
|--------|-----------------|-------------|-------------|
| **Setup Time** | 2 hours (provision, configure) | 15 minutes (task definition) | **87% faster** |
| **Processing Time** | 18 minutes (5 GB CSV) | 12 minutes (optimized container) | **33% faster** |
| **Cost per Run** | $2.50 (t3.xlarge × 0.5 hours) | $0.81 (4 vCPU × 0.2 hours) | **68% cheaper** |
| **Idle Cost** | $60/day (always running) | $0/day (serverless) | **100% savings** |
| **Scaling** | Manual | Automatic (1-10 tasks) | **Fully automated** |

**Cost Breakdown:**

```
ECS Fargate Pricing (per task):
├─ CPU: 4 vCPU × $0.04048/vCPU/hour × 0.2 hours = $0.032
├─ Memory: 16 GB × $0.004445/GB/hour × 0.2 hours = $0.014
├─ Ephemeral Storage: 20 GB × $0.000111/GB/hour × 0.2 hours = $0.0004
└─ Total per task: $0.046

Daily Cost (1 task/day):
└─ $0.046 × 1 = $0.046/day = $1.38/month

vs EC2 (t3.xlarge 24/7):
├─ $0.1664/hour × 24 hours × 30 days = $120/month
└─ Savings: $118.62/month (99% reduction) ✅
```

**Key Lessons:**

1. **Fargate is serverless** - No EC2 instances to manage, zero idle cost
2. **Task isolation** - Each run gets fresh environment, no state pollution
3. **Auto-scaling** - Scale from 1 to 100 tasks based on queue depth
4. **ECR integration** - Private Docker registry with vulnerability scanning
5. **Secrets Manager** - Securely inject database credentials at runtime

---

## EXERCISE 6.2: KUBERNETES-BASED DATA PIPELINE WITH AMAZON EKS

**Scenario:** Deploy Apache Airflow on EKS to orchestrate complex data pipelines with dependencies, retries, and monitoring.

**Architecture:**

```
Amazon EKS Cluster
├─ Node Group 1: System (t3.medium × 2)
│  └─ Airflow Web Server, Scheduler, PostgreSQL metadata DB
│
├─ Node Group 2: Workers (r5.2xlarge × 5, auto-scaling 3-20)
│  └─ Airflow Workers (Celery Executor)
│      ├─ Worker Pod 1: PySpark job (read 100 GB S3 → transform)
│      ├─ Worker Pod 2: API calls (enrich data from external services)
│      ├─ Worker Pod 3: Redshift COPY (load transformed data)
│      └─ Worker Pod 4-N: Parallel tasks
│
└─ Node Group 3: GPU (p3.2xlarge × 2, Spot)
   └─ ML Training Pods (TensorFlow/PyTorch)
       │
       ▼
Amazon S3 (Data Lake)
├─ Raw data: 500 GB/day
├─ Processed: 150 GB/day (Parquet)
└─ ML models: 10 GB

Monitoring:
├─ Prometheus (metrics)
├─ Grafana (dashboards)
└─ CloudWatch Container Insights
```

**Implementation:**

**Step 1: Create EKS Cluster**

```bash
# Install eksctl
curl --silent --location "https://github.com/weksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Create EKS cluster with eksctl
cat > cluster-config.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: data-pipeline-cluster
  region: us-east-1
  version: "1.29"

iam:
  withOIDC: true

managedNodeGroups:
  - name: system
    instanceType: t3.medium
    minSize: 2
    maxSize: 3
    desiredCapacity: 2
    labels:
      role: system
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
  
  - name: workers
    instanceType: r5.2xlarge
    minSize: 3
    maxSize: 20
    desiredCapacity: 5
    labels:
      role: worker
    tags:
      k8s.io/cluster-autoscaler/enabled: "true"
  
  - name: gpu-spot
    instanceType: p3.2xlarge
    minSize: 0
    maxSize: 5
    desiredCapacity: 0
    spot: true
    labels:
      role: ml-training
      nvidia.com/gpu: "true"
    taints:
      - key: nvidia.com/gpu
        value: "true"
        effect: NoSchedule
EOF

# Create cluster (takes 15-20 minutes)
eksctl create cluster -f cluster-config.yaml

# Configure kubectl
aws eks update-kubeconfig \
  --region us-east-1 \
  --name data-pipeline-cluster

# Verify cluster
kubectl get nodes
```

**Step 2: Deploy Apache Airflow on EKS**

```bash
# Add Airflow Helm repository
helm repo add apache-airflow https://airflow.apache.org
helm repo update

# Create namespace
kubectl create namespace airflow

# Create values file for Airflow
cat > airflow-values.yaml <<EOF
# Airflow configuration
executor: CeleryExecutor

# PostgreSQL (metadata database)
postgresql:
  enabled: true
  persistence:
    enabled: true
    size: 50Gi

# Redis (Celery broker)
redis:
  enabled: true

# Airflow components
webserver:
  replicas: 2
  resources:
    requests:
      cpu: "1000m"
      memory: "2Gi"
    limits:
      cpu: "2000m"
      memory: "4Gi"

scheduler:
  replicas: 2
  resources:
    requests:
      cpu: "1000m"
      memory: "2Gi"

workers:
  replicas: 5
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
  resources:
    requests:
      cpu: "2000m"
      memory: "8Gi"
    limits:
      cpu: "4000m"
      memory: "16Gi"
  nodeSelector:
    role: worker

# DAGs sync from Git
dags:
  gitSync:
    enabled: true
    repo: https://github.com/company/airflow-dags.git
    branch: main
    wait: 60

# Connections and variables
env:
  - name: AIRFLOW__CORE__LOAD_EXAMPLES
    value: "False"
  - name: AIRFLOW__WEBSERVER__EXPOSE_CONFIG
    value: "True"

# Service
service:
  type: LoadBalancer
EOF

# Install Airflow
helm install airflow apache-airflow/airflow \
  --namespace airflow \
  --values airflow-values.yaml \
  --timeout 10m

# Wait for deployment
kubectl wait --for=condition=ready pod \
  --all --namespace airflow \
  --timeout=600s

# Get Airflow Web UI URL
kubectl get svc -n airflow airflow-webserver

# Access Airflow (default credentials: admin/admin)
# http://<EXTERNAL-IP>:8080
```

**Step 3: Create Airflow DAG for Data Pipeline**

```python
# dags/daily_sales_pipeline.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.operators.emr import EmrCreateJobFlowOperator, EmrAddStepsOperator, EmrTerminateJobFlowOperator
from airflow.providers.amazon.aws.sensors.emr import EmrStepSensor
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime, timedelta
import boto3

default_args = {
    'owner': 'data-engineering',
    'depends_on_past': False,
    'start_date': datetime(2024, 6, 1),
    'email': ['data-eng@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'daily_sales_pipeline',
    default_args=default_args,
    description='Daily sales data processing pipeline',
    schedule_interval='0 2 * * *',  # 2 AM daily
    catchup=False,
    max_active_runs=1,
    tags=['sales', 'etl', 'production']
)

def validate_s3_data(**context):
    """Validate input data exists and meets quality criteria"""
    s3 = boto3.client('s3')
    execution_date = context['ds']  # YYYY-MM-DD
    
    bucket = 'sales-data'
    key = f'raw/{execution_date}/sales.csv'
    
    try:
        response = s3.head_object(Bucket=bucket, Key=key)
        file_size = response['ContentLength']
        
        # Validation: File size must be > 1 MB (not empty)
        if file_size < 1024 * 1024:
            raise ValueError(f"File too small: {file_size} bytes")
        
        print(f"✅ Validation passed: {file_size / 1024**2:.2f} MB")
        return {'bucket': bucket, 'key': key, 'size': file_size}
    
    except Exception as e:
        raise ValueError(f"Validation failed: {str(e)}")

# Task 1: Validate input data
validate_task = PythonOperator(
    task_id='validate_s3_data',
    python_callable=validate_s3_data,
    provide_context=True,
    dag=dag
)

# Task 2: Process data with EMR Spark
emr_cluster_config = {
    'Name': 'Sales-ETL-{{ ds }}',
    'ReleaseLabel': 'emr-7.0.0',
    'Applications': [{'Name': 'Spark'}],
    'Instances': {
        'InstanceGroups': [
            {
                'Name': 'Master',
                'InstanceRole': 'MASTER',
                'InstanceType': 'm5.xlarge',
                'InstanceCount': 1,
            },
            {
                'Name': 'Core',
                'InstanceRole': 'CORE',
                'InstanceType': 'r5.2xlarge',
                'InstanceCount': 3,
            }
        ],
        'KeepJobFlowAliveWhenNoSteps': False,
        'TerminationProtected': False,
    },
    'JobFlowRole': 'EMR_EC2_DefaultRole',
    'ServiceRole': 'EMR_DefaultRole',
    'LogUri': 's3://emr-logs/sales-etl/',
}

create_emr_cluster = EmrCreateJobFlowOperator(
    task_id='create_emr_cluster',
    job_flow_overrides=emr_cluster_config,
    aws_conn_id='aws_default',
    dag=dag
)

spark_step = [{
    'Name': 'Process Sales Data',
    'ActionOnFailure': 'TERMINATE_CLUSTER',
    'HadoopJarStep': {
        'Jar': 'command-runner.jar',
        'Args': [
            'spark-submit',
            '--deploy-mode', 'cluster',
            '--master', 'yarn',
            's3://scripts/sales_etl.py',
            '--input', 's3://sales-data/raw/{{ ds }}/',
            '--output', 's3://sales-processed/parquet/{{ ds }}/'
        ]
    }
}]

add_spark_step = EmrAddStepsOperator(
    task_id='add_spark_step',
    job_flow_id="{{ task_instance.xcom_pull(task_ids='create_emr_cluster', key='return_value') }}",
    steps=spark_step,
    aws_conn_id='aws_default',
    dag=dag
)

wait_for_spark = EmrStepSensor(
    task_id='wait_for_spark_completion',
    job_flow_id="{{ task_instance.xcom_pull(task_ids='create_emr_cluster', key='return_value') }}",
    step_id="{{ task_instance.xcom_pull(task_ids='add_spark_step')[0] }}",
    aws_conn_id='aws_default',
    dag=dag
)

# Task 3: Load to Redshift
load_redshift = PostgresOperator(
    task_id='load_to_redshift',
    postgres_conn_id='redshift_sales',
    sql="""
        COPY sales.fact_sales
        FROM 's3://sales-processed/parquet/{{ ds }}/'
        IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
        FORMAT AS PARQUET;
    """,
    dag=dag
)

# Task 4: Data quality check
def data_quality_check(**context):
    """Verify loaded data quality"""
    import psycopg2
    
    conn = psycopg2.connect(
        host='redshift-cluster.xyz.us-east-1.redshift.amazonaws.com',
        port=5439,
        database='sales',
        user='admin',
        password='{{ var.value.redshift_password }}'
    )
    
    cursor = conn.cursor()
    execution_date = context['ds']
    
    # Check row count
    cursor.execute(f"""
        SELECT COUNT(*)
        FROM sales.fact_sales
        WHERE order_date = '{execution_date}'
    """)
    row_count = cursor.fetchone()[0]
    
    # Check for nulls in critical columns
    cursor.execute(f"""
        SELECT
            SUM(CASE WHEN order_id IS NULL THEN 1 ELSE 0 END) AS null_orders,
            SUM(CASE WHEN amount <= 0 THEN 1 ELSE 0 END) AS invalid_amounts
        FROM sales.fact_sales
        WHERE order_date = '{execution_date}'
    """)
    null_orders, invalid_amounts = cursor.fetchone()
    
    cursor.close()
    conn.close()
    
    # Validations
    if row_count < 1000:
        raise ValueError(f"Too few rows loaded: {row_count} (expected > 1000)")
    
    if null_orders > 0:
        raise ValueError(f"Null order_ids found: {null_orders}")
    
    if invalid_amounts > 0:
        raise ValueError(f"Invalid amounts found: {invalid_amounts}")
    
    print(f"✅ Quality check passed: {row_count:,} rows loaded")

quality_check = PythonOperator(
    task_id='data_quality_check',
    python_callable=data_quality_check,
    provide_context=True,
    dag=dag
)

# Define task dependencies
validate_task >> create_emr_cluster >> add_spark_step >> wait_for_spark >> load_redshift >> quality_check
```

**Step 4: Deploy DAG to Airflow**

```bash
# Push DAG to Git repository (GitSync will auto-deploy)
git add dags/daily_sales_pipeline.py
git commit -m "Add daily sales pipeline DAG"
git push origin main

# Wait for GitSync (60 seconds)
sleep 60

# Verify DAG is loaded
kubectl exec -it -n airflow \
  $(kubectl get pods -n airflow -l component=scheduler -o jsonpath='{.items[0].metadata.name}') \
  -- airflow dags list | grep daily_sales

# Trigger DAG manually (test)
kubectl exec -it -n airflow \
  $(kubectl get pods -n airflow -l component=scheduler -o jsonpath='{.items[0].metadata.name}') \
  -- airflow dags trigger daily_sales_pipeline

# Monitor DAG execution
# Access Airflow UI: http://<LOAD-BALANCER-IP>:8080
```

**Results:**

| Metric | Traditional Airflow (EC2) | Airflow on EKS | Improvement |
|--------|---------------------------|----------------|-------------|
| **High Availability** | Single instance (SPOF) | Multi-replica (scheduler × 2, webserver × 2) | **99.9% uptime** |
| **Scalability** | Fixed (10 workers) | Auto-scaling (3-20 workers) | **2x capacity** |
| **Deployment Time** | 4 hours (manual setup) | 15 minutes (Helm chart) | **93% faster** |
| **Cost** | $500/month (always on) | $300/month (auto-scaling) | **40% cheaper** |
| **Infrastructure** | Manual (EC2, ALB, RDS) | Managed (EKS handles nodes) | **80% less ops** |

**Key Lessons:**

1. **EKS provides true HA** - Multi-AZ node groups, auto-healing pods
2. **Kubernetes auto-scaling** - HPA scales workers based on queue depth
3. **GitOps deployment** - DAGs auto-sync from Git (no manual deploy)
4. **Node groups** - Separate system, workers, GPU workloads
5. **Spot Instances** - Use for fault-tolerant tasks (70% savings)

---

(Continue with Exercise 6.3 and Q1-Q20...)

## EXERCISE 6.3: COST-OPTIMIZED BATCH PROCESSING WITH ECS SPOT INSTANCES

**Scenario:** Process 100,000 images daily (resize, watermark, compress) using ECS with Spot Instances for 90% cost savings.

**Architecture:**

```
Amazon S3 (Source)
└─ s3://images-raw/uploads/ (100,000 images/day, 50 MB each = 5 TB)
    │
    │ S3 Event → SQS Queue
    ▼
Amazon SQS Queue
├─ Queue depth: 0-100,000 messages
└─ Visibility timeout: 300 seconds
    │
    ▼
ECS Service (EC2 Launch Type with Spot)
├─ Task Definition: image-processor
│  ├─ CPU: 2 vCPU
│  ├─ Memory: 4 GB
│  └─ Container: image-processor:latest
│
├─ Capacity Provider Strategy:
│  ├─ Base: 2 tasks (On-Demand, always running)
│  └─ Weight: 100 (Spot Instances, scale 0-500)
│
└─ Auto Scaling:
   ├─ Target: SQS queue depth < 100 messages
   ├─ Scale-up: +50 tasks when queue > 1,000
   └─ Scale-down: -10 tasks when queue < 50
       │
       ▼
Amazon S3 (Processed)
└─ s3://images-processed/resized/ (compressed, watermarked)
```

**Implementation:**

```python
# image_processor.py (runs in ECS container)
import boto3
import os
import json
from PIL import Image
import io

s3 = boto3.client('s3')
sqs = boto3.client('sqs')

QUEUE_URL = os.environ['SQS_QUEUE_URL']
OUTPUT_BUCKET = os.environ['OUTPUT_BUCKET']

def process_image(input_bucket, input_key):
    """Download, resize, watermark, compress image"""
    
    # Download from S3
    response = s3.get_object(Bucket=input_bucket, Key=input_key)
    image_data = response['Body'].read()
    
    # Open image
    img = Image.open(io.BytesIO(image_data))
    
    # Resize (maintain aspect ratio)
    img.thumbnail((1920, 1080), Image.Resampling.LANCZOS)
    
    # Add watermark
    watermark = Image.open('/app/watermark.png')
    img.paste(watermark, (img.width - watermark.width - 10, img.height - watermark.height - 10), watermark)
    
    # Save to buffer with compression
    output_buffer = io.BytesIO()
    img.save(output_buffer, format='JPEG', quality=85, optimize=True)
    output_buffer.seek(0)
    
    # Upload to S3
    output_key = input_key.replace('/uploads/', '/resized/')
    s3.put_object(
        Bucket=OUTPUT_BUCKET,
        Key=output_key,
        Body=output_buffer,
        ContentType='image/jpeg'
    )
    
    return output_key

def main():
    """Poll SQS queue and process images"""
    
    while True:
        # Receive messages from SQS
        response = sqs.receive_message(
            QueueUrl=QUEUE_URL,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20
        )
        
        messages = response.get('Messages', [])
        
        if not messages:
            print("No messages in queue, waiting...")
            continue
        
        for message in messages:
            try:
                # Parse S3 event
                body = json.loads(message['Body'])
                s3_event = json.loads(body['Message'])
                
                for record in s3_event['Records']:
                    bucket = record['s3']['bucket']['name']
                    key = record['s3']['object']['key']
                    
                    print(f"Processing: s3://{bucket}/{key}")
                    
                    # Process image
                    output_key = process_image(bucket, key)
                    
                    print(f"✅ Processed: {output_key}")
                
                # Delete message from queue
                sqs.delete_message(
                    QueueUrl=QUEUE_URL,
                    ReceiptHandle=message['ReceiptHandle']
                )
                
            except Exception as e:
                print(f"❌ Error: {str(e)}")
                # Message will return to queue after visibility timeout

if __name__ == '__main__':
    main()
```

**ECS Task Definition with Spot Instances:**

```bash
# Create ECS capacity provider for Spot Instances
aws ecs create-capacity-provider \
  --name spot-capacity-provider \
  --auto-scaling-group-provider '{
    "autoScalingGroupArn": "arn:aws:autoscaling:us-east-1:123456789012:autoScalingGroup:...",
    "managedScaling": {
      "status": "ENABLED",
      "targetCapacity": 100,
      "minimumScalingStepSize": 1,
      "maximumScalingStepSize": 10
    },
    "managedTerminationProtection": "DISABLED"
  }'

# Create service with Spot capacity provider
aws ecs create-service \
  --cluster image-processing \
  --service-name image-processor \
  --task-definition image-processor \
  --desired-count 2 \
  --capacity-provider-strategy '[
    {
      "capacityProvider": "on-demand-capacity-provider",
      "weight": 0,
      "base": 2
    },
    {
      "capacityProvider": "spot-capacity-provider",
      "weight": 100,
      "base": 0
    }
  ]' \
  --enable-ecs-managed-tags \
  --propagate-tags SERVICE
```

**Auto Scaling Configuration:**

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/image-processing/image-processor \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 500

# Create scaling policy (target tracking on SQS queue depth)
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/image-processing/image-processor \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name sqs-queue-depth-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 100.0,
    "CustomizedMetricSpecification": {
      "MetricName": "ApproximateNumberOfMessagesVisible",
      "Namespace": "AWS/SQS",
      "Dimensions": [{
        "Name": "QueueName",
        "Value": "image-processing-queue"
      }],
      "Statistic": "Average"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'
```

**Results:**

| Metric | On-Demand | Spot Instances | Savings |
|--------|-----------|----------------|---------|
| **Cost per Hour** | $1.68 (10 × c5.large) | $0.17 (10 × c5.large Spot) | **90%** |
| **Daily Cost** | $40.32 (24 hours) | $4.08 (Spot) | **90%** |
| **Processing Time** | 2 hours (100K images) | 2 hours (same) | - |
| **Interruptions** | 0 | 3-5/day (auto-replaced) | Negligible |
| **Idle Cost** | $40/day (always on) | $0.34/day (2 base tasks) | **99%** |

**Monthly Cost Comparison:**

```
On-Demand (24/7, 10 tasks average):
└─ $40.32/day × 30 days = $1,209.60/month

Spot Instances (auto-scaling, 90% discount):
├─ Base (2 On-Demand tasks): $0.34/day × 30 = $10.20
├─ Peak (500 Spot tasks × 2 hours/day): $16.80/day × 30 = $504
└─ Total: $514.20/month
    Savings: $695.40/month (57% reduction) ✅
```

---

## BEGINNER QUESTIONS (Q1-Q5)

**Q1: What is the difference between ECS and EKS?**

**Answer:**

| Feature | Amazon ECS | Amazon EKS |
|---------|-----------|-----------|
| **Orchestrator** | AWS-native (proprietary) | Kubernetes (open-source) |
| **Learning Curve** | Easier (AWS-specific) | Steeper (Kubernetes knowledge required) |
| **Portability** | AWS only | Multi-cloud, on-premises |
| **Complexity** | Simpler (less config) | More complex (K8s manifests, Helm) |
| **Control Plane Cost** | Free | $0.10/hour ($73/month per cluster) |
| **Best For** | AWS-native workloads | Complex workflows, multi-cloud, K8s ecosystem |
| **Ecosystem** | AWS services only | Huge K8s ecosystem (Helm, operators, etc.) |

**When to use ECS:**
- Simple containerized applications
- AWS-only deployment
- Want managed control plane at no cost
- Team familiar with AWS, not Kubernetes
- Example: Microservices on Fargate, batch processing

**When to use EKS:**
- Need Kubernetes ecosystem (Airflow, Spark on K8s, Kubeflow)
- Multi-cloud or hybrid deployment
- Complex orchestration requirements
- Team has K8s expertise
- Example: Apache Airflow, ML pipelines, service mesh

---

**Q2: What is AWS Fargate and when should you use it?**

**Answer:**

AWS Fargate is a **serverless compute engine** for containers (works with both ECS and EKS). You don't manage EC2 instances—just define CPU/memory and run containers.

**Fargate vs EC2 Launch Type:**

| Feature | Fargate (Serverless) | EC2 Launch Type |
|---------|---------------------|-----------------|
| **Infrastructure** | No servers to manage | Manage EC2 instances |
| **Pricing** | Pay per vCPU/GB used | Pay for EC2 instances (even idle) |
| **Scaling** | Per-task granularity | Instance-level (coarse) |
| **Startup Time** | 30-60 seconds | Instant (if instances ready) |
| **Control** | Limited (no host access) | Full (SSH, custom AMIs) |
| **Cost (small workloads)** | Cheaper (no idle) | More expensive |
| **Cost (large, steady workloads)** | More expensive | Cheaper (EC2 Reserved) |

**Use Fargate when:**
- Variable workload (scales to zero)
- Don't want to manage clusters
- Small to medium containers (< 4 vCPU, < 30 GB)
- Cost: Pay only for task runtime

**Use EC2 when:**
- Large, steady workloads (Reserved Instances 40% cheaper)
- Need host-level control (GPUs, custom networking)
- Very large containers (> 4 vCPU, > 30 GB)
- Spot Instances for 70-90% savings

**Cost Example:**

```
Workload: Run 10 tasks, 2 vCPU, 4 GB each

Fargate:
├─ 10 tasks × 2 vCPU × $0.04048/vCPU/hr = $0.81/hr
├─ 10 tasks × 4 GB × $0.004445/GB/hr = $0.18/hr
└─ Total: $0.99/hr = $713/month (24/7)

EC2 (c5.large: 2 vCPU, 4 GB):
├─ 10 × c5.large On-Demand = $1.70/hr = $1,224/month
├─ 10 × c5.large Reserved (1-year) = $1.04/hr = $748/month
└─ 10 × c5.large Spot = $0.17/hr = $122/month ✅

Winner:
- Variable workload: Fargate (no idle cost)
- Steady 24/7: EC2 Spot (83% cheaper)
```

---

**Q3: What is Amazon ECR and how does it integrate with ECS/EKS?**

**Answer:**

Amazon ECR (Elastic Container Registry) is a fully managed Docker container registry for storing, managing, and deploying container images.

**Key Features:**

1. **Private & Public Repositories**
   - Private: For your organization only
   - Public: ECR Public Gallery (like Docker Hub)

2. **Security**
   - Image scanning (vulnerability detection)
   - Encryption at rest (KMS)
   - IAM-based access control
   - Lifecycle policies (auto-delete old images)

3. **Integration**
   - ECS: Native integration (pull images seamlessly)
   - EKS: Kubernetes ImagePullSecrets
   - CI/CD: GitHub Actions, Jenkins, CodePipeline

**Example: Push Image to ECR**

```bash
# Create repository
aws ecr create-repository --repository-name my-app

# Authenticate Docker
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag image
docker tag my-app:latest \
  123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# Enable image scanning
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true
```

**ECS Task Definition (using ECR image):**

```json
{
  "family": "my-app-task",
  "containerDefinitions": [{
    "name": "app",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
    "cpu": 256,
    "memory": 512
  }]
}
```

**Cost:**

```
ECR Pricing:
├─ Storage: $0.10/GB/month
├─ Data Transfer OUT: $0.09/GB (to internet)
└─ Data Transfer (ECR → ECS/EKS in same region): FREE

Example:
├─ 100 GB images stored: $10/month
├─ 1 TB transfers to ECS: $0 (same region)
└─ Total: $10/month
```

---

**Q4: How do you implement auto-scaling for ECS services?**

**Answer:**

ECS supports **Application Auto Scaling** based on CloudWatch metrics (CPU, memory, custom metrics like SQS queue depth).

**Auto-Scaling Configuration:**

```bash
# 1. Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 100

# 2. Create scaling policy (Target Tracking)
# Scale based on CPU utilization (target: 70%)
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 60,
    "ScaleInCooldown": 300
  }'

# 3. Custom metric: SQS queue depth
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name sqs-queue-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 100.0,
    "CustomizedMetricSpecification": {
      "MetricName": "ApproximateNumberOfMessagesVisible",
      "Namespace": "AWS/SQS",
      "Dimensions": [{
        "Name": "QueueName",
        "Value": "my-processing-queue"
      }],
      "Statistic": "Average"
    }
  }'
```

**Scaling Behavior:**

```
Scenario: SQS queue has 1,000 messages, target is 100 messages/task

Current: 5 tasks (handling 500 messages at target rate)
Required: 1,000 / 100 = 10 tasks

Auto-scaling:
├─ Detects: Queue depth > target
├─ Calculates: Need 10 tasks (vs current 5)
├─ Scales out: +5 tasks (respects max capacity)
├─ Wait: ScaleOutCooldown (60 seconds)
└─ Result: 10 tasks processing queue

When queue drops to 200 messages:
├─ Detects: Queue depth < target (only need 2 tasks)
├─ Wait: ScaleInCooldown (300 seconds, longer to avoid flapping)
├─ Scales in: -8 tasks (down to 2, respects min capacity)
└─ Result: 2 tasks (minimum)
```

---

**Q5: What are ECS task roles vs execution roles?**

**Answer:**

**Task Execution Role** = What ECS agent needs to run your container  
**Task Role** = What your container code needs to access AWS services

| Role | Purpose | Who Uses It | Permissions |
|------|---------|-------------|-------------|
| **Task Execution Role** | Pull image from ECR, send logs to CloudWatch | ECS agent | ECR pull, CloudWatch Logs write |
| **Task Role** | Access AWS services from application code | Your container | S3, DynamoDB, Secrets Manager, etc. |

**Example:**

```json
// Task Definition
{
  "family": "data-processor",
  "executionRoleArn": "arn:aws:iam::123:role/ecsTaskExecutionRole",  // ECS agent uses this
  "taskRoleArn": "arn:aws:iam::123:role/dataProcessorTaskRole",       // Your code uses this
  "containerDefinitions": [{
    "name": "processor",
    "image": "123.dkr.ecr.us-east-1.amazonaws.com/processor:latest",
    "environment": [
      {"name": "S3_BUCKET", "value": "data-lake"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/data-processor"
      }
    }
  }]
}
```

**Task Execution Role Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:log-group:/ecs/*"
    }
  ]
}
```

**Task Role Policy (for application):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::data-lake/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123:secret:db-password-*"
    }
  ]
}
```

**In Application Code:**

```python
# Your container code automatically uses Task Role (via IAM credentials)
import boto3

s3 = boto3.client('s3')  # Uses taskRoleArn credentials
secrets = boto3.client('secretsmanager')  # Uses taskRoleArn credentials

# Read from S3
s3.get_object(Bucket='data-lake', Key='data.csv')

# Get secret
secret = secrets.get_secret_value(SecretId='db-password')

# NO AWS credentials needed in code - IAM role handles it
```

---

(Continue with Q6-Q20...)

## INTERMEDIATE QUESTIONS (Q6-Q10)

**Q6: How do you optimize container images for faster deployments?**

**Answer:**

**Image Optimization Strategies:**

1. **Use Multi-Stage Builds** (reduce final image size)

```dockerfile
# Bad: Single-stage (500 MB image)
FROM python:3.11
COPY . /app
RUN pip install -r requirements.txt
CMD ["python", "app.py"]

# Good: Multi-stage build (150 MB image)
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]

# Result: 70% smaller image → 3x faster pulls
```

2. **Layer Caching** (reuse unchanged layers)

```dockerfile
# Bad: Code changes invalidate all layers
FROM python:3.11-slim
COPY . /app  # Entire app copied first
RUN pip install -r /app/requirements.txt  # Re-runs every time

# Good: Dependencies cached separately
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt  # Cached layer (rarely changes)
COPY . .  # Code layer (changes often, but doesn't invalidate deps)
CMD ["python", "app.py"]

# Result: 10x faster builds when only code changes
```

3. **Use .dockerignore** (exclude unnecessary files)

```bash
# .dockerignore
.git
__pycache__
*.pyc
.env
node_modules
tests/
README.md

# Result: 50% smaller build context → faster uploads to build server
```

4. **Minimize Layers** (combine RUN commands)

```dockerfile
# Bad: Many layers (increases image size)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get install -y git

# Good: Single layer
RUN apt-get update && apt-get install -y \
    curl \
    vim \
    git \
    && rm -rf /var/lib/apt/lists/*  # Clean up

# Result: 30% smaller image
```

**Results:**

| Optimization | Before | After | Improvement |
|--------------|--------|-------|-------------|
| **Multi-stage build** | 500 MB | 150 MB | 70% smaller |
| **Layer caching** | 5 min rebuild | 30 sec | 90% faster |
| **.dockerignore** | 200 MB context | 50 MB | 75% smaller |
| **Combined layers** | 550 MB | 385 MB | 30% smaller |

---

**Q7: How do you implement blue-green deployments with ECS?**

**Answer:**

**Blue-Green Deployment Strategy:**

```
Current State (Blue):
├─ ECS Service: my-app-blue
├─ Task Definition: my-app:v1
├─ ALB Target Group: blue-tg (100% traffic)
└─ Tasks: 10 running

New Deployment (Green):
├─ ECS Service: my-app-green
├─ Task Definition: my-app:v2
├─ ALB Target Group: green-tg (0% traffic initially)
└─ Tasks: 10 running (new version)

Switch Traffic:
├─ ALB Listener Rule: 0% blue, 100% green
└─ Rollback: Switch back to blue if issues
```

**Implementation with AWS CLI:**

```bash
# 1. Deploy green environment
aws ecs create-service \
  --cluster production \
  --service-name my-app-green \
  --task-definition my-app:v2 \
  --desired-count 10 \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...:targetgroup/green-tg/...",
    "containerName": "app",
    "containerPort": 8080
  }]'

# 2. Wait for green service to be stable
aws ecs wait services-stable \
  --cluster production \
  --services my-app-green

# 3. Run smoke tests against green target group
curl http://green-tg.alb.amazonaws.com/health

# 4. Switch traffic (gradually or all at once)
# Gradual: Update ALB listener rule weights
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/... \
  --default-actions '[
    {
      "Type": "forward",
      "ForwardConfig": {
        "TargetGroups": [
          {"TargetGroupArn": "arn:...blue-tg", "Weight": 90},
          {"TargetGroupArn": "arn:...green-tg", "Weight": 10}
        ]
      }
    }
  ]'

# Monitor metrics, then increase green to 100%
aws elbv2 modify-listener \
  --listener-arn arn:aws:elasticloadbalancing:...:listener/... \
  --default-actions '[
    {
      "Type": "forward",
      "ForwardConfig": {
        "TargetGroups": [
          {"TargetGroupArn": "arn:...green-tg", "Weight": 100}
        ]
      }
    }
  ]'

# 5. Decommission blue environment
aws ecs update-service \
  --cluster production \
  --service my-app-blue \
  --desired-count 0
```

**With AWS CodeDeploy (Automated):**

```yaml
# appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:...:task-definition/my-app:v2"
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 8080
        PlatformVersion: "LATEST"

Hooks:
  - BeforeInstall: "CodeDeployHook_BeforeInstall"
  - AfterInstall: "CodeDeployHook_AfterInstall"
  - AfterAllowTestTraffic: "CodeDeployHook_ValidateService"
  - BeforeAllowTraffic: "CodeDeployHook_PreTrafficCheck"
  - AfterAllowTraffic: "CodeDeployHook_PostTrafficCheck"
```

**Benefits:**

- **Zero downtime:** Green is fully tested before receiving traffic
- **Instant rollback:** Revert ALB rules to blue
- **A/B testing:** Split traffic 50/50 to compare versions
- **Reduced risk:** Gradual traffic shift (10% → 50% → 100%)

---

**Q8: What are the cost differences between ECS Fargate and EC2 launch types?**

**Answer:**

**Detailed Cost Comparison:**

**Scenario: Run 50 tasks, each 2 vCPU, 4 GB memory**

**Fargate (Serverless):**

```
Pricing:
├─ 50 tasks × 2 vCPU × $0.04048/vCPU/hour = $4.048/hour
├─ 50 tasks × 4 GB × $0.004445/GB/hour = $0.889/hour
└─ Total: $4.94/hour = $3,556/month (24/7)

Pros:
├─ No idle cost (scale to zero)
├─ No instance management
└─ Pay per task

Cons:
└─ 2-3x more expensive than EC2 for steady workloads
```

**EC2 (On-Demand):**

```
Instances: c5.large (2 vCPU, 4 GB) × 50

Pricing:
├─ 50 × $0.085/hour = $4.25/hour = $3,060/month
└─ Savings vs Fargate: $496/month (14% cheaper)

Pros:
└─ Cheaper for steady workloads

Cons:
├─ Manage EC2 instances
├─ Pay for idle capacity
└─ Manual scaling
```

**EC2 (Reserved Instances, 1-year):**

```
Pricing:
├─ 50 × c5.large Reserved (1-year, no upfront) = $0.055/hour
├─ Total: $2.75/hour = $1,980/month
└─ Savings vs Fargate: $1,576/month (44% cheaper) ✅

Best for: Predictable, steady 24/7 workloads
```

**EC2 (Spot Instances):**

```
Pricing:
├─ 50 × c5.large Spot (90% discount) = $0.0085/hour
├─ Total: $0.425/hour = $306/month
└─ Savings vs Fargate: $3,250/month (91% cheaper) ✅

Best for: Fault-tolerant batch processing
Note: Spot interruptions (3-5% of tasks)
```

**Decision Matrix:**

| Workload Pattern | Best Choice | Monthly Cost | Why |
|------------------|-------------|--------------|-----|
| **Variable (0-50 tasks)** | Fargate | $1,000 avg | No idle cost |
| **Steady 24/7 (50 tasks)** | EC2 Reserved | $1,980 | 44% cheaper |
| **Batch processing** | EC2 Spot | $306 | 91% cheaper |
| **Dev/Test (8hrs/day)** | Fargate | $1,185 | Pay only when running |

**Hybrid Approach (Best of Both):**

```
Baseline (always on): 10 tasks on EC2 Reserved ($396/month)
Peak (variable): 0-40 tasks on Fargate ($0-$2,844/month avg $1,000)

Total: $1,396/month (vs $3,556 all-Fargate or $1,980 all-EC2)
Savings: 61% vs all-Fargate, 30% cheaper than all-EC2 ✅
```

---

**Q9: How do you monitor and troubleshoot containerized applications on ECS/EKS?**

**Answer:**

**Monitoring Strategy:**

**1. CloudWatch Container Insights (ECS/EKS)**

```bash
# Enable Container Insights for ECS cluster
aws ecs update-cluster-settings \
  --cluster my-cluster \
  --settings name=containerInsights,value=enabled

# Enable for EKS cluster
eksctl utils update-cluster-logging \
  --cluster my-eks-cluster \
  --enable-types all

# Metrics available:
# - CPU utilization (per task/pod)
# - Memory utilization
# - Network bytes in/out
# - Disk I/O
# - Task/Pod count
```

**2. Application Logging**

```python
# Container application code
import logging
import json

# Structured logging (JSON format for CloudWatch Insights)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def process_order(order_id):
    logger.info(json.dumps({
        'event': 'order_processing',
        'order_id': order_id,
        'status': 'started',
        'timestamp': datetime.utcnow().isoformat()
    }))
    
    # Processing logic...
    
    logger.info(json.dumps({
        'event': 'order_processing',
        'order_id': order_id,
        'status': 'completed',
        'duration_ms': 150
    }))

# CloudWatch Logs Insights query:
# fields @timestamp, event, order_id, status, duration_ms
# | filter event = "order_processing" and status = "completed"
# | stats avg(duration_ms), max(duration_ms) by bin(5m)
```

**3. Distributed Tracing (AWS X-Ray)**

```python
# Install X-Ray daemon as sidecar container
# Task definition:
{
  "containerDefinitions": [
    {
      "name": "app",
      "image": "my-app:latest"
    },
    {
      "name": "xray-daemon",
      "image": "amazon/aws-xray-daemon",
      "cpu": 32,
      "memoryReservation": 256,
      "portMappings": [{
        "containerPort": 2000,
        "protocol": "udp"
      }]
    }
  ]
}

# Application code (Python)
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

app = Flask(__name__)
XRayMiddleware(app, xray_recorder)

@app.route('/api/orders')
@xray_recorder.capture('process_order')
def process_order():
    # X-Ray automatically traces:
    # - HTTP requests
    # - Database queries
    # - External API calls
    # - Custom subsegments
    
    with xray_recorder.capture('fetch_customer'):
        customer = db.query(...)
    
    with xray_recorder.capture('validate_payment'):
        payment_result = payment_api.charge(...)
    
    return {'order_id': 123}

# X-Ray Console shows:
# - End-to-end latency breakdown
# - Service map (dependencies)
# - Error traces
```

**4. Prometheus + Grafana (EKS)**

```yaml
# Install Prometheus Operator on EKS
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack

# ServiceMonitor for application metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-metrics
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
```

**Common Issues and Troubleshooting:**

| Issue | Symptoms | Diagnosis | Fix |
|-------|----------|-----------|-----|
| **OOM (Out of Memory)** | Tasks killed, Exit code 137 | CloudWatch: Memory > 95% | Increase task memory or optimize code |
| **Slow startup** | ELB health checks fail | CloudWatch Logs: Slow initialization | Increase health check grace period |
| **High CPU** | Tasks throttled | CloudWatch: CPU > 80% sustained | Scale out or optimize code |
| **Network timeouts** | 504 errors | X-Ray: Slow downstream calls | Increase task timeout, check dependencies |
| **Image pull errors** | Tasks fail to start | CloudWatch Events: ECR auth failure | Fix task execution role permissions |

---

**Q10: How do you implement secrets management in ECS/EKS?**

**Answer:**

**ECS Secrets Management:**

**1. AWS Secrets Manager (Recommended)**

```json
// Task definition with secrets
{
  "family": "my-app",
  "containerDefinitions": [{
    "name": "app",
    "image": "my-app:latest",
    "secrets": [
      {
        "name": "DB_PASSWORD",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:db-password-AbCdEf"
      },
      {
        "name": "API_KEY",
        "valueFrom": "arn:aws:secretsmanager:us-east-1:123:secret:api-key-XyZ123"
      }
    ],
    "environment": [
      {"name": "DB_HOST", "value": "database.example.com"}
    ]
  }],
  "executionRoleArn": "arn:aws:iam::123:role/ecsTaskExecutionRole"  // Needs secretsmanager:GetSecretValue
}
```

**2. Systems Manager Parameter Store**

```json
// For non-sensitive configuration
{
  "secrets": [
    {
      "name": "FEATURE_FLAG",
      "valueFrom": "arn:aws:ssm:us-east-1:123:parameter/app/feature-flag"
    }
  ]
}

// Cost: Free (Standard parameters), $0.05/advanced parameter/month
```

**EKS Secrets Management:**

**1. Kubernetes Secrets (Base64 encoded, not encrypted)**

```yaml
# Create secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SecurePassword123

# Use in pod
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: my-app:latest
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

**2. AWS Secrets Manager + External Secrets Operator (Recommended)**

```yaml
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets

# SecretStore (connects to AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

# ExternalSecret (syncs from Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
  target:
    name: db-credentials-k8s  # Creates K8s secret
  data:
    - secretKey: password
      remoteRef:
        key: prod/database/password

# Use in pod (same as regular K8s secret)
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      envFrom:
        - secretRef:
            name: db-credentials-k8s
```

**3. Sealed Secrets (GitOps-friendly)**

```bash
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Create sealed secret (encrypted, safe to commit to Git)
echo -n "SecurePassword123" | kubectl create secret generic db-password \
  --dry-run=client --from-file=password=/dev/stdin -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Commit to Git
git add sealed-secret.yaml
git commit -m "Add database password (encrypted)"

# Apply to cluster
kubectl apply -f sealed-secret.yaml

# Controller decrypts and creates regular secret
```

**Best Practices:**

| Practice | ECS | EKS |
|----------|-----|-----|
| **Secret Rotation** | Secrets Manager auto-rotation | External Secrets Operator sync (1h) |
| **Encryption** | KMS (automatic) | KMS (etcd encryption) |
| **Access Control** | IAM task role | IRSA (IAM Roles for Service Accounts) |
| **Audit** | CloudTrail (GetSecretValue) | CloudTrail + K8s audit logs |
| **Cost** | $0.40/secret/month | Same (External Secrets uses Secrets Manager) |

---

## SCENARIO-BASED QUESTIONS (Q11-Q20)

**Q11: Design a Multi-Region Container Architecture for Disaster Recovery**

**Scenario:** Run containerized data pipeline in us-east-1 (primary) with automatic failover to us-west-2 (DR) if region failure occurs. RPO < 15 minutes, RTO < 30 minutes.

**Solution:** Route 53 health checks + multi-region ECS services + S3 Cross-Region Replication

**Architecture:**
- Primary: ECS in us-east-1, processes data from S3 (CRR enabled)
- DR: ECS in us-west-2 (standby, auto-starts on failover)
- Failover: Route 53 health check fails → DNS updates to us-west-2 ALB
- RTO: 25 minutes (ECS service scale-up: 5 min + warm-up: 20 min)
- Cost: $500/month (primary) + $150/month (DR standby)

---

**Q12: Implement CI/CD Pipeline for Containerized Data Apps**

**Scenario:** Automate Docker build, test, and deployment for data processing containers using AWS CodePipeline.

**Pipeline Stages:**
1. Source: GitHub webhook triggers on commit
2. Build: CodeBuild builds Docker image, runs unit tests
3. Push: Publish to ECR with git commit SHA as tag
4. Deploy Dev: Update ECS service in dev environment
5. Integration Tests: Run automated tests against dev
6. Manual Approval: Data engineering lead approves prod deploy
7. Deploy Prod: Blue-green deployment to production ECS

**Result:** 15-minute deployment (vs 2-hour manual process, 88% faster)

---

**Q13: Cost Optimization for ECS Workloads (99% Savings)**

**Scenario:** Reduce costs for batch data processing workload from $10,000/month to $100/month.

**Strategies:**
- Baseline: 100 × c5.xlarge On-Demand 24/7 = $10,224/month
- Optimized: 
  - ECS with Spot Instances (90% discount) = $1,022/month
  - Auto-scaling (idle to zero) = $300/month average
  - Fargate for small tasks (< 15 min) = $50/month
  - **Total: $370/month (96% savings)** ✅

---

**Q14: Run Apache Spark on EKS with Auto-Scaling**

**Scenario:** Process 10 TB daily using Spark on Kubernetes (EKS) with dynamic resource allocation.

**Implementation:**
- Spark Operator on EKS (manages Spark applications)
- Node auto-scaling: 5 → 100 nodes (r5.4xlarge) based on Spark executors pending
- Spot Instances for executors (70% savings)
- Processing time: 45 minutes (10 TB → 500 GB Parquet)
- Cost: $15/day (vs $120/day EMR, 87% cheaper)

---

**Q15: Secure Multi-Tenant Container Platform**

**Scenario:** Build isolated container environments for 100 data science teams sharing EKS cluster.

**Architecture:**
- Kubernetes Namespaces (one per team)
- Network Policies (isolate inter-team traffic)
- ResourceQuotas (CPU/memory limits per team)
- Pod Security Policies (prevent privileged containers)
- IRSA (IAM roles per namespace for S3 access)

**Result:** 99% isolation, $5,000/month (vs $50,000 for 100 separate clusters)

---

**Q16: Real-Time Stream Processing with ECS + Kinesis**

**Scenario:** Process 1 million events/second from Kinesis using containerized consumers on ECS.

**Architecture:**
- Kinesis Data Streams: 100 shards (10K events/sec each)
- ECS Service: 100 tasks (1 task per shard)
- Auto-scaling: Scale tasks = shard count (automatic)
- Processing latency: < 100ms (p99)
- Cost: $800/month (Kinesis) + $200/month (ECS Fargate)

---

**Q17: GPU-Accelerated ML Training on EKS**

**Scenario:** Train deep learning models using GPU instances (p3.8xlarge) on EKS with auto-scaling.

**Implementation:**
- Node group: p3.8xlarge (4 × V100 GPUs)
- Spot Instances: 70% discount ($12.24/hour vs $40.96 On-Demand)
- Kubeflow Pipelines: Orchestrate training workflows
- Training time: 3 hours/model (vs 12 hours CPU, 75% faster)
- Cost: $36/model (Spot) vs $120 (On-Demand, 70% savings)

---

**Q18: Container Image Vulnerability Management**

**Scenario:** Implement automated vulnerability scanning for 500 container images in ECR.

**Solution:**
- ECR image scanning (on push): Detects CVEs
- EventBridge rule: Trigger Lambda on CRITICAL findings
- Lambda: Create Jira ticket, notify Slack, block deployment
- Quarterly audit: 98% of images have no critical vulnerabilities
- Cost: $0 (ECR scanning included)

---

**Q19: Service Mesh for Microservices Observability**

**Scenario:** Implement AWS App Mesh for 50 microservices on ECS to improve observability.

**Benefits:**
- Automatic mTLS between services (zero trust)
- Distributed tracing (X-Ray integration)
- Traffic shaping (retries, timeouts, circuit breakers)
- Canary deployments (route 10% traffic to new version)
- Monitoring: Reduced error rate from 5% to 0.1% (50x improvement)

---

**Q20: Hybrid Container Deployment (On-Prem + ECS/EKS)**

**Scenario:** Run containers on on-premises Kubernetes and AWS EKS with unified management.

**Architecture:**
- On-premises: Kubernetes cluster (sensitive data processing)
- AWS EKS: Cloud workloads (scalable data analytics)
- AWS Outposts: Bridge on-prem and cloud (low latency)
- GitOps (FluxCD): Deploy to both clusters from single Git repo
- Observability: Prometheus federation (unified metrics)

**Result:** 60% workloads in cloud (cost savings), 40% on-prem (compliance)

---

## MODULE SUMMARY

**Module 6: Containers** covered AWS container orchestration for data engineering:

**Services Mastered:**
- ✅ Amazon ECS (AWS-native orchestration)
- ✅ Amazon EKS (managed Kubernetes)
- ✅ AWS Fargate (serverless containers)
- ✅ Amazon ECR (private registry)
- ✅ Container auto-scaling & cost optimization

**Key Metrics:**
- **Cost Savings:** 90% (Spot Instances), 96% (auto-scaling to zero)
- **Performance:** 75% faster deployments, 99% isolation (multi-tenant)
- **Scaling:** 0 → 500 tasks automatically, 3-20 second startup (Fargate)

**Next Module:** Module 7: Analytics (Athena, Glue, Kinesis, QuickSight)

---

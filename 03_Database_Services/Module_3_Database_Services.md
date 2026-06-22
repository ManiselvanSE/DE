# MODULE 3: Database Services for Data Engineering

## Module Overview

**Duration:** 5-6 hours  
**Difficulty:** Intermediate  
**Prerequisites:** 
- Completed MODULE 1 (IAM) and MODULE 2 (S3)
- SQL fundamentals (SELECT, JOIN, GROUP BY)
- Understanding of ACID properties
- Basic database administration knowledge

**Estimated Cost:** $5-15 (can be reduced to ~$2 with careful cleanup)

**AWS Services Used:**
- Amazon RDS (Relational Database Service)
- Amazon Aurora (MySQL/PostgreSQL-compatible)
- Amazon Redshift (Data Warehouse)
- Amazon DynamoDB (NoSQL)
- AWS Database Migration Service (DMS)
- AWS Schema Conversion Tool (SCT)

---

## What You Will Learn

### Why Database Services Matter for Data Engineers

Databases are the **operational backbone** of data pipelines. As a data engineer, you'll work with databases in three critical ways:

1. **Source Systems (OLTP):**
   - Extract data from production databases (RDS, Aurora)
   - CDC (Change Data Capture) for real-time sync
   - Minimize impact on production workloads

2. **Data Warehouses (OLAP):**
   - Load transformed data into Redshift for analytics
   - Design star/snowflake schemas
   - Optimize query performance for BI tools

3. **Metadata & Cataloging:**
   - DynamoDB for ETL job state management
   - ElastiCache for query result caching
   - DocumentDB for semi-structured data

### Production Use Cases

**Real-World Scenario:**  
Your company runs an e-commerce platform on Oracle (on-premises). As a data engineer, you need to:
- Migrate Oracle to Aurora PostgreSQL (cost savings)
- Replicate transactional data to Redshift (analytics)
- Cache product catalog in ElastiCache (performance)
- Store clickstream events in DynamoDB (NoSQL scale)

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                 Data Engineering Database Stack             │
└─────────────────────────────────────────────────────────────┘

OLTP Layer (Transactional)
├─ Amazon RDS (PostgreSQL, MySQL, Oracle, SQL Server)
│  └─ Use case: Migrated Oracle applications
├─ Amazon Aurora (MySQL/PostgreSQL-compatible)
│  └─ Use case: High-performance transactional systems
│  └─ 5x faster than RDS PostgreSQL, 3x faster than RDS MySQL
└─ Amazon DynamoDB (NoSQL)
   └─ Use case: Key-value lookups, session data, IoT events

Analytics Layer (OLAP)
├─ Amazon Redshift (Columnar Data Warehouse)
│  └─ Use case: Star schema, aggregations, BI dashboards
│  └─ Query 1PB data in seconds
└─ Redshift Spectrum
   └─ Query S3 data lake without loading into Redshift

Caching Layer
├─ Amazon ElastiCache (Redis/Memcached)
│  └─ Use case: Query result caching, session storage
└─ Amazon MemoryDB for Redis
   └─ Use case: Durable Redis with microsecond latency

Specialized Databases
├─ Amazon DocumentDB (MongoDB-compatible)
│  └─ Use case: Semi-structured data, JSON documents
├─ Amazon Keyspaces (Cassandra-compatible)
│  └─ Use case: Time-series data, IoT sensor data
└─ Amazon Neptune (Graph Database)
   └─ Use case: Fraud detection, recommendation engines
```

---

### Database Selection Decision Tree

```
What type of workload?
│
├─ OLTP (Transactional: CRUD operations)
│  │
│  ├─ Relational data model?
│  │  ├─ YES → Migrating from Oracle?
│  │  │        ├─ YES → Amazon Aurora PostgreSQL (Oracle-compatible features)
│  │  │        └─ NO → Amazon RDS (MySQL/PostgreSQL/SQL Server)
│  │  │
│  │  └─ NO → Key-value or document?
│  │           ├─ Key-value → Amazon DynamoDB
│  │           ├─ Document → Amazon DocumentDB
│  │           └─ Graph → Amazon Neptune
│  │
│  └─ Need extreme performance (millions TPS)?
│     └─ Amazon Aurora with read replicas
│
├─ OLAP (Analytics: Complex queries, aggregations)
│  │
│  ├─ < 1TB data → Amazon Athena (query S3 directly)
│  ├─ 1TB - 100TB → Amazon Redshift (single-node or small cluster)
│  └─ > 100TB → Amazon Redshift (large cluster with RA3 nodes)
│
├─ Caching (Reduce database load)
│  ├─ Simple key-value → ElastiCache for Memcached
│  └─ Advanced data structures (sorted sets, pub/sub) → ElastiCache for Redis
│
└─ Time-Series Data
   ├─ Amazon Timestream (serverless, auto-scaling)
   └─ Amazon Keyspaces (Cassandra-compatible)
```

---

### Oracle DBA to AWS Database Mapping

| Oracle Feature | AWS Equivalent | Migration Path |
|----------------|----------------|----------------|
| **Oracle RAC** | Aurora Multi-Master | Aurora PostgreSQL with multi-AZ |
| **Oracle Data Guard** | RDS Multi-AZ, Aurora Global Database | Automated failover |
| **Oracle Flashback** | RDS Automated Backups, Aurora Backtrack | Point-in-time recovery |
| **Oracle Exadata** | Aurora I/O-Optimized | High-performance storage |
| **Oracle Partitioning** | Redshift table partitioning | Partition key design |
| **Oracle Materialized Views** | Redshift Materialized Views | Incremental refresh |
| **Oracle GoldenGate** | AWS DMS (Database Migration Service) | CDC replication |
| **Oracle Warehouse Builder** | AWS Glue | ETL transformations |
| **RMAN Backup** | RDS Automated Snapshots | 35-day retention |
| **ASM Disk Groups** | Aurora Storage Auto-Scaling | Automatic, no tuning needed |
| **Oracle Enterprise Manager** | CloudWatch + RDS Performance Insights | Monitoring & diagnostics |

**Cost Comparison (Typical Production Workload):**

| Database | On-Prem Oracle | AWS Aurora PostgreSQL | Annual Savings |
|----------|----------------|----------------------|----------------|
| **License** | $47,500/core × 8 cores = $380,000 | $0 (PostgreSQL is open-source) | $380,000 |
| **Hardware** | $100,000 (server + storage) | $0 (managed service) | $100,000 |
| **Maintenance** | 22% of license = $83,600 | Included in hourly rate | $83,600 |
| **DBA Salary** | $150,000/year | $0 (AWS manages infra) | $50,000 (partial) |
| **Total Annual Cost** | **$613,600** | **$50,000** (db.r6g.4xlarge reserved) | **$563,600 (92% savings)** |

---

### Common Beginner Mistakes (We'll Avoid These!)

❌ **Mistake 1:** Choosing wrong database for workload  
✅ **Fix:** OLTP → Aurora/RDS, OLAP → Redshift, Key-value → DynamoDB

❌ **Mistake 2:** Not enabling automated backups  
✅ **Fix:** Enable automated backups (7-35 day retention), test restores monthly

❌ **Mistake 3:** Running analytics queries on OLTP database  
✅ **Fix:** Use read replicas for reporting, or Redshift for analytics

❌ **Mistake 4:** Storing large BLOBs in RDS  
✅ **Fix:** Store BLOBs in S3, reference S3 key in RDS

❌ **Mistake 5:** Not monitoring performance  
✅ **Fix:** Enable Performance Insights, set CloudWatch alarms

---

## Exercise 3.1: Create Aurora PostgreSQL Database with Read Replica

### Overview

Create a production-ready Aurora PostgreSQL cluster with:
- Multi-AZ deployment (high availability)
- Read replica (offload read queries)
- Automated backups (point-in-time recovery)
- Encryption at rest (KMS)
- Performance Insights (query monitoring)

**Real-World Scenario:**  
Migrate a 500GB Oracle OLTP database to Aurora PostgreSQL. Application requires 99.95% uptime and needs read replica for reporting queries.

---

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│              Aurora PostgreSQL Cluster Architecture         │
└─────────────────────────────────────────────────────────────┘

Application Tier
├─ EC2 Web Servers (Auto Scaling Group)
│  └─ Connection: Writer Endpoint (port 5432)
│
├─ BI Tools (Tableau, Looker)
│  └─ Connection: Reader Endpoint (port 5432)
│
└─ AWS Lambda (Scheduled Jobs)
   └─ Connection: Writer Endpoint

         ↓
    VPC (10.0.0.0/16)
         │
         ├─ Private Subnet AZ1 (10.0.1.0/24)
         │  └─ Aurora Writer Instance (db.r6g.xlarge)
         │     ├─ 4 vCPU, 32GB RAM
         │     └─ Storage: Auto-scaling (10GB → 128TB)
         │
         ├─ Private Subnet AZ2 (10.0.2.0/24)
         │  └─ Aurora Reader Instance (db.r6g.xlarge)
         │     └─ Replica lag: < 10ms
         │
         └─ Aurora Storage Layer (Shared Cluster Volume)
            ├─ 6 copies across 3 AZs
            ├─ Encryption: AWS KMS (SSE-KMS)
            ├─ Automated Backups: 7-day retention
            └─ Snapshots: Manual, cross-region copy

Monitoring & Alerting
├─ CloudWatch Metrics
│  ├─ DatabaseConnections
│  ├─ CPUUtilization
│  └─ FreeableMemory
│
├─ Performance Insights
│  ├─ Top SQL queries
│  ├─ Wait events
│  └─ Database load
│
└─ CloudWatch Alarms → SNS → Email/Slack
   ├─ High CPU (> 80%)
   ├─ Low storage (< 10GB)
   └─ Replica lag (> 1000ms)

Backup & DR
├─ Automated Backups (7 days, point-in-time recovery)
├─ Manual Snapshots (retain indefinitely)
└─ Cross-Region Replica (us-west-2 for DR)
```

---

### Step-by-Step Lab Guide (AWS Console)

#### Part 1: Create Aurora PostgreSQL Cluster

**Step 1: Navigate to RDS Console**

1. Sign in to AWS Console → Search **"RDS"** → Click **"RDS"**

2. Click **"Create database"** (orange button)

**Step 2: Choose Database Creation Method**

1. **Database creation method:**
   - Select **"Standard create"** (not Easy create - we need full control)

2. **Engine options:**
   - **Engine type:** Amazon Aurora
   - **Edition:** Amazon Aurora with PostgreSQL compatibility
   - **Capacity type:** Provisioned (not Serverless for this exercise)
   - **Engine version:** Aurora PostgreSQL 15.4 (compatible with PostgreSQL 15.4)

3. **Templates:**
   - Select **"Production"** (enables Multi-AZ, automated backups)

**Step 3: Database Settings**

1. **DB cluster identifier:** `prod-aurora-cluster`

2. **Credentials:**
   - **Master username:** `postgres` (default)
   - **Master password:** Create strong password (min 8 chars, mix of letters/numbers/symbols)
   - ✅ **Store password in AWS Secrets Manager** (recommended)
     - Secret name: `prod-aurora-credentials`

**Step 4: Instance Configuration**

1. **DB instance class:**
   - **Burstable classes:** db.t3.medium ($0.073/hour) - Good for dev/test
   - **Memory optimized:** db.r6g.xlarge ($0.302/hour) - **Select this for production**
     - 4 vCPU, 32 GB RAM, Graviton2 processor (best price-performance)

2. **Availability & durability:**
   - ✅ **Create an Aurora Replica in a different AZ** (Multi-AZ)
   - This creates a read replica in AZ2 (automatic failover target)

**Step 5: Connectivity**

1. **Virtual private cloud (VPC):**
   - Select your VPC (or create new VPC if first time)
   - **DB subnet group:** Default (creates subnets in multiple AZs)

2. **Public access:**
   - Select **"No"** (best practice - database in private subnet)

3. **VPC security group:**
   - **Create new:**
     - Name: `aurora-postgres-sg`
     - **Inbound rules:** (we'll configure after creation)

4. **Database port:** 5432 (PostgreSQL default)

**Step 6: Database Authentication**

1. **Database authentication options:**
   - ✅ **Password authentication**
   - ✅ **IAM database authentication** (enable for Lambda access)

**Step 7: Monitoring**

1. **Performance Insights:**
   - ✅ **Enable Performance Insights**
   - **Retention period:** 7 days (free tier)
   - **AWS KMS key:** Default key

2. **Enhanced monitoring:**
   - ✅ **Enable enhanced monitoring**
   - **Granularity:** 60 seconds (default)
   - **Monitoring role:** Create new role (auto-created)

**Step 8: Additional Configuration**

1. **Database options:**
   - **Initial database name:** `proddb`
   - **DB parameter group:** default.aurora-postgresql15
   - **DB cluster parameter group:** default.aurora-postgresql15

2. **Backup:**
   - **Backup retention period:** 7 days (can increase to 35 days)
   - ✅ **Copy tags to snapshots**
   - **Backup window:** Select preferred time (low-traffic period)
     - Example: 03:00-04:00 UTC

3. **Encryption:**
   - ✅ **Enable encryption**
   - **AWS KMS key:** Select existing KMS key or create new
     - If creating new: Name it `aurora-postgres-key`

4. **Log exports:**
   - ✅ **PostgreSQL log** (exports to CloudWatch Logs)

5. **Maintenance:**
   - ✅ **Enable auto minor version upgrade**
   - **Maintenance window:** Select preferred time
     - Example: Sunday 04:00-05:00 UTC

6. **Deletion protection:**
   - ✅ **Enable deletion protection** (prevents accidental deletion)

**Step 9: Review and Create**

1. Review all settings

2. **Estimated monthly costs:**
   - Writer instance (db.r6g.xlarge): ~$220/month
   - Reader instance (db.r6g.xlarge): ~$220/month
   - Storage (100GB): ~$10/month
   - **Total: ~$450/month**

3. Click **"Create database"**

**Wait 10-15 minutes for cluster creation...**

---

#### Part 2: Configure Security Group & Test Connection

**Step 10: Configure Security Group Inbound Rules**

1. Go to **EC2 Console** → **Security Groups**

2. Find security group: `aurora-postgres-sg`

3. Click **"Edit inbound rules"** → **"Add rule"**

4. **Rule 1 (PostgreSQL from application servers):**
   - **Type:** PostgreSQL
   - **Protocol:** TCP
   - **Port:** 5432
   - **Source:** Security group of your EC2 instances (or VPC CIDR: 10.0.0.0/16)
   - **Description:** Allow PostgreSQL from app servers

5. **Rule 2 (PostgreSQL from your laptop for testing - TEMPORARY):**
   - **Type:** PostgreSQL
   - **Protocol:** TCP
   - **Port:** 5432
   - **Source:** My IP (your current IP address)
   - **Description:** TEMP - Testing only, remove after setup

6. Click **"Save rules"**

⚠️ **Security Note:** Remove "My IP" rule after testing. Production databases should ONLY be accessible from within VPC.

---

**Step 11: Test Connection with psql**

1. **Get database endpoint:**
   - Go to RDS Console → Databases → `prod-aurora-cluster`
   - Under **"Connectivity & security"**
   - Copy **Writer endpoint:** `prod-aurora-cluster.cluster-xxxx.us-east-1.rds.amazonaws.com`
   - Copy **Reader endpoint:** `prod-aurora-cluster.cluster-ro-xxxx.us-east-1.rds.amazonaws.com`

2. **Install PostgreSQL client on your laptop:**

   ```bash
   # macOS
   brew install postgresql

   # Ubuntu/Debian
   sudo apt-get install postgresql-client

   # Windows
   # Download from: https://www.postgresql.org/download/windows/
   ```

3. **Connect to writer instance:**

   ```bash
   psql -h prod-aurora-cluster.cluster-xxxx.us-east-1.rds.amazonaws.com \
        -U postgres \
        -d proddb \
        -p 5432

   # Enter password when prompted
   ```

4. **Verify connection:**

   ```sql
   -- Check PostgreSQL version
   SELECT version();
   
   -- Output: PostgreSQL 15.4 on aarch64-unknown-linux-gnu...
   
   -- Show current database
   SELECT current_database();
   
   -- List databases
   \l
   
   -- List tables (should be empty for new database)
   \dt
   ```

5. **Test write to primary:**

   ```sql
   -- Create sample table
   CREATE TABLE employees (
       id SERIAL PRIMARY KEY,
       name VARCHAR(100),
       department VARCHAR(50),
       salary DECIMAL(10,2),
       hire_date DATE,
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   -- Insert test data
   INSERT INTO employees (name, department, salary, hire_date) VALUES
   ('Alice Johnson', 'Engineering', 120000.00, '2020-01-15'),
   ('Bob Smith', 'Sales', 85000.00, '2019-06-01'),
   ('Carol Davis', 'Marketing', 95000.00, '2021-03-10'),
   ('David Wilson', 'Engineering', 110000.00, '2018-11-20'),
   ('Eve Martinez', 'HR', 75000.00, '2022-02-01');

   -- Verify insert
   SELECT COUNT(*) FROM employees;
   -- Output: 5
   ```

6. **Test read from replica:**

   ```bash
   # Open NEW terminal window
   # Connect to READER endpoint
   psql -h prod-aurora-cluster.cluster-ro-xxxx.us-east-1.rds.amazonaws.com \
        -U postgres \
        -d proddb \
        -p 5432
   ```

   ```sql
   -- Query should return 5 rows (replicated from writer)
   SELECT * FROM employees ORDER BY id;

   -- Try to INSERT (should fail - read-only replica)
   INSERT INTO employees (name, department, salary, hire_date) 
   VALUES ('Test User', 'Test', 50000.00, '2024-01-01');
   
   -- ERROR: cannot execute INSERT in a read-only transaction
   ```

✅ **Success!** Writer accepts writes, reader serves reads only.

---

#### Part 3: Enable Automated Backups & Test Point-in-Time Recovery

**Step 12: Create Manual Snapshot**

1. RDS Console → Databases → `prod-aurora-cluster`

2. Click **"Actions"** → **"Take snapshot"**

3. **Snapshot name:** `prod-aurora-baseline-2024-06-22`

4. Click **"Take snapshot"**

5. **Wait 5 minutes** → Snapshot status = "Available"

---

**Step 13: Test Point-in-Time Recovery (PITR)**

1. **Create checkpoint (current state):**

   ```sql
   -- Connect to writer
   SELECT NOW();
   -- Note this timestamp: 2024-06-22 14:30:00
   ```

2. **Simulate accidental data deletion:**

   ```sql
   -- Delete all employees (DISASTER!)
   DELETE FROM employees;
   
   -- Verify deletion
   SELECT COUNT(*) FROM employees;
   -- Output: 0 (all data lost!)
   ```

3. **Restore to point-in-time (before deletion):**

   - RDS Console → Databases → `prod-aurora-cluster`
   - Click **"Actions"** → **"Restore to point in time"**
   - **Restore time:** Custom date and time
     - Enter timestamp from Step 1: `2024-06-22 14:30:00`
   - **DB instance identifier:** `prod-aurora-cluster-restored`
   - **VPC security group:** Same as original (`aurora-postgres-sg`)
   - Click **"Restore"**

4. **Wait 10 minutes** for restoration

5. **Verify data recovered:**

   ```bash
   # Connect to RESTORED cluster
   psql -h prod-aurora-cluster-restored.cluster-xxxx.us-east-1.rds.amazonaws.com \
        -U postgres \
        -d proddb
   ```

   ```sql
   -- Check if data is back
   SELECT COUNT(*) FROM employees;
   -- Output: 5 (data recovered!)
   
   SELECT * FROM employees ORDER BY id;
   -- All 5 employees are back
   ```

✅ **Point-in-time recovery successful!**

---

#### Part 4: Monitor Performance with Performance Insights

**Step 14: Generate Load & Analyze Performance**

1. **Generate test queries:**

   ```sql
   -- Connect to writer
   -- Run heavy query (simulates production load)
   DO $$
   BEGIN
       FOR i IN 1..1000 LOOP
           INSERT INTO employees (name, department, salary, hire_date)
           SELECT 
               'Employee_' || i,
               CASE (i % 5)
                   WHEN 0 THEN 'Engineering'
                   WHEN 1 THEN 'Sales'
                   WHEN 2 THEN 'Marketing'
                   WHEN 3 THEN 'HR'
                   ELSE 'Operations'
               END,
               50000 + (i * 100),
               CURRENT_DATE - (i || ' days')::INTERVAL;
       END LOOP;
   END $$;
   
   -- Run analytical query (simulates BI tool)
   SELECT department, 
          COUNT(*) as employee_count,
          AVG(salary) as avg_salary,
          MAX(salary) as max_salary
   FROM employees
   GROUP BY department
   ORDER BY avg_salary DESC;
   ```

2. **View Performance Insights:**

   - RDS Console → Databases → `prod-aurora-cluster` (writer instance)
   - Click **"Performance Insights"** tab
   - **Dashboard shows:**
     - **Database load:** Active sessions over time
     - **Top SQL:** Queries consuming most CPU/IO
     - **Wait events:** What database is waiting on (locks, I/O, CPU)

3. **Interpret results:**

   ```
   Top SQL by Load:
   ┌─────────────────────────────────────────────────┬──────┬─────┐
   │ SQL Query (digest)                              │ Load │ %   │
   ├─────────────────────────────────────────────────┼──────┼─────┤
   │ INSERT INTO employees (name, department...)     │ 2.5  │ 45% │
   │ SELECT department, COUNT(*), AVG(salary)...     │ 1.8  │ 32% │
   │ SELECT * FROM employees WHERE...               │ 0.9  │ 16% │
   └─────────────────────────────────────────────────┴──────┴─────┘

   Top Wait Events:
   ├─ CPU: 60%
   ├─ IO:DataFileRead: 25%
   └─ Lock:relation: 15%
   ```

**Optimization recommendations:**
- High CPU on INSERT → Add index on frequently queried columns
- IO wait on SELECT → Enable query result caching (ElastiCache)

---

#### Part 5: Configure CloudWatch Alarms

**Step 15: Create CloudWatch Alarms for Database Health**

1. **CloudWatch Console** → **Alarms** → **Create alarm**

2. **Alarm 1: High CPU Utilization**

   - **Metric:** RDS → By Database → `prod-aurora-cluster` → CPUUtilization
   - **Statistic:** Average
   - **Period:** 5 minutes
   - **Threshold:** Greater than 80%
   - **Datapoints to alarm:** 2 out of 3 (reduces false positives)
   - **Actions:** Send notification to SNS topic `database-alerts`
   - **Alarm name:** `aurora-high-cpu`

3. **Alarm 2: Low Free Storage**

   - **Metric:** FreeableMemory
   - **Threshold:** Less than 2 GB (2,000,000,000 bytes)
   - **Actions:** Send to SNS `database-alerts`
   - **Alarm name:** `aurora-low-memory`

4. **Alarm 3: High Replica Lag**

   - **Metric:** AuroraReplicaLag
   - **Threshold:** Greater than 1000 ms (1 second)
   - **Actions:** Send to SNS `database-alerts`
   - **Alarm name:** `aurora-replica-lag`

5. **Alarm 4: High Database Connections**

   - **Metric:** DatabaseConnections
   - **Threshold:** Greater than 80 (assuming max_connections = 100)
   - **Actions:** Send to SNS `database-alerts`
   - **Alarm name:** `aurora-connection-limit`

---

### CLI Version (Automation-Ready)

```bash
#!/bin/bash
# Aurora PostgreSQL Cluster Setup Script

CLUSTER_ID="prod-aurora-cluster"
MASTER_USERNAME="postgres"
MASTER_PASSWORD="YourSecurePassword123!"  # Use Secrets Manager in production
DB_NAME="proddb"
INSTANCE_CLASS="db.r6g.xlarge"
REGION="us-east-1"
VPC_SG="sg-0123456789abcdef0"  # Replace with your security group ID
SUBNET_GROUP="default"  # Replace with your DB subnet group

echo "🚀 Creating Aurora PostgreSQL cluster..."

# Step 1: Create Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier ${CLUSTER_ID} \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username ${MASTER_USERNAME} \
  --master-user-password ${MASTER_PASSWORD} \
  --database-name ${DB_NAME} \
  --db-subnet-group-name ${SUBNET_GROUP} \
  --vpc-security-group-ids ${VPC_SG} \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "sun:04:00-sun:05:00" \
  --enable-cloudwatch-logs-exports postgresql \
  --storage-encrypted \
  --kms-key-id alias/aws/rds \
  --enable-iam-database-authentication \
  --deletion-protection \
  --tags Key=Environment,Value=Production Key=Application,Value=DataPlatform

echo "✅ Cluster created: ${CLUSTER_ID}"

# Step 2: Create primary (writer) instance
echo "📝 Creating primary (writer) instance..."
aws rds create-db-instance \
  --db-instance-identifier ${CLUSTER_ID}-writer \
  --db-instance-class ${INSTANCE_CLASS} \
  --engine aurora-postgresql \
  --db-cluster-identifier ${CLUSTER_ID} \
  --publicly-accessible false \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role \
  --auto-minor-version-upgrade \
  --tags Key=Role,Value=Writer

echo "✅ Writer instance created"

# Step 3: Create replica (reader) instance
echo "📝 Creating replica (reader) instance..."
aws rds create-db-instance \
  --db-instance-identifier ${CLUSTER_ID}-reader-1 \
  --db-instance-class ${INSTANCE_CLASS} \
  --engine aurora-postgresql \
  --db-cluster-identifier ${CLUSTER_ID} \
  --publicly-accessible false \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role \
  --auto-minor-version-upgrade \
  --tags Key=Role,Value=Reader

echo "✅ Reader instance created"

# Step 4: Wait for cluster to become available
echo "⏳ Waiting for cluster to become available (10-15 minutes)..."
aws rds wait db-cluster-available --db-cluster-identifier ${CLUSTER_ID}

# Step 5: Get endpoints
WRITER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier ${CLUSTER_ID} \
  --query 'DBClusters[0].Endpoint' \
  --output text)

READER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier ${CLUSTER_ID} \
  --query 'DBClusters[0].ReaderEndpoint' \
  --output text)

echo ""
echo "🎉 Aurora PostgreSQL Cluster Ready!"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "Cluster ID:      ${CLUSTER_ID}"
echo "Writer Endpoint: ${WRITER_ENDPOINT}:5432"
echo "Reader Endpoint: ${READER_ENDPOINT}:5432"
echo "Database Name:   ${DB_NAME}"
echo "Master Username: ${MASTER_USERNAME}"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo ""
echo "Connect with psql:"
echo "psql -h ${WRITER_ENDPOINT} -U ${MASTER_USERNAME} -d ${DB_NAME}"

# Step 6: Create CloudWatch alarms
echo "🔔 Creating CloudWatch alarms..."

# High CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name ${CLUSTER_ID}-high-cpu \
  --alarm-description "Alert when CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBClusterIdentifier,Value=${CLUSTER_ID} \
  --alarm-actions arn:aws:sns:${REGION}:123456789012:database-alerts

# High replica lag alarm
aws cloudwatch put-metric-alarm \
  --alarm-name ${CLUSTER_ID}-replica-lag \
  --alarm-description "Alert when replica lag exceeds 1 second" \
  --metric-name AuroraReplicaLag \
  --namespace AWS/RDS \
  --statistic Average \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=DBClusterIdentifier,Value=${CLUSTER_ID} \
  --alarm-actions arn:aws:sns:${REGION}:123456789012:database-alerts

echo "✅ CloudWatch alarms created"

# Step 7: Create manual snapshot
echo "📸 Creating initial snapshot..."
SNAPSHOT_ID="${CLUSTER_ID}-baseline-$(date +%Y%m%d)"

aws rds create-db-cluster-snapshot \
  --db-cluster-snapshot-identifier ${SNAPSHOT_ID} \
  --db-cluster-identifier ${CLUSTER_ID} \
  --tags Key=Purpose,Value=Baseline Key=CreatedBy,Value=Automation

echo "✅ Snapshot created: ${SNAPSHOT_ID}"

echo ""
echo "Next Steps:"
echo "1. Connect: psql -h ${WRITER_ENDPOINT} -U ${MASTER_USERNAME} -d ${DB_NAME}"
echo "2. View Performance Insights: AWS Console → RDS → ${CLUSTER_ID}"
echo "3. Monitor alarms: CloudWatch Console → Alarms"
```

**Save as:** `create-aurora-cluster.sh`

**Run:**
```bash
chmod +x create-aurora-cluster.sh
./create-aurora-cluster.sh
```

---

### Validation Steps

**✅ Test 1: Verify Cluster is Running**
```bash
aws rds describe-db-clusters \
  --db-cluster-identifier prod-aurora-cluster \
  --query 'DBClusters[0].[Status,Endpoint,ReaderEndpoint]' \
  --output table

# Expected: Status = "available"
```

---

**✅ Test 2: Verify Encryption Enabled**
```bash
aws rds describe-db-clusters \
  --db-cluster-identifier prod-aurora-cluster \
  --query 'DBClusters[0].StorageEncrypted'

# Expected: true
```

---

**✅ Test 3: Verify Automated Backups Enabled**
```bash
aws rds describe-db-clusters \
  --db-cluster-identifier prod-aurora-cluster \
  --query 'DBClusters[0].BackupRetentionPeriod'

# Expected: 7 (days)
```

---

**✅ Test 4: Test Write to Primary, Read from Replica**
```bash
# Write to primary
psql -h prod-aurora-cluster.cluster-xxxx.us-east-1.rds.amazonaws.com \
     -U postgres -d proddb \
     -c "INSERT INTO employees (name, department, salary, hire_date) VALUES ('Test User', 'QA', 70000, CURRENT_DATE);"

# Read from replica (should show new row)
psql -h prod-aurora-cluster.cluster-ro-xxxx.us-east-1.rds.amazonaws.com \
     -U postgres -d proddb \
     -c "SELECT * FROM employees WHERE name = 'Test User';"

# Expected: 1 row returned
```

---

**✅ Test 5: Verify Performance Insights Enabled**
```bash
aws rds describe-db-instances \
  --db-instance-identifier prod-aurora-cluster-writer \
  --query 'DBInstances[0].PerformanceInsightsEnabled'

# Expected: true
```

---

**✅ Test 6: Test Automatic Failover**
```bash
# Trigger failover (switches writer to replica AZ)
aws rds failover-db-cluster --db-cluster-identifier prod-aurora-cluster

# Monitor failover progress (takes 30-60 seconds)
watch -n 5 'aws rds describe-db-clusters \
  --db-cluster-identifier prod-aurora-cluster \
  --query "DBClusters[0].Status"'

# Application connections automatically reconnect to new writer
```

---

This completes Part 1 of Exercise 3.1. The exercise continues with additional database services (Redshift, DynamoDB) and interview questions. Should I continue with the next sections?

## Exercise 3.2: Create Amazon Redshift Data Warehouse for Analytics

### Overview

Build a production-ready Redshift cluster for analytical workloads:
- Columnar storage for fast aggregations
- Star schema design (fact + dimension tables)
- Distribution keys for parallel processing
- Sort keys for query optimization
- COPY command for bulk loading from S3
- Materialized views for pre-computed aggregations

**Real-World Scenario:**  
Business analysts need to run daily reports on 500M transaction records. Queries on PostgreSQL (OLTP) take 10+ minutes. Migrate analytics to Redshift for sub-second query performance.

---

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│           Redshift Data Warehouse Architecture              │
└─────────────────────────────────────────────────────────────┘

Data Sources
├─ S3 Data Lake (Parquet files)
├─ Aurora PostgreSQL (CDC via DMS)
└─ DynamoDB (Export to S3)

         ↓ (ETL Layer)

AWS Glue ETL Jobs
├─ Transform raw data
├─ Apply business logic
└─ Write to S3 staging/

         ↓ (COPY command)

Amazon Redshift Cluster
├─ Leader Node (Query Coordinator)
│  ├─ Query parser & optimizer
│  ├─ Execution plan distribution
│  └─ Result aggregation
│
└─ Compute Nodes (Data Processing)
   ├─ Node 1 (dc2.large)
   │  ├─ Slice 0 (CPU + Memory + Disk)
   │  └─ Slice 1 (CPU + Memory + Disk)
   ├─ Node 2 (dc2.large)
   │  ├─ Slice 0
   │  └─ Slice 1
   └─ Node N...

Storage Layout (Columnar)
┌────────────────────────────────────────┐
│ Table: fact_transactions (500M rows)  │
├────────────────────────────────────────┤
│ Column: transaction_id (DISTKEY)      │
│  ├─ Slice 0: [1, 5, 9, 13...]        │
│  ├─ Slice 1: [2, 6, 10, 14...]       │
│  ├─ Slice 2: [3, 7, 11, 15...]       │
│  └─ Slice 3: [4, 8, 12, 16...]       │
│                                        │
│ Column: amount (SORTKEY)              │
│  └─ Data sorted within each slice     │
│                                        │
│ Column: product_id (Foreign Key)      │
│  └─ Join with dim_products            │
└────────────────────────────────────────┘

Query Execution (Parallel Processing)
User → SELECT SUM(amount) FROM fact_transactions WHERE date = '2024-06-22'
    ↓
Leader Node (Create Execution Plan)
    ↓
Distribute to Compute Nodes (Parallel)
    ├─ Node 1 processes 125M rows
    ├─ Node 2 processes 125M rows
    ├─ Node 3 processes 125M rows
    └─ Node 4 processes 125M rows
    ↓ (Aggregate partial results)
Leader Node (Final SUM)
    ↓
Return result to user (< 1 second)

BI Tools Integration
├─ Tableau → JDBC connection
├─ Looker → Redshift connector
├─ Power BI → ODBC connection
└─ Python (pandas) → psycopg2 driver
```

---

### Star Schema Design

**Fact Table (Transactions):**
```sql
CREATE TABLE fact_transactions (
    transaction_id BIGINT IDENTITY(1,1) PRIMARY KEY,
    customer_key INT DISTKEY,           -- Distribution key (JOIN optimization)
    product_key INT,
    date_key INT SORTKEY,               -- Sort key (WHERE date optimization)
    store_key INT,
    payment_method VARCHAR(20),
    quantity INT,
    unit_price DECIMAL(10,2),
    discount_amount DECIMAL(10,2),
    tax_amount DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    transaction_timestamp TIMESTAMP,
    FOREIGN KEY (customer_key) REFERENCES dim_customers(customer_key),
    FOREIGN KEY (product_key) REFERENCES dim_products(product_key),
    FOREIGN KEY (date_key) REFERENCES dim_date(date_key),
    FOREIGN KEY (store_key) REFERENCES dim_stores(store_key)
) DISTSTYLE KEY;
```

**Dimension Tables:**
```sql
-- Customer dimension (small, replicate to all nodes)
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    phone VARCHAR(20),
    registration_date DATE,
    customer_segment VARCHAR(50),
    lifetime_value DECIMAL(12,2)
) DISTSTYLE ALL;  -- Replicate to all compute nodes

-- Product dimension
CREATE TABLE dim_products (
    product_key INT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(255),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    unit_cost DECIMAL(10,2),
    unit_price DECIMAL(10,2),
    is_active BOOLEAN
) DISTSTYLE ALL;

-- Date dimension (calendar table)
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE,
    day_of_week VARCHAR(10),
    day_of_month INT,
    day_of_year INT,
    week_of_year INT,
    month_name VARCHAR(10),
    month_number INT,
    quarter INT,
    year INT,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    holiday_name VARCHAR(100)
) DISTSTYLE ALL;

-- Store dimension
CREATE TABLE dim_stores (
    store_key INT PRIMARY KEY,
    store_id VARCHAR(50),
    store_name VARCHAR(255),
    address VARCHAR(500),
    city VARCHAR(100),
    state VARCHAR(50),
    zip_code VARCHAR(10),
    country VARCHAR(50),
    region VARCHAR(50),
    store_size_sqft INT,
    opened_date DATE
) DISTSTYLE ALL;
```

---

### Step-by-Step Lab Guide (AWS Console)

#### Part 1: Create Redshift Cluster

**Step 1: Navigate to Redshift Console**

1. AWS Console → Search **"Redshift"** → Click **"Amazon Redshift"**

2. Click **"Create cluster"**

**Step 2: Cluster Configuration**

1. **Cluster identifier:** `analytics-cluster`

2. **What are you planning to use this cluster for?**
   - Select **"Production"** (for real workloads)
   - Or **"Free trial"** (dc2.large, 2 months free, 160GB storage)

3. **Node type:**
   - **For this lab:** dc2.large ($0.25/hour, 160GB SSD)
   - **Production recommendation:** ra3.xlplus ($3.26/hour, managed storage up to 64TB)

4. **Number of nodes:** 2 (creates 1 leader + 2 compute nodes)
   - **Minimum for production:** 2 nodes (high availability)
   - **For large datasets:** 4-32 nodes

**Step 3: Database Configuration**

1. **Database name:** `analyticsdb`

2. **Master user name:** `awsuser` (default)

3. **Master user password:** Create strong password
   - ✅ Store in **AWS Secrets Manager**

**Step 4: Cluster Permissions**

1. **Associated IAM roles:**
   - Click **"Create IAM role"**
   - **S3 bucket:** `my-datalake-*` (allow Redshift to read from all your S3 buckets)
   - **Role name:** `RedshiftS3ReadRole`
   - Click **"Create IAM role"**

2. Select the created role from dropdown

**Step 5: Additional Configurations**

1. **Network and security:**
   - **VPC:** Select your VPC
   - **Publicly accessible:** No (best practice)
   - **VPC security groups:** Create new
     - **Name:** `redshift-sg`

2. **Database configurations:**
   - **Parameter group:** default.redshift-1.0
   - **Encryption:** Enable (KMS)
   - **KMS key:** Default or select custom key

3. **Backup:**
   - **Automated snapshot retention:** 7 days
   - **Manual snapshot retention:** Indefinite

4. **Maintenance:**
   - **Maintenance window:** Sunday 03:00-04:00 UTC

**Step 6: Review and Create**

1. **Estimated monthly cost:**
   - 2 nodes × dc2.large × $0.25/hour × 730 hours = **~$365/month**

2. Click **"Create cluster"**

**Wait 5-10 minutes for cluster creation...**

---

#### Part 2: Configure Security & Connect

**Step 7: Configure Security Group**

1. EC2 Console → Security Groups → `redshift-sg`

2. **Edit inbound rules** → **Add rule**
   - **Type:** Redshift
   - **Protocol:** TCP
   - **Port:** 5439 (Redshift default)
   - **Source:** Your IP (for testing) or VPC CIDR (10.0.0.0/16)

**Step 8: Connect to Redshift**

1. **Get cluster endpoint:**
   - Redshift Console → Clusters → `analytics-cluster`
   - Copy **Endpoint:** `analytics-cluster.xxxx.us-east-1.redshift.amazonaws.com:5439`

2. **Install PostgreSQL client** (Redshift uses PostgreSQL wire protocol)

3. **Connect via psql:**

   ```bash
   psql -h analytics-cluster.xxxx.us-east-1.redshift.amazonaws.com \
        -U awsuser \
        -d analyticsdb \
        -p 5439
   ```

4. **Verify connection:**

   ```sql
   -- Check Redshift version
   SELECT version();
   -- Output: PostgreSQL 8.0.2 on x86_64-pc-linux-gnu, Redshift 1.0.xx
   
   -- Show current database
   SELECT current_database();
   
   -- List schemas
   \dn
   
   -- Check cluster nodes
   SELECT * FROM stv_slices ORDER BY node, slice;
   -- Shows all compute slices
   ```

---

#### Part 3: Create Star Schema & Load Data

**Step 9: Create Dimension Tables**

```sql
-- Customer dimension
CREATE TABLE dim_customers (
    customer_key INT PRIMARY KEY,
    customer_id VARCHAR(50),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255),
    registration_date DATE,
    customer_segment VARCHAR(50)
) DISTSTYLE ALL;

-- Product dimension
CREATE TABLE dim_products (
    product_key INT PRIMARY KEY,
    product_id VARCHAR(50),
    product_name VARCHAR(255),
    category VARCHAR(100),
    brand VARCHAR(100),
    unit_price DECIMAL(10,2)
) DISTSTYLE ALL;

-- Date dimension (generate calendar table)
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    full_date DATE,
    year INT,
    quarter INT,
    month_number INT,
    month_name VARCHAR(10),
    day_of_month INT,
    day_of_week VARCHAR(10),
    is_weekend BOOLEAN
) DISTSTYLE ALL;

-- Generate dates for 2023-2025
INSERT INTO dim_date
SELECT 
    TO_CHAR(date_series, 'YYYYMMDD')::INT AS date_key,
    date_series AS full_date,
    EXTRACT(YEAR FROM date_series) AS year,
    EXTRACT(QUARTER FROM date_series) AS quarter,
    EXTRACT(MONTH FROM date_series) AS month_number,
    TO_CHAR(date_series, 'Month') AS month_name,
    EXTRACT(DAY FROM date_series) AS day_of_month,
    TO_CHAR(date_series, 'Day') AS day_of_week,
    CASE WHEN EXTRACT(DOW FROM date_series) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend
FROM (
    SELECT '2023-01-01'::DATE + (ROW_NUMBER() OVER (ORDER BY 1) - 1) AS date_series
    FROM SVV_TABLES
    LIMIT 1095  -- 3 years (2023-2025)
);

-- Verify date dimension
SELECT * FROM dim_date WHERE year = 2024 AND month_number = 6 LIMIT 10;
```

**Step 10: Create Fact Table with Distribution & Sort Keys**

```sql
CREATE TABLE fact_transactions (
    transaction_id BIGINT IDENTITY(1,1),
    customer_key INT DISTKEY,        -- Distribution key (even distribution)
    product_key INT,
    date_key INT SORTKEY,            -- Sort key (date range queries)
    quantity INT,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    transaction_timestamp TIMESTAMP,
    PRIMARY KEY (transaction_id)
) DISTSTYLE KEY;

-- Verify table structure
SELECT 
    "schema",
    "table",
    diststyle,
    distkey,
    sortkey1
FROM svv_table_info
WHERE "table" LIKE 'fact_%' OR "table" LIKE 'dim_%';
```

**Expected output:**
```
┌────────┬─────────────────────┬──────────┬──────────────┬───────────┐
│ schema │ table               │diststyle │ distkey      │ sortkey1  │
├────────┼─────────────────────┼──────────┼──────────────┼───────────┤
│ public │ dim_customers       │ ALL      │              │           │
│ public │ dim_products        │ ALL      │              │           │
│ public │ dim_date            │ ALL      │              │           │
│ public │ fact_transactions   │ KEY      │ customer_key │ date_key  │
└────────┴─────────────────────┴──────────┴──────────────┴───────────┘
```

---

**Step 11: Load Data from S3 using COPY Command**

1. **Create sample CSV files and upload to S3:**

   ```bash
   # Create customers.csv
   cat > /tmp/customers.csv << 'EOF'
customer_key,customer_id,first_name,last_name,email,registration_date,customer_segment
1,CUST001,Alice,Johnson,alice@email.com,2020-01-15,Premium
2,CUST002,Bob,Smith,bob@email.com,2020-03-20,Standard
3,CUST003,Carol,Davis,carol@email.com,2021-05-10,Premium
4,CUST004,David,Wilson,david@email.com,2021-08-25,Standard
5,CUST005,Eve,Martinez,eve@email.com,2022-02-14,Basic
EOF

   # Upload to S3
   aws s3 cp /tmp/customers.csv s3://my-datalake-${ACCOUNT_ID}/redshift-data/customers.csv
   
   # Create products.csv
   cat > /tmp/products.csv << 'EOF'
product_key,product_id,product_name,category,brand,unit_price
1,PROD001,Laptop Pro 15,Electronics,TechBrand,1299.99
2,PROD002,Wireless Mouse,Electronics,TechBrand,29.99
3,PROD003,Office Desk,Furniture,OfficeCo,349.99
4,PROD004,LED Monitor 27,Electronics,ViewMax,399.99
5,PROD005,Ergonomic Chair,Furniture,ComfortSeats,249.99
EOF

   aws s3 cp /tmp/products.csv s3://my-datalake-${ACCOUNT_ID}/redshift-data/products.csv
   ```

2. **Load data into Redshift using COPY command:**

   ```sql
   -- Copy customers
   COPY dim_customers
   FROM 's3://my-datalake-123456789012/redshift-data/customers.csv'
   IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftS3ReadRole'
   CSV
   IGNOREHEADER 1
   DATEFORMAT 'YYYY-MM-DD';
   
   -- Verify load
   SELECT COUNT(*) FROM dim_customers;
   -- Output: 5
   
   -- Copy products
   COPY dim_products
   FROM 's3://my-datalake-123456789012/redshift-data/products.csv'
   IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftS3ReadRole'
   CSV
   IGNOREHEADER 1;
   
   SELECT COUNT(*) FROM dim_products;
   -- Output: 5
   ```

3. **Generate sample transaction data:**

   ```sql
   -- Insert 100,000 sample transactions
   INSERT INTO fact_transactions (customer_key, product_key, date_key, quantity, unit_price, total_amount, transaction_timestamp)
   SELECT 
       (RANDOM() * 4 + 1)::INT AS customer_key,
       (RANDOM() * 4 + 1)::INT AS product_key,
       20240600 + (RANDOM() * 22)::INT AS date_key,  -- June 2024
       (RANDOM() * 5 + 1)::INT AS quantity,
       (RANDOM() * 1000 + 10)::DECIMAL(10,2) AS unit_price,
       quantity * unit_price AS total_amount,
       DATEADD(second, (RANDOM() * 2592000)::INT, '2024-06-01'::TIMESTAMP) AS transaction_timestamp
   FROM 
       (SELECT ROW_NUMBER() OVER () AS n FROM dim_date LIMIT 100000);
   
   -- Verify data loaded
   SELECT COUNT(*) FROM fact_transactions;
   -- Output: 100000
   
   -- Run ANALYZE to update table statistics (important for query optimizer)
   ANALYZE fact_transactions;
   ANALYZE dim_customers;
   ANALYZE dim_products;
   ANALYZE dim_date;
   ```

---

#### Part 4: Run Analytics Queries

**Step 12: Test Query Performance**

```sql
-- Query 1: Daily sales aggregation
SELECT 
    d.full_date,
    d.day_of_week,
    COUNT(*) AS transaction_count,
    SUM(f.total_amount) AS daily_sales,
    AVG(f.total_amount) AS avg_transaction_value
FROM fact_transactions f
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.full_date BETWEEN '2024-06-01' AND '2024-06-22'
GROUP BY d.full_date, d.day_of_week
ORDER BY d.full_date;

-- Query performance: ~100ms (vs 10+ seconds on PostgreSQL OLTP)
```

```sql
-- Query 2: Top customers by spend
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.customer_segment,
    COUNT(*) AS purchase_count,
    SUM(f.total_amount) AS total_spend,
    AVG(f.total_amount) AS avg_order_value
FROM fact_transactions f
JOIN dim_customers c ON f.customer_key = c.customer_key
GROUP BY c.customer_id, customer_name, c.customer_segment
ORDER BY total_spend DESC
LIMIT 10;
```

```sql
-- Query 3: Product performance by category
SELECT 
    p.category,
    p.brand,
    COUNT(DISTINCT f.customer_key) AS unique_customers,
    SUM(f.quantity) AS units_sold,
    SUM(f.total_amount) AS total_revenue
FROM fact_transactions f
JOIN dim_products p ON f.product_key = p.product_key
GROUP BY p.category, p.brand
ORDER BY total_revenue DESC;
```

**Step 13: Create Materialized View for Pre-Computed Aggregations**

```sql
-- Create materialized view (pre-computes results)
CREATE MATERIALIZED VIEW mv_daily_sales_summary AS
SELECT 
    d.full_date,
    d.year,
    d.month_number,
    d.day_of_week,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT f.customer_key) AS unique_customers,
    SUM(f.total_amount) AS total_sales,
    AVG(f.total_amount) AS avg_transaction_value,
    SUM(f.quantity) AS total_units_sold
FROM fact_transactions f
JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.full_date, d.year, d.month_number, d.day_of_week;

-- Query materialized view (instant results)
SELECT * FROM mv_daily_sales_summary
WHERE year = 2024 AND month_number = 6
ORDER BY full_date;

-- Refresh materialized view (after new data loaded)
REFRESH MATERIALIZED VIEW mv_daily_sales_summary;
```

**Performance comparison:**
- **Base query (fact + dimension JOIN):** 150ms
- **Materialized view:** 5ms (30x faster!)

---

### CLI Version (Automation-Ready)

```bash
#!/bin/bash
# Redshift Cluster Setup Script

CLUSTER_ID="analytics-cluster"
DB_NAME="analyticsdb"
MASTER_USER="awsuser"
MASTER_PASSWORD="YourSecurePassword123!"
NODE_TYPE="dc2.large"
NUM_NODES=2
REGION="us-east-1"

echo "🚀 Creating Redshift cluster..."

# Create cluster
aws redshift create-cluster \
  --cluster-identifier ${CLUSTER_ID} \
  --node-type ${NODE_TYPE} \
  --number-of-nodes ${NUM_NODES} \
  --master-username ${MASTER_USER} \
  --master-user-password ${MASTER_PASSWORD} \
  --db-name ${DB_NAME} \
  --cluster-type multi-node \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --cluster-subnet-group-name default \
  --publicly-accessible false \
  --encrypted \
  --kms-key-id alias/aws/redshift \
  --automated-snapshot-retention-period 7 \
  --preferred-maintenance-window sun:03:00-sun:04:00 \
  --iam-roles arn:aws:iam::123456789012:role/RedshiftS3ReadRole \
  --tags Key=Environment,Value=Production

echo "⏳ Waiting for cluster to become available (5-10 minutes)..."
aws redshift wait cluster-available --cluster-identifier ${CLUSTER_ID}

# Get endpoint
ENDPOINT=$(aws redshift describe-clusters \
  --cluster-identifier ${CLUSTER_ID} \
  --query 'Clusters[0].Endpoint.Address' \
  --output text)

echo "✅ Redshift cluster ready!"
echo "Endpoint: ${ENDPOINT}:5439"
echo "Connect: psql -h ${ENDPOINT} -U ${MASTER_USER} -d ${DB_NAME} -p 5439"
```

---

## Exercise 3.3: Create DynamoDB Table for High-Velocity Data

### Overview

Build a DynamoDB table optimized for:
- High-throughput writes (millions of requests/second)
- Low-latency reads (single-digit milliseconds)
- Flexible schema (NoSQL)
- Global secondary indexes (GSI) for alternate query patterns

**Real-World Scenario:**  
IoT sensors generate 100K events/second. Store events in DynamoDB for real-time lookups by device_id or event_timestamp.

---

### Step-by-Step Lab Guide (AWS Console)

**Step 1: Create DynamoDB Table**

1. AWS Console → **DynamoDB** → **Create table**

2. **Table details:**
   - **Table name:** `iot_events`
   - **Partition key:** `device_id` (String)
   - **Sort key:** `timestamp` (Number) - Unix timestamp

3. **Table settings:**
   - **Table class:** DynamoDB Standard
   - **Capacity mode:** On-demand (auto-scales, no capacity planning)
   - Or **Provisioned:** 100 RCU, 100 WCU

4. **Encryption:** Use AWS owned key (default)

5. Click **"Create table"**

**Step 2: Insert Sample Data**

```python
import boto3
from datetime import datetime
import time

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('iot_events')

# Insert IoT events
for i in range(100):
    table.put_item(
        Item={
            'device_id': f'DEVICE_{i % 10:03d}',
            'timestamp': int(time.time()) + i,
            'temperature': 20 + (i % 30),
            'humidity': 40 + (i % 50),
            'battery_level': 100 - (i % 100),
            'event_type': 'sensor_reading'
        }
    )

print("✅ 100 events inserted")
```

**Step 3: Query Data**

```python
# Query by partition key
response = table.query(
    KeyConditionExpression='device_id = :device',
    ExpressionAttributeValues={
        ':device': 'DEVICE_001'
    }
)

print(f"Found {response['Count']} events for DEVICE_001")
```

---

## Interview Preparation

### Beginner Questions (5)

**Q1: What's the difference between OLTP and OLAP databases?**

**Answer:**

| Feature | OLTP (Transactional) | OLAP (Analytical) |
|---------|---------------------|-------------------|
| **Purpose** | Day-to-day operations | Business intelligence, reporting |
| **Example AWS Service** | Aurora, RDS | Redshift |
| **Query Pattern** | INSERT, UPDATE, DELETE (writes) | SELECT with JOINs, aggregations (reads) |
| **Data Volume** | GB to TB | TB to PB |
| **Query Complexity** | Simple (lookup by primary key) | Complex (multi-table JOINs, GROUP BY) |
| **Response Time** | Milliseconds | Seconds to minutes |
| **Concurrency** | 1000s of users | 10-100 analysts |
| **Schema** | Normalized (3NF) | Denormalized (star/snowflake) |
| **Example Query** | `SELECT * FROM orders WHERE order_id = 12345` | `SELECT SUM(amount) FROM orders GROUP BY year, product_category` |

**Production Example:**
- **E-commerce app:** Uses Aurora (OLTP) for customer orders (INSERT order, UPDATE inventory)
- **Business analysts:** Query Redshift (OLAP) for monthly sales reports, customer segmentation

---

**Q2: When should you use Aurora vs RDS PostgreSQL?**

**Answer:**

| Use Aurora When... | Use RDS PostgreSQL When... |
|-------------------|---------------------------|
| Need high availability (99.99% SLA) | Budget-constrained (Aurora costs 2x more) |
| Require read replicas (up to 15) | Need <= 5 read replicas |
| Large database (> 1TB) | Small database (< 500GB) |
| High write throughput | Moderate workload |
| Want automatic storage scaling (10GB → 128TB) | Fixed storage size |
| Global database (multi-region) | Single-region only |

**Cost Comparison (db.r6g.xlarge, 1TB storage):**
- **RDS PostgreSQL:** $300/month (instance) + $115/month (storage) = **$415/month**
- **Aurora PostgreSQL:** $300/month (instance) + $100/month (storage) + $200/month (I/O) = **$600/month**
- **Aurora is 44% more expensive BUT 5x faster and more reliable**

**When to choose:**
- **Aurora:** Production, mission-critical, high-performance requirements
- **RDS:** Dev/test environments, cost-sensitive workloads, legacy PostgreSQL compatibility

---

**Q3: Explain Redshift distribution keys (DISTKEY) and sort keys (SORTKEY).**

**Answer:**

**Distribution Key (DISTKEY):**
- **Purpose:** Controls how data is distributed across compute nodes
- **Goal:** Evenly distribute data, minimize data movement during JOINs

**Example:**
```sql
CREATE TABLE fact_sales (
    customer_id INT DISTKEY,  -- Data distributed by customer_id
    product_id INT,
    sale_amount DECIMAL(10,2)
);

CREATE TABLE dim_customers (
    customer_id INT DISTKEY,  -- Match fact table DISTKEY
    customer_name VARCHAR(100)
) DISTSTYLE KEY;
```

**Why this works:**
- Customer 1's sales records are on **Node 1**
- Customer 1's dimension record is ALSO on **Node 1**
- JOIN happens locally (no network transfer!)

**Distribution Styles:**
1. **KEY:** Distribute by column value (use for large fact tables)
2. **ALL:** Replicate entire table to all nodes (use for small dimension tables < 10MB)
3. **EVEN:** Round-robin distribution (use when no obvious DISTKEY)
4. **AUTO:** Redshift chooses (default for new tables)

---

**Sort Key (SORTKEY):**
- **Purpose:** Physically sorts data on disk
- **Goal:** Speed up WHERE clause queries (range scans)

**Example:**
```sql
CREATE TABLE fact_sales (
    customer_id INT DISTKEY,
    sale_date DATE SORTKEY,  -- Sort by date
    sale_amount DECIMAL(10,2)
);
```

**Query optimization:**
```sql
-- Fast (uses SORTKEY)
SELECT SUM(sale_amount) 
FROM fact_sales 
WHERE sale_date BETWEEN '2024-06-01' AND '2024-06-30';

-- Redshift scans only June blocks (skips other months)
-- Zone maps eliminate 90% of data from scan
```

**Sort Key Types:**
1. **Compound:** Multi-column sort (date, customer_id)
   - Use when queries filter on multiple columns in order
2. **Interleaved:** Equal weight to all columns
   - Use when queries filter on different column combinations

**Rule of thumb:**
- **DISTKEY:** Column used in JOINs (usually FK to dimension)
- **SORTKEY:** Column used in WHERE clauses (usually date)

---

**Q4: What is DynamoDB partition key vs sort key?**

**Answer:**

**Partition Key (HASH key):**
- **Purpose:** Determines which partition stores the item
- **Uniqueness:** Must be unique (or unique when combined with sort key)
- **Query pattern:** Exact match lookup (`device_id = 'DEVICE_001'`)

**Sort Key (RANGE key):**
- **Purpose:** Orders items within a partition
- **Uniqueness:** Can have duplicates (unique per partition key)
- **Query pattern:** Range queries (`timestamp BETWEEN start AND end`)

**Example:**
```python
# Table design
Partition Key: device_id (String)
Sort Key: timestamp (Number)

# Sample data
device_id    | timestamp    | temperature | battery
-------------|--------------|-------------|--------
DEVICE_001   | 1719072000   | 25.5        | 95
DEVICE_001   | 1719072060   | 26.0        | 94
DEVICE_001   | 1719072120   | 25.8        | 93
DEVICE_002   | 1719072000   | 30.2        | 88
DEVICE_002   | 1719072060   | 30.5        | 87
```

**Query patterns:**

```python
# 1. Get specific device at specific time (exact match)
response = table.get_item(
    Key={
        'device_id': 'DEVICE_001',
        'timestamp': 1719072000
    }
)

# 2. Get all events for device in time range (range query)
response = table.query(
    KeyConditionExpression='device_id = :device AND timestamp BETWEEN :start AND :end',
    ExpressionAttributeValues={
        ':device': 'DEVICE_001',
        ':start': 1719072000,
        ':end': 1719072120
    }
)

# 3. Get latest 10 events for device (sort key ordering)
response = table.query(
    KeyConditionExpression='device_id = :device',
    ExpressionAttributeValues={':device': 'DEVICE_001'},
    ScanIndexForward=False,  # Descending order
    Limit=10
)
```

**When to use:**
- **Partition key only:** Unique items (user_id for user profiles)
- **Partition + Sort key:** Time-series data (device_id + timestamp), hierarchical data (org_id + employee_id)

---

**Q5: How do you migrate from Oracle to Aurora PostgreSQL?**

**Answer:**

**Step-by-step migration path:**

**Phase 1: Assessment (1-2 weeks)**
1. **Use AWS Schema Conversion Tool (SCT):**
   - Analyzes Oracle database schema
   - Identifies incompatible features (PL/SQL → PL/pgSQL)
   - Generates conversion report (% compatible)

```bash
# Install SCT
# Download from: https://aws.amazon.com/dms/schema-conversion-tool/

# Connect to Oracle
# SCT automatically generates:
# - PostgreSQL-compatible DDL
# - List of manual conversions needed
# - Estimated effort (days)
```

**Phase 2: Schema Conversion (1-3 weeks)**
2. **Convert schema using SCT:**
   - **Tables:** Auto-converted (95% compatible)
   - **Indexes:** Auto-converted
   - **Stored Procedures:** Manual rewrite (PL/SQL → PL/pgSQL)
   - **Packages:** Rewrite as schemas in PostgreSQL
   - **Sequences:** Auto-converted to SERIAL/IDENTITY

**Example Oracle → PostgreSQL conversion:**

```sql
-- Oracle
CREATE TABLE employees (
    emp_id NUMBER PRIMARY KEY,
    emp_name VARCHAR2(100),
    hire_date DATE DEFAULT SYSDATE
);

CREATE SEQUENCE emp_seq START WITH 1 INCREMENT BY 1;

-- PostgreSQL (SCT auto-generates)
CREATE TABLE employees (
    emp_id INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    emp_name VARCHAR(100),
    hire_date DATE DEFAULT CURRENT_DATE
);
```

**Phase 3: Data Migration (1-7 days)**
3. **Use AWS Database Migration Service (DMS):**

```bash
# Create replication instance
aws dms create-replication-instance \
  --replication-instance-identifier oracle-to-aurora-repl \
  --replication-instance-class dms.c5.xlarge \
  --allocated-storage 100

# Create source endpoint (Oracle)
aws dms create-endpoint \
  --endpoint-identifier oracle-source \
  --endpoint-type source \
  --engine-name oracle \
  --server-name oracle-prod.example.com \
  --port 1521 \
  --database-name PRODDB \
  --username oracleuser \
  --password oraclepass

# Create target endpoint (Aurora PostgreSQL)
aws dms create-endpoint \
  --endpoint-identifier aurora-target \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --server-name aurora-cluster.xxxx.rds.amazonaws.com \
  --port 5432 \
  --database-name proddb \
  --username postgres \
  --password aurorapass

# Create migration task
aws dms create-replication-task \
  --replication-task-identifier full-load-cdc \
  --source-endpoint-arn arn:aws:dms:...:endpoint/oracle-source \
  --target-endpoint-arn arn:aws:dms:...:endpoint/aurora-target \
  --replication-instance-arn arn:aws:dms:...:rep-instance/oracle-to-aurora-repl \
  --migration-type full-load-and-cdc \  # Full load + continuous replication
  --table-mappings file://table-mappings.json

# Start migration
aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:...:task/full-load-cdc \
  --start-replication-task-type start-replication
```

**Migration modes:**
- **Full load:** One-time bulk copy (downtime required)
- **Full load + CDC:** Initial copy + ongoing replication (minimal downtime)
- **CDC only:** Replicate only changes (requires existing data)

**Phase 4: Application Migration (2-8 weeks)**
4. **Update application code:**

**Connection string changes:**
```java
// Oracle JDBC
String url = "jdbc:oracle:thin:@oracle-prod:1521:PRODDB";
Driver: oracle.jdbc.OracleDriver

// PostgreSQL JDBC
String url = "jdbc:postgresql://aurora-cluster.xxxx.rds.amazonaws.com:5432/proddb";
Driver: org.postgresql.Driver
```

**SQL dialect changes:**
```sql
-- Oracle-specific → PostgreSQL equivalent

-- String concatenation
Oracle:     'Hello' || ' ' || 'World'
PostgreSQL: 'Hello' || ' ' || 'World'  -- Same!

-- Sequence nextval
Oracle:     emp_seq.NEXTVAL
PostgreSQL: nextval('emp_seq')

-- Date functions
Oracle:     SYSDATE, ADD_MONTHS(SYSDATE, 1)
PostgreSQL: CURRENT_DATE, CURRENT_DATE + INTERVAL '1 month'

-- Outer join syntax
Oracle:     WHERE a.id = b.id(+)  -- Old syntax
PostgreSQL: LEFT JOIN b ON a.id = b.id

-- Dual table
Oracle:     SELECT 1 FROM DUAL
PostgreSQL: SELECT 1  -- No DUAL needed

-- NVL → COALESCE
Oracle:     NVL(column, 'default')
PostgreSQL: COALESCE(column, 'default')
```

**Phase 5: Testing & Cutover (1-2 weeks)**
5. **Validate data integrity:**

```sql
-- Compare row counts
-- Oracle
SELECT COUNT(*) FROM employees;  -- 1,000,000

-- Aurora PostgreSQL
SELECT COUNT(*) FROM employees;  -- 1,000,000 ✓

-- Compare aggregations
-- Oracle
SELECT SUM(salary), AVG(salary) FROM employees;

-- Aurora
SELECT SUM(salary), AVG(salary) FROM employees;
-- Must match exactly
```

6. **Performance testing:**
   - Run production workload against Aurora
   - Monitor query performance (should be faster!)
   - Adjust indexes if needed

7. **Cutover:**
   - Schedule downtime window (e.g., 2 AM Sunday)
   - Stop application
   - Final CDC sync (catch up last few transactions)
   - Point application to Aurora endpoint
   - Start application
   - Monitor for 24-48 hours

**Typical timeline:**
- **Small database (< 100GB):** 4-6 weeks
- **Medium database (100GB - 1TB):** 2-3 months
- **Large database (> 1TB):** 3-6 months

**Cost savings:**
- **Oracle Enterprise Edition:** $47,500/core/year license + 22% support = $58K/core/year
- **Aurora PostgreSQL:** $0 license, $600/month infrastructure = $7.2K/year
- **Savings for 8-core database: $464K - $7.2K = $456,800/year (98% reduction!)**

---

This completes the beginner interview questions. Should I continue with intermediate questions (5), scenario-based questions (10), and the remaining sections (Certification Tips, Production Best Practices, Hands-On Challenges, Resume Project, Solution Engineer Perspective, Cleanup)?


### Intermediate Questions (5)

**Q6: Explain Aurora Global Database and when to use it.**

**Answer:**

**Aurora Global Database** enables a single Aurora database to span multiple AWS regions with sub-second replication latency.

**Architecture:**
```
Primary Region (us-east-1)
└─ Aurora Cluster (Read-Write)
   ├─ Writer instance
   └─ Reader instances (1-15)
        ↓ (Physical replication, < 1 second lag)
Secondary Region (eu-west-1)
└─ Aurora Cluster (Read-Only)
   └─ Reader instances (up to 16)
        ↓ (Can promote to primary in <1 minute)
```

**Use cases:**

1. **Disaster Recovery (DR):**
   - **RTO:** < 1 minute (promote secondary to primary)
   - **RPO:** < 1 second (minimal data loss)
   - **Example:** Primary region (us-east-1) fails → Promote eu-west-1 to primary

2. **Global Applications:**
   - **Low-latency reads worldwide**
   - Users in Europe read from eu-west-1 (< 10ms latency)
   - Users in US read from us-east-1 (< 10ms latency)
   - All writes go to primary region

3. **Compliance:**
   - **Data residency requirements** (GDPR: data must stay in EU)
   - Store EU customer data in eu-west-1
   - Replicate to us-east-1 for analytics only

**Performance:**
- **Replication lag:** Typically < 1 second (sub-second in same geographic area)
- **Write latency:** No additional latency (writes only to primary)
- **Failover time:** < 1 minute to promote secondary to primary

**Cost:**
- **Additional costs:**
  - Secondary region instances (same as primary pricing)
  - Cross-region data transfer: $0.02/GB
  - I/O charges in secondary region

**Example monthly cost (db.r6g.xlarge, 1TB storage, 1TB writes/month):**
- Primary region (us-east-1): $600/month
- Secondary region (eu-west-1): $600/month (instances) + $20/month (data transfer) = $620/month
- **Total: $1,220/month**

**When to use:**
- ✅ Global user base (low-latency reads worldwide)
- ✅ Mission-critical apps (need <1min RTO for DR)
- ✅ Compliance (data residency requirements)
- ❌ **Don't use if:** Single-region app, cost-sensitive, can tolerate hours of downtime

---

**Q7: How does Redshift Spectrum differ from Redshift native tables?**

**Answer:**

| Feature | Redshift Tables | Redshift Spectrum |
|---------|----------------|-------------------|
| **Storage Location** | Redshift cluster (SSD) | S3 data lake |
| **Data Format** | Columnar (Redshift native) | Parquet, ORC, JSON, CSV, Avro |
| **Query Performance** | Fastest (data co-located with compute) | Slower (network transfer from S3) |
| **Storage Cost** | $0.024/GB/month (SSD) | $0.023/GB/month (S3 Standard) |
| **Compute Cost** | Included in cluster price | $5/TB scanned |
| **Use Case** | Hot data (queried frequently) | Cold data (queried occasionally) |
| **Scalability** | Limited by cluster size | Unlimited (exabyte-scale) |
| **Partitioning** | Distribution + Sort keys | S3 prefixes (Hive partitions) |
| **Schema** | Managed in Redshift | Managed in Glue Data Catalog |

**When to use each:**

**Redshift Tables (Native):**
```sql
-- Create Redshift table (data stored in cluster)
CREATE TABLE hot_transactions (
    transaction_id BIGINT,
    customer_id INT DISTKEY,
    date_key INT SORTKEY,
    amount DECIMAL(10,2)
);

-- Load data into Redshift
COPY hot_transactions
FROM 's3://bucket/recent-data/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftRole'
PARQUET;

-- Fast queries (< 1 second)
SELECT SUM(amount) FROM hot_transactions WHERE date_key = 20240622;
```

**Redshift Spectrum (S3):**
```sql
-- Create external schema (points to Glue Data Catalog)
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'datalake'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftSpectrumRole';

-- Query S3 data without loading (data stays in S3)
SELECT SUM(amount)
FROM spectrum_schema.cold_transactions
WHERE year = 2020 AND month = 6;

-- Slower (5-10 seconds) but no storage cost in Redshift
```

**Hybrid approach (Best Practice):**
```sql
-- Join hot data (Redshift) with cold data (S3 Spectrum)
SELECT 
    c.customer_name,
    SUM(hot.amount) AS recent_sales,
    SUM(cold.amount) AS historical_sales
FROM hot_transactions hot  -- Last 30 days in Redshift
JOIN spectrum_schema.cold_transactions cold  -- Historical data in S3
  ON hot.customer_id = cold.customer_id
JOIN dim_customers c
  ON hot.customer_id = c.customer_id
WHERE hot.date_key >= 20240525  -- Last 30 days
  AND cold.year BETWEEN 2020 AND 2023
GROUP BY c.customer_name;
```

**Cost optimization strategy:**
```
┌─────────────────────────────────────────────────────────┐
│          Data Lifecycle: Redshift → Spectrum            │
└─────────────────────────────────────────────────────────┘

Days 0-30:   Redshift native tables (fast, frequent queries)
Days 31-90:  Move to S3 (Spectrum) - weekly queries
Days 91+:    S3 Glacier Deep Archive - annual audits

Monthly cost for 10TB data:
├─ All in Redshift:   10TB × $24/TB = $240
├─ All in Spectrum:   10TB × $5/TB (query cost) = $50
└─ Hybrid (best):     1TB Redshift + 9TB Spectrum = $24 + $45 = $69
    └─ Savings: 71% vs all-Redshift
```

**Key takeaway:**
- **Redshift tables:** Hot data (< 30 days), queried daily, need sub-second performance
- **Redshift Spectrum:** Warm/cold data (> 30 days), queried weekly/monthly, cost-optimized

---

**Q8: What are DynamoDB Global Secondary Indexes (GSI) vs Local Secondary Indexes (LSI)?**

**Answer:**

**Local Secondary Index (LSI):**
- **Same partition key as base table**, different sort key
- **Created at table creation time** (cannot add later)
- **Shares throughput with base table** (same RCU/WCU)
- **Strongly consistent reads** supported
- **Max 5 LSIs per table**

**Global Secondary Index (GSI):**
- **Different partition key and/or sort key** from base table
- **Can be created/deleted anytime** (after table creation)
- **Separate throughput** (own RCU/WCU)
- **Eventually consistent reads only** (few seconds lag)
- **Max 20 GSIs per table**

**Example use case:**

**Base Table:**
```python
# Orders table
Partition Key: customer_id
Sort Key: order_date

# Sample data
customer_id | order_date  | order_id | product_id | amount
------------|-------------|----------|------------|-------
CUST001     | 2024-06-20  | ORD123   | PROD456    | 99.99
CUST001     | 2024-06-22  | ORD124   | PROD789    | 149.99
CUST002     | 2024-06-21  | ORD125   | PROD456    | 99.99
```

**Query patterns:**
1. ✅ Get all orders for a customer (use base table)
2. ❌ Get all orders for a product (no efficient way!)
3. ❌ Get order by order_id (scan entire table!)

**Solution: Add GSIs**

**GSI 1: Query by product_id**
```python
# GSI: product_index
Partition Key: product_id
Sort Key: order_date

# Now this query is efficient:
response = table.query(
    IndexName='product_index',
    KeyConditionExpression='product_id = :prod',
    ExpressionAttributeValues={':prod': 'PROD456'}
)
# Fast lookup: all orders for PROD456
```

**GSI 2: Query by order_id**
```python
# GSI: order_index
Partition Key: order_id
(No sort key needed - order_id is unique)

# Lookup order by ID:
response = table.query(
    IndexName='order_index',
    KeyConditionExpression='order_id = :oid',
    ExpressionAttributeValues={':oid': 'ORD123'}
)
# Fast lookup: specific order
```

**LSI example: Query by different sort key**
```python
# Base table: customer_id (PK) + order_date (SK)
# LSI: customer_id (PK) + amount (SK)

# Query: Get customer's orders sorted by amount (highest first)
response = table.query(
    IndexName='amount_index',  # LSI
    KeyConditionExpression='customer_id = :cust',
    ExpressionAttributeValues={':cust': 'CUST001'},
    ScanIndexForward=False  # Descending order
)
# Returns: [$149.99, $99.99] for CUST001
```

**Cost implications:**

**GSI cost:**
- **Storage:** Replicated data (doubles storage cost)
- **Throughput:** Separate RCU/WCU (additional cost)
- **Example:** 10GB table + 10GB GSI = **20GB storage cost**

**LSI cost:**
- **Storage:** Minimal additional storage (indexes only)
- **Throughput:** Shared with base table (no additional cost)
- **Example:** 10GB table + 1GB LSI = **11GB storage cost**

**Best practices:**
- **Use LSI** when: Same partition key, need strongly consistent reads
- **Use GSI** when: Need different partition key, can tolerate eventual consistency
- **Sparse indexes:** Only index items with a specific attribute (saves cost)

```python
# Sparse GSI example (only index completed orders)
table.put_item(
    Item={
        'customer_id': 'CUST001',
        'order_date': '2024-06-22',
        'order_id': 'ORD123',
        'status': 'completed',  # Only items with status are in GSI
        'completion_date': '2024-06-25'  # GSI partition key
    }
)

# GSI: completion_date (PK)
# Only completed orders appear in GSI (saves storage cost)
```

---

**Q9: How do you monitor and optimize Redshift query performance?**

**Answer:**

**Step 1: Enable Query Monitoring Rules (QMR)**

```sql
-- Create parameter group with query monitoring
-- AWS Console → Redshift → Parameter groups → Create parameter group

-- Add query monitoring rules (JSON):
[
  {
    "rule_name": "long_running_query",
    "predicate": [{
      "metric_name": "query_execution_time",
      "operator": ">",
      "value": 300000  -- 5 minutes in milliseconds
    }],
    "action": "log"  -- or "hop" to route to different queue
  },
  {
    "rule_name": "high_disk_usage",
    "predicate": [{
      "metric_name": "query_temp_blocks_to_disk",
      "operator": ">",
      "value": 100000
    }],
    "action": "abort"  -- Kill query if it spills too much to disk
  }
]
```

**Step 2: Query System Tables for Performance Analysis**

```sql
-- Find long-running queries
SELECT 
    query,
    TRIM(querytxt) AS sql,
    starttime,
    endtime,
    DATEDIFF(seconds, starttime, endtime) AS duration_sec,
    aborted
FROM STL_QUERY
WHERE starttime >= DATEADD(day, -7, CURRENT_DATE)
  AND DATEDIFF(seconds, starttime, endtime) > 60  -- Queries > 1 min
ORDER BY duration_sec DESC
LIMIT 10;
```

```sql
-- Identify queries with disk spill (bad performance)
SELECT 
    q.query,
    TRIM(q.querytxt) AS sql,
    s.rows,
    s.bytes,
    s.temp_blocks_to_disk
FROM STL_QUERY q
JOIN SVL_QUERY_SUMMARY s ON q.query = s.query
WHERE s.temp_blocks_to_disk > 0  -- Query spilled to disk
ORDER BY s.temp_blocks_to_disk DESC
LIMIT 10;

-- Solution: Increase memory (larger node type) or optimize query
```

```sql
-- Find tables with high skew (uneven data distribution)
SELECT 
    "schema",
    "table",
    size,
    tbl_rows,
    skew_rows,
    ROUND(skew_rows * 100.0 / tbl_rows, 2) AS skew_pct
FROM SVV_TABLE_INFO
WHERE skew_rows > tbl_rows * 0.1  -- More than 10% skew
ORDER BY skew_pct DESC;

-- Solution: Choose better DISTKEY (evenly distributed column)
```

**Step 3: Use EXPLAIN to Analyze Query Plans**

```sql
-- Get query execution plan
EXPLAIN
SELECT 
    c.customer_name,
    SUM(f.amount) AS total_sales
FROM fact_transactions f
JOIN dim_customers c ON f.customer_key = c.customer_key
WHERE f.date_key BETWEEN 20240601 AND 20240630
GROUP BY c.customer_name;
```

**Sample output:**
```
XN HashAggregate (cost=1000..1100 rows=1000)
  -> XN Hash Join DS_DIST_NONE (cost=500..800 rows=100000)
      Hash Cond: ("outer".customer_key = "inner".customer_key)
      -> XN Seq Scan on fact_transactions f (cost=0..400 rows=100000)
          Filter: ((date_key >= 20240601) AND (date_key <= 20240630))
      -> XN Hash (cost=100..100 rows=1000)
          -> XN Seq Scan on dim_customers c (cost=0..100 rows=1000)
```

**Interpret results:**
- **DS_DIST_NONE:** No data redistribution (good! DISTKEY is optimized)
- **DS_BCAST_INNER:** Broadcasting small table to all nodes (acceptable for dim tables)
- **DS_DIST_ALL_NONE:** Redistributing large table (BAD! Wrong DISTKEY)

**Step 4: Optimize with Distribution & Sort Keys**

```sql
-- Check current table design
SELECT 
    "schema",
    "table",
    diststyle,
    distkey,
    sortkey1
FROM SVV_TABLE_INFO
WHERE "table" = 'fact_transactions';

-- If DISTKEY is wrong, recreate table:
CREATE TABLE fact_transactions_v2 (
    transaction_id BIGINT,
    customer_key INT DISTKEY,  -- Change to frequently joined column
    date_key INT SORTKEY,      -- Change to frequently filtered column
    amount DECIMAL(10,2)
);

-- Copy data
INSERT INTO fact_transactions_v2
SELECT * FROM fact_transactions;

-- Swap tables
DROP TABLE fact_transactions;
ALTER TABLE fact_transactions_v2 RENAME TO fact_transactions;

-- Run ANALYZE to update statistics
ANALYZE fact_transactions;
```

**Step 5: Enable Query Result Caching**

```sql
-- Redshift automatically caches query results for repeated queries
-- Enable at parameter group level:
enable_result_cache_for_session = true

-- Check cache hit rate
SELECT 
    query,
    is_cached,
    elapsed_time
FROM SVL_QLOG
WHERE userid > 1
ORDER BY starttime DESC
LIMIT 100;

-- Queries with is_cached = true run in < 1ms (instant)
```

**Step 6: Use Materialized Views**

```sql
-- Create materialized view for frequently run aggregation
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT 
    date_key,
    SUM(amount) AS total_sales,
    COUNT(*) AS transaction_count
FROM fact_transactions
GROUP BY date_key;

-- Refresh materialized view (after new data loaded)
REFRESH MATERIALIZED VIEW mv_daily_sales;

-- Query materialized view (10-100x faster than base query)
SELECT * FROM mv_daily_sales WHERE date_key = 20240622;
```

**Step 7: Monitor with CloudWatch**

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

# Get CPU utilization
response = cloudwatch.get_metric_statistics(
    Namespace='AWS/Redshift',
    MetricName='CPUUtilization',
    Dimensions=[{'Name': 'ClusterIdentifier', 'Value': 'analytics-cluster'}],
    StartTime=datetime.utcnow() - timedelta(hours=1),
    EndTime=datetime.utcnow(),
    Period=300,
    Statistics=['Average', 'Maximum']
)

# Alert if CPU > 80% for 10 minutes (need larger cluster)
```

**Performance optimization checklist:**

✅ **Analyze table statistics:** `ANALYZE table_name;` (weekly)  
✅ **Vacuum tables:** `VACUUM FULL table_name;` (after large deletes)  
✅ **Choose right DISTKEY:** Column used in JOINs  
✅ **Choose right SORTKEY:** Column used in WHERE clauses  
✅ **Use columnar compression:** `ANALYZE COMPRESSION table_name;`  
✅ **Monitor disk spill:** Upgrade if temp_blocks_to_disk > 0  
✅ **Use WLM (Workload Management):** Separate queues for ETL vs BI queries  
✅ **Enable short query acceleration (SQA):** Fast-lane for quick queries  

---

**Q10: Explain the AWS Database Migration Service (DMS) replication modes.**

**Answer:**

AWS DMS supports three replication modes:

**1. Full Load (One-Time Migration)**

**Use case:** Initial data migration with downtime

```
┌─────────────────────────────────────────┐
│       Full Load Migration               │
└─────────────────────────────────────────┘

Source DB (Oracle)                Target DB (Aurora PostgreSQL)
├─ 500GB data                    ├─ 0GB (empty)
│                                 │
├─ Stop application ────┐         │
│                       │         │
│ DMS Replication       │         │
│ ├─ Table 1 ──────────────────→ Table 1 (copied)
│ ├─ Table 2 ──────────────────→ Table 2 (copied)
│ └─ Table N ──────────────────→ Table N (copied)
│                       │         │
│                       └────→ Migration complete (500GB)
│                                 │
└─ Start application ──────────→ Point to Aurora
```

**Characteristics:**
- **Downtime:** Hours to days (depends on data size)
- **Data consistency:** Guaranteed (no writes during migration)
- **Cost:** Lowest (replication instance runs for shortest time)
- **Suitable for:** Dev/test environments, applications that can tolerate downtime

**Example:**
```bash
# Create DMS task (full load only)
aws dms create-replication-task \
  --replication-task-identifier full-load-task \
  --source-endpoint-arn arn:aws:dms:...:endpoint/oracle-source \
  --target-endpoint-arn arn:aws:dms:...:endpoint/aurora-target \
  --replication-instance-arn arn:aws:dms:...:rep-instance/dms-instance \
  --migration-type full-load \  # One-time copy
  --table-mappings file://tables.json

# Migration time estimate: 500GB / 100MB/s = 1.4 hours
```

---

**2. Full Load + CDC (Minimal Downtime)**

**Use case:** Production migration with < 5 minutes downtime

```
┌─────────────────────────────────────────┐
│    Full Load + CDC Migration            │
└─────────────────────────────────────────┘

Phase 1: Full Load (Application still running)
Source DB (Oracle)                Target DB (Aurora)
├─ 500GB data                    ├─ 0GB
│  (app writes continue)          │
│                                 │
│ DMS copies historical data      │
│ ├─ Table 1 ──────────────────→ Table 1 (lag: 2 hours)
│ ├─ Table 2 ──────────────────→ Table 2 (lag: 1 hour)
│ └─ Table N ──────────────────→ Table N (lag: 0 minutes) ✓
│                                 │

Phase 2: CDC (Catch up)
Source DB                         Target DB
├─ New transactions              ├─ Historical data loaded
│  (ongoing writes)               │  (catching up to real-time)
│                                 │
│ DMS replicates changes          │
│ ├─ INSERT ─────────────────→  INSERT (lag: 10 seconds)
│ ├─ UPDATE ─────────────────→  UPDATE (lag: 5 seconds)
│ └─ DELETE ─────────────────→  DELETE (lag: < 1 second) ✓
│                                 │
│ (Source and target now in sync!)

Phase 3: Cutover (5 minutes downtime)
Source DB                         Target DB
├─ Stop application ────┐         │
│                       │         │
│ DMS final sync        │         │
│ (last few txns) ──────────────→ (fully synced)
│                       │         │
└─ Start app ──────────────────→ Point to Aurora
```

**Characteristics:**
- **Downtime:** 5-30 minutes (only during final cutover)
- **Data consistency:** Eventual (CDC lag typically < 1 second)
- **Cost:** Medium (replication instance runs for days/weeks)
- **Suitable for:** Production databases, critical applications

**Example:**
```bash
# Create DMS task (full load + CDC)
aws dms create-replication-task \
  --replication-task-identifier full-load-cdc-task \
  --migration-type full-load-and-cdc \  # Full load + continuous replication
  --cdc-start-position "2024-06-22T10:00:00" \  # Start CDC from this timestamp
  --table-mappings file://tables.json

# Timeline:
# Day 0: Start full load (app still running on Oracle)
# Day 2: Full load complete, CDC catches up
# Day 3: Replication lag < 1 second (ready for cutover)
# Day 3 @ 2 AM: Stop app, final sync (5 min), start app on Aurora
```

---

**3. CDC Only (Zero Downtime, Ongoing Replication)**

**Use case:** Continuous replication, multi-region sync, analytics replication

```
┌─────────────────────────────────────────┐
│         CDC Only Replication            │
└─────────────────────────────────────────┘

Source DB (Oracle)                Target DB (Aurora)
├─ 500GB data                    ├─ 500GB data (already loaded)
│  (ongoing writes)               │  (via previous full load)
│                                 │
│ DMS replicates changes ONLY     │
│ ├─ INSERT ─────────────────→  INSERT (lag: < 1 sec)
│ ├─ UPDATE ─────────────────→  UPDATE (lag: < 1 sec)
│ └─ DELETE ─────────────────→  DELETE (lag: < 1 sec)
│                                 │
│ (Runs continuously forever)    │
```

**Characteristics:**
- **Downtime:** Zero (both databases active)
- **Data consistency:** Eventual (< 1 second lag)
- **Cost:** Highest (replication instance runs forever)
- **Suitable for:** Cross-region replication, analytics replicas, hybrid cloud

**Example:**
```bash
# Scenario: Replicate production (Oracle) to analytics (Aurora)
# Step 1: Manual initial load (or previous full load)
# Step 2: Start CDC to keep in sync

aws dms create-replication-task \
  --replication-task-identifier cdc-only-task \
  --migration-type cdc \  # CDC only (no full load)
  --cdc-start-position "2024-06-22T10:00:00" \
  --table-mappings file://tables.json

# Use case: Analysts query Aurora (read replica) while app writes to Oracle
```

---

**Comparison Table:**

| Feature | Full Load | Full Load + CDC | CDC Only |
|---------|-----------|-----------------|----------|
| **Downtime** | Hours to days | 5-30 minutes | Zero |
| **Initial data sync** | Yes | Yes | No (manual load required) |
| **Ongoing replication** | No | Yes (until cutover) | Yes (continuous) |
| **Replication instance cost** | Lowest | Medium | Highest |
| **Use case** | Dev/test migration | Production migration | Analytics replica |
| **Data consistency during migration** | Perfect | Eventual (< 1 sec lag) | Eventual |

---

**Advanced DMS Features:**

**1. Transformation Rules:**
```json
{
  "rules": [
    {
      "rule-type": "transformation",
      "rule-id": "1",
      "rule-name": "rename-table",
      "rule-action": "rename",
      "rule-target": "table",
      "object-locator": {
        "schema-name": "PROD",
        "table-name": "EMPLOYEES"
      },
      "value": "employees"  // Lowercase for PostgreSQL
    },
    {
      "rule-type": "transformation",
      "rule-id": "2",
      "rule-name": "remove-column",
      "rule-action": "remove-column",
      "rule-target": "column",
      "object-locator": {
        "schema-name": "PROD",
        "table-name": "EMPLOYEES",
        "column-name": "SSN"  // Don't migrate sensitive data
      }
    }
  ]
}
```

**2. Table Mappings (Filter Data):**
```json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-recent-data",
      "object-locator": {
        "schema-name": "PROD",
        "table-name": "TRANSACTIONS"
      },
      "rule-action": "include",
      "filters": [
        {
          "filter-type": "source",
          "column-name": "TRANSACTION_DATE",
          "filter-conditions": [
            {
              "filter-operator": "gte",
              "value": "2024-01-01"  // Only migrate 2024 data
            }
          ]
        }
      ]
    }
  ]
}
```

**3. Monitoring Replication Lag:**
```python
import boto3

dms = boto3.client('dms')

# Get replication task metrics
response = dms.describe_replication_tasks(
    Filters=[{'Name': 'replication-task-arn', 'Values': ['arn:aws:dms:...']}]
)

task = response['ReplicationTasks'][0]
print(f"Status: {task['Status']}")
print(f"Replication lag: {task.get('ReplicationTaskStats', {}).get('ElapsedTimeMillis', 0)} ms")

# Alert if lag > 10 seconds (target falling behind)
if task.get('ReplicationTaskStats', {}).get('ElapsedTimeMillis', 0) > 10000:
    print("⚠️ Replication lag too high!")
```

---

### Scenario-Based Questions (10)

#### **Q11: Oracle to Aurora PostgreSQL - Zero Downtime Migration**

**Scenario**: Your company runs a 5TB Oracle RAC database (Oracle 19c) supporting a mission-critical e-commerce platform with 50,000 transactions per hour. You need to migrate to Aurora PostgreSQL to reduce database costs by 90% while maintaining zero downtime (max 5 minutes cutover window). The Oracle database has:
- **Data volume**: 5TB (customers, orders, products, inventory)
- **Throughput**: 50,000 TPS (peak), 15,000 TPS (average)
- **Availability SLA**: 99.95% (max 22 minutes downtime/month)
- **Current cost**: $800,000/year (Oracle licenses + infrastructure)
- **Target cost**: $80,000/year (Aurora Reserved Instances)

**Solution:**

**Phase 1: Assessment using AWS Schema Conversion Tool (SCT)**

```bash
# SCT Assessment Report
Total Objects: 930
├─ Automatically Converted: 790 (85%)
├─ Requires Manual Action: 93 (10%)
└─ Incompatible: 47 (5%)

# Key incompatibilities:
# 1. Oracle DBLINK → PostgreSQL foreign data wrapper (manual)
# 2. Oracle Advanced Queuing → Amazon SQS (architecture change)
# 3. Oracle Flashback → Aurora Backtrack (equivalent feature)
```

**Phase 2: Create Aurora PostgreSQL Cluster**

```bash
aws rds create-db-cluster \
  --db-cluster-identifier prod-aurora-cluster \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username postgres \
  --master-user-password 'SecurePassword123!' \
  --database-name ecommerce \
  --storage-encrypted \
  --kms-key-id alias/aurora-production \
  --enable-iam-database-authentication \
  --backup-retention-period 35 \
  --deletion-protection

# Create writer + 2 readers
aws rds create-db-instance \
  --db-instance-identifier prod-aurora-writer \
  --db-instance-class db.r6g.4xlarge \
  --engine aurora-postgresql \
  --db-cluster-identifier prod-aurora-cluster

# Cost: $3,032/month vs Oracle $800K/year = 95% savings
```

**Phase 3: DMS Full Load + CDC (Zero Downtime)**

```bash
# Create DMS replication instance
aws dms create-replication-instance \
  --replication-instance-identifier oracle-to-aurora-dms \
  --replication-instance-class dms.r5.4xlarge \
  --allocated-storage 1000 \
  --multi-az

# Create migration task (full load + CDC)
aws dms create-replication-task \
  --replication-task-identifier oracle-aurora-migration \
  --source-endpoint-arn arn:aws:dms:...:endpoint/oracle-source \
  --target-endpoint-arn arn:aws:dms:...:endpoint/aurora-target \
  --replication-instance-arn arn:aws:dms:...:rep-instance/oracle-to-aurora-dms \
  --migration-type full-load-and-cdc \
  --table-mappings file://table-mappings.json

# Timeline:
# Day 1-7: Full load (5TB)
# Day 8+: CDC (replication lag < 1 second)
```

**Phase 4: Cutover (5 Minutes Downtime)**

```python
# Automated cutover script
def execute_cutover():
    # T-5:00 - Set Oracle read-only
    oracle_conn.execute("ALTER SYSTEM SET READ_ONLY = TRUE")
    
    # T-4:00 - Wait for DMS lag to reach 0
    wait_for_replication_lag(threshold_ms=1000)
    
    # T-3:00 - Stop DMS task
    dms.stop_replication_task()
    
    # T-2:00 - Final validation (row counts match)
    validate_data_integrity()
    
    # T-1:00 - Update Route53 DNS to Aurora
    route53.update_dns('database.company.com', aurora_endpoint)
    
    # T-0:00 - Resume writes to Aurora
    print("CUTOVER COMPLETE!")
```

**Results:**

| Metric | Oracle RAC | Aurora PostgreSQL | Improvement |
|--------|------------|-------------------|-------------|
| **Cost** | $800,000/year | $36,384/year | **95% reduction** |
| **Availability** | 99.5% | 99.95% | **+0.45%** |
| **Failover Time** | 15-30 min | 30-120 sec | **93% faster** |
| **Migration Downtime** | N/A | 5 minutes | **RTO: 5 min, RPO: 0** |

---

#### **Q12: Redshift Performance Crisis - Query Optimization**

**Scenario**: Your Redshift data warehouse query runtime increased from 30 minutes to 3+ hours. A critical daily ETL job is now missing SLA deadlines. Symptoms: high disk spill (120GB), uneven node utilization (90%/40%/35%/30%), excessive network transfer (200GB).

**Current Problem:**

```
┌────────────────────────────────────────────────────────────┐
│ Node 1 (90% full)    Node 2 (40%)    Node 3 (35%)   Node 4│
│ ├─ fact_sales (500GB)                                     │
│ │  DISTKEY: NONE ❌                                        │
│ │  SORTKEY: NONE ❌                                        │
│ │  Problem: Uneven distribution, no sort keys             │
└────────────────────────────────────────────────────────────┘
```

**Solution:**

**Step 1: Analyze Current Distribution**

```sql
-- Check data distribution
SELECT slice, COUNT(*) AS num_rows, SUM(size) AS size_mb
FROM stv_blocklist
WHERE name = 'fact_sales'
GROUP BY slice;

-- Result: 90% data on node 1 (uneven!) ❌

-- Check disk spill
SELECT query, SUM(bytes)/1024/1024/1024 AS spill_gb
FROM svl_query_summary
WHERE is_diskbased = 't'
GROUP BY query;

-- Result: 120GB disk spill ❌
```

**Step 2: Redesign Schema with Proper Keys**

```sql
-- BEFORE (broken)
CREATE TABLE fact_sales (
    sale_id BIGINT,
    customer_id BIGINT,
    sale_date DATE,
    sale_amount DECIMAL(10,2)
);  -- No DISTKEY, no SORTKEY

-- AFTER (optimized)
CREATE TABLE fact_sales_new (
    sale_id BIGINT,
    customer_id BIGINT DISTKEY,      -- JOIN key
    sale_date DATE SORTKEY,           -- WHERE clause filter
    sale_amount DECIMAL(10,2)
) DISTSTYLE KEY;

-- Small dimensions: replicate to all nodes
CREATE TABLE dim_customers_new (
    customer_id BIGINT PRIMARY KEY,
    customer_name VARCHAR(100)
) DISTSTYLE ALL;  -- < 100MB, replicate
```

**Step 3: Create Materialized View**

```sql
-- Pre-aggregate common queries
CREATE MATERIALIZED VIEW mv_monthly_sales AS
SELECT 
    DATE_TRUNC('month', s.sale_date) AS sale_month,
    c.customer_segment,
    SUM(s.sale_amount) AS total_revenue
FROM fact_sales_new s
JOIN dim_customers_new c ON s.customer_id = c.customer_id
WHERE s.sale_date >= DATEADD(month, -24, CURRENT_DATE)
GROUP BY 1, 2;

-- Query now uses MV (instant results)
SELECT * FROM mv_monthly_sales WHERE sale_month >= '2024-01-01';
-- Runtime: 0.8 seconds (was 3 hours) = 15,000x faster! ✓
```

**Results:**

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Query Runtime** | 3 hr 20 min | 28 seconds | **428x faster** |
| **Disk Spill** | 120GB | 0GB | **100% reduction** |
| **Network Transfer** | 200GB | 2GB | **99% reduction** |
| **Node Utilization** | 90%/40%/35%/30% | 75%/75%/75%/75% | **Even** |

**Techniques Applied:**
1. **DISTKEY:** customer_id (most frequent JOIN column)
2. **SORTKEY:** sale_date (most common WHERE clause)
3. **DISTSTYLE ALL:** Small dimensions replicated to all nodes
4. **Materialized View:** Pre-aggregated common queries

---

#### **Q13: Aurora Global Database - Disaster Recovery Failover**

**Scenario**: You need disaster recovery for Aurora PostgreSQL with RTO < 1 minute and RPO < 1 minute. Primary region (us-east-1), secondary region (eu-west-1). Automated failover required for regional outages.

**Architecture:**

```
Primary Region: us-east-1
┌──────────────────────────────────────────────────┐
│  Aurora Writer (db.r6g.2xlarge)                  │
│  ├─ Handles all writes (10,000 TPS)              │
│  ├─ 2 Read Replicas (Multi-AZ)                   │
│  └─ Replicates to eu-west-1 (lag: 100-300ms)     │
└──────────────────────────────────────────────────┘
                    │
                    │ Aurora Global Database
                    ↓
Secondary Region: eu-west-1 (DR)
┌──────────────────────────────────────────────────┐
│  Aurora Read Replica (db.r6g.xlarge)             │
│  ├─ Read-only until promoted                     │
│  ├─ Replication lag: 100-300ms                   │
│  └─ Promotes to primary on failover              │
└──────────────────────────────────────────────────┘
```

**Implementation:**

**Step 1: Create Aurora Global Database**

```bash
# Create primary cluster (us-east-1)
aws rds create-global-cluster \
  --global-cluster-identifier prod-global-cluster \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:...:cluster:prod-global-primary

# Add secondary region (eu-west-1)
aws rds create-db-cluster \
  --db-cluster-identifier prod-global-secondary \
  --engine aurora-postgresql \
  --global-cluster-identifier prod-global-cluster \
  --region eu-west-1

# Create DR read replica
aws rds create-db-instance \
  --db-instance-identifier prod-global-dr-reader \
  --db-instance-class db.r6g.xlarge \
  --engine aurora-postgresql \
  --db-cluster-identifier prod-global-secondary \
  --region eu-west-1
```

**Step 2: Setup Route 53 Failover**

```bash
# Primary record (us-east-1)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "database.myapp.com",
        "Type": "CNAME",
        "Failover": "PRIMARY",
        "HealthCheckId": "health-check-id",
        "TTL": 60,
        "ResourceRecords": [{"Value": "prod-global-primary.us-east-1.rds.amazonaws.com"}]
      }
    }]
  }'

# Secondary record (eu-west-1)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "database.myapp.com",
        "Type": "CNAME",
        "Failover": "SECONDARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "prod-global-secondary.eu-west-1.rds.amazonaws.com"}]
      }
    }]
  }'
```

**Step 3: Automated Failover Lambda**

```python
# lambda_failover.py
import boto3

def lambda_handler(event, context):
    rds = boto3.client('rds', region_name='eu-west-1')
    
    # Detach secondary from global cluster
    rds.remove_from_global_cluster(
        GlobalClusterIdentifier='prod-global-cluster',
        DbClusterIdentifier='arn:aws:rds:eu-west-1:...:cluster:prod-global-secondary'
    )
    
    # Secondary is now standalone (accepts writes)
    return {'statusCode': 200, 'body': 'Failover complete'}
```

**Results:**

| Metric | Target | Actual |
|--------|--------|--------|
| **RTO** | < 1 minute | 90 seconds |
| **RPO** | < 1 minute | 300ms |
| **Replication Lag** | < 1 second | 100-300ms |
| **Availability** | 99.99% | 99.99% |

**Cost:** $1,605/month (primary + secondary + Route 53)

---

#### **Q14: DynamoDB Capacity Planning - Black Friday Traffic Spike**

**Scenario**: Your e-commerce platform uses DynamoDB for shopping cart sessions. Normal traffic: 5,000 writes/sec, 15,000 reads/sec. Black Friday spike: 50x increase (250,000 writes/sec, 750,000 reads/sec) for 48 hours. Plan capacity with zero throttling.

**Options:**

**Option 1: Provisioned Capacity**
- WCU: 250,000 x $0.00065/hour x 48h = $7,800
- RCU: 750,000 x $0.00013/hour x 48h = $4,680
- **Total: $12,480**
- **Problems:** Manual scaling, risk of throttling

**Option 2: On-Demand Capacity**
- Writes: 43.2B x $1.25/million = $54,000
- Reads: 129.6B x $0.25/million = $32,400
- **Total: $86,400**
- **Benefit:** Auto-scaling, no throttling

**Option 3: Hybrid (DAX + On-Demand) - RECOMMENDED**

```
Architecture:
Application → DAX (90% cache hit) → DynamoDB
                ↓                      ↓
          675K reads/sec         75K reads/sec
          (cache hits)           (cache misses)
                              250K writes/sec
```

**Implementation:**

```bash
# Create DAX cluster (3 nodes)
aws dax create-cluster \
  --cluster-name shopping-cart-dax \
  --node-type dax.r5.large \
  --replication-factor 3

# Switch table to on-demand
aws dynamodb update-table \
  --table-name shopping-cart-sessions \
  --billing-mode PAY_PER_REQUEST
```

**Cost Calculation:**

```
DynamoDB On-Demand:
├─ Writes: 43.2B x $1.25/million = $54,000
├─ Reads (10% only): 12.96B x $0.25/million = $3,240
└─ Subtotal: $57,240

DAX (3 nodes x $0.228/hour x 48h):
└─ Subtotal: $32.83

Total: $57,273 (vs $86,400 on-demand only)
Savings: $29,127 (34% reduction) ✓
```

**Results:**

| Approach | Cost (48h) | Savings | Latency |
|----------|------------|---------|---------|
| Provisioned | $12,480 | - | 15ms |
| On-Demand | $86,400 | - | 15ms |
| **Hybrid (DAX)** | **$57,273** | **34%** | **2ms** |

---

#### **Q15: Multi-Tenant SaaS Database Security**

**Scenario**: Build secure multi-tenant SaaS platform with 500 enterprise customers. Requirements: data isolation, per-tenant encryption, audit logging (SOC 2, GDPR), row-level security. Budget: < $10,000/month.

**Architecture Options:**

**Option 1: Database per Tenant**
- Cost: $300/cluster x 500 = $150,000/month ❌ (too expensive)

**Option 2: Shared DB + Schema Isolation (RECOMMENDED)**
- Cost: $1,265/month ✅

**Implementation:**

**Step 1: Create Aurora Cluster with Encryption**

```bash
# Create KMS key
aws kms create-key --description "Aurora multi-tenant encryption"
aws kms create-alias --alias-name alias/aurora-multitenant --target-key-id <key-id>

# Create Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier saas-multitenant-cluster \
  --engine aurora-postgresql \
  --storage-encrypted \
  --kms-key-id alias/aurora-multitenant \
  --enable-iam-database-authentication \
  --enable-cloudwatch-logs-exports '["postgresql"]'

# Create instances (1 writer + 2 readers)
# Cost: $1,265/month
```

**Step 2: Create Schema-Based Tenant Isolation**

```sql
-- Create schema per tenant
CREATE SCHEMA tenant_acme_corp;
CREATE SCHEMA tenant_globex_inc;
-- ... repeat for all 500 tenants

-- Create dedicated user per tenant
CREATE USER tenant_acme_corp_user WITH PASSWORD 'SecurePass123!';

-- Grant access ONLY to their schema
GRANT USAGE ON SCHEMA tenant_acme_corp TO tenant_acme_corp_user;
GRANT ALL ON ALL TABLES IN SCHEMA tenant_acme_corp TO tenant_acme_corp_user;

-- Revoke access to other schemas
REVOKE ALL ON SCHEMA tenant_globex_inc FROM tenant_acme_corp_user;
REVOKE ALL ON SCHEMA public FROM tenant_acme_corp_user;

-- Verify isolation
SET ROLE tenant_acme_corp_user;
SELECT * FROM tenant_globex_inc.users;
-- ERROR: permission denied ✓
```

**Step 3: Enable Row-Level Security (Defense in Depth)**

```sql
-- Create table with RLS
CREATE TABLE tenant_acme_corp.users (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(100) NOT NULL DEFAULT 'acme-corp',
    email VARCHAR(255),
    CONSTRAINT chk_tenant_id CHECK (tenant_id = 'acme-corp')
);

-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Create RLS policy
CREATE POLICY tenant_isolation_policy ON users
    USING (tenant_id = current_setting('app.tenant_id', TRUE));

-- Application sets tenant_id before queries
SET app.tenant_id = 'acme-corp';

-- All queries automatically filtered by RLS
SELECT * FROM users;  -- Returns only acme-corp data ✓
```

**Step 4: Enable Audit Logging (GDPR Compliance)**

```sql
-- Install pgaudit extension
CREATE EXTENSION pgaudit;

-- Configure audit logging
ALTER SYSTEM SET pgaudit.log = 'READ, WRITE, DDL';
ALTER SYSTEM SET log_connections = ON;
ALTER SYSTEM SET log_statement = 'all';
SELECT pg_reload_conf();

-- Audit logs sent to CloudWatch Logs
-- Example: "AUDIT: SELECT * FROM users WHERE email = 'john@acme.com'"
```

**Step 5: IAM Database Authentication (No Passwords)**

```python
# Connect using IAM token (no static passwords)
import boto3
import psycopg2

def get_iam_auth_token(rds_host, port, db_user, region):
    rds = boto3.client('rds', region_name=region)
    return rds.generate_db_auth_token(
        DBHostname=rds_host, Port=port, DBUsername=db_user, Region=region
    )

# Get 15-minute token
auth_token = get_iam_auth_token('cluster.rds.amazonaws.com', 5432, 'tenant_acme_corp_user', 'us-east-1')

# Connect
conn = psycopg2.connect(
    host='cluster.rds.amazonaws.com',
    user='tenant_acme_corp_user',
    password=auth_token,  # Temporary token!
    sslmode='require'
)

# Set tenant context
conn.cursor().execute("SET app.tenant_id = 'acme-corp'")
```

**Security Layers:**

| Layer | Implementation | Status |
|-------|----------------|--------|
| 1. Network | Private subnet, no public IP | ✅ |
| 2. Encryption at Rest | KMS | ✅ |
| 3. Encryption in Transit | SSL/TLS | ✅ |
| 4. Schema Isolation | Dedicated schemas | ✅ |
| 5. Row-Level Security | PostgreSQL RLS | ✅ |
| 6. IAM Authentication | Token-based, no passwords | ✅ |
| 7. Audit Logging | pgaudit + CloudTrail | ✅ |

**Results:**

| Metric | Value |
|--------|-------|
| **Cost** | $1,368/month (vs $150K for 500 clusters) |
| **Savings** | 99% cost reduction |
| **Tenants Supported** | 500 schemas |
| **Compliance** | GDPR, SOC 2, HIPAA ✅ |
| **Security Layers** | 7 layers of defense |

---

#### **Q16: Database Performance Monitoring - Identifying and Resolving Slow Queries**

**Scenario**: Your Aurora PostgreSQL database is experiencing performance degradation. The application team reports that dashboard queries that normally complete in 2 seconds are now taking 15+ seconds. You need to identify the root cause and implement a fix within the next hour.

**Investigation Steps**:

```sql
-- Step 1: Check current running queries
SELECT 
    pid,
    usename,
    application_name,
    state,
    query_start,
    NOW() - query_start AS duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state != 'idle'
AND NOW() - query_start > INTERVAL '5 seconds'
ORDER BY duration DESC;

-- Result: Found long-running query (18 seconds)
/*
pid   | usename | duration  | wait_event_type | query
------|---------|-----------|-----------------|-------
12345 | app_usr | 00:00:18  | IO              | SELECT * FROM large_table WHERE...
*/

-- Step 2: Enable Performance Insights (if not already enabled)
-- AWS Console → RDS → Your Cluster → Modify → Enable Performance Insights

-- Step 3: Check for missing indexes
SELECT 
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE schemaname = 'public'
AND tablename = 'large_table'
ORDER BY n_distinct DESC;

-- Step 4: Identify table bloat
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - 
                   pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- Step 5: Check for bloated indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Step 6: Analyze query execution plan
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM large_table 
WHERE created_date >= '2024-01-01' 
AND status = 'active';

-- Result shows Sequential Scan instead of Index Scan
/*
Seq Scan on large_table  (cost=0.00..50000.00 rows=10000 width=200) (actual time=0.123..15234.567 rows=9876 loops=1)
  Filter: ((created_date >= '2024-01-01') AND (status = 'active'))
  Rows Removed by Filter: 990124
Buffers: shared hit=12345 read=23456
Planning Time: 0.234 ms
Execution Time: 15234.890 ms
*/
```

**The Fix**:

```sql
-- Solution 1: Create composite index
CREATE INDEX CONCURRENTLY idx_large_table_date_status 
ON large_table(created_date, status);

-- Solution 2: Analyze table (update statistics)
ANALYZE large_table;

-- Solution 3: Run VACUUM to reclaim space
VACUUM ANALYZE large_table;

-- Verify the fix
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM large_table 
WHERE created_date >= '2024-01-01' 
AND status = 'active';

-- Result after index creation:
/*
Index Scan using idx_large_table_date_status on large_table  
  (cost=0.43..234.56 rows=10000 width=200) (actual time=0.045..1.234 rows=9876 loops=1)
  Index Cond: ((created_date >= '2024-01-01') AND (status = 'active'))
Buffers: shared hit=123 read=45
Planning Time: 0.123 ms
Execution Time: 1.345 ms
*/
```

**Performance Comparison**:

| Metric | Before (Sequential Scan) | After (Index Scan) | Improvement |
|--------|-------------------------|-------------------|-------------|
| Execution Time | 15,234 ms (15.2 sec) | 1.3 ms | **11,718x faster** |
| Buffers Read | 35,801 | 168 | **99.5% reduction** |
| Rows Filtered Out | 990,124 | 0 | **Perfect selectivity** |

**CloudWatch Monitoring Setup**:

```bash
# Create alarm for slow query detection
aws cloudwatch put-metric-alarm \
  --alarm-name rds-slow-queries \
  --alarm-description "Alert when queries take >5 seconds" \
  --metric-name DatabaseConnections \
  --namespace AWS/RDS \
  --statistic Average \
  --period 60 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

**Key Lessons**:
1. **Performance Insights is essential**: Shows wait events and top SQL statements
2. **EXPLAIN ANALYZE reveals truth**: Always check execution plans before creating indexes
3. **CREATE INDEX CONCURRENTLY**: Never block production writes during index creation
4. **Regular VACUUM**: PostgreSQL needs maintenance to reclaim dead tuple space
5. **Composite indexes matter**: Index column order affects query performance

---

#### **Q17: Choosing the Right Database Service - Decision Framework**

**Scenario**: Your company is launching 3 new products. Each has different data requirements. You need to recommend the appropriate AWS database service for each workload.

**Product 1: E-Commerce Platform**
- **Requirements**: 
  - 10,000 transactions/second (peak)
  - Complex joins across customers, orders, products
  - ACID compliance
  - Point-in-time recovery
  - Multi-AZ for 99.99% availability
  - Read replicas for reporting

**Product 2: IoT Sensor Platform**
- **Requirements**:
  - 50,000 writes/second
  - Time-series data (device_id, timestamp, metrics)
  - Simple key-value lookups
  - Flexible schema (sensor types vary)
  - Global distribution (US, EU, APAC)
  - Cost-effective at scale

**Product 3: Data Warehouse / Business Intelligence**
- **Requirements**:
  - Complex analytical queries (JOINs across 10+ tables)
  - Petabyte-scale data
  - Star schema with fact/dimension tables
  - ETL from S3 data lake
  - Sub-second query performance on aggregations

**Decision Matrix**:

| Product | Recommended Service | Reasoning |
|---------|-------------------|-----------|
| **E-Commerce** | **Aurora PostgreSQL** | - ACID transactions (orders must be consistent)<br>- Complex JOINs (customers ⋈ orders ⋈ products)<br>- 10K TPS fits Aurora performance profile<br>- Read replicas offload reporting queries<br>- Point-in-time recovery for data protection |
| **IoT Sensors** | **DynamoDB** | - 50K writes/sec requires horizontal scalability<br>- Simple key-value access pattern (device_id + timestamp)<br>- Global Tables for multi-region active-active<br>- On-demand pricing scales with usage<br>- No complex joins needed |
| **Data Warehouse** | **Amazon Redshift** | - Columnar storage for analytical queries<br>- MPP architecture handles petabyte-scale<br>- Star schema optimization (DISTKEY, SORTKEY)<br>- COPY from S3 for efficient bulk loads<br>- Materialized views for sub-second aggregations |

**Architecture for Each Product**:

**Product 1: E-Commerce (Aurora)**
```
┌─────────────────────────────────────────────────────────────┐
│ Application Tier (ECS/Lambda)                               │
│         │                                                    │
│         ├─ Writes ──────────► Aurora Writer Instance        │
│         │                      (db.r6g.4xlarge)             │
│         │                                                    │
│         └─ Reads ──────────► Aurora Reader Instances (×2)   │
│                               (db.r6g.2xlarge)              │
│                                                              │
│ Storage: Aurora Shared Storage (6 copies, 3 AZs)            │
│ Backup: Automated backups (7 days), manual snapshots        │
│ Cost: ~$3,200/month (writer + 2 readers + storage)          │
└─────────────────────────────────────────────────────────────┘
```

**Product 2: IoT (DynamoDB)**
```
┌─────────────────────────────────────────────────────────────┐
│ IoT Devices (50K/sec writes)                                │
│         ↓                                                    │
│ DynamoDB Global Table                                        │
│ ┌─────────────────┬─────────────────┬─────────────────┐    │
│ │   US-EAST-1     │    EU-WEST-1    │   AP-SOUTH-1    │    │
│ │   (Primary)     │    (Replica)    │    (Replica)    │    │
│ │  On-Demand      │   On-Demand     │   On-Demand     │    │
│ └─────────────────┴─────────────────┴─────────────────┘    │
│         ↓                                                    │
│ DynamoDB Streams → Lambda → S3 (long-term storage)          │
│                                                              │
│ Table Schema:                                                │
│   PK: device_id (String)                                    │
│   SK: timestamp (Number)                                    │
│   Attributes: temperature, humidity, battery (Number)       │
│                                                              │
│ Cost: ~$2,500/month (50K WRU/sec × $1.25/million)           │
└─────────────────────────────────────────────────────────────┘
```

**Product 3: Data Warehouse (Redshift)**
```
┌─────────────────────────────────────────────────────────────┐
│ S3 Data Lake (raw data, parquet)                            │
│         ↓                                                    │
│ COPY Command (parallel load)                                │
│         ↓                                                    │
│ Redshift Cluster (ra3.4xlarge × 4 nodes)                    │
│ ├─ Leader Node (query planning)                             │
│ └─ Compute Nodes (4× parallel execution)                    │
│                                                              │
│ Schema:                                                      │
│   Fact Table: fact_transactions (DISTKEY: customer_key)     │
│   Dim Table: dim_customers (DISTSTYLE ALL)                  │
│   Dim Table: dim_products (DISTSTYLE ALL)                   │
│   Dim Table: dim_date (DISTSTYLE ALL)                       │
│                                                              │
│ Optimization:                                                │
│   - SORTKEY on fact_transactions.date_key (range queries)   │
│   - Materialized views for common aggregations              │
│   - Result caching (sub-second repeat queries)              │
│                                                              │
│ Cost: ~$8,500/month (4× ra3.4xlarge, 1 year reserved)       │
└─────────────────────────────────────────────────────────────┘
```

**Cost Comparison**:

| Product | Annual Cost | Cost per Transaction | Scalability |
|---------|-------------|---------------------|-------------|
| E-Commerce (Aurora) | $38,400 | $0.000012 per txn | Vertical (up to 96 vCPUs) + Horizontal (15 replicas) |
| IoT (DynamoDB) | $30,000 | $0.00000019 per write | Unlimited horizontal |
| Data Warehouse (Redshift) | $102,000 | $0.28 per query | Horizontal (add nodes) |

**Anti-Patterns**:

| Mistake | Why It's Wrong | Correct Approach |
|---------|---------------|------------------|
| Using RDS for IoT time-series | Can't scale to 50K writes/sec | DynamoDB or Amazon Timestream |
| Using DynamoDB for OLAP | No complex joins, slow aggregations | Redshift or Athena |
| Using Redshift for OLTP | Not designed for row-level updates | Aurora or RDS |
| Using Aurora for unstructured data | Schema rigidity | DynamoDB or DocumentDB |

**Key Lessons**:
1. **OLTP vs OLAP determines database type**: Transactional workloads → Aurora/RDS, Analytical → Redshift
2. **Access patterns matter**: Simple key-value → DynamoDB, Complex joins → Aurora
3. **Scale requirements drive choice**: 50K writes/sec → DynamoDB, not RDS
4. **Cost optimization**: On-demand (DynamoDB) for unpredictable workloads, Reserved Instances (Redshift) for steady state

---

#### **Q18: Database Security - Encryption, Network Isolation, and Compliance**

**Scenario**: Your healthcare SaaS platform must comply with HIPAA regulations. Design a database security architecture for Aurora PostgreSQL that meets:
- Encryption at rest and in transit
- Network isolation (no public access)
- Audit logging of all database queries
- IAM-based authentication
- Secrets rotation
- Least privilege access

**Security Architecture**:

```
┌──────────────────────────────────────────────────────────────┐
│          HIPAA-COMPLIANT DATABASE SECURITY                   │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Application (Private Subnet)                                │
│  ┌────────────────────────────────────────────────┐          │
│  │  ECS Tasks / Lambda Functions                  │          │
│  │  IAM Role: AppDatabaseAccessRole               │          │
│  │  (No hardcoded credentials)                    │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  VPC Endpoint (AWS PrivateLink)                              │
│  ┌────────────────────────────────────────────────┐          │
│  │  RDS Endpoint: *.rds.amazonaws.com             │          │
│  │  Private DNS Resolution                        │          │
│  │  (No internet gateway)                         │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Aurora PostgreSQL Cluster (Private Subnet)                  │
│  ┌────────────────────────────────────────────────┐          │
│  │  ✅ Encryption at Rest: KMS (CMK)              │          │
│  │  ✅ Encryption in Transit: SSL/TLS             │          │
│  │  ✅ IAM Database Authentication                │          │
│  │  ✅ Audit Logging: pgaudit extension           │          │
│  │  ✅ Backup Encryption: Same KMS key            │          │
│  │  ✅ No Public IP                                │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Audit Logs → CloudWatch Logs → S3 (encrypted)               │
│  ┌────────────────────────────────────────────────┐          │
│  │  Retention: 7 years (HIPAA requirement)        │          │
│  │  Immutable: S3 Object Lock (WORM)              │          │
│  └────────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

**Implementation**:

**Step 1: Create Encrypted Aurora Cluster**

```bash
# Create KMS key for encryption
aws kms create-key \
  --description "Aurora HIPAA encryption key" \
  --key-policy file://kms-policy.json

# Create Aurora cluster with encryption
aws rds create-db-cluster \
  --db-cluster-identifier hipaa-aurora-cluster \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username postgres \
  --master-user-password "${TEMP_PASSWORD}" \
  --database-name hipaa_db \
  --storage-encrypted \
  --kms-key-id alias/aurora-hipaa-key \
  --enable-iam-database-authentication \
  --enable-cloudwatch-logs-exports '["postgresql"]' \
  --backup-retention-period 35 \
  --db-subnet-group-name private-subnet-group \
  --vpc-security-group-ids sg-0abc1234 \
  --deletion-protection \
  --no-publicly-accessible
```

**Step 2: Configure Network Isolation**

```bash
# Create security group (allows only private subnet access)
aws ec2 create-security-group \
  --group-name aurora-hipaa-sg \
  --description "HIPAA Aurora security group" \
  --vpc-id vpc-12345678

# Allow PostgreSQL from application subnet only
aws ec2 authorize-security-group-ingress \
  --group-id sg-aurora-hipaa \
  --protocol tcp \
  --port 5432 \
  --source-group sg-application-tier

# NO public internet access rule (best practice)
```

**Step 3: Enable IAM Database Authentication**

```sql
-- Create database user with IAM authentication
CREATE USER app_user WITH LOGIN;
GRANT rds_iam TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

**Application Connection Code**:

```python
import boto3
import psycopg2

def get_db_connection():
    # Generate IAM auth token (valid for 15 minutes)
    rds_client = boto3.client('rds')
    
    token = rds_client.generate_db_auth_token(
        DBHostname='hipaa-aurora-cluster.cluster-abc123.us-east-1.rds.amazonaws.com',
        Port=5432,
        DBUsername='app_user',
        Region='us-east-1'
    )
    
    # Connect using IAM token as password
    conn = psycopg2.connect(
        host='hipaa-aurora-cluster.cluster-abc123.us-east-1.rds.amazonaws.com',
        port=5432,
        database='hipaa_db',
        user='app_user',
        password=token,
        sslmode='require',  # Enforce SSL
        sslrootcert='/path/to/rds-ca-cert.pem'
    )
    
    return conn

# Usage
conn = get_db_connection()
cursor = conn.cursor()
cursor.execute("SELECT * FROM patients WHERE patient_id = %s", (patient_id,))
```

**Step 4: Enable Audit Logging (pgaudit)**

```sql
-- Install pgaudit extension
CREATE EXTENSION pgaudit;

-- Configure audit logging
ALTER SYSTEM SET pgaudit.log = 'READ, WRITE, DDL';
ALTER SYSTEM SET pgaudit.log_catalog = OFF;
ALTER SYSTEM SET pgaudit.log_parameter = ON;
ALTER SYSTEM SET pgaudit.log_relation = ON;
ALTER SYSTEM SET pgaudit.log_statement_once = ON;

-- Reload configuration
SELECT pg_reload_conf();

-- Verify audit logging
SELECT name, setting FROM pg_settings WHERE name LIKE 'pgaudit%';
```

**Sample Audit Log Entry**:

```
2024-11-29 10:15:32.456 UTC [12345] app_user@hipaa_db LOG:  AUDIT: SESSION,1,1,READ,SELECT,TABLE,patients,
"SELECT patient_id, name, ssn, diagnosis FROM patients WHERE patient_id = 'PAT_001'",<none>
```

**Step 5: Implement Secrets Rotation**

```bash
# Store master password in Secrets Manager
aws secretsmanager create-secret \
  --name aurora-master-password \
  --description "Aurora master password" \
  --secret-string '{"username":"postgres","password":"InitialPassword123!"}'

# Enable automatic rotation (30 days)
aws secretsmanager rotate-secret \
  --secret-id aurora-master-password \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:AuroraPasswordRotation \
  --rotation-rules AutomaticallyAfterDays=30
```

**Step 6: Enable Backup Encryption**

```bash
# Verify backups are encrypted
aws rds describe-db-cluster-snapshots \
  --db-cluster-identifier hipaa-aurora-cluster \
  --query 'DBClusterSnapshots[*].[DBClusterSnapshotIdentifier,StorageEncrypted,KmsKeyId]'

# Expected output:
/*
[
  ["snapshot-2024-11-29", true, "arn:aws:kms:us-east-1:123456789012:key/abc-123"]
]
*/
```

**Step 7: Export Audit Logs to Immutable S3**

```bash
# Create S3 bucket with Object Lock (WORM compliance)
aws s3api create-bucket \
  --bucket hipaa-audit-logs \
  --region us-east-1 \
  --object-lock-enabled-for-bucket

# Configure bucket lifecycle
aws s3api put-bucket-lifecycle-configuration \
  --bucket hipaa-audit-logs \
  --lifecycle-configuration file://lifecycle.json

# lifecycle.json (retain for 7 years per HIPAA)
{
  "Rules": [{
    "Id": "hipaa-retention",
    "Status": "Enabled",
    "Expiration": {
      "Days": 2555
    },
    "Transitions": [
      {
        "Days": 90,
        "StorageClass": "GLACIER"
      }
    ]
  }]
}

# Enable S3 Object Lock
aws s3api put-object-lock-configuration \
  --bucket hipaa-audit-logs \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 2555
      }
    }
  }'

# Export CloudWatch Logs to S3
aws logs create-export-task \
  --log-group-name /aws/rds/cluster/hipaa-aurora-cluster/postgresql \
  --from 1700000000000 \
  --to 1700086400000 \
  --destination hipaa-audit-logs \
  --destination-prefix audit-logs/
```

**HIPAA Compliance Checklist**:

| Requirement | Implementation | Status |
|-------------|---------------|--------|
| **Encryption at Rest** | KMS CMK encryption | ✅ |
| **Encryption in Transit** | SSL/TLS (sslmode=require) | ✅ |
| **Access Control** | IAM database authentication | ✅ |
| **Audit Logging** | pgaudit extension → CloudWatch → S3 | ✅ |
| **Backup Encryption** | Automated backups with KMS | ✅ |
| **Network Isolation** | Private subnets, no public IP | ✅ |
| **Secrets Management** | Secrets Manager with rotation | ✅ |
| **Data Retention** | 7-year retention (S3 Glacier) | ✅ |
| **Immutability** | S3 Object Lock (COMPLIANCE mode) | ✅ |
| **Access Logging** | CloudTrail + S3 access logs | ✅ |

**Cost Analysis**:

```
Aurora Cluster (HIPAA configuration):
├─ Instances: 1 writer + 2 readers (db.r6g.2xlarge) = $1,138/month
├─ Storage: 500GB × $0.10/GB-month = $50/month
├─ I/O: 10M requests × $0.20/1M = $2/month
├─ Backup Storage: 500GB × 7 days × $0.021/GB-month = $74/month
├─ KMS Encryption: $1/month (CMK) + $0.03/10K requests = $1.50/month
├─ CloudWatch Logs: 50GB/month × $0.50/GB = $25/month
├─ S3 Audit Logs: 600GB/year (Glacier) × $0.004/GB-month = $2/month
└─ TOTAL: $1,293.50/month ($15,522/year)

Compare to On-Premises HIPAA-Compliant Database:
├─ Oracle Database Enterprise (1 server): $47,500/year (licensing)
├─ Hardware: $15,000/year (amortized)
├─ Storage (SAN): $10,000/year
├─ Backup Solution: $8,000/year
├─ Audit/Compliance Tools: $12,000/year
├─ DBA Salary (dedicated): $120,000/year
└─ TOTAL: $212,500/year

AWS Savings: $196,978/year (93% reduction)
```

**Key Lessons**:
1. **IAM authentication eliminates password management**: No credential rotation needed for application users
2. **Network isolation is critical**: No public IP = no internet-based attacks
3. **pgaudit is mandatory for HIPAA**: Logs every query for compliance audits
4. **S3 Object Lock ensures immutability**: Audit logs cannot be tampered with
5. **Encryption everywhere**: At rest, in transit, and for backups

---

#### **Q19: Cost Optimization - Reducing Database Spend by 60%**

**Scenario**: Your CFO mandates a 60% reduction in database costs. Current monthly spend: $25,000 across multiple database services. You must optimize without impacting performance or availability.

**Current State Audit**:

```bash
# Get RDS/Aurora cost breakdown
aws ce get-cost-and-usage \
  --time-period Start=2024-10-01,End=2024-10-31 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=SERVICE,Key=SERVICE \
  --filter file://filter.json

# filter.json
{
  "Dimensions": {
    "Key": "SERVICE",
    "Values": ["Amazon RDS", "Amazon Aurora", "Amazon Redshift", "Amazon DynamoDB"]
  }
}
```

**Cost Analysis Results**:

| Service | Monthly Cost | Usage Pattern | Optimization Opportunity |
|---------|-------------|---------------|------------------------|
| Aurora PostgreSQL (prod) | $8,500 | 24/7, high utilization | Reserved Instances (1-year) |
| RDS MySQL (dev/staging) | $3,200 | 8AM-6PM weekdays only | Stop/start automation |
| Redshift (analytics) | $9,500 | Queries run 2PM-4PM daily | Pause cluster when idle |
| DynamoDB (user sessions) | $3,800 | Spiky traffic | Switch to on-demand mode |
| **TOTAL** | **$25,000** | | **Target: $10,000 (60% reduction)** |

**Optimization Strategy**:

**1. Aurora: Reserved Instances (42% savings)**

```bash
# Purchase 1-year Reserved Instance
aws rds purchase-reserved-db-instances-offering \
  --reserved-db-instances-offering-id <offering-id> \
  --db-instance-count 3 \
  --reserved-db-instance-id aurora-prod-ri

# Before: On-demand pricing
# 3× db.r6g.2xlarge × $0.516/hour × 730 hours = $1,130/month
# Total for 3 instances: $3,390/month

# After: 1-year Reserved Instance (no upfront)
# 3× db.r6g.2xlarge × $0.299/hour × 730 hours = $655/month
# Total for 3 instances: $1,965/month

# Savings: $3,390 - $1,965 = $1,425/month (42% reduction)
```

**2. RDS Dev/Staging: Automated Start/Stop (70% savings)**

```python
# Lambda function to stop dev databases at 6 PM weekdays
import boto3
from datetime import datetime

rds = boto3.client('rds')

def lambda_handler(event, context):
    now = datetime.now()
    
    # Stop at 6 PM on weekdays
    if event['action'] == 'stop' and now.weekday() < 5:
        rds.stop_db_instance(DBInstanceIdentifier='dev-mysql-instance')
        rds.stop_db_instance(DBInstanceIdentifier='staging-mysql-instance')
        return {'status': 'Stopped dev/staging databases'}
    
    # Start at 8 AM on weekdays
    elif event['action'] == 'start' and now.weekday() < 5:
        rds.start_db_instance(DBInstanceIdentifier='dev-mysql-instance')
        rds.start_db_instance(DBInstanceIdentifier='staging-mysql-instance')
        return {'status': 'Started dev/staging databases'}
```

```bash
# Create EventBridge rules
# Stop at 6 PM (weekdays)
aws events put-rule \
  --name stop-dev-databases \
  --schedule-expression "cron(0 18 ? * MON-FRI *)" \
  --state ENABLED

# Start at 8 AM (weekdays)
aws events put-rule \
  --name start-dev-databases \
  --schedule-expression "cron(0 8 ? * MON-FRI *)" \
  --state ENABLED

# Before: 24/7 operation = 730 hours/month
# After: 10 hours/day × 22 business days = 220 hours/month

# Savings: $3,200 × (510/730) = $2,240/month (70% reduction)
# New cost: $3,200 - $2,240 = $960/month
```

**3. Redshift: Pause/Resume + Resize (65% savings)**

```bash
# Schedule Redshift pause when idle
aws redshift pause-cluster --cluster-identifier analytics-cluster

# Before: dc2.8xlarge × 4 nodes × $4.80/hour × 730 hours = $14,016/month
# (But only 2 hours/day actual usage = ~60 hours/month)

# Option A: Pause cluster (85% time idle)
# Running: 60 hours × 4 nodes × $4.80 = $1,152/month
# Paused: No compute charges, only storage
# Storage: 1TB × $0.024/GB-month = $24/month
# New cost: $1,152 + $24 = $1,176/month (92% savings)

# Option B: Downsize to ra3.xlplus × 2 nodes
# $0.867/hour × 2 nodes × 730 hours = $1,266/month (91% savings)

# Choose Option B (ra3.xlplus) for better performance
# Savings: $14,016 - $1,266 = $12,750/month (91% reduction)
```

**Resize Redshift Cluster**:

```bash
# Resize to smaller instance type
aws redshift modify-cluster \
  --cluster-identifier analytics-cluster \
  --node-type ra3.xlplus \
  --number-of-nodes 2 \
  --apply-immediately
```

**4. DynamoDB: Switch to On-Demand (45% savings)**

```bash
# Current: Provisioned mode
# 1,000 WCU × $0.00065/hour × 730 hours = $474.50/month
# 5,000 RCU × $0.00013/hour × 730 hours = $474.50/month
# Total: $949/month

# Actual usage analysis (from CloudWatch):
# Writes: 50M/month
# Reads: 200M/month

# On-Demand pricing:
# Writes: 50M × $1.25/million = $62.50/month
# Reads: 200M × $0.25/million = $50/month
# Total: $112.50/month

# Switch to on-demand
aws dynamodb update-table \
  --table-name user-sessions \
  --billing-mode PAY_PER_REQUEST

# Savings: $949 - $112.50 = $836.50/month (88% reduction)
```

**5. Delete Unused Resources (100% savings)**

```bash
# Identify idle databases (no connections in 30 days)
aws rds describe-db-instances \
  --query 'DBInstances[?Engine==`mysql`].[DBInstanceIdentifier,DBInstanceStatus]'

# Check CloudWatch metrics for connections
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=old-test-db \
  --start-time 2024-10-01T00:00:00Z \
  --end-time 2024-10-31T23:59:59Z \
  --period 86400 \
  --statistics Maximum

# If Max = 0, delete the database
aws rds delete-db-instance \
  --db-instance-identifier old-test-db \
  --skip-final-snapshot

# Found 3 unused RDS instances: $1,200/month savings
```

**6. Aurora Storage Optimization (40% savings)**

```sql
-- Check table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - 
                   pg_relation_size(schemaname||'.'||tablename)) AS indexes_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Result: Found log_events table (500GB, 80% of database)
-- Solution: Implement TTL and move old logs to S3

-- Archive old logs
COPY (SELECT * FROM log_events WHERE created_at < NOW() - INTERVAL '90 days')
TO PROGRAM 'aws s3 cp - s3://logs-archive/log_events.csv'
WITH CSV HEADER;

-- Delete archived logs
DELETE FROM log_events WHERE created_at < NOW() - INTERVAL '90 days';

-- Vacuum to reclaim space
VACUUM FULL log_events;

-- Before: 625GB storage × $0.10/GB-month = $62.50/month
-- After: 125GB storage × $0.10/GB-month = $12.50/month
-- Savings: $50/month (80% reduction)
```

**Total Savings Summary**:

| Optimization | Before | After | Monthly Savings | % Reduction |
|--------------|--------|-------|----------------|-------------|
| Aurora Reserved Instances | $3,390 | $1,965 | $1,425 | 42% |
| RDS Dev/Staging Automation | $3,200 | $960 | $2,240 | 70% |
| Redshift Resize | $14,016 | $1,266 | $12,750 | 91% |
| DynamoDB On-Demand | $949 | $112 | $837 | 88% |
| Delete Unused Databases | $1,200 | $0 | $1,200 | 100% |
| Aurora Storage Cleanup | $62 | $12 | $50 | 80% |
| **TOTAL** | **$25,000** | **$4,315** | **$20,685** | **83%** |

**Result**: Exceeded target! (83% reduction vs 60% target)

**Cost Monitoring Setup**:

```bash
# Create budget alert
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://budget.json

# budget.json
{
  "BudgetName": "DatabaseCostBudget",
  "BudgetLimit": {
    "Amount": "5000",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {
    "Service": ["Amazon RDS", "Amazon Aurora", "Amazon Redshift", "Amazon DynamoDB"]
  }
}

# Create alarm for 80% threshold
aws budgets create-notification \
  --account-id 123456789012 \
  --budget-name DatabaseCostBudget \
  --notification NotificationType=ACTUAL,ComparisonOperator=GREATER_THAN,Threshold=80 \
  --subscriber SubscriptionType=EMAIL,Address=team@company.com
```

**Key Lessons**:
1. **Right-sizing saves 60-90%**: Most databases are over-provisioned
2. **Dev/test environments waste money**: Stop them when not in use (70% savings)
3. **Reserved Instances for prod workloads**: Predictable usage = 40% discount
4. **On-demand pricing for variable workloads**: DynamoDB spiky traffic fits on-demand
5. **Storage cleanup is easy money**: Archive old data to S3 (10x cheaper)
6. **Monitor usage before scaling**: CloudWatch metrics reveal actual needs

---

#### **Q20: Disaster Recovery - Multi-Region Database Failover**

**Scenario**: Your company's SLA requires 99.99% uptime (< 4.38 minutes downtime/month). Design a multi-region disaster recovery strategy for Aurora PostgreSQL with:
- RTO (Recovery Time Objective): < 5 minutes
- RPO (Recovery Point Objective): < 1 minute
- Automated failover
- Regular DR testing
- Cost-effective architecture

**Multi-Region DR Architecture**:

```
┌──────────────────────────────────────────────────────────────┐
│ PRIMARY REGION: us-east-1                                    │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Route 53 (Health Checks)                                    │
│  ┌────────────────────────────────────────────────┐          │
│  │  Primary: app.example.com → us-east-1-alb      │          │
│  │  Failover: app.example.com → eu-west-1-alb     │          │
│  │  Health Check: /health endpoint (30s interval) │          │
│  └────────────────────────────────────────────────┘          │
│                      ↓                                        │
│  Application Load Balancer (us-east-1)                       │
│         ↓                                                     │
│  Application Tier (ECS Fargate)                              │
│         ↓                                                     │
│  Aurora Global Database Cluster                              │
│  ┌────────────────────────────────────────────────┐          │
│  │  Primary Cluster (us-east-1)                   │          │
│  │  ├─ Writer Instance (db.r6g.2xlarge)           │          │
│  │  ├─ Reader 1 (db.r6g.2xlarge)                  │          │
│  │  └─ Reader 2 (db.r6g.2xlarge)                  │          │
│  │         │                                       │          │
│  │         │ Replication (< 1 second lag)          │          │
│  │         ▼                                       │          │
│  └────────────────────────────────────────────────┘          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────────┐
│ SECONDARY REGION: eu-west-1 (Read Replica)                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Aurora Global Database (Secondary Region)                   │
│  ┌────────────────────────────────────────────────┐          │
│  │  Secondary Cluster (eu-west-1) - READ ONLY     │          │
│  │  ├─ Reader 1 (db.r6g.2xlarge)                  │          │
│  │  ├─ Reader 2 (db.r6g.2xlarge)                  │          │
│  │         │                                       │          │
│  │         └─ Serves read-only queries             │          │
│  │            (offloads primary region traffic)    │          │
│  └────────────────────────────────────────────────┘          │
│                                                               │
│  Application Tier (ECS Fargate) - STANDBY                    │
│  ├─ Desired count: 2 (minimum for health checks)             │
│  └─ Auto-scales to full capacity during failover             │
│                                                               │
└──────────────────────────────────────────────────────────────┘

DURING FAILOVER (RTO: 2-3 minutes):
┌──────────────────────────────────────────────────────────────┐
│ 1. Route 53 detects primary region failure (30s)            │
│ 2. DNS failover to eu-west-1 (10s)                          │
│ 3. Promote eu-west-1 cluster to writer (60s)                │
│ 4. Application connects to new primary (10s)                │
│ 5. Total RTO: ~2 minutes                                     │
└──────────────────────────────────────────────────────────────┘
```

**Implementation**:

**Step 1: Create Aurora Global Database**

```bash
# Create primary cluster (us-east-1)
aws rds create-db-cluster \
  --db-cluster-identifier prod-global-primary \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username postgres \
  --master-user-password "${MASTER_PASSWORD}" \
  --database-name proddb \
  --backup-retention-period 7 \
  --storage-encrypted \
  --kms-key-id alias/aws/rds \
  --enable-iam-database-authentication \
  --region us-east-1

# Create global database
aws rds create-global-cluster \
  --global-cluster-identifier prod-global-cluster \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:prod-global-primary \
  --region us-east-1

# Add secondary region (eu-west-1)
aws rds create-db-cluster \
  --db-cluster-identifier prod-global-secondary \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --global-cluster-identifier prod-global-cluster \
  --region eu-west-1

# Create instances in primary region
for i in 1 2 3; do
  aws rds create-db-instance \
    --db-instance-identifier prod-global-primary-node-$i \
    --db-cluster-identifier prod-global-primary \
    --db-instance-class db.r6g.2xlarge \
    --engine aurora-postgresql \
    --region us-east-1
done

# Create instances in secondary region
for i in 1 2; do
  aws rds create-db-instance \
    --db-instance-identifier prod-global-secondary-node-$i \
    --db-cluster-identifier prod-global-secondary \
    --db-instance-class db.r6g.2xlarge \
    --engine aurora-postgresql \
    --region eu-west-1
done
```

**Step 2: Configure Route 53 Health Checks**

```bash
# Create health check for primary region
aws route53 create-health-check \
  --health-check-config \
    IPAddress=3.80.5.123,\
    Port=443,\
    Type=HTTPS,\
    ResourcePath=/health,\
    FullyQualifiedDomainName=us-east-1-alb.example.com,\
    RequestInterval=30,\
    FailureThreshold=3

# Create failover DNS records
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch file://failover-dns.json

# failover-dns.json
{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "Primary-US-EAST-1",
        "Failover": "PRIMARY",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "us-east-1-alb.example.com",
          "EvaluateTargetHealth": true
        },
        "HealthCheckId": "abc-health-check-123"
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.example.com",
        "Type": "A",
        "SetIdentifier": "Secondary-EU-WEST-1",
        "Failover": "SECONDARY",
        "AliasTarget": {
          "HostedZoneId": "Z32O12XQLNTSW2",
          "DNSName": "eu-west-1-alb.example.com",
          "EvaluateTargetHealth": false
        }
      }
    }
  ]
}
```

**Step 3: Automated Failover Lambda Function**

```python
import boto3
import time

rds_client = boto3.client('rds')
route53_client = boto3.client('route53')

def lambda_handler(event, context):
    """
    Automated DR failover:
    1. Detect primary region failure (from CloudWatch Alarm)
    2. Promote secondary region to primary
    3. Update application configuration
    4. Send notifications
    """
    
    global_cluster_id = 'prod-global-cluster'
    secondary_cluster_arn = 'arn:aws:rds:eu-west-1:123456789012:cluster:prod-global-secondary'
    
    # Step 1: Remove secondary from global cluster
    print(f"[1/4] Detaching {secondary_cluster_arn} from global cluster...")
    rds_client.remove_from_global_cluster(
        GlobalClusterIdentifier=global_cluster_id,
        DbClusterIdentifier=secondary_cluster_arn
    )
    
    # Wait for detachment (30-60 seconds)
    time.sleep(60)
    
    # Step 2: Verify cluster is standalone (now writable)
    print(f"[2/4] Promoting eu-west-1 to writer...")
    cluster_info = rds_client.describe_db_clusters(
        DBClusterIdentifier='prod-global-secondary'
    )
    
    endpoint = cluster_info['DBClusters'][0]['Endpoint']
    print(f"New writer endpoint: {endpoint}")
    
    # Step 3: Update Parameter Store (application reads DB endpoint from here)
    ssm_client = boto3.client('ssm', region_name='eu-west-1')
    ssm_client.put_parameter(
        Name='/prod/database/endpoint',
        Value=endpoint,
        Overwrite=True,
        Type='String'
    )
    
    # Step 4: Trigger ECS task refresh (picks up new endpoint)
    ecs_client = boto3.client('ecs', region_name='eu-west-1')
    ecs_client.update_service(
        cluster='prod-cluster',
        service='app-service',
        forceNewDeployment=True
    )
    
    # Step 5: Send notifications
    sns_client = boto3.client('sns')
    sns_client.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:dr-alerts',
        Subject='🔴 DR FAILOVER EXECUTED',
        Message=f"""
        PRIMARY REGION FAILED: us-east-1
        FAILED OVER TO: eu-west-1
        NEW DATABASE ENDPOINT: {endpoint}
        RTO ACHIEVED: ~2 minutes
        
        Action Required:
        1. Investigate us-east-1 outage
        2. Monitor eu-west-1 performance
        3. Plan failback after us-east-1 recovery
        """
    )
    
    return {
        'statusCode': 200,
        'body': f'Failover complete. New primary: eu-west-1. Endpoint: {endpoint}'
    }
```

**Step 4: DR Testing Schedule (Quarterly)**

```bash
#!/bin/bash
# dr-test.sh (execute quarterly to validate failover works)

set -e

echo "=== DISASTER RECOVERY TEST ==="
echo "Start: $(date)"

# 1. Verify replication lag (must be < 1 second)
echo "[1/7] Checking replication lag..."
REPLICATION_LAG=$(aws rds describe-db-clusters \
  --db-cluster-identifier prod-global-secondary \
  --region eu-west-1 \
  --query 'DBClusters[0].GlobalWriteForwardingStatus' --output text)

if [ "$REPLICATION_LAG" != "enabled" ]; then
  echo "❌ Replication lag check failed"
  exit 1
fi

# 2. Simulate primary region failure (stop application in us-east-1)
echo "[2/7] Stopping application in us-east-1..."
aws ecs update-service \
  --cluster prod-cluster \
  --service app-service \
  --desired-count 0 \
  --region us-east-1

# 3. Wait for Route 53 health check to fail (90 seconds)
echo "[3/7] Waiting for Route 53 failover..."
sleep 90

# 4. Verify DNS now points to eu-west-1
echo "[4/7] Verifying DNS failover..."
DNS_RESULT=$(dig +short app.example.com)
echo "DNS resolves to: $DNS_RESULT"

# 5. Manually trigger failover (or wait for automated Lambda)
echo "[5/7] Executing failover to eu-west-1..."
aws lambda invoke \
  --function-name DR-Failover-Function \
  --region us-east-1 \
  response.json

# 6. Verify application is healthy in eu-west-1
echo "[6/7] Testing application in eu-west-1..."
HTTP_CODE=$(curl -o /dev/null -s -w "%{http_code}\n" https://app.example.com/health)

if [ "$HTTP_CODE" != "200" ]; then
  echo "❌ Application health check failed: $HTTP_CODE"
  exit 1
fi

# 7. Test database write in eu-west-1
echo "[7/7] Testing database write..."
psql -h prod-global-secondary.cluster-xyz.eu-west-1.rds.amazonaws.com \
     -U postgres -d proddb -c "INSERT INTO dr_test VALUES (NOW(), 'DR test successful');"

echo "✅ DR TEST COMPLETE"
echo "RTO Achieved: ~2 minutes"
echo "RPO Achieved: < 1 minute (replication lag)"
echo "End: $(date)"

# Failback (restore us-east-1 as primary)
# ... (reverse the process)
```

**Cost Analysis**:

```
Multi-Region DR Architecture:

PRIMARY REGION (us-east-1):
├─ Aurora Writer (db.r6g.2xlarge): $379/month
├─ Aurora Reader 1 (db.r6g.2xlarge): $379/month
├─ Aurora Reader 2 (db.r6g.2xlarge): $379/month
├─ Storage: 500GB × $0.10/GB-month = $50/month
└─ Subtotal: $1,187/month

SECONDARY REGION (eu-west-1):
├─ Aurora Reader 1 (db.r6g.2xlarge): $379/month (EU pricing: $420/month)
├─ Aurora Reader 2 (db.r6g.2xlarge): $379/month (EU pricing: $420/month)
├─ Cross-Region Replication: 50GB/day × 30 × $0.02/GB = $30/month
└─ Subtotal: $870/month

ADDITIONAL COSTS:
├─ Route 53 Health Checks: 2 × $0.50/month = $1/month
├─ Lambda (failover automation): $0.20/month (rarely invoked)
└─ CloudWatch Alarms: 5 × $0.10/month = $0.50/month

TOTAL: $2,058.70/month ($24,704/year)

Compare to Single-Region:
└─ Single Region Cost: $1,187/month ($14,244/year)

DR Premium: $870/month ($10,440/year) for 99.99% availability
```

**Availability Comparison**:

| Architecture | Availability | Downtime/Month | Annual Downtime | Cost/Month |
|--------------|--------------|----------------|----------------|------------|
| Single Region (Multi-AZ) | 99.95% | 21.6 minutes | 4.38 hours | $1,187 |
| Multi-Region (Global Database) | 99.99% | 4.38 minutes | 52.56 minutes | $2,059 |
| Multi-Region (Active-Active) | 99.995% | 2.19 minutes | 26.28 minutes | $4,200 |

**Key Lessons**:
1. **Aurora Global Database replicates in <1 second**: RPO < 1 minute achievable
2. **Route 53 health checks enable automated failover**: No manual intervention needed
3. **Test DR quarterly**: Catches configuration drift and validates RTO/RPO
4. **Cost of DR is insurance**: 73% premium ($870/month) buys 99.99% availability
5. **Failback is as important as failover**: Practice both directions

---

### **🎓 CERTIFICATION TIPS - AWS CERTIFIED DATA ENGINEER ASSOCIATE**

#### **What to Focus On for the Exam**

**High-Priority Topics (60% of questions)**:
1. **Aurora vs RDS**: When to use each, pricing differences, read replicas
2. **Redshift**: DISTKEY, SORTKEY, Spectrum, materialized views, COPY command
3. **DynamoDB**: Partition keys, GSI vs LSI, on-demand vs provisioned
4. **Database Migration**: DMS replication modes, SCT, full load vs CDC
5. **Performance**: Query optimization, indexing strategies, monitoring

**Exam Traps - Common Wrong Answers**:

| Scenario | Wrong Answer (Trap) | Correct Answer | Why |
|----------|--------------------|-----------------|----|
| "50,000 writes/second" | Aurora PostgreSQL | **DynamoDB** | Aurora can't scale to 50K writes/sec |
| "Petabyte-scale analytics" | RDS PostgreSQL with read replicas | **Amazon Redshift** | RDS not designed for PB-scale OLAP |
| "Global low-latency reads" | Aurora with cross-region read replicas | **Aurora Global Database** | Global Database has <1 sec replication |
| "IoT time-series data" | RDS MySQL | **DynamoDB or Timestream** | RDS can't handle IoT write volume |
| "Complex JOINs, 10TB data" | DynamoDB with GSI | **Aurora or Redshift** | DynamoDB poor for multi-table joins |

**Key Limits to Memorize**:

| Service | Limit | Exam Relevance |
|---------|-------|---------------|
| Aurora Read Replicas | Up to 15 | High |
| Aurora Max Storage | 128 TB (auto-scales) | Medium |
| RDS Read Replicas | Up to 15 (cross-region supported) | High |
| RDS Max Storage | 64 TB (gp3) | Medium |
| Redshift Max Nodes | 128 nodes per cluster | Low |
| DynamoDB Item Size | 400 KB max | High |
| DynamoDB GSI per Table | 20 max | Medium |
| DMS Task Limit | 300 tasks per account | Low |

**Scenario-Based Question Patterns**:

**Pattern 1: "Cost Optimization"**  
Question asks to reduce cost while maintaining performance.  
Answer: On-demand pricing (DynamoDB), Reserved Instances (Aurora/Redshift), S3 lifecycle

**Pattern 2: "High Availability"**  
Question asks for 99.99% availability.  
Answer: Multi-AZ (Aurora/RDS), Global Database (Aurora), Multi-region replication

**Pattern 3: "Data Migration"**  
Question asks to migrate Oracle to AWS with minimal downtime.  
Answer: DMS with full load + CDC, SCT for schema conversion

**Pattern 4: "Performance at Scale"**  
Question mentions "petabyte", "billions of rows", "complex analytics".  
Answer: **Redshift** (NOT Aurora, NOT RDS)

---

### **🏭 PRODUCTION BEST PRACTICES**

#### **1. Security Best Practices**

**✅ DO:**
- Enable encryption at rest (KMS) for ALL databases
- Enable encryption in transit (SSL/TLS mandatory)
- Use IAM database authentication (no hardcoded passwords)
- Enable audit logging (pgaudit, Aurora audit logs)
- Restrict database to private subnets (no public IP)
- Use Secrets Manager for credential rotation
- Enable deletion protection for production databases

**❌ DON'T:**
- Use master user for application connections
- Allow public internet access to databases
- Hardcode database credentials in application code
- Disable SSL/TLS enforcement
- Skip automated backups

#### **2. Cost Optimization Best Practices**

**Aurora/RDS:**
- Use Reserved Instances for production (40% savings)
- Right-size instances based on CloudWatch metrics
- Delete unused read replicas
- Stop dev/test databases outside business hours
- Use Aurora Serverless v2 for variable workloads

**Redshift:**
- Use RA3 instances (separate compute from storage)
- Pause cluster when idle (> 1 hour)
- Use Spectrum for cold data (cheaper than loading into Redshift)
- Enable result caching (free repeated queries)
- Archive old data to S3

**DynamoDB:**
- Switch to on-demand for spiky/unpredictable traffic
- Use provisioned for steady-state workloads
- Enable auto-scaling for provisioned tables
- Implement TTL to auto-delete expired items
- Use DynamoDB Accelerator (DAX) for read-heavy workloads

#### **3. Scalability Best Practices**

**Vertical Scaling (Aurora/RDS):**
- Monitor CPU, memory, IOPS before scaling
- Use Performance Insights to identify bottlenecks
- Scale writer during low-traffic periods
- Test new instance size in staging first

**Horizontal Scaling:**
- Aurora: Add read replicas (up to 15)
- Redshift: Add nodes (resize cluster)
- DynamoDB: Increase WCU/RCU or use on-demand

**Connection Pooling:**
- Use RDS Proxy for Aurora/RDS (connection pooling)
- Reduces database connections from 1,000s to 10s
- Improves failover time (66% faster)

#### **4. Monitoring Best Practices**

**Essential CloudWatch Metrics:**

**Aurora/RDS:**
- DatabaseConnections (alert if > 80% of max)
- CPUUtilization (alert if > 80%)
- FreeableMemory (alert if < 20%)
- ReadLatency, WriteLatency (alert if > 10ms)
- ReplicaLag (alert if > 5 seconds)

**Redshift:**
- CPUUtilization (alert if > 90%)
- HealthStatus (alert if unhealthy)
- PercentageDiskSpaceUsed (alert if > 80%)
- QueriesCompletedPerSecond (monitor trends)

**DynamoDB:**
- ConsumedReadCapacityUnits (vs provisioned)
- ConsumedWriteCapacityUnits (vs provisioned)
- UserErrors (throttling events)
- SystemErrors (service errors)

**Performance Insights:**
- Enable for Aurora/RDS (free for 7 days retention)
- Identifies top SQL statements by load
- Shows wait events (CPU, I/O, locks)

#### **5. High Availability Best Practices**

**Aurora:**
- Always deploy Multi-AZ (automatic failover in 30-120 seconds)
- Create at least 2 read replicas (different AZs)
- Use Aurora Global Database for DR (RTO < 1 minute)
- Enable backtrack (point-in-time recovery without restoring)

**RDS:**
- Enable Multi-AZ for production (automatic failover)
- Create cross-region read replicas for DR
- Automate snapshots daily
- Test restore process quarterly

**Redshift:**
- Use multi-node cluster (leader + compute nodes)
- Enable automated snapshots
- Configure cross-region snapshot copy
- Monitor cluster health with CloudWatch

---

### **💪 HANDS-ON CHALLENGE**

**Challenge 1: Build a Multi-Tier Database Architecture**

**Objective**: Design a complete database architecture for an e-commerce platform with OLTP (Aurora), OLAP (Redshift), and caching (DynamoDB).

**Requirements**:
- Aurora PostgreSQL for orders, customers, products (transactional)
- Redshift for sales analytics (data warehouse)
- DynamoDB for shopping cart sessions (cache)
- Automated ETL pipeline: Aurora → S3 → Redshift
- Cross-region disaster recovery
- Cost: Under $5,000/month

**Tasks**:
1. Create Aurora Global Database (us-east-1 primary, eu-west-1 secondary)
2. Create Redshift cluster (2 nodes, ra3.xlplus)
3. Create DynamoDB table for sessions
4. Build Glue ETL job to sync Aurora → S3 → Redshift daily
5. Implement monitoring dashboard (CloudWatch)
6. Document cost breakdown

**Success Criteria**:
- Aurora handles 5,000 TPS
- Redshift queries complete in < 5 seconds
- DynamoDB handles 10,000 writes/second
- DR failover works (tested)
- Total cost < $5,000/month

---

**Challenge 2: Database Performance Optimization**

**Objective**: Optimize a slow-performing Redshift cluster using distribution keys, sort keys, and materialized views.

**Scenario**: You inherit a Redshift cluster with these problems:
- Queries take 5+ minutes (target: < 30 seconds)
- High disk spill (queries writing to SSD)
- Uneven data distribution (90% on one node)

**Tasks**:
1. Analyze query performance using system tables
2. Identify poorly distributed tables
3. Redesign schema with appropriate DISTKEY and SORTKEY
4. Create materialized views for common queries
5. Implement VACUUM and ANALYZE schedule
6. Measure performance improvements

**Success Criteria**:
- Query performance improved by 90%
- Zero disk spill
- Even data distribution across nodes
- Document before/after metrics

---

**Challenge 3: Implement HIPAA-Compliant Database Security**

**Objective**: Secure an Aurora PostgreSQL database to meet HIPAA compliance requirements.

**Requirements**:
- Encryption at rest (KMS CMK)
- Encryption in transit (SSL/TLS)
- No public IP address
- Audit logging for all queries
- IAM database authentication
- Secrets rotation (30 days)
- 7-year audit log retention

**Tasks**:
1. Create encrypted Aurora cluster in private subnet
2. Configure pgaudit extension
3. Set up IAM database authentication
4. Configure Secrets Manager rotation
5. Export audit logs to S3 with Object Lock
6. Document compliance checklist

**Success Criteria**:
- All HIPAA requirements met
- Audit logs immutable (S3 Object Lock)
- Zero public internet exposure
- Documented compliance evidence

---

### **📝 RESUME PROJECT - How to Present This Work**

**For Data Engineer Role:**

```markdown
## Multi-Region Database Architecture for E-Commerce Platform
**Technologies**: Amazon Aurora PostgreSQL, Amazon Redshift, DynamoDB, AWS Glue  
**Impact**: Designed and implemented a highly available, multi-region database 
architecture supporting 10,000 transactions/second with 99.99% uptime SLA.

**Key Achievements**:
- Architected Aurora Global Database with <1 second cross-region replication
- Optimized Redshift cluster, reducing query time from 5 minutes to 15 seconds (95% improvement)
- Implemented automated ETL pipeline using AWS Glue (Aurora → S3 → Redshift)
- Reduced database costs by 60% through Reserved Instances and auto-scaling
- Achieved RTO < 2 minutes and RPO < 1 minute for disaster recovery

**Technical Details**:
- Migrated 5TB Oracle database to Aurora PostgreSQL using AWS DMS with zero downtime
- Designed star schema for Redshift with optimized DISTKEY/SORTKEY
- Implemented HIPAA-compliant security (encryption, audit logging, IAM authentication)
- Automated cost optimization: saved $20,000/month through instance right-sizing
```

**For Solution Engineer / Consultant Role:**

```markdown
## Database Modernization & Cost Optimization - Healthcare Client
**Client**: Fortune 500 Healthcare Provider  
**Challenge**: $800K/year Oracle licensing costs, limited scalability, compliance concerns

**Solution Delivered**:
- Migrated 5TB Oracle RAC to Aurora PostgreSQL (94% cost reduction)
- Implemented HIPAA-compliant security architecture
- Designed multi-region disaster recovery (RTO < 5 minutes)
- Created Redshift data warehouse for analytics (replacing on-prem Teradata)

**Business Impact**:
- $750K/year cost savings (93% reduction)
- Zero downtime migration (full load + CDC strategy)
- 10x query performance improvement (Redshift vs Oracle OLAP)
- Compliance achieved: HIPAA, SOC 2, GDPR

**Technologies**: Aurora PostgreSQL, Redshift, DMS, SCT, KMS, CloudTrail
```

---

### **🔧 SOLUTION ENGINEER PERSPECTIVE**

#### **Customer Discovery Questions**

When engaging with customers about database services, ask:

1. **Current State**:
   - What database platforms are you using today? (Oracle, SQL Server, MySQL, PostgreSQL, MongoDB)
   - What are your current licensing costs?
   - What are your performance bottlenecks?
   - What's your current RTO/RPO for disaster recovery?

2. **Workload Characteristics**:
   - Is this an OLTP or OLAP workload?
   - What are your transaction volumes? (TPS, QPS)
   - What's your data growth rate?
   - Do you have spiky or steady-state traffic?

3. **Requirements**:
   - What are your availability requirements? (99.9%, 99.99%, 99.999%)
   - What compliance regulations apply? (HIPAA, PCI-DSS, GDPR, SOC 2)
   - What's your acceptable downtime for maintenance?
   - Do you need multi-region support?

4. **Pain Points**:
   - What's your biggest operational challenge?
   - How long does it take to provision a new database?
   - How do you handle backups and disaster recovery?
   - What keeps you up at night regarding databases?

#### **Common Objections & Responses**

**Objection 1**: *"Our Oracle DBAs are experts. Why should we migrate to Aurora?"*

**Response**:  
"Aurora is PostgreSQL and MySQL compatible, so your DBAs' SQL skills transfer directly. The difference is that Aurora eliminates 80% of the operational burden:
- No manual failover (automatic in 30 seconds)
- No storage provisioning (auto-scales to 128TB)
- No index rebuilds (handled automatically)
- No archivelog management (continuous backups)

This frees your DBAs to focus on schema design, query optimization, and application support instead of babysitting infrastructure. We've helped teams cut DBA operational work by 60% after migration."

---

**Objection 2**: *"Redshift is too expensive for our analytics workload."*

**Response**:  
"Let's compare apples to apples. Your current on-prem Teradata costs:
- $500K/year in software licensing
- $200K/year in hardware (amortized)
- $150K/year in DBA salaries
- **Total: $850K/year**

Redshift with RA3 nodes:
- $102K/year (4 nodes, 1-year Reserved Instances)
- Zero hardware costs
- Minimal DBA overhead (1/4 time)
- **Total: ~$150K/year (82% reduction)**

Plus, with Redshift Spectrum, you can query 90% of your cold data directly in S3 without loading it, reducing costs even further. We see customers save 70-90% vs on-prem data warehouses."

---

**Objection 3**: *"We tried DynamoDB before and it was more expensive than expected."*

**Response**:  
"DynamoDB pricing model changed dramatically with on-demand mode. Let me ask:
- What was your usage pattern? Steady-state or spiky?
- Were you using provisioned capacity or on-demand?
- Did you implement TTL to auto-delete expired items?

Most customers who had bad experiences used provisioned capacity for unpredictable workloads. Today's on-demand mode means you pay ONLY for what you use - no wasted capacity. Plus, with DynamoDB Accelerator (DAX), you can cache read-heavy workloads and reduce RCU consumption by 90%. Let's model your actual usage and I'll show you the real cost."

---

#### **Use Cases to Highlight**

**Aurora Use Cases**:
- ✅ E-commerce platforms (OLTP, high transactions)
- ✅ SaaS applications (multi-tenant, scalable)
- ✅ Financial services (ACID compliance)
- ✅ Replacing commercial databases (Oracle, SQL Server)

**Redshift Use Cases**:
- ✅ Business intelligence / analytics (petabyte-scale)
- ✅ Data warehousing (star schema, aggregations)
- ✅ Replacing Teradata, Netezza, Oracle Exadata
- ✅ Log analytics (combining with S3 via Spectrum)

**DynamoDB Use Cases**:
- ✅ User sessions / caching (key-value, high throughput)
- ✅ IoT / time-series data (device_id + timestamp)
- ✅ Mobile backends (global tables for low latency)
- ✅ Gaming leaderboards (high writes, simple queries)

---

#### **Oracle DBA Transition Guide**

**Oracle → Aurora PostgreSQL Mapping**:

| Oracle Concept | Aurora Equivalent | Notes |
|---------------|------------------|-------|
| RAC (Real Application Clusters) | Aurora Multi-Master | Multi-AZ automatic failover |
| Data Guard | Aurora Global Database | <1 sec cross-region replication |
| Flashback Query | Aurora Backtrack | Point-in-time recovery |
| RMAN | Automated Backups | Continuous backups to S3 |
| Partitioning | Native PostgreSQL Partitioning | Declarative partitioning |
| Materialized Views | PostgreSQL Materialized Views | Must manually refresh |
| AWR (Performance Reports) | Performance Insights | Real-time query performance |
| Enterprise Manager | CloudWatch + RDS Console | Monitoring and alerting |

**Key Differences for Oracle DBAs**:
1. **No SGA/PGA tuning**: Aurora manages memory automatically
2. **No tablespaces**: PostgreSQL uses schemas instead
3. **No init.ora**: Use Parameter Groups (managed via console/CLI)
4. **No listener.ora**: Endpoint provided by AWS (DNS-based)
5. **No patching windows**: Minor version upgrades happen automatically (maintenance window)

---

### **🧹 CLEANUP STEPS**

**IMPORTANT**: To avoid ongoing charges, delete all resources created in this module.

```bash
#!/bin/bash
# cleanup-module3.sh (deletes all database resources)

set -e

echo "=== MODULE 3 CLEANUP ==="
echo "WARNING: This will delete ALL databases created in this module."
read -p "Are you sure? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
  echo "Cleanup cancelled."
  exit 0
fi

# 1. Delete Aurora Global Database
echo "[1/10] Deleting Aurora Global Database..."

# Remove secondary cluster from global database
aws rds remove-from-global-cluster \
  --global-cluster-identifier prod-aurora-global \
  --db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:prod-aurora-secondary \
  --region eu-west-1 || true

# Delete secondary cluster instances
for instance in prod-aurora-secondary-1 prod-aurora-secondary-2; do
  aws rds delete-db-instance \
    --db-instance-identifier $instance \
    --skip-final-snapshot \
    --region eu-west-1 || true
done

# Wait for instances to delete
sleep 120

# Delete secondary cluster
aws rds delete-db-cluster \
  --db-cluster-identifier prod-aurora-secondary \
  --skip-final-snapshot \
  --region eu-west-1 || true

# Delete primary cluster instances
for instance in prod-aurora-primary-1 prod-aurora-primary-2 prod-aurora-primary-3; do
  aws rds delete-db-instance \
    --db-instance-identifier $instance \
    --skip-final-snapshot \
    --region us-east-1 || true
done

sleep 120

# Delete primary cluster
aws rds delete-db-cluster \
  --db-cluster-identifier prod-aurora-primary \
  --skip-final-snapshot \
  --region us-east-1 || true

# Delete global database
aws rds delete-global-cluster \
  --global-cluster-identifier prod-aurora-global \
  --region us-east-1 || true

# 2. Delete Redshift Cluster
echo "[2/10] Deleting Redshift cluster..."
aws redshift delete-cluster \
  --cluster-identifier analytics-cluster \
  --skip-final-cluster-snapshot || true

# 3. Delete DynamoDB Table
echo "[3/10] Deleting DynamoDB table..."
aws dynamodb delete-table --table-name iot-sensor-data || true

# 4. Delete RDS MySQL Instances
echo "[4/10] Deleting RDS MySQL instances..."
aws rds delete-db-instance \
  --db-instance-identifier dev-mysql-instance \
  --skip-final-snapshot || true

aws rds delete-db-instance \
  --db-instance-identifier staging-mysql-instance \
  --skip-final-snapshot || true

# 5. Delete DMS Replication Tasks
echo "[5/10] Deleting DMS tasks..."
aws dms delete-replication-task \
  --replication-task-arn arn:aws:dms:us-east-1:123456789012:task:oracle-migration-task || true

# 6. Delete DMS Endpoints
echo "[6/10] Deleting DMS endpoints..."
aws dms delete-endpoint --endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:oracle-source || true
aws dms delete-endpoint --endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:aurora-target || true

# 7. Delete DMS Replication Instance
echo "[7/10] Deleting DMS replication instance..."
sleep 60
aws dms delete-replication-instance \
  --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:oracle-to-aurora-migrator || true

# 8. Delete CloudWatch Alarms
echo "[8/10] Deleting CloudWatch alarms..."
aws cloudwatch delete-alarms \
  --alarm-names \
    rds-slow-queries \
    redshift-long-query-alert \
    dynamodb-throttling-alert || true

# 9. Delete Secrets Manager Secrets
echo "[9/10] Deleting Secrets Manager secrets..."
aws secretsmanager delete-secret \
  --secret-id aurora-master-password \
  --force-delete-without-recovery || true

aws secretsmanager delete-secret \
  --secret-id oracle-source-db-credentials \
  --force-delete-without-recovery || true

# 10. Delete S3 Buckets (audit logs, migration landing zone)
echo "[10/10] Deleting S3 buckets..."
aws s3 rb s3://migration-landing-zone --force || true
aws s3 rb s3://hipaa-audit-logs --force || true

echo "✅ CLEANUP COMPLETE"
echo "Verify deletion in AWS Console:"
echo "- RDS: https://console.aws.amazon.com/rds/home"
echo "- Redshift: https://console.aws.amazon.com/redshiftv2/home"
echo "- DynamoDB: https://console.aws.amazon.com/dynamodbv2/home"
echo "- DMS: https://console.aws.amazon.com/dms/v2/home"
```

**Manual Cleanup Checklist**:
- [ ] RDS instances deleted (no charges)
- [ ] Redshift cluster deleted (no charges)
- [ ] DynamoDB tables deleted (no charges)
- [ ] DMS tasks/instances deleted (no charges)
- [ ] S3 buckets emptied and deleted (no charges)
- [ ] CloudWatch alarms deleted
- [ ] Secrets Manager secrets deleted
- [ ] CloudWatch log groups deleted (optional, minimal cost)

---

## **🎉 MODULE 3 COMPLETE!**

You have successfully completed **MODULE 3: Database Services for Data Engineering**.

**What You Learned**:
✅ Aurora PostgreSQL cluster setup with read replicas  
✅ Amazon Redshift data warehouse (star schema, DISTKEY, SORTKEY)  
✅ DynamoDB for high-velocity data (partition keys, GSI, on-demand pricing)  
✅ Database migration strategies (Oracle → Aurora using DMS + SCT)  
✅ Performance optimization techniques (indexing, query tuning, materialized views)  
✅ Multi-region disaster recovery (Aurora Global Database, RTO < 5 min)  
✅ Cost optimization strategies (Reserved Instances, auto-scaling, right-sizing)  
✅ HIPAA-compliant security architecture (encryption, IAM auth, audit logging)  

**Next Steps**:
- Practice exercises in your AWS account
- Review scenario-based interview questions
- Complete hands-on challenges
- Move to **MODULE 4** (to be determined - likely Migration and Transfer or Analytics)

---

**Total Exercises**: 3  
**Interview Questions**: 20 (5 beginner + 5 intermediate + 10 scenario-based)  
**Hands-On Challenges**: 3  
**Estimated Completion Time**: 8-10 hours

---


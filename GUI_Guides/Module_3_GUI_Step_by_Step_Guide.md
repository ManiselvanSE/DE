# MODULE 3: Database Services - GUI Step-by-Step Hands-On Guide

## Table of Contents
- [Exercise 3.1: Aurora PostgreSQL with Read Replicas](#exercise-31-aurora-postgresql-with-read-replicas)
- [Exercise 3.2: Redshift Data Warehouse](#exercise-32-redshift-data-warehouse)
- [Exercise 3.3: DynamoDB for IoT Data](#exercise-33-dynamodb-for-iot-data)

---

# Exercise 3.1: Aurora PostgreSQL with Read Replicas

**Duration:** 2-3 hours | **Estimated Cost:** ~$15-20 for lab session

## 🎯 Learning Objectives
- Create production-grade Aurora PostgreSQL cluster
- Configure Multi-AZ deployment for high availability
- Set up read replicas for read scaling
- Implement backups and point-in-time recovery
- Monitor database performance with Performance Insights
- Configure CloudWatch alarms

---

## PART 1: Create Aurora PostgreSQL Cluster

### Step 1: Navigate to RDS Console
1. Open your web browser and go to **AWS Management Console**
2. Sign in with your AWS account credentials
3. In the top search bar, type **"RDS"**
4. Click on **"RDS"** from the dropdown (or click "Services" → "Database" → "RDS")

### Step 2: Initiate Database Creation
1. On the RDS Dashboard, look for the **"Create database"** button (orange button)
2. Click **"Create database"**

### Step 3: Choose Database Creation Method
1. On the "Create database" page, you'll see two options:
   - **Standard create** ← Select this one
   - Easy create
2. Click the radio button for **"Standard create"**

### Step 4: Select Aurora Engine
1. Under **"Engine options"** section, you'll see several database engines
2. Click on **"Aurora (MySQL Compatible)"** or **"Aurora (PostgreSQL Compatible)"**
3. Select **"Aurora (PostgreSQL Compatible)"**
4. In the **"Edition"** dropdown, ensure **"Aurora PostgreSQL"** is selected

### Step 5: Choose Engine Version
1. Under **"Engine version"**, click the dropdown menu
2. Select **"Aurora PostgreSQL 15.4"** (or the latest stable version available)
3. Leave **"Show versions that support Serverless v2"** unchecked (for traditional provisioned cluster)

### Step 6: Select Template
1. Scroll down to the **"Templates"** section
2. You'll see three options:
   - Production ← Select this
   - Dev/Test
   - Free tier (not available for Aurora)
3. Click the radio button for **"Production"**

### Step 7: Configure Database Settings
1. Under **"Settings"** section:
   
   **DB cluster identifier:**
   - In the text box, type: `prod-aurora-cluster`
   - This is the unique name for your cluster
   
   **Credential Settings:**
   - **Master username:** Leave as `postgres` (or type your preferred username)
   - **Credential management:** Select **"Self managed"**
   - **Master password:** Click **"Auto generate a password"** OR
     - Manually type a password (must be at least 8 characters)
     - Re-enter password in **"Confirm password"** field
   - ✅ Check the box: **"Store password in AWS Secrets Manager"** (recommended)

### Step 8: Choose Instance Configuration
1. Scroll to **"Instance configuration"** section
2. **DB instance class:** Click **"Burstable classes (includes t classes)"** dropdown
3. Change to **"Memory optimized classes (includes r classes)"**
4. Select **"db.r6g.xlarge"** from the list
   - Shows: 4 vCPU, 32 GiB RAM
   - Note the pricing: ~$0.62/hour

### Step 9: Configure Availability & Durability
1. Under **"Availability & durability"** section:
2. ✅ Check: **"Create an Aurora Replica or Reader node in a different AZ"**
   - This enables Multi-AZ deployment
   - You'll see it automatically adds 1 replica

### Step 10: Configure Connectivity
1. Scroll to **"Connectivity"** section
2. **Virtual private cloud (VPC):** 
   - Select your default VPC OR
   - Create new VPC (if you want isolated network)
3. **DB subnet group:** Leave as default OR select existing
4. **Public access:** Select **"No"** (recommended for production)
   - This keeps the database private within VPC
5. **VPC security group:** 
   - Select **"Create new"**
   - In **"New VPC security group name"**, type: `aurora-postgres-sg`
6. **Availability Zone:** Leave as **"No preference"**
7. **Database port:** Leave as **5432** (default PostgreSQL port)

### Step 11: Configure Database Authentication
1. Scroll to **"Database authentication"** section
2. You'll see three options, check both:
   - ✅ **Password authentication**
   - ✅ **Password and IAM database authentication** ← Enable this
   - ⬜ Kerberos authentication

### Step 12: Configure Monitoring
1. Scroll to **"Monitoring"** section
2. **Turn on Performance Insights:**
   - ✅ Check **"Enable Performance Insights"**
   - **Retention period:** Select **"7 days (free tier)"**
   - **AWS KMS key:** Select **"(default) aws/rds"**
3. **Enhanced monitoring:**
   - ✅ Check **"Enable Enhanced Monitoring"**
   - **Granularity:** Select **"60 seconds"**
   - **Monitoring Role:** Select **"Default"** (creates rds-monitoring-role automatically)

### Step 13: Configure Additional Settings
1. Click **"Additional configuration"** to expand (if collapsed)
2. **Initial database name:**
   - Type: `proddb`
   - This creates the initial database upon cluster creation
3. **DB cluster parameter group:** Leave as default
4. **DB parameter group:** Leave as default

### Step 14: Configure Backup Settings
1. Under **"Backup"** section:
   - **Backup retention period:** Select **"7 days"** from dropdown
   - ✅ **Enable Backtrack** (optional, for point-in-time restore without restoring from backup)
   - **Backtrack window:** **24 hours**
   - **Backup window:** Select **"No preference"** OR choose specific time

### Step 15: Configure Encryption
1. Under **"Encryption"** section:
   - ✅ Check **"Enable encryption"**
   - **AWS KMS key:** Select **"(default) aws/rds"** OR choose custom key

### Step 16: Configure Log Exports
1. Under **"Log exports"** section:
2. Check the boxes to send logs to CloudWatch:
   - ✅ **Postgresql log**
   - This helps with debugging and monitoring

### Step 17: Configure Maintenance
1. Under **"Maintenance"** section:
   - ✅ Check **"Enable auto minor version upgrade"**
   - **Maintenance window:** Select **"No preference"** OR choose specific window

### Step 18: Enable Deletion Protection
1. Under **"Deletion protection"** section:
   - ✅ Check **"Enable deletion protection"**
   - This prevents accidental deletion

### Step 19: Review Estimated Costs
1. On the right side panel, you'll see **"Estimated monthly costs"**
2. Review: Should show approximately **$450-500/month** for production setup
3. Note: This is for continuous running; labs will cost much less

### Step 20: Create the Database Cluster
1. Scroll to the bottom of the page
2. Click the orange **"Create database"** button
3. You'll see a success message with option to **"View credential details"**
4. **IMPORTANT:** Click **"View credential details"** and save:
   - Master username
   - Master password
   - Endpoint (will be available after creation)
5. Click **"Close"**

### Step 21: Monitor Creation Progress
1. You're now on the **"Databases"** page
2. Look for your cluster: **"prod-aurora-cluster"**
3. **Status** will show:
   - **"Creating"** (first few minutes)
   - **"Backing-up"** (taking initial snapshot)
   - **"Available"** (ready to use - takes ~5-10 minutes)
4. You'll see 2 instances under the cluster:
   - `prod-aurora-cluster-instance-1` (Writer)
   - `prod-aurora-cluster-instance-1-<region>-1` (Reader)

### Step 22: Retrieve Cluster Endpoints
1. Once status is **"Available"**, click on the cluster name **"prod-aurora-cluster"**
2. Scroll to **"Connectivity & security"** tab
3. Note down these endpoints:
   - **Writer endpoint:** `prod-aurora-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com`
   - **Reader endpoint:** `prod-aurora-cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com`
4. Copy these endpoints to a text file for later use

---

## PART 2: Configure Security & Test Connection

### Step 23: Configure Security Group
1. In the RDS console, with your cluster selected, go to **"Connectivity & security"** tab
2. Under **"Security"**, click on the security group link **"aurora-postgres-sg"**
3. This opens the **EC2 Security Groups** console
4. Select the **"aurora-postgres-sg"** security group
5. Click on the **"Inbound rules"** tab at the bottom
6. Click **"Edit inbound rules"** button

### Step 24: Add Inbound Rule for Database Access
1. Click **"Add rule"** button
2. Configure the new rule:
   - **Type:** Select **"PostgreSQL"** from dropdown (automatically sets port 5432)
   - **Protocol:** TCP (auto-filled)
   - **Port range:** 5432 (auto-filled)
   - **Source:** Choose one:
     - **My IP** (if connecting from your computer) ← Recommended for testing
     - OR **Custom:** Enter your VPC CIDR (e.g., `10.0.0.0/16`)
   - **Description:** Type: `Allow PostgreSQL access for testing`
3. Click **"Save rules"** button

### Step 25: Install PostgreSQL Client (Local Computer)

**For macOS:**
1. Open **Terminal** application
2. If you have Homebrew, type: `brew install postgresql`
3. Press Enter and wait for installation

**For Windows:**
1. Download PostgreSQL from: https://www.postgresql.org/download/windows/
2. Run the installer
3. Follow the installation wizard
4. Only install the **"Command Line Tools"** component

**For Linux:**
```bash
sudo apt-get update
sudo apt-get install postgresql-client
```

### Step 26: Test Connection to Writer Endpoint
1. Open **Terminal** (macOS/Linux) or **Command Prompt** (Windows)
2. Copy this command template:
   ```bash
   psql -h <WRITER-ENDPOINT> -U postgres -d proddb
   ```
3. Replace `<WRITER-ENDPOINT>` with your actual writer endpoint
4. Example:
   ```bash
   psql -h prod-aurora-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com -U postgres -d proddb
   ```
5. Press **Enter**
6. When prompted for password, enter the master password you saved earlier
7. If successful, you'll see: `proddb=>`

### Step 27: Verify Database Connection
At the `proddb=>` prompt, run these commands:

1. **Check PostgreSQL version:**
   ```sql
   SELECT version();
   ```
   Press Enter → You'll see PostgreSQL version info

2. **List databases:**
   ```sql
   \l
   ```
   Press Enter → You'll see list of databases

3. **Check current database:**
   ```sql
   SELECT current_database();
   ```
   Press Enter → Should show `proddb`

### Step 28: Create Test Table
1. At the `proddb=>` prompt, copy and paste this command:
   ```sql
   CREATE TABLE employees (
     employee_id SERIAL PRIMARY KEY,
     name VARCHAR(100),
     department VARCHAR(50),
     salary DECIMAL(10,2),
     hire_date DATE
   );
   ```
2. Press **Enter**
3. You should see: `CREATE TABLE`

### Step 29: Insert Sample Data
1. Copy and paste this command:
   ```sql
   INSERT INTO employees (name, department, salary, hire_date) VALUES
   ('John Doe', 'Engineering', 75000.00, '2023-01-15'),
   ('Jane Smith', 'Marketing', 65000.00, '2023-02-20'),
   ('Bob Johnson', 'Engineering', 80000.00, '2023-03-10'),
   ('Alice Williams', 'Sales', 70000.00, '2023-04-05');
   ```
2. Press **Enter**
3. You should see: `INSERT 0 4`

### Step 30: Query the Data
1. Run this query:
   ```sql
   SELECT * FROM employees;
   ```
2. Press **Enter**
3. You should see a table with all 4 employees

### Step 31: Test Reader Endpoint (Read-Only Test)
1. Type `\q` and press **Enter** to quit the current connection
2. Now connect to the **Reader endpoint:**
   ```bash
   psql -h prod-aurora-cluster.cluster-ro-xxxxx.us-east-1.rds.amazonaws.com -U postgres -d proddb
   ```
3. Enter password when prompted

### Step 32: Verify Read-Only Behavior
1. Try to read data (should work):
   ```sql
   SELECT * FROM employees;
   ```
   ✅ This works → You see the data

2. Try to write data (should fail):
   ```sql
   INSERT INTO employees (name, department, salary, hire_date) VALUES ('Test User', 'IT', 50000.00, '2024-01-01');
   ```
   ❌ This might work! (Aurora readers can accept writes, they're just sent to writer)
   
   Note: Aurora automatically routes write operations to the writer endpoint

3. Exit the connection: `\q`

---

## PART 3: Backups & Point-in-Time Recovery

### Step 33: Take Manual Snapshot
1. Go back to **AWS RDS Console** in your browser
2. Navigate to **"Databases"**
3. Click on your cluster name: **"prod-aurora-cluster"**
4. Click the **"Actions"** dropdown button (top right)
5. Select **"Take snapshot"**

### Step 34: Configure Snapshot
1. **Snapshot name:** Type: `prod-aurora-manual-snapshot-1`
2. Click **"Take snapshot"** button
3. You'll be redirected to **"Snapshots"** page
4. **Status** will show:
   - **"Creating"** → Wait ~2-3 minutes
   - **"Available"** → Snapshot ready

### Step 35: Note Current Timestamp (Before Disaster Simulation)
1. Open a text editor on your computer
2. Note the current date and time exactly:
   - Example: `2024-06-24 10:30:00 UTC`
3. Save this timestamp (you'll need it for recovery)

### Step 36: Simulate Disaster - Delete Data
1. Connect to your database again via psql:
   ```bash
   psql -h <WRITER-ENDPOINT> -U postgres -d proddb
   ```
2. At the `proddb=>` prompt, delete some data:
   ```sql
   DELETE FROM employees WHERE name = 'John Doe';
   ```
3. Verify deletion:
   ```sql
   SELECT * FROM employees;
   ```
   You should see only 3 employees now (John Doe is gone)
4. Exit: `\q`

### Step 37: Perform Point-in-Time Recovery
1. Back in **RDS Console**, go to **"Databases"**
2. Click on your cluster: **"prod-aurora-cluster"**
3. Click **"Actions"** dropdown
4. Select **"Restore to point in time"**

### Step 38: Configure Restore Settings
1. **Restore time:**
   - Select **"Custom date and time"**
   - Enter the timestamp you noted in Step 35 (before deletion)
   - Or select **"Latest restorable time"**
2. **DB instance class:** Select **"db.r6g.xlarge"** (same as original)
3. **New DB cluster identifier:** Type: `prod-aurora-cluster-restored`
4. Leave other settings as default
5. Click **"Restore to point in time"** button

### Step 39: Monitor Restoration
1. You'll see the new cluster being created: **"prod-aurora-cluster-restored"**
2. **Status** will show:
   - **"Creating"** → Takes ~10-15 minutes
   - **"Available"** → Ready to use
3. Wait for status to become **"Available"**

### Step 40: Verify Data Recovery
1. Click on the restored cluster: **"prod-aurora-cluster-restored"**
2. Go to **"Connectivity & security"** tab
3. Copy the new **Writer endpoint**
4. Connect via psql:
   ```bash
   psql -h prod-aurora-cluster-restored.cluster-xxxxx.us-east-1.rds.amazonaws.com -U postgres -d proddb
   ```
5. Query the data:
   ```sql
   SELECT * FROM employees;
   ```
6. ✅ **Verify:** John Doe should be back (all 4 employees present)!
7. Exit: `\q`

---

## PART 4: Performance Monitoring with Performance Insights

### Step 41: Generate Load on Database
1. Connect to your **original** cluster (not restored):
   ```bash
   psql -h <ORIGINAL-WRITER-ENDPOINT> -U postgres -d proddb
   ```

2. Run this query multiple times to generate load (copy-paste 10 times):
   ```sql
   INSERT INTO employees (name, department, salary, hire_date) 
   SELECT 
     'Employee ' || generate_series(1, 1000),
     CASE (random() * 3)::int 
       WHEN 0 THEN 'Engineering'
       WHEN 1 THEN 'Marketing'
       WHEN 2 THEN 'Sales'
       ELSE 'HR'
     END,
     50000 + (random() * 50000),
     CURRENT_DATE - (random() * 365)::int;
   ```
   This inserts 1,000 employees each time

3. Run some heavy queries:
   ```sql
   SELECT department, AVG(salary), COUNT(*) 
   FROM employees 
   GROUP BY department 
   ORDER BY AVG(salary) DESC;
   ```

4. Keep the connection open for 2-3 minutes

### Step 42: Access Performance Insights Dashboard
1. In **RDS Console**, click on your cluster: **"prod-aurora-cluster"**
2. Click on the **"Monitoring"** tab (below the cluster name)
3. Scroll down to find **"Performance Insights"** section
4. Click **"View on Performance Insights"** button (or the metrics graph)

### Step 43: Explore Performance Insights Dashboard
You'll see a comprehensive dashboard with:

**Top Section - Database Load Chart:**
1. Look at the **"Database load"** graph (colored stacked area)
2. **X-axis:** Time (last hour by default)
3. **Y-axis:** Average Active Sessions (AAS)
4. **Colored areas:** Different wait events
   - **Green:** CPU usage
   - **Blue:** I/O waits
   - **Other colors:** Lock waits, network, etc.

**Key Metrics to Observe:**
- If load is **above the line** labeled "Max vCPU" → Database is overloaded
- If load is **below the line** → Database is healthy

### Step 44: Analyze Top SQL Queries
1. Scroll down to **"Top SQL"** section (below the load chart)
2. You'll see a table with columns:
   - **SQL:** The actual query (truncated)
   - **Load by waits (AAS):** How much load this query creates
   - **Executions:** How many times it ran
   - **Rows:** Number of rows affected
3. Click on any SQL query to see full text
4. **What to look for:**
   - Queries with high **Load by waits** → Need optimization
   - Queries with many **Executions** → Frequent operations
   - Missing indexes indicated by high I/O waits

### Step 45: Analyze Wait Events
1. In the dashboard, look at **"Top wait events"** section
2. Common wait events you might see:
   - **CPU:** Good (means database is actively processing)
   - **IO:DataFileRead:** Disk I/O (might need more memory or indexes)
   - **Lock:tuple:** Row-level locks (concurrent updates)
   - **Client:ClientRead:** Waiting for client to fetch results
3. Click on any wait event to see which queries are causing it

### Step 46: Adjust Time Range
1. At the top of Performance Insights, you'll see time range selector
2. Click the dropdown (default: **"1h"**)
3. Select different ranges:
   - **5 minutes** → Real-time monitoring
   - **1 hour** → Recent trends
   - **1 day** → Daily patterns
4. Use the **slider** below the chart to zoom into specific time periods

### Step 47: Filter by Database
1. At the top, next to "Dimension", click **"Filter by"**
2. Select **"Database"**
3. Choose **"proddb"** to see only traffic to this database
4. Remove filter by clicking the **X**

---

## PART 5: Configure CloudWatch Alarms

### Step 48: Navigate to CloudWatch
1. In AWS Console top search bar, type **"CloudWatch"**
2. Click on **"CloudWatch"** to open the service
3. In the left sidebar, click **"Alarms"** (under "Alarms" section)
4. Click **"All alarms"**

### Step 49: Create High CPU Alarm
1. Click **"Create alarm"** button (orange)
2. Click **"Select metric"** button

### Step 50: Select RDS CPU Metric
1. You'll see metric categories, click **"RDS"** tile
2. Click **"By Database Instance"** (not "By Cluster")
3. In the search box, type: `prod-aurora-cluster`
4. You'll see rows for both instances (writer and reader)
5. Find the row with:
   - **DBInstanceIdentifier:** `prod-aurora-cluster-instance-1` (writer)
   - **Metric Name:** `CPUUtilization`
6. ✅ Check the checkbox for this row
7. Click **"Select metric"** button (bottom right)

### Step 51: Configure CPU Alarm Threshold
1. **Metric name:** Should show `CPUUtilization`
2. **Statistic:** Select **"Average"** from dropdown
3. **Period:** Select **"5 minutes"**
4. Scroll to **"Conditions"** section:
   - **Threshold type:** Select **"Static"**
   - **Whenever CPUUtilization is...:** Select **"Greater"**
   - **than...:** Type: `80`
   - This means: Alert if CPU > 80% for 5 minutes
5. Click **"Next"** button (bottom right)

### Step 52: Configure SNS Topic for Notifications
1. Under **"Notification"** section:
   - **Alarm state trigger:** Select **"In alarm"**
   - **Send notification to the following SNS topic:** Select **"Create new topic"**
2. **Create a new topic:**
   - **Create a new topic...:** Leave selected
   - **Topic name:** Type: `aurora-alarms`
   - **Email endpoints:** Type your email address (e.g., `your.email@example.com`)
3. Click **"Create topic"** button
4. Click **"Next"** button

### Step 53: Complete CPU Alarm Creation
1. **Alarm name:** Type: `aurora-high-cpu-alert`
2. **Alarm description:** Type: `Alert when Aurora CPU exceeds 80%`
3. Click **"Next"** button
4. Review the alarm configuration
5. Click **"Create alarm"** button

### Step 54: Confirm SNS Subscription
1. **CHECK YOUR EMAIL** (the one you entered in Step 52)
2. You'll receive email: **"AWS Notification - Subscription Confirmation"**
3. Click the **"Confirm subscription"** link in the email
4. You'll see a browser page: **"Subscription confirmed!"**

### Step 55: Create Memory Alarm
1. Back in CloudWatch, click **"Create alarm"** again
2. Click **"Select metric"**
3. Navigate: **RDS** → **By Database Instance**
4. Search for your instance: `prod-aurora-cluster-instance-1`
5. Find and select **"FreeableMemory"** metric
6. Click **"Select metric"**

### Step 56: Configure Memory Alarm Threshold
1. **Statistic:** **"Average"**
2. **Period:** **"5 minutes"**
3. **Conditions:**
   - **Threshold type:** **"Static"**
   - **Whenever FreeableMemory is...:** **"Lower"**
   - **than...:** Type: `2000000000` (2 GB in bytes)
4. Click **"Next"**

### Step 57: Configure Notification
1. **Select an existing SNS topic:** Choose **"aurora-alarms"** (created earlier)
2. Click **"Next"**
3. **Alarm name:** Type: `aurora-low-memory-alert`
4. **Description:** Type: `Alert when free memory drops below 2GB`
5. Click **"Next"** → **"Create alarm"**

### Step 58: Create Replica Lag Alarm
1. Click **"Create alarm"**
2. **Select metric** → **RDS** → **By Database Instance**
3. Search for your **reader** instance: `prod-aurora-cluster-instance-1-<region>-1`
4. Select metric: **"AuroraReplicaLag"**
5. Click **"Select metric"**

### Step 59: Configure Replica Lag Threshold
1. **Statistic:** **"Average"**
2. **Period:** **"1 minute"**
3. **Conditions:**
   - **Threshold type:** **"Static"**
   - **Whenever AuroraReplicaLag is...:** **"Greater"**
   - **than...:** Type: `1000` (1000 milliseconds = 1 second)
4. Click **"Next"**
5. **SNS topic:** Select **"aurora-alarms"**
6. Click **"Next"**
7. **Alarm name:** `aurora-high-replica-lag`
8. **Description:** `Alert when replica lag exceeds 1 second`
9. **Create alarm**

### Step 60: Create Database Connections Alarm
1. Click **"Create alarm"**
2. **Select metric** → **RDS** → **By Database Instance**
3. Select your writer instance
4. Choose metric: **"DatabaseConnections"**
5. Click **"Select metric"**

### Step 61: Configure Connections Threshold
1. **Statistic:** **"Average"**
2. **Period:** **"5 minutes"**
3. **Conditions:**
   - **Threshold type:** **"Static"**
   - **Whenever DatabaseConnections is...:** **"Greater"**
   - **than...:** Calculate 80% of max connections:
     - For db.r6g.xlarge: Max connections ≈ 3000
     - 80% = 2400
     - Type: `2400`
4. Click **"Next"**
5. **SNS topic:** **"aurora-alarms"**
6. Click **"Next"**
7. **Alarm name:** `aurora-high-connections`
8. **Description:** `Alert when connections exceed 80% of maximum`
9. **Create alarm**

### Step 62: View All Alarms
1. In CloudWatch console, click **"All alarms"** in left sidebar
2. You should now see 4 alarms:
   - ✅ `aurora-high-cpu-alert`
   - ✅ `aurora-low-memory-alert`
   - ✅ `aurora-high-replica-lag`
   - ✅ `aurora-high-connections`
3. All should show **State: OK** (green checkmark)

---

## ✅ Exercise 3.1 Completion Checklist

- [ ] Created Aurora PostgreSQL cluster with Multi-AZ
- [ ] Configured security group for database access
- [ ] Connected via psql and created test data
- [ ] Tested read replica connection
- [ ] Created manual snapshot
- [ ] Performed point-in-time recovery
- [ ] Verified data restoration
- [ ] Explored Performance Insights dashboard
- [ ] Analyzed top SQL queries and wait events
- [ ] Created 4 CloudWatch alarms (CPU, Memory, Replica Lag, Connections)
- [ ] Confirmed SNS email subscription

**🎉 Congratulations!** You've completed Exercise 3.1!

---

# Exercise 3.2: Redshift Data Warehouse with Star Schema

**Duration:** 2-3 hours | **Estimated Cost:** ~$10-15 for lab session

## 🎯 Learning Objectives
- Create Redshift data warehouse cluster
- Design and implement star schema
- Optimize tables with DISTKEY and SORTKEY
- Load data from S3 using COPY command
- Create materialized views for performance
- Run analytical queries

---

## PART 1: Create Redshift Cluster

### Step 1: Navigate to Redshift Console
1. In AWS Management Console, search for **"Redshift"**
2. Click on **"Amazon Redshift"** from results
3. You'll land on the Redshift Dashboard

### Step 2: Initiate Cluster Creation
1. On the left sidebar, click **"Clusters"**
2. Click the **"Create cluster"** button (blue button, top right)

### Step 3: Choose Cluster Configuration Method
1. **Cluster configuration:** Two options appear:
   - **Production** ← Select this
   - **Free trial** (only if eligible and first time)
2. Click **"Production"** radio button

### Step 4: Configure Cluster Identifier and Size
1. **Cluster identifier:** Type: `analytics-cluster`
2. **What are you optimizing for?:**
   - Select **"Balanced"** (mix of compute and storage)
3. **Node type:** Click dropdown
   - Scroll to find **"dc2.large"** (Dense Compute)
   - Select **"dc2.large"** (160 GB SSD, 2 vCPU, 15 GB RAM)
4. **Number of nodes:** 
   - Move slider or type: `2`
   - This gives you 320 GB total storage
5. **Estimated cost:** Should show ~$0.50/hour (~$365/month)

### Step 5: Configure Database Settings
1. **Database name:** Type: `analyticsdb`
2. **Database port:** Leave as `5439` (default Redshift port)
3. **Master user name:** Type: `awsuser`
4. **Master user password:**
   - Select **"Auto generate password"** OR
   - Type a custom password (8+ characters, mix of letters and numbers)
5. If auto-generated, ✅ check: **"Store password in AWS Secrets Manager"**
   - **AWS KMS key:** Select **"(default) aws/secretsmanager"**

### Step 6: Create IAM Role for S3 Access
1. Scroll to **"Cluster permissions"** section
2. Click **"Manage IAM roles"**
3. Click **"Create IAM role"** button
4. **IAM role name:** Leave as auto-generated or type: `RedshiftS3ReadRole`
5. Under **"S3 buckets you can access":**
   - Select **"Any S3 bucket"** OR
   - Select **"Specific S3 buckets"** and enter bucket names
6. Click **"Create IAM role as default"**
7. You'll see the role appear in the list with **"default"** tag

### Step 7: Configure Network and Security
1. **Virtual private cloud (VPC):** Select your default VPC
2. **VPC security groups:**
   - Click **"Create new"** radio button
   - **New VPC security group name:** Type: `redshift-sg`
3. **Cluster subnet group:** Leave as **"default"**
4. **Publicly accessible:** Select **"No"** (recommended for production)
5. **Availability Zone:** Select **"No preference"**
6. **Enhanced VPC routing:** Leave unchecked (for simplicity)

### Step 8: Configure Encryption
1. Scroll to **"Database configurations"** section
2. **Encryption:**
   - Select **"Use AWS Key Management Service (AWS KMS)"**
   - **AWS KMS key:** Select **"(default) aws/redshift"** OR choose custom key
3. This encrypts data at rest

### Step 9: Configure Backup and Maintenance
1. **Automated snapshot retention period:**
   - Move slider or type: `7` days
2. **Manual snapshot retention period:**
   - Leave as **"Indefinite"**
3. **Maintenance window:**
   - Select **"Use custom window"**
   - **Day:** **"Sunday"**
   - **Start time:** **"03:00"** (3 AM)
   - **Duration:** **"1 hour"**

### Step 10: Review and Create Cluster
1. Scroll to bottom of page
2. Review all settings in the summary on the right
3. **Estimated cost:** Verify it shows ~$365/month
4. Click **"Create cluster"** button (orange)

### Step 11: Save Credentials
1. If you selected **"Auto generate password"**, a popup appears
2. Click **"View credentials"** OR **"Download credentials"**
3. **IMPORTANT:** Save these:
   - Master username: `awsuser`
   - Master password: `[random-generated-password]`
   - JDBC URL (for later)
4. Click **"Close"**

### Step 12: Monitor Cluster Creation
1. You're on the **"Clusters"** page
2. Find your cluster: **"analytics-cluster"**
3. **Status** column will show:
   - **"Creating"** → Takes ~5-10 minutes
   - **"Available"** → Ready to use
4. Wait for status to become **"Available"**

### Step 13: Retrieve Cluster Endpoint
1. Once **"Available"**, click on cluster name: **"analytics-cluster"**
2. In the cluster details page, go to **"General information"** section
3. Find and copy:
   - **Endpoint:** `analytics-cluster.xxxxxx.us-east-1.redshift.amazonaws.com:5439`
   - **JDBC URL:** `jdbc:redshift://analytics-cluster.xxxxxx.us-east-1.redshift.amazonaws.com:5439/analyticsdb`
4. Save these to a text file

---

## PART 2: Configure Security & Connect to Redshift

### Step 14: Configure Security Group
1. In the cluster details, scroll to **"Cluster properties"** section
2. Under **"Network and security settings"**, find **"VPC security groups"**
3. Click on the security group link: **"redshift-sg"**
4. This opens EC2 Security Groups console
5. Select the **"redshift-sg"** security group
6. Click **"Inbound rules"** tab
7. Click **"Edit inbound rules"**

### Step 15: Add Redshift Access Rule
1. Click **"Add rule"**
2. Configure:
   - **Type:** Select **"Redshift"** (automatically sets port 5439)
   - **Source:** Select **"My IP"** (for testing)
     - Or select **"Custom"** and enter your VPC CIDR
   - **Description:** Type: `Allow Redshift access for analytics`
3. Click **"Save rules"**

### Step 16: Connect Using psql (Local Computer)
1. Open **Terminal** (macOS/Linux) or **Command Prompt** (Windows)
2. Ensure PostgreSQL client is installed (from Exercise 3.1, Step 25)
3. Connect using this command template:
   ```bash
   psql -h <REDSHIFT-ENDPOINT> -U awsuser -d analyticsdb -p 5439
   ```
4. Replace `<REDSHIFT-ENDPOINT>` with your actual endpoint (without `:5439`)
5. Example:
   ```bash
   psql -h analytics-cluster.cxxxxx.us-east-1.redshift.amazonaws.com -U awsuser -d analyticsdb -p 5439
   ```
6. Press **Enter**
7. Enter your master password when prompted

### Step 17: Verify Connection
At the `analyticsdb=>` prompt:

1. **Check Redshift version:**
   ```sql
   SELECT version();
   ```
   You should see something like: `PostgreSQL 8.0.2 on i686... Redshift 1.0.xx`

2. **Check current database:**
   ```sql
   SELECT current_database();
   ```
   Should show: `analyticsdb`

---

## PART 3: Create Star Schema for Analytics

### Step 18: Understand Star Schema Design
Before creating tables, understand the structure:

**Star Schema Components:**
- **Fact Table** (center): `fact_transactions` - stores business events (sales, orders)
- **Dimension Tables** (points of star): Descriptive attributes
  - `dim_customers` - customer information
  - `dim_products` - product catalog
  - `dim_date` - calendar date attributes

**Redshift Optimization:**
- **DISTKEY:** How data is distributed across nodes
- **SORTKEY:** How data is sorted within each node
- **DISTSTYLE:** Distribution strategy (KEY, ALL, EVEN)

### Step 19: Create Dimension Table - Customers
At the `analyticsdb=>` prompt, copy and paste:

```sql
CREATE TABLE dim_customers (
  customer_key INTEGER NOT NULL,
  customer_id VARCHAR(50),
  customer_name VARCHAR(100),
  email VARCHAR(100),
  city VARCHAR(50),
  state VARCHAR(50),
  country VARCHAR(50),
  signup_date DATE,
  PRIMARY KEY (customer_key)
)
DISTSTYLE ALL;
```

**Press Enter**

**What this means:**
- `DISTSTYLE ALL` = Copy full table to ALL nodes (good for small dimension tables)
- This allows local JOINs without network transfer

You should see: `CREATE TABLE`

### Step 20: Create Dimension Table - Products
```sql
CREATE TABLE dim_products (
  product_key INTEGER NOT NULL,
  product_id VARCHAR(50),
  product_name VARCHAR(100),
  category VARCHAR(50),
  subcategory VARCHAR(50),
  brand VARCHAR(50),
  unit_price DECIMAL(10,2),
  PRIMARY KEY (product_key)
)
DISTSTYLE ALL;
```

**Press Enter** → You see: `CREATE TABLE`

### Step 21: Create Dimension Table - Date
```sql
CREATE TABLE dim_date (
  date_key INTEGER NOT NULL,
  full_date DATE,
  year INTEGER,
  quarter INTEGER,
  month INTEGER,
  month_name VARCHAR(20),
  week INTEGER,
  day_of_month INTEGER,
  day_of_week INTEGER,
  day_name VARCHAR(20),
  is_weekend BOOLEAN,
  PRIMARY KEY (date_key)
)
DISTSTYLE ALL;
```

**Press Enter** → `CREATE TABLE`

### Step 22: Generate Date Dimension Data
Now populate the date dimension with 3 years of data:

```sql
INSERT INTO dim_date
SELECT
  TO_CHAR(d, 'YYYYMMDD')::INTEGER AS date_key,
  d AS full_date,
  EXTRACT(YEAR FROM d) AS year,
  EXTRACT(QUARTER FROM d) AS quarter,
  EXTRACT(MONTH FROM d) AS month,
  TO_CHAR(d, 'Month') AS month_name,
  EXTRACT(WEEK FROM d) AS week,
  EXTRACT(DAY FROM d) AS day_of_month,
  EXTRACT(DOW FROM d) AS day_of_week,
  TO_CHAR(d, 'Day') AS day_name,
  CASE WHEN EXTRACT(DOW FROM d) IN (0, 6) THEN TRUE ELSE FALSE END AS is_weekend
FROM (
  SELECT '2023-01-01'::DATE + (seq - 1) AS d
  FROM (SELECT ROW_NUMBER() OVER () AS seq FROM dim_customers LIMIT 1095)
) dates;
```

**Note:** This might fail if dim_customers is empty. If so, use this simpler version:

```sql
INSERT INTO dim_date
WITH date_series AS (
  SELECT ('2023-01-01'::DATE + INTERVAL '1 day' * (ROW_NUMBER() OVER () - 1)) AS d
  FROM (SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5) a,
       (SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5) b,
       (SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5) c
  LIMIT 1095
)
SELECT
  TO_CHAR(d, 'YYYYMMDD')::INTEGER,
  d::DATE,
  EXTRACT(YEAR FROM d)::INTEGER,
  EXTRACT(QUARTER FROM d)::INTEGER,
  EXTRACT(MONTH FROM d)::INTEGER,
  TO_CHAR(d, 'Month'),
  EXTRACT(WEEK FROM d)::INTEGER,
  EXTRACT(DAY FROM d)::INTEGER,
  EXTRACT(DOW FROM d)::INTEGER,
  TO_CHAR(d, 'Day'),
  CASE WHEN EXTRACT(DOW FROM d) IN (0, 6) THEN TRUE ELSE FALSE END
FROM date_series;
```

**Press Enter** → `INSERT 0 1095` (3 years × 365 days)

### Step 23: Verify Date Dimension
```sql
SELECT * FROM dim_date LIMIT 10;
```

You should see 10 rows with date information.

### Step 24: Create Fact Table - Transactions
This is the main table with **optimized distribution**:

```sql
CREATE TABLE fact_transactions (
  transaction_id BIGINT NOT NULL,
  customer_key INTEGER NOT NULL,
  product_key INTEGER NOT NULL,
  date_key INTEGER NOT NULL,
  quantity INTEGER,
  unit_price DECIMAL(10,2),
  discount_percent DECIMAL(5,2),
  total_amount DECIMAL(12,2),
  PRIMARY KEY (transaction_id)
)
DISTKEY(customer_key)
SORTKEY(date_key);
```

**Press Enter** → `CREATE TABLE`

**What this means:**
- `DISTKEY(customer_key)` = Distribute rows across nodes based on customer_key
  - All transactions for same customer go to same node
  - Optimizes JOINs with dim_customers
- `SORTKEY(date_key)` = Sort data by date within each node
  - Optimizes time-based queries (daily/monthly reports)

### Step 25: Verify Table Structure
Check how tables are distributed:

```sql
SELECT 
  "table" AS table_name,
  size AS size_mb,
  diststyle,
  sortkey1
FROM svv_table_info
WHERE "table" IN ('dim_customers', 'dim_products', 'dim_date', 'fact_transactions')
ORDER BY "table";
```

**Press Enter**

You should see:
| table_name | size_mb | diststyle | sortkey1 |
|------------|---------|-----------|----------|
| dim_customers | 0 | ALL | |
| dim_date | 0 | ALL | |
| dim_products | 0 | ALL | |
| fact_transactions | 0 | KEY(customer_key) | date_key |

---

## PART 4: Load Sample Data

### Step 26: Insert Sample Customer Data
```sql
INSERT INTO dim_customers VALUES
(1, 'CUST001', 'John Smith', 'john.smith@email.com', 'New York', 'NY', 'USA', '2022-01-15'),
(2, 'CUST002', 'Jane Doe', 'jane.doe@email.com', 'Los Angeles', 'CA', 'USA', '2022-02-20'),
(3, 'CUST003', 'Bob Johnson', 'bob.j@email.com', 'Chicago', 'IL', 'USA', '2022-03-10'),
(4, 'CUST004', 'Alice Williams', 'alice.w@email.com', 'Houston', 'TX', 'USA', '2022-04-05'),
(5, 'CUST005', 'Charlie Brown', 'charlie.b@email.com', 'Phoenix', 'AZ', 'USA', '2022-05-12'),
(6, 'CUST006', 'Diana Prince', 'diana.p@email.com', 'Philadelphia', 'PA', 'USA', '2022-06-18'),
(7, 'CUST007', 'Edward Norton', 'edward.n@email.com', 'San Antonio', 'TX', 'USA', '2022-07-22'),
(8, 'CUST008', 'Fiona Green', 'fiona.g@email.com', 'San Diego', 'CA', 'USA', '2022-08-30'),
(9, 'CUST009', 'George Miller', 'george.m@email.com', 'Dallas', 'TX', 'USA', '2022-09-14'),
(10, 'CUST010', 'Helen Troy', 'helen.t@email.com', 'San Jose', 'CA', 'USA', '2022-10-25');
```

**Press Enter** → `INSERT 0 10`

### Step 27: Insert Sample Product Data
```sql
INSERT INTO dim_products VALUES
(1, 'PROD001', 'Laptop Pro 15', 'Electronics', 'Computers', 'TechBrand', 1299.99),
(2, 'PROD002', 'Wireless Mouse', 'Electronics', 'Accessories', 'TechBrand', 29.99),
(3, 'PROD003', 'Office Chair', 'Furniture', 'Seating', 'ComfortCo', 199.99),
(4, 'PROD004', 'Desk Lamp', 'Furniture', 'Lighting', 'BrightLight', 49.99),
(5, 'PROD005', 'Notebook Pack', 'Stationery', 'Paper', 'WriteCo', 9.99),
(6, 'PROD006', 'Pen Set', 'Stationery', 'Writing', 'WriteCo', 14.99),
(7, 'PROD007', 'Monitor 27"', 'Electronics', 'Displays', 'ViewTech', 349.99),
(8, 'PROD008', 'Keyboard Mechanical', 'Electronics', 'Accessories', 'TechBrand', 89.99),
(9, 'PROD009', 'Standing Desk', 'Furniture', 'Desks', 'ComfortCo', 499.99),
(10, 'PROD010', 'Headphones', 'Electronics', 'Audio', 'SoundMax', 149.99);
```

**Press Enter** → `INSERT 0 10`

### Step 28: Generate Large Transaction Dataset
Create 10,000 transactions with random data:

```sql
INSERT INTO fact_transactions
SELECT
  ROW_NUMBER() OVER () AS transaction_id,
  (RANDOM() * 9 + 1)::INTEGER AS customer_key,
  (RANDOM() * 9 + 1)::INTEGER AS product_key,
  TO_CHAR(
    '2023-01-01'::DATE + (RANDOM() * 730)::INTEGER,
    'YYYYMMDD'
  )::INTEGER AS date_key,
  (RANDOM() * 10 + 1)::INTEGER AS quantity,
  p.unit_price,
  (RANDOM() * 20)::DECIMAL(5,2) AS discount_percent,
  ((RANDOM() * 10 + 1)::INTEGER * p.unit_price * (1 - (RANDOM() * 0.2)))::DECIMAL(12,2) AS total_amount
FROM
  (SELECT unit_price FROM dim_products ORDER BY RANDOM() LIMIT 10000) p;
```

**Press Enter** → `INSERT 0 10000`

**Wait 10-30 seconds** for generation to complete.

### Step 29: Run ANALYZE for Query Optimization
After loading data, update statistics:

```sql
ANALYZE dim_customers;
ANALYZE dim_products;
ANALYZE dim_date;
ANALYZE fact_transactions;
```

**Press Enter after each** → You see: `ANALYZE`

**Why ANALYZE?** Redshift uses statistics to create optimal query plans.

### Step 30: Verify Data Load
```sql
SELECT 
  'Customers' AS table_name, COUNT(*) AS row_count FROM dim_customers
UNION ALL
SELECT 
  'Products', COUNT(*) FROM dim_products
UNION ALL
SELECT 
  'Dates', COUNT(*) FROM dim_date
UNION ALL
SELECT 
  'Transactions', COUNT(*) FROM fact_transactions;
```

**Press Enter**

You should see:
| table_name | row_count |
|------------|-----------|
| Customers | 10 |
| Products | 10 |
| Dates | 1095 |
| Transactions | 10000 |

---

## PART 5: Run Analytical Queries

### Step 31: Query - Daily Sales Summary
```sql
SELECT
  d.full_date,
  d.day_name,
  COUNT(DISTINCT f.transaction_id) AS total_transactions,
  SUM(f.quantity) AS total_items_sold,
  SUM(f.total_amount) AS total_revenue,
  AVG(f.total_amount) AS avg_transaction_value
FROM fact_transactions f
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year = 2023
  AND d.month = 6
GROUP BY d.full_date, d.day_name
ORDER BY d.full_date;
```

**Press Enter**

You'll see daily sales for June 2023 with metrics.

### Step 32: Query - Top 10 Customers by Revenue
```sql
SELECT
  c.customer_name,
  c.city,
  c.state,
  COUNT(f.transaction_id) AS total_orders,
  SUM(f.total_amount) AS total_spent,
  AVG(f.total_amount) AS avg_order_value
FROM fact_transactions f
JOIN dim_customers c ON f.customer_key = c.customer_key
GROUP BY c.customer_name, c.city, c.state
ORDER BY total_spent DESC
LIMIT 10;
```

**Press Enter**

Shows top spending customers.

### Step 33: Query - Product Category Performance
```sql
SELECT
  p.category,
  p.subcategory,
  COUNT(DISTINCT f.transaction_id) AS transactions,
  SUM(f.quantity) AS units_sold,
  SUM(f.total_amount) AS revenue,
  AVG(f.discount_percent) AS avg_discount
FROM fact_transactions f
JOIN dim_products p ON f.product_key = p.product_key
GROUP BY p.category, p.subcategory
ORDER BY revenue DESC;
```

**Press Enter**

Shows which product categories generate most revenue.

### Step 34: Query - Monthly Trend Analysis
```sql
SELECT
  d.year,
  d.month,
  d.month_name,
  COUNT(DISTINCT f.transaction_id) AS transactions,
  SUM(f.total_amount) AS revenue,
  SUM(f.total_amount) - LAG(SUM(f.total_amount)) OVER (ORDER BY d.year, d.month) AS revenue_change,
  ROUND(
    (SUM(f.total_amount) - LAG(SUM(f.total_amount)) OVER (ORDER BY d.year, d.month)) / 
    LAG(SUM(f.total_amount)) OVER (ORDER BY d.year, d.month) * 100,
    2
  ) AS revenue_change_pct
FROM fact_transactions f
JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.year, d.month, d.month_name
ORDER BY d.year, d.month;
```

**Press Enter**

Shows month-over-month revenue trends with percentage change.

### Step 35: Create Materialized View
Create a pre-computed aggregation for faster queries:

```sql
CREATE MATERIALIZED VIEW mv_daily_sales AS
SELECT
  f.date_key,
  d.full_date,
  d.day_name,
  d.is_weekend,
  COUNT(DISTINCT f.transaction_id) AS transaction_count,
  COUNT(DISTINCT f.customer_key) AS unique_customers,
  SUM(f.quantity) AS total_quantity,
  SUM(f.total_amount) AS total_revenue,
  AVG(f.total_amount) AS avg_transaction_value
FROM fact_transactions f
JOIN dim_date d ON f.date_key = d.date_key
GROUP BY f.date_key, d.full_date, d.day_name, d.is_weekend;
```

**Press Enter** → `SELECT 730` (or similar)

**What is Materialized View?**
- Pre-computed query results stored as a table
- Much faster than running the query each time
- Need to refresh when source data changes

### Step 36: Query Materialized View
```sql
SELECT
  full_date,
  day_name,
  transaction_count,
  unique_customers,
  total_revenue
FROM mv_daily_sales
WHERE full_date BETWEEN '2023-01-01' AND '2023-01-31'
ORDER BY full_date;
```

**Press Enter**

This query runs **much faster** than joining fact and dimension tables!

### Step 37: Refresh Materialized View
When you add new transaction data:

```sql
REFRESH MATERIALIZED VIEW mv_daily_sales;
```

**Press Enter** → `REFRESH MATERIALIZED VIEW`

---

## ✅ Exercise 3.2 Completion Checklist

- [ ] Created Redshift cluster (2-node, dc2.large)
- [ ] Configured security group for access
- [ ] Connected via psql
- [ ] Created star schema (3 dimension tables + 1 fact table)
- [ ] Optimized with DISTKEY and SORTKEY
- [ ] Generated date dimension (3 years)
- [ ] Loaded sample customer and product data
- [ ] Generated 10,000 transactions
- [ ] Ran ANALYZE on all tables
- [ ] Executed 4 analytical queries
- [ ] Created materialized view for performance
- [ ] Tested materialized view queries

**🎉 Congratulations!** You've completed Exercise 3.2!

---

# Exercise 3.3: DynamoDB for High-Velocity IoT Data

**Duration:** 1-2 hours | **Estimated Cost:** ~$2-5 for lab session

## 🎯 Learning Objectives
- Create DynamoDB table with partition key and sort key
- Understand on-demand vs. provisioned capacity
- Insert IoT event data
- Query data efficiently with partition key
- Use DynamoDB console for data exploration

---

## PART 1: Create DynamoDB Table

### Step 1: Navigate to DynamoDB Console
1. In AWS Management Console, search for **"DynamoDB"**
2. Click on **"DynamoDB"** from results
3. You'll land on DynamoDB Dashboard

### Step 2: Initiate Table Creation
1. In the left sidebar, click **"Tables"**
2. Click **"Create table"** button (orange, top right)

### Step 3: Configure Table Settings
1. **Table name:** Type: `iot_events`
2. **Partition key:**
   - Field name: Type: `device_id`
   - Type: Select **"String"** from dropdown
3. **Sort key:**
   - ✅ Check the checkbox: **"Add sort key"**
   - Field name: Type: `timestamp`
   - Type: Select **"Number"** from dropdown

**Why this design?**
- **Partition key (`device_id`):** Groups all events from same device together
- **Sort key (`timestamp`):** Orders events chronologically within each device
- Enables efficient queries like: "Get all events for DEVICE_001 between time X and Y"

### Step 4: Choose Table Class
1. **Table class:** Leave as **"DynamoDB Standard"** (selected by default)
   - Standard: Best for frequently accessed data
   - Standard-IA: Cheaper for infrequently accessed data

### Step 5: Select Capacity Mode
1. You'll see two capacity modes:

**Option A: On-Demand (Recommended for this lab)**
- Select **"On-demand"** radio button
- **Pros:** Auto-scales, pay only for requests, no capacity planning
- **Best for:** Unpredictable workloads, new applications
- **Pricing:** $1.25 per million write requests, $0.25 per million read requests

**Option B: Provisioned**
- Select **"Provisioned"** radio button
- **Read capacity units:** Type: `5` (1 unit = 4 KB/sec)
- **Write capacity units:** Type: `5` (1 unit = 1 KB/sec)
- ✅ Check: **"Use auto scaling"** (adjusts capacity automatically)
- **Best for:** Predictable traffic patterns

**For this lab:** Choose **On-demand** (simpler, no capacity planning needed)

### Step 6: Configure Encryption
1. Scroll to **"Encryption at rest"** section
2. **Encryption type:** Leave as **"Owned by Amazon DynamoDB"** (selected by default)
   - This is free and provides encryption
   - Alternatives: **AWS managed key** (aws/dynamodb) or **Customer managed key** (your KMS key)

### Step 7: Create Table
1. Scroll to the bottom
2. Review settings in the summary on the right
3. Click **"Create table"** button (orange)

### Step 8: Monitor Table Creation
1. You'll see your table: **"iot_events"** in the list
2. **Status** column will show:
   - **"Creating"** → Takes ~30-60 seconds
   - **"Active"** → Ready to use
3. Wait for status to become **"Active"**

---

## PART 2: Insert IoT Events via Console

### Step 9: Access Table Items
1. Click on your table name: **"iot_events"**
2. In the table details page, click **"Explore table items"** button (top right)
   - OR click **"Actions"** dropdown → **"Explore table items"**

### Step 10: Create First Item (Event)
1. Click **"Create item"** button
2. You'll see a form with your keys:
   - **device_id** (String)
   - **timestamp** (Number)

### Step 11: Fill in Event Data
1. **device_id (String):**
   - Click in the value field
   - Type: `DEVICE_001`

2. **timestamp (Number):**
   - Type: `1688123400` (Unix timestamp: 2023-06-30 12:30:00 UTC)

3. **Add more attributes** by clicking **"Add new attribute"** button:

   **a) Add temperature:**
   - Click **"Add new attribute"** dropdown
   - Select **"Number"**
   - **Attribute name:** Type: `temperature`
   - **Value:** Type: `25.5`

   **b) Add humidity:**
   - Click **"Add new attribute"** → **"Number"**
   - **Attribute name:** `humidity`
   - **Value:** `60.2`

   **c) Add status:**
   - Click **"Add new attribute"** → **"String"**
   - **Attribute name:** `status`
   - **Value:** `NORMAL`

   **d) Add battery_level:**
   - Click **"Add new attribute"** → **"Number"**
   - **Attribute name:** `battery_level`
   - **Value:** `85`

4. Your item should now have:
   - ✅ device_id: DEVICE_001
   - ✅ timestamp: 1688123400
   - ✅ temperature: 25.5
   - ✅ humidity: 60.2
   - ✅ status: NORMAL
   - ✅ battery_level: 85

5. Click **"Create item"** button (bottom right)

### Step 12: Create More Events
Repeat Step 10-11 to create 4 more items with these values:

**Item 2:**
- device_id: `DEVICE_001`
- timestamp: `1688123460` (1 minute later)
- temperature: `26.1`
- humidity: `61.5`
- status: `NORMAL`
- battery_level: `84`

**Item 3:**
- device_id: `DEVICE_001`
- timestamp: `1688123520` (2 minutes later)
- temperature: `28.3`
- humidity: `58.7`
- status: `WARNING`
- battery_level: `84`

**Item 4:**
- device_id: `DEVICE_002`
- timestamp: `1688123400`
- temperature: `22.0`
- humidity: `55.0`
- status: `NORMAL`
- battery_level: `92`

**Item 5:**
- device_id: `DEVICE_002`
- timestamp: `1688123460`
- temperature: `22.5`
- humidity: `56.2`
- status: `NORMAL`
- battery_level: `92`

### Step 13: View All Items
1. After creating all items, you'll see them in the **"Items returned"** section
2. **Items returned:** Should show `5`
3. You can see columns: device_id, timestamp, temperature, humidity, status, battery_level

---

## PART 3: Query Data Using Console

### Step 14: Query Events for Specific Device
1. At the top of the "Items" view, you'll see **"Scan/Query items"** section
2. Click the dropdown that says **"Scan"**
3. Change to **"Query"** (this is more efficient for our use case)

### Step 15: Build Query
1. **Partition key:**
   - Field: `device_id` (auto-filled)
   - Type: `String` (auto-filled)
   - **Value:** Type: `DEVICE_001`

2. **Sort key (optional):**
   - Leave empty for now (this gets all timestamps)

3. Click **"Run"** button

### Step 16: View Query Results
1. **Items returned:** Should show `3` (all events from DEVICE_001)
2. You'll see the 3 items we created for DEVICE_001
3. Notice they're sorted by timestamp (ascending by default)

### Step 17: Query with Sort Key Condition
1. Now let's query for a specific time range
2. Keep **Partition key** as: `device_id = DEVICE_001`
3. **Sort key:**
   - ✅ Check the checkbox next to **"timestamp"**
   - **Condition:** Select **"Between"** from dropdown
   - **Value 1:** Type: `1688123400`
   - **Value 2:** Type: `1688123480`
4. Click **"Run"**

### Step 18: Analyze Results
1. **Items returned:** Should show `2`
2. Only events between the two timestamps are shown
3. This demonstrates efficient time-range querying!

### Step 19: Scan All Items (Compare with Query)
1. Change dropdown from **"Query"** to **"Scan"**
2. Click **"Run"**
3. **Items returned:** Shows `5` (all items across all devices)
4. **Important:** Scan reads ALL data (expensive), Query reads only partition (cheap)

**Rule of Thumb:**
- **Use Query** when you know the partition key → Fast & cheap
- **Use Scan** only when you need all data → Slow & expensive

---

## PART 4: Using Python SDK (Optional - Advanced)

### Step 20: Install Boto3 (AWS Python SDK)
If you want to programmatically interact with DynamoDB:

1. Open **Terminal** (macOS/Linux) or **Command Prompt** (Windows)
2. Install boto3:
   ```bash
   pip install boto3
   ```
   OR if you have Python 3:
   ```bash
   pip3 install boto3
   ```

### Step 21: Configure AWS Credentials
1. Run:
   ```bash
   aws configure
   ```
2. Enter:
   - **AWS Access Key ID:** [Your access key]
   - **AWS Secret Access Key:** [Your secret key]
   - **Default region name:** us-east-1 (or your region)
   - **Default output format:** json

### Step 22: Create Python Script
1. Create a new file: `iot_events.py`
2. Copy this code:

```python
import boto3
import time
import random
from decimal import Decimal

# Create DynamoDB client
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('iot_events')

# Function to generate and insert events
def generate_events(num_events=100):
    print(f"Generating {num_events} IoT events...")
    
    for i in range(num_events):
        device_num = i % 10  # 10 devices (DEVICE_000 to DEVICE_009)
        
        event = {
            'device_id': f'DEVICE_{device_num:03d}',
            'timestamp': int(time.time()) + i,
            'temperature': Decimal(str(round(20 + random.uniform(0, 30), 2))),
            'humidity': Decimal(str(round(40 + random.uniform(0, 40), 2))),
            'status': random.choice(['NORMAL', 'WARNING', 'CRITICAL']),
            'battery_level': random.randint(10, 100)
        }
        
        # Insert into DynamoDB
        table.put_item(Item=event)
        
        if (i + 1) % 10 == 0:
            print(f"  Inserted {i + 1} events...")
    
    print(f"✅ Successfully inserted {num_events} events!")

# Function to query events for a device
def query_device_events(device_id):
    print(f"\nQuerying events for {device_id}...")
    
    response = table.query(
        KeyConditionExpression='device_id = :device',
        ExpressionAttributeValues={
            ':device': device_id
        },
        ScanIndexForward=False,  # Sort descending (newest first)
        Limit=10
    )
    
    print(f"Found {response['Count']} recent events:")
    for item in response['Items']:
        print(f"  Timestamp: {item['timestamp']}, Temp: {item['temperature']}°C, Status: {item['status']}")

# Main execution
if __name__ == "__main__":
    # Generate 100 events
    generate_events(100)
    
    # Query events for DEVICE_001
    query_device_events('DEVICE_001')
```

### Step 23: Run Python Script
1. In Terminal, run:
   ```bash
   python3 iot_events.py
   ```
2. You should see:
   ```
   Generating 100 IoT events...
     Inserted 10 events...
     Inserted 20 events...
     ...
   ✅ Successfully inserted 100 events!
   
   Querying events for DEVICE_001...
   Found 10 recent events:
     Timestamp: 1719234523, Temp: 35.2°C, Status: NORMAL
     ...
   ```

### Step 24: Verify in Console
1. Go back to DynamoDB Console
2. Click **"Explore table items"** on your `iot_events` table
3. Click **"Scan"** → **"Run"**
4. **Items returned:** Should now show `105` (5 manual + 100 from script)!

---

## PART 5: Monitor and Optimize

### Step 25: View Table Metrics
1. In DynamoDB console, with your table selected
2. Click the **"Monitor"** tab
3. Scroll to see metrics:
   - **Read capacity:** Shows consumed read units
   - **Write capacity:** Shows consumed write units
   - **Read throttle events:** Shows if queries were rate-limited (should be 0)
   - **Write throttle events:** Shows if writes were rate-limited (should be 0)

### Step 26: Check Table Size and Item Count
1. Click **"Overview"** tab
2. Under **"Table details"** section:
   - **Item count:** Shows approximate number of items (~105)
   - **Table size:** Shows storage used (few KB)
3. These update every ~6 hours, not real-time

### Step 27: Export Items (Optional)
1. Click **"Actions"** dropdown
2. Select **"Export to S3"**
3. Configure:
   - **S3 bucket:** Select an existing bucket or create new
   - **Export format:** Choose **"DynamoDB JSON"** or **"Amazon Ion"**
4. Click **"Export"** button
5. This creates a backup in S3 (useful for analytics)

---

## ✅ Exercise 3.3 Completion Checklist

- [ ] Created DynamoDB table `iot_events`
- [ ] Configured partition key (device_id) and sort key (timestamp)
- [ ] Chose capacity mode (On-demand or Provisioned)
- [ ] Manually created 5 IoT events via console
- [ ] Queried data for specific device
- [ ] Used sort key conditions for time-range queries
- [ ] Compared Query vs. Scan operations
- [ ] (Optional) Installed boto3 Python SDK
- [ ] (Optional) Ran Python script to insert 100 events
- [ ] Viewed table metrics and monitoring
- [ ] Understood table size and item count

**🎉 Congratulations!** You've completed Exercise 3.3 and all of Module 3!

---

# 🎓 Module 3 Complete Summary

## What You've Accomplished

### Exercise 3.1: Aurora PostgreSQL ✅
- Created production-grade Multi-AZ Aurora cluster
- Configured read replicas for scaling
- Implemented backups and point-in-time recovery
- Monitored performance with Performance Insights
- Set up CloudWatch alarms for proactive monitoring

### Exercise 3.2: Redshift Data Warehouse ✅
- Built a 2-node Redshift cluster
- Designed star schema for analytics
- Optimized with DISTKEY/SORTKEY for performance
- Loaded 10,000+ transactions
- Created materialized views
- Ran complex analytical queries

### Exercise 3.3: DynamoDB for IoT ✅
- Created NoSQL table with composite key
- Inserted high-velocity IoT events
- Performed efficient queries with partition keys
- Compared Query vs. Scan operations
- Used Python SDK for programmatic access

## Key Database Services Mastered
1. **Amazon Aurora** - Managed relational database (MySQL/PostgreSQL compatible)
2. **Amazon Redshift** - Columnar data warehouse for analytics
3. **Amazon DynamoDB** - NoSQL key-value store for high-velocity data

## Next Steps
- **Module 4:** Data Migration and Transfer Services
- **Module 5:** Compute Services for Data Engineering
- Continue practicing with real datasets

---

## 🧹 Cleanup (IMPORTANT - Avoid Charges)

After completing the exercises, delete resources to stop charges:

### Aurora Cleanup:
1. RDS Console → Clusters
2. Select `prod-aurora-cluster` and `prod-aurora-cluster-restored`
3. Actions → Delete
4. ⬜ Uncheck "Create final snapshot" (for lab)
5. ✅ Check "I acknowledge..."
6. Type: `delete me`
7. Delete

### Redshift Cleanup:
1. Redshift Console → Clusters
2. Select `analytics-cluster`
3. Actions → Delete
4. ⬜ Uncheck "Create final snapshot"
5. Delete

### DynamoDB Cleanup:
1. DynamoDB Console → Tables
2. Select `iot_events`
3. Delete → Confirm

**Estimated time to delete all resources: 10-15 minutes**

---

**Total Module 3 Lab Cost (if cleaned up within 4 hours): ~$25-35**

**🎉 Great job completing Module 3! You're now ready for Module 4.**

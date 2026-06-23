# MODULE 4: Migration and Transfer - GUI Step-by-Step Hands-On Guide

## Table of Contents
- [Exercise 4.1: Database Migration with AWS DMS](#exercise-41-database-migration-with-aws-dms)
- [Exercise 4.2: File Transfer with AWS DataSync](#exercise-42-file-transfer-with-aws-datasync)
- [Exercise 4.3: SFTP File Ingestion with AWS Transfer Family](#exercise-43-sftp-file-ingestion-with-aws-transfer-family)

---

# Exercise 4.1: Database Migration with AWS DMS

**Duration:** 3-4 hours | **Estimated Cost:** ~$20-30 for lab session

## 🎯 Learning Objectives
- Migrate MySQL database to Aurora MySQL using DMS
- Configure CDC (Change Data Capture) for real-time replication
- Monitor migration progress and validate data
- Perform cutover with minimal downtime

---

## PART 1: Create Source MySQL Database (Simulated)

### Step 1: Navigate to EC2 Console
1. Open **AWS Management Console**
2. In the search bar, type **"EC2"**
3. Click on **"EC2"** from the results

### Step 2: Launch EC2 Instance for MySQL
1. Click **"Launch instance"** button (orange, top right)
2. **Name:** Type: `mysql-source-db`

### Step 3: Choose AMI
1. Under **"Application and OS Images"**:
2. Select **"Amazon Linux 2023 AMI"** (first option, free tier eligible)

### Step 4: Choose Instance Type
1. **Instance type:** Select **"t3.medium"** (2 vCPU, 4 GB RAM)
   - Enough for MySQL database with sample data

### Step 5: Create Key Pair
1. **Key pair (login):** Click **"Create new key pair"**
2. **Key pair name:** Type: `mysql-db-key`
3. **Key pair type:** Select **"RSA"**
4. **Private key file format:** Select **".pem"** (macOS/Linux) or **".ppk"** (Windows)
5. Click **"Create key pair"**
6. **IMPORTANT:** File downloads automatically - save it securely!

### Step 6: Configure Network Settings
1. Under **"Network settings"**, click **"Edit"**
2. **VPC:** Leave as default VPC
3. **Subnet:** Leave as **"No preference"**
4. **Auto-assign public IP:** Select **"Enable"**
5. **Firewall (security groups):** Select **"Create security group"**
6. **Security group name:** Type: `mysql-source-sg`
7. Click **"Add security group rule"**:
   - **Type:** **"MySQL/Aurora"** (port 3306)
   - **Source:** **"My IP"** (for testing)

### Step 7: Configure Storage
1. **Storage:** Leave as **8 GB gp3**
2. Click **"Launch instance"** button
3. Wait for status to become **"Running"** (~2 minutes)

### Step 8: Connect to EC2 Instance
1. Select your instance: `mysql-source-db`
2. Click **"Connect"** button
3. Choose **"EC2 Instance Connect"** tab
4. Click **"Connect"** button (opens browser terminal)

### Step 9: Install MySQL on EC2
In the terminal, run these commands:

```bash
# Update system
sudo dnf update -y

# Install MySQL server
sudo dnf install mariadb105-server -y

# Start MySQL service
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Verify MySQL is running
sudo systemctl status mariadb
```

You should see **"active (running)"**

### Step 10: Secure MySQL Installation
```bash
sudo mysql_secure_installation
```

Answer the prompts:
- **Enter current password:** Press Enter (no password)
- **Set root password?** Type: `Y`
- **New password:** `MyStrongPassword123!`
- **Re-enter:** `MyStrongPassword123!`
- **Remove anonymous users?** `Y`
- **Disallow root login remotely?** `N` (we need remote access for DMS)
- **Remove test database?** `Y`
- **Reload privilege tables?** `Y`

### Step 11: Configure MySQL for DMS
```bash
# Connect to MySQL
sudo mysql -u root -p
# Enter password: MyStrongPassword123!
```

At the `mysql>` prompt, run:

```sql
-- Enable binary logging for CDC
SET GLOBAL binlog_format = 'ROW';
SET GLOBAL binlog_row_image = 'FULL';

-- Create database
CREATE DATABASE ecommerce;

-- Create DMS user with permissions
CREATE USER 'dms_user'@'%' IDENTIFIED BY 'DMSPassword123!';
GRANT ALL PRIVILEGES ON ecommerce.* TO 'dms_user'@'%';
GRANT REPLICATION CLIENT ON *.* TO 'dms_user'@'%';
GRANT REPLICATION SLAVE ON *.* TO 'dms_user'@'%';
FLUSH PRIVILEGES;

-- Use database
USE ecommerce;
```

### Step 12: Create Sample Schema
```sql
-- Create customers table
CREATE TABLE customers (
  customer_id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100),
  city VARCHAR(50),
  state VARCHAR(50),
  signup_date DATE
);

-- Create products table
CREATE TABLE products (
  product_id INT AUTO_INCREMENT PRIMARY KEY,
  product_name VARCHAR(100),
  category VARCHAR(50),
  price DECIMAL(10,2)
);

-- Create orders table
CREATE TABLE orders (
  order_id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT,
  product_id INT,
  quantity INT,
  order_date DATETIME,
  total_amount DECIMAL(10,2),
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### Step 13: Insert Sample Data
```sql
-- Insert customers (1000 rows)
INSERT INTO customers (name, email, city, state, signup_date)
SELECT 
  CONCAT('Customer ', n),
  CONCAT('customer', n, '@email.com'),
  CASE (n % 5)
    WHEN 0 THEN 'New York'
    WHEN 1 THEN 'Los Angeles'
    WHEN 2 THEN 'Chicago'
    WHEN 3 THEN 'Houston'
    ELSE 'Miami'
  END,
  CASE (n % 5)
    WHEN 0 THEN 'NY'
    WHEN 1 THEN 'CA'
    WHEN 2 THEN 'IL'
    WHEN 3 THEN 'TX'
    ELSE 'FL'
  END,
  DATE_SUB(CURDATE(), INTERVAL n DAY)
FROM (
  SELECT a.N + b.N * 10 + c.N * 100 + 1 AS n
  FROM 
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b,
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) c
) numbers
LIMIT 1000;

-- Insert products (100 rows)
INSERT INTO products (product_name, category, price)
SELECT 
  CONCAT('Product ', n),
  CASE (n % 4)
    WHEN 0 THEN 'Electronics'
    WHEN 1 THEN 'Furniture'
    WHEN 2 THEN 'Clothing'
    ELSE 'Books'
  END,
  ROUND(10 + (RAND() * 990), 2)
FROM (
  SELECT a.N + b.N * 10 + 1 AS n
  FROM 
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) b
) numbers
LIMIT 100;

-- Insert orders (5000 rows)
INSERT INTO orders (customer_id, product_id, quantity, order_date, total_amount)
SELECT 
  1 + FLOOR(RAND() * 1000) AS customer_id,
  1 + FLOOR(RAND() * 100) AS product_id,
  1 + FLOOR(RAND() * 10) AS quantity,
  DATE_SUB(NOW(), INTERVAL FLOOR(RAND() * 365) DAY) AS order_date,
  ROUND((1 + FLOOR(RAND() * 10)) * (10 + RAND() * 990), 2) AS total_amount
FROM (
  SELECT a.N + b.N * 10 + c.N * 100 + d.N * 1000 + 1 AS n
  FROM 
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) b,
    (SELECT 0 AS N UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) c,
    (SELECT 0 AS N UNION SELECT 1) d
) numbers
LIMIT 5000;
```

**Wait 30-60 seconds** for data insertion.

### Step 14: Verify Data
```sql
SELECT COUNT(*) FROM customers;  -- Should show 1000
SELECT COUNT(*) FROM products;   -- Should show 100
SELECT COUNT(*) FROM orders;     -- Should show 5000

-- Exit MySQL
EXIT;
```

### Step 15: Note EC2 Private IP
1. In EC2 Console, select your instance
2. In **"Details"** tab, find **"Private IPv4 address"**
3. Copy this IP (e.g., `172.31.45.123`)
4. Save it - you'll need it for DMS endpoint

---

## PART 2: Create Target Aurora MySQL Cluster

### Step 16: Navigate to RDS Console
1. AWS Console search bar → Type **"RDS"**
2. Click **"RDS"**

### Step 17: Create Aurora Cluster
1. Click **"Create database"**
2. **Choose engine:** Select **"Amazon Aurora"**
3. **Edition:** Select **"Amazon Aurora MySQL-Compatible Edition"**
4. **Engine version:** Select latest MySQL 8.0 compatible version
5. **Templates:** Select **"Production"** or **"Dev/Test"** (cheaper for lab)

### Step 18: Configure Cluster Settings
1. **DB cluster identifier:** Type: `prod-aurora-mysql`
2. **Master username:** Type: `admin`
3. **Master password:** Type: `AuroraPassword123!`
4. **Confirm password:** Re-enter password
5. **Instance configuration:** Select **"Burstable classes"** → **"db.t3.medium"** (for cost savings)

### Step 19: Configure Connectivity
1. **VPC:** Select same VPC as EC2 instance
2. **Public access:** Select **"Yes"** (for testing; in production use VPN/PrivateLink)
3. **VPC security group:** Create new: `aurora-target-sg`
4. **Database port:** Leave as **3306**

### Step 20: Additional Configuration
1. Expand **"Additional configuration"**
2. **Initial database name:** Type: `ecommerce`
3. **Backup retention period:** **7 days**
4. Click **"Create database"**
5. **Wait 5-10 minutes** for status to become **"Available"**

### Step 21: Get Aurora Endpoint
1. Click on cluster: `prod-aurora-mysql`
2. Go to **"Connectivity & security"** tab
3. Copy **"Writer endpoint"** (e.g., `prod-aurora-mysql.cluster-xxxxx.us-east-1.rds.amazonaws.com`)
4. Save this endpoint

### Step 22: Configure Aurora Security Group
1. In Aurora cluster details, click on the security group: `aurora-target-sg`
2. Click **"Inbound rules"** tab
3. Click **"Edit inbound rules"**
4. Click **"Add rule"**:
   - **Type:** **"MySQL/Aurora"**
   - **Source:** Search for and select `mysql-source-sg` (allows DMS access)
5. Click **"Add rule"** again:
   - **Type:** **"MySQL/Aurora"**
   - **Source:** **"My IP"** (for testing)
6. Click **"Save rules"**

---

## PART 3: Create DMS Replication Instance

### Step 23: Navigate to DMS Console
1. AWS Console search → Type **"DMS"** or **"Database Migration Service"**
2. Click **"AWS Database Migration Service"**

### Step 24: Create Replication Subnet Group
1. In left sidebar, click **"Subnet groups"**
2. Click **"Create subnet group"**
3. **Name:** Type: `dms-subnet-group`
4. **Description:** Type: `Subnet group for DMS replication instance`
5. **VPC:** Select your default VPC
6. **Add subnets:** Click **"Add all subnets"** button
7. Click **"Create subnet group"**

### Step 25: Create Replication Instance
1. In left sidebar, click **"Replication instances"**
2. Click **"Create replication instance"** button

### Step 26: Configure Replication Instance
1. **Name:** Type: `mysql-to-aurora-replication`
2. **Description:** Type: `DMS instance for MySQL to Aurora migration`
3. **Instance class:** Select **"dms.t3.medium"** (2 vCPU, 4 GB RAM)
   - For production: use dms.c5.xlarge or larger
4. **Engine version:** Leave as latest version
5. **Allocated storage (GB):** Type: `100`
6. **VPC:** Select your default VPC
7. **Multi-AZ:** Leave unchecked (for lab; enable for production)
8. **Publicly accessible:** Leave unchecked

### Step 27: Advanced Settings
1. Expand **"Advanced security and network configuration"**
2. **Replication subnet group:** Select `dms-subnet-group`
3. **VPC security group:** Create new OR select existing that allows access to both MySQL and Aurora
4. Click **"Create replication instance"**

### Step 28: Wait for Replication Instance
1. **Status** will show:
   - **"Creating"** → Takes ~5-8 minutes
   - **"Available"** → Ready to use
2. **IMPORTANT:** Wait for "Available" before proceeding

---

## PART 4: Create DMS Endpoints

### Step 29: Create Source Endpoint (MySQL on EC2)
1. In DMS console left sidebar, click **"Endpoints"**
2. Click **"Create endpoint"** button
3. **Endpoint type:** Select **"Source endpoint"**
4. **Endpoint identifier:** Type: `mysql-source-endpoint`
5. **Source engine:** Select **"MySQL"**

### Step 30: Configure Source Endpoint Details
1. **Access to endpoint database:** Select **"Provide access information manually"**
2. **Server name:** Enter EC2 private IP (from Step 15)
3. **Port:** `3306`
4. **SSL mode:** Select **"none"** (for simplicity; use SSL in production)
5. **User name:** `dms_user`
6. **Password:** `DMSPassword123!`
7. **Database name:** `ecommerce`

### Step 31: Test Source Endpoint
1. Scroll down to **"Test endpoint connection"**
2. **VPC:** Select your VPC
3. **Replication instance:** Select `mysql-to-aurora-replication`
4. Click **"Run test"** button
5. **Status** should show **"successful"** (green checkmark)
   - If failed, check security groups and credentials
6. Click **"Create endpoint"**

### Step 32: Create Target Endpoint (Aurora)
1. Click **"Create endpoint"** again
2. **Endpoint type:** Select **"Target endpoint"**
3. **Endpoint identifier:** Type: `aurora-target-endpoint`
4. **Target engine:** Select **"Amazon Aurora MySQL"**
5. ✅ Check: **"Select RDS DB instance"**

### Step 33: Configure Target Endpoint
1. **RDS instance:** Select `prod-aurora-mysql` from dropdown
2. **Access to endpoint database:** **"Provide access information manually"**
3. **Server name:** Auto-filled with Aurora endpoint
4. **Port:** `3306` (auto-filled)
5. **SSL mode:** Select **"none"** (or **"require"** for production)
6. **User name:** `admin`
7. **Password:** `AuroraPassword123!`
8. **Database name:** `ecommerce`

### Step 34: Test Target Endpoint
1. **VPC:** Select your VPC
2. **Replication instance:** Select `mysql-to-aurora-replication`
3. Click **"Run test"**
4. Wait for **"successful"** status
5. Click **"Create endpoint"**

---

## PART 5: Create and Run Replication Task

### Step 35: Create Database Migration Task
1. In DMS left sidebar, click **"Database migration tasks"**
2. Click **"Create task"** button

### Step 36: Configure Task Settings
1. **Task identifier:** Type: `mysql-to-aurora-full-load-cdc`
2. **Replication instance:** Select `mysql-to-aurora-replication`
3. **Source database endpoint:** Select `mysql-source-endpoint`
4. **Target database endpoint:** Select `aurora-target-endpoint`
5. **Migration type:** Select **"Migrate existing data and replicate ongoing changes"**
   - This does full load + CDC (Change Data Capture)

### Step 37: Task Settings
1. **Target table preparation mode:** Select **"Do nothing"**
   - We'll let DMS create tables automatically
2. **Include LOB columns in replication:** Select **"Limited LOB mode"**
3. **Max LOB size (KB):** Leave as **32**
4. **Enable CloudWatch logs:** ✅ Check this
5. **Enable validation:** ✅ Check this (validates data after migration)

### Step 38: Table Mappings
1. **Editing mode:** Select **"Wizard"**
2. Click **"Add new selection rule"**
3. **Schema:** Select **"Enter a schema"**
4. **Source name:** Type: `ecommerce`
5. **Source table name:** Type: `%` (wildcard for all tables)
6. **Action:** Select **"Include"**

### Step 39: Pre-migration Assessment (Optional)
1. Scroll to **"Pre-migration assessment"**
2. ✅ Check **"Turn on pre-migration assessment"**
3. This validates the migration before starting

### Step 40: Start Migration Task
1. **Start task on create:** ✅ Check this (starts immediately)
2. Click **"Create task"** button
3. You'll be redirected to the task details page

---

## PART 6: Monitor Migration Progress

### Step 41: View Task Overview
1. On the task details page, you'll see:
   - **Status:** "Starting" → "Running" → "Load complete, replication ongoing"
   - **% complete:** Progress bar
   - **Tables:** Number of tables being migrated

### Step 42: Monitor Table Statistics
1. Scroll down to **"Table statistics"** section
2. You'll see a table with:
   - **Table name:** customers, products, orders
   - **Full load rows:** Number of rows loaded (should match source counts)
   - **Full load conditioned rows:** Rows that met conditions
   - **Full load error rows:** Errors (should be 0)
   - **CDC inserts/updates/deletes:** Real-time changes

### Step 43: Check CloudWatch Logs
1. Scroll to **"Migration task logs"** section
2. Click **"View CloudWatch Logs"**
3. You'll see log streams:
   - **dms-task-[task-id]:** Main task log
4. Click on the log stream to view detailed logs
5. Look for:
   - **"Full load complete"** message
   - **"CDC started"** message
   - Any error messages

### Step 44: Test CDC (Change Data Capture)
1. Go back to your EC2 instance terminal (MySQL connection)
2. Connect to source MySQL:
   ```bash
   sudo mysql -u root -p ecommerce
   ```
3. Insert a new customer:
   ```sql
   INSERT INTO customers (name, email, city, state, signup_date)
   VALUES ('Test Customer', 'test@email.com', 'Seattle', 'WA', CURDATE());
   
   -- Get the ID
   SELECT LAST_INSERT_ID();
   -- Note this ID (e.g., 1001)
   
   EXIT;
   ```

### Step 45: Verify CDC in Aurora
1. Connect to Aurora target database:
   ```bash
   mysql -h prod-aurora-mysql.cluster-xxxxx.us-east-1.rds.amazonaws.com -u admin -p ecommerce
   ```
2. Enter password: `AuroraPassword123!`
3. Query for the new customer:
   ```sql
   SELECT * FROM customers WHERE email = 'test@email.com';
   ```
4. **You should see the new customer!** This proves CDC is working.
5. **Wait time:** Usually 1-5 seconds for CDC replication

### Step 46: View Data Validation Results
1. Back in DMS console, on the task page
2. Scroll to **"Data validation"** section
3. You'll see validation statistics:
   - **Validated:** Number of rows validated
   - **Mismatched:** Rows that don't match (should be 0)
   - **Pending:** Rows waiting for validation
4. Click **"View data validation results"** for details

---

## PART 7: Perform Cutover

### Step 47: Stop Application Writes (Simulated)
In a real migration:
1. Put application in "maintenance mode"
2. Stop all writes to source database
3. Wait for CDC to catch up (CDC latency = 0 seconds)

For this lab:
1. Just ensure no more writes are happening

### Step 48: Verify Replication Lag
1. In DMS task details, look at **"CDCLatencySource"** metric
2. Should be **0 seconds** or very low (< 1 second)
3. This means target is caught up with source

### Step 49: Final Data Validation
Connect to both databases and compare counts:

**Source MySQL:**
```bash
sudo mysql -u root -p ecommerce
```
```sql
SELECT 'customers' AS tbl, COUNT(*) AS cnt FROM customers
UNION ALL
SELECT 'products', COUNT(*) FROM products
UNION ALL
SELECT 'orders', COUNT(*) FROM orders;

EXIT;
```

**Target Aurora:**
```bash
mysql -h [AURORA-ENDPOINT] -u admin -p ecommerce
```
```sql
SELECT 'customers' AS tbl, COUNT(*) AS cnt FROM customers
UNION ALL
SELECT 'products', COUNT(*) FROM products
UNION ALL
SELECT 'orders', COUNT(*) FROM orders;

EXIT;
```

**Counts should match exactly!**

### Step 50: Update Application Connection String
In a real migration:
1. Update application configuration
2. Change database hostname from EC2 IP to Aurora endpoint
3. Restart application
4. Test application functionality

For this lab:
1. Just verify you can connect to Aurora successfully

### Step 51: Stop DMS Task
1. In DMS console, go to **"Database migration tasks"**
2. Select your task: `mysql-to-aurora-full-load-cdc`
3. Click **"Actions"** dropdown
4. Select **"Stop"**
5. Confirm by clicking **"Stop task"**

---

## ✅ Exercise 4.1 Completion Checklist

- [ ] Created EC2 instance with MySQL database
- [ ] Configured MySQL for CDC (binary logging)
- [ ] Created sample schema with 3 tables
- [ ] Inserted 6,100 rows of sample data
- [ ] Created Aurora MySQL cluster
- [ ] Created DMS replication instance
- [ ] Created DMS subnet group
- [ ] Created source endpoint (MySQL) and tested connection
- [ ] Created target endpoint (Aurora) and tested connection
- [ ] Created migration task with full load + CDC
- [ ] Monitored migration progress via table statistics
- [ ] Verified CDC by inserting test data
- [ ] Validated data consistency
- [ ] Performed cutover simulation

**🎉 Congratulations!** You've completed Exercise 4.1!

---

# Exercise 4.2: File Transfer with AWS DataSync

**Duration:** 2-3 hours | **Estimated Cost:** ~$10-15 for lab session

## 🎯 Learning Objectives
- Deploy DataSync agent (simulated on-premises)
- Transfer files from NFS to S3
- Schedule recurring transfers
- Monitor transfer performance

---

## PART 1: Set Up Source NFS Server (Simulated On-Premises)

### Step 1: Launch EC2 Instance for NFS
1. **EC2 Console** → **"Launch instance"**
2. **Name:** `nfs-source-server`
3. **AMI:** Amazon Linux 2023
4. **Instance type:** **t3.medium**
5. **Key pair:** Create new: `nfs-server-key`
6. **Security group:** Create new: `nfs-server-sg`
   - Add rule: **NFS (2049)** from **My IP**
   - Add rule: **SSH (22)** from **My IP**
7. **Storage:** **20 GB gp3**
8. Click **"Launch instance"**

### Step 2: Connect and Install NFS
1. Select instance → **"Connect"** → **"EC2 Instance Connect"**
2. In the terminal:

```bash
# Install NFS server
sudo dnf install nfs-utils -y

# Create directory for NFS export
sudo mkdir -p /exports/company-logs
sudo chmod 777 /exports/company-logs

# Create sample log files
for i in {1..100}; do
  sudo bash -c "echo 'Log entry $i at $(date)' > /exports/company-logs/log-file-$i.txt"
done

# Create large files (simulate real logs)
sudo bash -c "yes 'Sample log data line' | head -n 100000 > /exports/company-logs/large-log-1.txt"
sudo bash -c "yes 'Sample log data line' | head -n 100000 > /exports/company-logs/large-log-2.txt"

# Configure NFS exports
sudo bash -c 'echo "/exports/company-logs *(rw,sync,no_root_squash)" >> /etc/exports'

# Start NFS service
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
sudo exportfs -a

# Verify NFS is running
sudo exportfs -v
showmount -e localhost
```

### Step 3: Note NFS Server IP
1. In EC2 console, select the instance
2. Copy **"Private IPv4 address"** (e.g., `172.31.45.200`)
3. Save this IP - needed for DataSync

---

## PART 2: Deploy DataSync Agent

### Step 4: Launch DataSync Agent EC2 Instance
1. **EC2 Console** → **"Launch instance"**
2. **Name:** `datasync-agent`
3. **AMI:** Search for **"DataSync"** in AWS Marketplace
4. **Alternative method:** Use Amazon Linux and install DataSync agent

**For this lab, we'll use simpler approach:**
1. **AMI:** Amazon Linux 2023
2. **Instance type:** **t3.xlarge** (4 vCPU, 16 GB RAM - required for DataSync agent)
3. **Key pair:** Use existing or create new
4. **Security group:** Create new: `datasync-agent-sg`
   - **SSH (22)** from My IP
   - **HTTP (80)** from My IP (for activation)
   - **HTTPS (443)** outbound (for AWS communication)
   - **NFS (2049)** outbound to NFS server
5. **Storage:** **80 GB gp3**
6. Click **"Launch instance"**

### Step 5: Activate DataSync Agent
1. In EC2 console, note the **Public IPv4 address** of datasync-agent
2. Open browser, go to: `http://[AGENT-PUBLIC-IP]/`
3. You'll see DataSync agent activation page
4. **Activation key:** Will be auto-generated

**Note:** For real deployment, download DataSync OVA from AWS and deploy to VMware/Hyper-V. For this lab, we'll create the agent directly in AWS.

---

## PART 3: Create DataSync Locations and Task

### Step 6: Navigate to DataSync Console
1. AWS Console search → **"DataSync"**
2. Click **"AWS DataSync"**

### Step 7: Create Source Location (NFS)
1. In left sidebar, click **"Locations"**
2. Click **"Create location"** button
3. **Location type:** Select **"Network File System (NFS)"**

### Step 8: Configure NFS Source Location
1. **Agents:** 
   - For this lab, we'll simulate without agent
   - Select **"Create a new agent"** → But we'll use VPC endpoint approach
   
**Simplified approach for lab:**
1. Skip agent for now
2. Instead, we'll use **Amazon EFS** as source (simpler for lab)

**Let's restart with EFS:**

### Step 9: Create EFS File System (Easier Source)
1. AWS Console → Search **"EFS"**
2. Click **"Create file system"**
3. **Name:** `company-logs-source`
4. **VPC:** Default VPC
5. Click **"Create"**
6. Wait for status **"Available"** (~2 minutes)

### Step 10: Mount EFS and Add Files
1. Click on the EFS: `company-logs-source`
2. Click **"Attach"** button
3. Copy the mount command (NFS client):
   ```bash
   sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-xxxxx.efs.us-east-1.amazonaws.com:/ /mnt/efs
   ```
4. SSH into the NFS EC2 instance (or create new t3.micro instance)
5. Run:
   ```bash
   # Install NFS client
   sudo dnf install nfs-utils -y
   
   # Create mount point
   sudo mkdir -p /mnt/efs
   
   # Mount EFS (use command from step 3)
   sudo mount -t nfs4 -o nfsvers=4.1 [EFS-DNS-NAME]:/ /mnt/efs
   
   # Create sample files
   sudo mkdir -p /mnt/efs/logs/2024/06
   for i in {1..500}; do
     sudo bash -c "echo 'Application log entry $i at $(date)' > /mnt/efs/logs/2024/06/app-log-$i.txt"
   done
   
   # Create larger files
   sudo dd if=/dev/zero of=/mnt/efs/logs/2024/06/large-file-1.dat bs=1M count=100
   sudo dd if=/dev/zero of=/mnt/efs/logs/2024/06/large-file-2.dat bs=1M count=100
   
   # Verify
   ls -lh /mnt/efs/logs/2024/06/ | head -20
   df -h /mnt/efs
   ```

### Step 11: Create S3 Destination Bucket
1. AWS Console → **"S3"**
2. Click **"Create bucket"**
3. **Bucket name:** Type: `company-logs-archive-[your-account-id]` (must be globally unique)
   - Example: `company-logs-archive-123456789012`
4. **Region:** Same as EFS (e.g., us-east-1)
5. **Block Public Access:** Leave all checked (recommended)
6. **Versioning:** Enable (optional)
7. **Encryption:** Select **"Server-side encryption with Amazon S3 managed keys (SSE-S3)"**
8. Click **"Create bucket"**

### Step 12: Create IAM Role for DataSync
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity:** Select **"AWS service"**
3. **Use case:** Search and select **"DataSync"**
4. Click **"Next"**
5. **Permissions:** Should auto-attach **"AWSDataSyncFullAccess"**
6. Click **"Next"**
7. **Role name:** Type: `DataSyncS3AccessRole`
8. Click **"Create role"**
9. Click on the created role
10. Copy the **Role ARN** (e.g., `arn:aws:iam::123456789012:role/DataSyncS3AccessRole`)

### Step 13: Create DataSync Source Location (EFS)
1. Back to **DataSync Console** → **"Locations"** → **"Create location"**
2. **Location type:** Select **"Amazon EFS file system"**
3. **Region:** Your current region
4. **EFS file system:** Select `company-logs-source`
5. **Mount path:** Type: `/logs/2024/06` (the subdirectory we created)
6. **Subnet:** Select any subnet in your VPC
7. **Security groups:** Select default security group
8. Click **"Create location"**

### Step 14: Create DataSync Destination Location (S3)
1. Click **"Create location"** again
2. **Location type:** Select **"Amazon S3 bucket"**
3. **Region:** Your current region
4. **S3 bucket:** Select `company-logs-archive-[your-account-id]`
5. **S3 storage class:** Select **"S3 Standard"**
   - Or **"S3 Intelligent-Tiering"** for cost optimization
6. **Folder:** Leave empty (root of bucket) OR type: `archived-logs/`
7. **IAM role:** Click **"Autogenerate"** OR paste ARN from Step 12
8. Click **"Create location"**

---

## PART 4: Create and Execute DataSync Task

### Step 15: Create DataSync Task
1. In DataSync console, click **"Tasks"** in left sidebar
2. Click **"Create task"** button

### Step 16: Configure Source and Destination
1. **Source location:** Select the EFS location you created
2. **Destination location:** Select the S3 location you created
3. Click **"Next"**

### Step 17: Configure Task Settings
1. **Task name:** Type: `efs-to-s3-daily-sync`
2. **Task execution mode:** Select **"Transfer all data"**
   - Alternative: **"Transfer only changed data"** (for incremental)

### Step 18: Configure Data Transfer Options
1. **Verify data:** Select **"Verify only the data transferred"** (recommended)
2. **Copy ownership:** ✅ Check (preserves file permissions)
3. **Copy timestamps:** ✅ Check
4. **Bandwidth limit:** Leave empty (unlimited) OR set limit in MB/s
5. **Task logging:** Select **"Send logs to CloudWatch Logs"**
6. **Log level:** Select **"Log all transferred objects and files"**

### Step 19: Configure Schedule (Optional)
1. **Task schedule:** Select **"Schedule"**
2. **Frequency:** Select **"Daily"**
3. **Start time:** Select **"02:00 UTC"** (2 AM)
4. OR use **"Cron expression":** Type: `cron(0 2 * * ? *)`
5. Click **"Next"**

### Step 20: Review and Create
1. Review all settings
2. Scroll to bottom
3. Click **"Create task"**

### Step 21: Start Task Manually (First Run)
1. You'll see your task: `efs-to-s3-daily-sync`
2. **Status:** "Available"
3. Click the task name to open details
4. Click **"Start"** button (with default settings)
5. Click **"Start"** in the confirmation dialog

---

## PART 5: Monitor DataSync Transfer

### Step 22: View Task Execution
1. On the task details page, scroll to **"History"** tab
2. You'll see the current execution with:
   - **Status:** "Running" → "Success"
   - **Start time**
   - **Duration**
   - **Data transferred**

### Step 23: Monitor Real-Time Progress
While running, click on the execution
1. **Files transferred:** Counter showing files copied
2. **Data transferred:** Total GB/MB transferred
3. **Estimated time remaining:** Countdown
4. **Throughput:** MB/s transfer speed

### Step 24: View CloudWatch Logs
1. In task execution details, scroll to **"CloudWatch logs"**
2. Click **"View in CloudWatch Logs"**
3. You'll see detailed logs:
   ```
   Transferred: /app-log-1.txt
   Transferred: /app-log-2.txt
   Transferred: /large-file-1.dat
   ...
   ```

### Step 25: View CloudWatch Metrics
1. In DataSync console, click your task
2. Go to **"Monitoring"** tab
3. You'll see graphs:
   - **Files transferred:** Number of files
   - **Bytes transferred:** Total data size
   - **Bytes per second:** Transfer throughput

### Step 26: Verify Files in S3
1. Go to **S3 Console**
2. Click on bucket: `company-logs-archive-[your-account-id]`
3. Navigate to `archived-logs/` (if you specified folder)
4. You should see all files from EFS:
   - `app-log-1.txt`
   - `app-log-2.txt`
   - `large-file-1.dat`
   - `large-file-2.dat`
   - etc.
5. Click on any file to view details
6. **Size** and **Last modified** should match source

### Step 27: Compare File Counts
**Source EFS:**
```bash
# SSH to instance with EFS mounted
ls -1 /mnt/efs/logs/2024/06/ | wc -l
# Should show 502 files (500 small + 2 large)
```

**Destination S3:**
1. S3 Console → Your bucket
2. Navigate to folder
3. **Objects:** Should show **502**

**Counts should match!**

---

## ✅ Exercise 4.2 Completion Checklist

- [ ] Created EFS file system as source
- [ ] Mounted EFS and created 500+ sample files
- [ ] Created S3 bucket as destination
- [ ] Created IAM role for DataSync
- [ ] Created DataSync source location (EFS)
- [ ] Created DataSync destination location (S3)
- [ ] Created DataSync task with verification enabled
- [ ] Configured daily schedule (cron)
- [ ] Started task execution manually
- [ ] Monitored transfer progress in real-time
- [ ] Viewed CloudWatch logs and metrics
- [ ] Verified all files transferred to S3
- [ ] Confirmed file counts match

**🎉 Congratulations!** You've completed Exercise 4.2!

---

# Exercise 4.3: SFTP File Ingestion with AWS Transfer Family

**Duration:** 2-3 hours | **Estimated Cost:** ~$15-20 for lab session

## 🎯 Learning Objectives
- Create SFTP server with AWS Transfer Family
- Configure users with SSH keys and home directories
- Test SFTP upload from client
- Automate file processing with Lambda and EventBridge
- Implement file validation and quarantine

---

## PART 1: Create S3 Bucket Structure

### Step 1: Create S3 Bucket for SFTP
1. **S3 Console** → **"Create bucket"**
2. **Bucket name:** `partner-file-uploads-[your-account-id]`
3. **Region:** us-east-1 (or your preferred region)
4. **Versioning:** Enable (recommended)
5. **Encryption:** SSE-S3 (default)
6. Click **"Create bucket"**

### Step 2: Create Folder Structure
1. Click on the bucket you just created
2. Click **"Create folder"**
3. **Folder name:** `partner-a/` → Create
4. Inside `partner-a/`, create subfolders:
   - Click **"Create folder"** → `incoming/` → Create
   - Click **"Create folder"** → `processed/` → Create
   - Click **"Create folder"** → `quarantine/` → Create
5. Repeat for partner-b:
   - Create `partner-b/incoming/`
   - Create `partner-b/processed/`
   - Create `partner-b/quarantine/`

**Final structure:**
```
partner-file-uploads-123456789012/
├── partner-a/
│   ├── incoming/
│   ├── processed/
│   └── quarantine/
└── partner-b/
    ├── incoming/
    ├── processed/
    └── quarantine/
```

---

## PART 2: Create IAM Role for Transfer Family

### Step 3: Create Transfer Family Service Role
1. **IAM Console** → **"Roles"** → **"Create role"**
2. **Trusted entity type:** Select **"AWS service"**
3. **Use case:** Scroll down to **"Transfer"**
4. Select **"Transfer"** → Click **"Next"**

### Step 4: Attach S3 Access Policy
1. Click **"Create policy"** (opens new tab)
2. Switch to **"JSON"** tab
3. Paste this policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowListingOfUserFolder",
         "Effect": "Allow",
         "Action": "s3:ListBucket",
         "Resource": "arn:aws:s3:::partner-file-uploads-[YOUR-ACCOUNT-ID]",
         "Condition": {
           "StringLike": {
             "s3:prefix": [
               "${transfer:UserName}/*",
               "${transfer:UserName}"
             ]
           }
         }
       },
       {
         "Sid": "HomeDirObjectAccess",
         "Effect": "Allow",
         "Action": [
           "s3:PutObject",
           "s3:GetObject",
           "s3:GetObjectVersion",
           "s3:DeleteObject",
           "s3:DeleteObjectVersion"
         ],
         "Resource": "arn:aws:s3:::partner-file-uploads-[YOUR-ACCOUNT-ID]/${transfer:UserName}/*"
       }
     ]
   }
   ```
4. **Replace** `[YOUR-ACCOUNT-ID]` with your actual account ID
5. Click **"Next"**
6. **Policy name:** `TransferFamilyS3AccessPolicy`
7. Click **"Create policy"**

### Step 5: Complete Role Creation
1. Go back to the role creation tab
2. Click refresh button next to **"Create policy"**
3. Search for `TransferFamilyS3AccessPolicy`
4. ✅ Check the policy
5. Click **"Next"**
6. **Role name:** `TransferFamilyS3AccessRole`
7. Click **"Create role"**
8. Copy the **Role ARN** (needed later)

---

## PART 3: Create AWS Transfer Family SFTP Server

### Step 6: Navigate to Transfer Family Console
1. AWS Console search → **"Transfer Family"** or **"AWS Transfer"**
2. Click **"AWS Transfer Family"**

### Step 7: Create SFTP Server
1. Click **"Create server"** button

### Step 8: Choose Protocols
1. **Choose protocols:** ✅ Check **"SFTP (SSH File Transfer Protocol)"**
2. Click **"Next"**

### Step 9: Choose Identity Provider
1. **Identity provider:** Select **"Service managed"**
   - This uses Transfer Family's built-in user management
   - Alternative: **"Custom"** (Lambda-based auth) or **"AWS Directory Service"**
2. Click **"Next"**

### Step 10: Choose Endpoint
1. **Endpoint type:** Select **"Publicly accessible"**
   - For production with partners, use VPC endpoint or VPC with EIP
2. **Custom hostname:** Leave empty for now
   - Optional: Use Route 53 to create custom domain like `sftp.yourcompany.com`
3. Click **"Next"**

### Step 11: Choose Domain
1. **Domain:** Select **"Amazon S3"**
   - Alternative: **"Amazon EFS"** if you need file locking
2. Click **"Next"**

### Step 12: Configure Logging
1. **CloudWatch logging:** Select **"Create a new role"**
2. This auto-creates an IAM role for logging
3. Click **"Next"**

### Step 13: Review and Create
1. Review all settings:
   - Protocol: SFTP
   - Identity provider: Service managed
   - Endpoint: Publicly accessible
   - Domain: Amazon S3
2. Click **"Create server"**

### Step 14: Wait for Server to Start
1. **Server state** will show:
   - **"Starting"** → Takes ~2-3 minutes
   - **"Online"** → Ready to use
2. Wait for **"Online"** status

### Step 15: Get Server Endpoint
1. Click on your server ID (e.g., `s-1234567890abcdef0`)
2. In **"Endpoint details"** section, copy:
   - **Endpoint:** `s-1234567890abcdef0.server.transfer.us-east-1.amazonaws.com`
3. Save this endpoint - needed for SFTP client

---

## PART 4: Create SFTP Users

### Step 16: Generate SSH Key Pair (Local Computer)
On your local computer terminal:

```bash
# Create SSH key pair
ssh-keygen -t rsa -b 4096 -f partner-a-sftp-key -N ""

# This creates two files:
# - partner-a-sftp-key (private key - keep secret!)
# - partner-a-sftp-key.pub (public key - upload to AWS)

# View public key
cat partner-a-sftp-key.pub
```

**Copy the entire output** (starts with `ssh-rsa ...`)

### Step 17: Create User for Partner A
1. In Transfer Family console, with your server selected
2. Scroll to **"Users"** section
3. Click **"Add user"** button

### Step 18: Configure User Details
1. **User name:** Type: `partner-a`
   - **Important:** This must match the S3 folder name!
2. **IAM role:** Select `TransferFamilyS3AccessRole` (created in Step 5)
3. **Home directory:** Click **"Browse"** →
   - Select bucket: `partner-file-uploads-[your-account-id]`
   - Select folder: `partner-a/`
   - Click **"Choose"**
4. **Restricted:** ✅ Check **"Restrict to configured directory"**
   - This prevents user from navigating outside their folder

### Step 19: Add SSH Public Key
1. Scroll to **"SSH public keys"**
2. Click **"Add SSH public key"**
3. Paste the public key content from Step 16
4. Click **"Add key"**

### Step 20: Complete User Creation
1. Click **"Add"** button
2. User `partner-a` is now created

### Step 21: Create Second User (Partner B)
1. Generate another SSH key:
   ```bash
   ssh-keygen -t rsa -b 4096 -f partner-b-sftp-key -N ""
   cat partner-b-sftp-key.pub
   ```
2. Copy the public key
3. In Transfer Family console, click **"Add user"**
4. **User name:** `partner-b`
5. **IAM role:** `TransferFamilyS3AccessRole`
6. **Home directory:** `s3://partner-file-uploads-[id]/partner-b/`
7. **Restricted:** ✅ Check
8. **SSH public key:** Paste partner-b's public key
9. Click **"Add"**

---

## PART 5: Test SFTP Upload

### Step 22: Test SFTP Connection (Partner A)
On your local computer:

```bash
# Test SFTP connection
sftp -i partner-a-sftp-key partner-a@s-1234567890abcdef0.server.transfer.us-east-1.amazonaws.com

# If successful, you'll see:
# Connected to s-1234567890abcdef0.server.transfer.us-east-1.amazonaws.com.
# sftp>
```

**Troubleshooting:**
- If connection refused: Server not online yet (wait 2-3 min)
- If permission denied: Check SSH key file permissions (`chmod 600 partner-a-sftp-key`)
- If host key verification failed: Type `yes` to accept

### Step 23: Navigate SFTP Directory
At the `sftp>` prompt:

```bash
# List current directory
ls

# You should see:
# incoming
# processed
# quarantine

# Try to go up (should fail due to restriction)
cd ..
# Should get error: Couldn't canonicalize path

# Change to incoming
cd incoming

# List (should be empty)
ls
```

### Step 24: Create Test File and Upload
On your local computer, open another terminal:

```bash
# Create a test CSV file
cat > test-sales-data.csv << 'EOF'
order_id,customer_name,product,quantity,price,order_date
1001,John Doe,Laptop,1,1299.99,2024-06-24
1002,Jane Smith,Mouse,2,29.99,2024-06-24
1003,Bob Johnson,Keyboard,1,89.99,2024-06-24
1004,Alice Williams,Monitor,1,349.99,2024-06-24
1005,Charlie Brown,Headphones,1,149.99,2024-06-24
EOF

# Upload via SFTP
sftp -i partner-a-sftp-key partner-a@s-[SERVER-ID].server.transfer.us-east-1.amazonaws.com

# At sftp> prompt:
cd incoming
put test-sales-data.csv
ls

# You should see the file!
# -rw-rw-rw-    1 1000     1000          XXX Jun 24 10:30 test-sales-data.csv

# Exit SFTP
bye
```

### Step 25: Verify File in S3
1. Go to **S3 Console**
2. Navigate to: `partner-file-uploads-[id]/partner-a/incoming/`
3. You should see **test-sales-data.csv**
4. Click on it to view details
5. **Size** should match the file you uploaded

---

## PART 6: Automate Processing with Lambda

### Step 26: Create Lambda Function
1. **Lambda Console** → **"Create function"**
2. **Function name:** `ProcessPartnerFiles`
3. **Runtime:** Select **"Python 3.12"**
4. **Permissions:** Select **"Create a new role with basic Lambda permissions"**
5. Click **"Create function"**

### Step 27: Add S3 Permissions to Lambda Role
1. Scroll to **"Configuration"** tab → **"Permissions"**
2. Click on the **Role name** (opens IAM console)
3. Click **"Add permissions"** → **"Attach policies"**
4. Search for **"AmazonS3FullAccess"** (for lab; in production use least privilege)
5. ✅ Check the policy → Click **"Attach policies"**

### Step 28: Write Lambda Function Code
1. Go back to Lambda console → **"Code"** tab
2. Replace the code with:

```python
import json
import boto3
import urllib.parse
from datetime import datetime

s3 = boto3.client('s3')

def lambda_handler(event, context):
    print(f"Received event: {json.dumps(event)}")
    
    # Get bucket and key from S3 event
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])
        
        print(f"Processing file: s3://{bucket}/{key}")
        
        # Only process files in 'incoming' folder
        if '/incoming/' not in key:
            print(f"Skipping - not in incoming folder")
            continue
        
        # Validate file
        is_valid, error_msg = validate_file(bucket, key)
        
        # Determine destination folder
        partner_name = key.split('/')[0]  # e.g., 'partner-a'
        filename = key.split('/')[-1]
        
        if is_valid:
            # Move to processed
            dest_key = f"{partner_name}/processed/{filename}"
            print(f"Valid file - moving to {dest_key}")
        else:
            # Move to quarantine
            dest_key = f"{partner_name}/quarantine/{filename}"
            print(f"Invalid file ({error_msg}) - moving to {dest_key}")
        
        # Copy to destination
        copy_source = {'Bucket': bucket, 'Key': key}
        s3.copy_object(CopySource=copy_source, Bucket=bucket, Key=dest_key)
        
        # Add tags
        s3.put_object_tagging(
            Bucket=bucket,
            Key=dest_key,
            Tagging={
                'TagSet': [
                    {'Key': 'ProcessedDate', 'Value': datetime.now().isoformat()},
                    {'Key': 'Valid', 'Value': str(is_valid)},
                    {'Key': 'Partner', 'Value': partner_name}
                ]
            }
        )
        
        # Delete from incoming
        s3.delete_object(Bucket=bucket, Key=key)
        print(f"Moved from {key} to {dest_key}")
    
    return {
        'statusCode': 200,
        'body': json.dumps('Processing complete')
    }

def validate_file(bucket, key):
    """Validate CSV file structure"""
    try:
        # Download file
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response['Body'].read().decode('utf-8')
        
        # Check file extension
        if not key.endswith('.csv'):
            return False, "Not a CSV file"
        
        # Check if not empty
        if len(content) < 10:
            return False, "File too small or empty"
        
        # Check for required headers (example: sales data)
        lines = content.split('\n')
        if len(lines) < 2:
            return False, "Missing data rows"
        
        headers = lines[0].lower()
        required_fields = ['order_id', 'customer_name', 'product', 'quantity', 'price']
        
        for field in required_fields:
            if field not in headers:
                return False, f"Missing required field: {field}"
        
        # All validations passed
        return True, None
        
    except Exception as e:
        print(f"Validation error: {str(e)}")
        return False, str(e)
```

3. Click **"Deploy"** button

### Step 29: Configure Lambda Timeout
1. Go to **"Configuration"** tab → **"General configuration"**
2. Click **"Edit"**
3. **Timeout:** Change from 3 sec to **60 seconds**
4. **Memory:** **256 MB** (adequate for file processing)
5. Click **"Save"**

---

## PART 7: Configure S3 Event Notification

### Step 30: Create S3 Event Trigger
1. In Lambda console, click **"Add trigger"** button
2. **Select a source:** Choose **"S3"**

### Step 31: Configure S3 Trigger
1. **Bucket:** Select `partner-file-uploads-[your-account-id]`
2. **Event type:** Select **"All object create events"** OR
   - Select **"PUT"** (more specific)
3. **Prefix:** Type: `partner-a/incoming/`
   - This filters to only partner-a incoming files
4. **Suffix:** Leave empty OR type: `.csv` (only CSV files)
5. ✅ Acknowledge the recursive invocation statement
6. Click **"Add"**

### Step 32: Add Trigger for Partner B
1. Click **"Add trigger"** again
2. **Source:** S3
3. **Bucket:** Same bucket
4. **Event type:** PUT
5. **Prefix:** `partner-b/incoming/`
6. **Suffix:** `.csv`
7. Click **"Add"**

---

## PART 8: Test End-to-End Workflow

### Step 33: Upload Valid File via SFTP
```bash
# Create another test file
cat > valid-order-202406.csv << 'EOF'
order_id,customer_name,product,quantity,price,order_date
2001,Mike Davis,Tablet,1,599.99,2024-06-24
2002,Sarah Connor,Charger,3,19.99,2024-06-24
2003,Tom Hardy,Cable,5,9.99,2024-06-24
EOF

# Upload via SFTP
sftp -i partner-a-sftp-key partner-a@s-[SERVER-ID].server.transfer.[REGION].amazonaws.com
cd incoming
put valid-order-202406.csv
ls
bye
```

### Step 34: Upload Invalid File (Missing Required Field)
```bash
# Create invalid file (missing 'quantity' column)
cat > invalid-order.csv << 'EOF'
order_id,customer_name,product,price
3001,Invalid Customer,Bad Product,100.00
EOF

# Upload
sftp -i partner-a-sftp-key partner-a@s-[SERVER-ID].server.transfer.[REGION].amazonaws.com
cd incoming
put invalid-order.csv
ls
bye
```

### Step 35: Monitor Lambda Execution
1. **Lambda Console** → Click on `ProcessPartnerFiles`
2. Go to **"Monitor"** tab
3. Click **"View CloudWatch logs"**
4. Click on the latest log stream
5. You should see logs:
   ```
   Processing file: s3://partner-file-uploads-123/partner-a/incoming/valid-order-202406.csv
   Valid file - moving to partner-a/processed/valid-order-202406.csv
   
   Processing file: s3://partner-file-uploads-123/partner-a/incoming/invalid-order.csv
   Invalid file (Missing required field: quantity) - moving to partner-a/quarantine/invalid-order.csv
   ```

### Step 36: Verify File Movement in S3
1. Go to **S3 Console** → Your bucket
2. Navigate to `partner-a/incoming/` → **Should be empty!**
3. Navigate to `partner-a/processed/` → **valid-order-202406.csv** should be here
4. Navigate to `partner-a/quarantine/` → **invalid-order.csv** should be here

### Step 37: Check Object Tags
1. Click on `partner-a/processed/valid-order-202406.csv`
2. Go to **"Properties"** tab
3. Scroll to **"Tags"** section
4. You should see:
   - **ProcessedDate:** 2024-06-24T...
   - **Valid:** True
   - **Partner:** partner-a

---

## ✅ Exercise 4.3 Completion Checklist

- [ ] Created S3 bucket with partner folder structure
- [ ] Created IAM role for Transfer Family with S3 access
- [ ] Created AWS Transfer Family SFTP server
- [ ] Generated SSH key pairs for partners
- [ ] Created SFTP users (partner-a, partner-b)
- [ ] Tested SFTP connection and file upload
- [ ] Created Lambda function for file processing
- [ ] Implemented file validation logic
- [ ] Configured S3 event notifications to trigger Lambda
- [ ] Tested end-to-end workflow with valid file
- [ ] Tested with invalid file (moved to quarantine)
- [ ] Verified files moved to correct folders (processed/quarantine)
- [ ] Confirmed object tags applied correctly

**🎉 Congratulations!** You've completed Exercise 4.3 and all of Module 4!

---

# 🎓 Module 4 Complete Summary

## What You've Accomplished

### Exercise 4.1: Database Migration with AWS DMS ✅
- Migrated MySQL database to Aurora MySQL
- Configured full load + CDC for real-time replication
- Validated data consistency
- Performed simulated cutover

### Exercise 4.2: File Transfer with AWS DataSync ✅
- Transferred files from EFS to S3
- Configured scheduled daily transfers
- Monitored transfer performance with CloudWatch
- Verified data integrity

### Exercise 4.3: SFTP with AWS Transfer Family ✅
- Created secure SFTP server for partners
- Configured SSH key-based authentication
- Automated file processing with Lambda
- Implemented validation and quarantine workflow

## Key Services Mastered
1. **AWS DMS** - Database migration with minimal downtime
2. **AWS DataSync** - Automated file transfer with scheduling
3. **AWS Transfer Family** - Managed SFTP for partner integrations

## Next Steps
- **Module 5:** Compute Services (Lambda, Batch, EMR)
- Continue with remaining modules

---

## 🧹 Cleanup Resources

**DMS:**
- Stop and delete migration tasks
- Delete replication instances
- Delete endpoints

**DataSync:**
- Delete tasks and locations

**Transfer Family:**
- Delete users
- Delete server (~$0.30/hour - expensive!)

**S3:**
- Empty and delete buckets

**EC2/EFS:**
- Terminate instances
- Delete EFS file systems

**RDS:**
- Delete Aurora cluster

**Estimated Module 4 Lab Cost (4 hours): ~$45-65**

---

**🎉 Great job! Ready for Module 5 - Compute Services!**

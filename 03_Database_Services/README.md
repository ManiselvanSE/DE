# Module 3: Database Services for Data Engineering

[![Difficulty](https://img.shields.io/badge/Difficulty-Intermediate--Advanced-red)]()
[![Duration](https://img.shields.io/badge/Duration-8--10%20hours-blue)]()
[![Cost](https://img.shields.io/badge/Cost-$50--100-orange)]()

## 📖 Module Overview

This module provides comprehensive coverage of AWS database services for data engineering workloads. You'll learn Aurora, Redshift, DynamoDB, and database migration strategies, with a special focus on Oracle to AWS migration for experienced DBAs.

## 🎯 What You Will Learn

### Amazon Aurora
- ✅ PostgreSQL/MySQL-compatible managed database
- ✅ Multi-AZ deployments and automatic failover
- ✅ Aurora Global Database (cross-region replication)
- ✅ Aurora Serverless v2 (auto-scaling)
- ✅ Read replicas (up to 15)
- ✅ Backtrack (point-in-time recovery)

### Amazon Redshift
- ✅ Columnar data warehouse (petabyte-scale)
- ✅ Distribution keys (DISTKEY: KEY, ALL, EVEN, AUTO)
- ✅ Sort keys (SORTKEY: compound, interleaved)
- ✅ Redshift Spectrum (query S3 without loading)
- ✅ Star schema design (fact + dimension tables)
- ✅ COPY command (bulk load from S3)
- ✅ Materialized views (query acceleration)

### Amazon DynamoDB
- ✅ NoSQL key-value and document database
- ✅ Partition key vs sort key design
- ✅ Global Secondary Indexes (GSI) vs Local Secondary Indexes (LSI)
- ✅ On-demand vs provisioned capacity modes
- ✅ DynamoDB Streams and triggers
- ✅ Global Tables (multi-region active-active)

### Database Migration
- ✅ AWS DMS (Database Migration Service)
- ✅ AWS SCT (Schema Conversion Tool)
- ✅ Full load vs CDC (Change Data Capture)
- ✅ Oracle to Aurora PostgreSQL migration
- ✅ Zero-downtime migration strategies

## 📚 Content Included

### Exercises
1. **Exercise 3.1**: Aurora PostgreSQL cluster with read replicas
2. **Exercise 3.2**: Redshift data warehouse with star schema
3. **Exercise 3.3**: DynamoDB for high-velocity IoT data

### Interview Questions (20 Total)

#### Beginner Questions (5)
1. OLTP vs OLAP comparison
2. Aurora vs RDS (when to use each)
3. Redshift DISTKEY and SORTKEY explained
4. DynamoDB partition key vs sort key
5. Oracle to Aurora migration (5-phase approach)

#### Intermediate Questions (5)
6. Aurora Global Database (disaster recovery, RTO < 1 min)
7. Redshift Spectrum vs native tables (hybrid approach)
8. DynamoDB GSI vs LSI (use cases, cost implications)
9. Redshift query performance monitoring
10. AWS DMS replication modes (full load, CDC, full load + CDC)

#### Scenario-Based Questions (10 - Deep Dive)

Each scenario question includes **2,000+ words** with complete architecture diagrams, implementation code, performance benchmarks, cost analysis, and production best practices.

## 📂 Files in This Section

- `Module_3_Database_Services.md` - Complete module content (153KB, 4,613 lines)
- `README.md` - This navigation file

## 💰 Cost Estimate

**Total Cost: $50-100**

⚠️ **CRITICAL**: This is the most expensive module. Run cleanup scripts immediately after exercises!

**Cost if NOT cleaned up**:
- Aurora: $1,138/month (3 instances running 24/7)
- Redshift: $8,500/month (cluster running 24/7)
- Total: **$9,638/month** if left running

## 🚀 Quick Start

```bash
# Set AWS region
export AWS_DEFAULT_REGION=us-east-1

# Verify IAM permissions
aws sts get-caller-identity

# Create Aurora cluster (Example 3.1)
aws rds create-db-cluster \
  --db-cluster-identifier learning-aurora \
  --engine aurora-postgresql \
  --master-username postgres \
  --master-user-password YourPassword123!

# Start with Exercise 3.1
# Open: Module_3_Database_Services.md
```

## 📈 Navigation

- **Previous**: [Module 2: Amazon S3 Foundations](../02_Amazon_S3_Foundations/)
- **Next**: Module 4 (Coming Soon)
- **Main**: [Back to DE Home](../)

---

**Ready to start? Open `Module_3_Database_Services.md`** 🚀

**⚠️ Remember**: Run cleanup scripts after completing exercises to avoid $9,600+/month charges!

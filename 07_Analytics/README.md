# Module 7: Analytics Services

## Overview

This module covers AWS analytics services for querying, processing, and visualizing data at scale. Learn serverless SQL with Athena, real-time streaming with Kinesis, and data governance with Lake Formation.

**Duration:** 8-10 hours  
**Difficulty:** Intermediate-Advanced  
**Prerequisites:** Modules 1-6, SQL fundamentals, data lake concepts

---

## Key Services Covered

- ✅ **Amazon Athena** - Serverless SQL on S3 ($5/TB scanned)
- ✅ **AWS Glue Data Catalog** - Central metadata repository
- ✅ **Amazon Kinesis** - Real-time data streaming (Streams, Firehose, Analytics)
- ✅ **Amazon QuickSight** - Business intelligence dashboards
- ✅ **AWS Lake Formation** - Data lake governance and security

---

## Module Structure

### Hands-On Exercises (3)

1. **Exercise 7.1: Serverless Analytics with Athena**
   - Query 1 PB data lake with partition pruning
   - Cost: $0.05 (vs $5,000 full scan)
   - Parquet conversion (10× compression)
   - Duration: ~90 minutes

2. **Exercise 7.2: Real-Time Streaming with Kinesis**
   - Ingest 1M IoT events/minute
   - Kinesis Data Analytics for aggregation
   - Firehose to S3 with format conversion
   - Duration: ~120 minutes

3. **Exercise 7.3: Data Lake Governance with Lake Formation**
   - Row-level and column-level security
   - Multi-account access control
   - Tag-based permissions
   - Duration: ~75 minutes

### Question Sets (20 Questions)

#### Beginner (Q1-Q5)
- Q1: Amazon Athena overview
- Q2: Glue Data Catalog and Crawlers
- Q3: Kinesis Streams vs Firehose vs Analytics
- Q4: QuickSight for BI dashboards
- Q5: Lake Formation basics

#### Intermediate (Q6-Q10)
- Q6: Athena query optimization (partitions, Parquet, compression)
- Q7: Kinesis shard management and scaling
- Q8: Athena vs Redshift Spectrum comparison
- Q9: QuickSight SPICE in-memory engine
- Q10: Lake Formation security model

#### Scenario-Based (Q11-Q20)
- **Q11:** Multi-tenant SaaS data lake (row-level security)
- **Q12:** Real-time fraud detection (95ms p99 latency)
- **Q13:** Cost optimization (10 PB, 95% savings)
- **Q14:** Streaming ETL with Kinesis Data Analytics
- **Q15:** Federated queries (S3 + RDS + DynamoDB)
- **Q16:** Embedded analytics with QuickSight
- **Q17:** GDPR compliance (data deletion, encryption)
- **Q18:** Sub-second dashboards (CTAS + SPICE)
- **Q19:** Streaming data quality (Firehose Lambda)
- **Q20:** Hybrid analytics (on-prem + cloud)

---

## Learning Outcomes

After completing this module:

1. **Optimize query costs**
   - Partitioning: 99.99% reduction (10 GB vs 1 PB)
   - Parquet: 10× compression (100 GB → 10 GB)
   - Compression (Snappy/Zstd): 3× reduction
   - Combined: $0.05 vs $5,000 per query

2. **Build real-time pipelines**
   - Kinesis Streams: 1M events/sec (100 shards)
   - Data Analytics: Sub-second aggregation
   - Firehose: S3/Redshift/Elasticsearch delivery
   - End-to-end latency: < 1 second

3. **Implement data governance**
   - Lake Formation: Row/column-level security
   - Tag-based permissions (ABAC)
   - Cross-account access control
   - Audit logging (CloudTrail)

4. **Create visualizations**
   - QuickSight: Interactive dashboards
   - SPICE: In-memory sub-second queries
   - Embedded analytics: $0.30/session
   - Auto-refresh: Real-time updates

---

## Real-World Use Cases

### Serverless Data Lake (Athena)
- **E-commerce analytics:** 1 PB clickstream data
- **Cost:** $0.05/query (vs $5,000 unoptimized)
- **Performance:** 3 seconds (vs 5 minutes)
- **Savings:** 99.99% (partition pruning + Parquet)

### Real-Time Streaming (Kinesis)
- **IoT sensor monitoring:** 1M events/minute
- **Latency:** 200ms ingestion, 500ms processing
- **Scale:** 100 shards auto-scaling
- **Cost:** $216/month (vs $2,400 self-managed)

### Data Governance (Lake Formation)
- **Multi-tenant SaaS:** 500 customers
- **Row-level security:** Automatic query filtering
- **Compliance:** GDPR, HIPAA, SOC 2
- **Admin overhead:** 95% reduction vs manual grants

### Business Intelligence (QuickSight)
- **Executive dashboards:** 1,000 users
- **Query performance:** < 1 second (SPICE)
- **Cost:** $9/user/month (Enterprise)
- **Embedded:** $0.30/session for customers

---

## Key Metrics Achieved

| Metric | Result |
|--------|--------|
| **Cost Savings** | 99.99% (partition pruning) |
| **Compression** | 10× (Parquet vs CSV) |
| **Query Speed** | 10× faster (columnar format) |
| **Stream Throughput** | 1M events/sec (Kinesis) |
| **Dashboard Latency** | < 1 second (SPICE) |
| **Data Governance** | 95% less admin (Lake Formation) |

---

## Service Comparison

| Feature | Athena | Redshift Spectrum | EMR Presto |
|---------|--------|-------------------|------------|
| **Deployment** | Serverless | Cluster required | Cluster required |
| **Cost Model** | $5/TB scanned | Cluster + $5/TB | EC2 instances |
| **Concurrency** | High (100+ users) | Medium (50 slots) | Medium (config) |
| **Best For** | Ad-hoc queries | Heavy analytics | Custom frameworks |
| **Query Latency** | 1-10 seconds | < 1 second | 1-5 seconds |
| **Management** | None | Medium | High |

### Kinesis Components

| Service | Use Case | Latency | Cost |
|---------|----------|---------|------|
| **Data Streams** | Custom processing | 200ms | $0.015/shard-hour |
| **Firehose** | S3/Redshift load | 60 seconds | $0.029/GB |
| **Data Analytics** | SQL on streams | 1 second | $0.11/KPU-hour |

---

## Cost Optimization Strategies

### Athena Query Optimization

```
Optimization           | Data Scanned | Cost/Query | Savings
-----------------------|--------------|------------|--------
Unoptimized (CSV)      | 1 PB         | $5,000     | Baseline
+ Partitioning         | 10 GB        | $0.05      | 99.99%
+ Parquet              | 1 GB         | $0.005     | 99.9999%
+ Compression (Snappy) | 330 MB       | $0.00165   | 99.99997%
```

### Data Lake Lifecycle

```
Tier                  | Days | Cost/TB/Month | Use Case
----------------------|------|---------------|----------
S3 Standard           | 0-30 | $23           | Active queries
Intelligent-Tiering   | 30+  | $4-23         | Variable access
Glacier Deep Archive  | 180+ | $1            | Compliance
```

**10 PB data lake savings:** $230,000 → $10,000/month (95%)

---

## Files in This Module

- **Module_7_Analytics.md** (50 KB, 1,766 lines)
  - Complete module content
  - 3 hands-on exercises
  - 20 questions with answers
  - SQL queries and code examples

---

## Next Steps

After Module 7:

1. **Practice:** Run Athena queries with partitions and Parquet
2. **Experiment:** Set up Kinesis stream with real-time analytics
3. **Implement:** Lake Formation row-level security
4. **Continue:** Module 8: Application Integration (EventBridge, SQS, SNS, Step Functions)

---

**Module Author:** Claude Sonnet 4.5  
**Last Updated:** June 2026  
**Version:** 1.0

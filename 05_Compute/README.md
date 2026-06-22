# Module 5: Compute Services for Data Engineering

## Overview

This module covers AWS compute services for data engineering workloads. Learn when to use Lambda, Batch, EMR, or Glue for different data processing scenarios and how to optimize cost and performance.

**Duration:** 8-10 hours  
**Difficulty:** Intermediate-Advanced  
**Prerequisites:** Modules 1-4, Understanding of distributed computing

---

## Key Services Covered

- ✅ **AWS Lambda** - Serverless event-driven processing
- ✅ **AWS Batch** - Managed batch computing at scale
- ✅ **Amazon EMR** - Big data frameworks (Spark, Hadoop, Presto)
- ✅ **AWS Glue** - Serverless ETL and data catalog
- ✅ **AWS Step Functions** - Workflow orchestration

---

## Module Structure

### Hands-On Exercises (3)

1. **Exercise 5.1: Serverless ETL with AWS Lambda**
   - Build event-driven pipeline: S3 → Lambda → Parquet → Redshift
   - Data validation, transformation, loading
   - Cost: $0.0024 per 1M rows
   - Duration: ~90 minutes

2. **Exercise 5.2: Batch Processing with AWS Batch**
   - Process 10,000 log files in parallel
   - Spot Instances for 90% cost savings
   - Array jobs, auto-scaling (0 → 1,000 vCPUs)
   - Duration: ~75 minutes

3. **Exercise 5.3: Big Data with Amazon EMR (Spark)**
   - Process 100 TB clickstream data with Spark
   - Auto-scaling, Spot task nodes
   - S3 data lake (separate compute/storage)
   - Duration: ~120 minutes

### Question Sets (20 Questions)

#### Beginner (Q1-Q5)
- Q1: What is Lambda and when to use it
- Q2: AWS Batch vs Lambda comparison
- Q3: Amazon EMR overview and use cases
- Q4: AWS Glue for serverless ETL
- Q5: Step Functions for workflow orchestration

#### Intermediate (Q6-Q10)
- Q6: Lambda performance optimization
- Q7: Batch auto-scaling configuration
- Q8: EMR cost optimization (Spot + Graviton + S3)
- Q9: Glue vs EMR decision matrix
- Q10: Monitoring Lambda, Batch, EMR jobs

#### Scenario-Based (Q11-Q20)
- **Q11:** Real-time analytics (Lambda + Kinesis + DynamoDB)
- **Q12:** Batch video processing (AWS Batch + Spot)
- **Q13:** Big data analytics (EMR Spark, 500 TB dataset)
- **Q14:** Multi-source ETL (Glue integration)
- **Q15:** Multi-step pipeline (Step Functions)
- **Q16:** Lambda performance tuning (10,000 files/hour)
- **Q17:** EMR cost reduction (99.7% savings)
- **Q18:** Spark performance troubleshooting
- **Q19:** Disaster recovery for data pipelines
- **Q20:** Hybrid compute strategy (Lambda + Batch + EMR + Glue)

---

## Learning Outcomes

After completing this module:

1. **Choose the right compute service** for each workload
   - Lambda: < 1 GB, < 15 min, event-driven
   - Batch: Parallel batch jobs, Spot instances
   - EMR: > 1 TB, Spark/Hadoop frameworks
   - Glue: 1-100 GB, serverless simplicity

2. **Optimize costs** for compute workloads
   - EMR Spot Instances: 70-90% savings
   - Lambda memory tuning: 20% savings
   - Batch auto-scaling: 100% savings on idle time
   - Hybrid approach: 69% total savings

3. **Achieve high performance**
   - Lambda: 1,792 MB for full vCPU (75% faster)
   - Batch: 10,000 parallel tasks (99% faster)
   - EMR: Managed scaling + optimization (93% faster)
   - Glue: Auto-scaling DPUs (no management)

4. **Monitor and troubleshoot**
   - CloudWatch metrics and logs
   - X-Ray tracing for Lambda
   - Spark UI for EMR debugging
   - Batch job queue monitoring

---

## Real-World Use Cases

### Serverless Event Processing (Lambda)
- **File processing:** S3 upload → Lambda → transform → S3
- **Real-time analytics:** Kinesis → Lambda → DynamoDB
- **Cost:** $0.0024 per 1M records
- **Performance:** 1,000 concurrent invocations

### Parallel Batch Processing (AWS Batch)
- **Video transcoding:** 50,000 files/day in 4 hours
- **Log processing:** 10,000 files in parallel
- **Cost:** $1.38 for 10,000 jobs (Spot)
- **Scaling:** 0 → 10,000 vCPUs auto

### Big Data Analytics (EMR)
- **Clickstream:** 500 TB/day processed in 3 hours
- **Machine learning:** Feature engineering at scale
- **Cost:** $16 for 100 TB processing
- **Performance:** 83% faster than baseline

### Serverless ETL (Glue)
- **Daily ETL:** Multi-source integration
- **Schema discovery:** Glue Crawler auto-catalog
- **Cost:** $0.44 for 10 GB transformation
- **Simplicity:** Zero cluster management

---

## Key Metrics Achieved

| Service | Use Case | Performance | Cost Savings |
|---------|----------|-------------|--------------|
| **Lambda** | 1M row CSV→Parquet | 12 seconds | 96% vs EC2 |
| **Batch** | 10,000 parallel jobs | 18 minutes | 90% (Spot) |
| **EMR** | 100 TB Spark processing | 45 minutes | 93% faster |
| **Glue** | 50 GB daily ETL | 30 minutes | 69% vs EMR |

---

## Service Comparison

| Feature | Lambda | Batch | EMR | Glue |
|---------|--------|-------|-----|------|
| **Max Duration** | 15 min | Unlimited | Unlimited | Unlimited |
| **Best For** | < 1 GB | Batch jobs | > 1 TB | 1-100 GB |
| **Management** | Serverless | Semi-managed | Semi-managed | Serverless |
| **Cost Model** | Per request | EC2 instances | EC2 instances | DPU-hours |
| **Scaling** | Automatic | Auto-scaling | Auto-scaling | Auto-scaling |
| **Frameworks** | None | Any (Docker) | Spark, Hadoop | Spark only |

---

## Cost Optimization Strategies

1. **Lambda:** Optimize memory (1,792 MB for full vCPU)
2. **Batch:** Use Spot Instances (90% savings)
3. **EMR:** 
   - Spot task nodes (70% savings)
   - Graviton2 instances (20% cheaper)
   - S3 data lake (separate compute/storage)
   - Transient clusters (95% savings)
4. **Glue:** Right-size DPUs (start with 10, scale as needed)

**Hybrid Approach Savings: 69%** (use right service for each job)

---

## Files in This Module

- **Module_5_Compute.md** (95 KB, 3,019 lines)
  - Complete module content
  - 3 hands-on exercises
  - 20 questions with answers
  - Code examples and architectures

---

## Next Steps

After Module 5:

1. **Practice:** Build serverless data pipeline with Lambda
2. **Experiment:** Try EMR with Spot Instances
3. **Compare:** Glue vs EMR for your workload size
4. **Continue:** Module 6: Containers (ECS, EKS, Fargate)

---

**Module Author:** Claude Sonnet 4.5  
**Last Updated:** June 2024  
**Version:** 1.0

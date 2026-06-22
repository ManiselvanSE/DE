# Module 6: Containers for Data Engineering

## Overview

This module covers AWS container orchestration services for running scalable, portable data engineering workloads. Learn when to use ECS vs EKS vs Fargate and how to optimize costs with Spot Instances.

**Duration:** 8-10 hours  
**Difficulty:** Advanced  
**Prerequisites:** Modules 1-5, Docker fundamentals, Kubernetes basics

---

## Key Services Covered

- ✅ **Amazon ECS** - AWS-native container orchestration
- ✅ **Amazon EKS** - Managed Kubernetes for complex workflows
- ✅ **AWS Fargate** - Serverless compute for containers
- ✅ **Amazon ECR** - Private Docker registry
- ✅ **Spot Instances** - 70-90% cost savings for containers

---

## Module Structure

### Hands-On Exercises (3)

1. **Exercise 6.1: Containerized ETL with ECS Fargate**
   - Serverless data pipeline (S3 → Container → Redshift)
   - EventBridge scheduling
   - Secrets Manager integration
   - Cost: $0.81 per run
   - Duration: ~90 minutes

2. **Exercise 6.2: Kubernetes Data Pipeline with EKS**
   - Deploy Apache Airflow on EKS
   - Multi-node auto-scaling
   - GitOps deployment
   - High availability architecture
   - Duration: ~120 minutes

3. **Exercise 6.3: Cost-Optimized Batch with ECS Spot**
   - Process 100,000 images/day
   - Spot Instances (90% savings)
   - SQS-based auto-scaling
   - Cost: $4.08/day
   - Duration: ~75 minutes

### Question Sets (20 Questions)

#### Beginner (Q1-Q5)
- Q1: ECS vs EKS comparison
- Q2: Fargate vs EC2 launch types
- Q3: Amazon ECR overview
- Q4: ECS auto-scaling implementation
- Q5: Task roles vs execution roles

#### Intermediate (Q6-Q10)
- Q6: Container image optimization
- Q7: Blue-green deployments with ECS
- Q8: Cost comparison (Fargate vs EC2 vs Spot)
- Q9: Monitoring containerized apps
- Q10: Secrets management (ECS/EKS)

#### Scenario-Based (Q11-Q20)
- **Q11:** Multi-region DR (RPO < 15 min, RTO < 30 min)
- **Q12:** CI/CD pipeline for containers
- **Q13:** Cost optimization (96% savings)
- **Q14:** Spark on EKS with auto-scaling
- **Q15:** Multi-tenant container platform
- **Q16:** Real-time stream processing (1M events/sec)
- **Q17:** GPU-accelerated ML training
- **Q18:** Vulnerability management (ECR scanning)
- **Q19:** Service mesh with App Mesh
- **Q20:** Hybrid deployment (on-prem + cloud)

---

## Learning Outcomes

After completing this module:

1. **Choose the right container service**
   - ECS: AWS-native, simpler, free control plane
   - EKS: Kubernetes ecosystem, complex workflows
   - Fargate: Serverless, variable workloads
   - EC2: Large steady workloads, Spot savings

2. **Optimize container costs**
   - Fargate: Best for variable workloads (scale to zero)
   - EC2 Reserved: 44% cheaper for steady 24/7
   - Spot Instances: 90% savings for fault-tolerant
   - Auto-scaling: 100% savings on idle time

3. **Implement production patterns**
   - Blue-green deployments (zero downtime)
   - Auto-scaling (CPU, memory, custom metrics)
   - Secrets management (Secrets Manager, K8s External Secrets)
   - Multi-region DR (RPO < 15 min)

4. **Optimize images and deployments**
   - Multi-stage builds (70% smaller images)
   - Layer caching (90% faster rebuilds)
   - ECR vulnerability scanning (automated)
   - CI/CD pipelines (15-minute deployments)

---

## Real-World Use Cases

### Serverless ETL (ECS Fargate)
- **Daily sales processing:** 5 GB CSV → Parquet → Redshift
- **Cost:** $1.38/month (vs $120 EC2 24/7)
- **Savings:** 99% (serverless, no idle cost)

### Kubernetes Orchestration (EKS)
- **Apache Airflow:** Complex DAG workflows
- **High Availability:** Multi-replica scheduler/webserver
- **Auto-scaling:** 3-20 workers based on queue
- **Cost:** $300/month (vs $500 EC2-based)

### Cost-Optimized Batch (ECS Spot)
- **Image processing:** 100,000 images/day
- **Spot Instances:** 90% discount
- **Auto-scaling:** 0-500 tasks
- **Cost:** $4.08/day (vs $40.32 On-Demand)

### GPU ML Training (EKS)
- **Deep learning:** p3.8xlarge (4 × V100 GPUs)
- **Spot Instances:** $12.24/hour (vs $40.96 On-Demand)
- **Training time:** 3 hours/model
- **Cost:** $36/model (70% savings)

---

## Key Metrics Achieved

| Metric | Result |
|--------|--------|
| **Cost Savings** | 90% (Spot), 99% (Fargate idle) |
| **Image Size** | 70% smaller (multi-stage builds) |
| **Build Speed** | 90% faster (layer caching) |
| **Deployment** | 15 minutes (CI/CD automation) |
| **Scaling** | 0 → 500 tasks (auto-scaling) |
| **Uptime** | 99.9% (multi-AZ, auto-healing) |

---

## Service Comparison

| Feature | ECS | EKS | Fargate |
|---------|-----|-----|---------|
| **Control Plane** | Free | $73/month | Free |
| **Learning Curve** | Easy | Hard | Easy |
| **Portability** | AWS-only | Multi-cloud | AWS-only |
| **Best For** | Simple apps | K8s ecosystem | Variable loads |
| **Cost (steady)** | Low (EC2 Reserved) | Medium | High |
| **Cost (variable)** | Medium | Medium | Low |
| **Management** | Low | High | None |

---

## Cost Optimization Strategies

### Fargate vs EC2 Breakpoints

```
Workload Pattern          | Best Choice       | Monthly Cost
--------------------------|-------------------|-------------
Variable (0-50 tasks)     | Fargate          | $1,000 avg
Steady 24/7 (50 tasks)    | EC2 Reserved     | $1,980
Batch processing          | EC2 Spot         | $306
Dev/Test (8hrs/day)       | Fargate          | $1,185
```

### Hybrid Approach

```
Baseline: 10 tasks EC2 Reserved ($396/month)
Peak: 0-40 tasks Fargate ($0-$2,844, avg $1,000/month)
Total: $1,396/month

Savings: 61% vs all-Fargate, 30% vs all-EC2 ✅
```

---

## Files in This Module

- **Module_6_Containers.md** (71 KB, 2,323 lines)
  - Complete module content
  - 3 hands-on exercises
  - 20 questions with answers
  - Production architectures

---

## Next Steps

After Module 6:

1. **Practice:** Deploy containerized app on ECS Fargate
2. **Experiment:** Try EKS with Spot Instances
3. **Compare:** ECS vs EKS for your workload
4. **Continue:** Module 7: Analytics (Athena, Glue, Kinesis, QuickSight)

---

**Module Author:** Claude Sonnet 4.5  
**Last Updated:** June 2024  
**Version:** 1.0

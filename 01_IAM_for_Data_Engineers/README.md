# Module 1: IAM for Data Engineers

[![Difficulty](https://img.shields.io/badge/Difficulty-Beginner-green)]()
[![Duration](https://img.shields.io/badge/Duration-3--4%20hours-blue)]()
[![Cost](https://img.shields.io/badge/Cost-$0%20(Free%20Tier)-brightgreen)]()

## 📖 Module Overview

This module covers AWS Identity and Access Management (IAM) from a Data Engineering perspective. You'll learn how to secure data pipelines, implement least privilege access, and manage permissions across multiple AWS services.

## 🎯 What You Will Learn

- ✅ IAM users, groups, roles, and policies
- ✅ Managed vs customer managed vs inline policies
- ✅ Cross-account access with IAM roles
- ✅ Multi-factor authentication (MFA)
- ✅ Lake Formation row-level and column-level security
- ✅ Tag-based access control (ABAC)
- ✅ Just-in-time (JIT) access patterns
- ✅ Service roles for Glue, Lambda, EMR

## 📚 Content Included

### Exercises
1. **Exercise 1.1**: Create IAM users for data engineering team
2. **Exercise 1.2**: Implement cross-account S3 access
3. **Exercise 1.3**: Configure Lake Formation fine-grained access control

### Interview Questions (20 Total)
- **5 Beginner Questions**: What is IAM, Users vs Roles, Policies, Least Privilege, MFA
- **5 Intermediate Questions**: Managed policies, IAM roles, troubleshooting, permission evaluation, least privilege implementation
- **10 Scenario-Based Questions**: 
  - Cross-account data lake access
  - Glue job access denied troubleshooting
  - Multi-environment IAM structure (dev/staging/prod)
  - GDPR-compliant PII security (7-layer defense)
  - Oracle to AWS database migration IAM strategy
  - Time-limited access workflow (JIT)
  - Multi-team data isolation (tag-based)
  - Row-level & column-level security (Lake Formation)
  - KMS encryption troubleshooting
  - Infrastructure-as-code IAM (Terraform/CloudFormation)

## 📂 Files in This Section

- `Module_1_IAM_for_Data_Engineers.md` - Complete module content (98KB, 2,966 lines)
- `README.md` - This navigation file

## 🚀 Quick Start

```bash
# Configure AWS CLI
aws configure

# Verify IAM permissions
aws iam get-user

# Start with Exercise 1.1
# Open: Module_1_IAM_for_Data_Engineers.md
```

## 📈 Next Module

After completing this module, proceed to:
**[Module 2: Amazon S3 Foundations](../02_Amazon_S3_Foundations/)**

---

**Ready to start? Open `Module_1_IAM_for_Data_Engineers.md`** 🚀

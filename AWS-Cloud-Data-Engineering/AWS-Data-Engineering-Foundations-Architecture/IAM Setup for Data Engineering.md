# IAM Setup for Data Engineering

---

## PART 0: Understanding Why This Matters

### The Big Picture

Imagine a company with:

- Data Engineers
- Analysts
- Database Administrators
- Finance Teams
- Security Teams

Without access control:

- An analyst could accidentally delete production data.
- Interns could access confidential financial records.
- Hackers could gain full access if credentials leak.
- Nobody would know who changed what.

This is why IAM exists.

---

## What is IAM?

**IAM = Identity and Access Management**

Think of AWS like a secure office building:

| AWS Concept | Real-World Analogy |
|-------------|-------------------|
| AWS Account | Office Building |
| Cloud Resources | Rooms |
| IAM Users | Employees |
| IAM Roles | Job Titles |
| IAM Policies | Access Rules |

### Without IAM

Everyone has full admin access.

Result:
- High security risk
- Accidental deletions
- Compliance failures

### With IAM

Users and services get **only** the permissions they need.

This follows the **Principle of Least Privilege** — give the minimum permissions necessary.

---

## Real-World Example

| Role | Permissions |
|------|-------------|
| DataEngineerRole | Build pipelines, manage S3, Glue, Redshift |
| GlueServiceRole | Read/write S3 for Glue jobs |
| AnalystReadOnlyRole | Query data only |
| LambdaExecutionRole | Execute serverless processing |
| RedshiftIAMRole | Allow Redshift to access S3 |

---

## Why This Matters for Your Career

You will use IAM in:

- Real production systems
- Security audits
- Compliance environments
- Technical interviews
- Cloud architecture design

---

## PART 1: Lab Goals

By the end of this lab, you will:

- Understand IAM concepts
- Create multiple IAM roles
- Apply least privilege security
- Create custom IAM policies
- Configure AWS service permissions
- Document IAM architecture

---

## PART 2: What You Will Create

### Role 1: DataEngineerRole

**Purpose:** Main role for data engineers.

**Permissions:**
- AmazonS3FullAccess
- AWSGlueFullAccess
- AmazonRedshiftFullAccess
- AmazonEMRFullAccessPolicy_v2
- AmazonKinesisFullAccess
- AWSLambdaFullAccess
- CloudWatchLogsFullAccess

**Use Case:** Daily engineering work.

---

### Role 2: GlueServiceRole

**Purpose:** Used by AWS Glue jobs.

**Permissions:**
- AWSGlueServiceRole
- AmazonS3FullAccess
- CloudWatchLogsFullAccess
- SecretsManagerReadWrite

**Use Case:** Glue ETL processing.

---

### Role 3: LambdaExecutionRole

**Purpose:** Used by Lambda functions.

**Permissions:**
- AWSLambdaBasicExecutionRole
- AmazonS3FullAccess
- AmazonDynamoDBFullAccess
- AmazonKinesisFullAccess
- SecretsManagerReadWrite

**Use Case:** Serverless data processing.

---

### Role 4: RedshiftIAMRole

**Purpose:** Allows Redshift to access S3.

**Permissions:**
- AmazonS3FullAccess
- CloudWatchLogsFullAccess

**Use Case:** COPY commands from S3 into Redshift.

---

### Role 5: AnalystReadOnlyRole

**Purpose:** Read-only access for analysts.

**Permissions:**
- AmazonAthenaFullAccess
- AmazonRedshiftReadOnlyAccess
- AmazonQuickSightReadOnlyAccess
- AmazonS3ReadOnlyAccess

**Use Case:** Analytics and dashboards.

---

### Bonus Custom Policy: DataLakeBucketAccessPolicy

**Purpose:** Restrict access to `data-lake-*` buckets only.

**Security Enforcement:** Blocks unencrypted uploads.

**Compliance Benefit:** Supports GDPR, HIPAA, and security best practices.

---

## PART 3: Why These Decisions Matter

### Why Separate Roles?

**Bad Practice:** Everyone gets admin access.
- Result: One mistake can destroy infrastructure.

**Good Practice:** Different teams get specific permissions.
- Result: Reduced risk, better security, easier auditing.

### Why Restrict Services?

If a service is compromised:

- **Bad:** Attacker gains full AWS access.
- **Good:** Attacker can only access limited resources.

### Why Enforce Encryption?

Policy blocks uploads unless encrypted.

Benefits:
- Compliance
- Security
- Data protection

---

## Prerequisites

Before starting:

- AWS Account
- AWS Console access
- IAM admin permissions
- Text editor
- 3 hours available

AWS Console: [https://console.aws.amazon.com](https://console.aws.amazon.com)

---

## STEP 0: Login to AWS

### Step 0.1

Open: [https://console.aws.amazon.com](https://console.aws.amazon.com)

### Step 0.2

Enter:
- Email
- Password
- MFA code

Click: **Sign In**

### Step 0.3

Verify successful login. You should see:
- AWS Console dashboard
- Account name at top-right

---

## PART 4: Create IAM Roles

### Role 1: DataEngineerRole

#### Step 1.1: Open IAM Console
- Open AWS Console
- Search: IAM
- Open IAM service

#### Step 1.2: Create Role
- Click: **Roles**
- Click: **Create role**

#### Step 1.3: Select Trusted Entity
- Choose: AWS Service → EC2
- Click: **Next**

#### Step 1.4: Add Permissions

Attach these policies:

| Policy |
|--------|
| AmazonS3FullAccess |
| AWSGlueFullAccess |
| AmazonRedshiftFullAccess |
| AmazonEMRFullAccessPolicy_v2 |
| AmazonKinesisFullAccess |
| AWSLambdaFullAccess |
| CloudWatchLogsFullAccess |

#### Step 1.5: Name the Role

- **Role Name:** `DataEngineerRole`
- **Description:** Role for data engineers to access S3, Glue, Redshift, EMR, Kinesis, Lambda, and CloudWatch

Click: **Create role**

---

### Role 2: GlueServiceRole

**Trusted Entity:** AWS Service → Glue

**Policies:**

| Policy |
|--------|
| AWSGlueServiceRole |
| AmazonS3FullAccess |
| CloudWatchLogsFullAccess |
| SecretsManagerReadWrite |

**Role Name:** `GlueServiceRole`

---

### Role 3: LambdaExecutionRole

**Trusted Entity:** AWS Service → Lambda

**Policies:**

| Policy |
|--------|
| AWSLambdaBasicExecutionRole |
| AmazonS3FullAccess |
| AmazonDynamoDBFullAccess |
| AmazonKinesisFullAccess |
| SecretsManagerReadWrite |

**Role Name:** `LambdaExecutionRole`

---

### Role 4: RedshiftIAMRole

**Trusted Entity:** AWS Service → Redshift

**Policies:**

| Policy |
|--------|
| AmazonS3FullAccess |
| CloudWatchLogsFullAccess |

**Role Name:** `RedshiftIAMRole`

---

### Role 5: AnalystReadOnlyRole

**Trusted Entity:** AWS Service → EC2

**Policies:**

| Policy |
|--------|
| AmazonAthenaFullAccess |
| AmazonRedshiftReadOnlyAccess |
| AmazonQuickSightReadOnlyAccess |
| AmazonS3ReadOnlyAccess |

**Role Name:** `AnalystReadOnlyRole`

---

## PART 5: Create Custom Policy

### Step 6: Open Policies

- IAM Console
- Click: **Policies**
- Click: **Create policy**

### Step 7: JSON Policy

Select: **JSON**

Paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListDataLakeBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::data-lake-*"
    },
    {
      "Sid": "ReadWriteDataLakeObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::data-lake-*/*"
    },
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::data-lake-*/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

**Policy Name:** `DataLakeBucketAccessPolicy`

**Description:** Custom policy to access data lake S3 bucket with encryption enforcement.

Click: **Create policy**

---

## PART 6: Verify Your Work

### Verify Roles

Go to: **IAM → Roles**

You should see:
- DataEngineerRole
- GlueServiceRole
- LambdaExecutionRole
- RedshiftIAMRole
- AnalystReadOnlyRole

### Verify Policies

Open: **DataEngineerRole**

Ensure all 7 policies are attached.

---

## PART 7: Documentation

Create: `Lab_1_1_IAM_Setup.txt`

Add:

```
=== LAB 1.1: IAM SETUP DOCUMENTATION ===

Date Created:
AWS Account ID:

ROLES CREATED:

1. DataEngineerRole
2. GlueServiceRole
3. LambdaExecutionRole
4. RedshiftIAMRole
5. AnalystReadOnlyRole

CUSTOM POLICY:
DataLakeBucketAccessPolicy
```

---

## Success Checklist

| Task | Status |
|------|--------|
| DataEngineerRole created | ✓ |
| GlueServiceRole created | ✓ |
| LambdaExecutionRole created | ✓ |
| RedshiftIAMRole created | ✓ |
| AnalystReadOnlyRole created | ✓ |
| Custom policy created | ✓ |
| Roles verified | ✓ |
| Documentation saved | ✓ |

---

## Cleanup

IAM roles are **FREE**.

Do **NOT** delete them if:
- Continuing future labs
- Using them later

Safe to keep: **$0/month cost**

---

## What You Learned

- IAM fundamentals
- Least privilege principle
- AWS managed policies
- Custom IAM policies
- Service roles
- Security best practices
- Role-based access control
- Encryption enforcement

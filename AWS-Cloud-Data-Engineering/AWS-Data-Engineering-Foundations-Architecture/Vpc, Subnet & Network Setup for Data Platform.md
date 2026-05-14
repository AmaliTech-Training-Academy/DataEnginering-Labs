# VPC, Subnets & Network Setup for Data Platform

---

## PART 0 — Understanding Why Networking Matters

### The Big Picture

Imagine you are building a data engineering platform for a major bank.

**The Bank's Requirements:**

- Customer financial data must never be exposed to the internet
- Database servers must be locked down
- Only approved applications can access databases
- All access must be auditable
- Data must travel securely and be encrypted

### What Happens Without Proper Networking?

**Bad Scenario — No VPC Security:**
- Database port accidentally exposed to the internet
- Hackers scan and discover open port
- Database becomes accessible
- Customer data is stolen
- Company suffers financial and reputational damage

**Good Scenario — Secure VPC Design:**
- Database placed in a private subnet
- No direct internet access exists
- Attackers cannot reach the database
- Data remains protected
- Infrastructure remains secure and compliant

### Real-World Security Breaches

| Company | Year | Impact |
|---------|------|--------|
| Target | 2013 | 40M credit cards stolen |
| Yahoo | 2016 | 500M accounts compromised |
| Equifax | 2017 | 147M records exposed |

---

## What is a VPC?

**VPC = Virtual Private Cloud**

A VPC is your private network inside AWS.

### Example Architecture

```
AWS Account
│
└── VPC (10.0.0.0/16)
    │
    ├── Public Subnet (10.0.1.0/24)
    │   └── NAT Gateway
    │
    ├── Private Subnet 1 (10.0.2.0/24)
    │   └── Databases
    │
    ├── Private Subnet 2 (10.0.3.0/24)
    │   └── Applications
    │
    ├── Internet Gateway
    │
    ├── VPC Endpoints
    │   ├── S3 Endpoint
    │   ├── DynamoDB Endpoint
    │   └── Secrets Manager Endpoint
    │
    ├── Security Groups
    │
    └── Route Tables
```

---

## Why This Lab Matters

### Career Importance

**Every Company Uses VPCs:**
- AWS
- Google Cloud
- Azure
- Enterprise environments

**Networking Skills Increase Value:**
- Security engineers are highly paid
- Cloud networking is foundational
- Preventing breaches protects businesses

**Real Interview Skill:**

You will eventually be asked: *"Design a secure cloud network architecture."*

This lab teaches exactly that.

---

## PART 1 — Lab Objectives

By the end of this lab, you will:

- Understand AWS networking fundamentals
- Create a production-ready VPC
- Configure public and private subnets
- Create secure route tables
- Implement NAT and VPC Endpoints
- Configure security groups
- Optimize networking costs

---

## PART 2 — Target Architecture

### Final Network Design

```
VPC: data-platform-vpc (10.0.0.0/16)

├── PUBLIC SUBNET (10.0.1.0/24)
│   └── NAT Gateway
│
├── PRIVATE SUBNET 1 (10.0.2.0/24)
│   └── RDS Databases
│
├── PRIVATE SUBNET 2 (10.0.3.0/24)
│   └── EC2 / Lambda / Glue
│
├── INTERNET GATEWAY
│
├── ROUTE TABLES
│   ├── Public → IGW
│   └── Private → NAT
│
├── SECURITY GROUPS
│   ├── sg-public-nat
│   ├── sg-private-compute
│   └── sg-private-db
│
└── VPC ENDPOINTS
    ├── S3 Gateway
    ├── DynamoDB Gateway
    └── Secrets Manager Interface
```

---

## PART 3 — Design Decisions

### Why Public and Private Subnets?

**Bad Design:**
```
Internet ↔ Databases ↔ Applications
```

**Good Design:**
```
Internet → Load Balancer → Applications → Databases
```

This creates layered security.

### Why Multiple Availability Zones?

Using multiple AZs provides redundancy.

```
us-east-1a
├── Public Subnet
└── Private Subnet 1

us-east-1b
└── Private Subnet 2
```

If one AZ fails, the other continues operating.

### Why VPC Endpoints?

**Without Endpoints:**
```
Private Server → NAT → Internet → S3
```

**With Endpoints:**
```
Private Server → VPC Endpoint → S3
```

**Benefits:**
- Lower cost
- Faster traffic
- More secure
- No internet exposure

---

## PART 4 — Prerequisites

Before beginning, ensure you have:

- AWS account access
- IAM roles from Lab 1.1
- AWS Console access
- Region set to `us-east-1`
- Text editor for documentation
- 4 hours available

---

## PART 5 — Create the VPC

### Step 1 — Open VPC Console

- AWS Console
- Search: **VPC**
- Open VPC Dashboard

### Step 2 — Create VPC

| Setting | Value |
|---------|-------|
| Name | `data-platform-vpc` |
| CIDR | `10.0.0.0/16` |
| IPv6 | None |
| Tenancy | Default |

---

## PART 6 — Create Subnets

### Public Subnet

| Setting | Value |
|---------|-------|
| Name | `public-subnet-1a` |
| AZ | `us-east-1a` |
| CIDR | `10.0.1.0/24` |

### Private Subnet 1

| Setting | Value |
|---------|-------|
| Name | `private-subnet-1a` |
| AZ | `us-east-1a` |
| CIDR | `10.0.2.0/24` |

### Private Subnet 2

| Setting | Value |
|---------|-------|
| Name | `private-subnet-1b` |
| AZ | `us-east-1b` |
| CIDR | `10.0.3.0/24` |

---

## PART 7 — Create Internet Gateway

| Setting | Value |
|---------|-------|
| Name | `data-platform-igw` |

**Attach to:** `data-platform-vpc`

---

## PART 8 — Create Elastic IP

Allocate an Elastic IP for the NAT Gateway.

Save:
- Elastic IP Address
- Allocation ID

---

## PART 9 — Create NAT Gateway

| Setting | Value |
|---------|-------|
| Name | `data-platform-nat` |
| Subnet | `public-subnet-1a` |
| Elastic IP | Previously allocated EIP |

---

## PART 10 — Create Route Tables

### Public Route Table

| Destination | Target |
|-------------|--------|
| `10.0.0.0/16` | Local |
| `0.0.0.0/0` | Internet Gateway |

**Associate:** `public-subnet-1a`

### Private Route Table

| Destination | Target |
|-------------|--------|
| `10.0.0.0/16` | Local |
| `0.0.0.0/0` | NAT Gateway |

**Associate:** `private-subnet-1a`, `private-subnet-1b`

---

## PART 11 — Create Security Groups

### SG 1 — Public NAT SG

| Setting | Value |
|---------|-------|
| Name | `sg-public-nat` |
| Rule | HTTPS from `0.0.0.0/0` |

### SG 2 — Private Compute SG

**Rules:**
- Allow all traffic from itself
- Allow all traffic from `sg-public-nat`

### SG 3 — Private Database SG

| Port | Source |
|------|--------|
| 3306 (MySQL) | `sg-private-compute` |
| 5432 (PostgreSQL) | `sg-private-compute` |

---

## PART 12 — Create VPC Endpoints

### S3 Gateway Endpoint

| Setting | Value |
|---------|-------|
| Service | `com.amazonaws.us-east-1.s3` |
| Type | Gateway |
| Route Table | `private-route-table` |

### DynamoDB Gateway Endpoint

| Setting | Value |
|---------|-------|
| Service | `com.amazonaws.us-east-1.dynamodb` |
| Type | Gateway |

### Secrets Manager Interface Endpoint

| Setting | Value |
|---------|-------|
| Service | `com.amazonaws.us-east-1.secretsmanager` |
| Type | Interface |
| Subnets | `private-subnet-1a`, `private-subnet-1b` |
| Security Group | `sg-private-compute` |

---

## PART 13 — Verification Checklist

### VPC
- [ ] `data-platform-vpc` exists

### Subnets
- [ ] Public subnet exists
- [ ] Two private subnets exist

### Gateways
- [ ] Internet Gateway attached
- [ ] NAT Gateway available

### Route Tables
- [ ] Public RT routes to IGW
- [ ] Private RT routes to NAT

### Security Groups
- [ ] All 3 SGs created correctly

### VPC Endpoints
- [ ] S3 endpoint available
- [ ] DynamoDB endpoint available
- [ ] Secrets Manager endpoint available

---

## PART 14 — Documentation Template

Create: `Lab_1_2_VPC_Documentation.txt`

Document:
- VPC IDs
- Subnet IDs
- Route Table IDs
- Security Group IDs
- Endpoint IDs
- NAT Gateway ID
- Elastic IP

---

## PART 15 — Cost Optimization

### Estimated Costs

| Resource | Monthly Cost |
|----------|-------------|
| NAT Gateway | ~$232 |
| Secrets Manager Endpoint | ~$7 |
| S3 Endpoint | FREE |
| DynamoDB Endpoint | FREE |
| Internet Gateway | FREE |

---

## ⚠️ IMPORTANT — Delete NAT Gateway

Delete after lab completion to avoid charges.

**Delete:**
- NAT Gateway
- Elastic IP

**Keep:**
- VPC
- Subnets
- Route Tables
- Security Groups
- VPC Endpoints

---

## What You Learned

### Core AWS Networking Concepts
- VPCs
- Subnets
- Route Tables
- NAT Gateways
- Internet Gateways
- Security Groups
- VPC Endpoints

### Security Principles
- Least privilege
- Network isolation
- Layered security
- Private infrastructure design

### Cost Optimization
- Avoid unnecessary NAT usage
- Use Gateway Endpoints when possible

### High Availability
- Multi-AZ architecture
- Redundancy planning

---

## Final Success Criteria

- [ ] VPC created successfully
- [ ] Public and private subnets configured
- [ ] Internet Gateway attached
- [ ] NAT Gateway operational
- [ ] Route tables configured
- [ ] Security groups secured
- [ ] VPC endpoints working
- [ ] Documentation completed
- [ ] NAT Gateway deleted after lab

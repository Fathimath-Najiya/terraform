# 🚀 Deploying Secure EC2 Instances with a Shared RDS Database using Terraform

![AWS](https://img.shields.io/badge/AWS-Cloud-orange?style=flat&logo=amazon-aws)
![Terraform](https://img.shields.io/badge/Terraform-IaC-purple?style=flat&logo=terraform)
![MySQL](https://img.shields.io/badge/MySQL-8.0-blue?style=flat&logo=mysql)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat)

---

## 📌 Project Overview

This project demonstrates how to use **Terraform** to provision a complete, production-grade AWS infrastructure — including a custom VPC, two EC2 web servers, and a shared MySQL RDS database — entirely through Infrastructure as Code (IaC).

The core goal is to prove that multiple EC2 instances can share a single RDS database, a common real-world pattern for scalable web application backends.

---

## 🏗️ Architecture

```
                        Internet
                           │
               ┌───────────▼───────────┐
               │   Internet Gateway    │
               │      (AppIGW)         │
               └───────────┬───────────┘
                           │
               ┌───────────▼─────────────────────────────────┐
               │              AWS VPC (AppVPC)                │
               │             CIDR: 10.0.0.0/16                │
               │                                              │
               │  ┌──────────────────┐  ┌──────────────────┐  │
               │  │   AppSubnet1     │  │   AppSubnet2     │  │
               │  │   10.0.1.0/24    │  │   10.0.2.0/24    │  │ 
               │  │   us-east-1a     │  │   us-east-1b     │  │
               │  │  ┌────────────┐  │  │  ┌────────────┐  │  │
               │  │  │ WebServer1 │  │  │  │ WebServer2 │  │  │
               │  │  │ EC2+EIP    │  │  │  │ EC2+EIP    │  │  │
               │  │  └─────┬──────┘  │  │  └──────┬─────┘  │  │
               │  └────────│──────────┘  └─────────│───────┘  │
               │           │   Security: WebTrafficSG │       │
               │           └──────────┬───────────────┘       │
               │                      ▼                       │
               │          ┌───────────────────────┐           │
               │          │  AppDatabase (RDS)    │           │
               │          │  MySQL 8.0            │           │
               │          │  db.t3.micro          │           │
               │          └───────────────────────┘           │
               └──────────────────────────────────────────────┘
```

---

## 🛠️ AWS Resources Created

| Resource | Name | Details |
|---|---|---|
| VPC | AppVPC | CIDR: 10.0.0.0/16, DNS enabled |
| Subnet 1 | AppSubnet1 | 10.0.1.0/24 · us-east-1a |
| Subnet 2 | AppSubnet2 | 10.0.2.0/24 · us-east-1b |
| Security Group | WebTrafficSG | Ports: 22, 80, 443, 3306 |
| Network Interface | nw-interface1 | ENI for WebServer1 |
| Network Interface | nw-interface2 | ENI for WebServer2 |
| Internet Gateway | AppIGW | VPC internet access |
| Route Table | AppRouteTable | Default route → AppIGW |
| Elastic IP | public_ip1 | Static IP for WebServer1 |
| Elastic IP | public_ip2 | Static IP for WebServer2 |
| EC2 Instance | WebServer1 | t2.micro · AppSubnet1 |
| EC2 Instance | WebServer2 | t2.micro · AppSubnet2 |
| DB Subnet Group | app-db-subnet-group | Spans both AZs |
| RDS Instance | AppDatabase | MySQL 8.0 · db.t3.micro · 20GB |

---

## 📁 File Structure

```
terraform-ec2-rds/
├── provider.tf     # AWS provider + region configuration
├── vpc.tf          # VPC, subnets, SG, ENIs, IGW, route table, EIPs
├── ec2.tf          # WebServer1 + WebServer2 EC2 instances
├── rds.tf          # DB subnet group + RDS MySQL instance
└── outputs.tf      # instance1_id, instance2_id, route_table_ID
```

---

## ⚙️ Tech Stack

- **Terraform** — Infrastructure as Code (IaC)
- **AWS VPC** — Custom virtual network with multi-AZ public subnets
- **AWS EC2** — t2.micro · Amazon Linux 2023 · AMI: ami-06c68f701d8090592
- **AWS RDS** — MySQL 8.0.44 · db.t3.micro · 20GB gp2 storage
- **AWS Security Groups** — Port-level inbound/outbound traffic control
- **Elastic IPs** — Static public IPv4 addresses
- **AWS CLI** — Key pair creation and SSH access

---

## 📋 Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/downloads) v1.0+
- [AWS CLI](https://aws.amazon.com/cli/) installed and configured
- Active AWS account with IAM permissions for EC2, VPC, RDS

---

## 🚀 Deployment

### 1. Clone the repository
```bash
git clone https://github.com/Fathimath-Najiya/terraform.git
cd terraform
```

### 2. Configure AWS credentials
```bash
aws configure
```
> ⚠️ Never hardcode credentials in .tf files. Use aws configure or environment variables.

### 3. Create EC2 Key Pair
```bash
aws ec2 create-key-pair \
  --key-name my-ec2-key \
  --query 'KeyMaterial' \
  --output text > my-ec2-key.pem

chmod 600 my-ec2-key.pem
```

### 4. Initialize Terraform
```bash
terraform init
```

### 5. Review the plan
```bash
terraform plan
```

### 6. Deploy
```bash
terraform apply
```
Type `yes` when prompted. RDS provisioning takes ~5 minutes.

---

## ✅ Verifying the Shared Database

### SSH into WebServer1
```bash
ssh -i my-ec2-key.pem ec2-user@<WebServer1_Public_IP>
```

### Install MySQL client
```bash
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install mysql-community-client -y
```

### Connect to RDS from WebServer1
```bash
mysql -h <RDS_Endpoint> -P 3306 -u admin -p
```

### Create a test table
```sql
USE appdatabase;
CREATE TABLE test_table (id INT PRIMARY KEY, message VARCHAR(100));
INSERT INTO test_table VALUES (1, 'Hello from WebServer1');
SHOW DATABASES;
```

### SSH into WebServer2 and query the same table
```bash
ssh -i my-ec2-key.pem ec2-user@<WebServer2_Public_IP>
mysql -h <RDS_Endpoint> -P 3306 -u admin -p
```
```sql
USE appdatabase;
SELECT * FROM test_table;
-- ✅ Returns the row created from WebServer1 — shared DB confirmed!
```

---

## 📤 Terraform Outputs

| Output | Description |
|---|---|
| `instance1_id` | EC2 Instance ID of WebServer1 |
| `instance2_id` | EC2 Instance ID of WebServer2 |
| `route_table_ID` | Route Table ID of AppRouteTable |

---

## 🔐 Security Improvements for Production

| Issue | Current (Learning) | Production Fix |
|---|---|---|
| DB Password | Hardcoded in .tf | AWS Secrets Manager or TF_VAR_ env variable |
| Port 3306 | Open to 0.0.0.0/0 | Restrict to EC2 security group ID only |
| AWS Keys | provider.tf | IAM roles or AWS CLI profile |
| Terraform State | Local tfstate | S3 backend + DynamoDB state locking |
| Hardcoded values | Inline in resources | variables.tf + terraform.tfvars |
| RDS access | publicly_accessible = true | Private subnet + bastion host |

---

## 🧹 Cleanup

```bash
terraform destroy
```
Removes all 14 AWS resources. Type `yes` when prompted.

---

## 💡 Key Learnings

- Provisioning a multi-AZ VPC with Terraform from scratch
- How security groups control EC2-to-RDS connectivity
- Elastic IPs vs default public IPs — why static IPs matter
- Terraform state, idempotency, and the plan → apply workflow
- How to verify shared database architecture across multiple servers
- Production best practices for secrets, state, and network security

---

## 👩‍💻 Author

**Fathimath Najiya CK**  
AWS Certified Solutions Architect – Associate | AWS Certified Cloud Practitioner  
B.Tech CSE · College of Engineering Vadakara · CGPA 8.35

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=flat&logo=linkedin)](https://linkedin.com/in/fathimath-najiya-ck-108127258/)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?style=flat&logo=github)](https://github.com/Fathimath-Najiya)

---

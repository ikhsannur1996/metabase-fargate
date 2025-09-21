# Deploying Metabase on Amazon ECS with Fargate, Network, and PostgreSQL RDS Setup

This guide walks through deploying Metabase on AWS ECS using Fargate with a managed PostgreSQL RDS backend and the necessary AWS networking components.

---

## Overview

- Prepare the network: VPC, public/private subnets, security groups, internet gateway, NAT gateway.
- Create a PostgreSQL RDS database for Metabase backend.
- Deploy Metabase container on ECS Cluster with Fargate launch type.
- Use Application Load Balancer (ALB) to expose Metabase UI.
- Secure communication: ECS tasks connect privately to RDS.

---

## Step 0: Prepare Network Infrastructure

1. **Create a VPC** with:
   - At least two public subnets (for ALB and NAT Gateway).
   - At least two private subnets (for ECS tasks and RDS).

2. **Attach an Internet Gateway** to enable internet access for public subnets.

3. **Set up Route Tables**:
   - Public subnets route internet-bound traffic to internet gateway.
   - Private subnets route internet-bound traffic via NAT Gateway in public subnet.

4. **Create Security Groups**:
   - ALB Security Group: Allow inbound HTTP (80) or HTTPS (443) from anywhere.
   - ECS Tasks Security Group: Allow inbound traffic from ALB SG on port 3000.
   - RDS Security Group: Allow inbound PostgreSQL (port 5432) from ECS Tasks SG only.

---

## Step 1: Create PostgreSQL RDS Instance

1. Go to RDS > Create Database.
2. Choose Engine: PostgreSQL.
3. Use "Standard Create".
4. Configure instance:
   - DB instance identifier: `metabase-db`
   - Instance class: `db.t3.medium` (or as needed)
   - Storage: General Purpose SSD (20GB or more)
   - Multi-AZ: optional
   - Network & Security: select your VPC and **private subnets**
   - Assign RDS security group created earlier
5. Set master username/password.
6. Launch database and note the endpoint.

---

## Step 2: Create ECS Cluster (Fargate)

1. In ECS console, choose **Clusters** > **Create Cluster**.
2. Select **Networking only (Fargate)**.
3. Name the cluster: `metabase-fargate-cluster`.
4. Create the cluster.

---

## Step 3: Create ECS Task Definition for Metabase

1. Go to ECS > Task Definitions > Create new Task Definition.
2. Select **FARGATE** launch type.
3. Configure:
   - Task Definition Name: `metabase-task`
   - Task Role: create if needed
   - Network Mode: `awsvpc`
   - CPU: 0.5 vCPU | Memory: 1GB
4. Add container:
   - Container Name: `metabase-container`
   - Image: `metabase/metabase:latest` (or specific version)
   - Port Mappings: container port `3000`
   - Environment Variables:
     ```
     MB_DB_TYPE=postgres
     MB_DB_HOST=<RDS-endpoint>
     MB_DB_PORT=5432
     MB_DB_DBNAME=<database-name>
     MB_DB_USER=<master-username>
     MB_DB_PASS=<master-password>
     ```
5. Create task definition.

---

## Step 4: Create and Deploy ECS Service with ALB

1. In ECS cluster, choose **Create Service**.
2. Launch type: **FARGATE**.
3. Service name: `metabase-service`.
4. Number of tasks: `1` (scale later as needed).
5. Networking:
   - Choose your VPC and **private subnets**.
   - Attach ECS task security group.
   - Enable **auto-assign public IP** only if using public subnets (usually false here).
6. Load Balancer:
   - Choose **Application Load Balancer**.
   - Create new ALB in **public subnets**.
   - Listener port 80 (or 443 for HTTPS).
   - Target group listens on port 3000 forwarding to ECS tasks.
   - Attach ALB security group.
7. Create service. Wait for stable running status.

---

## Step 5: Access and Test Metabase

- Access Metabase via the ALB DNS name.
- Complete initial setup wizard using RDS database credentials.
- Verify Metabase connects to PostgreSQL backend and loads data.

---

## Summary Table

| Step                | Description                               | Details/Settings                                  |
|---------------------|-------------------------------------------|--------------------------------------------------|
| Network Preparation | Create VPC, subnets, IGW, NAT, SGs         | Public and private subnets, ALB/ECS/RDS SGs       |
| PostgreSQL RDS      | Create PostgreSQL DB in private subnet      | db.t3.medium, SG allows ECS SG on port 5432       |
| ECS Cluster (Fargate) | Create cluster                               | `metabase-fargate-cluster`                        |
| Task Definition     | Metabase container config                    | 0.5 vCPU, 1GB, port 3000, DB environment vars     |
| ECS Service + ALB   | Deploy service with Load Balancer             | Private subnets for tasks, ALB in public subnet   |
| Test                | Access UI via ALB DNS                         | Verify Metabase operational                         |

---

This setup ensures secure, highly available Metabase deployment using AWS best practices with ECS Fargate, RDS Postgres, and properly segmented network.

---

Would you like a CloudFormation or Terraform template example to automate this entire setup?

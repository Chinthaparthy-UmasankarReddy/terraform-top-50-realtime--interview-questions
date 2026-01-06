
# Terraform + AWS VPC Interview Guide 

This document summarizes common Terraform + AWS **VPC** interview questions and concise, production‑style answers with example Terraform configurations.

---

## 1. VPC Basics and Core Concepts

### 1.1 How do you explain a VPC?

- A VPC is a logically isolated virtual network in AWS where you define your own IP range, subnets, route tables, and gateways. 
- It lets you control private vs public subnets, routing, and connectivity (internet, VPN, Direct Connect, VPC peering, Transit Gateway). 

Key points:

- CIDR block (e.g., `10.0.0.0/16`) and subnetting strategy are foundational design decisions. 

---

## 2. Creating a Basic VPC with Terraform

### 2.1 How do you create a VPC with public and private subnets?

Typical pattern:

- One VPC with a `/16` CIDR.  
- Public subnets in multiple AZs, each with a route to an Internet Gateway (IGW).  
- Private subnets using a NAT gateway for outbound internet access.

Example:

```hcl
provider "aws" {
  region = "ap-south-1"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "main-vpc"
    Environment = "prod"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

# Public subnet (AZ a)
resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true

  tags = {
    Name        = "public-a"
    Environment = "prod"
  }
}

# Private subnet (AZ a)
resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "ap-south-1a"

  tags = {
    Name        = "private-a"
    Environment = "prod"
  }
}

# Public route table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public_a_assoc" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}
```

You should be able to explain:

- Why public subnets have `map_public_ip_on_launch = true` and a default route to IGW. 
- Why private subnets do not have direct internet routes and often use NAT gateways instead.  

---

## 3. NAT Gateways, Private Subnets, and Internet Access

### 3.1 How do you allow instances in private subnets to access the internet?

Pattern:

- Create a NAT gateway in a public subnet with an Elastic IP.  
- Associate private subnets with a route table that routes `0.0.0.0/0` to the NAT gateway.

Example:

```hcl
# EIP for NAT
resource "aws_eip" "nat_eip" {
  vpc = true
}

# NAT Gateway in public subnet
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_a.id

  tags = {
    Name = "main-nat"
  }
}

# Private route table
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }

  tags = {
    Name = "private-rt"
  }
}

resource "aws_route_table_association" "private_a_assoc" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.private.id
}
```

Explain:

- Why NAT gateways must be in a **public** subnet with a route to the internet.  
- That outbound traffic from private subnets appears to come from the NAT IP, while inbound is blocked by default.
---

## 4. VPC Peering and Connectivity

### 4.1 How do you connect two VPCs using Terraform?

Use `aws_vpc_peering_connection` and update route tables in both VPCs.
Example (simplified, same account/region):

```hcl
resource "aws_vpc" "app_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_vpc" "shared_vpc" {
  cidr_block = "10.1.0.0/16"
}

resource "aws_vpc_peering_connection" "app_shared" {
  vpc_id      = aws_vpc.app_vpc.id
  peer_vpc_id = aws_vpc.shared_vpc.id
  auto_accept = true

  tags = {
    Name = "app-shared-peering"
  }
}

# Routes from app to shared
resource "aws_route" "app_to_shared" {
  route_table_id            = aws_route_table.app_private.id
  destination_cidr_block    = aws_vpc.shared_vpc.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.app_shared.id
}

# Routes from shared to app
resource "aws_route" "shared_to_app" {
  route_table_id            = aws_route_table.shared_private.id
  destination_cidr_block    = aws_vpc.app_vpc.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.app_shared.id
}
```

You should mention:

- Peering does not support transitive routing by default (no automatic “hub‑and‑spoke” via a third VPC). 
- For large, multi‑VPC topologies, Transit Gateway is often preferred.
---

## 5. VPC Design Best Practices

### 5.1 How do you choose VPC and subnet CIDRs?

Good answers include:

- Use non‑overlapping RFC1918 ranges (e.g., `10.0.0.0/16`, `10.1.0.0/16`) across environments and regions.  
- Plan for growth; use larger VPC CIDRs and smaller subnets per AZ (`/24`, `/20`) to support scaling.  
- Reserve separate VPCs for different trust boundaries (e.g., prod vs non‑prod).  

---

### 5.2 How do you organize VPC code in Terraform?

Common patterns:

- VPC as a **separate core networking module** used by app stacks.  
- One VPC module reused across environments (dev/stage/prod) with different CIDRs and tags via variables.  

Example module call:

```hcl
module "vpc" {
  source = "../../modules/vpc"

  name               = "prod"
  cidr_block         = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]
}
```

Often, teams also use battle‑tested open‑source VPC modules rather than hand‑rolling everything.  

---

## 6. VPC Endpoints (S3, DynamoDB, and Others)

### 6.1 How do you create a VPC endpoint to reach S3 privately?

Use `aws_vpc_endpoint` with `service_name` like `com.amazonaws.<region>.s3` and associate it with route tables.
Example:

```hcl
data "aws_region" "current" {}

resource "aws_vpc_endpoint" "s3_endpoint" {
  vpc_id          = aws_vpc.main.id
  service_name    = "com.amazonaws.${data.aws_region.current.name}.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.private.id
  ]

  tags = {
    Name = "s3-endpoint"
  }
}
```

Explain:

- Gateway endpoints (S3/DynamoDB) integrate with route tables; interface endpoints use ENIs and security groups.  
- VPC endpoints allow private access to AWS services without using public internet paths.
---

## 7. NACLs vs Security Groups

### 7.1 How do you compare Network ACLs and Security Groups in a VPC?

Key differences:

- NACLs: stateless, subnet‑level, ordered rules (allow/deny).  
- Security groups: stateful, instance/ENI level, only allow rules.  
- Many designs keep NACLs simple (mostly allow) and rely on security groups for detailed restrictions.  

In Terraform, NACLs are defined with `aws_network_acl` and `aws_network_acl_rule`, but in many setups they stay close to default while SGs do most of the fine‑grained protection.

---

## 8. Example: Full VPC Skeleton

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "prod-vpc"
    Environment = "prod"
  }
}

# IGW
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}

# Public subnets in two AZs
resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "public_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "ap-south-1b"
  map_public_ip_on_launch = true
}

# Private subnets
resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.11.0/24"
  availability_zone = "ap-south-1a"
}

resource "aws_subnet" "private_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.12.0/24"
  availability_zone = "ap-south-1b"
}

# Route tables (public + private with NAT) would follow similarly...
```

 

[1](https://terraform-handson-projects.hashnode.dev/advanced-scenario-based-terraform-interview-questions-and-answers)
[2](https://k21academy.com/terraform/terraform-interview-questions/)
[3](https://www.interviewbit.com/terraform-interview-questions/)
[4](https://k21academy.com/terraform-iac/terraform-interview-questions/)
[5](https://aws.plainenglish.io/380-top-aws-vpc-scenario-based-interview-questions-and-answers-with-simple-explanations-c6a917058c49)
[6](https://www.heyvaldemar.com/10-real-terraform-interview-questions-and-expert-answers-2025-devops-guide)
[7](https://www.devopsschool.com/blog/top-50-interview-questions-and-answers-on-aws-vpc/)
[8](https://www.reddit.com/r/Terraform/comments/191rsfc/assessing_terraform_coding_skills_whats_your/)
[9](https://zerotomastery.io/blog/terraform-interview-questions/)
[10](https://www.linkedin.com/posts/sahil-kumar-gupta-217b9222b_devops-aws-docker-activity-7384210242710016001-PV4L)

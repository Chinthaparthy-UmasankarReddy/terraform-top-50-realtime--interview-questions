
# Terraform: Inbound and Outbound (Ingress/Egress) Rules

This guide explains how to configure **inbound** and **outbound** rules for AWS Security Groups using Terraform. In security groups, inbound rules are called **ingress**, and outbound rules are called **egress**.

---

## 1. Concept Overview

- **Inbound (ingress)**: Controls traffic **coming into** your resource (destination is the instance/ENI).  
  Example: HTTP/HTTPS from the internet, SSH from office IPs.

- **Outbound (egress)**: Controls traffic **leaving** your resource (source is the instance/ENI).  
  Example: EC2 calling external APIs, app server connecting to a database or S3.

Key properties:

- Security groups are **stateful**: if inbound traffic is allowed, the response traffic is automatically allowed back out, and vice versa, without extra rules.  
- AWS console defaults to “allow all outbound,” but in Terraform you must define egress explicitly; otherwise outbound is denied.
---

## 2. Basic Example: Web Security Group (Ingress + Egress)

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Web SG with explicit inbound and outbound rules"
  vpc_id      = aws_vpc.main.id

  # ---------------------------
  # INBOUND (ingress) RULES
  # ---------------------------

  # HTTP from anywhere
  ingress {
    description = "HTTP from internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS from anywhere
  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH only from office IP range
  ingress {
    description = "SSH from office"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.0/24"]
  }

  # ---------------------------
  # OUTBOUND (egress) RULES
  # ---------------------------

  # Allow all outbound traffic (common default)
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "web-sg"
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}
```

Explanation:

- Inbound rules open ports 80/443 globally and 22 only to a specific CIDR range.  
- Outbound rule allows all protocols and ports to any IPv4 address, matching AWS’s typical default “allow all egress.”
---

## 3. Restricting Outbound Traffic

For tighter security, restrict outbound to only necessary ports (e.g., HTTP/HTTPS):

```hcl
resource "aws_security_group" "locked_down_sg" {
  name   = "locked-down-sg"
  vpc_id = aws_vpc.main.id

  # Example: only HTTPS inbound from a load balancer SG
  ingress {
    description     = "HTTPS from ALB"
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }

  # Outbound: allow only HTTP and HTTPS
  egress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Good talking points:

- Limiting outbound reduces the blast radius if an instance is compromised, because it cannot connect freely to arbitrary destinations.  
- For internal‑only services, outbound can be restricted further to specific CIDRs or VPC endpoints.

---

## 4. Using Separate Rule Resources

You can also manage rules with `aws_security_group_rule` for more granular control:

```hcl
resource "aws_security_group" "db_sg" {
  name   = "db-sg"
  vpc_id = aws_vpc.main.id
}

# Inbound from app SG on DB port
resource "aws_security_group_rule" "db_ingress_from_app" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.db_sg.id
  source_security_group_id = aws_security_group.app_sg.id
}

# Outbound: only to VPC CIDR
resource "aws_security_group_rule" "db_egress_vpc" {
  type              = "egress"
  from_port         = 0
  to_port           = 0
  protocol          = "-1"
  security_group_id = aws_security_group.db_sg.id
  cidr_blocks       = ["10.0.0.0/16"]
}
```

Benefits:

- Changing one rule does not force recreation of the whole security group, which can help in larger environments.
---

## 5. Summary Cheat‑Sheet

- **Inbound = ingress**: who/what can reach your instance (ports, protocols, source CIDRs/SGs)
- **Outbound = egress**: where your instance can connect to (destination CIDRs/ports). 
- Terraform **removes AWS’s default egress allow**, so you must define outbound explicitly (open or restricted).  
- Security groups are **stateful**: return traffic is automatically allowed if the initial direction was permitted.



[1](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
[2](https://cyberpanel.net/blog/aws-security-group-terraform)

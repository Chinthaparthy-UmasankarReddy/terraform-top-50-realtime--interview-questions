
# Terraform: Different EC2 Instances Within the Same Region

This guide shows how to define **multiple, different EC2 instances in the same AWS region** using Terraform. It uses shared provider configuration and separate resources (or a module) per instance type/role.

---

## 1. Directory Layout

For a simple, single‑state project:

```text
project/
├── main.tf
├── variables.tf
└── outputs.tf
```

You can also split into modules later (e.g., `modules/ec2_app`, `modules/ec2_bastion`) if needed.

---

## 2. Provider and Common Variables

`variables.tf`:

```hcl
variable "region" {
  type    = string
  default = "ap-south-1"
}

variable "key_name" {
  type = string
}

variable "public_subnet_id" {
  type = string
}

variable "private_subnet_id" {
  type = string
}

variable "bastion_sg_id" {
  type = string
}

variable "app_sg_id" {
  type = string
}

variable "db_sg_id" {
  type = string
}
```

`main.tf` (provider):

```hcl
terraform {
  required_version = ">= 1.4.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

---

## 3. AMI Data Sources (Shared)

Use data sources to look up suitable AMIs once and reuse them across instances.

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

If you have specialized AMIs per role, you can add more data sources or use variables.

---

## 4. Different EC2 Instances in the Same Region

### 4.1 Bastion Host (Public, Small)

```hcl
resource "aws_instance" "bastion" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = var.public_subnet_id
  vpc_security_group_ids = [var.bastion_sg_id]
  key_name               = var.key_name

  associate_public_ip_address = true

  tags = {
    Name        = "bastion"
    Environment = "prod"
    Role        = "bastion"
  }
}
```

Characteristics:

- Public subnet, public IP, small size.  
- SSH entry point for admins.

---

### 4.2 Application Server (Private, Medium)

```hcl
resource "aws_instance" "app" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.medium"
  subnet_id              = var.private_subnet_id
  vpc_security_group_ids = [var.app_sg_id]
  key_name               = var.key_name

  associate_public_ip_address = false

  root_block_device {
    volume_size = 50
    volume_type = "gp3"
    encrypted   = true
  }

  tags = {
    Name        = "app-server"
    Environment = "prod"
    Role        = "app"
  }
}
```

Characteristics:

- Private subnet (access via bastion / ALB).  
- Medium size for application workload.  
- Encrypted, larger root volume.

---

### 4.3 Database‑Like Instance (Private, Larger, Extra Volume)

```hcl
resource "aws_instance" "db_like" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.large"
  subnet_id              = var.private_subnet_id
  vpc_security_group_ids = [var.db_sg_id]
  key_name               = var.key_name

  associate_public_ip_address = false

  root_block_device {
    volume_size = 50
    volume_type = "gp3"
    encrypted   = true
  }

  ebs_block_device {
    device_name = "/dev/xvdb"
    volume_size = 200
    volume_type = "gp3"
    encrypted   = true
  }

  tags = {
    Name        = "db-like"
    Environment = "prod"
    Role        = "db"
  }
}
```

Characteristics:

- Private subnet only.  
- Larger instance type, separate data volume.  
- Suitable pattern when using EC2 instead of RDS (for practice or special cases).

---

## 5. Outputs

`outputs.tf`:

```hcl
output "bastion_public_ip" {
  value       = aws_instance.bastion.public_ip
  description = "Public IP of the bastion host"
}

output "app_private_ip" {
  value       = aws_instance.app.private_ip
  description = "Private IP of the app server"
}

output "db_like_private_ip" {
  value       = aws_instance.db_like.private_ip
  description = "Private IP of the db-like instance"
}
```

---

## 6. Applying the Configuration

Example `terraform.tfvars`:

```hcl
region            = "ap-south-1"
key_name          = "my-keypair"
public_subnet_id  = "subnet-0123456789abcdef0"
private_subnet_id = "subnet-0fedcba9876543210"
bastion_sg_id     = "sg-0aaa..."
app_sg_id         = "sg-0bbb..."
db_sg_id          = "sg-0ccc..."
```

Commands:

```bash
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"
```

---

## 7. Interview Talking Points

When asked about “different EC2 instances within the same region,” highlight:

- One provider block with a single region, **multiple `aws_instance` resources** with different roles and configurations.  
- Same VPC/region, but different subnets, security groups, instance types, and storage to fit each role.  
- How you’d evolve this into modules for reusability (e.g., `modules/bastion_ec2`, `modules/app_ec2`) if the setup grows.  

```

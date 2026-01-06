
# Terraform: Different EC2 Instances in Different Regions

This guide shows how to deploy **multiple EC2 instances in different AWS regions** using one Terraform configuration. It uses multiple AWS provider blocks with aliases, one per region, and separate EC2 resources bound to each provider.

---

## 1. High‑Level Idea

- Use **one state** and **multiple providers**, each with a different `region` and `alias`.  
- For each EC2 instance, specify which provider to use via `provider = aws.<alias>`.  
- Optionally share variables (AMI IDs, instance types) or customize per region.

This is a common approach for multi‑region patterns such as active/passive or pilot‑light architectures.

---

## 2. Provider Configuration for Multiple Regions

`main.tf` (providers):

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

# Primary region, e.g., ap-south-1 (Mumbai)
provider "aws" {
  alias  = "mumbai"
  region = "ap-south-1"
}

# Secondary region, e.g., ap-southeast-1 (Singapore)
provider "aws" {
  alias  = "singapore"
  region = "ap-southeast-1"
}
```

Key points to mention in interviews:

- Each provider has a unique `alias`.  
- You can still have a “default” provider if you want, but for multi‑region it is clearer to always use aliases.
---

## 3. AMI Data Sources Per Region

You usually fetch AMIs **per region**, because AMI IDs differ between regions.

```hcl
# AMI in Mumbai
data "aws_ami" "amazon_linux_mumbai" {
  provider    = aws.mumbai
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# AMI in Singapore
data "aws_ami" "amazon_linux_singapore" {
  provider    = aws.singapore
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

---

## 4. EC2 in Region A (e.g., Mumbai)

```hcl
resource "aws_instance" "app_mumbai" {
  provider = aws.mumbai

  ami           = data.aws_ami.amazon_linux_mumbai.id
  instance_type = "t3.micro"

  # These subnet/SG IDs must exist in ap-south-1
  subnet_id              = "subnet-mumbai-123456"
  vpc_security_group_ids = ["sg-mumbai-app-123456"]
  key_name               = "keypair-mumbai"

  tags = {
    Name        = "app-mumbai"
    Environment = "prod"
    Region      = "ap-south-1"
  }
}
```

Notes:

- This instance is created in **ap‑south‑1** because it uses `provider = aws.mumbai`.  
- Subnet and security group IDs must belong to the VPC in that region.

---

## 5. EC2 in Region B (e.g., Singapore)

```hcl
resource "aws_instance" "app_singapore" {
  provider = aws.singapore

  ami           = data.aws_ami.amazon_linux_singapore.id
  instance_type = "t3.micro"

  # These subnet/SG IDs must exist in ap-southeast-1
  subnet_id              = "subnet-singapore-123456"
  vpc_security_group_ids = ["sg-singapore-app-123456"]
  key_name               = "keypair-singapore"

  tags = {
    Name        = "app-singapore"
    Environment = "prod"
    Region      = "ap-southeast-1"
  }
}
```

This instance lives in **ap‑southeast‑1**, fully independent of the Mumbai one.

---

## 6. Variables and Outputs

`variables.tf` (optional, to avoid hard‑coding IDs):

```hcl
variable "mumbai_subnet_id" {
  type = string
}

variable "mumbai_sg_ids" {
  type = list(string)
}

variable "singapore_subnet_id" {
  type = string
}

variable "singapore_sg_ids" {
  type = list(string)
}

variable "mumbai_key_name" {
  type = string
}

variable "singapore_key_name" {
  type = string
}
```

Update resources:

```hcl
resource "aws_instance" "app_mumbai" {
  provider = aws.mumbai

  ami                    = data.aws_ami.amazon_linux_mumbai.id
  instance_type          = "t3.micro"
  subnet_id              = var.mumbai_subnet_id
  vpc_security_group_ids = var.mumbai_sg_ids
  key_name               = var.mumbai_key_name

  tags = {
    Name        = "app-mumbai"
    Environment = "prod"
  }
}

resource "aws_instance" "app_singapore" {
  provider = aws.singapore

  ami                    = data.aws_ami.amazon_linux_singapore.id
  instance_type          = "t3.micro"
  subnet_id              = var.singapore_subnet_id
  vpc_security_group_ids = var.singapore_sg_ids
  key_name               = var.singapore_key_name

  tags = {
    Name        = "app-singapore"
    Environment = "prod"
  }
}
```

`outputs.tf`:

```hcl
output "mumbai_instance_public_ip" {
  value       = aws_instance.app_mumbai.public_ip
  description = "Public IP of EC2 in ap-south-1"
}

output "singapore_instance_public_ip" {
  value       = aws_instance.app_singapore.public_ip
  description = "Public IP of EC2 in ap-southeast-1"
}
```

---

## 7. Applying the Configuration

Example `terraform.tfvars`:

```hcl
mumbai_subnet_id      = "subnet-aaaa..."
mumbai_sg_ids         = ["sg-aaaa..."]
mumbai_key_name       = "keypair-mumbai"

singapore_subnet_id   = "subnet-bbbb..."
singapore_sg_ids      = ["sg-bbbb..."]
singapore_key_name    = "keypair-singapore"
```

Commands:

```bash
terraform init
terraform plan  -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"
```

Terraform will provision **both EC2 instances** in their respective regions in a single run.

---

## 8. Interview Talking Points

When asked about “different EC2 within different regions,” emphasize:

- Use of **multiple AWS providers with aliases**, one per region.  
- **Per‑region AMIs** and network resources (VPC, subnets, SGs) – IDs are not shared across regions.
- Same codebase/state can safely manage multi‑region resources when providers are clearly separated.  
- Optional evolution: split into per‑region modules, or even separate state per region if team size and blast‑radius concerns grow.

```



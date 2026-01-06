
# Terraform + EC2 Interview Guide 

This document summarizes common Terraform + EC2 interview questions and concise, production‑style answers, with example Terraform configurations. It targets roughly 3–4 years of hands‑on experience with AWS and Terraform.[web:17][web:20]

---

## 1. Terraform State & Workflow

### 1.1 How does Terraform track EC2 resources? What happens if state is lost or edited?

Terraform uses a **state file** to map resources in code (e.g., `aws_instance.web`) to real AWS resources (e.g., `i-0123456789abcdef`). This mapping allows Terraform to detect drift and compute create/update/destroy actions during `plan` and `apply`.[web:17][web:20]

If the state file is lost or manually edited:

- Terraform may think resources do not exist and try to recreate them.
- It may attempt to destroy resources that still exist but are no longer in state.
- You can end up with orphaned or duplicated infrastructure and potential downtime.

**Best practices:**

- Store state remotely (e.g., S3 + DynamoDB lock in AWS) with encryption and versioning.[web:16]
- Never hand‑edit the state file; instead, use `terraform state` commands when needed.[web:17]
- Isolate state per environment (dev/stage/prod) and per major system where appropriate.[web:14]

Example backend configuration (S3 + DynamoDB):

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "envs/prod/ec2/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

---

### 1.2 Typical workflow: from code to running EC2

A common Terraform workflow:

1. **Write/Update Code**  
   - Define providers, VPC, subnets, security groups, and EC2 (or ASG/launch template) resources.
2. **Initialize**  
   - Run `terraform init` to download providers and configure the backend.[web:17]
3. **Plan**  
   - Run `terraform plan` to preview changes; review for destructive actions.[web:17]
4. **Apply**  
   - Run `terraform apply` to provision or update resources.
5. **Maintain**  
   - Use additional `plan/apply` cycles to introduce changes, and keep state in a remote backend.

Minimal EC2 example:

```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_instance" "web" {
  ami                    = "ami-0abcdef1234567890"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  key_name               = aws_key_pair.main.key_name

  tags = {
    Name        = "web-prod"
    Environment = "prod"
  }
}
```

---

## 2. Structuring Terraform for Multiple Environments

### 2.1 How do you structure Terraform for dev/stage/prod?

Typical pattern:

- Create **reusable modules** (e.g., `modules/ec2_app`) that define VPC, security groups, EC2/ASG, etc.
- Create **separate environment folders** (e.g., `envs/dev`, `envs/prod`) with different variable values and separate state backends.[web:14]
- Use remote state per environment instead of sharing one state across all envs.[web:14][web:19]

Example layout:

```text
modules/
  ec2_app/
    main.tf
    variables.tf
    outputs.tf
envs/
  dev/
    main.tf
    dev.tfvars
  prod/
    main.tf
    prod.tfvars
```

Module (`modules/ec2_app/main.tf`):

```hcl
variable "instance_type" {}
variable "env" {}
variable "subnet_id" {}
variable "sg_ids" {
  type = list(string)
}

resource "aws_instance" "app" {
  ami                    = "ami-0abcdef1234567890"
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.sg_ids

  tags = {
    Name        = "app-${var.env}"
    Environment = var.env
  }
}
```

Prod environment (`envs/prod/main.tf`):

```hcl
module "ec2_app" {
  source        = "../../modules/ec2_app"
  env           = "prod"
  instance_type = "t3.medium"
  subnet_id     = "subnet-123"
  sg_ids        = ["sg-123"]
}
```

---

## 3. EC2 Provisioning Details

### 3.1 Key arguments in `aws_instance` / `aws_launch_template` for production

For production EC2, important settings usually include:[web:4]

- **Compute**: `ami`, `instance_type`
- **Networking**: `subnet_id`, `vpc_security_group_ids`, `associate_public_ip_address`
- **IAM**: `iam_instance_profile` for least‑privilege access to AWS services
- **Storage**: `root_block_device`, `ebs_block_device` for size, type, and encryption
- **Bootstrap**: `user_data` or `user_data_base64` for configuration
- **Security**: `metadata_options` (IMDSv2), security groups, KMS keys
- **Observability**: `monitoring = true` (detailed CloudWatch metrics)
- **Tagging**: Name, Environment, Owner, CostCenter, etc.

Example:

```hcl
resource "aws_instance" "app" {
  ami                    = data.aws_ami.app.id
  instance_type          = "t3.medium"
  subnet_id              = aws_subnet.app_private_1.id
  vpc_security_group_ids = [aws_security_group.app_sg.id]
  iam_instance_profile   = aws_iam_instance_profile.app_profile.name
  key_name               = aws_key_pair.main.key_name

  associate_public_ip_address = false

  root_block_device {
    volume_size = 50
    volume_type = "gp3"
    encrypted   = true
  }

  ebs_block_device {
    device_name = "/dev/xvdb"
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true
  }

  metadata_options {
    http_tokens   = "required"
    http_endpoint = "enabled"
  }

  monitoring = true

  tags = {
    Name        = "app-prod-1"
    Environment = "prod"
    Owner       = "platform-team"
    CostCenter  = "cc-1234"
  }
}
```

---

## 4. AMI Selection & User Data

### 4.1 How do you handle AMI selection?

Approaches:

- Use an `aws_ami` **data source** with filters for name, owner, and virtualization type to always find the latest matching AMI.[web:20]
- For strict reproducibility, pass AMI IDs via variables or from a separate image pipeline (e.g., Packer/Golden AMI).[web:20]
- Use module variables to override AMI per environment if necessary.

Example AMI lookup:

```hcl
data "aws_ami" "app" {
  owners      = ["123456789012"]
  most_recent = true

  filter {
    name   = "name"
    values = ["my-app-base-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

### 4.2 How do you pass bootstrap scripts (user_data)?

- Use `user_data` or `user_data_base64` on `aws_instance` or in `aws_launch_template`.[web:20]
- Keep scripts in separate template files and use `templatefile()` to inject variables.
- Use cloud‑init or shell scripts to install and configure software.

Example with `templatefile`:

```hcl
locals {
  user_data = templatefile("${path.module}/user_data.sh.tmpl", {
    env      = var.env
    app_name = var.app_name
    region   = var.region
  })
}

resource "aws_instance" "app" {
  # ...
  user_data = local.user_data
}
```

`user_data.sh.tmpl`:

```bash
#!/bin/bash
echo "Environment: ${env}" >> /etc/motd
echo "App: ${app_name}" >> /etc/motd
amazon-linux-extras install nginx1 -y
systemctl enable nginx
systemctl start nginx
```

---

## 5. Networking & Security

### 5.1 How do you control whether an instance gets a public IP?

Options:

- On `aws_instance`, use `associate_public_ip_address` (requires appropriate subnet setting).
- On `aws_network_interface`, set `associate_public_ip_address`, then attach ENI to the instance.
- Use public subnets for internet‑facing instances and private subnets with NAT for internal services.[web:20]

Instance‑level:

```hcl
resource "aws_instance" "bastion" {
  ami           = data.aws_ami.bastion.id
  instance_type = "t3.micro"

  subnet_id                   = aws_subnet.public_1.id
  associate_public_ip_address = true

  # ...
}
```

Using ENI:

```hcl
resource "aws_network_interface" "eni" {
  subnet_id                   = aws_subnet.public_1.id
  private_ips                 = ["10.0.1.10"]
  security_groups             = [aws_security_group.web_sg.id]
  associate_public_ip_address = true
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.app.id
  instance_type = "t3.micro"

  network_interface {
    network_interface_id = aws_network_interface.eni.id
    device_index         = 0
  }
}
```

---

### 5.2 How do you design and manage security groups?

Typical practices:

- Define **role‑based security groups** (web, app, db, bastion, etc.).
- Use CIDR rules for public access (e.g., HTTP/HTTPS from `0.0.0.0/0` if needed).
- Use security‑group‑to‑security‑group rules for internal communication between tiers.[web:20]
- Keep security groups reusable and consistent across environments.

Example:

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow HTTP/HTTPS from internet and SSH from office"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.0/24"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-sg"
  }
}
```

---

### 5.3 How do you associate Elastic IPs, and when are they useful?

- Use `aws_eip` to allocate a static IP and `aws_eip_association` (or `instance`/`network_interface` fields) to bind it to an instance or ENI.[web:20]
- EIPs are useful when:
  - You need stable IPs for IP whitelisting.
  - You use legacy systems that depend on fixed IPs.
  - You manage VPN peers or external integrations via IP.

Example:

```hcl
resource "aws_eip" "bastion_eip" {
  vpc = true
}

resource "aws_eip_association" "bastion_assoc" {
  instance_id   = aws_instance.bastion.id
  allocation_id = aws_eip.bastion_eip.id
}
```

---

## 6. Storage & AMIs

### 6.1 How do you configure EBS volumes?

Options:

- Use `root_block_device` and `ebs_block_device` inside `aws_instance`.
- Use separate `aws_ebs_volume` + `aws_volume_attachment` to control lifecycle independently (common for data volumes).[web:20]

Example on instance:

```hcl
resource "aws_instance" "db" {
  # ...

  root_block_device {
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true
  }

  ebs_block_device {
    device_name = "/dev/xvdb"
    volume_size = 500
    volume_type = "gp3"
    encrypted   = true
    kms_key_id  = aws_kms_key.ebs.arn
  }
}
```

Separate volume:

```hcl
resource "aws_ebs_volume" "logs" {
  availability_zone = "ap-south-1a"
  size              = 200
  type              = "gp3"
  encrypted         = true
}

resource "aws_volume_attachment" "logs_attach" {
  device_name = "/dev/xvdc"
  volume_id   = aws_ebs_volume.logs.id
  instance_id = aws_instance.app.id
}
```

---

### 6.2 How do you create a custom AMI and reuse it?

Typical flow:

- Use a build pipeline or Packer to create golden images, then output the AMI ID.
- Make the AMI ID available to Terraform via variables or data sources (e.g., SSM parameter, tag filtering).[web:20]
- Optionally, use `aws_ami_from_instance` to snapshot a configured instance, but this is less common in pure Terraform flows.

Example:

```hcl
resource "aws_ami_from_instance" "app_image" {
  name               = "app-base-${timestamp()}"
  source_instance_id = aws_instance.app_build.id
}
```

In another stack:

```hcl
variable "app_ami_id" {}

resource "aws_instance" "app" {
  ami           = var.app_ami_id
  instance_type = "t3.medium"
}
```

---

### 6.3 How do you change volume size/type without data loss?

- Let AWS **modify existing volumes** in place where possible instead of replacing them.
- Avoid forcing recreation of critical volumes with `create_before_destroy` unless you have a migration process.
- For root volumes, consider:

  - Using `lifecycle { ignore_changes = [root_block_device] }` and handling resizing manually.
  - Or carefully planning downtime and migration.

Example ignoring root changes:

```hcl
resource "aws_instance" "db" {
  # ...

  root_block_device {
    volume_size = 100
  }

  lifecycle {
    ignore_changes = [root_block_device]
  }
}
```

For data volumes, you can increase `size` and then expand the filesystem inside the OS.

---

## 7. Auto Scaling & High Availability (Quick Overview)

### 7.1 How do you set up an Auto Scaling Group (ASG) for EC2?

- Create a **launch template** with AMI, instance type, security groups, and user data.
- Create an `aws_autoscaling_group` referencing the launch template.
- Spread instances across subnets in multiple AZs and attach an ALB/NLB target group.[web:20]

Example:

```hcl
resource "aws_launch_template" "app_lt" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.app.id
  instance_type = "t3.medium"

  network_interfaces {
    security_groups = [aws_security_group.app_sg.id]
  }

  tag_specifications {
    resource_type = "instance"

    tags = {
      Role        = "app"
      Environment = "prod"
    }
  }
}

resource "aws_autoscaling_group" "app_asg" {
  desired_capacity    = 3
  max_size            = 5
  min_size            = 2
  vpc_zone_identifier = [
    aws_subnet.app_private_1.id,
    aws_subnet.app_private_2.id,
  ]

  launch_template {
    id      = aws_launch_template.app_lt.id
    version = "$Latest"
  }

  target_group_arns = [aws_lb_target_group.app_tg.arn]
  health_check_type = "EC2"
}
```

---

## 8. Tagging, Safety, and Best Practices

### 8.1 Tagging

- Apply consistent tags: `Name`, `Environment`, `Owner`, `CostCenter`, `Application`, `ManagedBy = Terraform`.
- Use locals or variables for tag maps and merge them across resources.

Example:

```hcl
locals {
  base_tags = {
    ManagedBy = "Terraform"
    Owner     = "platform-team"
  }
}

resource "aws_instance" "app" {
  # ...
  tags = merge(local.base_tags, {
    Name        = "app-prod-1"
    Environment = "prod"
  })
}
```

### 8.2 Safety and review

- Always run and review `terraform plan` before `apply`.
- Use separate IAM roles with least privilege for CI/CD pipelines that run Terraform.
- For critical resources, use `lifecycle { prevent_destroy = true }` and explicit override when needed.

Example:

```hcl
resource "aws_instance" "critical_app" {
  # ...
  lifecycle {
    prevent_destroy = true
  }
}
...




[1](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-manage)
[2](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create

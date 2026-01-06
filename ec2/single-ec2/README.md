
# Terraform Single EC2 Instance Guide 

This document shows how to design, explain, and implement a **single EC2 instance** using Terraform, including provider setup, networking assumptions, user data, and best practices.

---

## 1. Minimal Single EC2 Example

### 1.1 Basic `main.tf` for one instance

This is the smallest realistic example: one provider + one EC2 instance. In practice, you usually reference an existing VPC, subnet, and security group.

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
  region = "ap-south-1"
}

resource "aws_instance" "example" {
  ami           = "ami-0abcdef1234567890" # Replace with valid AMI
  instance_type = "t3.micro"

  tags = {
    Name        = "single-ec2-example"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

Key talking points:

- Explain that this uses the default VPC and default security group if no VPC/subnet/SG are specified (which is fine only for simple demos). 
- Mention that `t3.micro` or `t2.micro` are typical small instance types used in examples and may fall into free tier in some regions.

---

## 2. Single EC2 in a Custom VPC

### 2.1 EC2 using explicit subnet and security group

A more production‑like single instance references a specific subnet and security group instead of defaults.

```hcl
provider "aws" {
  region = "ap-south-1"
}

# Assume these already exist or are created in other modules:
# - aws_vpc.main
# - aws_subnet.public
# - aws_security_group.ssh_http

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.ssh_http.id]
  key_name               = aws_key_pair.main.key_name

  associate_public_ip_address = true

  tags = {
    Name        = "web-single-prod"
    Environment = "prod"
    Role        = "web"
  }
}
```

What to highlight in an interview:

- Using a **data source** to fetch the latest Amazon Linux 2 AMI by filter.
- Explicitly attaching the instance to a **public subnet** and controlled security group rather than relying on defaults.  
- `associate_public_ip_address` is needed (along with a public subnet route to IGW) for direct internet access.  

---

## 3. Single EC2 with `user_data` (Bootstrap)

### 3.1 Add `user_data` to configure the instance

This pattern shows installing a simple web server via cloud‑init/bash.

```hcl
locals {
  web_user_data = <<-EOF
    #!/bin/bash
    yum update -y
    amazon-linux-extras install nginx1 -y
    systemctl enable nginx
    systemctl start nginx
    echo "Hello from Terraform EC2" > /usr/share/nginx/html/index.html
  EOF
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.ssh_http.id]
  key_name               = aws_key_pair.main.key_name

  user_data = local.web_user_data

  tags = {
    Name        = "web-single-with-userdata"
    Environment = "dev"
  }
}
```

Discussion points:

- `user_data` is run once at first boot and is ideal for simple provisioning (installing packages, writing configs).  
- For more complex setups, you might move to configuration management (Ansible) or baking AMIs with Packer.  

---

## 4. Single EC2 with EBS Configuration

### 4.1 Configure root and additional data volume

```hcl
resource "aws_instance" "db_like" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.small"
  subnet_id              = aws_subnet.private.id
  vpc_security_group_ids = [aws_security_group.db_sg.id]

  # Root volume
  root_block_device {
    volume_size = 30
    volume_type = "gp3"
    encrypted   = true
  }

  # Extra data volume
  ebs_block_device {
    device_name = "/dev/xvdb"
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true
  }

  tags = {
    Name        = "single-db-like"
    Environment = "stage"
  }
}
```

Points to mention:

- You can control **size, type, and encryption** of root and data volumes directly in the instance resource. 
- For independent lifecycle or attaching to different instances later, you use `aws_ebs_volume` + `aws_volume_attachment` instead.  

---

## 5. Recommended Interview Talking Points for “Single EC2”

When asked to “create a single EC2 instance with Terraform”, strong answers include:

- **Provider + backend**: mention configuring provider (region, credentials) and usually a remote backend, even if not fully shown in the snippet.  
- **AMI strategy**: prefer data sources (`aws_ami`) with filters or a passed‑in AMI ID from a golden‑image pipeline.  
- **Networking clarity**: explicitly choose subnet, security groups, and whether the instance is public or private (with NAT if needed).  
- **Bootstrap**: show awareness of `user_data` for initial configuration and logging for troubleshooting.
- **Tagging**: always tag instances with `Name`, `Environment`, `Owner` or similar for operations and cost tracking.  

---

## 6. Basic Workflow Commands

For any of these examples, you should be able to describe this sequence:[web:20]

```bash
terraform init      # download providers, set up backend
terraform validate  # syntax checks
terraform plan      # show what will be created/changed
terraform apply     # actually create the EC2 instance
terraform destroy   # tear down the instance when finished
```




[1](https://spacelift.io/blog/terraform-ec2-instance)
[2](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)
[3](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-manage)
[4](https://blog.purestorage.com/purely-technical/how-to-create-an-aws-instance-with-terraform/)
[5](https://github.com/terraform-aws-modules/terraform-aws-ec2-instance)
[6](https://www.youtube.com/watch?v=XFR1wWARbR4)
[7](https://notes.kodekloud.com/docs/Terraform-Basics-Training-Course/Terraform-Provisioners/AWS-EC2-with-Terraform)
[8](https://jumpcloud.com/blog/how-to-launch-amazon-linux-ec2-instance-terraform)
[9](https://www.youtube.com/watch?v=E0j40hLwjhI)
[10](https://basic-instance-using-terraform.hashnode.dev/deploying-a-basic-aws-ec2-instance-terraform)

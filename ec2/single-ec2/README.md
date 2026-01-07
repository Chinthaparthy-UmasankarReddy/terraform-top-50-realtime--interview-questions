
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

Here's the complete, self-contained Terraform file combining your explicit subnet/security group EC2 pattern with all required infrastructure:

```hcl
provider "aws" {
  region = "ap-south-1"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "main-vpc-prod"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "main-igw-prod"
  }
}

# Public Subnet (ap-south-1a)
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "public-subnet-prod"
  }
}

# Route Table for Public Subnet
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = {
    Name = "public-rt-prod"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group for SSH (22) and HTTP (80) - Production-like
resource "aws_security_group" "ssh_http" {
  name_prefix = "web-sg-prod"
  vpc_id      = aws_vpc.main.id

  # SSH access (restrict to your IP in production)
  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTP access
  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # All outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ssh-http-sg-prod"
  }
}

# Generate SSH Key Pair
resource "tls_private_key" "main" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "main" {
  key_name   = "web-key-prod"
  public_key = tls_private_key.main.public_key_openssh
}

# Save private key locally
resource "local_file" "private_key" {
  content  = tls_private_key.main.private_key_pem
  filename = "web-key-prod.pem"
  file_permission = "0400"
}

# Data source for latest Amazon Linux 2 AMI (interview highlight)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# EC2 Instance using explicit subnet and security group (interview highlight)
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id              # Explicit public subnet
  vpc_security_group_ids = [aws_security_group.ssh_http.id]  # Explicit security group

  key_name               = aws_key_pair.main.key_name

  associate_public_ip_address = true  # Required for internet access in custom VPC

  # User data to install and start Apache (demo web server)
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Terraform EC2 Demo - $(hostname -f)</h1>" > /var/www/html/index.html
              EOF

  tags = {
    Name        = "web-single-prod"
    Environment = "prod"
    Role        = "web"
  }
}

# Output public IP for easy access
output "instance_public_ip" {
  value = aws_instance.web.public_ip
}

output "private_key_path" {
  value = local_file.private_key.filename
}
```

## Interview Highlights
- **Explicit dependencies**: References specific `aws_subnet.public` and `aws_security_group.ssh_http` instead of defaults
- **AMI data source**: Dynamically fetches latest Amazon Linux 2 AMI using filters
- **Public subnet routing**: `associate_public_ip_address = true` + IGW route table enables internet access
- **Production-ready**: Key pair generation, proper security group rules, user data for web server
- **Self-contained**: No external module dependencies

## Usage
```bash
terraform init
terraform plan
terraform apply
ssh -i web-key-prod.pem ec2-user@$(terraform output -raw instance_public_ip)
curl $(terraform output -raw instance_public_ip)
```

This single file creates everything needed and demonstrates production-like explicit resource referencing perfect for interviews.
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

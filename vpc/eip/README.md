
# Terraform: Elastic IP (EIP) Guide

This guide explains how to create and manage **Elastic IPs (EIPs)** with Terraform and attach them to EC2 instances or network interfaces.

An Elastic IP is a **static public IPv4 address** that remains associated with your AWS account until you explicitly release it. It provides a stable endpoint that survives EC2 stop/start cycles.

---

## 1. Key Concepts

- **Static public IP**: Unlike a normal public IP, an EIP does not change when you stop/start an instance; it stays until you disassociate or release it.  
- **Per region**: EIPs are allocated in a specific region and can only be used there.  
- **Cost model**:
  - Typically free when associated with a running instance.
  - Charged when allocated but **not** associated, or when you hold more than one per instance (check current AWS pricing).  
- **Association targets**:
  - Directly to an **EC2 instance**.
  - To a **network interface (ENI)**, which can then be attached to instances.

---

## 2. Basic Terraform Example: EIP + EC2 Instance

This example creates one EC2 instance and one Elastic IP, and associates them.

```hcl
terraform {
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

# Example VPC + subnet assumed to exist
# resource "aws_vpc" "main" { ... }
# resource "aws_subnet" "public" { ... }

resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami                    = "ami-0abcdef1234567890" # replace with a valid AMI
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  # Do NOT auto-assign a random public IP
  associate_public_ip_address = false

  tags = {
    Name = "web-with-eip"
  }
}

# Allocate an Elastic IP in the VPC
resource "aws_eip" "web_eip" {
  vpc = true

  tags = {
    Name = "web-eip"
  }
}

# Associate the EIP with the instance
resource "aws_eip_association" "web_eip_assoc" {
  instance_id   = aws_instance.web.id
  allocation_id = aws_eip.web_eip.id
}
```

What happens:

- Terraform allocates an EIP and then associates it with `aws_instance.web`.  
- The instance has no auto‑assigned public IP; instead, the EIP becomes its stable public address.
---

## 3. EIP on a Network Interface (ENI)

Associating EIPs with ENIs gives more flexibility (e.g., move EIP between instances).

```hcl
resource "aws_network_interface" "web_eni" {
  subnet_id       = aws_subnet.public.id
  security_groups = [aws_security_group.web_sg.id]

  tags = {
    Name = "web-eni"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  network_interface {
    network_interface_id = aws_network_interface.web_eni.id
    device_index         = 0
  }

  tags = {
    Name = "web-with-eni-and-eip"
  }
}

resource "aws_eip" "web_eni_eip" {
  vpc = true

  tags = {
    Name = "web-eni-eip"
  }
}

resource "aws_eip_association" "web_eni_eip_assoc" {
  network_interface_id = aws_network_interface.web_eni.id
  allocation_id        = aws_eip.web_eni_eip.id
}
```

Benefits:

- You can detach the ENI (with its EIP) from one instance and attach it to another for failover or blue/green rotations.  

---

## 4. Common Use Cases

Typical EIP use cases:

- **Bastion/Jump Host**: stable IP for SSH access from fixed office IPs or VPNs.  
- **Legacy whitelisting**: external partners or services that whitelist your public IP.  
- **Manual DNS mapping**: static A records pointing to an instance that shouldn’t change IP when restarted.
---

## 5. Best Practices

- Release unused EIPs to avoid unnecessary charges.  
- Prefer EIPs only when you **need** a fixed public IP; many modern designs front instances with a load balancer and don’t require per‑instance EIPs.  
- Combine EIPs with proper security groups and, ideally, private subnets where only bastions have EIPs.

---

## 6. Commands

Typical workflow:

```bash
terraform init
terraform plan
terraform apply
```

After `apply`, check:

- The EC2 instance is running in the expected subnet.  
- The EIP shows as “associated” and pings/HTTP requests reach the instance via that IP. 


[1](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-ec2-elastic-ip.html)
[2](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/ec2-example-elastic-ip-addresses.html)
[3](https://notes.kodekloud.com/docs/AWS-Networking-Fundamentals/Core-Networking-Services/Elastic-IP-Demo)
[4](https://notes.kodekloud.com/docs/AWS-Networking-Fundamentals/Core-Networking-Services/Elastic-IP)
[5](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
[6](https://github.com/heedsoftware/auto-assign-elastic-ip/blob/master/README.md)
[7](https://github.com/sebdah/aws-ec2-assign-elastic-ip/blob/master/README.md)
[8](https://github.com/licarpen/aws-developer/blob/dev/README.md)
[9](https://www.archerimagine.com/articles/aws/aws-elastic-ip-tutorial.html)
[10](https://amitblog.hashnode.dev/optimizing-your-aws-infrastructure-a-guide-to-launch-templates-and-elastic-ip)

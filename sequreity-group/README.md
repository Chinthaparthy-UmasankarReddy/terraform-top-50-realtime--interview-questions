
# Terraform + AWS Security Group Interview Guide 

This document summarizes common Terraform + AWS **Security Group (SG)** interview questions and concise, production‑style answers with example Terraform configurations.

---

## 1. Basics: What Is a Security Group?

### 1.1 How do you explain a security group?

- A security group is a **virtual firewall** attached to ENIs/instances that controls inbound (`ingress`) and outbound (`egress`) traffic at the instance level. 
- Rules are stateful: if inbound traffic is allowed, the response is automatically allowed without an explicit outbound rule. 

You should distinguish:

- Security groups (stateful, instance level) vs NACLs (stateless, subnet level).

---

## 2. Creating Security Groups with Terraform

### 2.1 How do you create a basic web server security group?

Use `aws_security_group` with `ingress` and `egress` blocks.

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow HTTP/HTTPS from internet and SSH from office"
  vpc_id      = aws_vpc.main.id

  # HTTP
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # HTTPS
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # SSH from office only
  ingress {
    description = "SSH from office"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.0/24"]
  }

  # Allow all outbound
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

Key points:

- Use **CIDR blocks** for internet or corporate ranges and **security group references** for internal traffic.
---

### 2.2 How do you attach a security group to an instance?

- With `aws_instance`, use `vpc_security_group_ids` (recommended in VPC). 
- With `aws_network_interface`, specify `security_groups` and later attach ENI to instances.

Example:

```hcl
resource "aws_instance" "web" {
  ami                    = data.aws_ami.app.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  tags = {
    Name = "web-prod-1"
  }
}
```

---

## 3. Designing Rules: Ingress, Egress, and References

### 3.1 How do you design SGs for multi‑tier apps (web/app/db)?

Typical pattern:

- `web_sg`: internet HTTP/HTTPS, SSH from bastion/office.  
- `app_sg`: only allow traffic from `web_sg` on app port. 
- `db_sg`: only allow DB port from `app_sg`.
Example:

```hcl
resource "aws_security_group" "app_sg" {
  name        = "app-sg"
  description = "App tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "App traffic from web tier"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

This shows:

- **Security‑group‑to‑security‑group** rules for internal flows instead of wide CIDRs.
---

### 3.2 When do you use separate `aws_security_group_rule` resources?

You use **separate rule resources** when:

- You want more granular control and plan output per rule.  
- You need to avoid replacing the whole SG when changing one rule.  

Example:

```hcl
resource "aws_security_group" "db_sg" {
  name        = "db-sg"
  description = "DB tier"
  vpc_id      = aws_vpc.main.id
}

resource "aws_security_group_rule" "db_ingress_from_app" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.db_sg.id
  source_security_group_id = aws_security_group.app_sg.id
}
```

---

## 4. Best Practices & Organization

### 4.1 How do you organize security groups in Terraform?

Common approaches:

- One module per **tier** (web/app/db) that exposes SG IDs as outputs.  
- Central SG module for **shared patterns** (e.g., SSH from bastion, monitoring access).  
- Use naming/tagging standards for clarity.

Example module output:

```hcl
output "web_sg_id" {
  value = aws_security_group.web_sg.id
}
```

You can also use community modules (e.g., `terraform-aws-modules/security-group/aws`) to standardize common patterns.

---

### 4.2 What are some security best practices for SGs?

Strong answers include:

- Restrict SSH/RDP to specific IP ranges or bastion hosts, **never** `0.0.0.0/0` in production.  
- Use SG references instead of broad CIDRs between tiers.  
- Limit open ports to what the application truly needs.  
- Regularly audit security groups for unused or overly permissive rules; manage drift via Terraform.  

---

## 5. Reusability, Modules, and Multiple Environments

### 5.1 How do you reuse SG logic across multiple environments?

- Wrap SG definitions in a **module** and pass variables for ports, CIDRs, and tags.  
- Use different `tfvars` files or environment modules (dev/stage/prod) to adjust allowed IPs and ports. 

Example module call:

```hcl
module "web_sg" {
  source = "../../modules/web_sg"

  vpc_id              = aws_vpc.main.id
  allowed_http_cidrs  = ["0.0.0.0/0"]
  allowed_ssh_cidrs   = ["203.0.113.0/24"]
  environment         = "prod"
}
```

Module skeleton:

```hcl
variable "vpc_id" {}
variable "allowed_http_cidrs" {
  type = list(string)
}
variable "allowed_ssh_cidrs" {
  type = list(string)
}
variable "environment" {}

resource "aws_security_group" "web_sg" {
  name        = "web-${var.environment}"
  description = "Web SG for ${var.environment}"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.allowed_http_cidrs
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_ssh_cidrs
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 6. Drift, Import, and Troubleshooting

### 6.1 How do you handle SG drift (manual changes in AWS)?

- Always run `terraform plan` to detect differences in SG rules. 
- Decide whether to **accept** the console change (update code) or **revert** it (apply Terraform state).  
- Avoid manual security group edits in production; use Terraform pull requests for changes.  

### 6.2 How do you bring an existing SG under Terraform?

1. Add a matching `aws_security_group` resource to code.  
2. Use `terraform import aws_security_group.example sg-1234567890abcdef`.  
3. Optionally import individual rules as `aws_security_group_rule` resources.  
4. Run `terraform plan` and align HCL with the current configuration.  

---

## 7. Example: Public Web + Private App Architecture

```hcl
# Web SG (public)
resource "aws_security_group" "web_sg" {
  name   = "web-prod"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
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

# App SG (private, only from web)
resource "aws_security_group" "app_sg" {
  name   = "app-prod"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

This demonstrates:

- Public web tier accepting HTTP, private app tier only reachable from web tier.  
- Clear separation of responsibilities and least privilege for network paths.



[1](https://github.com/terraform-aws-modules/terraform-aws-security-group)
[2](https://www.youtube.com/watch?v=Ldj9HWv4CpE)
[3](https://www.jit.io/resources/cloud-sec-tools/developers-tutorial-to-managing-aws-security-groups-through-terraform)
[4](https://zeet.co/blog/terraform-aws-security-group)
[5](https://dev.to/suzuki0430/terraform-beginners-aws-security-group-management-3pl3)
[6](https://www.reddit.com/r/aws/comments/puoty4/aws_terraform_how_do_you_manage_your_security/)
[7](https://github.com/cloudposse/terraform-aws-security-group)
[8](https://www.reddit.com/r/aws/comments/bx2tfu/managing_security_group_configuration_drift_with/)
[9](https://dev.to/aws-builders/how-to-terraform-multiple-security-group-with-varying-configuration-1638)
[10](https://www.reddit.com/r/Terraform/comments/drw9pg/best_practices_for_terraform_aws_security_group/)

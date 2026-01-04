
# Terraform Interview Guide: 50 Production-Ready Questions & Answers

**Terraform Interview Answers: Senior Engineer Perspective**

As a Terraform expert with 5+ years production experience across AWS, Azure, and multi-cloud environments, here are detailed answers to all 50 questions with real-world examples and code.

## Basic Concepts (1-10)

### 1. What is Terraform and its core workflow (init, plan, apply)?

Terraform is HashiCorp's Infrastructure as Code (IaC) tool that provisions and manages infrastructure through declarative configuration files written in HCL (HashiCorp Configuration Language). 

**Core Workflow:**
```
1. terraform init    # Download providers/plugins, initialize backend
2. terraform plan    # Preview changes (dry-run)
3. terraform apply   # Execute changes
4. terraform destroy # Cleanup (optional)
```

**Example:**
```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = "t3.micro"
  tags = {
    Name = "web-server"
  }
}
```

**Real-world:** Plan shows exactly what will change, preventing surprises in production.

### 2. How does Terraform differ from Ansible or Puppet?

| Aspect | Terraform | Ansible | Puppet |
|--------|-----------|---------|--------|
| **Purpose** | Infrastructure provisioning | Configuration management | Configuration management |
| **State** | Maintains state file | Stateless | Agent-based state |
| **Idempotent** | Declarative (desired state) | Procedural (steps) | Declarative |
| **Execution** | Pull-based (agentless) | Push/pull (agentless/agents) | Agent-based |
| **Use Case** | VPCs, VMs, DBs | App config, packages | Enterprise config mgmt |

**Best Practice:** Use Terraform for "golden path" infra + Ansible for app-layer config.

### 3. What is HCL, and why is it used in Terraform?

HCL (HashiCorp Configuration Language) is a declarative, human-readable language combining JSON-like structure with Terraform-specific functions.

**Example HCL vs JSON:**
```hcl
# HCL - Concise, readable
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "prod-vpc"
  }
}

# Equivalent JSON - Verbose
{
  "resource": {
    "aws_vpc": {
      "main": {
        "cidr_block": "10.0.0.0/16",
        "tags": { "Name": "prod-vpc" }
      }
    }
  }
}
```

**Advantages:** Native loops (`for_each`), conditionals, functions (`join()`, `cidrsubnet()`).

### 4. Explain Terraform providers and their role.

Providers are plugins that interact with APIs (AWS, Azure, GCP, Kubernetes, etc.). Terraform core is provider-agnostic.

**Multi-provider example:**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "kubernetes" {
  config_path = "~/.kube/config"
}
```

**Production tip:** Always pin versions to prevent breaking changes.

### 5. What are resources, data sources, and variables?

- **Resources**: Managed infrastructure (create/update/delete)
- **Data sources**: Read-only existing infrastructure
- **Variables**: Input parameters

```hcl
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
}
```

### 6. Describe outputs and their importance.

Outputs expose resource attributes for other configurations or humans.

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID for other modules"
}

output "public_ips" {
  value = aws_instance.web[*].public_ip
  description = "List of web server IPs"
}
```

**Cross-module usage:**
```hcl
# vpc/outputs.tf
output "vpc_id" { value = aws_vpc.main.id }

# app/main.tf
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "vpc/terraform.tfstate"
  }
}

resource "aws_subnet" "app" {
  vpc_id = data.terraform_remote_state.vpc.outputs.vpc_id
}
```

### 7. What happens during `terraform init`?

```bash
$ terraform init
```

**Actions:**
1. Downloads providers from registry
2. Initializes backend (S3, Consul, etc.)
3. Creates `.terraform` directory
4. Creates `terraform.lock.hcl` (dependency lockfile)
5. Validates config syntax

**Production output:**
```
Terraform has been successfully initialized!
Providers required by configuration:
├── provider[registry.terraform.io/hashicorp/aws] 5.40.0
└── backend "s3" (known after apply)
```

### 8. Explain `terraform validate` vs. `terraform plan`.

```bash
# validate - syntax + static analysis only
terraform validate

# plan - full preview including API calls
terraform plan -out=tfplan
```

**Validate catches:**
- Syntax errors
- Missing providers
- Invalid HCL types
- Circular dependencies

**Plan catches:**
- API permission issues
- Resource conflicts
- Drift detection

**Best practice:** Always validate → plan → apply.

### 9. What is the Terraform registry?

Public/private repository for providers and modules.

```hcl
# Public module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
  
  cidr = "10.0.0.0/16"
}

# Private module (Terraform Cloud Enterprise)
module "vpc" {
  source = "app.terraform.io/my-org/vpc/aws"
}
```

### 10. How do you destroy resources with Terraform?

```bash
# Preview destruction
terraform plan -destroy

# Execute destruction
terraform destroy

# Targeted destruction
terraform destroy -target=aws_instance.web
```

**Production safety:**
```bash
# Auto-approve with plan file
terraform plan -out=destroy.plan -destroy
terraform apply destroy.plan
```

## State Management (11-20)

### 11. What is the Terraform state file, and why is it critical?

State file (`terraform.tfstate`) maps Terraform config to real infrastructure.

**Structure:**
```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "resources": [{
    "mode": "managed",
    "type": "aws_instance",
    "name": "web",
    "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
    "instances": [{ "attributes": { "id": "i-1234567890abcdef0" } }]
  }]
}
```

**Critical because:**
- Tracks resource addresses → real IDs
- Detects drift (config vs. reality)
- Enables `plan` without destroying everything

### 12. Risks of using local state in production?

```
❌ NEVER use local state in production!
```

**Risks:**
- Single point of failure (laptop dies)
- No collaboration/locking
- Git commit risk (secrets exposure)
- Manual backup required

**Production backend:**
```hcl
terraform {
  backend "s3" {
    bucket         = "prod-terraform-state"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### 13. How do you configure remote state with S3 backend?

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "myorg-prod-tfstate"
    key            = "prod/app/terraform.tfstate"
    region         = "us-east-1"
    
    # State locking
    dynamodb_table = "terraform-state-locks"
    
    # Encryption
    encrypt        = true
    
    # KMS encryption
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/abc123"
  }
}
```

**CLI init:**
```bash
terraform init -reconfigure -backend-config="bucket=myorg-prod-tfstate" \
               -backend-config="key=prod/app/terraform.tfstate"
```

### 14. What is state locking, and how does DynamoDB help?

**Problem:** Two engineers run `apply` simultaneously → corruption.

**Solution:** DynamoDB lock table
```hcl
# Single table for all state files
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name = "Terraform State Locks"
  }
}
```

**Lock in action:**
```
$ terraform apply
Acquiring state lock... SUCCEEDED
```

### 15. Explain `terraform state mv` and `terraform state rm`.

**State mv** - Rename/move resources in state:
```bash
# Move resource to new address
terraform state mv 'aws_instance.web' 'aws_instance.web_server'

# Move to module
terraform state mv 'aws_vpc.main' 'module.vpc.aws_vpc.main'
```

```markdown
## State Management (11-20) - Continued

### 15. Explain `terraform state mv` and `terraform state rm`.

**State mv** - Rename/move resources in state:
```bash
# Move resource to new address
terraform state mv 'aws_instance.web' 'aws_instance.web_server'

# Move to module
terraform state mv 'aws_vpc.main' 'module.vpc.aws_vpc.main'
```

**State rm** - Remove from state (doesn't destroy):
```bash
# Remove manually created resource
terraform state rm aws_instance.legacy

# Remove entire module
terraform state rm module.database
```

**Production use:** Cleanup after `import`, fix refactors.

### 16. How to detect and fix configuration drift?

**Drift types:**
1. **Intentional** - Manual changes in console
2. **Unintentional** - Provider/API changes

**Detection:**
```bash
terraform plan  # Shows +/-/~ changes
```

**Fix options:**
```bash
# Option 1: Revert manual changes
terraform apply

# Option 2: Remove from state
terraform state rm aws_instance.drifted

# Option 3: Import corrected resource
terraform import aws_instance.web i-1234567890abcdef0
```

**Proactive:** AWS Config + Terraform Cloud Drift Detection.

### 17. Migrate state between backends without downtime?

**Live migration process:**
```bash
# 1. Backup current state
terraform state pull > backup.tfstate

# 2. Init new backend (copy mode)
terraform init -backend-config="new-s3-config"

# 3. Copy state to new backend
terraform state push backup.tfstate

# 4. Switch to new backend
terraform init -reconfigure

# 5. Verify
terraform plan  # Should show no changes
```

**Production:** Test in staging first, coordinate team outage window.

### 18. Secure state file with encryption and access controls?

**Complete security setup:**
```hcl
terraform {
  backend "s3" {
    bucket      = "secure-tfstate"
    key         = "prod/app/terraform.tfstate"
    encrypt     = true
    kms_key_id  = aws_kms_key.tfstate.arn  # Customer-managed
  }
}

# KMS key policy
resource "aws_kms_key" "tfstate" {
  description             = "Terraform state encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

# S3 bucket policy - least privilege
resource "aws_s3_bucket_policy" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  policy = jsonencode({
    Statement = [{
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.tfstate.arn}/*"
      Condition = {
        StringNotEquals = {
          "aws:PrincipalOrgID" = "o-xxxxx"
        }
      }
    }]
  })
}
```

### 19. What is Terraform Cloud/HCP for state management?

**Terraform Cloud features:**
- Hosted state with auto-backups
- VCS integration (GitHub/GitLab)
- Role-based access (workspace/policy sets)
- Drift detection + notifications
- Cost estimation
- Private module registry

**Migration from S3:**
```hcl
# Remove S3 backend block
terraform {
  # backend "s3" {}  # <-- Comment out
  
  # Terraform Cloud auto-configures
  required_providers {
    aws = { source = "hashicorp/aws" }
  }
}
```

### 20. Handle state corruption or loss scenarios?

**Prevention first:**
- S3 versioning + MFA delete
- Cross-region replication
- Regular `state pull` backups
- Terraform Cloud snapshots

**Recovery process:**
```bash
# Scenario 1: State corruption
cp terraform.tfstate.backup terraform.tfstate
terraform plan  # Verify recovery

# Scenario 2: Complete loss
# 1. Recreate from config (empty state)
rm terraform.tfstate*
terraform apply  # Reimports everything

# Scenario 2b: Selective recovery
terraform import aws_instance.web i-1234567890abcdef0
```

## Modules and Reusability (21-25)

### 21. What are Terraform modules, and how do you create one?

**Module structure:**
```
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
└── README.md
```

**Complete VPC module:**
```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.tags, {
    Name = var.name
  })
}

resource "aws_subnet" "private" {
  for_each = { for k, v in var.private_subnets : k => v }
  
  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  
  tags = merge(var.tags, {
    Name = "${var.name}-private-${each.key}"
    Type = "private"
  })
}
```

**Usage:**
```hcl
module "prod_vpc" {
  source = "./modules/vpc"
  
  name           = "prod"
  cidr           = "10.0.0.0/16"
  private_subnets = {
    "0" = { cidr = "10.0.1.0/24", az = "us-east-1a" }
    "1" = { cidr = "10.0.2.0/24", az = "us-east-1b" }
  }
  tags = { Environment = "prod", Owner = "platform" }
}
```

### 22. Best practices for module versioning and publishing?

**Semantic versioning:**
```
v1.0.0  # Initial stable
v1.1.0  # New features
v1.0.1  # Bug fixes
v2.0.0  # Breaking changes
```

**Publishing to private registry:**
```bash
# Tag and push
git tag v1.2.0
git push origin v1.2.0

# Terraform Cloud private module
terraform publish provider=aws module=vpc version=1.2.0
```

**Consumer pinning:**
```hcl
module "vpc" {
  source  = "myorg/vpc/aws"
  version = "1.2.0"  # Exact version
}
```

### 23. Pass complex data to modules using objects/maps?

**Advanced inputs:**
```hcl
# variables.tf
variable "subnets" {
  type = map(object({
    cidr          = string
    az            = string
    type          = string
    auto_scaling  = optional(bool, false)
    max_capacity  = optional(number, 3)
  }))
}

# main.tf
resource "aws_autoscaling_group" "this" {
  for_each = {
    for k, v in var.subnets : k => v
    if v.auto_scaling
  }
  
  name                = "${var.name}-${each.key}"
  vpc_zone_identifier = aws_subnet.this[each.key].id
  max_size            = each.value.max_capacity
}
```

### 24. Root module vs. child modules: design principles?

**Architecture:**
```
root/
├── backend.tf          # Global state
├── providers.tf        # Global providers
├── vpc/              # Child module
├── eks/              # Child module  
├── rds/              # Child module
└── main.tf           # Orchestration
```

**main.tf (orchestration):**
```hcl
module "network" {
  source = "./vpc"
  env    = var.env
}

module "eks" {
  source = "./eks"
  
  vpc_id = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}

module "database" {
  source = "./rds"
  
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.database_subnet_ids
  eks_sg_id  = module.eks.node_security_group_id
}
```

### 25. Reuse modules across accounts with private registry?

**Multi-account setup:**
```
organization/
├── landing-zone/     # Shared baseline
├── prod/
│   ├── us-east-1/
│   └── eu-west-1/
└── staging/
    └── us-east-1/
```

**Private module registry (Terraform Cloud):**
```hcl
module "vpc" {
  source  = "app.terraform.io/myorg/networking/aws"
  version = "~> 2.0"
  
  account_id = data.aws_caller_identity.current.account_id
  environment = var.environment
}
```

## Advanced HCL and Loops (26-30)

### 26. Count vs. for_each: when to use each?

```hcl
# ❌ Avoid count - index-based, fragile
resource "aws_instance" "web" {
  count = 3
  ami   = "ami-123"
  # Problem: instance 1 destroy → indices shift
}

# ✅ for_each - key-based, stable
resource "aws_instance" "web" {
  for_each = toset(["app1", "app2", "app3"])
  ami      = "ami-123"
  # Destroy app2 → app1/app3 unchanged
}
```

**Decision matrix:**
| Use Case | count | for_each |
|----------|-------|----------|
| Fixed number | ✅ | ✅ |
| Dynamic list/map | ❌ | ✅ |
| Needs index | ✅ | Use `keys()` |

### 27. Dynamic blocks for conditional configurations?

**Security groups with dynamic rules:**
```hcl
resource "aws_security_group" "web" {
  name = "web-sg"
  
  dynamic "ingress" {
    for_each = var.enable_ssh ?  : [][1]
    
    content {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/16"]
    }
  }
  
  dynamic "ingress" {
    for_each = var.web_ports
    
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### 28. Locals for computed values: real-world example?

**CIDR calculator:**
```hcl
locals {
  vpc_cidr      = "10.0.0.0/16"
  public_cidrs  = cidrsubnets(local.vpc_cidr, 2, 0, 1)
  private_cidrs = cidrsubnets(local.vpc_cidr, 2, 2, 3)
  
  tags = {
    ManagedBy   = "terraform"
    Environment = var.env
    Owner       = "platform-team"
  }
  
  name = "${var.env}-${var.app}-${data.aws_region.current.name}"
}
```

### 29. Depends_on for implicit dependency resolution?

```hcl
# Implicit (preferred)
resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id  # Terraform infers dependency
}

# Explicit (when implicit fails)
resource "aws_instance" "db" {
  subnet_id = aws_subnet.private.id
  
  depends_on = [
    aws_nat_gateway.public,
    aws_route_table.private
  ]
}
```

### 30. Terraform functions like join(), coalesce()?

```hcl
locals {
  eks_subnet_ids = join(",", aws_subnet.private[*].id)
  instance_type = coalesce(var.instance_type, "t3.medium")
  nat_cidr = cidrsubnet(var.vpc_cidr, 8, 1)
}
```

## Workspaces and Environments (31-35)

### 31. What are workspaces, and when to avoid them?

**Workspaces create isolated state files:**
```bash
terraform workspace new dev
terraform workspace new staging  
terraform workspace new prod
```

**✅ Better approach:** Directory per environment
```
environments/
├── dev/main.tf
├── staging/main.tf
└── prod/main.tf
```

### 32. Separate directories vs. workspaces for envs?

**Directory structure (recommended):**
```
terraform-live/
├── prod/
│   ├── data/
│   ├── global/
│   └── us-east-1/
├── staging/
│   └── us-east-1/
└── dev/
    └── us-east-1/
```

### 33. Promote changes across dev/staging/prod?

**GitOps workflow:**
```
main (prod)    <-- staging    <-- dev (feature branches)
```

### 34. Variable sets in Terraform Cloud for envs?

**Workspace variable sets:**
```
Variable Set: "AWS Credentials"  # Shared across workspaces
Variable Set: "Network"         # Environment-specific
```

### 35. Handle multi-region workspaces?

**Multi-region strategy:**
```
prod/
├── global/           # IAM roles, VPC peering
├── us-east-1/        # Primary region
├── eu-west-1/        # DR region
└── ap-southeast-2/   # APAC
```

## Best Practices and Production (36-40)

### 36. Provider version pinning and why it matters?

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40.0"
    }
  }
}
```

### 37. Sensitive data handling with variables/secrets?

```hcl
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/app/db/password"
}
```

### 38. Zero-downtime deployments with lifecycle rules?

```hcl
lifecycle {
  create_before_destroy = true
}
```

### 39. Taint/untaint for problematic resources?

```bash
terraform taint aws_instance.web
terraform untaint aws_instance.web
```

### 40. Import existing resources safely?

```bash
terraform import aws_instance.imported i-1234567890abcdef0
```

## Provisioners and Integration (41-45)

### 41. Local-exec vs. remote-exec provisioners?

```hcl
provisioner "local-exec" {
  command = "echo 'Building AMI...'"
}

provisioner "remote-exec" {
  inline = ["sudo yum update -y"]
}
```

### 42. When to use null_resource?

```hcl
resource "null_resource" "deploy_k8s" {
  triggers = {
    config_sha = filesha256("${path.module}/k8s-manifests/")
  }
}
```

### 43. Integrate Terraform with Ansible/Chef?

**Pattern:** Terraform → generate inventory → Ansible

### 44. CI/CD pipelines: GitHub Actions/Atlantis?

```yaml
- run: terraform plan -out=tfplan
- run: terraform apply -auto-approve tfplan  # Only on main
```

### 45. Policy as Code with OPA/Sentinel?

```rego
deny[msg] {
  bucket.values.acl == "public-read"
  msg := "S3 buckets must not have public ACL"
}
```

## Troubleshooting and Scenarios (46-50)

### 46. Plan shows unexpected changes: debug steps?

```bash
terraform refresh
terraform state show aws_instance.web
TF_LOG=DEBUG terraform plan
```

### 47. Apply hangs or partial failure: recovery?

```bash
terraform state rm aws_instance.web
terraform apply -target=aws_security_group.web
```

### 48. Multi-team collaboration without conflicts?

```
s3://tfstate/
├── platform/global.tfstate
├── team-web/app.tfstate
└── team-api/app.tfstate
```

### 49. Cost optimization with Terraform modules?

```hcl
instance_type = var.optimize ? "t3a.micro" : "t3.micro"
```

### 50. Blue-green deployment pattern in Terraform?

**Process:**
1. Deploy green alongside blue
2. Validate → cutover traffic
3. Destroy blue

---

**Best Practices Summary:**
- Always use remote state (S3 + DynamoDB)
- Pin provider/module versions
- Separate environments by directory
- Use modules for all reusable infra
- CI/CD with approval gates
- Secrets via SSM/Secrets Manager
```
```

[1](https://www.datacamp.com/blog/terraform-interview-questions)

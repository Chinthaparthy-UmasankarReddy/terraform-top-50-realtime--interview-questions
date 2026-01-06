
# Terraform EC2 for Dev and Prod (Shared Module Pattern)

This layout uses a shared EC2 module and two environment folders (`dev` and `prod`) with separate state and variables, so the **code stays the same, only configuration changes**.

---

## 1. Directory Structure

```text
project-root/
├── modules/
│   └── ec2_app/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── envs/
    ├── dev/
    │   ├── main.tf
    │   ├── dev.tfvars
    │   └── backend.tf
    └── prod/
        ├── main.tf
        ├── prod.tfvars
        └── backend.tf
```

- `modules/ec2_app` holds reusable EC2 logic.  
- `envs/dev` and `envs/prod` wire the module with different instance types, sizes, tags, etc., and have **separate state backends**. 

---

## 2. EC2 Module (Shared for All Envs)

`modules/ec2_app/variables.tf`:

```hcl
variable "env" {}
variable "instance_type" {}
variable "subnet_id" {}
variable "security_group_ids" {
  type = list(string)
}
variable "key_name" {}
variable "ami_id" {}
```

`modules/ec2_app/main.tf`:

```hcl
resource "aws_instance" "app" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids
  key_name               = var.key_name

  tags = {
    Name        = "app-${var.env}"
    Environment = var.env
    ManagedBy   = "Terraform"
  }
}
```

`modules/ec2_app/outputs.tf`:

```hcl
output "instance_id" {
  value = aws_instance.app.id
}

output "public_ip" {
  value = aws_instance.app.public_ip
}
```

This module is **identical** for dev and prod; only inputs differ.

---

## 3. Dev Environment

`envs/dev/backend.tf` (example remote state):

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-dev"
    key            = "ec2/app/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks-dev"
    encrypt        = true
  }
}
```

`envs/dev/main.tf`:

```hcl
provider "aws" {
  region = "ap-south-1"
}

module "ec2_app" {
  source             = "../../modules/ec2_app"
  env                = "dev"
  instance_type      = var.instance_type
  subnet_id          = var.subnet_id
  security_group_ids = var.security_group_ids
  key_name           = var.key_name
  ami_id             = var.ami_id
}
```

`envs/dev/dev.tfvars`:

```hcl
instance_type      = "t3.micro"
subnet_id          = "subnet-dev-public-1"
security_group_ids = ["sg-dev-app"]
key_name           = "dev-keypair"
ami_id             = "ami-0dev123456789abc" # dev/cheaper/base AMI
```

---

## 4. Prod Environment

`envs/prod/backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-prod"
    key            = "ec2/app/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks-prod"
    encrypt        = true
  }
}
```

`envs/prod/main.tf`:

```hcl
provider "aws" {
  region = "ap-south-1"
}

module "ec2_app" {
  source             = "../../modules/ec2_app"
  env                = "prod"
  instance_type      = var.instance_type
  subnet_id          = var.subnet_id
  security_group_ids = var.security_group_ids
  key_name           = var.key_name
  ami_id             = var.ami_id
}
```

`envs/prod/prod.tfvars`:

```hcl
instance_type      = "t3.large"
subnet_id          = "subnet-prod-private-1"
security_group_ids = ["sg-prod-app"]
key_name           = "prod-keypair"
ami_id             = "ami-0prod123456789abc" # hardened/golden AMI
```

Here, prod uses:

- Bigger instance type and private subnet.  
- Different security group and hardened AMI.

---

## 5. Commands and Flow

From `envs/dev`:

```bash
terraform init
terraform apply -var-file=dev.tfvars
```

From `envs/prod` (often via CI/CD):

```bash
terraform init
terraform apply -var-file=prod.tfvars
```

Interview points to emphasize:

- **Same module, different tfvars** → DRY and safer promotion from dev to prod.  
- **Separate backends & state** for dev and prod to avoid accidental cross‑impact. 

If you share your current layout (folders/modules), this can be adapted exactly to your repo.

[1](https://stackoverflow.com/questions/78988059/how-can-i-create-two-separate-environments-dev-and-prod-for-deploying-aws-reso)
[2](https://www.gruntwork.io/blog/how-to-manage-multiple-environments-with-terraform-using-workspaces)
[3](https://www.reddit.com/r/devops/comments/1bs7vx8/how_to_promote_aws_terraform_from_staging_to_prod/)
[4](https://dev.to/kalio/provisioning-an-ec2-instance-with-terraform-5737)
[5](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)
[6](https://blog.devops.dev/building-a-cloud-dev-environment-with-terraform-and-aws-a-complete-guide-e8beaddb6bc1)
[7](https://cloudwithdj.com/building-a-dev-environment-on-aws-using-terraform-part-ii/)
[8](https://awstip.com/infrastructure-as-code-building-a-production-ready-aws-environment-with-terraform-88d10b81ddae)
[9](https://www.reddit.com/r/aws/comments/1bc8yda/dev_setup_with_terraform/)

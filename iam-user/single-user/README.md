
# Terraform: Single IAM User Guide

This guide shows how to create and manage a **single AWS IAM user** with Terraform, including optional console access and access keys, following least‑privilege and security best practices.

## 1. Files and Structure

```text
iam-single-user/
├── main.tf
├── variables.tf
└── outputs.tf
```

---

## 2. Provider and Variables

`variables.tf`:

```hcl
variable "aws_region" {
  type    = string
  default = "ap-south-1"
}

variable "user_name" {
  type    = string
  default = "terraform-managed-user"
}

variable "tags" {
  type = map(string)
  default = {
    ManagedBy = "Terraform"
    Purpose   = "DemoSingleUser"
  }
}
```

`main.tf` (provider + basic backend example):

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  # Optional: remote state for IAM
  # backend "s3" {
  #   bucket         = "company-terraform-state"
  #   key            = "iam/single-user.tfstate"
  #   region         = "ap-south-1"
  #   dynamodb_table = "terraform-locks"
  #   encrypt        = true
  # }
}

provider "aws" {
  region = var.aws_region
}
```

Best practice is to keep IAM state in a secure, versioned, locked backend (such as S3 + DynamoDB).

---

## 3. Single IAM User (Core Resource)

```hcl
resource "aws_iam_user" "single" {
  name = var.user_name
  path = "/terraform-managed/"

  # force_destroy = false means user must be clean before destroy
  force_destroy = false

  tags = var.tags
}
```

Key points:

- `force_destroy = false` prevents the user from being destroyed while access keys or MFA devices still exist, which is safer in most environments. 
- The `path` helps group Terraform‑managed users logically (e.g., `/terraform-managed/`).
---

## 4. Optional: Programmatic Access (Access Key)

For a system or CI user, you may create an access key. Treat the secret as **sensitive** and store it securely (e.g., password manager or secrets manager).
```hcl
resource "aws_iam_access_key" "single_key" {
  user = aws_iam_user.single.name
}
```

`outputs.tf`:

```hcl
output "user_name" {
  value       = aws_iam_user.single.name
  description = "IAM user name"
}

output "access_key_id" {
  value       = aws_iam_access_key.single_key.id
  description = "Access key ID (DO NOT commit this anywhere)"
  sensitive   = true
}

output "secret_access_key" {
  value       = aws_iam_access_key.single_key.secret
  description = "Secret access key (store securely, not in Git)"
  sensitive   = true
}
```

Best practices to mention:

- Never commit access keys to version control; outputs should be `sensitive = true` and quickly moved into a safe store. 
- Prefer roles and temporary credentials for automation; use users only when necessary.
---

## 5. Optional: Console Access for Human User

To give this user AWS Console access with a one‑time password reset:

```hcl
resource "aws_iam_user_login_profile" "single_console" {
  user                    = aws_iam_user.single.name
  password_reset_required = true
}
```

In real setups, you would combine this with baseline policies enforcing MFA at the account level and attach appropriate permissions via groups or policies.  

---

## 6. Permissions (Attach Policy to the User)

Example: attach AWS managed `ReadOnlyAccess` directly to the user.

```hcl
resource "aws_iam_user_policy_attachment" "readonly_attach" {
  user       = aws_iam_user.single.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

For least privilege, prefer **custom policies** scoped only to required actions/resources, created via `aws_iam_policy` and attached to the user or (better) to a group.  

---

## 7. Usage

Initialize and apply:

```bash
terraform init
terraform plan
terraform apply
```

After `apply`:

- Confirm the user exists with `aws iam get-user --user-name <user_name>`.  
- Copy the access key outputs once and store them securely; Terraform cannot show the secret again later. 

---

## 8. Interview Talking Points

When asked about “IAM single user with Terraform,” you can mention:

- Use `aws_iam_user` as the core resource; optionally add `aws_iam_access_key` and `aws_iam_user_login_profile` for programmatic and console access.  
- Keep credentials out of code and outputs marked as sensitive; prefer roles and SSO for humans and long‑lived IAM users only for specific cases.  
- Use a secure remote backend for IAM state and avoid manual changes in AWS to prevent drift. 



[1](https://www.youtube.com/watch?v=arCaEMMITH0)
[2](https://spacelift.io/blog/terraform-iam-user)
[3](https://blog.purestorage.com/purely-technical/how-to-use-aws-iam-with-terraform/)
[4](https://scalr.com/learning-center/streamlining-aws-iam-role-creation-with-terraform-a-practical-guide/)
[5](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user)
[6](https://www.reddit.com/r/aws/comments/1ef5nzt/how_to_configure_iam_using_terraform/)
[7](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/manage-aws-permission-sets-dynamically-by-using-terraform.html)
[8](https://github.com/cloudposse/terraform-aws-iam-system-user)
[9](https://discuss.hashicorp.com/t/terraform-resources-for-aws-iam-identity-center-successor-to-aws-single-sign-on/47310)
[10](https://stackoverflow.com/questions/40631977/how-do-i-use-terraform-to-maintain-manage-iam-users)















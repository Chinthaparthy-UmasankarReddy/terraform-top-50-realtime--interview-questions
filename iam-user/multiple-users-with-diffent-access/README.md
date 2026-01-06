
# Terraform: Multiple IAM Users With Different Access

This guide shows how to manage **multiple IAM users, each with different permissions**, using Terraform. It uses:
- One IAM user per person/service.
- One IAM group + policy per access level (e.g., readonly, dev-power, admin-lite).
- Each user is added to the appropriate group.

---

## 1. Structure

```text
iam-multi-users-different-access/
├── main.tf
├── variables.tf
└── outputs.tf
```

---

## 2. Variables and Provider

`variables.tf`:

```hcl
variable "aws_region" {
  type    = string
  default = "ap-south-1"
}

variable "readonly_users" {
  type    = list(string)
  default = ["alice"]
}

variable "dev_power_users" {
  type    = list(string)
  default = ["bob"]
}

variable "admin_lite_users" {
  type    = list(string)
  default = ["carol"]
}
```

`main.tf` (provider):

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
  region = var.aws_region
}
```

---

## 3. IAM Users (All Users)

Create one IAM user per name across all three lists.

```hcl
locals {
  all_users = toset(
    concat(
      var.readonly_users,
      var.dev_power_users,
      var.admin_lite_users
    )
  )
}

resource "aws_iam_user" "users" {
  for_each = local.all_users

  name = each.value
  path = "/terraform-managed/"

  tags = {
    ManagedBy = "Terraform"
    Type      = "human"
  }
}
```

This gives you distinct IAM users with consistent tagging.
---

## 4. Groups and Policies (Different Access Levels)

### 4.1 Read‑only Group and Policy

```hcl
resource "aws_iam_group" "readonly" {
  name = "readonly-group"
  path = "/terraform-managed/"
}

resource "aws_iam_policy" "readonly_policy" {
  name        = "ReadonlyPolicy"
  description = "Read-only access to most AWS resources"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ec2:Describe*", "s3:List*", "s3:Get*", "rds:Describe*", "iam:List*"]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_group_policy_attachment" "readonly_attach" {
  group      = aws_iam_group.readonly.name
  policy_arn = aws_iam_policy.readonly_policy.arn
}
```

### 4.2 Dev‑Power Group and Policy

```hcl
resource "aws_iam_group" "dev_power" {
  name = "dev-power-group"
  path = "/terraform-managed/"
}

resource "aws_iam_policy" "dev_power_policy" {
  name        = "DevPowerPolicy"
  description = "Developers can manage dev resources but not IAM"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:*",
          "s3:*",
          "rds:*",
          "cloudwatch:*",
          "logs:*"
        ]
        Resource = "*"
      },
      {
        Effect   = "Deny"
        Action   = ["iam:*", "organizations:*", "account:*"]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_group_policy_attachment" "dev_power_attach" {
  group      = aws_iam_group.dev_power.name
  policy_arn = aws_iam_policy.dev_power_policy.arn
}
```

### 4.3 Admin‑Lite Group and Policy

```hcl
resource "aws_iam_group" "admin_lite" {
  name = "admin-lite-group"
  path = "/terraform-managed/"
}

resource "aws_iam_policy" "admin_lite_policy" {
  name        = "AdminLitePolicy"
  description = "Broad admin-like access with guardrails on critical services"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["*"]
        Resource = "*"
      },
      {
        Effect = "Deny"
        Action = [
          "iam:DeleteUser",
          "iam:DeleteRole",
          "iam:DetachRolePolicy",
          "iam:PutUserPolicy",
          "organizations:*",
          "account:*"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_group_policy_attachment" "admin_lite_attach" {
  group      = aws_iam_group.admin_lite.name
  policy_arn = aws_iam_policy.admin_lite_policy.arn
}
```

These examples are intentionally simple; in real environments you would tighten scopes and use least‑privilege based on activity and IAM Access Analyzer.
---

## 5. Assign Users to Groups (Different Access Per User)

### 5.1 Read‑only Users

```hcl
resource "aws_iam_user_group_membership" "readonly_membership" {
  for_each = toset(var.readonly_users)

  user = aws_iam_user.users[each.value].name

  groups = [
    aws_iam_group.readonly.name
  ]
}
```

### 5.2 Dev‑Power Users

```hcl
resource "aws_iam_user_group_membership" "dev_power_membership" {
  for_each = toset(var.dev_power_users)

  user = aws_iam_user.users[each.value].name

  groups = [
    aws_iam_group.dev_power.name
  ]
}
```

### 5.3 Admin‑Lite Users

```hcl
resource "aws_iam_user_group_membership" "admin_lite_membership" {
  for_each = toset(var.admin_lite_users)

  user = aws_iam_user.users[each.value].name

  groups = [
    aws_iam_group.admin_lite.name
  ]
}
```

Each user ends up in exactly one group; you can allow multiple membership if you want combined permissions.

---

## 6. Optional: Programmatic Access Keys Per User

For CI/service users, not recommended for humans when SSO/roles are available.

```hcl
resource "aws_iam_access_key" "keys" {
  for_each = aws_iam_user.users

  user = each.value.name
}

output "user_access_keys" {
  value = {
    for uname, key in aws_iam_access_key.keys :
    uname => {
      access_key_id     = key.id
      secret_access_key = key.secret
    }
  }
  sensitive = true
}
```

Store these secrets in a secure system immediately; do not commit them to Git. 

---

## 7. Apply

```bash
terraform init
terraform plan
terraform apply
```

You can customize which users go into which access level by editing `readonly_users`, `dev_power_users`, or `admin_lite_users` in `terraform.tfvars`.

---

## 8. Interview Talking Points

When explaining “multiple users with different access”:

- Use **groups + policies** to represent access levels, then assign users to those groups instead of attaching policies directly per user.  
- Use `for_each` over user lists to create many users with clean Terraform code and consistent tagging. 
- Apply **least‑privilege** and adjust policies over time using IAM Access Analyzer/CloudTrail data to reduce over‑permissioning.  
- Prefer roles and Identity Center (permission sets) for large orgs, using users only where necessary.

[1](https://www.reddit.com/r/Terraform/comments/wc5hsd/how_do_you_handle_aws_permissions_for_terraform/)
[2](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/manage-aws-permission-sets-dynamically-by-using-terraform.html)
[3](https://blog.purestorage.com/purely-technical/how-to-use-aws-iam-with-terraform/)
[4](https://stackoverflow.com/questions/51273227/whats-the-most-efficient-way-to-determine-the-minimum-aws-permissions-necessary)
[5](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user)
[6](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/security.html)
[7](https://discuss.hashicorp.com/t/handling-aws-permissions-across-modules/49429)
[8](https://community.amazonquicksight.com/t/guide-for-aws-permissions-when-using-terraform/46489)
[9](https://terrateam.io/blog/using-multiple-aws-iam-roles)
[10](https://www.reddit.com/r/Terraform/comments/11ry6wi/authenticating_to_multiple_aws_accounts_for/)

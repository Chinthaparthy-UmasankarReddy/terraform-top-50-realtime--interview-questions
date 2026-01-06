For “multiple IAM users with the same access,” the recommended pattern is: **one group or shared policy**, attach that to the group, and add all users to that group. This keeps permissions consistent and easier to manage than attaching policies user‑by‑user.

# Terraform: Multiple IAM Users With the Same Access

This guide shows how to create **multiple IAM users** that all share the **same permissions** by using a group and a shared policy.

---

## 1. Files and Structure

```text
iam-multi-users/
├── main.tf
├── variables.tf
└── outputs.tf
```

---

## 2. Variables

`variables.tf`:

```hcl
variable "aws_region" {
  type    = string
  default = "ap-south-1"
}

variable "user_names" {
  description = "IAM users that should all have the same permissions"
  type        = list(string)
  default     = ["dev-user-1", "dev-user-2"]
}
```

---

## 3. Provider

`main.tf` (provider block):

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

## 4. IAM Users (Multiple)

Create one `aws_iam_user` per entry in `var.user_names` using `for_each`.

```hcl
resource "aws_iam_user" "users" {
  for_each = toset(var.user_names)

  name = each.value
  path = "/terraform-managed/"

  tags = {
    ManagedBy = "Terraform"
    Team      = "dev"
  }
}
```

This produces users like:

- `/terraform-managed/dev-user-1`
- `/terraform-managed/dev-user-2`

---

## 5. Group + Policy (Shared Access)

### 5.1 IAM Group

```hcl
resource "aws_iam_group" "dev_group" {
  name = "dev-group"
  path = "/terraform-managed/"
}
```

### 5.2 Shared Policy (Example: S3 Read/Write to One Bucket)

```hcl
resource "aws_iam_policy" "dev_group_policy" {
  name        = "DevGroupS3Access"
  description = "Allow dev users to read/write a specific S3 bucket"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = [
          "arn:aws:s3:::my-dev-bucket",
          "arn:aws:s3:::my-dev-bucket/*"
        ]
      }
    ]
  })
}
```

### 5.3 Attach Policy to Group (Once)

```hcl
resource "aws_iam_group_policy_attachment" "dev_group_attach" {
  group      = aws_iam_group.dev_group.name
  policy_arn = aws_iam_policy.dev_group_policy.arn
}
```

All group members now inherit this policy.
---

## 6. Add All Users to the Group

Use `aws_iam_user_group_membership` to place all users in the same group.

```hcl
resource "aws_iam_user_group_membership" "dev_group_membership" {
  for_each = aws_iam_user.users

  user = each.value.name

  groups = [
    aws_iam_group.dev_group.name
  ]
}
```

Now every user in `var.user_names`:

- Is created as an IAM user.
- Is added to `dev-group`.
- Gets identical permissions via the shared group policy.

---

## 7. Optional: Access Keys for Each User

If these are **programmatic** users (not recommended for humans when SSO/roles are available), you can issue keys per user. Handle secrets very carefully.
```hcl
resource "aws_iam_access_key" "user_keys" {
  for_each = aws_iam_user.users

  user = each.value.name
}

output "access_keys" {
  value = {
    for uname, key in aws_iam_access_key.user_keys :
    uname => {
      access_key_id     = key.id
      secret_access_key = key.secret
    }
  }

  sensitive = true
}
```

---

## 8. Apply

```bash
terraform init
terraform plan
terraform apply
```

---

## 9. Interview Talking Points

When asked about “multiple IAM users with the same access,” highlight:

- **Use a group**: put shared permissions on the group, not per‑user; all users inherit the same policy. 
- Use `for_each` over a list of usernames to create many users with small, clean Terraform code.  
- Avoid duplicating policies; one policy attached to the group simplifies updates and auditing.  
- For humans, prefer SSO/roles; long‑lived users/keys are mainly for legacy or specific automation cases. 



[1](https://zeet.co/blog/terraform-multiple-users)
[2](https://www.firefly.ai/academy/automating-iam-across-cloud-platforms-with-terraform)
[3](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/govern-permission-sets-aft.html)
[4](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/manage-aws-permission-sets-dynamically-by-using-terraform.html)
[5](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user_group_membership)
[6](https://www.reddit.com/r/Terraform/comments/11ry6wi/authenticating_to_multiple_aws_accounts_for/)
[7](https://terrateam.io/blog/using-multiple-aws-iam-roles)
[8](https://discuss.hashicorp.com/t/managing-iam-permissions-across-multiple-workspaces-in-tfe/74145)
[9](https://www.abbey.io/blog/create-and-manage-aws-iam-users-and-roles-with-terraform/)
[10](https://stackoverflow.com/questions/71905931/multiple-iam-users-in-terraform)

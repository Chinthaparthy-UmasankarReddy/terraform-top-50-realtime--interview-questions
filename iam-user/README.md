
# Terraform + AWS IAM User Interview Guide 

This document summarizes common Terraform + AWS IAM **user** interview questions and concise, production‑style answers, with example Terraform configurations. It targets roughly 3–4 years of hands‑on AWS and Terraform experience.

---

## 1. IAM User Concepts & Terraform State

### 1.1 How does Terraform track IAM users? What if state is lost or edited?

Terraform tracks IAM users, policies, and access keys in the **state file**, mapping logical resources (e.g., `aws_iam_user.dev_user`) to actual IAM users in AWS (e.g., `arn:aws:iam::123456789012:user/dev-user`).  
If the state is lost or manually edited, Terraform may think IAM users do not exist and try to recreate them, potentially regenerating access keys and breaking applications that depend on the old credentials.

**Best practices:**

- Store state remotely (e.g., S3 backend with DynamoDB locking) and enable versioning.]  
- Do not keep long‑lived IAM access keys in code or version control; use outputs securely and rotate keys regularly. 
- Use `terraform state` commands (not manual file edits) for imports and corrections. 

---

## 2. Creating IAM Users with Terraform

### 2.1 How do you create an IAM user?

Use `aws_iam_user` to define the user and attach tags; optionally define console password and access keys with separate resources. 

Example:

```hcl
resource "aws_iam_user" "dev_user" {
  name = "dev-user"

  tags = {
    Team        = "dev"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

### 2.2 How do you create console access and access keys for a user?

- Console access: use `aws_iam_user_login_profile` to set an initial password and password reset requirement.
- Programmatic access: use `aws_iam_access_key` to create access keys, then provide them securely to users or systems (e.g., secret manager, password manager). 

Example:

```hcl
resource "aws_iam_user_login_profile" "dev_user_console" {
  user                    = aws_iam_user.dev_user.name
  password_reset_required = true
}

resource "aws_iam_access_key" "dev_user_key" {
  user = aws_iam_user.dev_user.name
}

output "dev_user_access_key_id" {
  value       = aws_iam_access_key.dev_user_key.id
  description = "Access key ID for dev-user"
  sensitive   = true
}

output "dev_user_secret_access_key" {
  value       = aws_iam_access_key.dev_user_key.secret
  description = "Secret access key for dev-user"
  sensitive   = true
}
```

---

## 3. Attaching Policies to IAM Users

### 3.1 Difference between AWS‑managed, customer‑managed, and inline policies in Terraform?

- **AWS‑managed**: predefined policies by AWS (e.g., `AmazonS3ReadOnlyAccess`), attached via ARN using `aws_iam_user_policy_attachment`. 
- **Customer‑managed**: reusable policies you define with `aws_iam_policy`, then attach to multiple users/roles/groups.  
- **Inline policies**: `aws_iam_user_policy` resources directly attached to a specific user; good for very user‑specific permissions but less reusable.

Example – attach AWS‑managed policy:

```hcl
resource "aws_iam_user_policy_attachment" "dev_user_s3_ro" {
  user       = aws_iam_user.dev_user.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}
```

Example – customer‑managed policy:

```hcl
resource "aws_iam_policy" "dev_s3_rw" {
  name        = "DevS3ReadWrite"
  description = "Read/write access to dev S3 bucket"

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

resource "aws_iam_user_policy_attachment" "dev_user_s3_rw_attach" {
  user       = aws_iam_user.dev_user.name
  policy_arn = aws_iam_policy.dev_s3_rw.arn
}
```

Example – inline policy:

```hcl
resource "aws_iam_user_policy" "dev_user_inline" {
  name = "DevUserInlinePolicy"
  user = aws_iam_user.dev_user.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ec2:DescribeInstances"]
        Resource = "*"
      }
    ]
  })
}
```

---

## 4. IAM Groups, Roles, and Least Privilege

### 4.1 How do you manage IAM groups for multiple users?

- Use `aws_iam_group` to define groups like `developers`, `ops`, `readonly`.  
- Attach policies to groups using `aws_iam_group_policy_attachment` so you do not repeat attachments per user.  
- Add users to groups with `aws_iam_user_group_membership` to scale permission management. 

Example:

```hcl
resource "aws_iam_group" "developers" {
  name = "developers"
}

resource "aws_iam_group_policy_attachment" "developers_power_user" {
  group      = aws_iam_group.developers.name
  policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
}

resource "aws_iam_user" "dev_user1" {
  name = "dev-user-1"
}

resource "aws_iam_user" "dev_user2" {
  name = "dev-user-2"
}

resource "aws_iam_user_group_membership" "dev_group_membership" {
  user = aws_iam_user.dev_user1.name

  groups = [
    aws_iam_group.developers.name,
  ]
}
```

You can similarly add multiple users or define separate memberships.

### 4.2 IAM user vs role – how do you explain and use them in Terraform?

- **IAM user**: represents a person or long‑lived identity; can have console passwords and access keys.  
- **IAM role**: assumed by users, services, or external accounts; does not have long‑lived keys and is better for workloads (EC2/EKS/Lambda) and temporary elevated access.
In Terraform:

- Use `aws_iam_role` + `aws_iam_role_policy_attachment` for workloads and cross‑account access.  
- Use users only where necessary (e.g., human users) and prefer roles with SSO for production.  

---

## 5. Access Keys, Rotation, and Security

### 5.1 How do you manage IAM access keys securely?

- Create access keys with `aws_iam_access_key` but treat the secret as **sensitive output** and immediately store it in a secure system (password manager, secrets manager, Vault, etc.). 
- Avoid committing any keys to Git, logs, or chat; mark outputs as `sensitive = true` and limit who can see Terraform outputs.
- Enforce rotation policies and minimum necessary permissions on each user’s attached policies. 

Example (secure outputs):

```hcl
resource "aws_iam_access_key" "ci_user_key" {
  user = aws_iam_user.ci_user.name
}

output "ci_user_access_key_id" {
  value     = aws_iam_access_key.ci_user_key.id
  sensitive = true
}

output "ci_user_secret_access_key" {
  value     = aws_iam_access_key.ci_user_key.secret
  sensitive = true
}
```

### 5.2 How do you rotate keys with Terraform?

- Create a **new** `aws_iam_access_key` for the user.  
- Update all systems to use the new key.  
- Then **destroy** (or mark inactive) the old key using Terraform (remove it from code and apply).

A safer pattern is to have, at most, two keys per user (one active, one being rotated) and explicitly manage them in Terraform so state matches reality.

---

## 6. Importing Existing IAM Users into Terraform

### 6.1 How do you bring existing IAM users under Terraform management?

Steps:

1. Define the user resource in Terraform (e.g., `resource "aws_iam_user" "existing_user" { name = "alice" }`).  
2. Use `terraform import aws_iam_user.existing_user alice` to populate state.
3. Run `terraform plan` and reconcile configuration with reality (tags, policies, etc.).  

Same pattern applies for `aws_iam_user_policy`, `aws_iam_user_policy_attachment`, and `aws_iam_access_key`. This avoids Terraform destroying or recreating existing users and keys unexpectedly.

---

## 7. Tagging, MFA, and Best Practices

### 7.1 Tagging IAM users

- Use tags to capture team, environment, cost center, and whether the user is human vs service.  
- Enforce consistent tags via locals or modules.

Example:

```hcl
locals {
  base_user_tags = {
    ManagedBy = "Terraform"
    Type      = "human"
  }
}

resource "aws_iam_user" "alice" {
  name = "alice"

  tags = merge(local.base_user_tags, {
    Team        = "data"
    Environment = "prod"
  })
}
```

### 7.2 MFA and security posture

Even though MFA settings aren’t fully modeled as Terraform resources for each user, good answers include:

- Require MFA for console users via IAM policies or account settings.  
- Use roles + SSO (e.g., AWS IAM Identity Center) instead of many long‑lived IAM users.  
- Give users minimal permissions and use groups and roles to manage access centrally.

---

## 8. Example: Simple IAM Stack for a Team

```hcl
# Group for developers
resource "aws_iam_group" "developers" {
  name = "developers"
}

# Limited access policy for developers
resource "aws_iam_policy" "developers_policy" {
  name        = "DevelopersLimitedAccess"
  description = "Developers can view most resources and manage dev stack only"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ec2:Describe*", "iam:List*", "cloudwatch:List*", "cloudwatch:Get*"]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "cloudformation:CreateStack",
          "cloudformation:UpdateStack",
          "cloudformation:DeleteStack"
        ]
        Resource = "arn:aws:cloudformation:ap-south-1:123456789012:stack/dev-*/*"
      }
    ]
  })
}

resource "aws_iam_group_policy_attachment" "developers_policy_attach" {
  group      = aws_iam_group.developers.name
  policy_arn = aws_iam_policy.developers_policy.arn
}

# Example developer user
resource "aws_iam_user" "dev_user" {
  name = "dev-user"

  tags = {
    Team        = "dev"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}

resource "aws_iam_user_group_membership" "dev_user_membership" {
  user = aws_iam_user.dev_user.name

  groups = [
    aws_iam_group.developers.name,
  ]
}
```




[1](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-manage)
[2](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/aws-create)

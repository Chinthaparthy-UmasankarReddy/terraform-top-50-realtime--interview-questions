
# Terraform + AWS S3 Interview Guide

This document summarizes common Terraform + AWS S3 interview questions and concise, production‑style answers, with example Terraform configurations, for about 3–4 years of hands‑on AWS/Terraform experience.

---

## 1. S3 Basics & Terraform State

### 1.1 How do you create a basic S3 bucket with Terraform?

Use the `aws_s3_bucket` resource with at least a unique bucket name and region‑appropriate settings.

```hcl
provider "aws" {
  region = "ap-south-1"
}

resource "aws_s3_bucket" "logs" {
  bucket = "my-company-logs-ap-south-1"

  tags = {
    Name        = "logs-bucket"
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}
```

Key talking points:

- Bucket names are **globally unique**.
- You typically add tags from day one for cost and ownership tracking.
---

### 1.2 How does Terraform state interact with S3?

- S3 buckets and related resources (bucket policy, lifecycle rules, etc.) are all tracked in the Terraform state file like any other AWS resource.  
- If you use **S3 as a remote backend** for Terraform state, that is a separate bucket (often versioned, encrypted, restricted) from your application buckets.

Example backend:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "envs/prod/core/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

You should be able to explain:

- Difference between “S3 for state backend” vs “S3 buckets you manage as resources”.  
- Why versioning, encryption, and tight IAM on the state bucket are important.
---

## 2. Bucket Configuration: Versioning, Encryption, Logging

### 2.1 How do you enable versioning?

Use `versioning` block on `aws_s3_bucket` or a separate `aws_s3_bucket_versioning` resource depending on provider version.

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-company-data-ap-south-1"
}

resource "aws_s3_bucket_versioning" "data_versioning" {
  bucket = aws_s3_bucket.data.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

Why:

- Protects against accidental deletes/overwrites.
- Often considered a best practice for critical or state‑like data.

---

### 2.2 How do you configure server‑side encryption (SSE)?

Use `aws_s3_bucket_server_side_encryption_configuration` (new style) or `server_side_encryption_configuration` nested block.

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "data_sse" {
  bucket = aws_s3_bucket.data.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
      kms_master_key_id = aws_kms_key.s3_data.arn
    }
  }
}
```

Key points:

- For compliance, SSE with KMS (`aws:kms`) is often required.
- You can use AWS‑managed KMS keys or customer‑managed keys depending on requirements.

---

### 2.3 How do you configure access logging for an S3 bucket?

Use a dedicated **log bucket** and configure `aws_s3_bucket_logging` on the source bucket.

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-company-access-logs-ap-south-1"
}

resource "aws_s3_bucket" "static_site" {
  bucket = "my-company-static-site-ap-south-1"
}

resource "aws_s3_bucket_logging" "static_site_logging" {
  bucket = aws_s3_bucket.static_site.id

  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "static-site/"
}
```

You should mention:

- Log bucket usually blocks public access and is tightly permissioned.
- Logging is important for audit and incident response.

---

## 3. Security: Public Access, Policies, and ACLs

### 3.1 How do you secure an S3 bucket with Terraform?

Common measures:

- **Block public access** (account or bucket level).
- Use **bucket policies** + **IAM** instead of ACLs where possible.
- Enforce SSE, TLS, and sometimes specific principals.

Example: block public access and restrict access to a VPC endpoint or IAM role:

```hcl
resource "aws_s3_bucket_public_access_block" "data_block_public" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Bucket policy allowing only a specific IAM role:

```hcl
data "aws_iam_role" "app_role" {
  name = "app-role"
}

resource "aws_s3_bucket_policy" "data_policy" {
  bucket = aws_s3_bucket.data.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowAppRoleAccess"
        Effect = "Allow"
        Principal = {
          AWS = data.aws_iam_role.app_role.arn
        }
        Action   = ["s3:GetObject", "s3:PutObject"]
        Resource = [
          "${aws_s3_bucket.data.arn}/*"
        ]
      }
    ]
  })
}
```

Mention:

- Avoid public buckets unless for specific use cases (e.g., static websites), and then control tightly with policies.  
- Prefer IAM + bucket policy over ACLs for modern setups.

---

### 3.2 How do you create a **public** static website bucket safely?

- Turn off “block all public access” only for that bucket.
- Use a restrictive policy that only allows public `GetObject` and nothing else.

Example:

```hcl
resource "aws_s3_bucket" "website" {
  bucket = "my-company-website-ap-south-1"

  website {
    index_document = "index.html"
    error_document = "error.html"
  }
}

resource "aws_s3_bucket_public_access_block" "website_pab" {
  bucket                  = aws_s3_bucket.website.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_policy" "website_policy" {
  bucket = aws_s3_bucket.website.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = ["s3:GetObject"]
        Resource  = "${aws_s3_bucket.website.arn}/*"
      }
    ]
  })
}
```

---

## 4. Lifecycle Rules, Storage Classes, and Cost Optimization

### 4.1 How do you configure lifecycle rules (e.g., transition to Glacier)?

Use `aws_s3_bucket_lifecycle_configuration` or a `lifecycle_rule` block to transition or expire objects.

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "data_lifecycle" {
  bucket = aws_s3_bucket.data.id

  rule {
    id     = "transition-logs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }
}
```

You should be able to explain:

- Different storage classes (STANDARD, STANDARD_IA, ONEZONE_IA, INTELLIGENT_TIERING, GLACIER, DEEP_ARCHIVE).  
- Using lifecycle rules to lower storage costs over time.

---

### 4.2 How do you enforce retention for compliance?

- Use lifecycle `expiration` to delete objects after a certain period.
- Combine with **object lock** and versioning in more advanced compliance scenarios (though object lock is not fully Terraform‑managed in all flows yet).

Mention that for strict compliance (e.g., WORM), you might use:

- S3 Object Lock with retention modes and compliance/legal holds.
- Dedicated policies preventing deletion by most principals.

---

## 5. Cross‑Account Access & CloudFront

### 5.1 How do you allow cross‑account access to an S3 bucket?

Typical pattern:

- In **bucket account**, create a bucket policy granting access to IAM roles or principals in another AWS account.  
- Optionally, use KMS key with cross‑account grants if SSE‑KMS is enabled.

Example (allow read access to another account’s role):

```hcl
variable "external_account_id" {}

resource "aws_s3_bucket_policy" "data_cross_account" {
  bucket = aws_s3_bucket.data.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowExternalAccountRead"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.external_account_id}:role/external-analytics-role"
        }
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*"
        ]
      }
    ]
  })
}
```

---

### 5.2 How do you integrate S3 with CloudFront using Terraform?

High‑level:

- Create S3 bucket for content.
- Create `aws_cloudfront_distribution` with S3 bucket as origin and appropriate cache behavior.
Very simplified example:

```hcl
resource "aws_s3_bucket" "static" {
  bucket = "my-company-static-cdn-ap-south-1"
}

resource "aws_cloudfront_distribution" "cdn" {
  enabled = true

  origin {
    domain_name = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id   = "s3-static-origin"
  }

  default_cache_behavior {
    target_origin_id       = "s3-static-origin"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

You should mention:

- Often, S3 origin access is restricted to CloudFront using Origin Access Control/Identity instead of public S3 access.  

---

## 6. Importing Existing Buckets and Drift

### 6.1 How do you bring an existing S3 bucket under Terraform?

Steps:

1. Add the resource to Terraform code:  
   `resource "aws_s3_bucket" "existing" { bucket = "my-existing-bucket" }`  
2. Run `terraform import aws_s3_bucket.existing my-existing-bucket`.  
3. Add related resources (policy, lifecycle, encryption) and import them as well.  
4. Run `terraform plan` and align configuration with live settings to remove drift.

Explain that this avoids Terraform trying to recreate or destroy a production bucket inadvertently.

---

## 7. Tagging, Naming, and Best Practices

### 7.1 How do you tag and name S3 buckets consistently?

- Use standardized naming conventions per environment and application.
- Keep tags consistent across buckets for cost allocation, ownership, and environment.

Example:

```hcl
locals {
  base_tags = {
    ManagedBy = "Terraform"
    Owner     = "platform-team"
  }
}

resource "aws_s3_bucket" "app_data" {
  bucket = "my-company-app-data-prod-ap-south-1"

  tags = merge(local.base_tags, {
    Environment = "prod"
    Application = "my-app"
  })
}
```

### 7.2 S3 security best practices you should mention

When asked open‑ended questions like *“How do you secure S3?”*, good points:
- Block public access by default; allow public only for specific, audited use cases.  
- Use bucket policies + IAM, avoid legacy ACLs where possible.  
- Enable versioning and server‑side encryption (preferably KMS).  
- Use lifecycle rules to manage data retention and cost.  
- Enable access logging and monitor via CloudTrail/CloudWatch and security tooling.  

---





[1](https://www.finalroundai.com/blog/terraform-interview-questions)
[2](https://thinkcloudly.com/blog/aws-interview-questions/aws-s3-interview-questions/)
[3](https://www.heyvaldemar.com/10-real-terraform-interview-questions-and-expert-answers-2025-devops-guide)
[4](https://www.interviews.chat/questions/terraform-engineer)
[5](https://www.secondtalent.com/interview-guide/terraform/)
[6](https://aws.plainenglish.io/369-most-frequently-asked-terraform-interview-questions-with-detailed-answers-f52e6406fd71)
[7](https://k21academy.com/terraform/terraform-interview-questions/)
[8](https://www.reddit.com/r/Terraform/comments/16y9fi3/gauging_terraform_skill_level_for_a_staff/)
[9](https://www.interviewbit.com/terraform-interview-questions/)
[10](https://www.geeksforgeeks.org/devops/terraform-interview-questions/)

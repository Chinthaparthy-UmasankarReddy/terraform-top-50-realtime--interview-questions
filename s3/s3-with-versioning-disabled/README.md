For an S3 bucket with **versioning disabled**, the main point is that you either leave versioning unset (bucket stays unversioned), or explicitly configure versioning as “unversioned/disabled” depending on the AWS provider version. AWS itself only supports three states conceptually: unversioned (default), versioning‑enabled, and versioning‑suspended; once enabled, a bucket cannot be returned to truly unversioned, only suspended.

# Terraform: S3 Bucket With Versioning Disabled

This guide shows how to define an AWS S3 bucket in Terraform with **versioning disabled/unmanaged**, and what that means in practice.

---

## 1. Basic S3 Bucket Without Versioning

The simplest way to keep a bucket **unversioned** is to not configure versioning at all. New S3 buckets are unversioned by default.

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

resource "aws_s3_bucket" "no_versioning" {
  bucket = "my-no-versioning-bucket-ap-south-1"

  tags = {
    Name        = "no-versioning-bucket"
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

Key points:

- With **no versioning block**, AWS keeps the bucket in the default unversioned state. 
- This is suitable only if you explicitly do **not** want historical object versions (e.g., for non‑critical temporary data).  

---

## 2. Explicit Versioning Configuration (Suspended vs Enabled)

Once versioning is enabled, AWS does not allow going back to truly “unversioned”; you can only **suspend** versioning. Terraform models this with a versioning configuration resource/blocks:

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-logs-bucket-ap-south-1"

  tags = {
    Name        = "logs-bucket"
    Environment = "stage"
  }
}

resource "aws_s3_bucket_versioning" "logs_versioning" {
  bucket = aws_s3_bucket.logs.id

  versioning_configuration {
    status = "Suspended" # or "Enabled"
  }
}
```

Notes:

- `status = "Suspended"` means no new versions are created, but old versions remain and can still be accessed. 
- For a bucket that was **never versioned**, omitting `aws_s3_bucket_versioning` keeps it unversioned; adding it with `"Suspended"` typically reflects a versioning‑suspended state in Terraform’s model for newer provider versions.  

---

## 3. When to Keep Versioning Disabled

Good interview/real‑world talking points:

- **State / critical data buckets**: best practice is to **enable** versioning (e.g., Terraform state bucket) to protect against accidental deletes; disabling here is usually discouraged. 
- **Non‑critical, high‑churn data** (temporary exports, scratch space) may intentionally remain unversioned to save cost and avoid clutter.  
- Security/scan tools often flag “versioning disabled” as a risk for important data, so you should be ready to justify why a bucket is intentionally unversioned or suspended. 

---

## 4. Summary Patterns

- **Brand‑new bucket, no version history needed** → omit any versioning configuration; bucket stays unversioned.  
- **Bucket that must never grow history for cost reasons** → keep unversioned or use `Suspended` with clear justification and monitoring.  
- **Any important or long‑lived data (logs, backups, state)** → prefer versioning enabled and explain why disabling is a bad idea except for specific cases.  



[1](https://github.com/hashicorp/terraform-provider-aws/issues/23718)
[2](https://github.com/hashicorp/terraform-provider-aws/issues/30824)
[3](https://github.com/orgs/gruntwork-io/discussions/729)
[4](https://dev.to/codebrewster/using-terraform-to-create-a-bucket-with-versioning-enabled-extra-an-easy-to-use-script-456m)
[5](https://docs.aws.amazon.com/codeguru/detector-library/terraform/disabled-s3-versioning-terraform/)
[6](https://www.reddit.com/r/aws/comments/e1f0yh/how_to_permanently_remove_s3_versioning/)
[7](https://stackoverflow.com/questions/54122531/terraform-rewriting-tag-and-versioning-info-when-using-aws-s3-bucket)
[8](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
[9](https://nasa.github.io/cumulus/docs/deployment/terraform-best-practices/)
[10](https://stackoverflow.com/questions/71532246/unable-to-add-versioning-configuration-for-multiple-aws-s3-in-terraform-version)

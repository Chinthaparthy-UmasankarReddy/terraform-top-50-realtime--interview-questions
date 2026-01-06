An S3 bucket with **versioning enabled** keeps multiple versions of each object so you can recover from accidental overwrite or delete, and this is strongly recommended for important data like logs, backups, and Terraform state.

# Terraform: S3 Bucket With Versioning Enabled

This guide shows how to create an AWS S3 bucket with **object versioning enabled** using Terraform. Versioning protects against accidental overwrites and deletions by retaining previous versions of objects.

---

## 1. Simple Bucket With Versioning Enabled (Inline Block)

A straightforward pattern is to enable versioning directly inside the bucket resource (syntax depends on provider version, but the idea is the same).
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

resource "aws_s3_bucket" "versioned" {
  bucket = "my-versioned-bucket-ap-south-1"

  tags = {
    Name        = "versioned-bucket"
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "versioned_config" {
  bucket = aws_s3_bucket.versioned.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

Key points:

- `status = "Enabled"` turns on versioning; new versions are stored whenever you overwrite or delete an object.  
- Old versions remain accessible until lifecycle rules remove them, which is useful for recovery but has cost implications.
---

## 2. Bucket With Versioning and Encryption (Common Production Pattern)

For important data (e.g., logs, Terraform state) you typically enable **both versioning and encryption**.
```hcl
resource "aws_s3_bucket" "secure_versioned" {
  bucket = "my-secure-versioned-bucket-ap-south-1"

  tags = {
    Name        = "secure-versioned-bucket"
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "secure_versioned_config" {
  bucket = aws_s3_bucket.secure_versioned.id

  versioning_configuration {
    status = "Enabled"
  }
}

# SSE with S3-managed keys (SSE-S3)
resource "aws_s3_bucket_server_side_encryption_configuration" "secure_sse" {
  bucket = aws_s3_bucket.secure_versioned.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

Why this matters:

- Versioning + encryption is a standard recommendation for buckets holding critical data such as Terraform state, logs, and backups.[web:105][web:110]  
- Versioning allows rollbacks of state files or log recovery after accidental deletion or corruption.  

---

## 3. Using a Versioned Bucket for Terraform State

When using S3 as a remote backend, enabling versioning on the **state bucket** is considered a best practice.

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tf-state-prod"
    key            = "envs/prod/core/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks-prod"
    encrypt        = true
  }
}
```

Configure that state bucket (outside the backend block) as a versioned bucket:

```hcl
resource "aws_s3_bucket" "tf_state" {
  bucket = "my-tf-state-prod"

  tags = {
    Name        = "tf-state-prod"
    Environment = "prod"
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "tf_state_versioning" {
  bucket = aws_s3_bucket.tf_state.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

Benefits:

- If someone corrupts or deletes the state file, you can restore an older version from S3 history. 

---

## 4. Lifecycle and Cost Considerations

Because every overwrite or delete creates a new version, storage can grow quickly:

- Use lifecycle rules to transition old versions to cheaper storage classes or expire them after some time.  
- For critical data (e.g., state, compliance logs), retention periods are often long; for other buckets, shorter retention can control cost.
---

## 5. Interview Talking Points

When asked about “S3 with versioning enabled” in Terraform, mention:

- Enable versioning via `aws_s3_bucket_versioning` (or equivalent inline block) with `status = "Enabled"`.  
- Versioning is strongly recommended for state buckets, logs, and backups because it allows recovery from accidental changes. 
- Combine versioning with encryption, lifecycle rules, and restricted access for a production‑grade S3 design.  



[1](https://dev.to/jumpingcats/using-terraform-to-create-a-bucket-with-versioning-enabled-extra-an-easy-to-use-script-456m)
[2](https://dev.to/codebrewster/using-terraform-to-create-a-bucket-with-versioning-enabled-extra-an-easy-to-use-script-456m)
[3](https://github.com/hashicorp/terraform-provider-aws/issues/22221/linked_closing_reference)
[4](https://nasa.github.io/cumulus/docs/deployment/terraform-best-practices/)
[5](https://stackoverflow.com/questions/54122531/terraform-rewriting-tag-and-versioning-info-when-using-aws-s3-bucket)
[6](https://www.youtube.com/watch?v=rm9CvOGVBdQ)
[7](https://docs.aws.amazon.com/codeguru/detector-library/terraform/disabled-s3-versioning-terraform/)
[8](https://developer.hashicorp.com/terraform/language/backend/s3)
[9](https://stackoverflow.com/questions/71532246/unable-to-add-versioning-configuration-for-multiple-aws-s3-in-terraform-version)
[10](https://www.reddit.com/r/Terraform/comments/lehvdk/how_to_use_aws_s3_bucket_as_version_control/)


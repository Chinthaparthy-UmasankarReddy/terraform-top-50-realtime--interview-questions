
# Terraform: Inbound & Outbound “All” Rules (Security Groups)

This guide shows how to configure AWS Security Groups in Terraform when you want to **allow all inbound**, **allow all outbound**, or **both**, and why this is usually only acceptable for testing.

> Important: In production, broad `0.0.0.0/0` rules are a security risk and should be replaced with least‑privilege rules.

---

## 1. Allow-All Inbound and Allow-All Outbound

This security group allows **any traffic from anywhere** in and out. Use only in isolated labs.

```hcl
resource "aws_security_group" "allow_all" {
  name        = "allow-all"
  description = "ALLOW ALL inbound and outbound (test only)"
  vpc_id      = aws_vpc.main.id

  # INBOUND: allow all
  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"          # all protocols
    cidr_blocks = ["0.0.0.0/0"] # all IPv4
  }

  # OUTBOUND: allow all
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "allow-all"
    Environment = "lab"
  }
}
```

Notes:

- `protocol = "-1"` means all protocols.
- `from_port = 0`, `to_port = 0` with `-1` is interpreted as “all ports.”
- `0.0.0.0/0` means any IPv4 address.

---

## 2. Allow-All Outbound, Restricted Inbound (Common Pattern)

Typical real‑world pattern: **inbound restricted, outbound open**.

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Restricted inbound, all outbound"
  vpc_id      = aws_vpc.main.id

  # INBOUND: only HTTP/HTTPS from internet
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # OUTBOUND: allow all
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 3. Block-All Inbound and Block-All Outbound

To **deny all** traffic, simply omit both `ingress` and `egress` and set explicit “block all” via separate rules if needed. A minimal “no traffic” SG:

```hcl
resource "aws_security_group" "deny_all" {
  name        = "deny-all"
  description = "No inbound, no outbound"
  vpc_id      = aws_vpc.main.id

  # No ingress blocks  => no inbound
  # No egress blocks   => no outbound (Terraform does NOT add default allow-all)
}
```

Or, with explicit rule resources, you just don’t create any rules for that SG.

---

## 4. Quick Interview Summary

- **Allow-all inbound & outbound**: use `protocol = "-1"`, ports 0–0, CIDR `0.0.0.0/0` for both `ingress` and `egress` – but only for test labs.  
- **Safe default**: restricted inbound + allow-all outbound.  
- **Deny-all**: no `ingress` and no `egress` blocks; Terraform will not add the AWS console’s default egress rule, so nothing is allowed.



[1](https://dev.to/aws-builders/automated-way-to-restrict-all-inbound-and-outbound-rules-from-aws-default-security-groups-22o1)
[2](https://orca.security/resources/blog/default-security-group-is-being-used/)
[3](https://stackoverflow.com/questions/78115191/make-aws-default-security-groups-limit-all-inbound-and-outbound-traffic)
[4](https://stackoverflow.com/questions/71695580/aws-security-group-what-is-needed-in-inbound-and-outbound-rules)
[5](https://www.corestack.io/aws-security-best-practices/aws-security-group-best-practices/)
[6](https://docs.aws.amazon.com/vpc/latest/userguide/default-security-group.html)
[7](https://docs.aws.amazon.com/quicksuite/latest/userguide/vpc-security-groups.html)
[8](https://www.youtube.com/watch?v=CW_3D1tL3_I)
[9](https://www.reddit.com/r/aws/comments/18y0gpj/security_groups_outbound/)
[10](https://www.reddit.com/r/aws/comments/nmihli/security_groups_inbound_and_outbound_purpose/)

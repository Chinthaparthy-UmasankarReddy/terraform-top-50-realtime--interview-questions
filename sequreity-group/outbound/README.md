In security groups, “outbound” means **egress** rules: what traffic your instance is allowed to send out. Terraform models this with `egress` blocks (or `aws_security_group_rule` with `type = "egress"`).

## Allow all outbound traffic

Terraform removes AWS’s default allow‑all egress, so you must define it explicitly if you want it.[1]

```hcl
resource "aws_security_group" "app_sg" {
  name        = "app-sg"
  description = "App SG with all outbound"
  vpc_id      = aws_vpc.main.id

  # inbound rules (ingress) here...

  # outbound: allow all
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

This lets the instance reach any destination, any port, any protocol, which is AWS’s default UX but must be explicit in Terraform.[7][1]

## Restrict outbound traffic

You can instead allow only specific outbound ports, for example HTTP/HTTPS only.[3]

```hcl
resource "aws_security_group" "web_sg" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  # inbound rules...

  # outbound HTTP
  egress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # outbound HTTPS
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Key interview points:

- Outbound rules control **where instances can talk to**, not who can reach them; that is handled by inbound (`ingress`).[7]
- Terraform **does not keep AWS’s default allow‑all egress**, so you must define egress explicitly, either open or restricted.[1]

[1](https://stackoverflow.com/questions/55023605/aws-and-terraform-default-egress-rule-in-security-group)
[2](https://stackoverflow.com/questions/56186225/how-to-list-the-inbound-and-outbound-rules-for-aws-security-group-using-terrafor)
[3](https://www.jit.io/resources/cloud-sec-tools/developers-tutorial-to-managing-aws-security-groups-through-terraform)
[4](https://www.youtube.com/watch?v=n2hxAW29hFY)
[5](https://github.com/terraform-aws-modules/terraform-aws-security-group)
[6](https://dev.to/af/hcl-practice-pt2-security-groups-nat-internet-gateways-and-route-tables-with-terraform-on-aws-11nj)
[7](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
[8](https://www.reddit.com/r/Terraform/comments/1ci6blz/aws_security_group_rules_terraform/)
[9](https://dev.to/suzuki0430/terraform-beginners-aws-security-group-management-3pl3)
[10](https://www.ducktyped.org/p/an-illustrated-guide-to-aws-security)

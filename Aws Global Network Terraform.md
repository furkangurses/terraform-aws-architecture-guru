# aws-global-network-terraform

> Multi-region AWS network fabric provisioned entirely with Terraform — VPC mesh, Transit Gateway peering, VPC peering, S3 gateway endpoints, and EC2 test harness. Remote state in S3 + DynamoDB locking.

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.5-7B42BC?logo=terraform&logoColor=white)
![AWS Provider](https://img.shields.io/badge/AWS%20Provider-~%3E5.0-FF9900?logo=amazon-aws&logoColor=white)
![Regions](https://img.shields.io/badge/Regions-us--east--1%20%7C%20us--west--2%20%7C%20ap--east--1-blue)
![State Backend](https://img.shields.io/badge/State-S3%20%2B%20DynamoDB-green)

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repository Layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Remote State Backend Setup](#remote-state-backend-setup)
- [Deployment Order](#deployment-order)
- [Per-Region VPC Design](#per-region-vpc-design)
- [Transit Gateway Peering](#transit-gateway-peering)
- [VPC Peering (Alternative)](#vpc-peering-alternative)
- [EC2 Test Harness](#ec2-test-harness)
- [Cost Considerations](#cost-considerations)
- [TGW vs VPC Peering — Decision Matrix](#tgw-vs-vpc-peering--decision-matrix)
- [Engineering Notes](#engineering-notes)
- [Teardown](#teardown)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Global Network Fabric                       │
│                                                                     │
│  us-east-1 (10.101.0.0/16)     us-west-2 (10.103.0.0/16)          │
│  ┌─────────────────────┐       ┌─────────────────────┐             │
│  │  VPC: prod-main     │       │  VPC: prod-main     │             │
│  │  ├─ pub-b/c/d       │       │  ├─ pub-a/b/c       │             │
│  │  ├─ priv-b/c/d      │       │  ├─ priv-a/b/c      │             │
│  │  ├─ NAT GW x3       │       │  ├─ NAT GW x3       │             │
│  │  ├─ S3 GW Endpoint  │       │  ├─ S3 GW Endpoint  │             │
│  │  └─ TGW / VPC Peer  │◄─────►│  └─ TGW / VPC Peer  │            │
│  └─────────────────────┘       └─────────────────────┘             │
│            ▲                             ▲                          │
│            └──────────────┬─────────────┘                          │
│                           │                                         │
│              ap-east-1 (10.106.0.0/16)                             │
│              ┌─────────────────────┐                               │
│              │  VPC: prod-main     │                               │
│              │  ├─ pub-a/b/c       │                               │
│              │  ├─ priv-a/b/c      │                               │
│              │  ├─ NAT GW x3       │                               │
│              │  ├─ S3 GW Endpoint  │                               │
│              │  └─ TGW / VPC Peer  │                               │
│              └─────────────────────┘                               │
└─────────────────────────────────────────────────────────────────────┘
```

Each VPC is a self-contained, multi-AZ network (3 public + 3 private subnets). Networks are connected in a **full mesh** using either Transit Gateway peering or VPC peering (not both simultaneously). The two approaches are implemented in separate Terraform stacks and can be swapped out independently.

---

## Repository Layout

```
.
├── global/
│   └── prod/
│       ├── _backend.tf           # S3 remote state — global scope
│       ├── _outputs.tf           # VPN CIDR output (example global variable)
│       ├── _variables.tf
│       └── main/
│               ├── _backend.tf   # S3 remote state — global/prod/main
│               ├── _outputs.tf   # default_tags output consumed by all stacks
│               ├── _variables.tf # Cost center, environment, managed-by tags
│               ├── ec2/          # Multi-region EC2 test harness + IAM profile
│               ├── transit-gateway-peering/   # TGW mesh (mutually exclusive with vpcpeering)
│               └── vpc-peering/               # VPC peer mesh (mutually exclusive with TGW)
│
├── us-east-1/
│   └── prod/
│       └── main/
│           ├── _backend.tf
│           ├── _region.tf        # AWS provider pinned to us-east-1
│           ├── _remotes.tf       # Data sources: global + global/prod/main state
│           ├── _variables.tf     # CIDR, AZs, subnet ranges
│           ├── _outputs.tf       # VPC ID, subnet IDs, route table IDs
│           ├── vpc.tf            # terraform-aws-modules/vpc/aws
│           └── endpoints.tf      # S3 gateway endpoint
│
├── us-west-2/
│   └── prod/main/  (same structure, CIDR 10.103.0.0/16)
│
├── ap-east-1/
│   └── prod/main/  (same structure, CIDR 10.106.0.0/16)
│
└── modules/
    ├── transit-gateway-peering/  # Custom: TGW peering attachment + acceptor + route table entries
    └── vpc-peering-routes/       # Custom: for_each over route table IDs → peering routes
```

> **Design decision — one directory per region per stack:** Stacks are intentionally separated so that a change to `ap-east-1` never triggers a plan diff in `us-east-1`. This also allows independent state locking and parallel `terraform apply` across regions during provisioning.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Terraform ≥ 1.5 | `brew install hashicorp/tap/terraform` or [tfenv](https://github.com/tfutils/tfenv) |
| AWS CLI v2 | Credentials configured via `aws configure` or environment variables |
| S3 bucket (globally unique) | Created manually before first `terraform init` — see [backend setup](#remote-state-backend-setup) |
| DynamoDB table | `LockID` (String) partition key — see below |
| EC2 key pair (per region) | Same key pair name must exist in all three regions if using the test harness |

### AWS credential check

```bash
aws sts get-caller-identity
# Expected output:
# {
#     "UserId": "AIDA...",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/deploy-svc"
# }
```

---

## Remote State Backend Setup

### S3 Bucket

Created once, manually, before any Terraform is run. Bucket name must be globally unique — prefix with your org or domain slug.

```bash
# Replace <ORG_SLUG> with your organisation identifier
BUCKET_NAME="<ORG_SLUG>-terraform-global-network-state"
REGION="us-east-1"

aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION"

# Enable versioning (recommended — enables state file recovery)
aws s3api put-bucket-versioning \
  --bucket "$BUCKET_NAME" \
  --versioning-configuration Status=Enabled

# Block all public access
aws s3api put-public-access-block \
  --bucket "$BUCKET_NAME" \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Tag for cost tracking
aws s3api put-bucket-tagging \
  --bucket "$BUCKET_NAME" \
  --tagging 'TagSet=[{Key=CostCenter,Value=platform},{Key=ManagedBy,Value=manual}]'
```

### DynamoDB Lock Table

```bash
aws dynamodb create-table \
  --table-name "terraform-state-locking" \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

> **Note:** DynamoDB table names are account-scoped, not globally unique. `terraform-state-locking` can be reused verbatim without changes to any backend configuration in this repo.

### Backend configuration (per stack)

Each stack's `_backend.tf` follows the directory path as the S3 object key:

```hcl
# global/prod/main/_backend.tf
terraform {
  backend "s3" {
    bucket         = "<ORG_SLUG>-terraform-global-network-state"
    key            = "global/prod/main/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locking"
    encrypt        = true
  }
}
```

```hcl
# us-east-1/prod/main/_backend.tf
terraform {
  backend "s3" {
    bucket         = "<ORG_SLUG>-terraform-global-network-state"
    key            = "us-east-1/prod/main/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locking"
    encrypt        = true
  }
}
```

> **If you fork this repo**, update the `bucket` value in every `_backend.tf`. The `key` values are intentionally designed to mirror the directory structure and should not need changes.

---

## Deployment Order

Dependencies flow top-to-bottom. Stacks in the same group can be applied in parallel.

```
Step 1 ── global/prod/              (VPN CIDR placeholder — no real resources)
Step 2 ── global/prod/main/         (global default_tags output)

Step 3 ── [PARALLEL]
          ├── us-east-1/prod/main/  (VPC + S3 endpoint)
          ├── us-west-2/prod/main/  (VPC + S3 endpoint)
          └── ap-east-1/prod/main/  (VPC + S3 endpoint)

Step 4 ── global/prod/main/ec2/     (IAM profile, security groups, EC2 instances)

Step 5 ── [CHOOSE ONE — mutually exclusive]
          ├── global/prod/main/transit-gateway-peering/
          └── global/prod/main/vpc-peering/
```

### Initialise and apply (single stack)

```bash
cd us-east-1/prod/main

terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

### Parallel regional deployment (Steps 3)

```bash
# Terminal 1
(cd us-east-1/prod/main && terraform init && terraform apply -auto-approve)

# Terminal 2
(cd us-west-2/prod/main && terraform init -upgrade && terraform apply -auto-approve)

# Terminal 3
(cd ap-east-1/prod/main && terraform init -upgrade && terraform apply -auto-approve)
```

---

## Per-Region VPC Design

Each regional stack deploys **32 AWS resources** via the [`terraform-aws-modules/vpc/aws`](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws) module.

<details>
<summary>Resource inventory (32 per region)</summary>

| Resource | Count | Notes |
|---|---|---|
| `aws_vpc` | 1 | DNS hostnames + resolution enabled |
| `aws_subnet` (public) | 3 | One per AZ |
| `aws_subnet` (private) | 3 | One per AZ |
| `aws_internet_gateway` | 1 | |
| `aws_internet_gateway_attachment` | 1 | Separate Terraform object from IGW |
| `aws_eip` | 3 | One per NAT GW |
| `aws_nat_gateway` | 3 | One per AZ — eliminates cross-AZ NATGW traffic charges |
| `aws_route_table` (public) | 1 | Shared across all public subnets |
| `aws_route_table` (private) | 3 | One per AZ — routes to AZ-local NAT GW |
| `aws_route_table_association` | 6 | Public x3 + Private x3 |
| `aws_route` (IGW) | 1 | `0.0.0.0/0 → igw` on public RT |
| `aws_route` (NATGW) | 3 | `0.0.0.0/0 → natgw` on each private RT |
| `aws_vpc_endpoint` (S3 gateway) | 1 | Attached to all three private route tables |

</details>

### Module call — `vpc.tf`

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "us-east-1-prod-main-vpc"
  cidr = var.vpc_cidr   # 10.101.0.0/16

  azs             = var.availability_zones       # ["us-east-1b", "us-east-1c", "us-east-1d"]
  public_subnets  = var.public_subnet_cidrs      # ["10.101.0.0/24", "10.101.1.0/24", "10.101.2.0/24"]
  private_subnets = var.private_subnet_cidrs     # ["10.101.10.0/24", "10.101.11.0/24", "10.101.12.0/24"]

  public_subnet_suffix  = "pub"
  private_subnet_suffix = "priv"

  enable_dns_hostnames = true
  enable_dns_support   = true

  enable_nat_gateway  = true
  single_nat_gateway  = false   # One NAT GW per AZ — no cross-AZ charges
  reuse_nat_ips       = true
  external_nat_ip_ids = aws_eip.nat[*].id

  tags = merge(
    var.default_tags,
    data.terraform_remote_state.global_prod_main.outputs.default_tags
  )
}

resource "aws_eip" "nat" {
  count  = length(var.availability_zones)
  domain = "vpc"
  tags   = merge(var.default_tags, { Name = "us-east-1-prod-main-natgw-eip-${count.index}" })
}
```

### S3 Gateway Endpoint — `endpoints.tf`

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"

  # Associate with all private route tables to keep same-region S3 traffic off the NAT GW
  route_table_ids = flatten(module.vpc.private_route_table_ids)

  tags = merge(var.default_tags, { Name = "us-east-1-prod-main-s3-gw-endpoint" })
}
```

> **Why `flatten()`?** The VPC module outputs `private_route_table_ids` as a list-of-lists. The `aws_vpc_endpoint` resource expects a flat list. This is a known quirk of the module's output shape.

---

## Transit Gateway Peering

Deployed from a single directory (`global/prod/main/transit-gateway-peering/`) across all three regions in one `terraform apply`. This stack creates **63 resources** total.

### What the HashiCorp TGW module covers

- Transit Gateway resource per region
- VPC attachment (private subnets only)
- Default TGW route table with local VPC CIDR entry

### What the custom `modules/transit-gateway-peering` adds

- Cross-region peering attachment (requester side)
- Peering attachment acceptor (destination region)
- TGW route table entries for remote VPC CIDRs
- VPC private subnet route table entries pointing at the local TGW

### Peering attachment pattern

```hcl
# modules/transit-gateway-peering/main.tf

resource "aws_ec2_transit_gateway_peering_attachment" "this" {
  transit_gateway_id      = var.source_tgw_id
  peer_transit_gateway_id = var.destination_tgw_id
  peer_region             = var.destination_region

  tags = merge(var.tags, {
    Name = "${var.name}-peering-attachment"
    Side = "requester"
  })
}

resource "aws_ec2_transit_gateway_peering_attachment_accepter" "this" {
  provider                        = aws.destination
  transit_gateway_attachment_id   = aws_ec2_transit_gateway_peering_attachment.this.id
  transit_gateway_id              = var.destination_tgw_id

  tags = merge(var.tags, {
    Name = "${var.name}-peering-attachment"
    Side = "accepter"
  })
}
```

### Route table entries (VPC subnets)

```hcl
# Loop over all private route table IDs in the source VPC
resource "aws_route" "to_remote_via_tgw" {
  for_each = toset(var.source_vpc_route_table_ids)

  route_table_id         = each.value
  destination_cidr_block = var.destination_cidr   # e.g., 10.106.0.0/16
  transit_gateway_id     = var.source_tgw_id
}
```

### Known behaviour — timeout on first apply

Transit gateway VPC attachments can take 2–3 minutes per region. On the first `apply`, TGW route table entries occasionally receive a `400 DependencyViolation` because the attachment hasn't fully propagated before the route API call is made. **Mitigation:** run `terraform apply` a second time. The TGW and attachment already exist; only the 6 failed route entries are created, completing in under 30 seconds.

```bash
cd global/prod/main/transit-gateway-peering

terraform init -upgrade
terraform apply         # May end with 400 on route table entries
terraform apply         # Idempotent second pass — creates only the remaining routes
```

---

## VPC Peering (Alternative)

Deployed from `global/prod/main/vpc-peering/`. Creates **24 resources** — no Transit Gateway overhead.

### Requester / Acceptor split

AWS requires the peering acceptor to be a distinct resource when VPCs are in different regions, even within the same account:

```hcl
# us-east-1 → us-west-2 peering
resource "aws_vpc_peering_connection" "use1_usw2" {
  provider    = aws.us-east-1
  vpc_id      = data.terraform_remote_state.use1.outputs.vpc_id
  peer_vpc_id = data.terraform_remote_state.usw2.outputs.vpc_id
  peer_region = "us-west-2"
  auto_accept = false   # Cross-region — must use accepter resource

  # DNS resolution cannot be enabled on requester until connection is ACTIVE
  # Uncomment AFTER first apply:
  # requester {
  #   allow_remote_vpc_dns_resolution = true
  # }

  tags = merge(local.default_tags, { Name = "use1-to-usw2-peer" })
}

resource "aws_vpc_peering_connection_accepter" "use1_usw2" {
  provider                  = aws.us-west-2
  vpc_peering_connection_id = aws_vpc_peering_connection.use1_usw2.id
  auto_accept               = true

  accepter {
    allow_remote_vpc_dns_resolution = true
  }

  tags = merge(local.default_tags, { Name = "use1-to-usw2-peer-accepter" })
}
```

> **AWS constraint (not a Terraform limitation):** DNS resolution on the *requester* side of a cross-region VPC peering connection cannot be enabled until the connection status is `active`. The workaround is a two-pass apply: first apply with the `requester` block commented out, then uncomment and apply again.

### Route injection (custom module)

```hcl
# modules/vpc-peering-routes/main.tf

# Add remote VPC route to every route table in the source VPC
resource "aws_route" "source_to_destination" {
  for_each = toset(var.source_route_table_ids)

  route_table_id            = each.value
  destination_cidr_block    = var.destination_cidr
  vpc_peering_connection_id = var.peering_connection_id
}

# Add source VPC route to every route table in the destination VPC
resource "aws_route" "destination_to_source" {
  provider = aws.destination

  for_each = toset(var.destination_route_table_ids)

  route_table_id            = each.value
  destination_cidr_block    = var.source_cidr
  vpc_peering_connection_id = var.peering_connection_id
}
```

---

## EC2 Test Harness

One Amazon Linux 2 instance per region, deployed into the first private subnet of each VPC. Used exclusively to validate cross-region connectivity via ICMP ping.

**Access method:** AWS Systems Manager Session Manager — no SSH keys, no bastion host, no inbound port 22.

### AMI selection (dynamic)

```hcl
data "aws_ami" "amazon_linux2_use1" {
  provider    = aws.us-east-1
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

### IAM instance profile

```hcl
resource "aws_iam_role" "ssm_ec2" {
  name = "prod-main-ec2-ssm-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = local.default_tags
}

locals {
  instance_policies = [
    "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore",
    "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy",
  ]
}

resource "aws_iam_role_policy_attachment" "ssm_ec2" {
  for_each   = toset(local.instance_policies)
  role       = aws_iam_role.ssm_ec2.name
  policy_arn = each.value
}

resource "aws_iam_instance_profile" "ssm_ec2" {
  name = "prod-main-ec2-ssm-profile"
  role = aws_iam_role.ssm_ec2.name
}
```

> **IAM policy attachment limit:** `aws_iam_role_policy_attachment` with `for_each` works cleanly up to the AWS hard limit of 10 managed policies per role. If you need more, consider an inline policy with a JSON document.

### Connectivity validation

```bash
# Connect via Session Manager (no SSH required)
aws ssm start-session \
  --target i-0abc1234567890def \
  --region us-east-1

# From the session — ping ap-east-1 instance private IP
ping -c 10 10.106.10.45
```

Expected results:

| Connection type | ~RTT (us-east-1 → ap-east-1) |
|---|---|
| Transit Gateway | ~187 ms |
| VPC Peering | ~200–205 ms |

> The slightly higher RTT with VPC peering in these measurements is likely attributable to transient network conditions rather than a structural difference. In same-region tests, VPC peering is typically 0.2–0.3 ms faster than TGW due to one fewer hop.

---

## Cost Considerations

### NAT Gateway

| Configuration | Hourly cost | Risk |
|---|---|---|
| 1 NAT GW per AZ (this repo) | Higher | No cross-AZ charge; no SPOF |
| 1 NAT GW shared across AZs | Lower | Cross-AZ data charge ($0.01/GB); single point of failure |

### S3 Gateway Endpoint

Routing same-region S3 traffic through the gateway endpoint (already configured in `endpoints.tf`) eliminates NAT GW data processing charges for S3. Traffic is also entirely private — it never traverses the public AWS network.

### Cross-region data transfer

| Scenario | Charges |
|---|---|
| Source EC2 → TGW attachment | $0.00 |
| TGW processing | $0.02/GB |
| Cross-region transfer | $0.02/GB |
| Destination TGW processing | $0.00 |
| Source EC2 → VPC peering | $0.00 |
| VPC peering outbound | $0.01/GB |
| Cross-region transfer | $0.02/GB |
| VPC peering inbound | $0.01/GB |

**Net cost per GB cross-region: identical** ($0.04/GB) for TGW and VPC peering. TGW VPC attachments carry an additional hourly charge (~$0.05/hr per attachment-AZ) which becomes negligible under moderate traffic.

---

## TGW vs VPC Peering — Decision Matrix

| Capability | Transit Gateway | VPC Peering |
|---|---|---|
| Same-region auto-accept | ✅ | ✅ |
| Cross-region auto-accept | ❌ | ❌ |
| Subnet-level attachment granularity | ✅ | ❌ (VPC-level only) |
| Same-region private DNS → private IP | ✅ | ✅ |
| Cross-region private DNS → private IP | ✅ | ✅ |
| Cross-region public DNS → private IP (EC2) | ❌ | ✅ |
| Flow log monitoring | ✅ (TGW flow logs) | ❌ |
| On-premises connectivity (VPN / Direct Connect) | ✅ | ❌ |
| Max connections per VPC | ~5,000 attachments | 125 peers |
| Provisioning time per connection | 2–4 min | < 30 sec |
| Cross-region data cost | Equal | Equal |

**Choose Transit Gateway when:** you need on-premises integration, flow log visibility, subnet-level routing granularity, or expect > 125 VPCs in the mesh.

**Choose VPC Peering when:** your mesh is small (< 10 VPCs), provisioning speed matters, you need cross-region public-DNS-to-private-IP resolution, and you want zero TGW attachment hourly cost.

---

## Engineering Notes

### Tag strategy

All resources carry a standard tag set sourced from the `global/prod/main` remote state output. Individual stacks override `tf_repo_directory` to point to the exact directory that manages a resource — making `git blame`-equivalent lookups trivial from the AWS console.

```
CostCenter       = "platform"
ManagedBy        = "terraform"
Environment      = "prod"
tf_repo_directory = "us-east-1/prod/main"   # overridden per stack
```

### CIDR design for scalability

The current design assigns `/16` per VPC with no regional supernet. This requires explicit per-VPC CIDR entries in TGW route tables. A more scalable alternative (useful from initial build, difficult to retrofit):

```
Region 1: 10.96.0.0/13  → VPCs: 10.96.0.0/16, 10.97.0.0/16, ...
Region 2: 10.104.0.0/13 → VPCs: 10.104.0.0/16, 10.105.0.0/16, ...
Region 3: 10.112.0.0/13 → VPCs: 10.112.0.0/16, 10.113.0.0/16, ...
```

With supernets, adding a second VPC in a region requires no changes to TGW route tables in other regions — the `/13` summary route already covers it.

### `.gitignore` rationale

```gitignore
**/.terraform/         # 288 MB provider binary per directory — never commit
*.tfstate              # State files live in S3; local copies are stale by definition
*.tfstate.backup
*.tfplan               # Plan outputs contain sensitive values
.terraform.lock.hcl    # Optionally commit this for provider version pinning
```

### Idempotency and the 400 timeout

The TGW route table 400 error on first apply is a known race condition between the TGW attachment reaching `available` state and Terraform immediately attempting to create route table entries. Running `terraform apply` a second time is the correct resolution — the apply is idempotent and only touches the failed resources.

### Using Terraform to bootstrap the backend

It is possible (and philosophically consistent) to manage the S3 bucket and DynamoDB table with Terraform itself. The recommended workflow:

1. Write and apply a local-backend stack to create the S3 bucket and DynamoDB table.
2. Add an S3 backend block to that same stack.
3. Run `terraform init` — Terraform will prompt to migrate the local state into S3.
4. The bootstrap stack is now self-hosted.

For infrastructure that is genuinely static post-creation (the bucket never changes), manual provisioning via CLI is an equally valid and lower-complexity choice.

---

## Teardown

Destroy in reverse dependency order. Regional VPCs cannot be destroyed while a VPC peering connection or TGW VPC attachment references them.

```bash
# 1. Remove the connectivity layer first (whichever is active)
cd global/prod/main/transit-gateway-peering && terraform destroy -auto-approve
# OR
cd global/prod/main/vpc-peering && terraform destroy -auto-approve

# 2. Remove EC2 test harness
cd global/prod/main/ec2 && terraform destroy -auto-approve

# 3. Destroy regional VPCs in parallel
(cd us-east-1/prod/main && terraform destroy -auto-approve) &
(cd us-west-2/prod/main && terraform destroy -auto-approve) &
(cd ap-east-1/prod/main && terraform destroy -auto-approve) &
wait

# 4. Remove global tag outputs
cd global/prod/main && terraform destroy -auto-approve
cd global/prod      && terraform destroy -auto-approve
```

> After teardown, the S3 bucket will contain only near-empty `.tfstate` files (< 500 bytes each). DynamoDB will retain the stale lock entries. Both services remain well within AWS Free Tier at this point and can be left in place for future use or deleted manually.

- [ ] TGW / VPC peering stack destroyed
- [ ] EC2 stack destroyed
- [ ] All three regional VPCs destroyed
- [ ] Global stacks destroyed
- [ ] NAT Gateway hourly billing confirmed stopped (check Cost Explorer next day)

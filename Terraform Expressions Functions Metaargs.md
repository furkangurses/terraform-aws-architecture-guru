# terraform-expressions-functions-metaargs

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.2.0-7B42BC?logo=terraform&logoColor=white)
![Provider: AWS](https://img.shields.io/badge/Provider-AWS%20~%3E4.16-FF9900?logo=amazon-aws&logoColor=white)
![HCL](https://img.shields.io/badge/Language-HCL-blueviolet)
![License: MIT](https://img.shields.io/badge/License-MIT-green)

Reference implementations for Terraform's expression system, built-in function library, and meta-argument patterns. Covers the constructs that separate declarative boilerplate from genuinely reusable, idiomatic HCL.

---

## Table of Contents

- [Repository Layout](#repository-layout)
- [Expressions](#expressions)
  - [Type System](#type-system)
  - [Operators & Version Constraints](#operators--version-constraints)
  - [for Expressions](#for-expressions)
  - [Splat Expressions](#splat-expressions)
- [Built-in Functions](#built-in-functions)
  - [Terraform Console](#terraform-console)
  - [Function Reference by Category](#function-reference-by-category)
  - [Worked Examples](#worked-examples)
- [Meta-arguments](#meta-arguments)
  - [count](#count)
  - [for_each](#for_each)
  - [lifecycle](#lifecycle)
  - [depends_on](#depends_on)
  - [provider (alias)](#provider-alias)
- [Key Takeaways / Engineering Notes](#key-takeaways--engineering-notes)
- [Prerequisites](#prerequisites)

---

## Repository Layout

```
.
├── expressions/
│   ├── for/
│   │   └── main.tf          # for expression over variable list
│   └── splat/
│       ├── main.tf          # EC2 instance with multiple EBS volumes
│       └── outputs.tf       # splat operator to collect all volume IDs
├── functions/
│   ├── functions.tf         # timestamp, formatdate, file(), crypto hashing
│   └── test-users/
│       └── main.tf          # sandbox for console-driven function testing
└── meta-arguments/
    ├── count/
    │   ├── main.tf          # EC2 fleet via count, named with count.index
    │   └── outputs.tf       # splat over count-managed instances
    ├── for_each/
    │   └── main.tf          # IAM accounts from toset(), each.key binding
    ├── lifecycle/
    │   └── main.tf          # prevent_destroy guard on stateful resources
    └── depends_on/
        └── main.tf          # explicit ordering: instance before IAM grants
```

---

## Expressions

### Type System

Terraform's type system is small but its implications are large. Knowing which type a value is determines what operations are valid and how state serialization behaves.

| Type | HCL Example | Notes |
|------|-------------|-------|
| `string` | `"us-east-1"` | UTF-8, double-quoted |
| `number` | `3`, `0.5` | Integer or floating-point |
| `bool` | `true` / `false` | Used in conditional and lifecycle arguments |
| `list(T)` / tuple | `["10.0.0.0/24", "10.0.1.0/24"]` | Zero-indexed; element type must be uniform for `list` |
| `map(T)` / object | `{ env = "prod", region = "eu-west-1" }` | String-keyed; values accessed by label |
| `null` | `null` | Signals argument omission; triggers provider default |

> **Design note:** Prefer `null` over sentinel strings like `""` or `"none"` when you want Terraform to fall back to a provider default. Passing an empty string is not the same as omitting the argument — many providers treat them differently.

#### Index Notation

```hcl
# List element access (zero-indexed)
local.availability_zones[0]   # "us-east-1a"
local.availability_zones[1]   # "us-east-1b"

# Map/object attribute access
local.tags["environment"]     # or local.tags.environment
```

---

### Operators & Version Constraints

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # allows 5.x, blocks 6.0
    }
  }
}
```

| Operator | Meaning | Example |
|----------|---------|---------|
| `>=` | Greater than or equal | `>= 1.5.0` — any version from 1.5.0 upward |
| `~>` (pessimistic) | Rightmost component may increment | `~> 5.4` allows `5.4 – 5.99`, blocks `6.0` |
| `~>` (patch-level) | When three components given | `~> 5.4.2` allows `5.4.2 – 5.4.99`, blocks `5.5` |
| `!=` | Exclusion | `!= 5.7.0` skip a broken release |

> The `~>` operator is called the **pessimistic constraint operator**. It is the standard recommendation in the Terraform registry for provider pinning because it picks up bug-fix releases within a minor series without risking breaking changes from a major bump.

---

### for Expressions

Use `for` to transform a collection inline — normalise tags, generate ARN lists, invert a map, etc.

```hcl
variable "subnet_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
}

# Produce a map of index → CIDR (useful for aws_subnet with for_each)
locals {
  indexed_cidrs = { for idx, cidr in var.subnet_cidrs : idx => cidr }
  # { 0 = "10.0.0.0/24", 1 = "10.0.1.0/24", 2 = "10.0.2.0/24" }

  # Uppercase all environment tag values
  normalised_tags = { for k, v in var.resource_tags : k => upper(v) }
}
```

**Filtering with `if`:**

```hcl
locals {
  prod_subnets = [for s in var.subnets : s.cidr if s.env == "prod"]
}
```

---

### Splat Expressions

Splat (`[*]`) is the idiomatic way to project a single attribute across all instances of a resource managed by `count` or across a list/set.

```hcl
# expressions/splat/main.tf
resource "aws_instance" "app" {
  count         = 3
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"

  ebs_block_device {
    device_name = "/dev/sda1"
    volume_size = 20
    volume_type = "gp3"
  }

  ebs_block_device {
    device_name = "/dev/sdb"
    volume_size = 100
    volume_type = "gp3"
    encrypted   = true
  }
}
```

```hcl
# expressions/splat/outputs.tf
output "instance_ids" {
  description = "IDs of all app instances"
  value       = aws_instance.app[*].id
}

output "data_volume_ids" {
  description = "Volume IDs for the secondary data disks across all instances"
  value       = aws_instance.app[*].ebs_block_device[*].volume_id
}

output "second_instance_public_ip" {
  description = "Public IP of instance index 1 (zero-based)"
  value       = aws_instance.app[1].public_ip
}
```

> **Caveats:** The legacy dot-star syntax (`aws_instance.app.*.id`) still works as of Terraform 1.x but is considered deprecated. Use bracket notation (`[*]`) in all new code. Splat is only valid on resources managed by `count` or on list/set values — it does not work on `for_each`-managed resources (use `values()` and `for` there instead).

---

## Built-in Functions

### Terraform Console

The console is the fastest way to validate function output without touching state or cloud APIs.

```bash
# Requires an initialised working directory
terraform init
terraform console
```

```hcl
# Arithmetic
> 2 + 2
4

> pow(2, 10)
1024

# Network helpers
> cidrnetmask("10.42.0.0/16")
"255.255.0.0"

> cidrsubnets("10.0.0.0/16", 4, 4, 4, 4)
tolist(["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20", "10.0.48.0/20"])

# Type coercion
> toset(["eu-west-1", "us-east-1", "ap-southeast-1"])
toset(["ap-southeast-1", "eu-west-1", "us-east-1"])

# String manipulation
> replace("arn:aws:iam::123456789012:role/my-role", "my-role", "ci-role")
"arn:aws:iam::123456789012:role/ci-role"

# Crypto — SHA-256 digest of a deployment artifact path
> base64sha256("s3://artifacts/api-service/v2.3.1/api-service.zip")
"cGF0aC10by1hcnRpZmFjdC1oYXNo..."
```

> `terraform console` creates a transient `.terraform.tfstate` in the working directory while open. Always `exit` or `Ctrl-D` before running `apply` or `plan` — having two processes hold state simultaneously causes lock errors.

---

### Function Reference by Category

<details>
<summary>Numeric</summary>

| Function | Signature | Returns |
|----------|-----------|---------|
| `abs` | `abs(number)` | Absolute value |
| `ceil` | `ceil(number)` | Nearest integer ≥ input |
| `floor` | `floor(number)` | Nearest integer ≤ input |
| `max` | `max(a, b, ...)` | Largest value |
| `min` | `min(a, b, ...)` | Smallest value |
| `pow` | `pow(base, exp)` | Exponentiation |
| `log` | `log(number, base)` | Logarithm |

</details>

<details>
<summary>String</summary>

| Function | Use case |
|----------|----------|
| `format` | `format("%-20s %s", var.env, var.region)` |
| `formatlist` | Apply `format` across a list |
| `join` | `join(",", var.security_group_ids)` |
| `split` | `split("/", "eu-west-1/prod/api")` |
| `replace` | Regex or literal substitution |
| `trimspace` | Strip leading/trailing whitespace from user inputs |
| `chomp` | Remove trailing newline — useful with `file()` |
| `lower` / `upper` | Case normalisation for tag values |
| `substr` | Slice a string by position |
| `regex` | Return first match; fails if no match |
| `regexall` | Return all matches as a list |

</details>

<details>
<summary>Collection</summary>

| Function | Use case |
|----------|----------|
| `length` | Number of elements in list/map/string |
| `toset` | Deduplicate a list; required argument for `for_each` |
| `tolist` | Convert set → list (order not guaranteed) |
| `tomap` | Convert object → map |
| `flatten` | Collapse nested lists into a single list |
| `distinct` | Remove duplicates while preserving order |
| `concat` | Merge two or more lists |
| `merge` | Deep-merge two or more maps (last-write-wins) |
| `keys` / `values` | Extract map keys or values as a list |
| `lookup` | `lookup(map, key, default)` — safe map access |
| `contains` | Test list/set membership |
| `zipmap` | Build a map from parallel key/value lists |

</details>

<details>
<summary>Filesystem</summary>

| Function | Use case |
|----------|----------|
| `file(path)` | Inline a local file as a string (cloud-init, policy JSON) |
| `filebase64(path)` | Inline a binary file as base64 |
| `templatefile(path, vars)` | Render a `.tftpl` template with variable substitution |
| `pathexpand` | Expand `~` in user-supplied paths |

</details>

<details>
<summary>Date & Time</summary>

| Function | Use case |
|----------|----------|
| `timestamp()` | RFC 3339 timestamp at plan time — useful for tagging |
| `formatdate(spec, timestamp)` | Reformat RFC 3339 string to human-readable |
| `timeadd(time, duration)` | Add a duration string (`"24h"`, `"30m"`) to a timestamp |

</details>

<details>
<summary>Hash & Crypto</summary>

| Function | Use case |
|----------|----------|
| `sha256(string)` | Hex-encoded SHA-256 digest |
| `sha512(string)` | Hex-encoded SHA-512 digest |
| `base64sha256(string)` | Base64-encoded SHA-256 — required by some AWS API fields |
| `base64sha512(string)` | Base64-encoded SHA-512 |
| `md5(string)` | MD5 hex digest — for S3 `etag` comparisons, not security |
| `bcrypt(string)` | bcrypt hash — useful for initial password seeds |

</details>

<details>
<summary>IP Network</summary>

| Function | Use case |
|----------|----------|
| `cidrnetmask(prefix)` | Convert CIDR to dotted-decimal netmask |
| `cidrhost(prefix, hostnum)` | Generate a specific host address within a CIDR |
| `cidrsubnet(prefix, newbits, netnum)` | Carve a subnet out of a larger block |
| `cidrsubnets(prefix, newbits...)` | Generate multiple subnets in one call |

</details>

---

### Worked Examples

#### Tagging with `timestamp` and `formatdate`

```hcl
# functions/functions.tf
resource "aws_iam_user" "ci_service_account" {
  name = "ci-deploy-${var.environment}"

  tags = {
    managed_by     = "terraform"
    environment    = var.environment
    # Raw RFC 3339 — machine-readable, good for log correlation
    provisioned_at = timestamp()
    # Human-readable local date for the ops dashboard
    provisioned_on = formatdate("DD MMM YYYY, HH:mm ZZZ", timestamp())
  }
}
```

#### Cloud-init with `file()` and `templatefile()`

```hcl
resource "aws_instance" "bastion" {
  ami           = data.aws_ami.amazon_linux2.id
  instance_type = "t3.micro"

  # Static script — no variable interpolation needed
  user_data = file("${path.module}/scripts/harden-bastion.sh")
}

resource "aws_instance" "app_server" {
  ami           = data.aws_ami.amazon_linux2.id
  instance_type = "m5.large"

  # Templated cloud-init — injects runtime values
  user_data = templatefile("${path.module}/templates/cloud-init.tftpl", {
    app_version    = var.app_version
    s3_config_path = "s3://${var.config_bucket}/app/${var.environment}/config.yaml"
    log_group      = aws_cloudwatch_log_group.app.name
  })
}
```

#### Subnet allocation with `cidrsubnets`

```hcl
locals {
  vpc_cidr = "10.100.0.0/16"

  # Carve /20 subnets (4096 addresses each) — 4 AZs × 3 tiers
  subnet_cidrs = cidrsubnets(local.vpc_cidr, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4)

  public_cidrs   = slice(local.subnet_cidrs, 0, 4)
  private_cidrs  = slice(local.subnet_cidrs, 4, 8)
  database_cidrs = slice(local.subnet_cidrs, 8, 12)
}
```

---

## Meta-arguments

Meta-arguments are reserved arguments accepted by every resource and module block. They modify Terraform's default behaviour rather than configuring the resource itself.

| Meta-argument | Scope | Primary use |
|---------------|-------|-------------|
| `count` | resource, module | Replicate a resource N times; addressed by integer index |
| `for_each` | resource, module | Replicate a resource once per map/set entry; addressed by key |
| `lifecycle` | resource | Override create/update/destroy behaviour |
| `depends_on` | resource, module | Inject explicit ordering when implicit dependency graph is insufficient |
| `provider` | resource, module | Route a resource to a non-default provider (multi-region, multi-account) |

> `count` and `for_each` are mutually exclusive within a single block. All others may coexist freely.

---

### count

Best suited for **homogeneous fleets** where instances are interchangeable and you don't need stable identity across replacements.

```hcl
# meta-arguments/count/main.tf
resource "aws_instance" "worker" {
  count = var.worker_count

  ami           = data.aws_ami.amazon_linux2.id
  instance_type = var.instance_type
  subnet_id     = var.private_subnet_ids[count.index % length(var.private_subnet_ids)]

  tags = {
    Name = "worker-${count.index + 1}-${var.environment}"
    Role = "worker"
  }
}
```

```hcl
# meta-arguments/count/outputs.tf
output "worker_ids" {
  description = "Instance IDs for all worker nodes"
  value       = aws_instance.worker[*].id
}

output "worker_private_ips" {
  description = "Private IPs — pass to load balancer target group"
  value       = aws_instance.worker[*].private_ip
}
```

> **Operational caveat:** If you reduce `count` from 5 to 3, Terraform destroys `worker[3]` and `worker[4]`. If the surviving instances were shifted in index (e.g., you removed index 1), Terraform will destroy and re-create everything above that index. For stateful workloads, prefer `for_each` with meaningful keys.

---

### for_each

Best suited for **heterogeneous resources** with meaningful identity — IAM roles, DNS records, security groups, S3 buckets, etc. Keys survive reordering; only the affected resource changes on update.

```hcl
# meta-arguments/for_each/main.tf
variable "service_accounts" {
  type = map(object({
    policy_arns = list(string)
    tags        = map(string)
  }))

  default = {
    "ci-deploy" = {
      policy_arns = ["arn:aws:iam::aws:policy/PowerUserAccess"]
      tags        = { team = "platform", env = "ci" }
    }
    "lambda-exec" = {
      policy_arns = [
        "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole",
        "arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess"
      ]
      tags = { team = "backend", env = "prod" }
    }
  }
}

resource "aws_iam_user" "service" {
  for_each = var.service_accounts

  name = each.key
  tags = merge(each.value.tags, { managed_by = "terraform" })
}

resource "aws_iam_user_policy_attachment" "service" {
  for_each = {
    for pair in flatten([
      for user, cfg in var.service_accounts : [
        for arn in cfg.policy_arns : { user = user, arn = arn }
      ]
    ]) : "${pair.user}/${pair.arn}" => pair
  }

  user       = each.value.user
  policy_arn = each.value.arn
}
```

---

### lifecycle

Controls what Terraform does at create, update, and destroy phases for a given resource.

```hcl
# meta-arguments/lifecycle/main.tf
resource "aws_rds_cluster" "primary" {
  cluster_identifier = "aurora-prod-${var.environment}"
  engine             = "aurora-postgresql"
  engine_version     = "15.3"
  database_name      = var.db_name
  master_username    = var.db_username
  master_password    = var.db_password

  lifecycle {
    # Block accidental `terraform destroy` on the production database
    prevent_destroy = true

    # Allow manual rotation of the master password without Terraform
    # replacing the cluster (Aurora requires a replace on password change)
    ignore_changes = [master_password]

    # Provision the replacement cluster before decommissioning the old one
    # Required for zero-downtime cluster replacement
    create_before_destroy = true
  }
}
```

| Option | Type | Behaviour |
|--------|------|-----------|
| `prevent_destroy` | `bool` | Hard error if a plan would destroy this resource |
| `ignore_changes` | `list(attr)` | Drift in listed attributes does not trigger update |
| `create_before_destroy` | `bool` | New resource is live before old one is removed |
| `replace_triggered_by` | `list(ref)` | Force replacement when referenced resource or attribute changes |

> To temporarily override `prevent_destroy` for a legitimate decommission: comment out the `lifecycle` block, run `terraform destroy -target`, then restore it. Do not remove `prevent_destroy` permanently in the same commit that performs the destroy — that makes the guard invisible in the git diff.

---

### depends_on

Terraform builds an implicit dependency graph from resource references. `depends_on` is needed only when a dependency exists that Terraform cannot infer — typically through side effects, not through direct attribute references.

```hcl
# meta-arguments/depends_on/main.tf

# IAM policy must exist before the instance profile can be attached,
# but Terraform can't infer this because we're passing the ARN as a
# data source lookup, not a direct reference.
resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.ec2.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "ec2" {
  name = "ec2-ssm-profile-${var.environment}"
  role = aws_iam_role.ec2.name

  depends_on = [aws_iam_role_policy_attachment.ssm]
}

resource "aws_instance" "app" {
  ami                  = data.aws_ami.amazon_linux2.id
  instance_type        = "m5.large"
  iam_instance_profile = aws_iam_instance_profile.ec2.name

  # The instance must not launch until the profile is fully provisioned;
  # IAM propagation lag means the instance would start without SSM access
  # if we relied solely on the implicit dependency chain.
  depends_on = [aws_iam_instance_profile.ec2]
}
```

> Use `depends_on` sparingly. Overuse signals that your module's resource graph is unclear and makes plans slower. If you find yourself using it frequently, consider restructuring outputs so that resources reference each other directly.

---

### provider (alias)

Route specific resources to a non-default region or account by declaring an aliased provider and referencing it via the `provider` meta-argument.

```hcl
# Multi-region active-passive setup
provider "aws" {
  region = "eu-west-1"  # primary
}

provider "aws" {
  alias  = "dr"
  region = "eu-central-1"  # disaster recovery
}

resource "aws_s3_bucket" "primary_artifacts" {
  bucket = "${var.project}-artifacts-primary"
}

resource "aws_s3_bucket" "dr_artifacts" {
  provider = aws.dr
  bucket   = "${var.project}-artifacts-dr"
}

# Replicate from primary to DR
resource "aws_s3_bucket_replication_configuration" "primary_to_dr" {
  bucket = aws_s3_bucket.primary_artifacts.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.dr_artifacts.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

---

## Key Takeaways / Engineering Notes

### Expressions

- **Prefer `for` over repeated `lookup` calls.** A `for` expression that transforms a map at plan time is easier to review and more resilient to schema drift than repeated imperative lookups.
- **`null` is not a string.** Assigning `null` to an optional argument tells Terraform to omit it entirely; assigning `""` passes an empty string to the provider, which may be invalid or trigger unintended behaviour.
- **Splat only works on `count`-managed resources.** For `for_each` resources, use `values(aws_instance.app)[*].id` or a `for` expression.

### Functions

- **`templatefile` over `file` + string interpolation.** Complex cloud-init or policy documents become unmaintainable when assembled via string interpolation in HCL. Externalise them as `.tftpl` files and pass variables explicitly.
- **`timestamp()` is evaluated at plan time,** meaning every `terraform plan` generates a new value. This triggers perpetual drift for tags. If you want a stable creation timestamp, write it once via `null_resource` + `triggers`, or use `formatdate` only for display — not as a resource tag that drives replacement.
- **Cryptographic functions are for checksums, not secrets.** `sha256()` and friends are useful for computing S3 object ETags or Lambda source hashes. Never use them to obscure sensitive values — use `sensitive()` and a secrets manager instead.
- **You cannot define custom functions in HCL.** If your expressions are growing unwieldy, the correct refactor is to push logic into locals or a module, not to wish for user-defined functions.

### Meta-arguments

- **`count` vs `for_each` decision rule:** If the instances are interchangeable and order is stable, use `count`. If they have distinct identity (name, config, destination) or if the set size changes in ways that would shift indices, use `for_each`.
- **`prevent_destroy` is a guardrail, not a lock.** It only works within a `terraform destroy` or plan that includes a destroy. Someone can still delete the resource from the AWS console, or remove the `lifecycle` block and re-apply. Pair it with AWS resource-level deletion protection where available (e.g., RDS deletion protection, S3 Object Lock).
- **`depends_on` on a module is coarse.** It serialises the entire module, not individual resources inside it. If you only need to order one resource, target it directly to avoid unnecessarily slowing down your apply.
- **The implicit dependency graph is almost always sufficient.** Terraform's reference tracking is comprehensive. If you feel you need `depends_on`, first check whether you can express the dependency as a direct attribute reference — that is cleaner and shows intent in the plan output.

---

## Prerequisites

- Terraform `>= 1.5.0`
- AWS credentials configured (`aws configure`, environment variables, or instance profile)
- An S3 backend or local state (local state is used in these examples for portability)

```bash
# Validate any module in this repo
cd meta-arguments/for_each
terraform init
terraform validate
terraform plan -var-file="../../env/dev.tfvars"
```

```bash
# Interactive function testing — no infrastructure created
cd functions
terraform init
terraform console
```

**Official references:**
- [Terraform Expressions](https://developer.hashicorp.com/terraform/language/expressions)
- [Terraform Functions](https://developer.hashicorp.com/terraform/language/functions)
- [Meta-arguments](https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on)

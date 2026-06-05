# terraform-patterns

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.4.6-7B42BC?logo=terraform&logoColor=white)
![AWS Provider](https://img.shields.io/badge/AWS_Provider-~%3E5.0-FF9900?logo=amazon-aws&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue)
![Status](https://img.shields.io/badge/Status-Active-brightgreen)

Personal reference repository for production Terraform patterns: local values, dynamic blocks, data sources, version constraints, workspace management, and credential hygiene. Configurations target AWS but the patterns are provider-agnostic.

---

## Table of Contents

- [Repository Layout](#repository-layout)
- [Local Values](#local-values)
- [Dynamic Blocks](#dynamic-blocks)
- [Data Sources](#data-sources)
- [Version Constraints](#version-constraints)
- [Credential Management](#credential-management)
- [Importing Existing Infrastructure](#importing-existing-infrastructure)
- [Workspace Management](#workspace-management)
- [Environment Variables](#environment-variables)
- [Engineering Notes](#engineering-notes)
- [Prerequisites](#prerequisites)

---

## Repository Layout

```
terraform-patterns/
├── locals/               # Local value patterns (IAM, tagging, port maps)
├── dynamic-blocks/       # Dynamic ingress/egress rule generation
├── data-sources/         # AMI lookup, remote state, external scripts
├── import/               # Importing unmanaged infrastructure
├── workspaces/           # Multi-workspace side-by-side environment testing
└── README.md
```

Each directory is a self-contained root module with its own `terraform.tf` (provider/version constraints) and `main.tf`.

---

## Local Values

**Directory:** `locals/`

Locals centralise expressions that are referenced multiple times without exposing them as overridable inputs. The key distinction from variables: locals are **not** overridden at runtime — they encode stable, shared facts about the infrastructure (account lists, tag sets, CIDR ranges, port maps).

### IAM User Provisioning

```hcl
# locals/main.tf

locals {
  # Permanent service accounts — changed via code review, not at plan time
  iam_accounts = toset([
    "svc-ci-deployer",
    "svc-metrics-exporter",
    "svc-backup-agent",
    "svc-audit-reader",
  ])

  common_tags = {
    managed_by  = "terraform"
    team        = "platform"
    cost_centre = "infra-prod"
    created_at  = timestamp()
  }
}

resource "aws_iam_user" "service_accounts" {
  for_each = local.iam_accounts

  name = each.key
  tags = local.common_tags
}
```

### Usage Pattern

| Scenario | Use `local` | Use `variable` |
|---|---|---|
| Tag set applied to all resources in a module | ✅ | — |
| CIDR block list shared across multiple SGs | ✅ | — |
| Environment name supplied by CI pipeline | — | ✅ |
| Instance type configurable per workspace | — | ✅ |
| Port list that never changes per deployment | ✅ | — |

> **Syntax reminder:** the block declaration is `locals {}` (plural); references inside expressions are `local.<name>` (singular).

---

## Dynamic Blocks

**Directory:** `dynamic-blocks/`

Dynamic blocks eliminate repetitive nested block declarations (ingress rules, lifecycle policies, logging configs, replica regions). The `for_each` inside a `dynamic` block iterates over a collection — typically sourced from locals or variables.

### Security Group with Parameterised Rules

Static approach for reference — **do not replicate this pattern:**

```hcl
# ❌ Static — duplicates structure, hard to diff in PRs
resource "aws_security_group" "app" {
  name = "app-sg"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}
```

Dynamic block approach — **preferred:**

```hcl
# dynamic-blocks/main.tf

locals {
  inbound_rules = {
    https_public = { port = 443,  cidr = "0.0.0.0/0"   }
    http_public  = { port = 80,   cidr = "0.0.0.0/0"   }
    app_internal = { port = 8080, cidr = "10.0.0.0/8"  }
    grpc_mesh    = { port = 9090, cidr = "172.16.0.0/12" }
  }

  outbound_rules = {
    rds_postgres  = { port = 5432,  cidr = "10.10.0.0/16" }
    elasticache   = { port = 6379,  cidr = "10.10.0.0/16" }
    https_egress  = { port = 443,   cidr = "0.0.0.0/0"    }
  }
}

resource "aws_security_group" "app_server" {
  name        = "app-server-sg"
  description = "Application tier — managed by Terraform"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = local.inbound_rules
    content {
      description = ingress.key
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }

  dynamic "egress" {
    for_each = local.outbound_rules
    content {
      description = egress.key
      from_port   = egress.value.port
      to_port     = egress.value.port
      protocol    = "tcp"
      cidr_blocks = [egress.value.cidr]
    }
  }

  tags = local.common_tags
}
```

<details>
<summary>Dynamic blocks are also valid inside <code>data</code> and <code>provider</code> blocks</summary>

```hcl
# Parameterise assume_role_policy conditions dynamically
data "aws_iam_policy_document" "boundary" {
  dynamic "statement" {
    for_each = var.allowed_actions
    content {
      effect    = "Allow"
      actions   = [statement.value]
      resources = ["*"]
    }
  }
}
```

</details>

---

## Data Sources

**Directory:** `data-sources/`

Data sources query provider APIs (or external systems) at plan time, injecting live values into the configuration without hard-coding them.

### AMI Lookup — Latest Hardened Image

```hcl
# data-sources/main.tf

data "aws_ami" "ubuntu_22_04_cis" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "state"
    values = ["available"]
  }

  lifecycle {
    postcondition {
      condition     = self.tags != null
      error_message = "AMI must carry tags — untagged images are not approved for use."
    }
  }
}

resource "aws_instance" "bastion" {
  ami           = data.aws_ami.ubuntu_22_04_cis.id
  instance_type = "t3.micro"

  tags = {
    Name        = "bastion-host"
    ami_sourced = data.aws_ami.ubuntu_22_04_cis.name
  }
}
```

### Remote State Reference

```hcl
# Read VPC outputs from a separately managed networking stack
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "acme-terraform-state-prod"
    key    = "networking/vpc/terraform.tfstate"
    region = "eu-west-1"
  }
}

locals {
  vpc_id             = data.terraform_remote_state.networking.outputs.vpc_id
  private_subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
}
```

### Common Data Source Use-Cases

| Data Source | Typical Use |
|---|---|
| `aws_ami` | Pin to latest approved base image |
| `aws_caller_identity` | Inject account ID into IAM ARN policies |
| `aws_availability_zones` | Spread subnets across all AZs automatically |
| `terraform_remote_state` | Consume outputs from another Terraform root |
| `external` | Pull dynamic values from scripts (e.g., Vault token, K8s cluster CA) |
| `aws_secretsmanager_secret_version` | Read credentials at plan time without storing them in state |

---

## Version Constraints

**Directory:** `locals/terraform.tf` (shared across modules)

```hcl
terraform {
  required_version = ">= 1.4.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # Accepts 5.x; blocks 6.x until explicitly bumped
    }
  }
}
```

### Constraint Operator Reference

| Operator | Meaning | Example | When to use |
|---|---|---|---|
| `= 1.4.6` | Exact version only | `"= 1.4.6"` | Reproducing a known-good state exactly |
| `!= 1.5.2` | Exclude a broken release | `"!= 1.5.2"` | Known regression in a specific release |
| `>= 1.4.6` | Lower bound only | `">= 1.4.6"` | Terraform core version — forward-compatible |
| `>= 1.4, < 2.0` | Explicit range | `">= 1.4, < 2.0"` | Prevent major version jump without review |
| `~> 5.0` | Pessimistic constraint | `"~> 5.0"` | Provider plugins — auto-patch but block major |
| `~> 5.4.0` | Patch-only | `"~> 5.4.0"` | High-stability environments needing patch control |

> **Recommendation:** Use `~> X.Y` (minor-level pessimistic) for provider plugins in most environments. Use `~> X.Y.Z` (patch-level) only in air-gapped or compliance-constrained environments where even minor provider bumps require change approval.

Upgrade provider to the latest permitted version:

```bash
terraform init -upgrade
```

---

## Credential Management

**Directory:** N/A — operational practice, not a Terraform module.

### Approach Comparison

| Method | Team-safe | Secret in VCS risk | Recommended for |
|---|---|---|---|
| Hard-coded in provider block | ❌ | 🔴 High | **Never** |
| `~/.aws/credentials` via `aws configure` | ❌ Single user | 🟢 None | Local dev only |
| `TF_VAR_*` environment exports | ⚠️ Session-scoped | 🟢 None | CI/CD runners (ephemeral) |
| `terraform.tfvars` | ⚠️ Only with `.gitignore` | 🔴 High if committed | Solo projects, with caution |
| Terraform Cloud workspace env vars (sensitive) | ✅ | 🟢 None | Teams using Terraform Cloud |
| HashiCorp Vault + `vault` provider | ✅ | 🟢 None | **Production — recommended** |

### Environment Variable Pattern (CI/CD)

```bash
# Set in CI environment (GitHub Actions secret, GitLab CI variable, etc.)
export TF_VAR_aws_access_key="AKIA..."
export TF_VAR_aws_secret_key="..."
export TF_VAR_aws_session_token="..."   # if using assumed roles
```

```hcl
# provider.tf — no credentials in source
variable "aws_access_key"    { sensitive = true }
variable "aws_secret_key"    { sensitive = true }
variable "aws_session_token" { sensitive = true; default = "" }

provider "aws" {
  region     = var.aws_region
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
  token      = var.aws_session_token != "" ? var.aws_session_token : null
}
```

### Vault Integration Sketch

```hcl
provider "vault" {
  address = "https://vault.internal.acme.com"
  token   = var.vault_token   # Sourced from CI secret or IAM auth
}

data "vault_generic_secret" "aws_creds" {
  path = "aws/creds/terraform-deployer"
}

provider "aws" {
  region     = "eu-west-1"
  access_key = data.vault_generic_secret.aws_creds.data["access_key"]
  secret_key = data.vault_generic_secret.aws_creds.data["secret_key"]
  token      = data.vault_generic_secret.aws_creds.data["security_token"]
}
```

> Dynamic credentials via Vault's AWS secrets engine means tokens are short-lived, automatically rotated, and never stored in state.

---

## Importing Existing Infrastructure

**Directory:** `import/`

Use `terraform import` to bring unmanaged resources (hand-crafted in the console, built with CloudFormation, or owned by another team's runbook) under Terraform management without recreating them.

### Workflow

```bash
# 1. Write a skeleton resource block — values can be placeholders
# 2. Initialise the directory
terraform init

# 3. Import the remote resource into local state
terraform import aws_instance.legacy_api i-0a1b2c3d4e5f67890

# 4. Inspect the generated state to extract actual attribute values
terraform show -json | jq '.values.root_module.resources[] | select(.address=="aws_instance.legacy_api")'

# 5. Update main.tf to match the state exactly, then verify
terraform plan   # Target: "No changes"
```

### Skeleton Resource Template

```hcl
# import/main.tf — fill attributes from `terraform show` output after import
resource "aws_instance" "legacy_api" {
  ami           = ""          # populate from state
  instance_type = ""          # populate from state

  tags = {
    Name = ""                 # populate from state
  }
}
```

> - **Goal state:** `terraform plan` shows **no changes** after populating attributes from the state file.
> - If plan shows `1 to destroy, 1 to add`, attributes in `main.tf` still diverge from the imported state.
> - `terraform import` only writes state — it does **not** generate `.tf` configuration. In Terraform ≥ 1.5, the `import` block with `generate` can scaffold the config automatically.

---

## Workspace Management

**Directory:** `workspaces/`

Workspaces provide isolated state files from a single configuration — useful for parallel environment testing without duplicating module code.

> **Scope:** Terraform CLI workspaces only. Terraform Cloud treats each workspace as a separate, isolated working directory — this pattern does not apply there.

### Quick Reference

```bash
terraform workspace list          # Show all workspaces; * = current
terraform workspace show          # Print current workspace name
terraform workspace new staging   # Create and switch to 'staging'
terraform workspace select prod   # Switch to existing 'prod' workspace
terraform workspace delete old    # Delete workspace (must not be current)
```

### Multi-Environment Pattern

```hcl
# workspaces/variables.tf — no defaults; values supplied per workspace run
variable "instance_type" { type = string }
variable "region"        { type = string }
variable "environment"   { type = string }

# workspaces/main.tf
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu_22_04_cis.id
  instance_type = var.instance_type

  tags = {
    Name        = "app-${var.environment}"
    environment = var.environment
  }
}
```

```bash
# Provision staging
terraform workspace new staging
terraform apply \
  -var="instance_type=t3.micro" \
  -var="region=eu-west-1" \
  -var="environment=staging"

# Provision prod in parallel, different spec
terraform workspace new prod
terraform apply \
  -var="instance_type=m5.large" \
  -var="region=eu-west-1" \
  -var="environment=prod"
```

### State File Layout

```
terraform.tfstate.d/
├── staging/
│   └── terraform.tfstate
└── prod/
    └── terraform.tfstate
```

> **Caveats:**
> - Workspaces share provider config and backend config. Use variable-driven regions/accounts rather than separate `provider` blocks.
> - Deleting a workspace (`terraform workspace delete`) also deletes its state file. Destroy resources first.
> - For truly isolated environments (separate AWS accounts, blast-radius separation), prefer separate root modules over workspaces.

---

## Environment Variables

Terraform respects several `TF_*` environment variables that influence runtime behaviour without modifying `.tf` files.

| Variable | Effect | Example |
|---|---|---|
| `TF_LOG` | Log verbosity | `export TF_LOG=DEBUG` |
| `TF_LOG_PATH` | Write logs to file | `export TF_LOG_PATH=/tmp/tf.log` |
| `TF_VAR_<name>` | Set variable value | `export TF_VAR_region=eu-west-1` |
| `TF_CLI_ARGS_apply` | Default flags for `apply` | `export TF_CLI_ARGS_apply="-auto-approve"` |
| `TF_DATA_DIR` | Override `.terraform/` location | `export TF_DATA_DIR=/tmp/tf-data` |
| `TF_PLUGIN_CACHE_DIR` | Shared provider cache across modules | `export TF_PLUGIN_CACHE_DIR=~/.terraform.d/plugin-cache` |
| `TF_WORKSPACE` | Select workspace non-interactively | `export TF_WORKSPACE=staging` |

### Shared Plugin Cache

Prevents re-downloading the same provider binary across multiple root modules — useful on developer machines and in CI to cut initialisation time.

```bash
mkdir -p ~/.terraform.d/plugin-cache
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"

# Or pass inline for a single init
terraform init -plugin-dir="$HOME/.terraform.d/plugin-cache"
```

Add to shell profile for persistence:

```bash
echo 'export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"' >> ~/.zshrc
```

---

## Engineering Notes

### On Locals vs Variables

- Variables are the **API surface** of a module — they are what callers control.
- Locals are **internal constants** — they compute or alias values for readability and DRY without exposing them as configurable inputs.
- If a value is derived from other values (e.g., `"${var.env}-${var.app_name}"`), it belongs in a local, not repeated inline.

### On Dynamic Blocks

- Dynamic blocks shine when the _structure_ of a nested block is repeated, not just a value. If you're copy-pasting `ingress {}` blocks, a dynamic block is the right fix.
- Keep `content {}` bodies thin. Push complex expressions into locals first, then reference them from `content`.
- Dynamic blocks work inside `resource`, `data`, and `provider` blocks but **not** inside the `terraform {}` block.

### On Data Sources

- Data sources add an implicit dependency: if the external resource is missing, `plan` fails. Document what resources must pre-exist.
- The `lifecycle { postcondition {} }` pattern on data sources catches bad data early (e.g., unapproved AMIs, empty outputs from remote state) rather than propagating them silently into resource attributes.
- Avoid using `data.terraform_remote_state` across team boundaries unless you own both stacks — prefer explicit variable passing or a service registry pattern.

### On Workspaces

- Workspaces are an operational tool, not an architectural one. They're appropriate for short-lived parallel tests; they're a poor substitute for proper environment isolation.
- Production workloads should use separate state backends (separate S3 buckets/prefixes), separate AWS accounts, and separate root modules — not workspaces.

### On Credential Hygiene

- `sensitive = true` on a variable prevents the value from appearing in CLI output but **does not encrypt it in state**. Use a remote backend with encryption at rest (S3 + KMS, Terraform Cloud) for any sensitive values that end up in state.
- Prefer OIDC/IAM role federation in CI (GitHub Actions OIDC → AWS IAM role) over static access keys entirely. No keys to rotate, no keys to leak.

---

## Prerequisites

- [ ] Terraform `>= 1.4.6` — [install guide](https://developer.hashicorp.com/terraform/install)
- [ ] AWS CLI configured (`aws configure` or environment variables)
- [ ] AWS IAM permissions: EC2, IAM, S3 (for relevant modules)
- [ ] `jq` for parsing `terraform show -json` output

```bash
terraform -version
aws sts get-caller-identity   # Verify credentials are active
```

---

> Configurations in this repository create billable AWS resources. Run `terraform destroy` in each module directory when done. Review `terraform plan` output before every `apply`.

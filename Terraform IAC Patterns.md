# terraform-iac-patterns

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.4.0-7B42BC?logo=terraform&logoColor=white)
![AWS Provider](https://img.shields.io/badge/AWS%20Provider-~%3E4.66-FF9900?logo=amazon-aws&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue)
![Status](https://img.shields.io/badge/status-active-brightgreen)

A reference implementation covering variable management patterns, reusable module architecture, and structured logging/observability for AWS infrastructure provisioning with Terraform. Intended as an internal knowledge base for teams standardizing IaC practices.

---

## Table of Contents

- [Repository Structure](#repository-structure)
- [Variable Management](#variable-management)
  - [Input Variable Declarations](#input-variable-declarations)
  - [Value Assignment Strategies](#value-assignment-strategies)
  - [Sensitive Values via Environment Variables](#sensitive-values-via-environment-variables)
  - [Precedence Reference](#precedence-reference)
- [Module Architecture](#module-architecture)
  - [Local Modules](#local-modules)
  - [Public Registry Modules](#public-registry-modules)
  - [Multi-User / Multi-Environment Consumption](#multi-user--multi-environment-consumption)
- [Observability & Logging](#observability--logging)
  - [Log Levels](#log-levels)
  - [Logging to File](#logging-to-file)
  - [Persistent Logging via Shell Profile](#persistent-logging-via-shell-profile)
  - [Scoped Logging by Subsystem](#scoped-logging-by-subsystem)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Shell Aliases for Workflow Efficiency](#shell-aliases-for-workflow-efficiency)
- [Engineering Notes](#engineering-notes)

---

## Repository Structure

```
terraform-iac-patterns/
├── modules/
│   └── ec2-webserver/          # Reusable compute module
│       ├── main.tf
│       └── variables.tf
├── environments/
│   └── prod/
│       ├── main.tf             # Root module — calls child modules
│       ├── variables.tf        # Variable declarations
│       ├── outputs.tf
│       └── terraform.tfvars    # Non-sensitive environment values
├── team-configs/
│   ├── platform-team/
│   │   └── main.tf             # Platform team's module consumption config
│   └── app-team/
│       └── main.tf             # App team's module consumption config
└── logs/
    └── terraform/              # Persistent log output (gitignored)
```

> `.gitignore` must exclude `*.tfvars` containing secrets, `terraform.tfstate*`, `.terraform/`, and `logs/`.

---

## Variable Management

### Input Variable Declarations

Variable blocks define the contract for a module — what inputs it accepts, their types, and optional defaults. Declarations live in `variables.tf` and are intentionally kept free of runtime values.

```hcl
# modules/ec2-webserver/variables.tf

variable "vpc_id" {
  type        = string
  description = "ID of the VPC in which the instance will be deployed."
}

variable "subnet_id" {
  type        = string
  description = "Target subnet ID. Must belong to the VPC above."
}

variable "ami_id" {
  type        = string
  description = "EC2 AMI ID. Must be region-specific and free-tier eligible for non-prod."
}

variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance type. Defaults to t3.micro; override for prod workloads."
}

variable "environment" {
  type        = string
  description = "Deployment environment tag: prod | staging | dev."

  validation {
    condition     = contains(["prod", "staging", "dev"], var.environment)
    error_message = "environment must be one of: prod, staging, dev."
  }
}

variable "service_name" {
  type        = string
  description = "Service identifier used in Name tags and hostname conventions."
}
```

Calling a variable inside a resource block uses the `var.<name>` syntax:

```hcl
# modules/ec2-webserver/main.tf

resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = {
    Name        = "${var.service_name}-${var.environment}"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

---

### Value Assignment Strategies

There are four common patterns for supplying values to declared variables. Which one to use depends on whether the value is static, environment-specific, or sensitive.

#### 1. `terraform.tfvars` — Non-sensitive environment values

The default values file. Automatically loaded by `terraform plan` / `apply` if present in the working directory.

```hcl
# environments/prod/terraform.tfvars

instance_type = "t3.medium"
environment   = "prod"
service_name  = "payments-api"
ami_id        = "ami-0c02fb55956c7d316"   # Amazon Linux 2023, us-east-1
```

#### 2. `*.auto.tfvars` — Per-environment overrides

Useful when a repository manages multiple environments from the same module. Terraform automatically loads all `*.auto.tfvars` files in the working directory.

```hcl
# environments/staging/staging.auto.tfvars

instance_type = "t3.micro"
environment   = "staging"
service_name  = "payments-api"
ami_id        = "ami-0c02fb55956c7d316"
```

#### 3. `-var` flag — Ad-hoc or CI/CD pipeline overrides

Highest precedence. Useful for one-off deployments or when injecting values from a CI pipeline without modifying files.

```bash
terraform apply \
  -var="service_name=payments-api" \
  -var="environment=prod" \
  -var="instance_type=t3.large"
```

#### 4. `-var-file` flag — Explicit file targeting

Useful when `tfvars` files live outside the working directory or when multiple value sets are maintained.

```bash
# Point to a file in a separate vars directory
terraform apply -var-file="../../config/prod.tfvars"

# Or reference environment-specific configs in CI
terraform plan -var-file="envs/${DEPLOY_ENV}.tfvars"
```

#### Interactive prompt (no default, no value supplied)

If a variable has no default and no value is provided through any mechanism, Terraform prompts at runtime. This is **not recommended for automated pipelines** — it will block execution.

```
var.service_name
  Service identifier used in Name tags and hostname conventions.

  Enter a value: payments-api
```

---

### Sensitive Values via Environment Variables

Hard-coding AWS credentials or API tokens into `.tf` or `.tfvars` files is a security anti-pattern. Environment variables prefixed with `TF_VAR_` are read by Terraform at runtime without ever touching disk.

**Variable declarations** (no defaults, no values):

```hcl
# variables.tf

variable "aws_access_key_id" {
  type      = string
  sensitive = true
}

variable "aws_secret_access_key" {
  type      = string
  sensitive = true
}
```

**Provider configuration** referencing those variables:

```hcl
# main.tf

provider "aws" {
  region     = var.aws_region
  access_key = var.aws_access_key_id
  secret_key = var.aws_secret_access_key
}
```

**Exporting values in CI or a restricted shell** (note the leading space to suppress shell history):

```bash
 export TF_VAR_aws_access_key_id="AKIA..."
 export TF_VAR_aws_secret_access_key="wJalr..."
```

> **Design decision**: In production pipelines, prefer IAM instance profiles, OIDC federation (GitHub Actions → AWS), or a secrets manager (HashiCorp Vault, AWS Secrets Manager) over exporting raw credentials. The `TF_VAR_*` pattern is a stepping stone, not an end state.

To clear variables from the current session without restarting:

```bash
export TF_VAR_aws_access_key_id=
export TF_VAR_aws_secret_access_key=
```

---

### Precedence Reference

When the same variable is supplied through multiple mechanisms, Terraform applies the following resolution order (highest to lowest):

| Priority | Mechanism | Example |
|---|---|---|
| **1 (highest)** | `-var` flag / `-var-file` flag | `terraform apply -var="env=prod"` |
| **2** | `*.auto.tfvars` / `*.auto.tfvars.json` | `prod.auto.tfvars` |
| **3** | `terraform.tfvars.json` | `terraform.tfvars.json` |
| **4** | `terraform.tfvars` | `terraform.tfvars` |
| **5 (lowest)** | `TF_VAR_*` environment variables | `export TF_VAR_env=prod` |

> `terraform.tfvars` **houses values only**. `variables.tf` handles declarations — symbolic names, types, descriptions, and optional defaults. Mixing both concerns into a single file creates maintenance debt in shared module repositories.

---

## Module Architecture

### Local Modules

Local modules encapsulate a reusable resource group. The key design principle: **consumers assign values, modules define structure**. No consumer should ever modify files inside the module directory.

**Module directory** (`modules/ec2-webserver/`):

```hcl
# modules/ec2-webserver/main.tf

terraform {
  required_version = ">= 1.4.0"
}

resource "aws_subnet" "this" {
  vpc_id            = var.vpc_id
  cidr_block        = var.cidr_block
  availability_zone = var.availability_zone

  tags = {
    Name      = "${var.service_name}-subnet-${var.environment}"
    ManagedBy = "terraform"
  }
}

resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.this.id

  tags = {
    Name        = "${var.service_name}-${var.environment}"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

```hcl
# modules/ec2-webserver/variables.tf

variable "vpc_id"            { type = string }
variable "cidr_block"        { type = string }
variable "availability_zone" { type = string }
variable "ami_id"            { type = string }
variable "instance_type"     { type = string; default = "t3.micro" }
variable "environment"       { type = string }
variable "service_name"      { type = string }
```

**Consumer configuration** (`team-configs/platform-team/main.tf`):

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.10.0.0/16"

  tags = { Name = "platform-vpc-prod" }
}

module "api_server" {
  source = "../../modules/ec2-webserver"

  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.10.1.0/24"
  availability_zone = "us-east-1a"
  ami_id            = "ami-0c02fb55956c7d316"
  instance_type     = "t3.medium"
  environment       = "prod"
  service_name      = "internal-api"
}
```

When referencing resources that live inside a module, Terraform uses the fully-qualified path:

```
module.<module_name>.<resource_type>.<resource_name>
# e.g.
module.api_server.aws_instance.this
```

---

### Public Registry Modules

The [Terraform Registry](https://registry.terraform.io) hosts production-grade community modules. For AWS, `terraform-aws-modules` is the de facto standard — the VPC module alone exceeds 40M downloads.

**Root module consuming VPC + EC2 Instance registry modules:**

```hcl
# environments/prod/main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = var.vpc_name
  cidr = var.vpc_cidr

  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  enable_nat_gateway   = true
  single_nat_gateway   = false   # HA: one NAT GW per AZ
  enable_dns_hostnames = true

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = var.project_name
  }
}

module "app_cluster" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "5.5.0"

  count = var.instance_count

  name          = "${var.service_name}-node-${count.index}"
  ami           = var.ami_id
  instance_type = var.instance_type

  subnet_id              = module.vpc.private_subnets[count.index % length(module.vpc.private_subnets)]
  vpc_security_group_ids = [module.vpc.default_security_group_id]

  tags = {
    Name        = "${var.service_name}-node-${count.index}"
    Environment = var.environment
    ManagedBy   = "terraform"
    Cluster     = var.service_name
  }
}
```

```hcl
# environments/prod/variables.tf

variable "aws_region"            { type = string; default = "us-east-1" }
variable "vpc_name"              { type = string }
variable "vpc_cidr"              { type = string; default = "10.0.0.0/16" }
variable "availability_zones"    { type = list(string) }
variable "private_subnet_cidrs"  { type = list(string) }
variable "public_subnet_cidrs"   { type = list(string) }
variable "environment"           { type = string }
variable "project_name"          { type = string }
variable "service_name"          { type = string }
variable "instance_count"        { type = number; default = 3 }
variable "ami_id"                { type = string }
variable "instance_type"         { type = string; default = "t3.micro" }
```

```hcl
# environments/prod/outputs.tf

output "app_cluster_private_ips" {
  description = "Private IP addresses for all cluster nodes — update CMDB / DNS records."
  value       = module.app_cluster[*].private_ip
}

output "vpc_id" {
  description = "VPC ID for cross-stack references."
  value       = module.vpc.vpc_id
}

output "private_subnet_ids" {
  description = "Private subnet IDs. Pass to EKS, RDS, or ALB modules as needed."
  value       = module.vpc.private_subnets
}
```

> The `[*]` syntax is a **splat expression** — shorthand for collecting a single attribute across all instances of a resource or module in a list. Equivalent to `[for inst in module.app_cluster : inst.private_ip]`.

**Downloading modules without full init:**

```bash
# Fetch/refresh remote modules only — does not re-initialize the backend or provider
terraform get -update
```

---

### Multi-User / Multi-Environment Consumption

The module pattern allows multiple teams to consume the same vetted module code with different input values. No consumer ever modifies the module source.

```
modules/ec2-webserver/        ← shared, owned by platform team
    main.tf                   ← never modified by consumers
    variables.tf

team-configs/
    platform-team/
        main.tf               ← platform team's values only
    app-team/
        main.tf               ← app team's values only; different region, AMI, instance size
```

Each consumer directory is an independent Terraform working directory:

```bash
# Platform team deploy
cd team-configs/platform-team
terraform init && terraform apply

# App team deploy (different region, different state)
cd team-configs/app-team
terraform init && terraform apply
```

Module initialization output confirms the local module is registered:

```
Initializing modules...
- api_server in ../../modules/ec2-webserver

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...
```

---

## Observability & Logging

### Log Levels

Terraform exposes structured logging via the `TF_LOG` environment variable. Levels map to decreasing verbosity:

| Level | Use Case |
|---|---|
| `TRACE` | Maximum detail — full provider gRPC calls, state diffs, plugin handshakes. Share with HashiCorp support or when debugging provider bugs. |
| `DEBUG` | Provider plugin negotiation, backend requests, module resolution. Useful for diagnosing init failures. |
| `INFO`  | Lifecycle events, version checks, CLI argument parsing. Low noise. |
| `WARN`  | Deprecation notices, soft constraint violations. |
| `ERROR` | Terminal failures only. Equivalent to no logging in most cases — errors surface in stdout regardless. |

```bash
# Trace a specific apply for a production change
TF_LOG=TRACE terraform apply -var-file="prod.tfvars" 2>&1 | tee /tmp/apply-$(date +%Y%m%d-%H%M%S).log

# Debug a provider authentication failure
TF_LOG=DEBUG terraform init

# Info-level for routine CI pipelines where logs are indexed
TF_LOG=INFO terraform plan -out=tfplan
```

---

### Logging to File

When `TF_LOG_PATH` is set, log output is redirected from stdout to a file. The log is **appended** on each command, creating a running audit trail of all Terraform operations in the session.

```bash
# Set log level and output path for the session
export TF_LOG=TRACE
export TF_LOG_PATH="${HOME}/logs/terraform/tf-$(date +%Y%m%d).log"

# All subsequent terraform commands write to this file
terraform init
terraform plan  -out=tfplan
terraform apply tfplan
```

<details>
<summary>Example log structure (terraform apply, truncated)</summary>

```
2024-01-15T09:12:34.123Z [INFO]  Terraform version: 1.6.4
2024-01-15T09:12:34.124Z [INFO]  Go runtime version: go1.21.5
2024-01-15T09:12:34.125Z [INFO]  CLI args: []string{"terraform", "apply", "tfplan"}
2024-01-15T09:12:34.201Z [DEBUG] provider: starting provider: provider="registry.terraform.io/hashicorp/aws"
2024-01-15T09:12:34.890Z [TRACE] provider.terraform-provider-aws: configuring provider with access credentials
2024-01-15T09:12:35.100Z [INFO]  backend/local: starting Apply operation
2024-01-15T09:12:35.450Z [TRACE] statemgr.Filesystem: locking terraform.tfstate
...
2024-01-15T09:13:10.881Z [INFO]  backend/local: Apply complete, 3 resources added
```

</details>

> The log file must be excluded from version control. Add `logs/` and `*.log` to `.gitignore`.

---

### Persistent Logging via Shell Profile

For environments where continuous audit logging is a compliance requirement, export logging variables in `~/.bashrc` or `~/.bash_profile`. This ensures all Terraform invocations are captured regardless of which project directory is active.

```bash
# ~/.bashrc — append below existing content

# Terraform persistent audit logging
export TF_LOG=DEBUG
export TF_LOG_PATH="${HOME}/logs/terraform/tf-$(date +%Y%m%d).log"
```

Ensure the log directory exists:

```bash
mkdir -p "${HOME}/logs/terraform"
```

Apply the change without restarting the shell:

```bash
source ~/.bashrc
```

To disable logging for the current session without editing the profile:

```bash
export TF_LOG=
export TF_LOG_PATH=
```

---

### Scoped Logging by Subsystem

For large deployments with many providers, full `TF_LOG` output can be overwhelming. Terraform supports subsystem-scoped logging to isolate noise:

```bash
# Log only Terraform core behavior (state management, graph evaluation)
export TF_LOG_CORE=TRACE

# Log only provider plugin communication (gRPC calls to AWS provider)
export TF_LOG_PROVIDER=DEBUG
```

Subsystem scoping is particularly useful when troubleshooting third-party or unverified providers (e.g., Libvirt for KVM, Proxmox) where provider-level bugs are more common than core issues.

---

## Troubleshooting Reference

Common errors and their root causes:

| Error | Root Cause | Resolution |
|---|---|---|
| `Error: inconsistent dependency lock file` | `terraform init` has not been run, or `.terraform.lock.hcl` is stale | Run `terraform init -upgrade` |
| `Error: Failed to query available provider packages` | Version constraint references a non-existent provider version | Correct the version string in `required_providers`; use `~>` for minor-range pinning |
| `Error: Invalid reference` | String value passed without interpolation syntax, or bare variable name used outside quotes | Wrap string literals in `""`, use `"${var.name}"` for interpolation inside strings |
| `Error: Missing required argument` | A variable with no default was not supplied a value | Supply via `-var`, `tfvars`, or `TF_VAR_*` |
| `Error: Provider configuration not present` | Module calls a provider not configured in the root module | Add the required `provider` block to the root module |

**Validating configuration before apply:**

```bash
terraform fmt -recursive       # Normalize formatting across all .tf files
terraform validate             # Syntax and type-checking without cloud API calls
terraform plan -out=tfplan     # Dry-run with explicit plan artifact for review
terraform show tfplan          # Human-readable plan review before apply
terraform apply tfplan         # Apply the reviewed, saved plan
```

**Community resources for unresolved issues:**

- [HashiCorp Discuss — Terraform](https://discuss.hashicorp.com/c/terraform) — post with `TF_LOG=TRACE` output attached
- [hashicorp/terraform GitHub Issues](https://github.com/hashicorp/terraform/issues) — for suspected core bugs; include Terraform version, provider version, OS, and minimal reproduction config
- [Stack Overflow — terraform tag](https://stackoverflow.com/questions/tagged/terraform)

---

## Shell Aliases for Workflow Efficiency

Add to `~/.bash_aliases` (sourced from `~/.bashrc` via the standard `if` block):

```bash
# ~/.bash_aliases

alias ti='terraform init'
alias tv='terraform validate'
alias tfmt='terraform fmt -recursive'
alias tp='terraform plan -out=tfplan'
alias ta='terraform apply tfplan'
alias td='terraform destroy'
alias tshow='terraform show'
alias tstate='terraform state list'
alias toutput='terraform output'
```

> Avoid aliasing `ta` to `terraform apply -auto-approve`. The confirmation prompt exists as a forcing function — removing it in shared or production environments increases the blast radius of misapplied changes. Reserve `-auto-approve` for CI pipelines where the plan has already been reviewed and approved as a separate pipeline stage.

Apply without relogging:

```bash
source ~/.bash_aliases
```

---

## Engineering Notes

**On variable file separation:**
`terraform.tfvars` is for values. `variables.tf` is for declarations. Blurring this boundary — e.g., setting defaults directly in `.tfvars` or cramming values into `variables.tf` — creates ambiguity about where the source of truth lives in a shared module. Keep them separate.

**On module ownership:**
Modules should be treated like published APIs. Once consumed by more than one team, changes to a module's variable interface are breaking changes. Use semantic versioning if storing modules in a Git registry, and pin consumers to a specific `ref` or `version`.

**On credentials in Terraform:**
The `TF_VAR_*` pattern for credentials is an improvement over hard-coding, but still puts secrets in process environment memory. For production workloads:
- Use IAM roles with EC2 instance profiles or ECS task roles
- Use OIDC federation for CI/CD (GitHub Actions, GitLab CI)
- Use HashiCorp Vault's AWS secrets engine for short-lived dynamic credentials

**On the `terraform refresh` command:**
`terraform refresh` is deprecated. Use `terraform apply -refresh-only` instead. It performs the same state reconciliation but runs through the full plan/confirm workflow, giving you visibility into what the refresh would change before committing it.

**On log verbosity in CI:**
`TF_LOG=INFO` is a reasonable default for CI pipelines feeding into a log aggregator (Datadog, Splunk, CloudWatch Logs). Reserve `TRACE` for local troubleshooting sessions — trace logs for a 26-resource apply easily exceed 5,000 lines and will bloat your log storage costs at scale.

**On count vs. for_each:**
`count` with `count.index` for naming (as shown in the IAM user example) works but produces fragile state — deleting index 1 of 3 causes Terraform to renumber index 2 to index 1, triggering an unnecessary destroy/recreate. Prefer `for_each` with a set of strings for production resources where element identity matters.

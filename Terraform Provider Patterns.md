# terraform-provider-patterns

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.5-7B42BC?logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-Provider-FF9900?logo=amazonaws&logoColor=white)
![VMware](https://img.shields.io/badge/VMware-vSphere-607078?logo=vmware&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-blue)

Reference implementations and engineering notes covering Terraform provider configuration patterns: multi-region AWS deployments, VMware vSphere automation, Docker/Kubernetes resource management, remote state backends, provider lock files, and credential strategies.

---

## Table of Contents

- [Repository Structure](#repository-structure)
- [Provider Overview](#provider-overview)
- [AWS — Multi-Region with Aliases](#aws--multi-region-with-aliases)
- [VMware vSphere](#vmware-vsphere)
- [Docker & Container Providers](#docker--container-providers)
- [Kubernetes (EKS)](#kubernetes-eks)
- [Remote State Backend — S3](#remote-state-backend--s3)
- [Provider Lock File Management](#provider-lock-file-management)
- [AWS Credential Strategies](#aws-credential-strategies)
- [Engineering Notes](#engineering-notes)

---

## Repository Structure

```
.
├── aws-multi-region/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── aws-remote-backend-s3/
│   ├── main.tf
│   └── backend.tf
├── vmware-vsphere/
│   └── main.tf
├── docker/
│   └── main.tf
├── kubernetes-eks/
│   └── main.tf
└── libvirt-kvm/
    ├── provider.tf
    └── vm.tf
```

---

## Provider Overview

| Provider | Source | Tier | Use Case |
|---|---|---|---|
| `aws` | `hashicorp/aws` | Official | EC2, IAM, S3, VPC, EKS |
| `vsphere` | `hashicorp/vsphere` | Official | VM lifecycle on vCenter-managed ESXi |
| `docker` | `kreuzwerker/docker` | Verified | Local/remote Docker image & container management |
| `kubernetes` | `hashicorp/kubernetes` | Official | K8s deployments, services, configmaps |
| `libvirt` | `dmacvicar/libvirt` | Community | KVM/QEMU VM provisioning via libvirt API |

> **Note:** Community providers (e.g. `dmacvicar/libvirt`) are not HashiCorp-verified. Pin versions strictly and validate checksums before use in production pipelines.

---

## AWS — Multi-Region with Aliases

When infrastructure spans multiple AWS regions within a single Terraform project, provider aliases decouple the default region from secondary targets. A common pattern: primary workload in `us-east-1`, DR or edge nodes in `eu-west-1` or `ap-southeast-1`.

### Configuration

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"
    }
  }
}

# Primary region — used by all resources that omit the `provider` argument
provider "aws" {
  region = "us-east-1"
}

# Secondary region — must be referenced explicitly via alias
provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}
```

### Usage in Resources

```hcl
# Deploys into us-east-1 (default provider)
resource "aws_instance" "app_primary" {
  ami           = "ami-0c02fb55956c7d316"  # Amazon Linux 2023, us-east-1
  instance_type = "t3.medium"

  tags = {
    Name        = "app-primary"
    Environment = "production"
    Region      = "us-east-1"
  }
}

# Deploys into eu-west-1 using the aliased provider
resource "aws_instance" "app_dr" {
  provider = aws.eu_west

  ami           = "ami-0d71ea30463e0ff49"  # Amazon Linux 2023, eu-west-1
  instance_type = "t3.medium"

  tags = {
    Name        = "app-dr"
    Environment = "production"
    Region      = "eu-west-1"
  }
}
```

> **Design decision:** AMIs are region-scoped. Always verify the correct AMI ID for each target region — a mismatch here causes a plan-time error, not a runtime failure, which is preferable but still needs CI guard rails.

---

## VMware vSphere

The vSphere provider targets vCenter, not individual ESXi hosts directly. Terraform issues API calls to vCenter, which distributes workload to the appropriate ESXi nodes in the cluster. Free ESXi (no vCenter) is not supported.

### Provider Configuration

```hcl
terraform {
  required_providers {
    vsphere = {
      source  = "hashicorp/vsphere"
      version = "~> 2.6"
    }
  }
}

provider "vsphere" {
  user                 = var.vsphere_user
  password             = var.vsphere_password
  vsphere_server       = var.vsphere_server   # vCenter FQDN or IP
  allow_unverified_ssl = false
}
```

### Data Sources — Reading Existing Infrastructure

```hcl
data "vsphere_datacenter" "dc" {
  name = "dc-prod-01"
}

data "vsphere_datastore" "nvme_store" {
  name          = "nvme-ds-01"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_compute_cluster" "cluster" {
  name          = "cluster-prod-01"
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_network" "vlan_app" {
  name          = "VLAN-100-App"
  datacenter_id = data.vsphere_datacenter.dc.id
}
```

> Data sources model pre-existing vSphere objects. Resources like `vsphere_datastore` and `vsphere_network` can also be *created* via Terraform resource blocks — but `data` blocks are read-only lookups that enable resources to reference existing infrastructure by name.

### Virtual Machine Resource

```hcl
resource "vsphere_virtual_machine" "web_node" {
  name             = "web-node-01"
  resource_pool_id = data.vsphere_compute_cluster.cluster.resource_pool_id
  datastore_id     = data.vsphere_datastore.nvme_store.id
  guest_id         = "debian11_64Guest"

  num_cpus = 4
  memory   = 8192

  network_interface {
    network_id = data.vsphere_network.vlan_app.id
  }

  disk {
    label            = "disk0"
    size             = 80
    thin_provisioned = true
  }
}
```

<details>
<summary>Variable definitions for vSphere credentials</summary>

```hcl
variable "vsphere_user" {
  description = "vCenter service account username"
  type        = string
  sensitive   = true
}

variable "vsphere_password" {
  description = "vCenter service account password"
  type        = string
  sensitive   = true
}

variable "vsphere_server" {
  description = "vCenter FQDN or IP address"
  type        = string
}
```

Pass credentials via environment variables to avoid storing secrets in `.tfvars`:

```bash
export TF_VAR_vsphere_user="svc-terraform@vsphere.local"
export TF_VAR_vsphere_password="$(vault kv get -field=password secret/vsphere/svc-terraform)"
export TF_VAR_vsphere_server="vcenter.internal.example.com"
```

</details>

---

## Docker & Container Providers

The `kreuzwerker/docker` provider manages the full Docker resource lifecycle — images, containers, volumes, and networks — using the Docker daemon socket.

### Provider Configuration

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = ">= 3.0.1, < 4.0.0"
    }
  }
}

provider "docker" {
  # Linux default — often no explicit host needed
  # host = "unix:///var/run/docker.sock"

  # For remote daemon over TLS:
  # host     = "tcp://docker-host.internal:2376"
  # cert_path = "/etc/docker/certs"
}
```

### Image + Container

```hcl
resource "docker_image" "envoy_proxy" {
  name         = "envoyproxy/envoy:v1.29-latest"
  keep_locally = true
}

resource "docker_container" "envoy" {
  name  = "envoy-edge-proxy"
  image = docker_image.envoy_proxy.image_id

  ports {
    internal = 10000
    external = 10000
    protocol = "tcp"
  }

  ports {
    internal = 9901
    external = 9901
    protocol = "tcp"  # Admin interface
  }

  volumes {
    host_path      = abspath("${path.module}/envoy.yaml")
    container_path = "/etc/envoy/envoy.yaml"
    read_only      = true
  }

  restart = "unless-stopped"
}
```

```bash
# Verify after apply
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
terraform show
```

---

## Kubernetes (EKS)

The Kubernetes provider is commonly paired with the AWS provider to target an EKS cluster. Cluster endpoint and CA certificate are consumed from the EKS data source at plan time.

```hcl
data "aws_eks_cluster" "main" {
  name = var.cluster_name
}

data "aws_eks_cluster_auth" "main" {
  name = var.cluster_name
}

provider "kubernetes" {
  host                   = data.aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", var.cluster_name]
  }
}
```

```hcl
resource "kubernetes_deployment" "api_gateway" {
  metadata {
    name      = "api-gateway"
    namespace = "platform"
    labels = {
      app     = "api-gateway"
      version = "v2"
    }
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "api-gateway"
      }
    }

    template {
      metadata {
        labels = {
          app = "api-gateway"
        }
      }

      spec {
        container {
          name  = "api-gateway"
          image = "ghcr.io/example-org/api-gateway:v2.4.1"

          port {
            container_port = 8080
          }

          resources {
            requests = {
              cpu    = "250m"
              memory = "256Mi"
            }
            limits = {
              cpu    = "500m"
              memory = "512Mi"
            }
          }
        }
      }
    }
  }
}
```

---

## Remote State Backend — S3

Storing state remotely enables team collaboration, state locking, and versioned history. The S3 backend combined with DynamoDB for state locking is the standard AWS pattern.

### Backend Configuration

```hcl
terraform {
  backend "s3" {
    bucket         = "example-org-tfstate-prod"
    key            = "platform/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Creating the S3 Bucket and DynamoDB Lock Table (one-time bootstrap)

```hcl
resource "aws_s3_bucket" "tfstate" {
  bucket = "example-org-tfstate-prod"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "tf_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

> **Caveats:**
> - The bootstrap resources above must be applied *before* migrating the backend. Run with a local backend first, then add the `backend "s3"` block and run `terraform init` to migrate.
> - `prevent_destroy = true` on the state bucket guards against accidental `terraform destroy` of the bucket itself.
> - Without the DynamoDB lock table, concurrent applies from separate CI runners can corrupt state.

---

## Provider Lock File Management

The `.terraform.lock.hcl` file pins provider versions and hardware-platform checksums. It should be committed to version control.

### Adding Cross-Platform Checksums

In a team environment where engineers work across Linux, macOS, and Windows, generate checksums for all target platforms upfront so no one needs to run `terraform init` after a pull:

```bash
terraform providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64
```

This populates `h1:` hashes for each platform combination into the lock file:

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.40.0"
  constraints = "~> 5.40"
  hashes = [
    "h1:abc123...linux_amd64",
    "h1:def456...linux_arm64",
    "h1:ghi789...darwin_amd64",
    "h1:jkl012...darwin_arm64",
    "h1:mno345...windows_amd64",
  ]
}
```

After running the lock command, commit the updated lock file:

```bash
git add .terraform.lock.hcl
git commit -m "chore: add cross-platform provider checksums for AWS 5.40.0"
```

### Using a Private Mirror

For air-gapped environments or when proxying through Artifactory:

```bash
terraform providers lock \
  -fs-mirror=/opt/terraform/mirror \
  -platform=linux_amd64
```

---

## AWS Credential Strategies

### Named Profiles

Use named profiles in `~/.aws/credentials` to separate environment-specific access keys without changing code:

```ini
[default]
aws_access_key_id     = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[prod]
aws_access_key_id     = AKIAI44QH8DHBEXAMPLE
aws_secret_access_key = je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY

[staging]
aws_access_key_id     = AKIAIOSFODNN7STAGING
aws_secret_access_key = stagingtokenExampleValue123EXAMPLEKEY
```

### Provider Block with Profile Selection

```hcl
provider "aws" {
  region                   = "us-east-1"
  profile                  = var.aws_profile
  shared_credentials_files = ["/opt/ci/credentials/.aws/credentials"]
}
```

```hcl
variable "aws_profile" {
  description = "AWS named profile to use for this workspace"
  type        = string
  default     = "default"
}
```

Pass profile selection at runtime:

```bash
# Local dev targeting staging
terraform apply -var="aws_profile=staging"

# CI/CD — inject via environment
export TF_VAR_aws_profile="prod"
terraform apply
```

> For CI/CD pipelines, prefer IAM roles with OIDC federation (GitHub Actions, GitLab CI) over long-lived access keys entirely. Named profiles are most useful for local developer workflows with multiple AWS accounts.

---

## Engineering Notes

### On Data Sources vs. Resources

Data sources (`data "vsphere_datacenter"`) are read-only lookups of infrastructure that exists *outside* this Terraform configuration — whether pre-provisioned manually, by another Terraform workspace, or by another tool entirely. Resources (`resource "vsphere_virtual_machine"`) manage object lifecycle. Mixing them correctly is the key to modular Terraform at scale: shared networking and identity infrastructure lives in its own workspace and is consumed via data sources by application workspaces.

### On Provider Aliases

Aliases exist at the provider level, not the resource level. This means you cannot dynamically assign a provider at runtime — the alias must be known at plan time. For highly dynamic multi-region deployments, evaluate whether `for_each` over a map of provider configurations (using `provider` meta-argument) or a module-per-region pattern is more maintainable.

### On Remote State and Team Collaboration

- [ ] S3 bucket versioning enabled
- [ ] DynamoDB state locking configured
- [ ] Bucket encryption at rest (AES-256 or KMS)
- [ ] Block public access on state bucket
- [ ] IAM policy restricts state bucket access to Terraform service accounts only
- [ ] `prevent_destroy = true` on bucket resource

### On Community Providers

Community providers (not under `hashicorp/` namespace) should be treated with the same scrutiny as any open-source dependency. Check: last release date, GitHub stars/forks, open issues, test coverage, and whether the maintainer has a security disclosure policy. Pin to a specific patch version; don't use `>=` constraints without an upper bound.

### On the Lock File

Never `.gitignore` the lock file. It is not auto-generated noise — it is a security artifact. The `h1:` hashes verify both the version *and* the platform binary, preventing supply-chain attacks where a compromised binary is published under an existing version number.

---

*Personal reference repo — configurations tested against Terraform 1.7.x, AWS Provider 5.x, vSphere Provider 2.6.x.*

# terraform-gcp-compute-engine

![Terraform](https://img.shields.io/badge/Terraform-1.x-7B42BC?logo=terraform&logoColor=white)
![Google Cloud](https://img.shields.io/badge/Google_Cloud-Compute_Engine-4285F4?logo=googlecloud&logoColor=white)
![Provider](https://img.shields.io/badge/hashicorp%2Fgoogle-4.0.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Terraform module for provisioning a hardened Compute Engine instance on GCP — including VPC, subnet, firewall rules, and SSH key injection. Intended as a reference implementation for reproducible, auditable VM provisioning in environments where golden-image pipelines aren't yet in place.

---

## Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Configuration Reference](#configuration-reference)
- [Usage](#usage)
- [Firewall Policy](#firewall-policy)
- [SSH Access](#ssh-access)
- [Outputs](#outputs)
- [Engineering Notes](#engineering-notes)
- [Teardown](#teardown)

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  GCP Project: project-1-XXXXXXX                          │
│                                                          │
│  ┌─────────────────────────────────────────────────┐     │
│  │  VPC: tf_network                                │     │
│  │                                                 │     │
│  │  ┌──────────────────────────────────────┐       │     │
│  │  │  Subnet: subnet-1 (10.0.1.0/24)      │       │     │
│  │  │  Region: us-east4                    │       │     │
│  │  │                                      │       │     │
│  │  │  ┌────────────────────────────────┐  │       │     │
│  │  │  │  google_vm_one                 │  │       │     │
│  │  │  │  Machine: f1-micro             │  │       │     │
│  │  │  │  OS: Debian 11                 │  │       │     │
│  │  │  │  Zone: us-east4-c              │  │       │     │
│  │  │  │  Int IP: 10.0.1.2              │  │       │     │
│  │  │  │  Ext IP: <ephemeral NAT>        │  │       │     │
│  │  │  └────────────────────────────────┘  │       │     │
│  │  └──────────────────────────────────────┘       │     │
│  │                                                 │     │
│  │  Firewall: allow-ssh (ingress TCP/22, 0.0.0.0/0)│     │
│  └─────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────┘
```

**Resources provisioned (4 total):**

| Resource | Terraform Name | GCP Type |
|---|---|---|
| VPC network | `tf_network` | `google_compute_network` |
| Subnetwork | `tf_subnetwork` | `google_compute_subnetwork` |
| VM instance | `google_vm_one` | `google_compute_instance` |
| Firewall rule | `allow-ssh` | `google_compute_firewall` |

---

## Prerequisites

### GCP Account Setup

- [ ] GCP account created at [cloud.google.com](https://cloud.google.com)
- [ ] Dedicated project created (do **not** reuse a production project)
- [ ] Compute Engine API enabled for the project
- [ ] Billing account attached to the project
- [ ] IAM permissions: `compute.instances.*` and `compute.firewalls.*`

> **Cost note:** An `f1-micro` instance in `us-east4` runs at approximately $0.0076/hour (~$5.50/month). Ephemeral external IPs add ~$0.004/hour when in use. Always run `terraform destroy` when finished to avoid unintended charges.

### Local Toolchain

```bash
# Verify Terraform
terraform version
# Terraform v1.x.x

# Verify gcloud CLI
gcloud version
# Google Cloud SDK 430.x.x

# Authenticate gcloud to your GCP project
gcloud init

# Issue application-default credentials (required by the Terraform Google provider)
gcloud auth application-default login
```

### SSH Key Pair

```bash
# Generate an RSA key pair for VM access
ssh-keygen -t rsa -b 4096 -C "ops@your-org.com" -f ./keys/ssh_key -N ""

# Output: ./keys/ssh_key (private) and ./keys/ssh_key.pub (public)
# The public key is injected into the VM via the `metadata` block in google.tf
```

---

## Configuration Reference

### `google.tf` — Provider & Variables

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "= 4.0.0"   # pinned; avoids breaking changes on re-init
    }
  }
}

provider "google" {
  project = "project-1-387809"   # replace with your project ID
  region  = "us-east4"
}
```

> **Why pin to `= 4.0.0`?** The GCP provider has a history of introducing breaking argument changes between minor versions (e.g., `network_interface` subfields in 4.x vs 5.x). Pinning ensures reproducible `terraform init` across machines and CI runners.

### VM Instance Block

```hcl
resource "google_compute_instance" "google_vm_one" {
  name         = "google-vm-one"
  machine_type = "f1-micro"
  zone         = "us-east4-c"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  metadata = {
    ssh-keys = "your-username:${file("./keys/ssh_key.pub")}"
  }

  metadata_startup_script = <<-EOT
    #!/bin/bash
    apt-get update -y
    apt-get install -y curl htop
  EOT

  network_interface {
    network    = google_compute_network.tf_network.id
    subnetwork = google_compute_subnetwork.tf_subnetwork.id

    access_config {
      # Ephemeral external IP assigned by GCP
    }
  }
}
```

### Network & Subnetwork

```hcl
resource "google_compute_network" "tf_network" {
  name                    = "tf-network"
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "tf_subnetwork" {
  name          = "subnet-1"
  ip_cidr_range = "10.0.1.0/24"
  region        = "us-east4"
  network       = google_compute_network.tf_network.id
}
```

---

## Usage

```bash
# Clone the repo and navigate to the working directory
git clone https://github.com/your-org/terraform-gcp-compute-engine.git
cd terraform-gcp-compute-engine

# Format check
terraform fmt -check

# Initialise — downloads hashicorp/google 4.0.0
terraform init

# Validate configuration syntax and provider schema
terraform validate

# Preview the execution plan
terraform plan -out=tfplan

# Apply (creates 4 resources, ~30 seconds)
terraform apply tfplan
```

Expected output on success:

```
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.

Outputs:

instance_public_ip = "34.86.xxx.xxx"
```

---

## Firewall Policy

```hcl
resource "google_compute_firewall" "allow_ssh" {
  name    = "allow-ssh"
  network = google_compute_network.tf_network.id

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]
  direction     = "INGRESS"
}
```

> **Production consideration:** `source_ranges = ["0.0.0.0/0"]` is acceptable for ephemeral dev/test instances, but in any shared or long-lived environment this should be scoped to your corporate egress IPs, a bastion host CIDR, or an IAP (Identity-Aware Proxy) tunnel. GCP's IAP-based SSH eliminates the need for a public SSH port entirely.

---

## SSH Access

Once `terraform apply` completes, connect using the private key:

```bash
# Using the output IP directly
ssh -i ./keys/ssh_key your-username@$(terraform output -raw instance_public_ip)

# Verify the OS
cat /etc/debian_version
# 11.7
```

Alternatively, SSH via the GCP Console (Cloud Shell or browser-based SSH) — GCP transfers the instance's metadata keys automatically.

---

## Outputs

```hcl
output "instance_public_ip" {
  description = "NAT-assigned external IP of the Compute Engine instance"
  value       = join("", google_compute_instance.google_vm_one.network_interface[0].access_config[*].nat_ip)
}
```

> **Why `join()` here?** `access_config` is a list (even when a single NAT IP is configured), so direct attribute access returns a tuple. `join("", [...])` collapses it to a plain string suitable for downstream use — for example, passing to an Ansible inventory or a DNS record resource.

---

## Engineering Notes

<details>
<summary><strong>GCP naming conventions vs AWS/Azure</strong></summary>

| Concept | AWS | Azure | GCP (Terraform) |
|---|---|---|---|
| Virtual machine | `aws_instance` | `azurerm_linux_virtual_machine` | `google_compute_instance` |
| Virtual network | `aws_vpc` | `azurerm_virtual_network` | `google_compute_network` |
| Subnet | `aws_subnet` | `azurerm_subnet` | `google_compute_subnetwork` |
| Security group / NSG | `aws_security_group` | `azurerm_network_security_group` | `google_compute_firewall` |
| Public IP | `public_ip` attribute | `azurerm_public_ip` resource | `access_config {}` block → `nat_ip` |

GCP combines what AWS separates into `aws_internet_gateway` + routing tables into the compute network model — external IPs are managed at the NIC level via `access_config`, not as standalone gateway resources.

</details>

<details>
<summary><strong>Authentication flow</strong></summary>

The `hashicorp/google` provider resolves credentials in this order:

1. `GOOGLE_APPLICATION_CREDENTIALS` env var pointing to a service account JSON key
2. Application Default Credentials (ADC) — the file written by `gcloud auth application-default login`
3. GCE metadata server (when running Terraform inside a GCP VM with an attached service account)

For CI/CD pipelines, option 1 with a dedicated, least-privilege service account is the correct approach. ADC is fine for local development.

</details>

<details>
<summary><strong>Zone availability fallback</strong></summary>

`us-east4-c` is the primary zone used here. If `terraform apply` fails with a quota or availability error, try:

```hcl
zone = "us-east4-a"   # or "us-east4-b"
```

GCP zone capacity can be asymmetric, especially for `f1-micro` (which is a shared-core machine type subject to spot-style preemption pressure during high-demand periods).

</details>

---

## Teardown

```bash
# Destroy all 4 resources (~60 seconds)
terraform destroy

# Verify in GCP Console: Compute Engine → VM Instances should be empty
# Or via CLI:
gcloud compute instances list --project=project-1-XXXXXXX
# Listed 0 items.
```

> Do not rely on `terraform.tfstate` alone to confirm teardown. Always cross-check in the GCP Console or via `gcloud` CLI to ensure no orphaned resources remain billable.

---

## References

- [Terraform Registry — hashicorp/google provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [google_compute_instance resource docs](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance)
- [GCP IAP TCP tunneling (SSH without public IP)](https://cloud.google.com/iap/docs/using-tcp-forwarding)
- [GCP Pricing Calculator](https://cloud.google.com/products/calculator)
- [gcloud CLI installation](https://cloud.google.com/sdk/docs/install)

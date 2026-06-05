# Terraform — Azure Provider: VM Provisioning with Network Stack

![Terraform](https://img.shields.io/badge/Terraform-3.0.0-7B42BC?logo=terraform&logoColor=white)
![AzureRM](https://img.shields.io/badge/Provider-AzureRM_3.0.0-0078D4?logo=microsoftazure&logoColor=white)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu_22.04_LTS-E95420?logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Status-Reference-green)

Terraform configuration for provisioning a production-style Linux VM on Azure, including all prerequisite network infrastructure: VNet, subnet, public IP, NIC, and NSG. Intended as a hands-on reference for the AzureRM provider's resource dependency model.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Resource Group: rg-eastus-01                            │
│                                                          │
│   ┌─────────────────────────────────────────────┐        │
│   │  Virtual Network: 10.0.0.0/16               │        │
│   │                                             │        │
│   │   ┌─────────────────────────────────────┐   │        │
│   │   │  Subnet (internal): 10.0.2.0/24     │   │        │
│   │   │                                     │   │        │
│   │   │   ┌──────────────────────────────┐  │   │        │
│   │   │   │  NIC (nic-01)                │  │   │        │
│   │   │   │  ├─ Private IP  (dynamic)    │  │   │        │
│   │   │   │  └─ Public IP   (dynamic) ◄──┼──┼───┼── SSH  │
│   │   │   └──────────────┬───────────────┘  │   │        │
│   │   │                  │  NSG: allow 22   │   │        │
│   │   │          ┌───────▼───────┐          │   │        │
│   │   │          │  azure-vm-01  │          │   │        │
│   │   │          │  Standard_B1s │          │   │        │
│   │   │          │  Ubuntu 22.04 │          │   │        │
│   │   │          └───────────────┘          │   │        │
│   │   └─────────────────────────────────────┘   │        │
│   └─────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

**Resource dependency chain**: Resource Group → VNet → Subnet + Public IP → NIC (binds subnet + PIP) → NSG association → Linux VM

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Terraform | ≥ 1.0 | Pinned provider at `3.0.0` |
| Azure CLI | ≥ 2.49 | Used for `az login` auth flow |
| Azure account | — | Free tier covers `Standard_B1s` for testing |
| SSH key pair | RSA or ED25519 | Must be pre-generated; path referenced in config |

### Azure CLI Installation (Debian/Ubuntu)

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Verify
az version
```

### Authentication

```bash
# Standard interactive login (opens browser)
az login

# MFA / service principal environments
az login --tenant <tenant-id> --use-device-code

# Confirm active subscription
az account show --query "{subscription:name, id:id}" -o table
```

> **Note:** `az login` writes a token to `~/.azure/`. This token is what the AzureRM Terraform provider uses when no `client_id`/`client_secret` are supplied in the provider block — suitable for local development. For CI/CD, use a service principal via environment variables (`ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`).

### Generate SSH Key Pair

```bash
ssh-keygen -t rsa -b 4096 -f ./keys/ssh-key -N ""
# Public key path referenced in Terraform: ./keys/ssh-key.pub
```

---

## Configuration Reference

<details>
<summary><strong>azure.tf — Full annotated source</strong></summary>

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "=3.0.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# --- Resource Group ---
resource "azurerm_resource_group" "rg_one" {
  name     = "rg-eastus-01"
  location = "East US"
}

# --- Networking ---
resource "azurerm_virtual_network" "network_one" {
  name                = "vnet-eastus-01"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg_one.location
  resource_group_name = azurerm_resource_group.rg_one.name
}

resource "azurerm_subnet" "subnet_a" {
  name                 = "internal"
  resource_group_name  = azurerm_resource_group.rg_one.name
  virtual_network_name = azurerm_virtual_network.network_one.name
  address_prefixes     = ["10.0.2.0/24"]
}

resource "azurerm_public_ip" "public_ip" {
  name                = "pip-azure-vm-01"
  resource_group_name = azurerm_resource_group.rg_one.name
  location            = azurerm_resource_group.rg_one.location
  allocation_method   = "Dynamic"
}

# --- Security ---
resource "azurerm_network_security_group" "sg_ssh" {
  name                = "nsg-allow-ssh"
  location            = azurerm_resource_group.rg_one.location
  resource_group_name = azurerm_resource_group.rg_one.name

  security_rule {
    name                       = "allow-ssh-inbound"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# --- Network Interface ---
resource "azurerm_network_interface" "nic_one" {
  name                = "nic-01"
  location            = azurerm_resource_group.rg_one.location
  resource_group_name = azurerm_resource_group.rg_one.name

  ip_configuration {
    name                          = "nic-01"
    subnet_id                     = azurerm_subnet.subnet_a.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.public_ip.id
  }
}

resource "azurerm_network_interface_security_group_association" "example" {
  network_interface_id      = azurerm_network_interface.nic_one.id
  network_security_group_id = azurerm_network_security_group.sg_ssh.id
}

# --- Compute ---
resource "azurerm_linux_virtual_machine" "azure_vm_one" {
  name                = "azure-vm-01"
  resource_group_name = azurerm_resource_group.rg_one.name
  location            = azurerm_resource_group.rg_one.location
  size                = "Standard_B1s"
  admin_username      = "adminuser"
  network_interface_ids = [
    azurerm_network_interface.nic_one.id
  ]

  admin_ssh_key {
    username   = "adminuser"
    public_key = file("../keys/ssh-key.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }
}

# --- Outputs ---
output "resource_group_name" {
  value = azurerm_resource_group.rg_one.name
}

output "public_ip_address" {
  value = azurerm_linux_virtual_machine.azure_vm_one.public_ip_address
}
```

</details>

---

## Usage

### Deploy

```bash
# 1. Initialise — downloads azurerm provider plugin
terraform init

# 2. Validate syntax and provider schema
terraform validate

# 3. Preview execution plan
terraform plan -out=tfplan

# 4. Apply (creates 8 resources)
terraform apply tfplan
```

Expected output after apply:

```
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

public_ip_address   = "20.x.x.x"
resource_group_name = "rg-eastus-01"
```

### Connect

```bash
# Retrieve outputs at any time without re-running plan
terraform output

# SSH into the VM
ssh -i ./keys/ssh-key adminuser@$(terraform output -raw public_ip_address)
```

### Tear Down

```bash
terraform destroy
```

> **Always destroy after testing.** `Standard_B1s` costs ~$0.011/hr. Azure may also retain a `NetworkWatcher` resource in a separate resource group after destroy — delete it manually via the portal or with `az network watcher delete`.

---

## Resource Inventory

| # | Terraform Resource | Azure Type | Purpose |
|---|---|---|---|
| 1 | `azurerm_resource_group.rg_one` | Resource Group | Logical container for all resources |
| 2 | `azurerm_virtual_network.network_one` | Virtual Network | Private address space (10.0.0.0/16) |
| 3 | `azurerm_subnet.subnet_a` | Subnet | Internal segment (10.0.2.0/24) |
| 4 | `azurerm_public_ip.public_ip` | Public IP | Dynamically allocated ingress IP |
| 5 | `azurerm_network_security_group.sg_ssh` | NSG | Inbound allow TCP/22; all outbound open |
| 6 | `azurerm_network_interface.nic_one` | Network Interface | Binds subnet + PIP to the VM |
| 7 | `azurerm_network_interface_security_group_association` | NSG ↔ NIC binding | Attaches NSG policy to the NIC |
| 8 | `azurerm_linux_virtual_machine.azure_vm_one` | Linux VM | Ubuntu 22.04 LTS, Standard_B1s |

---

## Engineering Notes

### AzureRM vs AWS Provider — Mental Model Differences

The AzureRM provider requires explicit resource group scoping on virtually every resource (`resource_group_name`). There is no implicit global namespace — every resource lives inside a group. This is architecturally cleaner than AWS's implicit account-level scope, but means more boilerplate per resource block.

### NSG Attachment Model

Azure NSGs can be associated at the **subnet** level or the **NIC** level. This config uses NIC-level association (`azurerm_network_interface_security_group_association`), which gives per-VM control. For fleet deployments, subnet-level association scales better and reduces resource count.

### Dynamic Public IP Caveat

`allocation_method = "Dynamic"` means the public IP is not assigned until the VM starts, and it changes on stop/start cycles. For anything beyond ephemeral dev use, switch to `"Static"` and add a DNS label:

```hcl
resource "azurerm_public_ip" "public_ip" {
  allocation_method   = "Static"
  sku                 = "Standard"
  domain_name_label   = "my-vm-prod-01"
  ...
}
```

### Provider Version Pinning

`version = "=3.0.0"` uses the exact-match constraint intentionally. The AzureRM provider has historically introduced breaking schema changes between minor versions (argument renames, deprecated sub-blocks). Unpinned, `terraform init` will pull the latest and may break source image references or OS disk arguments silently. Lock this and upgrade deliberately.

### Production Hardening Checklist

- [ ] Replace `source_address_prefix = "*"` in NSG with a known CIDR (bastion host, VPN gateway, office egress)
- [ ] Switch public IP to `Static` + `Standard` SKU; associate with a DNS label
- [ ] Move SSH key path to a `variable` with `sensitive = true`; source from Vault or Azure Key Vault
- [ ] Add `azurerm_managed_disk` + separate data disk for workload storage
- [ ] Parameterise `location` and `size` via `variables.tf` for multi-environment parity
- [ ] Store Terraform state remotely in Azure Blob Storage with state locking via `azurerm` backend

<details>
<summary><strong>Remote state backend config (Azure Blob)</strong></summary>

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "prod/azure-vm/terraform.tfstate"
  }
}
```

Bootstrap the storage account once:

```bash
az group create --name rg-tfstate --location eastus
az storage account create \
  --name tfstateaccount \
  --resource-group rg-tfstate \
  --sku Standard_LRS \
  --encryption-services blob

az storage container create \
  --name tfstate \
  --account-name tfstateaccount
```

</details>

---

## Repository Layout

```
.
├── keys/
│   ├── ssh-key          # Private key (git-ignored)
│   └── ssh-key.pub      # Public key injected into VM at provisioning time
└── azure/
    ├── azure.tf         # All resources in single file (monolith for reference clarity)
    └── README.md
```

For production, decompose into:

```
azure/
├── provider.tf      # terraform{} + provider{}
├── variables.tf     # input variable declarations
├── main.tf          # resource blocks
├── outputs.tf       # output blocks
└── terraform.tfvars # environment-specific values (git-ignored)
```

---

## Reference

- [AzureRM Provider Docs (3.0.0)](https://registry.terraform.io/providers/hashicorp/azurerm/3.0.0/docs)
- [Azure VM Sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes)
- [Azure NSG Overview](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Terraform AzureRM Backend](https://developer.hashicorp.com/terraform/language/backend/azurerm)

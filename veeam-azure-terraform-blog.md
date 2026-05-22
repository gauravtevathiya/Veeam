# Deploy Veeam Backup & Replication v13 on Azure using Terraform

> **Automate your enterprise backup infrastructure on Microsoft Azure with Infrastructure-as-Code**

---

## Table of Contents

1. [Introduction](#introduction)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Project Structure](#project-structure)
5. [Terraform Configuration — Step by Step](#terraform-configuration)
6. [Security Considerations](#security-considerations)
7. [Deployment Guide](#deployment-guide)
8. [Post-Deployment Steps](#post-deployment-steps)
9. [Cost Estimate](#cost-estimate)
10. [Conclusion](#conclusion)

---

## Introduction

Veeam Backup & Replication v13 is one of the most powerful and widely-used enterprise backup solutions available today. When combined with **Microsoft Azure** as the infrastructure platform and **Terraform** as the provisioning tool, you get a fully automated, repeatable, and version-controlled backup infrastructure.

In this blog, we will walk through a complete Terraform configuration to deploy Veeam Backup & Replication v13 from the **Azure Marketplace** — including the Virtual Network, Subnet, NSG, Storage Account, Public IP, NIC, and the Veeam Windows VM itself.

**What we'll build:**
- A dedicated Azure Resource Group in Central India
- A Virtual Network with a public subnet
- A Network Security Group (NSG) with inbound/outbound rules
- An Azure Storage Account (for Veeam backup repository)
- A Windows VM running Veeam B&R v13 (from Azure Marketplace)
- A Static Public IP for remote access

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│              Azure Resource Group: CCD-CI-RG         │
│                   (Central India)                     │
│                                                       │
│  ┌─────────────────────────────────────────────┐     │
│  │         VNET: CCD-CI-VNET01                 │     │
│  │         Address Space: 10.0.0.0/24          │     │
│  │                                             │     │
│  │  ┌───────────────────────────────────────┐  │     │
│  │  │     Subnet: CCD-PUBLIC-SUBNET         │  │     │
│  │  │     Prefix: 10.0.0.0/24              │  │     │
│  │  │                                       │  │     │
│  │  │  ┌─────────────────────────────────┐  │  │     │
│  │  │  │   VM: CCD-VEEAM-VBR01          │  │  │     │
│  │  │  │   Size: Standard_D4s_v3        │  │  │     │
│  │  │  │   OS: Veeam B&R v13 (Windows)  │  │  │     │
│  │  │  │   NIC ──► Public IP (Static)   │  │  │     │
│  │  │  └─────────────────────────────────┘  │  │     │
│  │  │                                       │  │     │
│  │  └───────────────────────────────────────┘  │     │
│  │                      │                      │     │
│  │              NSG: CCD-CI-NSG                │     │
│  └─────────────────────────────────────────────┘     │
│                                                       │
│  Storage Account: ccdstgveeambackup (LRS, Std)        │
└─────────────────────────────────────────────────────┘
```

---

## Prerequisites

Before you begin, make sure you have the following:

| Requirement | Details |
|---|---|
| Azure Subscription | Active subscription with Contributor access |
| Terraform | v1.3.0 or later |
| Azure CLI | Logged in via `az login` |
| Veeam License | Valid Veeam B&R license key |
| Azure Marketplace | Accept the Veeam marketplace terms |

### Accept the Marketplace Terms

Before deploying from the Marketplace, you **must** accept the legal terms. Run the following command:

```bash
az vm image terms accept \
  --publisher veeam \
  --offer veeam-backup-replication \
  --plan veeam-backup-replication-v13
```

---

## Project Structure

```
veeam-azure-terraform/
├── main.tf           # All resource definitions
├── variables.tf      # (Optional) Variable declarations
├── outputs.tf        # Output values
├── providers.tf      # AzureRM provider config
└── README.md         # This blog (in markdown)
```

---

## Terraform Configuration

### 1. Provider Configuration (`providers.tf`)

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  required_version = ">= 1.3.0"
}

provider "azurerm" {
  features {}
}
```

---

### 2. Local Variables

All the configurable values are defined in `locals` block — making it easy to change names and settings from a single place.

```hcl
locals {
  location             = "Central India"
  resource_group_name  = "CCD-CI-RG"
  vnet_name            = "CCD-CI-VNET01"
  vnet_address_space   = ["10.0.0.0/24"]
  subnet_name          = "CCD-PUBLIC-SUBNET"
  subnet_prefix        = ["10.0.0.0/24"]
  nsg_name             = "CCD-CI-NSG"
  storage_account_name = "ccdstgveeambackup"
  vm_name              = "CCD-VEEAM-VBR01"
  vm_size              = "Standard_D4s_v3"
  admin_user           = "azureadmin"
  admin_password       = "P@ssw0rd1234!"   # ⚠️ Use Azure Key Vault in production!
}
```

> **⚠️ Security Warning:** Never commit passwords in plain text to a public repository. Use `terraform.tfvars` + `.gitignore`, or **Azure Key Vault** for production workloads.

---

### 3. Resource Group

```hcl
resource "azurerm_resource_group" "rg" {
  name     = local.resource_group_name
  location = local.location
}
```

---

### 4. Virtual Network and Subnet

```hcl
resource "azurerm_virtual_network" "vnet" {
  name                = local.vnet_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = local.vnet_address_space
}

resource "azurerm_subnet" "subnet" {
  name                 = local.subnet_name
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = local.subnet_prefix
}
```

---

### 5. Network Security Group (NSG)

```hcl
resource "azurerm_network_security_group" "nsg" {
  name                = local.nsg_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "Allow-All-Inbound"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "Allow-All-Outbound"
    priority                   = 101
    direction                  = "Outbound"
    access                     = "Allow"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_subnet_network_security_group_association" "nsg_assoc" {
  subnet_id                 = azurerm_subnet.subnet.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}
```

> **⚠️ Production Tip:** The `Allow-All-Inbound` rule is fine for testing/dev, but in production, restrict inbound to only required ports — `9392/TCP` (Veeam console), `3389/TCP` (RDP from trusted IPs only), `443/TCP` (HTTPS).

---

### 6. Storage Account

```hcl
resource "azurerm_storage_account" "storage" {
  name                     = local.storage_account_name
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
  min_tls_version          = "TLS1_2"
}
```

This storage account will serve as a **Scale-Out Backup Repository (SOBR)** target or a direct Veeam backup repository.

---

### 7. Public IP and Network Interface

```hcl
resource "azurerm_public_ip" "pip" {
  name                = "${local.vm_name}-PIP"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_interface" "nic" {
  name                = "${local.vm_name}-NIC"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id
  }
}
```

---

### 8. Veeam Windows VM (Azure Marketplace)

```hcl
resource "azurerm_windows_virtual_machine" "veeam_vm" {
  name                = local.vm_name
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = local.vm_size
  admin_username      = local.admin_user
  admin_password      = local.admin_password

  network_interface_ids = [azurerm_network_interface.nic.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "veeam"
    offer     = "veeam-backup-replication"
    sku       = "veeam-backup-replication-v13"
    version   = "latest"
  }

  plan {
    publisher = "veeam"
    product   = "veeam-backup-replication"
    name      = "veeam-backup-replication-v13"
  }
}
```

The `plan` block is **mandatory** when deploying from Azure Marketplace. Without it, Terraform will throw an error.

---

### 9. Output

```hcl
output "public_ip" {
  value = azurerm_public_ip.pip.ip_address
}
```

---

## Security Considerations

Before pushing to GitHub or using in production, address these points:

| Issue | Recommendation |
|---|---|
| Hardcoded password | Move to `terraform.tfvars` + add to `.gitignore`, or use Azure Key Vault |
| Allow-All NSG rules | Restrict to specific ports and source IPs |
| Public IP exposed | Use Azure Bastion for RDP instead of public IP |
| No state backend | Configure Azure Blob Storage as remote state backend |
| No encryption at rest | Enable Azure Disk Encryption for the OS disk |

### Recommended `.gitignore` for Terraform

```
# .gitignore
*.tfvars
*.tfstate
*.tfstate.backup
.terraform/
.terraform.lock.hcl
crash.log
```

---

## Deployment Guide

Follow these steps to deploy:

```bash
# Step 1: Clone the repository
git clone https://github.com/YOUR_USERNAME/veeam-azure-terraform.git
cd veeam-azure-terraform

# Step 2: Login to Azure
az login

# Step 3: Accept Marketplace terms
az vm image terms accept \
  --publisher veeam \
  --offer veeam-backup-replication \
  --plan veeam-backup-replication-v13

# Step 4: Initialize Terraform
terraform init

# Step 5: Review the plan
terraform plan

# Step 6: Apply the configuration
terraform apply -auto-approve

# Step 7: Get the Public IP
terraform output public_ip
```

---

## Post-Deployment Steps

After the VM is deployed:

1. **RDP into the VM** using the public IP and credentials
2. **Launch Veeam B&R Console** from the desktop shortcut
3. **Enter your Veeam license key** (or use the Community Edition — free for up to 10 workloads)
4. **Add the Azure Storage Account** as a backup repository:
   - Go to Backup Infrastructure → Backup Repositories → Add Repository
   - Select "Microsoft Azure Blob Storage"
   - Provide the storage account name and access key
5. **Configure backup jobs** for your VMs, physical servers, or cloud workloads
6. **Set up email notifications** from Veeam → General Options → Email Settings

---

## Cost Estimate

| Resource | SKU | Estimated Monthly Cost |
|---|---|---|
| VM: Standard_D4s_v3 | 4 vCPU / 16 GB RAM | ~$140–$160 USD |
| OS Disk: Premium_LRS | 128 GB default | ~$20 USD |
| Storage Account: LRS | Depends on data | ~$5–$50 USD |
| Public IP: Standard | Static | ~$4 USD |
| **Total** | | **~$170–$235 USD/month** |

> Costs vary by region and usage. Use the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/) for exact estimates.

---

## Conclusion

With this Terraform configuration, you can spin up a fully functional **Veeam Backup & Replication v13** environment on Azure in under 10 minutes. The use of locals, marketplace image references, and the `plan` block makes this configuration clean, readable, and reusable.

**Next steps to explore:**
- Add a **data disk** for the Veeam backup repository
- Configure **Azure Blob Storage** as immutable backup storage (Object Lock)
- Set up **Veeam ONE** for monitoring and analytics
- Implement **Azure Bastion** to remove the public IP exposure
- Add **Terraform remote state** with Azure Blob Storage

---

## References

- [Veeam Backup & Replication v13 Documentation](https://www.veeam.com/documentation-guides-datasheets.html)
- [Veeam on Azure Marketplace](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/veeam.veeam-backup-replication)
- [AzureRM Terraform Provider Docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Terraform Best Practices](https://developer.hashicorp.com/terraform/language/style)

---

*Written with ❤️ | Infrastructure as Code | Azure | Veeam | Terraform*

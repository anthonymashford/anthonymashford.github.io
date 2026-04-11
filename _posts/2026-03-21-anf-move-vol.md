---
layout*: post
date: 2026-03-21 12:00
title: Non-disruptively Move an ANF Volume Using Terraform
subtitle: Deploying Azure NetApp Files with Terraform — Including Non-Disruptive Volume Migration.
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, AVD, FSLogix, Zone Redundant]
author: Anthony Mashford
---

# Deploying Azure NetApp Files with Terraform — Including Non-Disruptive Volume Migration

Azure NetApp Files (ANF) is Microsoft's enterprise-grade, high-performance NFS and SMB file storage service built directly into Azure. Whether you're running Oracle databases, SAP HANA, high-throughput AI/ML pipelines, or persistent Kubernetes volumes, ANF delivers the low latency and throughput that general-purpose Azure storage simply can't match.

In this blog, we'll walk through two things:

1. **Part 1 — Building a basic ANF lab** using Terraform: a Resource Group, ANF Account, Capacity Pool, and an NFS volume in UK South.
2. **Part 2 — Non-disruptive volume migration** between capacity pools using Terraform — upgrading from Standard to Premium service level without any downtime.

---

## Prerequisites

Before you begin, make sure you have the following in place:

- An active Azure subscription with the `Microsoft.NetApp` resource provider registered
- Terraform v1.5 or later installed
- The Azure CLI installed and authenticated (`az login`)
- The `azurerm` Terraform provider v4.x (note: v4 introduced breaking changes from v3)
- A virtual network and delegated subnet for ANF already provisioned, or included in your Terraform config

> **Note on subnet delegation:** ANF requires a dedicated subnet delegated to `Microsoft.NetApp/volumes`. This cannot be shared with other services. We'll include this in the Terraform below.

---

## Part 1 — Building the ANF Lab

### What We're Deploying

| Resource | Name | Details |
|---|---|---|
| Resource Group | `rg-anf-lab-uks` | UK South |
| Virtual Network | `vnet-anf-lab-uks` | 10.0.0.0/16 |
| Subnet | `snet-anf-lab-uks` | 10.0.1.0/24, delegated to ANF |
| ANF Account | `anf-account-lab` | UK South |
| Capacity Pool | `pool-standard-4tb` | Standard service level, 4 TiB |
| NFS Volume | `vol-nfs-lab-001` | 1 TiB, NFS 4.1 |

### Provider Configuration

Start with your `providers.tf`. The `azurerm` v4 provider requires an explicit `subscription_id` — this is a breaking change from v3 and a common source of confusion when upgrading.

```hcl
# providers.tf

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
  required_version = ">= 1.5.0"
}

provider "azurerm" {
  subscription_id = var.subscription_id
  features {}
}
```

### Variables

```hcl
# variables.tf

variable "subscription_id" {
  description = "Azure Subscription ID"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "uksouth"
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default = {
    environment = "lab"
    managed_by  = "terraform"
    workload    = "anf"
  }
}
```

### Networking

ANF requires a dedicated subnet with a specific service delegation. In `azurerm` v4, the delegation block uses inline `service_delegation` syntax.

> **Important:** The `name` value in the `service_delegation` block is case-sensitive. It must be exactly `Microsoft.NetApp/volumes` — any variation will cause a provider error.

```hcl
# network.tf

resource "azurerm_resource_group" "anf_lab" {
  name     = "rg-anf-lab-uks"
  location = var.location
  tags     = var.tags
}

resource "azurerm_virtual_network" "anf_vnet" {
  name                = "vnet-anf-lab-uks"
  location            = azurerm_resource_group.anf_lab.location
  resource_group_name = azurerm_resource_group.anf_lab.name
  address_space       = ["10.0.0.0/16"]
  tags                = var.tags
}

resource "azurerm_subnet" "anf_subnet" {
  name                 = "snet-anf-lab-uks"
  resource_group_name  = azurerm_resource_group.anf_lab.name
  virtual_network_name = azurerm_virtual_network.anf_vnet.name
  address_prefixes     = ["10.0.1.0/24"]

  delegation {
    name = "anf-delegation"

    service_delegation {
      name = "Microsoft.NetApp/volumes"
      actions = [
        "Microsoft.Network/networkinterfaces/*",
        "Microsoft.Network/virtualNetworks/subnets/join/action"
      ]
    }
  }
}
```

### ANF Account and Capacity Pool

The ANF Account is the top-level logical container for capacity pools and volumes. The capacity pool defines the service level and overall provisioned size.

```hcl
# anf.tf

resource "azurerm_netapp_account" "anf_account" {
  name                = "anf-account-lab"
  location            = azurerm_resource_group.anf_lab.location
  resource_group_name = azurerm_resource_group.anf_lab.name
  tags                = var.tags
}

resource "azurerm_netapp_pool" "standard_pool" {
  name                = "pool-standard-4tb"
  location            = azurerm_resource_group.anf_lab.location
  resource_group_name = azurerm_resource_group.anf_lab.name
  account_name        = azurerm_netapp_account.anf_account.name

  service_level = "Standard"
  size_in_tb    = 4

  tags = var.tags
}
```

### NFS Volume

The volume is where your workloads will actually mount and write data. We're creating a 1 TiB NFS 4.1 volume with a basic export policy allowing read/write from the local VNet range.

```hcl
# volume.tf

resource "azurerm_netapp_volume" "nfs_vol_001" {
  name                = "vol-nfs-lab-001"
  location            = azurerm_resource_group.anf_lab.location
  resource_group_name = azurerm_resource_group.anf_lab.name
  account_name        = azurerm_netapp_account.anf_account.name
  pool_name           = azurerm_netapp_pool.standard_pool.name

  volume_path         = "vol-nfs-lab-001"
  service_level       = "Standard"
  subnet_id           = azurerm_subnet.anf_subnet.id
  storage_quota_in_gb = 1024  # 1 TiB

  protocols = ["NFSv4.1"]

  export_policy_rule {
    rule_index          = 1
    allowed_clients     = ["10.0.0.0/16"]
    protocols_enabled   = ["NFSv4.1"]
    unix_read_write     = true
    root_access_enabled = false
  }

  tags = var.tags
}
```

> **A note on `protocols_enabled`:** This field accepts an enum, not a free-text string. Valid values are `NFSv3`, `NFSv4.1`, and `CIFS`. Using `nfsv4.1` or `NFS4.1` will cause a provider validation error.

### Deploying

```bash
terraform init
terraform plan -var="subscription_id=<your-subscription-id>"
terraform apply -var="subscription_id=<your-subscription-id>"
```

Once complete, you'll have a fully functional ANF volume mounted-ready in UK South, accessible from anything in your VNet.

---

## Part 2 — Non-Disruptive Volume Migration to a Premium Pool

One of ANF's most powerful operational capabilities is the ability to move a volume between capacity pools **without any downtime**. This is a live, online operation — your workloads continue to read and write throughout the migration.

This is genuinely useful in practice. You might start a workload on a Standard pool during development and move it to Premium or Ultra once it hits production, without scheduling a maintenance window.

### How It Works

ANF pool migration works by updating the `pool_name` and `service_level` attributes on the volume resource. Under the hood, Azure rebalances the volume to the new pool while it remains fully online. From a Terraform perspective, you're simply changing two attributes and running `terraform apply`.

### Step 1 — Create the Premium Capacity Pool

Add a second pool to your `anf.tf`:

```hcl
# anf.tf — add this block

resource "azurerm_netapp_pool" "premium_pool" {
  name                = "pool-premium-4tb"
  location            = azurerm_resource_group.anf_lab.location
  resource_group_name = azurerm_resource_group.anf_lab.name
  account_name        = azurerm_netapp_account.anf_account.name

  service_level = "Premium"
  size_in_tb    = 4

  tags = var.tags
}
```

Apply this change first to provision the pool before migrating the volume:

```bash
terraform apply -var="subscription_id=<your-subscription-id>" -target=azurerm_netapp_pool.premium_pool
```

### Step 2 — Update the Volume to Reference the New Pool

In `volume.tf`, update the `pool_name` and `service_level` on your existing volume resource:

```hcl
# volume.tf — updated

resource "azurerm_netapp_volume" "nfs_vol_001" {
  name                = "vol-nfs-lab-001"
  location            = azurerm_resource_group.anf_lab.location
  resource_group_name = azurerm_resource_group.anf_lab.name
  account_name        = azurerm_netapp_account.anf_account.name

  # Updated: point to the premium pool
  pool_name           = azurerm_netapp_pool.premium_pool.name
  service_level       = "Premium"

  volume_path         = "vol-nfs-lab-001"
  subnet_id           = azurerm_subnet.anf_subnet.id
  storage_quota_in_gb = 1024

  protocols = ["NFSv4.1"]

  export_policy_rule {
    rule_index          = 1
    allowed_clients     = ["10.0.0.0/16"]
    protocols_enabled   = ["NFSv4.1"]
    unix_read_write     = true
    root_access_enabled = false
  }

  tags = var.tags
}
```

### Step 3 — Apply the Migration

```bash
terraform plan -var="subscription_id=<your-subscription-id>"
```

The plan output will show something like:

```
  ~ resource "azurerm_netapp_volume" "nfs_vol_001" {
      ~ pool_name     = "pool-standard-4tb" -> "pool-premium-4tb"
      ~ service_level = "Standard" -> "Premium"
    }
```

No destroy. No recreate. Just an in-place update.

```bash
terraform apply -var="subscription_id=<your-subscription-id>"
```

Azure will execute the pool change operation. The volume remains mounted and available throughout — typically completing within a few minutes depending on volume size and data volume.

### Step 4 — (Optional) Decommission the Standard Pool

Once the migration is complete and verified, you can remove the Standard pool from your config and apply:

```hcl
# Remove or comment out the standard pool resource block
# resource "azurerm_netapp_pool" "standard_pool" { ... }
```

```bash
terraform apply -var="subscription_id=<your-subscription-id>"
```

> **Important:** A capacity pool cannot be deleted while it still contains volumes. Attempting to do so will result in an error. Always migrate or delete all volumes from a pool before removing it.

---

## Summary

In this walkthrough we've covered:

- Deploying a complete ANF environment in UK South using Terraform and the `azurerm` v4 provider
- Key provider quirks to be aware of: mandatory `subscription_id`, case-sensitive delegation naming, and `protocols_enabled` enum values
- Creating a Standard capacity pool and NFS 4.1 volume
- Adding a Premium pool and performing a non-disruptive, live volume migration with a simple Terraform update

Non-disruptive pool migration is one of ANF's standout operational capabilities, and Terraform makes it straightforward to manage declaratively. You get the agility to match your storage service level to your workload's actual requirements — without scheduling downtime.

---

## Complete File Structure

```
anf-lab/
├── providers.tf
├── variables.tf
├── network.tf
├── anf.tf
└── volume.tf
```

All configuration shown above is production-ready for a lab environment. For production deployments, consider adding a `terraform.tfvars` file, remote state backend (Azure Storage), and additional export policy rules scoped to specific client CIDRs.

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new){:target="_blank"} page.

---

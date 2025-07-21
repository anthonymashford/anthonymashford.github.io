---
layout*: post
date: 2025-06-20 16:00
title: Implementing BCDR Features in Azure NetApp Files
subtitle: Using Infrastructure-as-Code to deploy Azure NetApp Files Features
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Backup, Replication, Snapshots]
author: Anthony Mashford
---

## Introduction

In this blog, will we be looking at implementing features available in Azure NetApp Files (ANF) that can be used to support Business Continuity and Disaster Recovery (BCDR).

This blog is a follow on from last months artcile about Business Continuity and Disaster Recovery (BCDR) capabilities in Azure NetApp Files (ANF). If you would like to recap on BCDR features in ANF, you can read that blog post [Beyond Performance](https://www.azuretechlab.com/2025-05-19-ANF-BCDR/){:target="_blank"}.

## Using Infrastructure-as-Code to deploy Azure NetApp Files BCDR Features

Infrastructure-as-Code (IaC) empowers teams to automate the provisioning and management of cloud resources with precision and repeatability, transforming manual configurations into scalable code. When it comes to deploying Azure NetApp Files—a high-performance file storage service for enterprise workloads—IaC offers a streamlined path to consistency and compliance across environments. While a wide variety of tools such as Bicep, ARM templates, and Pulumi are available to implement IaC, this blog focuses on **Terraform** due to its broad adoption, modular architecture, and robust support for Azure services. Using Terraform, we’ll demonstrate how to declaratively define and deploy Azure NetApp Files resources, enabling seamless integration into your DevOps workflows.

## What are we going to build?

We are going to build a lab environment that will include the following resources:

- Resource Group
- Virtual Network
- Subnets - A **Servers** subnet and an **ANF** subnet that has subnet delegation to the Microsoft.NetApp/Volumes service.
- Azure NetApp Files **Account**
- Azure NetApp Files **Volume**
- Azure NetApp Files **Backup Vault**
- Azure NetApp Files **Backup Policy**
- Azure NetApp Files **Snapshot Policy**
- Configure Azure NetApp **Files Volume Replication**

In the following sections we will see examples of how to implement these features.

## Creating the Reource Group

First, we are going to create an Azure Resource Group. We will do this in the **main** file. This file will set the Terraform version requirement, the provider and features. You will need to add your subscription ID to this file were indicated.

~~~Terraform
terraform {
  required_version = ">= 1.0.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 4.34.0"
    }
  }
}

provider "azurerm" {
  subscription_id = "<<YOUR SUDID HERE>>"
  features {
    netapp {
      prevent_volume_destruction             = false
      delete_backups_on_backup_vault_destroy = true
    }
  }
}

resource "azurerm_resource_group" "anf_bcdr_rg" {
  name     = "anf-bcdr-rg"
  location = "UK South"
  tags = {
    Environment = "lab"
  }
}
~~~

## Creating Virtual Network and Subnets

Next, we are going to create the Virtual Network (VNet) and Subnets. One subnet will have access **delegated** to the **Microsoft.NetApp/volumes** service. This allows ANF to be brought into the VNet and accessible to the network.

The code example below shows the creation on the VNet and Subnets.

~~~Terraform
resource "azurerm_virtual_network" "vnet_anf" {
  name                = "vnet-anf"
  location            = azurerm_resource_group.anf_bcdr_rg.location
  resource_group_name = azurerm_resource_group.anf_bcdr_rg.name
  address_space       = ["172.16.0.0/16"]
  tags = {
    Environment = "lab"
  }
}

resource "azurerm_subnet" "subnet_servers" {
  name                 = "subnet-servers"
  resource_group_name  = azurerm_resource_group.anf_bcdr_rg.name
  virtual_network_name = azurerm_virtual_network.vnet_anf.name
  address_prefixes     = ["172.16.1.0/24"]
}

resource "azurerm_subnet" "subnet_anf" {
  name                 = "subnet-anf"
  resource_group_name  = azurerm_resource_group.anf_bcdr_rg.name
  virtual_network_name = azurerm_virtual_network.vnet_anf.name
  address_prefixes     = ["172.16.2.0/24"]

  delegation {
    name = "netapp"

    service_delegation {
      name    = "Microsoft.Netapp/volumes"
      actions = ["Microsoft.Network/networkinterfaces/*", "Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}
~~~

## Creating Azure NetApp Files Account

An Azure NetApp Files account is a top-level resource within the Azure ecosystem that enables you to deploy enterprise-grade, high-performance file storage backed by NetApp technology.

The code example below shows how to create and ANF Account.

~~~Terraform
resource "azurerm_netapp_account" "anf_account" {
  name                = "anf-lab"
  location            = azurerm_resource_group.anf_bcdr_rg.location
  resource_group_name = azurerm_resource_group.anf_bcdr_rg.name
  tags = {
    Environment = "lab"
  }
}
~~~

## Creating ANF Backup Vault

Azure NetApp Files Backup Vault is a low-cost, long-retention solution that securely stores snapshots offloaded from Azure NetApp Files volumes. It protects against accidental deletion and compliance risks while preserving recovery points outside the primary ANF environment, making backups space-efficient by storing only incremental changes. Users can restore backups to new volumes in the same region and manage vaults with both automated policies and manual options. Though multiple vaults are supported, using a single vault per subscription simplifies operations. Pricing is based on storage used and restore traffic, with no setup fees. It's a strategic tool for optimising data protection and TCO conversations around ANF.

The code example below shows how to create the ANF Backup Vault.

~~~Terraform
resource "azurerm_netapp_backup_vault" "anf_backup_vault" {
  name                = "anf-lab-backup-vault"
  resource_group_name = azurerm_resource_group.anf_bcdr_rg.name
  location            = azurerm_resource_group.anf_bcdr_rg.location
  account_name        = azurerm_netapp_account.anf_account.name
  tags = {
    Environment = "lab"
  }
}
~~~

## Creating ANF Backup Policy

An Azure NetApp Files Backup Policy is a configuration that automates the scheduling and retention of backups for your volumes, helping ensure consistent, compliant, and long-term data protection. It lets you define how frequently backups are taken—daily, weekly, or monthly—and how many of each to retain, with the system capturing a baseline snapshot upon policy activation and transferring backups to Azure Blob Storage. Once assigned to a volume and its backup vault, the policy runs independently to safeguard data without manual intervention, providing a scalable and cost-effective solution for disaster recovery, governance, and operational continuity.

The code example below shows how to create the ANF Backup Policy.

~~~Terraform
resource "azurerm_netapp_backup_policy" "anf_backup_policy" {
  name                    = "anf-lab-backup-policy"
  resource_group_name     = azurerm_resource_group.anf_bcdr_rg.name
  location                = azurerm_resource_group.anf_bcdr_rg.location
  account_name            = azurerm_netapp_account.anf_account.name
  enabled                 = true
  daily_backups_to_keep   = 7
  weekly_backups_to_keep  = 4
  monthly_backups_to_keep = 12

  tags = {
    Environment = "lab"
  }
}
~~~

## Craeting ANF Snapshot Policy

An Azure NetApp Files snapshot policy is a configurable data protection tool that automatically creates and retains point-in-time backups for volumes hosted in Azure NetApp Files. These policies enable administrators to set the schedule for snapshots—whether hourly, daily, weekly, or monthly—and determine how long each backup is kept. By defining these settings, organisations can maintain reliable data recovery points while managing storage consumption and meeting compliance requirements. It’s a key component of enterprise-grade data management in the Azure environment, helping to safeguard important data against deletion or corruption and making recovery straightforward when needed.

The code example below shows how to create an ANF Snapshot Policy.

~~~Terraform
resource "azurerm_netapp_snapshot_policy" "anf_snapshot_policy" {
  name                = "anf-lab-snapshot-policy"
  location            = azurerm_resource_group.anf_bcdr_rg.location
  resource_group_name = azurerm_resource_group.anf_bcdr_rg.name
  account_name        = azurerm_netapp_account.anf_account.name
  enabled             = true

  hourly_schedule {
    snapshots_to_keep = 12
    minute            = 0
  }

  daily_schedule {
    snapshots_to_keep = 7
    hour              = 0
    minute            = 15
  }

  weekly_schedule {
    snapshots_to_keep = 4
    days_of_week      = ["Sunday"]
    hour              = 0
    minute            = 30
  }

  monthly_schedule {
    snapshots_to_keep = 12
    days_of_month     = [1]
    hour              = 0
    minute            = 45
  }
}
~~~

## Creating ANF Capacity Pool

An Azure NetApp Files capacity pool is a scalable allocation of storage resources within your Azure subscription, designed to host one or more volumes. It lets you provision storage in tiers—Flexible, Standard, Premium, or Ultra—allowing you to balance cost and performance depending on workload needs. Capacity pools simplify management by decoupling storage provisioning from compute, and they support dynamic adjustments, meaning you can scale volume sizes or performance levels on the fly. Ideal for scenarios like enterprise databases, virtualization, and analytics, capacity pools make it easy to align storage usage with business priorities while maintaining agility.

The code example below shows how to create an ANF Capacity Pool.

~~~Terraform
resource "azurerm_netapp_pool" "anf_pool" {
  name                = "anf-lab-pool"
  location            = azurerm_resource_group.anf_bcdr_rg.location
  resource_group_name = azurerm_resource_group.anf_bcdr_rg.name
  account_name        = azurerm_netapp_account.anf_account.name
  size_in_tb          = 1
  service_level       = "Standard"
}
~~~

## Creating ANf Volume

An Azure NetApp Files (ANF) volume is a high-performance, scalable storage resource allocated within a capacity pool in Azure, designed to support enterprise-grade workloads such as databases, virtual desktops, and containerised applications. Each volume is provisioned with a chosen service level—Standard, Premium or Ultra—which determines its throughput capability, and is accessed via a delegated subnet within an Azure virtual network. ANF volumes support multiple protocols including NFS and SMB, and offer advanced capabilities such as snapshot policies, cross-region replication, and integration with proximity placement groups for improved latency. Export policies govern access rules, and volumes can be customised with quotas, encryption and tiering features to suit varied performance and compliance requirements.

The code example below shows how to create an ANF Volumes. It will also apply the Backup Policy and Snapshot Policy we created previously.

~~~Terraform
resource "azurerm_netapp_volume" "anf_volume_src" {
  depends_on          = [azurerm_netapp_snapshot_policy.anf_snapshot_policy]
  name                = "anf-lab-volume-src"
  location            = azurerm_resource_group.anf_bcdr_rg.location
  resource_group_name = azurerm_resource_group.anf_bcdr_rg.name
  account_name        = azurerm_netapp_account.anf_account.name
  pool_name           = azurerm_netapp_pool.anf_pool.name
  subnet_id           = azurerm_subnet.subnet_anf.id
  network_features    = "Standard"
  volume_path         = "anf-lab-volume-src"
  storage_quota_in_gb = 100
  service_level       = "Standard"
  zone                = "1"

  data_protection_snapshot_policy {
    snapshot_policy_id = azurerm_netapp_snapshot_policy.anf_snapshot_policy.id
  }

  data_protection_backup_policy {
    backup_policy_id = azurerm_netapp_backup_policy.anf_backup_policy.id
    backup_vault_id  = azurerm_netapp_backup_vault.anf_backup_vault.id
    policy_enabled   = true
  }

  lifecycle {
    prevent_destroy = false
  }
}
~~~

## Creating ANF replication Volume

Azure NetApp Files (ANF) replication volumes provide asynchronous cross-region or cross-zone data protection by replicating a source volume in one Azure region or zone to a destination volume in another. This setup supports business continuity during regional outages or disasters, with replication schedules available at 10-minute, hourly or daily intervals depending on performance requirements and data change frequency. The destination volume remains read-only under normal conditions and becomes writable during failover, enabling rapid recovery with minimal downtime—typically under a minute for storage activation. Replication is available across various regional and non-regional pairs and service levels, offering flexibility in cost and performance optimisation.

The code example below shows how to create an ANF Replication Volume.

~~~Terraform
resource "azurerm_netapp_volume" "anf_volume_dst" {
  depends_on          = [azurerm_netapp_volume.anf_volume_src]
  name                = "anf-lab-volume-dst"
  location            = azurerm_resource_group.anf_bcdr_rg.location
  resource_group_name = azurerm_resource_group.anf_bcdr_rg.name
  account_name        = azurerm_netapp_account.anf_account.name
  pool_name           = azurerm_netapp_pool.anf_pool.name
  subnet_id           = azurerm_subnet.subnet_anf.id
  network_features    = "Standard"
  volume_path         = "anf-lab-volume-dst"
  storage_quota_in_gb = 100
  service_level       = "Standard"
  zone                = "2"

  data_protection_replication {
    endpoint_type             = "dst"
    remote_volume_location    = azurerm_netapp_volume.anf_volume_src.location
    remote_volume_resource_id = azurerm_netapp_volume.anf_volume_src.id
    replication_frequency     = "10minutes"
  }

  lifecycle {
    prevent_destroy = false
  }
}
~~~

## Summary

Azure NetApp Files is far more than high-performance cloud storage—it’s a strategic asset in crafting robust Business Continuity and Disaster Recovery (BCDR) plans. With built-in cross-region or cross-zone replication, rapid snapshot restoration, and automated backups to long-term vaults, ANF equips organisations to preserve data integrity and availability amid unexpected disruptions. Designed for enterprise-grade workloads, its native integration with Azure services ensures seamless operations and adherence to compliance standards across cloud environments. By reducing recovery time objectives (RTOs), reinforcing recovery point objectives (RPOs), and aligning storage policies with practical resilience goals, Azure NetApp Files helps safeguard customer data and sustain operational continuity when it's most critical.

I hope this short blog post about using Terraform to configure Azure NetApp Files Business Continuity and Disaster Recovery features using IaC method with Terraform has been useful. You can find a complete example of the Terraform script in my GitHub repo located [in this Github Repo](https://github.com/anthonymashford/Terraform-Examples/blob/c29c099a9beea079ff666a4c4a7bc870f93173e4/TF-Creating%20Azure%20NetApp%20Files%20Backup%20Vault%20and%20Backup%20Policy/main.tf){:target="_blank"}

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://github.com/anthonymashford/ANF-BCDR-Terraform/tree/main){:target="_blank"} page.

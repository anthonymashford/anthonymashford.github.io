---
layout*: post
date: 2025-01-14 12:00
title: Configure Azure NetApp Files Backup Using Terraform
subtitle: Using Terraform to configure Azure NetApp Files Backup Vault & Backup Policy
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf-backup.svg
share-img: /assets/img/anf-backup.svg
tags: [Blog, Azure, Azure NetApp Files]
author: Anthony Mashford
---

## Introduction

Ensuring the security and availability of data has become paramount for businesses of all sizes. Azure NetApp Files provides enterprise-grade storage and backup solutions in the cloud, offering high performance and security for critical applications. However, managing these resources manually can be time-consuming and error-prone. This is where Infrastructure as Code (IaC) tools like Terraform come into play.

Terraform, developed by HashiCorp, is a powerful open-source tool that enables the safe and efficient management of cloud infrastructure. With Terraform, you can define your Azure NetApp Files Backup configuration in code, allowing for automation and repeatability.

In this blog post, we’ll delve into the step-by-step process of using Terraform to configure Azure NetApp Files Backup, we will also add Azure NetApp Files Snapshot Policy for good measure 😊.

Whether you’re an IT professional looking to streamline your operations or a developer interested in automating infrastructure, this guide will provide you with the knowledge and tools needed to leverage Terraform for this purpose.

## What are we going to build?

To allow for the setup of Backup Policy and Backup Vault, we will need some additional resources in place to start with. In this lab we will be building the following.

- Azure Resource Group
- Virtual Network
- Subnet with delegation for the Microsoft.NetApp/volumes
- Azure Netapp Files (ANF) account
- ANF capacity pool (Standard service level)
- ANF volume
- Snapshot policy
- Backup Policy
- Backup Vault

## Building the base lab resources

First, we will build the base lab resources requried to allow for the configuration of ANF Backup Policy and ANF Backup Vault. Below is a snippet showing the terraform provider and the creation of a resource group. Please note, in Terraform version 4, you will need to specify the subscription ID, see example below.

~~~Terraform
provider "azurerm" {
  features {}
  # You will need to add your subscription_id here
  subscription_id = "<<Add your sudID here>>"
}

resource "azurerm_resource_group" "rg-bkuplab" {
  name     = "rg-anf-weu-bkuplab"
  location = "West Europe"
}
~~~

Next, we will add a virtual network and a subnet, with delegation to Azure NetApp Files. The snippet below is an example, change IP address ranges and names as needed.

~~~Terraform
resource "azurerm_virtual_network" "rg-bkuplab" {
  name                = "vnet-anf-weu-bkuplab"
  address_space       = ["172.16.0.0/16"]
  location            = azurerm_resource_group.rg-bkuplab.location
  resource_group_name = azurerm_resource_group.rg-bkuplab.name
}

resource "azurerm_subnet" "anf_subnet" {
  name                 = "subnet-anf"
  resource_group_name  = azurerm_resource_group.rg-bkuplab.name
  virtual_network_name = azurerm_virtual_network.rg-bkuplab.name
  address_prefixes     = ["172.16.1.0/24"]

  delegation {
    name = "netapp"

    service_delegation {
      name    = "Microsoft.Netapp/volumes"
      actions = ["Microsoft.Network/networkinterfaces/*", "Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}
~~~

Thats the lab resources created, now lets configure an Azure NetApp Files account. In this example, I have included a tag, always use tags, they make life a lot easier when trying to figure out what resources are being used for or who owns them etc.

~~~Terraform
resource "azurerm_netapp_account" "anf-account" {
  name                = "anf-account"
  location            = azurerm_resource_group.rg-bkuplab.location
  resource_group_name = azurerm_resource_group.rg-bkuplab.name
  tags = {
    environment = "bkuplab"
  }
}
~~~

## Creating a Snapshot Policy

In this section we will create a Snapshot Policy in the ANF account we created in the previous section. The example below creates a Snapshot Policy with hourly, daily, weekly and monthly intervals.

~~~Terraform
resource "azurerm_netapp_snapshot_policy" "anf-snapshot-policy" {
  name                = "anf-snapshot-policy"
  location            = azurerm_resource_group.rg-bkuplab.location
  resource_group_name = azurerm_resource_group.rg-bkuplab.name
  account_name        = azurerm_netapp_account.anf-account.name
  enabled             = true

  hourly_schedule {
    minute            = 0
    snapshots_to_keep = 1
  }

  daily_schedule {
    minute            = 15
    hour              = 01
    snapshots_to_keep = 7
  }

  weekly_schedule {
    minute            = 30
    hour              = 01
    days_of_week      = ["Sunday"]
    snapshots_to_keep = 4
    }

  monthly_schedule {
    minute            = 45
    hour              = 01
    days_of_month     = [1]
    snapshots_to_keep = 12
    }
  
}
~~~

## Creating a Backup Policy

Now, we will created the ANF Backup Policy with daily, weekly and monthly retentions and attach it to our ANF account from the previous section.

~~~Terraform
resource "azurerm_netapp_backup_policy" "anf-backup-policy" {
  name                = "anf-backup-policy"
  location            = azurerm_resource_group.rg-bkuplab.location
  resource_group_name = azurerm_resource_group.rg-bkuplab.name
  account_name        = azurerm_netapp_account.anf-account.name

  enabled                 = true
  daily_backups_to_keep   = 7
  weekly_backups_to_keep  = 4
  monthly_backups_to_keep = 12
}
~~~

## Creating a Backup Vault

The next code snippet will create an ANF Backup Vault to store your backups and attached it to our ANF account.

~~~Terraform
resource "azurerm_netapp_backup_vault" "anf-backup-vault" {
  name                = "anf-backup-vault"
  location            = azurerm_resource_group.rg-bkuplab.location
  resource_group_name = azurerm_resource_group.rg-bkuplab.name
  account_name        = azurerm_netapp_account.anf-account.name
}
~~~

## Create an Azure NetApp Capacity Pool

The snippet below shows an example of an Azure NetApp Files capacity pool in our ANF account with 4 TiB of Standard level storage. The capacity pool will be created in our ANF account we created earlier.

~~~Terraform
resource "azurerm_netapp_pool" "anf-pool" {
  name                = "anf-pool"
  location            = azurerm_resource_group.rg-bkuplab.location
  resource_group_name = azurerm_resource_group.rg-bkuplab.name
  account_name        = azurerm_netapp_account.anf-account.name
  size_in_tb          = 4
  service_level       = "Standard"
}
~~~

## Create an ANF Volume

In this section we will create an ANF volume in the capacity pool we created above. We will also attach the Snapshot Policy and Backup Policy and target the Backup Vault to store our backups.

~~~Terraform
resource "azurerm_netapp_volume" "anf-vol" {
  name                = "anf-vol"
  location            = azurerm_resource_group.rg-bkuplab.location
  resource_group_name = azurerm_resource_group.rg-bkuplab.name
  account_name        = azurerm_netapp_account.anf-account.name
  pool_name           = azurerm_netapp_pool.anf-pool.name
  subnet_id           = azurerm_subnet.anf_subnet.id
  volume_path         = "bkuplab"
  service_level       = "Standard"
  storage_quota_in_gb = 100
  data_protection_snapshot_policy {
    snapshot_policy_id = azurerm_netapp_snapshot_policy.anf-snapshot-policy.id
  }

    data_protection_backup_policy {
        backup_vault_id = azurerm_netapp_backup_vault.anf-backup-vault.id
        backup_policy_id = azurerm_netapp_backup_policy.anf-backup-policy.id
        policy_enabled = true   
    }

  lifecycle {
    prevent_destroy = true
  }
}
~~~

## Summary

I hope this short blog post about using Terraform to configure Azure NetApp Files Backup has been useful. You can find a complete example of the Terraform script in my GitHub repo located [here](https://github.com/anthonymashford/Terraform-Examples/blob/c29c099a9beea079ff666a4c4a7bc870f93173e4/TF-Creating%20Azure%20NetApp%20Files%20Backup%20Vault%20and%20Backup%20Policy/main.tf){:target="_blank"}

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new){:target="_blank"} page.

---
layout*: post
date: 2026-04-28 12:00
title: The Fort Knox of File Shares in Azure
subtitle: Azure NetApp Files - Secure by Default, Resilient by Design
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, AVD, FSLogix, Zone Redundant]
author: Anthony Mashford
---

## Azure NetApp Files: Secure by Default, Resilient by Design

When architecting enterprise-grade file storage in the cloud, security and resilience aren't just checkboxes—they are the foundation. As cloud architects, we are constantly evaluating services to ensure they meet stringent compliance and availability requirements. Enter **Azure NetApp Files (ANF)**. 

Built on **bare-metal NetApp hardware natively integrated into Azure data centres**, ANF isn't just fast; it is arguably the most secure and resilient File Storage service in Azure, backed by an impressive **99.99% availability SLA**. Let’s dive into the architecture that makes ANF secure by default and resilient by design, effectively turning your storage into a true File Fortress.

When designing solutions around ANF, it is crucial to align with the [Microsoft Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-netapp-files), particularly the pillars of **Security and Reliability**. While ANF provides a robust, enterprise-grade foundation, the cloud **Shared Responsibility Model** dictates that while Microsoft and NetApp secure the underlying physical hardware and PaaS infrastructure, you—the customer—are responsible for securing your data *in* the cloud. This means the onus is on the architect to properly configure network isolation, enforce encryption standards, manage access controls, and design a highly available topology using the tools ANF provides.

---

## Secure by Default: Building the File Fortress

Security in ANF is multi-layered, ensuring your data is protected both in transit and at rest.

### 1. VNet Isolation (Network Security)

Unlike many PaaS services that rely on public endpoints and complex firewall rules, ANF is **inherently private**. It requires a **delegated subnet** within your Azure Virtual Network (VNet). This means your storage has **no public IP address** and is completely isolated from the public internet. Access is strictly controlled via your VNet, Network Security Groups (NSGs), and ExpressRoute/VPN connections.

### 2. Data in Transit: Protocol-Level Encryption

While VNet isolation keeps traffic off the public internet, ANF provides robust mechanisms to encrypt data as it travels between your compute clients and the storage volumes, ensuring protection against packet sniffing and man-in-the-middle attacks:

* **SMB3 Encryption:** For Windows workloads, ANF fully supports SMB 3.0 and SMB 3.1.1 encryption. You can enforce encryption at the volume level, ensuring that all data in flight is secured using **AES-128-CCM or AES-256-GCM** cryptographic suites. *Caution: While ANF processes this efficiently, enabling SMB encryption inherently introduces a performance overhead. It is highly recommended to benchmark this against your specific workload to ensure latency and throughput requirements are still met.*
* **NFSv4.1 with Kerberos:** For Linux workloads requiring high security, ANF supports NFSv4.1 with Kerberos authentication. By configuring `krb5p` (Kerberos 5 with Privacy), you ensure that not only the authentication headers but the **entire payload data is encrypted in transit**.
* **Active Directory/LDAP over TLS:** When ANF communicates with your Active Directory for identity mapping and access control (especially in dual-protocol or NFSv4.1 scenarios), it supports **LDAP over TLS (LDAPS)**. This guarantees that user credentials and identity lookups are never sent in clear text.
* **Network-Layer Encryption:** If you are accessing ANF from on-premises, data in transit can be further secured using **Azure VPN Gateways (IPsec) or ExpressRoute with MACsec**, providing an end-to-end encrypted tunnel before the protocol-level encryption even takes over.

### 3. Encryption at Rest: Single, Double, and Customer-Managed Keys (CMK)

Data at rest is always encrypted in ANF using **FIPS 140-2 compliant AES-256 encryption**. But ANF takes it further:

* **Single Encryption:** Enabled by default using Microsoft-managed keys.
* **Double Encryption:** For highly regulated industries, ANF supports **double encryption at rest**, encrypting data at both the hardware (drive level) and software (volume level). This ensures that even if the physical media or the storage controller is compromised, the data remains unreadable. *Caution: Double encryption comes at a premium and incurs additional costs. It should be carefully evaluated against your specific compliance mandates and budget constraints.*
* **Customer-Managed Keys (CMK):** You can bring your own keys via **Azure Key Vault**. This gives you ultimate cryptographic control, allowing you to revoke access to the storage instantly by disabling the key.

---

## Resilient by Design: You have design choices to meet business requirements

Hardware fails, and natural disasters happen. To support its 99.99% SLA, ANF provides architectural flexibility to survive catastrophic events and ensure continuous business operations. Powered by NetApp's industry-leading **SnapMirror** technology, replication in ANF is highly efficient, transferring only changed data blocks across the wire. 

**Replication Intervals:** Whether you are using cross-zone or cross-region replication, the asynchronous SnapMirror replication schedules can be configured to run every **10 minutes, one hour, or one day**. This flexibility allows architects to strictly align storage replication with the business's Recovery Point Objective (RPO).

### 1. Cross-Zone and Cross-Region Protection

* **Cross-Zone Replication:** You can replicate your ANF volumes synchronously or asynchronously across **Availability Zones (AZs)** within the same region. This ensures a zero or near-zero RPO if a specific data centre goes dark.
* **Cross-Region Replication (CRR):** For true Disaster Recovery, ANF allows asynchronous replication to a **paired Azure region**. 
* **Cross-Zone-Region Protection:** The ultimate architecture. You can replicate from a specific zone in your primary region to a specific zone in your secondary region, giving you **granular control over your blast radius**.

### 2. Snapshot Technology

A defining feature of NetApp technology is its **Redirect-on-Write (RoW) Snapshots**. Unlike traditional backups that take hours and consume massive storage, ANF snapshots are **instantaneous and space-efficient**, regardless of volume size.

* **Ransomware Protection:** Because snapshots are **read-only and immutable at the storage layer**, they are your best defense against ransomware. If a malicious actor encrypts your active file system, you can revert the entire volume to a clean snapshot in seconds.
* **Accidental Deletion/Corruption:** Users can easily restore individual files from the hidden `.snapshot` directory without IT intervention.

### 3. Azure NetApp Files Backup Vault

For long-term retention and compliance, the **ANF Backup Vault** extends the snapshot capability. It offloads your snapshots to cost-effective Azure storage, **completely independent of the volume itself**. If a volume is accidentally deleted, the backups in the vault remain safe and can be restored to a new volume.

---

## Infrastructure as Code: Deploying Secure ANF with Terraform

To truly embrace the cloud, we must deploy via code. Below is a Terraform snippet demonstrating how to deploy a secure ANF account with Customer-Managed Keys, a capacity pool configured for Double Encryption, and a primary volume in Zone 1 replicating to a secondary volume in Zone 2.

```hcl
# 1. Define the NetApp Account with Customer Managed Keys (CMK)
resource "azurerm_netapp_account" "secure_anf" {
  name                = "anf-secure-account"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location

  identity {
    type = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.anf_identity.id]
  }

  # CMK configuration linking to Azure Key Vault
  encryption {
    key_vault_key_id          = azurerm_key_vault_key.anf_cmk.id
    user_assigned_identity_id = azurerm_user_assigned_identity.anf_identity.id
  }
}

# 2. Create the Backup Vault for Long-Term Retention
resource "azurerm_netapp_backup_vault" "anf_vault" {
  name                = "anf-backup-vault"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  account_name        = azurerm_netapp_account.secure_anf.name
}

# 3. Create a Backup Policy
resource "azurerm_netapp_backup_policy" "daily_backup" {
  name                = "daily-backup-policy"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  account_name        = azurerm_netapp_account.secure_anf.name
  daily_backups_to_keep = 30
  enabled             = true
}

# 4. Create the Capacity Pool with Double Encryption Enabled
resource "azurerm_netapp_pool" "premium_pool" {
  name                = "anf-premium-pool"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  account_name        = azurerm_netapp_account.secure_anf.name
  service_level       = "Premium"
  size_in_tb          = 4
  
  # Enable Double Encryption at the pool level
  encryption_type     = "Double"
}

# 5. Create the Primary VNet Isolated Volume (Zone 1)
resource "azurerm_netapp_volume" "secure_volume_primary" {
  name                = "secure-data-vol-primary"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  account_name        = azurerm_netapp_account.secure_anf.name
  pool_name           = azurerm_netapp_pool.premium_pool.name
  volume_path         = "secure-data-primary"
  service_level       = "Premium"
  subnet_id           = azurerm_subnet.anf_delegated_subnet.id
  storage_quota_in_gb = 1024
  zone                = "1" # Explicitly place in Availability Zone 1
  
  network_features    = "Standard"

  # Attach Backup Policy and Vault
  data_protection_backup {
    backup_vault_id  = azurerm_netapp_backup_vault.anf_vault.id
    backup_policy_id = azurerm_netapp_backup_policy.daily_backup.id
    vault_enabled    = true
  }
}

# 6. Create the Secondary Volume for Cross-Zone Replication (Zone 2)
resource "azurerm_netapp_volume" "secure_volume_secondary" {
  name                = "secure-data-vol-secondary"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  account_name        = azurerm_netapp_account.secure_anf.name
  pool_name           = azurerm_netapp_pool.premium_pool.name
  volume_path         = "secure-data-secondary"
  service_level       = "Premium"
  subnet_id           = azurerm_subnet.anf_delegated_subnet.id
  storage_quota_in_gb = 1024
  zone                = "2" # Place in Availability Zone 2 for high availability
  
  network_features    = "Standard"

  # Configure Cross-Zone Replication using SnapMirror technology
  data_protection_replication {
    endpoint_type             = "dst"
    remote_volume_location    = azurerm_resource_group.rg.location
    remote_volume_resource_id = azurerm_netapp_volume.secure_volume_primary.id
    # Replication intervals can be set to: "10minutely", "Hourly", or "Daily"
    replication_schedule      = "Hourly"
  }
}
```

### Summary

Azure NetApp Files is not just a storage migration target; it is a strategic architectural choice for your most demanding enterprise workloads, including **SAP HANA, High-Performance Computing (HPC), Azure VMware Solution (AVS), Azure Kubernetes Service (AKS), General File Shares, and mission-critical databases such as Oracle and SQL**. By leveraging VNet isolation, robust in-transit encryption protocols, CMK with double encryption at rest, and NetApp's legendary SnapMirror and snapshot technologies, you can build an impenetrable, highly available File Fortress in Azure that easily meets a 99.99% SLA. 

To continue your journey in mastering Azure NetApp Files, I highly recommend exploring the following official Microsoft Learn resources:

* [Azure NetApp Files Official Documentation](https://learn.microsoft.com/en-us/azure/azure-netapp-files/)
* [Security FAQs for Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-security-overview)
* [Understand Cross-Region Replication in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/cross-region-replication-introduction)
* [Azure NetApp Files Backup Vaults](https://learn.microsoft.com/en-us/azure/azure-netapp-files/snapshots-introduction)

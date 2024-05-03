---
layout: post
date: 2024-05-03 14:00
title: Microsoft Announces Azure NetApp Files Backup is now Generally Available
subtitle: Expanding Azure NetApp Files data protection capabilities
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf-announcement.png
share-img: /assets/img/anf-announcement.png
tags: [Blog, Azure, Azure NetApp Files]
author: Anthony Mashford
---

## Microsoft Announces Azure NetApp Files Backup is now Generally Available

Today Microsoft announced the general availability of Azure NetApp Files Backup.
![](/assets/img/celebration.jpeg)

### Overview
The Azure NetApp Files backup feature is designed to address long-term data protection needs. It complements the existing volume snapshots, which are primarily used for near-term recovery or cloning.

Backups created by this service are stored in Azure storage, independent of volume snapshots. This separation ensures that backups remain accessible even if the original volume snapshots are deleted or modified.

### Regional Availability
The Azure NetApp Files backup feature is available in the following regions - [ANF Backup Regional Availability](https://learn.microsoft.com/en-us/azure/azure-netapp-files/backup-introduction#supported-regions)

### Backup Storage
- Backups are stored in Azure storage, which provides durability and redundancy across multiple data centers.
- This means that your backup data is not tied to the specific Azure NetApp Files volume. Instead, it resides in a separate storage layer, ensuring its availability even if the original volume is deleted.

### Backup Vaults
- In order to use Azure NetApp File backup, a Backup vault is required. The backup vault is an organisational unit used to manage backups. 
- It is possible to create multiple Backup vaults, however, it is recommended to only use one vault.

### Backup Types
- Azure NetApp Files backup supports both policy-based (scheduled) backups and manual (on-demand) backups.
- Scheduled backups allow you to define backup policies based on your organisation’s requirements, while manual backups give you flexibility for ad-hoc backup operations.

### Restoring a Backup
It is important to note that it is only possible to restore a backup from within the same Azure NetApp Files account. Other considerations are as follows:
- Volumes that are larger than 10TiB can take many hours to complete a restore operation.
- Backups can only be restored to new volumes.
- Backups can be restored to a different capacity pool, within the same Azure NetApp Files account.
- Volumes cannot be mounted until the restore operation has completed.
- Protected volumes configured with Basic Networking features can be restored to volumes with Standard Network features and vice versa.

### Cost Model
- Pricing for Azure NetApp Files backup is based on the total amount of storage consumed by the backup.
- There are no setup charges or minimum usage fees.
- Backup restore costs are determined by the total amount of backup capacity restored during the billing cycle.

### Summary
In summary, the Azure NetApp Files backup feature provides a robust solution for long-term data protection, ensuring your critical data remains safe, compliant, and easily recoverable. By offloading backups to Azure storage, it extends the service’s capabilities and enhances your overall data management strategy.

For more information, you can refer to the official [Microsoft Learn Documentation](https://learn.microsoft.com/en-us/azure/azure-netapp-files/backup-introduction)

Additionally, check out [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new) for more updates on the service.




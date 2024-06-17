---
layout*: post
date: 2024-06-17 14:00
title: Azure NetApp Files Large Volumes now GA
subtitle: Expanding Horizons with Azure NetApp Files Large Volumes
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf-large-volume.png
share-img: /assets/img/anf-large-volume.png
tags: [Blog, Azure, Azure NetApp Files]
author: Anthony Mashford
---

## Azure NetApp Files Large Volumes now Generally Available

Recently, Microsoft announced that Azure NetApp Files now offers large volume capabilities, allowing for the creation of volumes ranging from 50 TiB to 500 TiB. This exceeds the standard volume size limit of 100 TiB and supports use cases requiring larger single namespace volumes, such as High-Performance Computing (HPC) in Electronic Design Automation (EDA) and Oil & Gas (O&G) industries.

Additionally, these large volumes support replication across zones and regions, enhancing data resilience for HPC, AI/ML, and extensive file content repositories. This ensures continuous operations for HPC simulations and EDA, provides robust data protection for AI/ML datasets, and secures large, infrequently changed files in content repositories. With these replication capabilities, Azure NetApp Files enhances data security and operational reliability.

## Considerations

Below are a some requirements and considerations when customers are looking to consume Azure NetApp Files Large Volumes.

- It is not possible to convert an Azure NetApp Files regular volume to a **Large Volume**
- When creating a Large Volume, the quota needs to be larger than **50TiB**
- A Large Volume cannot be resided to less than 30% of its lowest provisioned size
- A Large Volume cannot be resized to less than 50TiB
- A single Large Volume cannot exceed 500TiB
- Azure NetApp Files Backup does not support LArge Volumes
- Azure NetApp Files Application Volume Groups does not support Large Volumes
- Azure NetApp Files Large Volumes are currently not suited to workloads such as SAP HANA, Oracle, SQL Server
- Ceilings for throughput for all performance tiers of Large Volumes are capped by the existing 100TiB maximum capacity limits. Volumes can be grown to 500TiB, however, the throughput ceiling will be capped as per the table below:
  
| Service Tier | Quota (TiB) | Throughput (MiB/s) |
| ------------ | ----------- | ------------------ |
| Standard     | 50 to 500   | 1,600              |
| Premium      | 50 to 500   | 6,400              |
| Ultra        | 50 to 500   | 10,240             |

## Support Regions

A list of supported regions can be found [here](https://learn.microsoft.com/en-us/azure/azure-netapp-files/large-volumes-requirements-considerations#supported-regions) 

## Creating a Large Volume

Prior to creating a Large Volume, you must request an **increase in the regional capacity** for your subscription.

You will also need to register the Azure NetApp Files Large Volume feature. This is accomplished using a registration form which can be found [here](https://forms.microsoft.com/pages/responsepage.aspx?id=v4j5cvGGr0GRqy180BHbR2Qj2eZL0mZPv1iKUrDGvc9UN09MRlQ2RUkzWDVWQTU4WlBaV0U2NDAxRCQlQCN0PWcu)

Once you have registered the Large Volumes feature and your capacity increase has been confirmed, you will be able to create Azure NetApp Files volumes up to 500TiB in size.

During volume creation, simply select **Yes** for **Large Volume** and continue to create and manage to volume as any other regular volume.

## Summary

Azure NetApp Files’ Large Volumes feature enables the creation of volumes from 50 TiB to 500 TiB, catering to workloads that demand larger storage with a single namespace. The service now includes cross-zone and cross-region replication, offering enhanced data protection and availability for critical workloads, and large file content repositories, ensuring business continuity and data integrity across various scenarios.

To allow you to keep up to date with service enhancements and feature updates, checkout the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new) page for more updates on the service.

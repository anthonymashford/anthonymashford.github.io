---
layout*: post
date: 2024-08-11 18:00
title: Default Outbound Access for VM's is Being Retired
subtitle: Microsoft will soon be removing default outbound access to the internet for VM's
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/nat-gateway.svg
share-img: /assets/img/nat-gateway.svg
tags: [Blog, Azure, Azure Networking, NAT Gateway]
author: Anthony Mashford
---

## Introduction

It's not talked about much at the moment, but this rather major will impact every Azure customer and might trip up some people.

Since the beginning, outbound access has been enabled default. On September 30th, 2025, default outbound access for newly deployed services will be retired, across all Azure regions. More details on that announcement [here](https://azure.microsoft.com/en-gb/updates/default-outbound-access-for-vms-in-azure-will-be-retired-transition-to-a-new-method-of-internet-access/)

This means that newly created VM's after this date will not be able to connect to the internet by default. Existing VM's will continue to connect, but it is recommended that you implement an explicit method to deliver outbound access. 

Now, whilst this is a bit of a pain to deal with, its not a bad thing. Once transitioned, it will give you greater control over how VM's talk to the internet.

So, what options do you have...

## Transition outbound access for VM's

Below are a some requirements and considerations when customers are looking to consume Azure NetApp Files Large Volumes.

- It is not possible to convert an Azure NetApp Files regular volume to a **Large Volume**
- When creating a Large Volume, the quota needs to be larger than **50TiB**
- A Large Volume cannot be resized to less than 30% of its lowest provisioned size
- A Large Volume cannot be resized to less than **50TiB**
- A single Large Volume cannot exceed **500TiB**
- Azure NetApp Files Backup does not support Large Volumes
- Azure NetApp Files Application Volume Groups does not support Large Volumes
- Azure NetApp Files Large Volumes are currently not suited to workloads such as SAP HANA, Oracle, SQL Server
- Ceilings for throughput for all performance tiers of Large Volumes are capped by the existing 100TiB maximum capacity limits. Volumes can be grown to 500TiB, however, the **throughput ceiling will be capped** as per the table below:
  
| Service Tier | Quota (TiB) | Throughput (MiB/s) |
| ------------ | ----------- | ------------------ |
| Standard     | 50 to 500   | 1,600              |
| Premium      | 50 to 500   | 6,400              |
| Ultra        | 50 to 500   | 10,240             |

## Supported Regions

A list of supported regions can be found [here](https://learn.microsoft.com/en-us/azure/azure-netapp-files/large-volumes-requirements-considerations#supported-regions) 

## Creating a Large Volume

Prior to creating a Large Volume, you must request an **increase in the regional capacity** for your subscription.

You will also need to register the Azure NetApp Files Large Volume feature. This is accomplished using a registration form which can be found [here](https://forms.microsoft.com/pages/responsepage.aspx?id=v4j5cvGGr0GRqy180BHbR2Qj2eZL0mZPv1iKUrDGvc9UN09MRlQ2RUkzWDVWQTU4WlBaV0U2NDAxRCQlQCN0PWcu)

Once you have registered the Large Volumes feature and your capacity increase has been confirmed, you will be able to create Azure NetApp Files volumes up to 500TiB in size.

During volume creation, simply select **Yes** for **Large Volume** and continue to create and manage to volume as any other regular volume.

![](/assets/img/select-large-volume.png)

## Summary

Azure NetApp Files’ Large Volumes feature enables the creation of volumes from 50 TiB to 500 TiB, catering to workloads that demand larger storage with a single namespace. The service now includes cross-zone and cross-region replication, offering enhanced data protection and availability for **critical workloads**, and large file content repositories, ensuring business continuity and data integrity across various scenarios.

To allow you to keep up to date with service enhancements and feature updates, checkout the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new) page for more updates on the service.

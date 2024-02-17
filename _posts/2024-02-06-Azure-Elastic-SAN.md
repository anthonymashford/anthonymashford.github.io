---
layout: post
date: 2024-02-06 01:00
title: Deploying Azure Elastic SAN
subtitle: An Introduction to a new Azure native storage service
cover-img: /assets/img/blogbanner.jpeg
thumbnail-img: /assets/img/elastic-san.svg
share-img: /assets/img/elastic-san.svg
tags: [Blog, Azure, Azure Elastic SAN, Storage, Block Storage]
author: Anthony Mashford
---

## Microsoft Azure Elastic SAN is now GA

Recently Microsoft announced that Azure Elastic SAN is now generally available. 

## Overview
 
Azure Elastic SAN (Storage Area Network) is a cloud-native service offered by Microsoft. Here's an overview:

**Purpose**: Azure Elastic SAN is designed to optimise workloads and the integration between large scale databases and performance-intensive mission-critical applications.

**Interoperability**: It works with multiple types of compute resources such as Azure Virtual Machines, Azure VMware Solutions, and Azure Kubernetes Service.

**Compatibility**: Azure Elastic SAN volumes can connect to a wide variety of compute resources using the internet Small Computer Systems Interface (iSCSI) protocol.

**Simplified Provisioning and Management**: Elastic SAN simplifies deploying and managing storage at scale through grouping and policy enforcement.

**Performance**: With an Elastic SAN, it's possible to scale your performance up to millions of IOPS, with double-digit GB/s throughput, and have single-digit millisecond latency.

**Cost Optimization and Consolidation**: Cost optimization can be achieved with Elastic SAN since you can increase your SAN storage in bulk.

**Resources**: Each Azure Elastic SAN has two internal resources: Volume groups and volumes.

## Creating a SAN Instance

1. Login to the Azure portal and browse for **Elastic SAN**
2. On the Elastic SAN blade, click **Create**
![](/assets/img/esan-create.png)
3. On the **Basics** page, complete the Projects and Instance details as well as the Elastic SAN resource provisioning. Once completed, click **Next**
![](/assets/img/esan-basics.png)
4. On the **Networking** page, choose whether or not you want to enable **Public** access. Exercise caution at this point, as enabling public access will make your Elastic SAN instance accessible over the Internet. Once completed, click **Next**
![](/assets/img/esan-networking.png)
5. On the **Volume Groups** page, click **Create volume group**.
![](/assets/img/esan-volumegroup.png)
6. On the volume group basics page, name your volume group.
![](/assets/img/esan-vgbasics.png)
7. On the volume group networking page, click **Create a private endpoint**
![](/assets/img/esan-vgnetworking-pe.png)
8. On the **Create private endpoint** page, fill in the name and select the virtual network and subnet. You can also choose to integrate with private DNS. CLick **OK**
![](/assets/img/esan-create-pe.png)
9. On the next page, click **Create**
10. Then, click **Next**
11. On the **Volume** page, name your volume and choose its size (in GiB). Click **Next**
![](/assets/img/esan-volume.png)
12. On the Tags page, add your tags as appropriate.
![](/assets/img/esan-tags.png)
13. Finally, click ** Review + Create**
![](/assets/img/esan-reviewcreate.png)
14. That's it, you've now successfully created your Elastic SAN instance.
![](/assets/img/esan-success.png)

## Summary
Azure Elastic SAN (Storage Area Network) is a cloud-native service that offers a scalable, cost-effective, high-performance, and comprehensive storage solution for a range of compute options. It provides an end-to-end experience similar to on-premises SAN.

It extends storage support beyond Azure Virtual Machines to include Azure Kubernetes Service (AKS), Azure VMware Solution, cloud-native applications, and edge appliances through industry standard interface iSCSI protocols. It also offers high levels of resiliency without having to set up multiple deployments of SAN arrays and a replication policy.

So, what are you waiting for, get logged into Azure and have a go at setting up your own Elastic SAN instance 😊
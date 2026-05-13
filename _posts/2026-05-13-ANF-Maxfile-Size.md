---
layout*: post
date: 2026-05-13 12:00
title: Big News for Big Data
subtitle: Azure NetApp Files Now Supports 64 TiB Files!
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, AVD, FSLogix, Zone Redundant]
author: Anthony Mashford
---

## Introduction

Hello, cloudy people! As a Cloud Solutions Architect, I spend a lot of time helping enterprises design robust, scalable, and high-performance environments in Azure. One of the most critical foundational elements of any major cloud migration is storage. Today, I am thrilled to share an absolutely massive update—literally and figuratively.

The **General Availability of Azure NetApp Files support for large files up to 64 TiB on regular volumes**!

If you are running data-intensive applications, migrating large-scale VMware environments, or dealing with colossal datasets, this is the game-changer you've been waiting for. Let's dive into what this means for your architecture and why it’s such a pivotal enhancement.

---

## The Big Picture: Breaking the 16 TiB Barrier

Azure NetApp Files (ANF) has long been the gold standard for high-performance, low-latency file storage in Azure. It is the go-to service for demanding workloads like SAP, Oracle, and High-Performance Computing (HPC).

Previously, regular ANF volumes (which scale up to 100 TiB) had a per-file size limit of 16 TiB. While sufficient for many scenarios, enterprise data is growing at an unprecedented rate. Moving a massive 40 TiB or 60 TiB virtual disk to the cloud often meant architects had to split disks or completely redesign storage layouts. This added complexity, time, and significant risk to migration projects.

With this new GA release, **individual file sizes can now reach up to 64 TiB**.

This capability is **available right now in all Azure NetApp Files enabled regions** and is fully supported across all service levels: **Flexible, Standard, Premium, and Ultra**.

### The Lead Use Case: Azure VMware Solution (AVS)

The absolute hero scenario for this capability is the **Azure VMware Solution (AVS)**. With 64 TiB file support, you can now take your largest, most demanding on-premises VMware virtual disks (VMDKs) and migrate them "as-is" directly into AVS datastores backed by Azure NetApp Files.

But it doesn't stop at AVS. This enhancement is a massive win for:

* **HPC, EDA, and Oil & Gas/Seismic workloads** that rely on single-file datasets larger than 16 TiB.
* **Analytics and Machine Learning (ML) scenarios** that need to operate directly on gargantuan data repositories without partitioning.

---

## How Does It Work? (Spoiler: It’s Effortless)

As an architect, my favorite kind of update is the one that requires zero heavy lifting from the IT team. Large file support in Azure NetApp Files is designed to be completely frictionless:

* **No Special Configuration:** This isn't a premium add-on or a hidden toggle. New and existing regular ANF volumes automatically support files up to 64 TiB. No feature flags to flip, no settings to change, and absolutely no need to re-provision your storage.
* **Volume Size Unchanged:** This update increases the per-file limit. The maximum volume size for regular ANF volumes remains a robust 100 TiB.
* **Full Feature Compatibility:** All the enterprise-grade capabilities you rely on—snapshots, backups, cross-region replication, and extreme performance throughput—continue to work seamlessly with these massive files.
* **Completely Transparent:** Applications won't know the difference. If you copy a 50 TiB database file or VMDK onto an ANF volume, it just works. No application-side tuning or architectural changes are required.

---

## The Value Prop: Why You Should Care

Scaling up our file size limit isn't just a technical flex; it translates directly into tangible benefits across the board.

### Technical Benefits

* **Direct AVS VMDK Support:** Run up to 64 TiB VMDKs natively. Eliminate disk splitting, cut file sprawl by up to 4x, and simplify your enterprise storage operations with zero application changes.
* **Cleaner Storage Designs:** Fewer, larger files drastically reduce fragmentation. This simplifies your volume layout, storage planning, and ongoing daily administration.
* **No Feature Tradeoffs:** You get the massive scale without losing the safety nets. Snapshots and backups work just as flawlessly on a 64 TiB file as they do on a 64 GiB file.

### Business Benefits

* **Accelerated Cloud Migrations:** Time is money. By migrating massive datasets "as-is," you avoid costly, month-long re-engineering projects and drastically speed up your time-to-cloud.
* **Broader AVS Adoption:** Remove the hesitation. Organizations can now confidently move their entire VMware estate—including their absolute largest, mission-critical VMs—into Azure VMware Solution.
* **Futureproofing:** Data isn't shrinking. This update ensures your Azure environment can scale seamlessly with your future data demands, freeing up your business to pursue new, ambitious projects without storage bottlenecks.

### Economic Benefits

* **Consolidate and Save:** Storing a 60 TiB dataset on a single volume avoids the overhead and stranded capacity of splitting data across multiple volumes. You can cut migration and management costs by up to 30%.
* **Lower OpEx:** Consolidation means less administrative effort. Fewer volumes and files mean less troubleshooting, easier management, and a happier IT operations team.
* **Optimized Resource Utilization:** Maximize the utility of every volume without artificial per-file limits. You avoid overprovisioning and ensure you are only paying for the storage you truly need.

---

## Summary

The jump from 16 TiB to 64 TiB per file on regular volumes is a monumental step forward for enterprise storage in Azure. It removes the friction from large-scale migrations, de-risks VMware estate moves, and allows architects to design cleaner, more cost-effective solutions.

Are you ready to move your heaviest workloads to the cloud? Azure NetApp Files is ready to handle them.

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new){:target="_blank"} page.

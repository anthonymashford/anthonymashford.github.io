---
layout*: post
date: 2025-08-29 12:00
title: Azure NetApp Files - August 2025 Preview Highlights
subtitle: The latest updates from Microsoft for Azure NetApp Files
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Backup, Replication]
author: Anthony Mashford
---

This month Microsoft introduced powerful capabilities to improve performance, efficiency, and data management flexibility. 

Here are some of the latest enhancements and preview features released for Azure NetApp Files.

---

## 🚀 Short-term Clones (Preview)

Short-term clones in Azure NetApp Files provide **space-efficient, instant read/write access** to data by creating temporary thin clones from existing volume snapshots. This eliminates the need for full data copies and enables significant capacity savings.

**Key Benefits:**

- Ideal for software development, analytics, disaster recovery, and testing
- Supports large datasets with quick refreshes from the latest snapshots
- Temporary and space-efficient for up to one month
- Consumes capacity only for incremental changes

This feature accelerates development and analytics workflows, improves quality and resilience, and reduces costs. Now available in preview across all Azure NetApp Files supported regions.

---

## 💾 Backup Support for Large Volumes (Preview)

Azure NetApp Files backup now supports **large volumes** by moving point-in-time snapshot copy data to **low-cost Azure storage**. This addresses long-term retention, data protection, and compliance needs.

**Highlights:**

- Efficient data protection for initial and incremental backups
- Registration required to use this feature

---

## 🔄 Restore Individual Files Using Single-File Restore (Preview)

With **single-file restore from backup**, you can now restore individual files from the Azure NetApp Files backup vault without restoring the entire volume.

**Advantages:**

- Save time and cost by restoring only necessary files
- Now available in preview

---

## 📊 File Access Logs – Now Generally Available (GA)

File access logs provide **enterprise-grade visibility** into file-level operations across SMB3, NFSv4.1, and dual-protocol volumes.

**Use Cases:**

- Monitor access patterns
- Detect unauthorised activity
- Support compliance investigations
- Optimise data usage

This feature enhances security and operational insight, aligning with the Well-Architected Framework's best practices.

---

## ❄️ Flexible Service Level with Cool Access (Preview)

Azure NetApp Files now supports **cool access** for the Flexible service level, enabling automatic tiering of infrequently accessed data to **lower-cost Azure storage**.

**Benefits:**

- Optimise performance and cost
- Decouple storage capacity and throughput
- Maintain configured throughput levels with cool access

**Note:** Registration is required for the Flexible service level before enrolling in this preview.

---

## Summary

The August 2025 updates to Azure NetApp Files deliver powerful enhancements that improve operational efficiency, reduce costs, and increase flexibility. From instant cloning and scalable backups to granular file restores and intelligent data tiering, these features empower organisations to streamline workflows, enhance data protection, and optimise resource usage. By adopting these innovations, enterprises can achieve greater agility and resilience in their cloud storage strategies.

I hope this short blog post about using Terraform to configure Azure NetApp Files Business Continuity and Disaster Recovery features using IaC method with Terraform has been useful.

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://github.com/anthonymashford/ANF-BCDR-Terraform/tree/main){:target="_blank"} page.

Stay tuned for more updates and enhancements in Azure NetApp Files!

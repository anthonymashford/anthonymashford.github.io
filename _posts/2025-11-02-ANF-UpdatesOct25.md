---
layout*: post
date: 2025-11-02 12:00
title: Azure NetApp Files – What’s New in October 2025
subtitle: Enhancing data protection, integration, and intelligence — October brings new power to Azure NetApp Files.
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Backup, Replication]
author: Anthony Mashford
---

# Azure NetApp Files – What’s New in October 2025

The Azure NetApp Files team continues to deliver innovation at pace, bringing new capabilities that simplify data protection, enhance interoperability, and expand identity integration. This month’s updates include two major features now generally available and two exciting previews that extend Azure NetApp Files into new realms of flexibility and integration.

---

## 🔹 Single File Restore from Backup – Now Generally Available

Restoring individual files from Azure NetApp Files backups is now **generally available (GA)**.

With **Single File Restore**, you can recover specific files directly from the Azure NetApp Files backup vault without needing to restore an entire volume. This means faster recovery times, reduced storage consumption, and lower costs when addressing accidental deletions or file-level corruption.

By restoring only what’s necessary, you save both time and money — a simple but powerful enhancement for operational efficiency and business continuity.

### 💡 Why this matters

File-level recovery is one of the most requested data protection capabilities. Instead of restoring large datasets or entire volumes, IT teams can now surgically restore only what’s required — minimising downtime, avoiding unnecessary data transfer, and accelerating recovery for end users. This feature brings Azure NetApp Files backup in line with enterprise-grade expectations for precision and speed.

---

## 🔹 Backup Support for Large Volumes – Now Generally Available

Backup for **large Azure NetApp Files volumes** has also reached **GA**.

This capability moves point-in-time snapshot data to **low-cost Azure storage**, addressing **long-term retention, data protection, and compliance** requirements.

Using an **efficient, high-speed data mover**, Azure NetApp Files backup ensures both initial and incremental backups complete quickly — even for the largest datasets.

> To use backup on large volumes, make sure to register the **large volumes AFEC**.

With this GA release, enterprises managing high-capacity workloads can confidently protect their data at scale while optimising cost and performance.

### 💡 Why this matters

As enterprise data grows, so do the challenges of backup duration, cost, and compliance. Large volume backup support ensures that even multi-terabyte workloads can be protected efficiently and economically. This enhancement gives organisations confidence to use Azure NetApp Files for their most demanding data estates — without compromise on protection or performance.

---

## 🧩 Object REST API (Preview)

The new **Object REST API** (an **S3-compatible REST API**) for Azure NetApp Files introduces an exciting bridge between **file-based storage** and **modern cloud-native services**.

This API enables seamless integration of Azure NetApp Files data with **Microsoft Fabric**, **Azure AI**, and other Azure offerings — all **without needing to move or replicate data**.

Key benefits include:

* **Native S3-compatible read/write access** for modern application integration
* **Real-time analytics and AI workloads** directly on existing datasets
* **Enhanced security** with data remaining protected in Azure NetApp Files
* **Cost efficiency** by eliminating redundant copies and data movement

This preview opens the door to new use cases in **machine learning, advanced analytics**, and **real-time business intelligence** — accelerating innovation while maintaining enterprise-grade data governance.

### 💡 Why this matters

The Object REST API transforms Azure NetApp Files from a high-performance storage platform into a data hub for modern analytics and AI. By enabling direct S3-compatible access, organisations can unlock new insights from existing datasets without duplicating or migrating data. This is a major step forward for customers looking to integrate storage with Microsoft Fabric, AI, and data intelligence solutions — all while maintaining performance, governance, and security.

---

## 🔐 Support for FreeIPA, OpenLDAP, and Red Hat Directory Server (Preview)

Azure NetApp Files now supports **FreeIPA**, **OpenLDAP**, and **Red Hat Directory Server**, extending integration options for **LDAP-based identity services**.

This preview allows organisations to leverage their **existing directory infrastructure** for authentication and access control — simplifying identity management and enhancing compliance.

Ideal for **Linux-based workloads**, **HPC environments**, and **hybrid deployments**, this feature provides:

* **Seamless identity integration** across enterprise environments
* **Improved scalability and security** through native LDAP support
* **Consistent access control** across platforms and workloads

Available in all Azure NetApp Files supported regions, this capability streamlines deployment and ensures compatibility with the most common enterprise directory services.

### 💡 Why this matters

Enterprise IT environments are rarely homogeneous. Many rely on multiple directory services to manage Linux and hybrid workloads. By supporting FreeIPA, OpenLDAP, and Red Hat Directory Server, Azure NetApp Files aligns more closely with real-world enterprise identity ecosystems. This improves flexibility, reduces integration friction, and strengthens security by using familiar, proven directory tools.

---

## 🚀 Summary

October’s updates demonstrate the Azure NetApp Files team’s ongoing commitment to innovation across data protection, integration, and identity.

* **Single File Restore** and **Large Volume Backup** (GA) deliver tangible operational and cost benefits.
* **Object REST API** and **LDAP integration** (preview) pave the way for new data-driven and hybrid identity scenarios.

Azure NetApp Files continues to evolve as the enterprise-grade, Azure-native storage platform that simplifies performance, protection, and scalability — all while unlocking new possibilities across the Microsoft cloud ecosystem.

I hope this short blog post about using Terraform to configure Azure NetApp Files Business Continuity and Disaster Recovery features using IaC method with Terraform has been useful.

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://github.com/anthonymashford/ANF-BCDR-Terraform/tree/main){:target="_blank"} page.

---

## 📚 Learn More

* [What’s new in Azure NetApp Files – Microsoft Learn](https://learn.microsoft.com/azure/azure-netapp-files/whats-new)
* [Azure NetApp Files documentation](https://learn.microsoft.com/azure/azure-netapp-files/)
* [Introduction to Azure NetApp Files – Microsoft Learn module](https://learn.microsoft.com/training/modules/intro-to-azure-netapp-files/)
* Previous monthly updates:

  * [Azure NetApp Files – September 2025 Updates](#)
  * [Azure NetApp Files – August 2025 Updates](#)

---


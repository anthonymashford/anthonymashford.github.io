---
layout*: post
date: 2024-12-08 12:00
title: Updated Plug-in for AVS Cloud Backup for VMs
subtitle: Enhancing Data Protection for Virtual Machines on Azure NetApp Files NFS Datastores (Preview)
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/avs-anf.png
share-img: /assets/img/avs-anf.png
tags: [Blog, Azure, Azure NetApp Files]
author: Anthony Mashford
---

## Introduction
In the ever-evolving world of cloud computing, safeguarding data remains a top priority for organisations. As businesses increasingly rely on cloud infrastructure, ensuring the security and recoverability of data is paramount. We are thrilled to announce the preview of enhanced Cloud Backup capabilities for Virtual Machines on Azure NetApp Files Datastores integrated with Azure VMware Solution.

### Seamless Integration with Azure NetApp Files Backup
One of the standout features of this enhancement is the seamless integration with Azure NetApp Files Backup. This integration offers a fully managed backup solution that caters to long-term recovery, archiving, and compliance needs. Users can now mount a datastore from a snapshot or an Azure NetApp Files Backup to restore files effortlessly. The flexibility to mount these backups to either the original Azure VMware Solution host or an alternate host further enhances your data recovery options.

### Advanced VMDK Attachment Capabilities
With the new enhancements, Cloud Backup for Virtual Machines introduces the ability to attach one or more Virtual Machine Disk (VMDK) files from a backup. These VMDKs can be attached to the parent VM, an alternate VM on the same Azure VMware Solution host, or even to an alternate VM on a different host managed by the same vCenter instance. This feature provides increased flexibility and efficiency in managing and recovering VM data, ensuring that your critical data is always accessible.

### Versatile Restoration Options
Versatility in data restoration is another key highlight of these new capabilities. Users can now restore a virtual machine to an alternate location on the same Azure VMware Solution host or to a different host managed by the same vCenter instance. Additionally, this solution supports the restoration of individual guest files and folders from a snapshot or an Azure NetApp Files Backup, allowing for granular recovery. This means you can recover specific data without needing to restore entire virtual machines, saving both time and resources.

### Summary
The enhanced Cloud Backup capabilities for Virtual Machines on Azure NetApp Files Datastores represent a significant advancement in data protection. By integrating with Azure NetApp Files Backup and introducing advanced features like VMDK attachment and versatile restoration options, we are committed to providing robust, flexible, and user-friendly solutions to meet the evolving needs of our customers.

I encourage you to explore these new capabilities and see how they can enhance your data protection strategy. 

For more information, check out this link about [Cloud Backup for Virtual Machines](https://learn.microsoft.com/en-us/azure/azure-vmware/install-cloud-backup-virtual-machines){:target="_blank"}

Additionally, check out [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new){:target="_blank"} for more updates on the service.

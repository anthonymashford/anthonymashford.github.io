---
layout*: post
date: 2024-05-28 13:00
title: Azure NetApp Files support for multiple Active Directory connections (Preview)
subtitle: Expanding Azure NetApp Files data protection capabilities
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/multi-ad-thumb.png
share-img: /assets/img/multi-ad-thumb.png
tags: [Blog, Azure, Azure NetApp Files]
author: Anthony Mashford
---

## Azure NetApp Files - Support for Multiple Active Directory connections (Public Preview)

### Introduction

Originally, Azure NetApp Files (ANF) only supported one Active Directory (AD) connection per subscription per region. This meant that customers with multiple AD environments wanting to consume ANF would need multiple subscriptions, not ideal!

### Overview

The new feature in Azure NetApp Files now supports a unique Active Directory (AD) connection for each NetApp account. This allows every NetApp account to establish its own connection to an AD Forest and Domain, enabling the management of multiple AD connections within a single region under one subscription. This improvement provides each NetApp account with its own AD connection, promoting operational isolation and accommodating specialised hosting scenarios. It's possible to set up AD connections multiple times across various NetApp accounts. The creation of SMB volumes in Azure NetApp Files is linked to AD connections in the NetApp account, making the management of AD environments more scalable, efficient, and streamlined. Please note that this feature is currently in preview.

This feature allows every NetApp account in an Azure subscription to establish its own AD connection. After setting up, the AD connection linked to the NetApp account is utilized when an SMB volume, a NFSv4.1 Kerberos volume, or a dual-protocol volume is created. In other words, Azure NetApp Files can accommodate multiple AD connections within a single Azure subscription, provided that multiple NetApp accounts are in use.

**Note:** Each Active Directory configuration is linked to the parent ANF account.

### Register the Resource Provider

The multi-AD feature is currently in **Public Preview**. In order to use it, you need to register the resource provider. Once enabled, the feature works in the backend.

1. Registering the Multi-AD feature.
   
~~~
Register-AzProviderFeature -ProviderNamespace Microsoft.NetApp -FeatureName ANFMultipleActiveDirectory
~~~

2. Checking to status of the feature registration. **Note:** This can take up to 60 minutes to complete.

~~~
Get-AzProviderFeature -ProviderNamespace Microsoft.NetApp -FeatureName ANFMultipleActiveDirectory
~~~

### Checking ANF AD configuration
To check the AD configuration, once you have created a new ANF account, under overview, checked the **AD Type** setting, see image below.

![](/assets/img/anf-multi-ad.png)

### Summary

In summary, Microsoft continue to evolve the  Azure NetApp Files service, this additional capability continues this evolution. The ability to to have multiple AD configurations within a single subscription allows customers to be more flexible and granular with their deployments of Azure NetApp Files.

To allow you to keep up to date with service enhancements and feature updates, checkout the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new) page for more updates on the service.







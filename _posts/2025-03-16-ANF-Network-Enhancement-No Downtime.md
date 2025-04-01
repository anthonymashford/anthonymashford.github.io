---
layout*: post
date: 2025-03-16 16:00
title: Update to Azure NetApp Files networking
subtitle: Edit networking feature with no downtime now GA
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf-announcement.png
share-img: /assets/img/anf-announcement.png
tags: [Blog, Azure, Azure NetApp Files, Networking]
author: Anthony Mashford
---

## Azure NetApp Files: Seamless Network Feature Upgrades with Zero Downtime

Azure NetApp Files continues to redefine the cloud storage experience with its latest enhancement: the ability to edit network features—specifically upgrading from Basic to Standard network features—without any downtime. This groundbreaking feature is now generally available across all Azure NetApp Files regions, offering users a seamless and consistent virtual networking experience.

## What Are Standard Network Features?

Standard Network Features provide a robust virtual networking framework that enhances the security posture and connectivity options for Azure NetApp Files volumes. These features include:

- **Higher IP limits**: Ideal for workloads requiring extensive IP address allocations.
- **Support for Network Security Groups (NSGs)**: Enabling fine-grained control over inbound and outbound traffic.
- **User-Defined Routes (UDRs)**: Allowing customized routing for specific network traffic patterns.
- **Enhanced Connectivity**: Includes support for VNet Global Peering

## The No-Downtime Advantage

Previously, upgrading network features for existing Azure NetApp Files volumes could result in service interruptions. With this new capability, users can now transition from Basic to Standard network features without impacting their workloads. This is particularly beneficial for businesses that require high availability and cannot afford downtime during network configuration changes.

## How It Works

The process to edit network features is straightforward and can be done via the Azure portal, CLI, or API. Users can modify the network features of existing volumes to leverage the advanced capabilities of Standard Network Features. This flexibility ensures that organizations can adapt their storage solutions to evolving business needs without disruption.

## Availability

This feature is available in all Azure NetApp Files regions, making it accessible to a global audience. Whether you're running workloads in North America, Europe, Asia, or any other supported region, you can take advantage of this enhancement to optimize your cloud storage experience.

For more detailed guidance on configuring network features, you can refer to the official documentation [here](https://learn.microsoft.com/en-us/azure/azure-netapp-files/configure-network-features){:target="_blank"}.

## Summary

This update underscores Azure NetApp Files' commitment to providing innovative and user-centric solutions. By enabling no-downtime upgrades for network features, Microsoft empowers businesses to achieve greater agility and resilience in their cloud operations.

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new){:target="_blank"} page.

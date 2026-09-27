---
layout*: post
date: 2026-09-27 12:00
title: ANF & Entra Kerberos...The DNS Puzzle
subtitle: Making Microsoft Entra Kerberos Authentication Work with Azure NetApp Files
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, W365 Cloud PC, Zone Redundant]
author: Anthony Mashford
---

## Introduction

In my previous blog, [Who Let the Dog Out?](https://www.azuretechlab.com/2026-09-24-ANF-Entra-Auth/) - I covered the exciting announcement of the **Public Preview of Microsoft Entra Kerberos authentication for Azure NetApp Files SMB volumes**. This new capability removes one of the long-standing dependencies in Azure NetApp Files SMB environments: the requirement for clients to have line-of-sight connectivity to Active Directory Domain Services (AD DS) domain controllers for authentication. Instead, Microsoft Entra ID can issue Kerberos tickets directly for SMB access, enabling a much more cloud-native identity model.

That's fantastic news for anyone trying to modernise their file services architecture.

But, as with most things in IT, solving one problem often reveals the next challenge.

## Authentication Solved. Now What About DNS?

While Microsoft Entra Kerberos authentication removes the dependency on AD DS for authentication, clients still need to **find the Azure NetApp Files SMB endpoint**.

After all, Kerberos tickets are great, but they don't help much if your clients can't resolve the name of the file share they're trying to access.

This is where DNS becomes an important design consideration.

There are several ways to handle name resolution depending on your environment:

- Traditional Active Directory integrated DNS
- Standalone DNS servers running in Azure
- DNS forwarding solutions
- Azure DNS Private Resolver
- Azure Private DNS Zones
- Other hybrid DNS architectures

Each has its place depending on your environment, operational model, and existing investments.

For this blog, however, I'm focusing on a specific scenario that I've been testing in my lab.

## My Test Environment

My lab is deliberately designed to be as cloud-native as possible.

The environment consists of:

- Azure Virtual Desktop (AVD)
- Session Hosts that are **Microsoft Entra joined**
- **No** Active Directory domain join
- Azure NetApp Files **SMB** volumes using Microsoft **Entra Kerberos** authentication
- Azure Private DNS Zones for **name resolution**

In other words, there's **not a domain controller in sight.**

If you've spent years deploying file services, that sentence may feel slightly unsettling. Don't worry, I felt the same way the first time.

It's a bit like turning up to a barbecue and discovering somebody has successfully cooked without using fire.

Apparently, it can be done.

## Why Azure Private DNS Zones?

For this lab, Azure Private DNS Zones provided a straightforward way to ensure that my AVD session hosts could resolve the names required to connect to Azure NetApp Files SMB shares.

By creating the appropriate DNS records within a private DNS zone and linking it to the virtual networks containing my AVD hosts, name resolution remains entirely within Azure.

The result is:

- No dependency on traditional Active Directory DNS
- No requirement to deploy additional DNS virtual machines
- Simplified architecture
- A fully cloud-native approach to SMB access

When a user signs into their AVD session host, Microsoft Entra ID provides the Kerberos authentication path, while Azure Private DNS ensures the SMB endpoint can be found.

One handles identity.

The other handles location.

Think of it as the cloud equivalent of knowing both *who* you are and *where you're going*.

Both are surprisingly useful.

## Important Considerations

Before everyone starts deleting domain controllers and celebrating with a victory coffee, it's important to remember that this capability remains in **Public Preview**.

Preview features are designed for evaluation and testing. Functionality, requirements, and supported scenarios may change before general availability.

Microsoft has also made it clear that Microsoft Entra Kerberos authentication for Azure NetApp Files currently applies to **SMB volumes only**. NFS, NFSv4.1 Kerberos, and dual-protocol volumes continue to rely on Active Directory Domain Services (AD DS) for authentication and authorisation.

As always, make sure you review the latest Microsoft documentation and validate any design before considering production deployment.

## The Bigger Picture

What excites me most about this preview isn't just the removal of a technical dependency.

It's what it represents.

For years, cloud architects have been working towards reducing infrastructure overhead and eliminating components that exist solely to support legacy requirements.

Microsoft Entra Kerberos authentication for Azure NetApp Files moves SMB workloads further down that journey.

We're now seeing a future where:

- Authentication is cloud-native
- Devices are Microsoft Entra joined
- Virtual desktops can operate without traditional domain joins
- File services remain SMB-based
- DNS can be delivered through Azure-native services

That's a compelling direction for organisations looking to modernise their environments without abandoning the applications and workflows that still rely on SMB file shares.

## Final Thoughts

My Azure Virtual Desktop lab proved that Microsoft Entra joined session hosts can successfully access Azure NetApp Files SMB shares using Microsoft Entra Kerberos authentication, while Azure Private DNS Zones provide the name resolution needed to make the whole experience work smoothly.

Is Azure Private DNS the only answer?

Absolutely not.

There are multiple approaches to solving the DNS challenge, and the right choice will depend on your organisation's architecture, operational model, and existing investments.

For my lab, though, it provided a clean, simple, and highly cloud-native solution.

Just remember: **this feature is still in Public Preview**. Test thoroughly, expect changes, and please don't rip out your existing infrastructure on a Friday afternoon.

Trust me on that last one.

## Over to You

I'd love to hear how others are approaching this challenge.

Are you using Azure Private DNS Zones, Azure DNS Private Resolver, traditional DNS servers, or something completely different?

How are you handling name resolution for Microsoft Entra Kerberos-enabled Azure NetApp Files deployments?

Share your experiences, lessons learned, and any creative solutions you've discovered in the comments. The more we learn from each other, the easier it becomes for everyone to adopt this exciting new capability and move one step closer to a truly cloud-native SMB experience.

For more information on this feature, check out [Understand Microsoft Entra Kerberos with Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/understand-entra-id){:target="_blank"} page.

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new){:target="_blank"} page.

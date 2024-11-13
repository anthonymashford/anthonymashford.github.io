---
layout*: post
date: 2024-08-11 18:00
title: Default Outbound Access for VM's is Being Retired
subtitle: Microsoft will soon be removing default outbound access to the internet for VM's
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/gateway.png
share-img: /assets/img/gateway.png
tags: [Blog, Azure, Azure Networking, NAT Gateway]
author: Anthony Mashford
---

## Introduction

It's not talked about much at the moment, but this rather major will impact every Azure customer and might trip up some people.

Since the beginning, outbound access has been enabled default. On September 30th, 2025, default outbound access for newly deployed services will be retired, across all Azure regions. More details on that announcement [here](https://azure.microsoft.com/en-gb/updates/default-outbound-access-for-vms-in-azure-will-be-retired-transition-to-a-new-method-of-internet-access/){:target="_blank"}

This means that newly created VM's after this date will not be able to connect to the internet by default. Existing VM's will continue to connect, but it is recommended that you implement an explicit method to deliver outbound access. 

Now, whilst this is a bit of a pain to deal with, its not a bad thing. Once transitioned, it will give you greater control over how VM's talk to the internet.

So, what options do you have...

## What You Need to Know

Azure is set to retire its default outbound access for virtual machines (VMs) on September 30, 2025. This change aims to enhance security and provide more control over internet connectivity for Azure resources.

## What is Default Outbound Access?
In Azure, VMs created in a virtual network without explicit outbound connectivity are assigned a default outbound public IP address. This IP address enables outbound internet connectivity but is subject to change and is not recommended for production workloads.

## Why the Change?
The move to retire default outbound access is part of Azure's transition to a **secure-by-default model**. By requiring explicit outbound connectivity methods, Azure aims to:

- **Provide greater control** over how and when VMs connect to the internet.

- **Ensure traceable IP resources** that you own, beneficial for measurement and troubleshooting.

- **Prevent disruptions** caused by public IP address changes.

## Transitioning to Explicit Outbound Connectivity
To ensure a smooth transition, Microsoft recommends using explicit outbound connectivity methods such as:

- Azure NAT Gateway

- Azure Load Balancer outbound rules

- Directly attached Azure public IP addresses

These methods offer more predictable and secure internet access for your VMs.

## Next Steps
If you have existing VMs using default outbound access, they will continue to work after the retirement date. However, it is strongly recommended to transition to an explicit method to avoid potential disruptions.

For more detailed information, you can refer to the Microsoft Learn article [here](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/default-outbound-access){:target="_blank"}
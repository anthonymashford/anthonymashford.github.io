---
layout*: post
date: 2026-09-24 12:00
title: Who Let the Dog Out?
subtitle: WMicrosoft Entra Kerberos for Azure NetApp Files Is in Public Preview
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/entra.svg
share-img: /assets/img/entra.svg
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, Monitoring, Zone Redundant]
author: Anthony Mashford
---

## Introduction

Every SMB design conversation I've had in the last few years has a moment. We've agreed the service level, the capacity pool is sized, snapshots and replication are sorted, and everyone's feeling good. Then someone asks:

> "So… where are the domain controllers going to live?"

Cue the sigh. The answer has always been "somewhere Azure NetApp Files can reach them". In practice that means extending AD DS into Azure, or building network line-of-sight back to on-premises domain controllers, and then patching, monitoring and babysitting all of it. You end up running an identity estate just so a file share can check who's knocking.

As of today, there's another option. **Microsoft Entra Kerberos authentication for Azure NetApp Files SMB volumes is now in public preview.**

## The short version

Azure NetApp Files now supports Microsoft Entra Kerberos authentication for SMB volumes. Users can access them with either of two identity types:

- **Hybrid identities**: user accounts that originate in on-premises AD DS and are synchronised to Microsoft Entra ID, giving one identity for both on-premises and cloud resources.
- **Cloud-only identities**: user accounts created and managed exclusively in Microsoft Entra ID, with no on-premises presence at all.

The headline is that you **don't need to extend on-premises AD DS into Azure**, and you **don't need line-of-sight network connectivity between Azure NetApp Files and on-premises domain controllers**.

One authentication approach, two identity models, no domain controllers in the SMB authentication path.

## Why this matters

Plenty of organisations have moved most of their estate onto Microsoft Entra ID. Then SMB file shares turn up and drag them back to a hard dependency on AD DS, with domain controllers to deploy, maintain and keep reachable.

Entra Kerberos removes that dependency for SMB:

- **No domain controllers to deploy or maintain** just to authenticate SMB access.
- **Simpler deployment** for cloud-centric environments.
- **Support for both identity types** over SMB, so you can serve the hybrid estate you have and the cloud-only estate you're moving towards.

It's particularly relevant if you're a cloud-first organisation that has never had on-prem AD, or a hybrid enterprise that would rather not stretch AD DS into every Azure region it lands in.

## How it works

Two things have to happen, and it helps to keep them separate.

### 1. The user gets a Kerberos ticket from Entra ID

Microsoft Entra ID issues Kerberos tickets to authenticated users when they sign in. The client presents those tickets when it accesses an Azure NetApp Files SMB volume. The volume validates the Entra-issued ticket, and there's no domain controller in the access path.

### 2. Azure NetApp Files authenticates to Entra ID and Microsoft Graph

For ANF to take part, it needs its own trusted identity in your tenant. That uses a certificate-based flow:

1. You create an **app registration** in Microsoft Entra ID. Using the REST or Microsoft Graph APIs requires OAuth 2.0 tokens, so the application is configured with the required permissions and a **certificate**.
2. The certificate's private key is stored in **Azure Key Vault**.
3. Azure NetApp Files retrieves the private key from Key Vault and uses it to **sign a client assertion** (a signed JSON Web Token).
4. ANF exchanges that assertion for an **access token** from Microsoft Entra ID.
5. ANF uses the token to authenticate against the **Microsoft Graph** endpoint.

In other words, the certificate proves who ANF is, and Entra ID issues it a token to use against Graph. No passwords are sitting in a config file.

```text
 ┌───────────────────────┐   sign-in          ┌──────────────────────┐
 │ Client                │ ─────────────────▶ │  Microsoft Entra ID  │
 │ (hybrid or cloud-only │ ◀───────────────── │                      │
 │  identity)            │   Kerberos ticket  └──────────▲───────────┘
 └──────────┬────────────┘                               │ signed JWT → access token
            │ SMB + Entra-issued ticket                   │
            ▼                                            │
 ┌───────────────────────┐   private key      ┌──────────┴───────────┐
 │ Azure NetApp Files    │ ◀───────────────── │  Azure Key Vault     │
 │ SMB volume            │                    └──────────────────────┘
 │                       │ ── HTTPS 443 via NAT gateway ──▶ graph.microsoft.com
 └───────────────────────┘

          Domain controllers in this picture: 0
```

## The networking bit (don't skip this)

ANF needs **reliable outbound connectivity to Microsoft Graph** for the certificate-based authentication to work.

| Azure cloud            | Endpoint              | Port | Protocol      |
| ---------------------- | --------------------- | ---- | ------------- |
| Azure public (global)  | `graph.microsoft.com` | 443  | TCP (HTTPS)   |

To get there:

- **A NAT gateway is required** to route Azure NetApp Files traffic to Microsoft Entra ID. **Only the Standard SKU** is supported.
- **Create and associate the NAT gateway before you create SMB volumes** that use Entra Kerberos. If you do it the other way round, you'll be the one sighing.
- Your **NSGs, UDRs and firewalls** must allow that outbound HTTPS traffic.

One detail for anyone working outside the public cloud: Entra ID and Microsoft Graph **endpoints differ by Azure cloud environment**, so make sure whatever your NAT gateway can reach matches the cloud you're in. The documentation currently lists the Azure public (global) cloud.

## Anatomy of a Microsoft Entra ID connection

Everything is tied together by a **Microsoft Entra ID connection**, which needs five inputs:

| Component | What it's for |
| --- | --- |
| **Application ID** | The app registration configured with the required permissions and certificate, used to obtain OAuth 2.0 tokens |
| **Domain name** | The AD domain synchronised with Entra ID (hybrid) or your custom domain |
| **Azure Key Vault URI** | Where ANF fetches the certificate and private key it uses to get tokens |
| **Certificate name** | The certificate in Key Vault associated with your app registration |
| **SMB server prefix** | The prefix for the SMB server FQDN that clients use to mount the volume |

### Connection lifecycle

A connection object has a lifecycle state, which is handy when you're troubleshooting at 4:55 on a Friday:

- **Created**: the connection exists and is available for use.
- **In use**: the connection is associated with one or more SMB volumes.
- **Deleted**: the connection has been removed.
- **Error**: something's wrong and it needs attention.

A connection that's **In use** has live SMB volumes depending on it. Before you modify or delete one, work out which volumes depend on it.

## Hybrid vs cloud-only at a glance

| | Hybrid identity | Cloud-only identity |
| --- | --- | --- |
| **Where the user originates** | On-premises AD DS | Microsoft Entra ID |
| **On-premises presence** | Yes, synchronised to Entra ID | None |
| **AD DS extended into Azure?** | Not required | Not required |
| **Line-of-sight to on-prem DCs?** | Not required | Not required |
| **Typical fit** | Existing enterprises modernising | Cloud-native and greenfield organisations |

## When to use it, and when to stick with AD DS

**Reach for Entra Kerberos when:**

- You're managing cloud-only or hybrid identities.
- You want to avoid extending on-premises AD DS into Azure.
- You don't need line-of-sight connectivity to domain controllers.
- Your workload is SMB, and you don't need NFS, dual-protocol or NFSv4.1 Kerberos volumes.

**Stay on AD DS when:**

- You need **NFS, dual-protocol or NFSv4.1 Kerberos** volumes. These still require AD DS.
- Your access model depends on **on-premises domain controllers being contacted directly** for authentication.

This is an SMB feature, so the NFS and multiprotocol crowd will need to wait.

## Disaster recovery: plan it now

If you use cross-region replication, the destination region needs its own supporting pieces:

- A **NetApp account in both** the source and destination regions.
- The **Microsoft Entra ID configuration available in the destination region**, including reachability to the Azure Key Vault that holds the application's private key.
- A **NAT gateway and network configuration in the destination region** that allows outbound HTTPS to the right Microsoft Graph endpoint for that cloud.

A failover is a bad time to find out the DR region can't reach Graph.

## What does it cost?

**Nothing extra.** Using Microsoft Entra Kerberos authentication for hybrid and cloud-only identities doesn't inherently incur additional charges. You'll still pay for supporting components such as the NAT gateway and Key Vault, but that's small compared with running domain controller VMs around the clock just so a file share can check IDs.

See the [Azure NetApp Files pricing page](https://azure.microsoft.com/pricing/details/netapp/) and the [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/).

## The preview small print

⚠️ **There's no SLA during preview.** Preview features are provided under the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/): as-is, not covered by the SLA, and subject to change. Build it in a sandbox, test it thoroughly and send feedback. Keep your payroll share where it is for now.

Registration steps, client configuration and the current list of feature limitations are in the configuration guide linked below. Read it before you start, not halfway through.

## Summary

For years the price of admission for enterprise SMB on Azure NetApp Files has been a domain controller or two. Entra Kerberos authentication removes that requirement. You don't have to extend AD DS into Azure or keep line-of-sight to on-prem DCs, and it supports both the hybrid identities you already have and the cloud-only identities you're heading towards.

It's early. It's SMB-only, it needs a Standard NAT gateway and some Graph plumbing, and there's no SLA yet. But the direction is right, and it's been a long time coming.

Register the feature, try it out and tell us what breaks. The more people use the preview, the better it'll be at GA.

## Learn more

- [Understand Microsoft Entra Kerberos hybrid and cloud-only identities in Azure NetApp Files](https://learn.microsoft.com/azure/azure-netapp-files/understand-entra-id)
- [Configure Microsoft Entra Kerberos authentication for Azure NetApp Files](https://learn.microsoft.com/azure/azure-netapp-files/configure-entra-kerberos-authentication-for-hybrid-cloud-identities)
- [What's new in Azure NetApp Files](https://learn.microsoft.com/azure/azure-netapp-files/whats-new)
- [Azure NetApp Files pricing](https://azure.microsoft.com/pricing/details/netapp/)
- [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/)

---

*Questions, war stories or strong opinions about Kerberos? Find me on LinkedIn.*

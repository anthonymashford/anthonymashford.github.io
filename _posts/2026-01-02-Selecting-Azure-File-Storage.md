---
layout*: post
date: 2026-01-02 12:00
title: Selecting File Storage on Azure and Avoiding False Economies
subtitle: How to prevent under-scoped designs and unplanned spend driven by hidden requirements.
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Backup, Replication, AVD, FSLogix, User Profiles]
author: Anthony Mashford
---

## Introduction

Credit where it’s due: **Dan Soderholm** from Microsoft who recently shared a Warren Buffett quote with me, **"Price is what you pay, value is what you get"- Warren Buffett.** That line stuck with me, and it’s exactly what prompted this blog post.

If you have a some time free, then I think you'll find this blog article useful. However, If you're in a hurry, please jump the [TL;DR](#tldr) for a cutdown version.

Selecting a file storage service in Azure can look deceptively simple: compare the $/GiB headline rate, pick the cheapest option that meets today’s capacity requirement, and move on. In practice, that approach often creates a “false economy” — a design that appears cost-efficient on day one, but accumulates unplanned spend as soon as real-world requirements surface.

File storage is rarely just “some capacity.” It is performance (throughput and IOPS) under peak load, predictable latency for user experience and batch windows, and availability characteristics that align to business impact. It is also integration: identity and access controls, networking and security boundaries, backup and retention, ransomware protection, monitoring, and the operational effort to run it reliably at scale. Many of these elements don’t show up in a price-per-GB comparison, yet they directly influence what you must provision, how you must architect, and ultimately what you will pay.

To make this practical, we’ll use a familiar and demanding scenario as our reference architecture: **Azure Virtual Desktop (AVD)** with **FSLogix profile storage**. Profile containers are a classic example of where “cheap per GB” can become expensive quickly—because the user experience is sensitive to latency and metadata performance, concurrency spikes are normal at logon/logoff, and resilience and data protection requirements often increase once the service is in production. What looks like a straightforward file share decision can become a critical dependency that determines session host density, login times, operational overhead, and the real monthly bill.

The result is a pattern many architects and decision makers have seen before: a file service selected for its low unit price, followed by a rapid sequence of “small” additions — higher tiers to recover performance, extra capacity to hit throughput targets, more snapshots and backup to meet compliance, additional components to close security gaps, and workarounds to address protocol, access, or migration constraints. Individually, each change can look reasonable. Collectively, they can turn an apparently economical decision into an under-scoped design with an expanding cost envelope.

This blog is about avoiding that trap. We’ll walk through the hidden requirements that commonly drive spend in Azure file storage designs, how to surface them early, and how to evaluate options based on the **total solution cost** — not just the storage line item. The goal isn’t to steer you to the most expensive service; it’s to help you select the right service, sized correctly, with fewer surprises and a clearer cost model from the start. In the next section we will use a use case where net work file storage is required to suport and Azure Virtual Desktop deployment.

---

### TL;DR

* Picking Azure file storage based on **headline £/GiB** often creates a **false economy**: once real requirements show up (performance, latency, security, backup, operations), costs and complexity grow fast.
* File storage is not “just capacity” — you’re really buying **IOPS/throughput**, **predictable latency**, **availability**, **identity integration**, **secure networking**, and **data protection**.
* Use case: **AVD + FSLogix** for **4,000 users** (single persona). Profiles are **10 GiB max** each → **44,000 GiB (~43 TiB)** including 10% growth headroom.
* Performance envelope (ignoring concurrency for this exercise): **180,000 peak IOPS** and **60,000 steady-state IOPS**. Login performance and a latency-sensitive app make **consistent low latency** critical.
* Two options compared (UK South):

  * **Azure Files Premium (Provisioned v2)**: looks cheaper on paper but can introduce “hidden” costs and design overhead (e.g., **private endpoints + data processed charges**, sharding for performance, and tuning provisioned IOPS/throughput).
  * **Azure NetApp Files Standard**: **ANF Premium is unnecessary** because Standard can meet the required performance; cost is simpler (capacity-based) and the service is **VNet-injected / no public endpoint by design**.
* Bottom line: even if Azure Files totals slightly less in a calculator snapshot, **ANF delivers more value** for FSLogix (predictability, simplicity, security-by-design) once you factor in the real-world requirements—because the most expensive storage choice is the one you have to redesign after go-live.

---

## Use Case – AVD User Profile Storage

The customer is building a greenfield Azure Virtual Desktop (AVD) environment for **4,000 users** with a **single user persona**, using **FSLogix profile containers** to provide a consistent user experience across session hosts. In this architecture, the profile store is not an afterthought—it sits directly on the logon path, influences application start-up behaviour, and quickly becomes a productivity issue if it is under-sized or delivers inconsistent performance.

From a performance standpoint, FSLogix profile containers tend to generate lots of small, metadata-heavy operations with pronounced bursts at logon/logoff. Using the customer’s metrics, the design must cope with up to **45 IOPS per user at peak** and **15 IOPS per user at steady state**. For the purpse of this excercise will will not be factoring in concurrency. If you model those figures at the full user count, that equates to:

* **Peak IOPS:** 4,000 × 45 = **180,000 IOPS**
* **Steady-state IOPS:** 4,000 × 15 = **60,000 IOPS**

Not every user will hit peak IOPS simultaneously for long periods, but these figures are still valuable—they illustrate the order of magnitude and why selecting a service based on £/GiB alone can be a false economy. **Login times are explicitly important** to avoid delays and lost productivity, and the environment also includes a subset of users running a **latency-sensitive application**, which increases the need for *predictable* low-latency storage under contention, not just “enough capacity”.

### Capacity requirement (with 10% overhead)

The maximum user profile size is **10 GiB**. For **4,000 users**, the baseline profile capacity is:

* 4,000 × 10 GiB = **40,000 GiB**

Adding a **10% overhead** to account for growth:

* 40,000 GiB × 1.10 = **44,000 GiB**

Converted to TiB (using 1 TiB = 1,024 GiB):

* 44,000 / 1,024 = **42.97 TiB** (≈ **43 TiB**)

So, the FSLogix profile store should be planned for **44,000 GiB (~43 TiB)** of capacity, *before* factoring in any additional space for snapshots, backup retention, or DR copies—items that often become the “hidden” drivers of storage consumption and cost in real deployments.

We will use the estimated storage and IOPs required to calculate the cost of hosting user profiles.

---

### Weighing up the costs

For FSLogix profile containers, we’ll keep the options deliberately narrow and focus on two managed, production-grade services in UK South: Azure Files Premium (SSD) Provisioned v2 and Azure NetApp Files (ANF) – Standard service level.

Azure Files Premium Provisioned v2 is the most “native” fit for Windows profile storage: it’s SMB-based, fully managed, and aligns well with the operational model most AVD teams already understand (storage account + file share + permissions). The key design point is that Premium is a provisioned service: you pay for what you reserve, not just what you consume—so sizing for peak behaviour matters. That’s helpful when logon performance is non-negotiable, but it also means it’s easy to over-specify capacity (and therefore cost) just to hit performance headroom, especially if you later discover additional requirements such as snapshots, soft delete, backup, or higher resiliency options.

Azure NetApp Files (Standard) is the alternative when you want predictable performance and consistently low latency for mixed user activity (including those latency-sensitive application workflows you called out). ANF uses performance-aligned pricing and capacity pools, which tends to make “performance per pound” more transparent once you understand how the service is sized. In this scenario, ANF Premium is discounted because the required peak IOPS (~180,000) can be delivered using the Standard service level—avoiding the classic trap of paying for a higher tier when the workload doesn’t actually need it.

On cost, if we estimate profile capacity at 44,000 GiB (4,000 users × 10 GiB × 1.10 growth overhead), then illustrative list pricing for UK South comes out roughly as follows (capacity charges only, excluding data protection, data transfer, and any additional feature-related meters):

#### Azure Files Premium (Provisioned v2)

Azure Files Premium (SSD) Provisioned v2: **~$5624/month**

Private Endpoints (x8) including Bandwdith (22TiB/in 22TiB/out): **~$508/month**

> Total: **~$6132/month**

#### Azure NetApp Files

Azure NetApp Files Standard (Service Level): **~$6643/month**

> Total: **~$6643**

*Important reminder:* Even though the Azure Files storage offering comes out marginally cheaper than Azure NetApp Files, it is important to remember these points.

* There are no hidden costs with ANF, you simply pay for the amount of storage consumed.
* AF has hidden costs that can be extremely difficult to quantify.
* ANF has no public endpoint. It is secure by design.
* AF has public endpoints that need to be locked down.
* ANF injects straight into the VNet.
* It is recommend to use private endpoints with AF. These add to the cost.

> **Note:** Costs are estimated and accurate at the time of writing.

---

## Pros and cons in this FSLogix scenario

### Azure Files Premium (Pro v2)

#### AF Pros

* Native Azure Files model; familiar operational patterns.
* Separately provision capacity/IOPS/throughput (strong FinOps control when you tune it).
* More grannular scaling (1GiB increments)
  
#### AF Cons

* 180k peak IOPS drives sharding (multiple storage accounts/shares), increasing operational complexity.
* File Shares can be grown but you need to weight 24hrs before they can be shrunk back down,
* Private connectivity via Private Endpoints adds endpoint-hour and data-processed premiums.
* If you leave “recommended” throughput/IOPS defaults untouched, you can accidentally provision (and pay for) far more than needed.
* Challening to integrated with Active Directory.

### Azure NetApp Files Standard

#### ANF Pros

* Predictable performance model tied to service level and capacity; Standard can be sized to meet this workload’s envelope without moving to Premium.
* Can be flexed up or down withint he same service level.
* Low-latency, sub milli-second.
* Enterprise file service design that often performs well for latency-sensitive, interactive profiles.
* First-party Azure native storage offering.

#### ANF Cons

* 1 TiB provisioning increments can lead to small rounding overhead.
* Marginally more expensive than AFPv2.

---

## The decision logic that avoids false economies

If you want a simple, defensible decision narrative for decision-makers, use this:

1. Start with experience: login time and latency-sensitive apps make profile storage a productivity dependency, not a commodity.
2. Translate IO to architecture: 180k peak IOPS forces Azure Files sharding and pushes you to be explicit about throughput and private networking.
3. Model “security is not optional”: private endpoints and data processed charges are part of the design, not an afterthought.
4. Right-size tiers: ANF Premium is discounted here because Standard—properly sized—meets the throughput envelope implied by 180k IOPS at small IO sizes.
5. Validate with the Pricing Calculator: treat any blog estimate (including this one) as directional; the calculator is the authoritative view for UK South under your agreement.

---

### Summary

The storage line item is rarely what breaks the budget. The budget breaks when you select a file service using a single metric, then discover you also need more performance, more consistency, private connectivity, and more operational scaffolding to make it suitable for production.

For AVD + FSLogix at 4,000 users, the safer approach is to lead with experience and predictability, then make cost decisions with eyes open—because the most expensive option is the one you have to redesign after go-live.

So, yes I agree, if you look at ANF price per GiB, then it does seem expensive. However, once you take into account all the additional services and complexity of Azure Files for this workload, ANF actually gives you more features, more performance and most importantly, more value.

To quote Warren Buffett (again). **"Price is what you pay, value is what you get".**

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://github.com/anthonymashford/ANF-BCDR-Terraform/tree/main){:target="_blank"} page.

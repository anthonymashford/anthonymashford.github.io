---
layout*: post
date: 2026-10-11 12:00
title: Azure NetApp Files Large Volume Breakthrough Mode
subtitle: Six Endpoints, One Very Fast Volume
cover-img: /assets/img/breakthrough-mode-header.png
thumbnail-img: /assets/img/breakthrough-mode-thumbnail.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, W365 Cloud PC, Zone Redundant]
author: Anthony Mashford
---

## Introduction

*What it is, what it does, and where it fits (and where it doesn't).*

---

## TL;DR

- **What:** an Azure NetApp Files large volume with **six storage endpoints** on **dedicated capacity**.
- **Why:** to go beyond the 12,800 MiB/s large-volume ceiling. The Microsoft docs quote up to **50 GiB/s** and the GA announcement up to **80 GiB/s**.
- **Size:** **2,400 GiB to 2,400 TiB**, with roughly **15.9 billion files**.
- **For:** EDA, HPC, AI/ML pipelines and other parallel, metadata-heavy workloads.
- **Not for:** database data and log volumes, or converting an existing regular volume.
- **How:** register large volumes, request breakthrough mode, then select it at volume creation.

---

Some feature names are quietly descriptive. Others sound like something a stunt coordinator shouts before a car goes through a shop window.

**Breakthrough mode** is firmly in the second camp.

The good news is that, unusually, the name is earned. Breakthrough mode removes the throughput ceiling that applies to standard Azure NetApp Files large volumes. It does that with a sensible piece of architecture rather than anything magical. Let's walk through what it is, the numbers behind it, and the caveats worth knowing before anyone gets carried away.

---

## A quick recap: regular and large volumes

A **regular** Azure NetApp Files volume is a perfectly respectable bit of kit. It scales to 100 TiB, holds roughly 2.1 billion files, and tops out at 4,500 MiB/s on the Premium, Ultra and Flexible service levels. Most workloads are entirely happy here, including file shares, SAP HANA and VDI.

**Large volumes** raised the bar considerably. They range from 50 TiB to 1 PiB (more on request), use 64-bit file IDs to hold around **15.9 billion files**, and have a throughput ceiling of **12,800 MiB/s**. That makes them a good fit for EDA, AI datasets, media libraries and anything with an alarming number of files.

Some workloads, however, look at 12,800 MiB/s and see a limit to work around rather than a comfortable ceiling. Breakthrough mode is for them.

---

## So what is breakthrough mode?

In one sentence:

> **Breakthrough mode gives a single large volume six storage endpoints on dedicated capacity.**

Two ideas are doing the heavy lifting there.

### 🛣️ Six endpoints instead of one

A normal volume is reached through a single storage endpoint. Think of it as one good road into town. Smooth tarmac, well maintained, and completely gridlocked at 08:45 on a Monday.

Breakthrough mode gives you **six roads** into the same town. It's the same volume, the same namespace and the same data, but with six parallel data paths for clients to use. More paths means more concurrency, and concurrency is what HPC and EDA workloads depend on.

Microsoft describes this as delivering high-concurrency data paths while *simplifying* network management. That's a rare pairing in networking, where "more paths" and "simpler" don't usually turn up together.

### 🏠 Dedicated capacity, so no noisy neighbours

This is arguably the more important half. Breakthrough mode volumes live on storage systems **reserved for you**. They aren't shared, and they aren't "mostly yours".

In shared infrastructure there's always the chance that another workload has a busy afternoon and nudges your latency. With dedicated capacity there is no neighbour. The result is **predictable, isolated performance**, which is exactly what you want when a chip-design regression run has a deadline and a senior stakeholder attached to it.

---

## The numbers

"It's fast" is what people say right before something turns out not to be. So here are the specifics.

### Capacity

| | Regular volume | Large volume | **Breakthrough mode** |
| --- | --- | --- | --- |
| Minimum size | 50 GiB | 50 TiB | **2,400 GiB** |
| Maximum size | 100 TiB | 1 PiB (2 PiB on request) | **2,400 TiB (≈2 PiB)** |
| Max files | ~2.1 billion | ~15.9 billion | ~15.9 billion |

The minimum size is worth a second look. A standard large volume starts at 50 TiB, but breakthrough mode starts at **2,400 GiB**, roughly 2.3 TiB. You don't need to commit to a petabyte to get access to the extra performance.

### Throughput

Standard large volumes cap out at **12,800 MiB/s**. With breakthrough mode, that predefined ceiling goes away. Microsoft's requirements page quotes **up to 50 GiB/s**, depending on workload characteristics and placement. NetApp's general availability announcement on 24 September 2026 raised the headline figure to **up to 80 GiB/s per volume**.

That's per volume, not per account or per region.

### What the benchmarks showed

Microsoft ran FIO scale-out tests against a single breakthrough mode volume. The results came back at roughly:

- **~50,000 MiB/s** pure sequential reads
- **~21,200 MiB/s** pure sequential writes
- **~1.8 million** pure random read IOPS
- **~994,000** pure random write IOPS

Relative to the other volume types (baseline = a regular volume):

| Metric | Large volume vs regular | **Breakthrough vs regular** | **Breakthrough vs large volume** |
| --- | --- | --- | --- |
| Max read throughput | 2.7× | **14.4×** | **3.9×** |
| Max write throughput | 3.4× | **8.4×** | **2.5×** |
| Read IOPS | 1.8× | **4.8×** | **2.6×** |
| Write IOPS | 3.9× | **8.2×** | **2.1×** |

Reads scale more dramatically than writes. That's normal for storage, but it's worth bearing in mind when sizing a write-heavy workload, because the read figures make for a much better slide.

---

## Who is it for?

Not everyone, and that's fine. Breakthrough mode is built for workloads that are **massively parallel, metadata-heavy, or simply enormous**:

- **⚡ Electronic Design Automation (EDA):** thousands of client workstations working through billions of small files with heavy metadata operations. Microsoft's own testing singles this out as a particularly strong fit.
- **🧪 High-Performance Computing (HPC):** scale-out compute clusters that need to read a lot of data, very quickly, all at once.
- **🤖 AI/ML data pipelines:** feeding training and preprocessing jobs fast enough that expensive GPUs aren't sat idle waiting for data.
- **📁 Anything that has outgrown a large volume:** if you've hit 12,800 MiB/s and started splitting datasets across volumes just to get more throughput, this removes the need for that workaround.

---

## The caveats

Better to cover these now than to have them discovered halfway through a deployment.

- **🚪 You have to request it.** Breakthrough mode needs a request and a feature registration before first use. More on that below.
- **🗄️ It isn't for database data and log volumes.** Large volumes in general aren't currently recommended for HANA, Oracle, SQL Server and similar. If a database needs more than one volume's worth of throughput, the guidance is multiple regular volumes deployed with application volume groups.
- **🧩 No application volume groups.** You can't create a large volume through an application volume group.
- **🔄 No conversions.** A regular volume can't be converted into a large volume, so plan for it from day one.
- **🚚 No migration assistant.** The migration assistant isn't supported for breakthrough mode volumes, so plan data movement another way.
- **❄️ Cool access comes later.** You can enable cool access on a breakthrough mode volume, but only after the volume has been created.
- **📍 Regional availability depends on dedicated capacity.** At GA, it's available in regions where dedicated capacity has been ordered and provisioned for you. Talk to your Microsoft or NetApp account team early. Capacity planning rewards foresight and punishes improvisation.

On the plus side, it works across the **Flexible, Standard, Premium and Ultra** service levels, so you aren't forced into the most expensive tier to get started.

---

## How to get started

### 1. Make sure large volumes are registered

If the subscription hasn't used large volumes before, register that feature first and make sure your regional capacity quota has been increased:

```bash
az account set --subscription <subscriptionId>
az feature register --namespace Microsoft.NetApp --name ANFLargeVolumes

# Can take up to an hour to change from "Registering" to "Registered"
az feature show --namespace Microsoft.NetApp --name ANFLargeVolumes
```

### 2. Request breakthrough mode and check the registration

Submit the breakthrough mode request, which is linked from the [requirements page](https://learn.microsoft.com/en-us/azure/azure-netapp-files/large-volumes-requirements-considerations?tabs=azurecli#register-for-breakthrough-mode). Then check the feature state:

```bash
az feature show --namespace Microsoft.NetApp --name ANFBreakthroughMode
```

Or, if you prefer PowerShell:

```powershell
Get-AzProviderFeature -ProviderNamespace Microsoft.NetApp -FeatureName ANFBreakthroughMode
```

Once it shows **Registered**, a **Breakthrough mode** option appears when you create a large volume in the portal.

---

## Summary - Final thoughts

Breakthrough mode is one of those rare features where the name isn't just bravado. It does break through a ceiling, and it does so with sound engineering: more parallel paths, on hardware nobody else is allowed to touch.

---

### References

- [Requirements and considerations for Azure NetApp Files large volumes](https://learn.microsoft.com/en-us/azure/azure-netapp-files/large-volumes-requirements-considerations?tabs=azurecli#register-for-breakthrough-mode)
- [Understand large volumes in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/large-volumes)
- [Large volume breakthrough mode performance benchmarks for Linux](https://learn.microsoft.com/en-us/azure/azure-netapp-files/performance-large-volume-breakthrough-mode-linux)
- [NetApp: General availability of large volumes breakthrough mode](https://www.netapp.com/product-updates/anf-large-volumes-breakthrough-mode/)

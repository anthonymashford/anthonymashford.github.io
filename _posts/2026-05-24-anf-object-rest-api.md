---
layout*: post
date: 2026-05-24 12:00
title: Stop Photocopying Your Data
subtitle: Getting Insights from Azure NetApp Files with the Object REST API
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, AVD, FSLogix, Zone Redundant]
author: Anthony Mashford
---

## Introduction

Every analytics project seems to begin with the same conversation. Someone points at a slide and asks, "Where does the data live?" Someone else, looking slightly haunted, says, "On the NFS volumes." There is a pause. And then the words that have launched a thousand Data Factory pipelines: *"Right, so we'll need to copy it into a Data Lake first."*

Cue six weeks of pipeline design, an awkward conversation with InfoSec about a second copy of regulated data, and a procurement ticket for additional storage that nobody budgeted for.

This is the well-worn path for getting file-based enterprise data into analytics platforms that prefer object storage. It works. It also produces duplicate data, fragile pipelines, and the persistent feeling that you're paying twice for the privilege of asking your own data a question.

The [Azure NetApp Files object REST API](https://techcommunity.microsoft.com/blog/azurearchitectureblog/unlocking-advanced-data-analytics--ai-with-azure-netapp-files-object-rest-api/4486098) is Microsoft's way of saying: *"Maybe don't do that."*.

## What it actually is (and what it isn't)

The object REST API exposes an existing Azure NetApp Files (ANF) volume through an **S3‑compatible REST interface**. A directory on the volume becomes a bucket; the files beneath become objects whose keys mirror their paths. The data stays exactly where it is. NFS and SMB clients carry on as before. Anything S3‑aware can now reach the same data using a familiar `GET`/`PUT`/`LIST`/`DELETE` vocabulary.

Microsoft calls this *file/object duality*. One copy of the data. Two access patterns. No replication job at 2 a.m.

A few things worth being honest about up front:

- **It is not a full S3 reimplementation.** The operation set is intentionally scoped to what analytics and AI platforms actually use — object listing, read, write, and delete. If a workflow depends on more exotic corners of the S3 API surface, check the documentation before committing.
- **Authentication uses S3‑style access keys, secured with TLS and certificate‑based trust.** Familiar to anyone who has wired up an S3 client; reassuring to anyone who has to sign it off from a security perspective.
- **A bucket is a scoped view onto a directory.** It isn't a separate storage tier, and it doesn't move anything. The bucket is a lens, not a container.
- **The data plane is still ANF.** Performance, snapshots, replication, encryption, the VNet you've already designed around — all unchanged.

In short: ANF remains ANF. The object REST API is simply a second doorway into the same room.

## Why this matters more than it first appears

The cheap framing is *"S3 on ANF — neat."* The interesting framing is what disappears when you don't have to move the data:

- The duplicate copy. And its bill.
- The pipeline that copies it. And its on‑call rotation.
- The lag between data being produced and being analysable.
- The governance gap between the "real" data and the "analytics" data, which inevitably drift apart and start disagreeing about reality.

That last one is the quiet killer. When analytics runs on a copy, the copy becomes a second source of truth. Lineage becomes a polite fiction. Auditors get twitchy. The object REST API removes the copy, and with it the second source of truth.

## A real‑world scenario: insights across a clinical genomics estate

Consider a research‑led pharmaceutical organisation running a translational genomics programme. The setup is familiar to anyone who has worked alongside HPC in life sciences:

- Sequencers and alignment pipelines write **BAM, CRAM, VCF, and FASTQ** files directly to ANF volumes over NFS. Throughput matters; the volumes are doing real work.
- A bioinformatics team uses HPC compute (NFS‑mounted) for primary and secondary analysis.
- A separate data science team wants to run **Azure Databricks** for cohort‑level analytics, feature engineering, and ML model training across the full corpus of variant call data.
- A governance team wants everything visible in **Microsoft Fabric / OneLake** for catalogued search, lineage, and reporting — with no data leaving the controlled environment.

Before the object REST API, this organisation had three unappealing options:

1. **Copy the data into ADLS Gen2.** Doubles the storage footprint of a multi‑petabyte dataset. Requires a copy pipeline that breaks whenever a new study lands. Creates a parallel data estate that drifts.
2. **Bolt NFS into Databricks via init scripts and FUSE.** Possible. Brittle. A support conversation waiting to happen.
3. **Quietly not do the analytics.** Surprisingly popular as a coping strategy.

With the object REST API, the picture changes:

- Sequencer output continues to land on ANF over NFS, exactly as before. The wet‑lab‑facing pipeline is untouched.
- A bucket is exposed over the relevant directories. Databricks reads variant files directly via S3‑compatible APIs, runs Spark jobs across cohorts, and writes derived feature tables back to the same volume.
- OneLake is connected to the ANF bucket using a **shortcut**, so Fabric users can query, catalogue, and govern the same files without copying them into OneLake itself.
- Microsoft Purview now sees *one* dataset, not three. Lineage stops being aspirational.

The analytics team can now answer questions the organisation simply wasn't asking before — not because they were uninteresting, but because the cost and time of getting the data into a queryable shape outweighed the curiosity:

- *Which variants of uncertain significance appear most frequently across the cohort, and how does that distribution shift as new samples arrive?*
- *Which alignment pipeline version produced the variants underpinning this conclusion — and can we re‑run the analysis against a corrected pipeline output without recopying anything?*
- *Across the last 18 months of sequencing runs, where are the systematic quality dips that correlate with reagent lot changes?*

These are insights *about the data on the volume*, generated by tools that don't speak NFS, working directly against files that never moved.

## The architectural pattern

![Architectural Pattren](/assets/img/anf-object-rest-api-architecture.svg)

```text
+----------------------+      NFS / SMB         +-----------------------+
|  HPC, sequencers,    |  <------------------>  |                       |
|  application servers |                        |  Azure NetApp Files   |
+----------------------+                        |  volume               |
                                                |  (single copy)        |
+----------------------+   S3-compatible REST   |                       |
|  Databricks,         |  <------------------>  |  + object REST API    |
|  OneLake shortcuts,  |                        |    bucket abstraction |
|  AI agents, tooling  |                        |                       |
+----------------------+                        +-----------------------+
```

***Same data set. Two connection methods. Simple.***

## Things worth thinking about before switching it on

The object REST API doesn't magically solve every analytics problem, and pretending otherwise is how trust gets lost. A few honest considerations:

- **Object semantics are not file semantics.** Object workflows tend to treat data as immutable blobs. If a downstream job is going to overwrite the same key repeatedly, think about how that interacts with anything still reading the file over NFS.
- **Access patterns drive cost and performance.** ANF is exceptional at high‑throughput, low‑latency file IO. S3‑style access against the same volume inherits those characteristics, but listing a bucket with millions of objects is a very different cost profile from streaming a single large file. Design accordingly.
- **Scope buckets deliberately.** A bucket is a directory exposure. Keep bucket boundaries aligned with security boundaries that you can defend — credentials, after all, are scoped to the bucket context.
- **Governance is your friend.** The fact that every volume *can* be exposed as a bucket does not mean every volume *should* be.

## Summary

There's a longer‑running shift behind this feature, and it's worth naming. For most of the last decade, the architectural pattern for "I want to do analytics on enterprise data" has been *move the data to where the analytics lives*. The object REST API quietly inverts that: ***bring the analytics to where the data lives***.

For organisations whose primary, performance‑sensitive, governed data already sits on ANF, that inversion saves a great deal of effort, money, and accidental drift. It also makes a class of analytics economically viable that previously wasn't.

The insights were always in the data. They just used to require a forklift to get at them.

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new){:target="_blank"} page.

---

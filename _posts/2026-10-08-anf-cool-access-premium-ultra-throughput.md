---
layout*: post
date: 2026-10-08 12:00
title: ANF Storage with cool access enhancement 
subtitle: Maximum Effort, Minimum Penalty - ANF Cool Access Gets a Throughput Upgrade
cover-img: /assets/img/anf-cool-access-premium-ultra-banner.png
thumbnail-img: /assets/img/anf-cool-access-icon.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, W365 Cloud PC, Zone Redundant]
author: Anthony Mashford
---

## Introduction

Cool access on Premium and Ultra no longer charges a flat throughput tax on the whole volume. Throughput is now calculated per volume from how much data actually sits on each tier, so mostly-hot volumes keep almost all of their performance.

*Hey there, and welcome! If you only wanted the quick version, the title pretty much covers it. But if you’ve got a few minutes, grab a coffee (or a chimichanga, no judgement here) and stick around. There are a couple of formulas coming up, but I promise they’re much friendlier than they look.*

For a while, enabling cool access on a Premium or Ultra volume felt like getting a superpower with a catch. Lower storage cost? Lovely. A throughput cut applied to the entire quota, whether you tiered 1 TiB or 9? Less lovely. That catch has now been quietly removed, and it's worth understanding exactly how.

## Cool access in 60 seconds

Cool access moves inactive data blocks (and their snapshots) from the ANF hot tier to an Azure storage account, where they're billed at a lower rate. Microsoft notes that cold data is often more than 50% of total capacity, so this adds up quickly.

1. New blocks land warm on the hot tier.
2. A temperature scan watches activity. Blocks that go untouched for the **coolness period** (2 to 183 days, default 31) turn cold.
3. A tiering scan bundles cold blocks into 4 MB objects and moves them to the cool tier. Allow up to 48 hours for this to kick in.
4. Random reads warm blocks back up to the hot tier. Large sequential reads, such as backup or antivirus scans, don't, unless you change the cool access retrieval policy.

Metadata never leaves the hot tier, so file-count-heavy workloads like EDA, version control and home directories stay snappy. Cool-tier data is billed at the same rate across all service levels, plus hot-to-cool transfer costs.

*Think of it as a little healing factor for your storage bill. The data you’re not using quietly saves you money, and the data you need is right there whenever you call for it. Everybody wins!.*

## The old deal: a flat tax on the whole quota

Under the original model, turning on cool access in a Premium or Ultra pool swapped the per-TiB throughput rate for a lower one, applied to the full quota regardless of how much data was actually cold.

| Service level | Baseline (no cool access) | With cool access (legacy) |
| --- | --- | --- |
| Premium | 64 MiB/s per TiB of quota | 36 MiB/s per TiB of quota |
| Ultra | 128 MiB/s per TiB of quota | 68 MiB/s per TiB of quota |

A 10 TiB Premium volume drops from 640 MiB/s to 360 MiB/s the moment cool access is enabled. Tier 200 GiB or 9 TiB, the answer was the same: a 44% cut. On Ultra, it's closer to 47%.

*It’s like a motorway where one caravan pootling along in the slow lane means every lane gets a 40 mph limit. The caravan’s perfectly happy. Everyone in the fast lane, less so.*

## The new deal: pay for what's actually cold

Throughput is now split by tier. Hot data keeps its full service-level rate, and only the data in the cool tier runs at 16 MiB/s per TiB. The limit is recalculated at regular intervals as data moves between tiers.

### Premium

```text
Max throughput (MiB/s) = (Hot tier TiB × 64) + (Cool tier TiB × 16)
```

### Ultra

```text
Max throughput (MiB/s) = (Hot tier TiB × 128) + (Cool tier TiB × 16)
```

Here's how a 10 TiB volume compares under both models:

| Volume (10 TiB) | Hot / cool split | New limit (MiB/s) | Legacy limit (MiB/s) |
| --- | --- | --- | --- |
| Premium | 8 / 2 TiB | 544 | 360 |
| Premium | 5 / 5 TiB | 400 | 360 |
| Premium | 2 / 8 TiB | 256 | 360 |
| Ultra | 8 / 2 TiB | 1,056 | 680 |
| Ultra | 5 / 5 TiB | 720 | 680 |
| Ultra | 2 / 8 TiB | 384 | 680 |

The 8/2 Premium case is Microsoft's own example: 512 + 32 = 544 MiB/s, a 51% uplift over the legacy 360 MiB/s.

Now the honest bit. The new model isn't a free lunch for every volume. Once roughly 58% of a Premium volume (or 54% of an Ultra volume) is in the cool tier, the new formula produces a *lower* ceiling than the legacy flat rate. That's fair, though: if most of the volume is cold, most of it isn't being read, and you're paying cool-tier prices for it.

*Maximum effort for hot data. Minimum effort for the stuff you haven't touched since, who knows when.*

## The fine print (read it, please!)

- **Auto QoS only.** The split calculation applies to Premium and Ultra volumes in auto QoS pools.
- **100 GiB threshold.** It only kicks in once a volume has more than 100 GiB in the cool tier. Below that, hot and cool limits aren't split.
- **Per volume, not per pool.** Two volumes in the same pool with different amounts of cold data will have different ceilings. That's expected, not a bug.
- **Quota still matters.** Throughput scales linearly with provisioned capacity, so growing the quota raises the limit.
- **Existing pools keep the legacy rate.** Pools where cool access was enabled before the update stay on 36 or 68 MiB/s per TiB. To get the new model, create a new capacity pool and move the volumes into it.
- **Region availability.** The updated throughput model is listed for 48 regions, including UK South, UK West, North Europe, West Europe, East US and East US 2. Check the [region list](https://learn.microsoft.com/en-us/azure/azure-netapp-files/cool-access-introduction#throughput-for-premium-and-ultra-service-levels) before planning.

*Yes, "move your volumes to a new pool" is a step. No, it isn't a multiverse-ending event. Volume moves between pools are online. Simple.*

## Who wins, and what to do next

The biggest winners are performance-sensitive volumes with a modest cold tail: databases with old log or archive files, SAP landscapes, EDA project areas, and VDI profile shares. Before this change, many teams simply wouldn't enable cool access on Premium or Ultra because the throughput hit wasn't worth the saving. That trade-off has largely gone.

1. **Find legacy pools.** List Premium and Ultra pools that had cool access enabled before the update; they're still on the flat rate.
2. **Check the cold ratio.** Use the *Volume cool tier size* metric. Below roughly 55% cold, the new model gives you more headroom.
3. **Move to a new pool.** Create a fresh auto QoS pool and move the volumes across to pick up the new limits.
4. **Re-run the numbers.** Use the [cool access cost estimator](https://aka.ms/anfcoolaccesscalc) and the formulas above to confirm the saving and the ceiling per volume.
5. **Reconsider the "no" list.** Volumes you excluded from cool access for performance reasons deserve another look.

*Fourth-wall break: if your volume is 90% cold and you're worried about throughput, the problem isn't the formula. It's that you're reading a blog instead of archiving that data.*

## Summary - The post-credits scene

Cool access on Premium and Ultra has gone from "save money, lose a chunk of performance" to "save money on what's cold, keep performance on what's hot". It's a small formula change with a big effect on which volumes are worth tiering.

*And if you stayed until the end of the credits, congratulations. There's no teaser. Just go and check your capacity pools.*

## Sources

- [Azure NetApp Files storage with cool access: Throughput for Premium and Ultra service levels](https://learn.microsoft.com/en-us/azure/azure-netapp-files/cool-access-introduction#throughput-for-premium-and-ultra-service-levels) (Microsoft Learn)
- [Azure NetApp Files cool access effective price estimator](https://aka.ms/anfcoolaccesscalc)

---
layout*: post
date: 2026-05-19 12:00
title: Defying Data Gravity - ANF Cache Volumes Are Now GA
subtitle: RBringing the working set to the compute, without moving the data.
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, AVD, FSLogix, Zone Redundant]
author: Anthony Mashford
---

## Azure NetApp Files Cache Volumes: Now Generally Available

Bringing the data closer to where the compute lives has always been one of the harder problems in hybrid storage. You can move the dataset (expensive, slow, and stale before the copy finishes), replicate it (cost, complexity, and another 3am pager you didn't ask for), or simply accept the latency penalty of dragging the same blocks across the WAN over and over like a particularly stubborn dog with a particularly large stick. None of these spark joy.

**Azure NetApp Files cache volumes** — now generally available — give you a fourth option, and a much more dignified one: a persistent, high-performance cache in Azure that sits in front of an ONTAP-based origin volume, serving the hot working set at cloud-local speeds while leaving the authoritative dataset exactly where it is. The data stays put. The bits travel. Everybody wins, including your ExpressRoute bill.

## What a cache volume actually is

A cache volume is a cloud-resident, sparsely populated copy of an external origin volume. It holds only the actively accessed blocks — not the full dataset. The origin remains the source of truth and can be hosted on:

- **On-premises NetApp ONTAP**
- **Cloud Volumes ONTAP (CVO)**
- **Amazon FSx for NetApp ONTAP**

Under the hood, this is ONTAP FlexCache delivered as a first-party Azure service. You get the proven caching behaviour ONTAP customers have leaned on for years, but provisioned, billed, and operated like any other Azure NetApp Files volume. No new console to learn, no new bill to forget about, no new vendor relationship to add to your already-overpopulated calendar.

When a client reads data already in the cache, it's served locally at ANF performance. When a client reads something not yet cached, the cache fetches the block from the origin, stores it, and serves it. Every subsequent read for that block is local. Over time, the working set warms up and the WAN traffic falls off a cliff — which, unlike most things that fall off cliffs, is the desired outcome.

## Reads are interesting. Writes are where it gets clever

Cache volumes support two write modes, and the choice materially changes the workload characteristics:

**Write-around** is the safe option. The cache forwards writes to the origin, waits patiently for the origin to confirm the write is on stable storage, and only then tells the client "yes, all done." Strong consistency. Predictable behaviour. The downside is that every single write pays a round-trip across the network, which adds up in a hurry on a chatty workload.

**Write-back** commits the write to stable storage at the cache, acknowledges the client immediately, and propagates to the origin asynchronously in the background. The result is near-local write performance for clients connected to the cache. This is the unlock for distributed pipelines where writes are frequent and latency matters — and where you can tolerate the origin being a few moments behind the cache in exchange for not making your render farm wait on a transatlantic round trip.

Pick deliberately. Write-back is the right answer for build farms, render pipelines, and anything scratch-heavy. Write-around is the right answer when the origin must reflect every commit before the client moves on — finance ledgers, regulated workloads, anything where "eventually" is not an answer your auditor wants to hear.

## Where this fits

Cache volumes aren't a replacement for cross-region replication, and they aren't a migration tool. They're a distribution mechanism. The scenarios where they earn their place:

- **Burst compute in Azure against an on-prem dataset.** EDA, seismic processing, AI/ML training data, media rendering — any workload where the data lives on-prem but the compute makes sense in the cloud.
- **WAN/ExpressRoute cost reduction.** Once the hot set is cached, you stop paying egress and circuit cost for the same blocks over and over.
- **Globally distributed teams against a central dataset.** Engineering, design, and content teams collaborating on a shared namespace without each office hitting the origin directly.
- **Hybrid migrations in flight.** Stand the cache up in Azure, point your Azure-resident workloads at it, and let the working set warm up naturally before you commit to a full data move.

## Protocol and identity support

Cache volumes support **NFSv3, NFSv4.1, SMB, dual-protocol, and LDAP**-backed configurations — the same protocol surface you'd expect from a regular ANF volume.

One thing worth calling out: SMB cache volumes require a **dedicated Active Directory connection** on the NetApp account. Shared AD configurations aren't supported for SMB caches. If you've spent the last six months consolidating accounts under a shared AD pattern, sorry — plan for that constraint up front rather than discovering it three steps into your deployment.

## What you need before you build one

Cache volumes peer two ONTAP-speaking clusters across an Azure-to-external network path. That means real network connectivity, not just an idea of it:

- **ExpressRoute or site-to-site VPN** between the external ONTAP cluster and the ANF delegated subnet.
- **Bidirectional firewall rules** between all intercluster (IC) LIFs on both sides for:
  - ICMP
  - TCP 11104
  - TCP 11105
  - HTTPS

Get this validated before you start, not after. The single most common failure mode in cache deployments is partial IC LIF reachability — packets flow one way, the peer never establishes, and you spend an afternoon staring at the control plane wondering why ONTAP is sulking when the real answer is sitting in a firewall rule somewhere with someone else's name on it.

## Creating a cache volume — the shape of it

Cache volumes are managed through the Azure NetApp Files REST API (`Microsoft.NetApp/.../caches`). The flow is essentially a four-step state machine:

1. **`PUT` the cache.** You provide the file path, size, protocol, cache subnet, peering subnet, and the origin cluster information — peer cluster name, peer addresses, peer SVM, and peer volume name.

2. **Cluster peer.** Poll the cache with a `GET` until `cacheState = ClusterPeeringOfferSent`. Then call `listPeeringPassphrases` to retrieve the cluster peering command and passphrase. Run that command on the external ONTAP system. **You have 30 minutes** before the offer expires — which sounds generous until you realise it's also the exact window in which someone will inevitably interrupt you with a "quick question." Miss the window and you'll be deleting the cache and starting again.

3. **SVM (vserver) peer.** Wait until `cacheState = VserverPeeringOfferSent`, then run the `vserver peer accept` command on the external ONTAP system. **You have 12 minutes** on this one — so save the coffee run for after.

4. **Cache becomes usable.** When `cacheState` and `provisioningState` both transition to `Succeeded`, the cache is mountable.

If you're standing up multiple caches against the same external cluster, the existing cluster peer (and often the SVM peer) is reused — so subsequent caches skip straight past steps 2 and 3.

A minimal NFSv4.1 cache request body looks like this:

```json
{
  "location": "westus",
  "zones": ["1"],
  "properties": {
    "filePath": "cache1",
    "size": 53687091200,
    "protocolTypes": ["NFSv4"],
    "cacheSubnetResourceId": "/subscriptions/.../subnets/subnet1",
    "peeringSubnetResourceId": "/subscriptions/.../subnets/subnet1",
    "encryptionKeySource": "Microsoft.NetApp",
    "originClusterInformation": {
      "peerClusterName": "origin_cluster",
      "peerAddresses": ["1.2.3.4"],
      "peerVserverName": "origin_svm",
      "peerVolumeName": "origin_volume"
    },
    "exportPolicy": { "rules": [ /* ... */ ] }
  }
}
```

The `peerClusterName` must match the external cluster name exactly — character for character, case included. Most failed deployments at this stage are a typo in that field. If yours isn't working, look there first, then look there again, and then ask a colleague to look at it because by that point you can't see it any more.

## Day-two operations

- **Update** — `PATCH` the cache to change properties, including toggling `writeBack` between `Enabled` and `Disabled`.
- **Delete** — straightforward `DELETE`, with one caveat: if `writeBack` is enabled, you must `PATCH` to disable write-back first, then delete. That ordering forces pending writes to drain to the origin before the cache disappears. It exists because the alternative is data loss, and nobody wants to write that incident report.
- **Modify cluster peer** — if the IC LIF addresses on the origin change, a `POST` to `modifyClusterPeer` updates them. You provide the full peer address list, not a delta. And if you have multiple caches sharing a cluster peer, you only need to call this against one of them.

## Summary

Cache volumes quietly change the unit of hybrid storage architecture. You stop asking "where does the dataset live" and start asking "where does the working set need to be right now." The dataset stays where it makes sense — for governance, gravity, cost, or because moving it requires three signatures and a small ceremony. The working set follows the compute.

For workloads that have been sitting awkwardly between on-prem ONTAP and Azure compute, hoping nobody notices the latency, this is the bridge that's been missing. GA means SLA-backed, production-supported, and ready for the workloads you've quietly been holding back from the cloud.

---

*If you're planning a cache volume deployment and want to talk through the design — origin sizing, write mode selection, peering topology, or anything network-adjacent — drop me a message.*

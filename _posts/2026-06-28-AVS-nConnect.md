---
layout*: post
date: 2026-06-28 12:00
title: Four pipes are better than one
subtitle: NFS nconnect comes to Azure VMware Solution
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, AVD, Azure Vmware Solution, AVS, Zone Redundant]
author: Anthony Mashford
---

## Introduction

There's a particular kind of storage problem that's quietly maddening. The array isn't the bottleneck. The capacity pool has headroom to spare. The service level is generous. And yet a demanding workload still hits a wall — not because storage ran out of anything, but because the *path* to storage ran out of lanes.

For Azure NetApp Files (ANF) datastores on Azure VMware Solution (AVS), that lane has historically been a single TCP connection. One ESXi host, one NFS datastore, one stream. Dependable, predictable, and — when you really lean on it — a polite little chokepoint.

That changes now. **ANF support for the NFS `nconnect` mount option on AVS is generally available.** Each AVS ESXi host can establish up to four parallel TCP connections (`nconnect=4`) to a single ANF NFS datastore. More lanes, same motorway.

![Architectural Pattren](/assets/img/nconnect-diagram.svg)

## The single-connection tax

NFS has long used one TCP connection per datastore per host. That one stream is rock solid, but it caps how much I/O a host can drive to a given datastore, regardless of how much the underlying volume could actually serve.

The mechanism is worth understanding, because it explains why this matters more than "number go up". A single connection can only have so many operations in flight at once. When a busy workload exceeds that, the service applies flow control until in-flight operations complete and resources free up. Each pause is tiny — often microseconds — but multiply microseconds across millions of operations and they stop being a rounding error. They surface as higher, less predictable latency exactly when the workload is at its busiest and you'd least like it to be.

The traditional fix was to spread the load across *more* datastores. It worked, but it was a workaround dressed up as a design. You ended up running and managing more datastores than the workload itself ever needed, purely to buy additional TCP connections. Microsoft's own scaling data tells the story plainly: moving from one datastore to two lifted total throughput by roughly 76% on a single host, and going from one to four delivered around a 163% increase. The gain came almost entirely from spreading load across more network flows — not from any change inside the guest VMs. Storage was never the limiting factor. Connection count was.

## What nconnect actually does

`nconnect` is a client-side NFS capability. Instead of funnelling all datastore traffic down one TCP connection, the client opens several and distributes I/O across them. VMware introduced multiple TCP connections for a single NFSv3 datastore in vSphere 8.0 Update 2, and on-premises NetApp customers adopted it briskly to get more out of their high-speed networks. The same option now lands on AVS with ANF.

With `nconnect=4`, each ESXi host opens four parallel connections to one datastore. Four connections means roughly four times as many operations can be in flight simultaneously, which is the whole game: more concurrency means less queuing, and less queuing means latency stays flat as the workload gets heavy rather than creeping upward under pressure. You get higher aggregate throughput and IOPS *and* more consistent response times from the very same datastore. Higher and steadier are not usually a package deal, so it's worth pausing to appreciate that you get both here.

The headline benefit is what it does to your design. A single datastore with `nconnect=4` can approach the performance that previously took roughly four datastores to achieve, because it removes the single-connection bottleneck rather than working around it. If you've been provisioning datastores in fours like some kind of storage-administrator superstition, you can stop.

## The bit your change-advisory board will appreciate

No changes are required to virtual machines or applications. Once the datastore is mounted with the option, the VMs carry on exactly as before, blissfully unaware that the plumbing underneath them just grew three extra pipes. The improvement lives entirely at the host-to-datastore layer.

That's the rare upgrade that's genuinely boring to roll out, which in production storage is the highest compliment available.

## Where the ceilings really are

Honesty matters more than enthusiasm here, so a few caveats worth stating plainly.

`nconnect=4` raises the per-datastore ceiling; it does not abolish ceilings. Aggregate throughput can still be bounded by network bandwidth, AVS SKU limits, or the service-level ceiling of the ANF volume itself. The NIC in particular is easy to forget: the AV64 SKU ships 100-GbE NICs, while the other SKUs use 25-GbE, and an individual network flow such as an NFS mount can be limited by that 25-GbE interface. Parallelism helps precisely because it spreads work across multiple flows, but the per-flow physics still apply.

And `nconnect` is not mutually exclusive with the multi-datastore approach — it's complementary. You can combine `nconnect=4` with multiple datastores (up to 64 per AVS cluster) when you need to scale further still. The difference is that you now reach for additional datastores because the workload genuinely warrants it, not because a single connection left performance stranded on the table.

It's supported on both AVS Gen 1 and Gen 2 private clouds, so this isn't gated behind your newest environment.

## Quick reference

| | Before (`nconnect=1`) | After (`nconnect=4`) |
|---|---|---|
| TCP connections per datastore, per host | 1 | up to 4 |
| Scaling lever | add more datastores | parallelism within a datastore (then add datastores if needed) |
| Concurrency | limited by single connection | ~4× operations in flight |
| Latency under load | creeps up as queuing builds | stays flatter under concurrency |
| VM / application changes | n/a | none |
| Datastores to hit a target | often several | frequently one |

## Summary

So what should you do with this? If you've been managing a sprawl of datastores whose only job was to manufacture more TCP connections, this is your invitation to simplify. Consolidate where it makes sense, mount with `nconnect=4`, and let a single datastore carry workloads that previously demanded a handful. Databases and other sustained-I/O workloads are the obvious candidates — they're the ones that felt the single-connection tax most acutely.

Fewer datastores to provision, monitor, and reason about; more performance from each one; and no guest-side changes to schedule. For a feature whose entire premise is "open three more connections", that's a remarkably good trade.

---

### Further reading

- [Azure VMware Solution datastore performance considerations for Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/performance-azure-vmware-solution-datastore)
- [Attach Azure NetApp Files datastores to Azure VMware Solution hosts](https://learn.microsoft.com/en-us/azure/azure-vmware/attach-azure-netapp-files-to-azure-vmware-solution-hosts)
- [Azure NetApp Files datastore performance benchmarks for Azure VMware Solution](https://learn.microsoft.com/en-us/azure/azure-netapp-files/performance-benchmarks-azure-vmware-solution)
- [Linux NFS mount options best practices for Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/performance-linux-mount-options)
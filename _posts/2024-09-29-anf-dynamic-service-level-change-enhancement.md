---
layout*: post
date: 2024-09-29 16:00
title: Azure NetApp Files Dynamic Service Level Change Enhancement
subtitle: Shortened Wait Time for Azure NetApp Files
cover-img: /assets/img/24hrs.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files]
author: Anthony Mashford
---

# Dynamic Service Level Change Enhancement: Shortened Wait Time for Azure NetApp Files

In the fast-paced world of cloud computing, agility and cost optimisation are paramount. Businesses need the flexibility to adjust storage performance quickly without incurring unnecessary costs. To meet this demand, **Azure NetApp Files** has introduced a key enhancement: a **shortened wait time for changing to lower service levels**. 

This update reduces the wait time for moving volumes to a lower service level after an upgrade from **seven days** to **24 hours**, giving organisations a powerful tool to manage costs more efficiently while maintaining performance agility.

## Understanding Azure NetApp Files Service Levels

Azure NetApp Files offers three distinct service levels: **Standard**, **Premium**, and **Ultra**. Each tier is tailored to different performance needs:

1. **Standard**: Ideal for workloads with moderate throughput needs.
2. **Premium**: Suitable for more demanding workloads like databases and enterprise applications.
3. **Ultra**: Designed for high-performance workloads that require the lowest latency and highest throughput.

The ability to change service levels dynamically allows organisations to scale up when performance demands spike and scale down when they taper off. This flexibility enables businesses to optimise their cloud infrastructure, providing the necessary performance while minimising expenses.

## What Has Changed?

Previously, once a volume was moved to a higher service level, such as from Standard to Premium or Ultra, the volume had to remain at that level for a minimum of seven days before it could be downgraded to a lower tier. This restriction limited the agility of businesses that needed to temporarily boost performance for short-term needs. 

With the new **24-hour** wait time, users now have a much quicker turnaround to revert to lower service levels, making it easier to manage temporary performance needs without paying for high performance longer than necessary.

### Key Benefits of the Shortened Wait Time

1. **Improved Cost Optimisation**: Organisations can quickly scale back to a lower service level after an intensive period of data processing, reducing the costs associated with running workloads at a higher tier.
  
2. **Agility for Dynamic Workloads**: Many modern workloads experience spikes in demand, such as during month-end financial reporting or peak e-commerce activity. This change ensures businesses can handle such spikes with optimal performance while quickly returning to a cost-effective level when demand subsides.

3. **Reduced Operational Overhead**: With the reduced wait time, the process of manually managing service levels becomes more straightforward. Teams no longer need to plan for long waiting periods when adjusting performance tiers, freeing up time and resources for other operational tasks.

## Practical Use Cases

1. **Data-Intensive Applications**: Companies running large analytics jobs or AI training models often require increased performance for short bursts. With this enhancement, they can scale up to Ultra during processing and then return to Standard or Premium levels once the job is completed.

2. **Seasonal or Peak Periods**: E-commerce platforms or businesses with seasonal spikes in traffic can now scale up their Azure NetApp Files volumes for peak demand and scale down immediately after the rush, avoiding unnecessary costs during off-peak periods.

3. **Testing and Development**: When testing applications or workloads under higher performance tiers, teams can now quickly revert to lower tiers after testing is complete, reducing infrastructure expenses during the development cycle.

## How to Implement the Dynamic Service Level Change

To take advantage of this enhancement, users can easily adjust their service levels through the Azure portal, APIs, or CLI. 

1. **Increase to a Higher Service Level**: Scale your volume up when you anticipate increased performance needs, such as Ultra for high-intensity workloads.
   
2. **Lower Service Level After 24 Hours**: Once the workload subsides, you can adjust the volume back to a lower service tier after the 24-hour wait period, reducing costs.

## Summary

This enhancement is a significant improvement for organisations looking to optimise their cloud infrastructure. By reducing the wait time from seven days to just 24 hours, Azure NetApp Files now allows businesses to be more responsive to changes in workload demand. The ability to quickly return to lower service levels makes it easier to keep cloud storage costs under control while still ensuring performance when it matters most.

If you are using Azure NetApp Files, this change should be a welcome improvement to your cost management and performance tuning strategy, allowing you to benefit from a more dynamic, flexible cloud storage environment. If you are not using Azure NetApp Files,.... why not start a proof of concept to see how you can increase performance and lower costs.😊

For more information, check out this link about [Dynamic Service Level Change Enhancement](https://learn.microsoft.com/en-us/azure/azure-netapp-files/dynamic-change-volume-service-level){:target="_blank"}

Additionally, check out [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new){:target="_blank"} for more updates on the service.
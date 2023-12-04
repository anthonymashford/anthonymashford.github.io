---
layout: post
date: 2023-12-4 10:00
title: Azure storage has never been so cool
subtitle: Storage can be boring, but now it's cool! 
cover-img: /assets/img/anf-cool-arctic.png
thumbnail-img: /assets/img/anf-cool.png
share-img: /assets/img/anf-cool.png
tags: [Blog, Azure, Azure NetApp Files, Storage]
author: Anthony Mashford
---
{:.box-note}
## Festive Tech Calendar 2023

{:.box-note}
This blog article is also part of the 2023 [Festive Tech Calendar](https://festivetechcalendar.com/). This year the Festive Tech Calendar Team are raising money for the [@RaspberryPi_org](https://www.raspberrypi.org/donate/) Foundation. The team believe its important to support charities who do great work. This year they hope to rasie £5000 for this awesome charity! If you would like to donate please visit their [Just Giving Page](https://www.justgiving.com/page/festive-tech-calendar-2023)


## Introduction

Recently Microsoft announced the availability of **Cool-Access tier** for Azure NetApp Files (Public Preview).

Azure NetApp Files is a Microsoft first-party file storage service that provides enterprise-grade functionality to customers. It offers three service levels: Ultra, Premium, and Standard. However, it also offers a cool-access tier (Public Preview) that allows customers to save costs while maintaining the same enterprise-grade functionality for their file storage.

The cool-access feature moves cold (infrequently accessed) data transparently to a cheaper storage tier, reducing the cost of Azure NetApp Files storage. The cool-access feature is enabled at the Capacity Pool level and can configured on a volume by volume basis. Customers can specify the number of days (the coolness period, ranging from 7 to 183 days) for inactive data to be considered "cool". Once data has reached the specified age, it is then tier to the cool-access layer. The meta data still resides in the volume, so the end users still see their data, but the blocks reside at the cool -access level.

## Considerations

- The cool-access feature is currently in **Public Preview**
- During the preview, cool-access is only available in certain regions. Link here to [Regional availability](https://learn.microsoft.com/en-us/azure/azure-netapp-files/cool-access-introduction#supported-regions)
- No guarantee is provided for any maximum latency for client workload for any of the service tiers.
- This feature is available only at the Standard service level. It's not supported for the Ultra or Premium service level.
- Although cool access is available for the Standard service level, how you're billed for using the feature differs from the Standard service level charges.
- You can convert an existing Standard service-level capacity pool into a cool-access capacity pool to create cool access volumes. However, once the capacity pool is enabled for cool access, you can't convert it back to a non-cool-access capacity pool.
- A cool-access capacity pool can contain both volumes with cool access enabled and volumes with cool access disabled.
- After the capacity pool is configured with the option to support cool access volumes, the setting can't be disabled at the capacity pool level. However, you can turn on or turn off the cool access setting at the volume level anytime. Turning off the cool access setting at the volume level stops further tiering of data.
- Standard storage with cool access is supported only on capacity pools of the auto QoS type.
- An auto QoS capacity pool enabled for standard storage with cool access cannot be converted to a capacity pool using manual QoS.
- You can't use large volumes with Standard storage with cool access.
- See Resource limits for Azure NetApp Files for maximum number of volumes supported for cool access per subscription per region.
- If the volume is in a cross-region replication (CRR) relationship as a source volume, you can enable cool access on it only if the mirror state is Mirrored. Enabling cool access on the source volume automatically enables cool access on the destination volume.
- If the volume is in a CRR relationship as a destination volume (data protection volume), enabling cool access on it is not supported.

## Configure Cool-Access

The cool-access tier feature is currently in Public Preview, to enable access to the feature you first need to register the resource provider. To register the feature, run the Azure PowerShell command below.

~~~
Register-AzProviderFeature -ProviderNamespace Microsoft.NetApp -FeatureName ANFCoolAccess
~~~

It can take up to one hour for the feature to be registered. To check the feature registration status, run the Azure PowerShell command below.

~~~
Get-AzProviderFeature -ProviderNamespace Microsoft.NetApp -FeatureName ANFCoolAccess
~~~

## Summary

The Azure NetApp Files cool access tier is an excellent way for customers to save costs while maintaining enterprise-grade functionality for their file storage. By enabling standard volumes with cool access, customers can transparently store data more cost-effectively on Azure storage accounts based on its access pattern, resulting in overall cost savings 

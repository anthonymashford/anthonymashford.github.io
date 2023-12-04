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


## Azure storage has never been so cool

Recently Microsoft announced the availability in Public Preview of Cool-Access tier for Azure NetApp Files.

Azure NetApp Files is a Microsoft first-party file storage service that provides enterprise-grade functionality to customers. It offers three service levels: Ultra, Premium, and Standard. However, it also offers a cool-access tier (Public Preview) that allows customers to save costs while maintaining the same enterprise-grade functionality for their file storage.

The cool-access feature moves cold (infrequently accessed) data transparently to a cheaper storage tier, reducing the cost of storage. Throughput to and from the volume remains the same for the standard storage enabled with cool-access. The cool-access feature is enabled at the Capacity Pool level and can configured on a volume by volume basis. Customers can specify the number of days (the coolness period, ranging from 7 to 183 days) for inactive data to be considered "cool". Once data reached the specified age it is then tier to the cool-access layer. The meata data still resides in the volume, so the end users still see their data, but the blocks reside at the cool -access level.

To take advantage of the cool access tier, customers can enable standard volumes with the ability to transparently store data more cost-effectively on Azure storage accounts based on its access pattern. This option lets you free up storage that resides within Azure NetApp Files volumes by moving data blocks to the lower cost cool tier, resulting in overall cost savings.

In summary, Azure NetApp Files cool access tier is an excellent way for customers to save costs while maintaining enterprise-grade functionality for their file storage. By enabling standard volumes with cool access, customers can transparently store data more cost-effectively on Azure storage accounts based on its access pattern, resulting in overall cost savings 

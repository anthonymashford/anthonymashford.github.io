---
layout: post
date: 2023-12-5 18:00
title: Volume Quotas now GA
subtitle: Azure NetApp Files User & Group Quotas are now GA
cover-img: /assets/img/blogbanner.jpeg
thumbnail-img: /assets/img/anf-quotas.png
share-img: /assets/img/anf-quotas.png
tags: [Blog, Azure, Azure NetApp Files]
author: Anthony Mashford
---

## Azure NetApp Files Volume Quotas Generally Available

Microsoft recently annouced that the user & group quota feature for Azure NetApp Files is now GA.

Using this feature, you can limit the amount of storage capacity that a user or group can consume in an Azure NetApp Files volume by setting user and/or group quotas on volumes. User and group quotas are different from volume quotas in that they further restrict volume capacity consumption at the user and group level.

To set a volume quota, you can use the Azure portal or the Azure NetApp Files API to specify the maximum storage capacity for a volume. Once you set the volume quota, it defines the size of the volume, and there’s no restriction on how much capacity any user can consume.

To restrict users’ capacity consumption, you can set a user and/or group quota. You can set default and/or individual quotas. Once you set user or group quotas, users can’t store more data in the volume than the specified user or group quota limit.

### Default User Quota
The default user quota, once applied to a volume, will limit capacity to all users accessing that volume. For example, if the default quota is set to 50GiB, that is the maximum a single user can consume.

### Individual User Quotas
Individual quotas can be assigned to specific users who have access to the volume. This is done using the UID for UNIX users and the SID for Windows users. Multiple individual quotas can be set per volume and can be smaller or larger than the default quota. Once assigned, that user will only be able to consume the amount of storage as dictated by the individual quota.

### Default & Individual Group Quota
Like user quotas, group quotas can be applied to volumes. Please note that **Group** quotas cannot be applied to SMB or dual-protocol volumes, only NFS volumes. The default quota automatically applies a limit to all users in all groups. Please note that is is possible that a single user might consume the whole group quota assigned to a volume. Individual group quotas apply to all users in a single group. These quotas are assigned using the UNIX for UID.

### Summary

Once a quota has been applied, the end users will only see the capacity that has been allowed as part of their quota. By combining user and group quotas, you can ensure that storage capacity is distributed efficiently and prevent any single user, or group of users, from consuming excessive amounts of storage.
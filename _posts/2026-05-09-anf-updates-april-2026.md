---
layout*: post
date: 2026-05-09 12:00
title: April Updates for ANF
subtitle: Reflecting on Azure NetApp Files Updates from April '26
cover-img: /assets/img/atl-banner.png
thumbnail-img: /assets/img/anf.png
share-img: /assets/img/anf.png
tags: [Blog, Azure, Azure NetApp Files, Terraform, Backup, Replication, AVD, FSLogix, Zone Redundant]
author: Anthony Mashford
---

## Reflecting on Azure NetApp Files Updates from April '26

April brought a useful mix of updates to Azure NetApp Files: two preview features graduating to general availability, two new previews, and a handful of quality-of-life improvements that change defaults rather than introducing entirely new capabilities. Taken together, they continue a multi-quarter pattern of Microsoft and NetApp pulling more enterprise data protection and operational tooling into the core service rather than leaving them as opt-in extras. The shape of the platform a year ago was a high-performance shared storage service with a growing collection of enterprise capabilities bolted around it; the shape now is recognisably a data platform, with security, resilience, and operational tooling presented as first-class concerns rather than configuration toggles. April's release is a clear continuation of that direction.

What follows is a detailed walkthrough of each item, with the kind of architectural and operational context that's useful if you're either running ANF in production today or designing for it in the coming quarter.

## Backup enabled by default (preview)

Backup is now provisioned automatically when you create a new volume, with the option to opt out where it isn't wanted. On the surface this sounds like a minor configuration change, but the operational impact is bigger than the headline suggests, and it's worth thinking through the implications across several dimensions.

### Why the default change matters

The previous default left backup as a deliberate enable. In tightly governed environments with mature provisioning pipelines, that wasn't a problem, because backup configuration was wired into the IaC modules and applied consistently. In every other environment, which is most environments, the default left a gap. Volumes provisioned through portal click-through, ad-hoc Terraform runs, or quick proof-of-concept workflows would routinely end up in production without any backup configured. That gap usually surfaced only when an audit or an incident forced it into view, which is the worst possible time to discover it. Flipping the default reframes the conversation from "did someone remember to enable backup" to "did someone explicitly choose not to", which is a healthier posture for any platform that holds business data.

It's worth being precise about what backup means in the ANF context, because the terminology gets used loosely. ANF backup is the capability that vaults snapshots to a separate backup vault, providing a recovery point that survives volume deletion and lives outside the volume's own snapshot timeline. It complements rather than replaces the snapshot policies that operate on the volume itself. The new default applies to the vaulted backup capability, which means new volumes now come with that secondary protection layer in place from creation, rather than relying on snapshots alone for the first portion of their lifecycle.

### Practical considerations for IaC and automation

For anyone managing ANF through Terraform with the `azurerm` provider, this default change has direct implications. Existing modules that don't explicitly set backup-related properties will now inherit the new default behaviour for new volumes, which can produce drift between what your code says and what the platform actually has. The right response is to make backup configuration explicit in your modules either way, set to whatever your standard requires, so that the behaviour is deterministic regardless of platform defaults. This is a good general practice for anything that has a default value at the platform level and a configurable value at the resource level, since defaults can and do change.

For provisioning automation that uses Bicep, ARM templates, or the Azure CLI, the same principle applies. Don't rely on platform defaults to express intent, because intent expressed in code outlasts platform behaviour and is easier to audit. If your standard is backup-on for production and backup-off for ephemeral workloads, encode that explicitly in the templates rather than relying on the default to do the right thing for each case.

### Cost implications

This changes the cost shape of a new volume. Backup capacity, the data movement to populate the backup vault, and the ongoing incremental updates all now show up by default rather than as a deliberate addition. For dev/test or scratch volumes where the cost-benefit doesn't justify backup, the opt-out path matters, and the financial difference can be meaningful at scale, particularly for organisations running large numbers of short-lived volumes for testing or training purposes. Make sure your provisioning automation handles the opt-out explicitly for those scenarios rather than absorbing the new default by accident.

Cost governance frameworks that depend on tagging or naming conventions to identify volumes that should or shouldn't have backup will need a refresh too, since the previous logic of "explicitly enabled means production" no longer holds. The new logic is "explicitly disabled means ephemeral", which is the inverse of what most existing frameworks were built around.

### Policy and governance

Organisations using policy-driven governance should revisit their Azure Policy definitions. Policies designed to enforce backup on new volumes may now be redundant, since the default already does the work, but policies covering existing volumes remain as relevant as ever. There's also a useful policy pattern available now around opt-out: a policy that flags or denies volumes where backup has been explicitly disabled, which gives you a clean audit trail of where the exception cases live and why.

The feature is available in all ANF-supported regions and is currently in preview, so it's worth validating the behaviour in a non-production subscription before assuming it'll apply uniformly across your estate.

## One Active Directory connection per NetApp account is now the default (GA)

This capability went GA back in May 2025, but as of April it's now the default behaviour and no longer requires feature registration. Each NetApp account can connect to its own Active Directory forest and domain, which removes the previous single-AD-per-region-per-subscription constraint that had been a significant architectural limitation.

### The original constraint and why it mattered

To put the original limitation in context, ANF historically allowed only one Active Directory connection per region per subscription. That meant any organisation operating across multiple AD forests or domains under a shared Azure tenant had to make awkward compromises. Common workarounds included splitting workloads across separate subscriptions purely for AD reasons, deploying into different regions to access different AD configurations, or restructuring AD itself to fit the storage layer, none of which were sensible answers to a storage configuration problem. For service providers and managed hosting platforms, the constraint was particularly painful, since per-tenant AD isolation is a baseline requirement rather than an optional refinement.

Bringing the AD connection down to NetApp account scope means the integration model now matches how most enterprises actually run their identity infrastructure. A NetApp account becomes the natural boundary for an AD integration, which lines up cleanly with how RBAC, billing, and operational ownership tend to be structured. SMB volume creation is now tied to the AD connection at the NetApp account level, which is a cleaner architectural pattern than the previous arrangement where the AD connection floated above the account hierarchy.

### Implications for multi-tenant and hosting scenarios

For organisations operating multi-tenant hosting platforms, this is the model you've probably been waiting for. Each tenant can have its own NetApp account with its own AD connection, providing genuine isolation at the identity layer without subscription gymnastics. The provisioning workflow becomes consistent across tenants regardless of which AD they happen to integrate with, which reduces the cognitive overhead for the operations team and makes onboarding new tenants a routine activity rather than an architectural exercise.

For service providers, this also opens up cleaner patterns around delegated administration. The NetApp account becomes a natural unit of customer-facing configuration, and AD connection management can be delegated to the customer or held centrally depending on the service model. Either way, the boundary sits where it should, which is at the level of the resource that the customer cares about.

### Implications for mergers, acquisitions, and complex AD topologies

Organisations going through mergers and acquisitions tend to inherit AD topologies that don't fit cleanly into a single integration model. The old constraint forced these scenarios into one of the workarounds mentioned above, often with significant operational overhead. The new model accommodates them naturally, since each acquired entity can be onboarded with its own NetApp account and its own AD connection without disturbing the existing setup.

The same applies to organisations with separate forests for production and non-production, security-segregated environments, or geographic separation of identity infrastructure. The previous constraint forced these to share a single AD connection per region, which often meant compromising on the segregation that the AD design was specifically intended to provide. The new default puts the boundary back where the security architecture intended it to be.

### Migration considerations for existing accounts

There is one carve-out worth knowing about, and it matters if you've been on the platform for any length of time. Accounts that were created during the original shared AD preview retain their earlier behaviour and don't automatically pick up the new default. If you have existing NetApp accounts, it's worth auditing them to understand which model each one is operating under, since the operational and security implications differ.

For accounts on the legacy shared model, migration to the per-account model isn't always a simple toggle. SMB volumes have existing AD bindings that need to be considered, and any tooling or automation built around the assumption of shared AD behaviour will need to be reviewed. This is a reasonable area to invest some planning effort in over the next quarter or two, since the strategic direction of the platform is clearly towards the per-account model.

### LDAP and identity integration

The same architectural principle applies to LDAP-based identity integrations, which have benefited from broader directory service support over the past several months, including FreeIPA, OpenLDAP, and Red Hat Directory Server. The combination of per-account AD connections and broader LDAP support means ANF now accommodates a wider range of identity topologies than at any point in its history, which is particularly valuable for Linux-heavy estates or hybrid environments where AD isn't the only identity store in play.

## Advanced ransomware protection (GA)

Advanced ransomware protection has moved out of preview and into general availability. The capability monitors volumes for suspicious activity using a combination of file extension profiling, entropy analysis, and I/OPS pattern detection. When something looks wrong, ANF takes a point-in-time snapshot automatically, raises a notification through the Azure Activity log, and retains the attack report for 30 days.

### How the detection methodology works

The detection methodology is worth understanding in some detail, because it explains both the value and the limits of the feature, and because it's the kind of thing that comes up in security reviews where surface-level descriptions don't satisfy the questions being asked.

File extension profiling tracks the kinds of extensions being written to a volume and flags shifts that look anomalous. This catches the more naive ransomware variants that rename files with characteristic extensions, of which there are well-documented lists across the industry. On its own, this signal would be weak, since attackers can trivially use uncharacteristic extensions or preserve the originals, but as one input among several it adds useful corroboration.

Entropy analysis is more interesting and significantly more robust. Encrypted content has characteristically high entropy compared to typical document, database, or media content, because good encryption produces output that is statistically indistinguishable from random data. That signature is hard for an attacker to disguise without giving up the encryption itself, which means entropy analysis remains effective even against ransomware variants that go to lengths to avoid detection through other channels. The trade-off is that some legitimate content also has high entropy, particularly compressed archives, encrypted database files, and certain media formats, which means entropy alone would generate false positives. The combination with other signals is what makes it work in production.

I/OPS pattern detection looks for the rapid sequential rewrite behaviour that ransomware tends to produce when it works through a directory tree encrypting files in order. This is a different fingerprint from normal application I/O, where reads and writes tend to be more interleaved and less uniformly distributed across files. The pattern signature is most distinctive when ransomware is operating at speed, which is precisely when you most want to catch it.

Combining these three signals gives the system a credible detection layer rather than a marketing label. The automatic snapshot on detection is the right response, since it gives you a clean recovery point before the attack progresses further, and the snapshot itself is taken using the same infrastructure that handles routine snapshot policies, so there's no novel code path involved in the recovery mechanism.

### What this is, and what it isn't

Architectural conversations need to be clear about what this feature does and doesn't address. It's a detection and rapid-recovery capability built on top of ANF's existing snapshot infrastructure, designed to catch encryption events at the storage layer and ensure you have a known-good state to recover from. It's not a replacement for endpoint protection, network segmentation, identity-layer controls, or any of the other components of a defence-in-depth strategy, all of which need to be in place anyway to prevent the attacker from reaching the volume in the first place.

The boundary is important because security reviews sometimes treat storage-layer ransomware detection as a substitute for upstream controls, which it isn't. The value of this feature is that it provides a backstop: even if everything upstream fails and an attacker reaches the volume, you get rapid detection and a clean recovery point, which dramatically reduces the blast radius of the incident. That's a meaningful contribution to a defence-in-depth posture, but it sits at the end of the chain rather than the beginning.

### Operational integration

The integration with the Azure Activity log means detection events flow into the same monitoring fabric as the rest of the Azure estate, which makes operational integration straightforward. Alerts can be wired into existing channels, events can be routed to Sentinel for correlation with other signals from endpoint, network, and identity sources, and the full set of standard Azure monitoring tools applies. For organisations running Microsoft Defender for Storage or Defender for Cloud more broadly, the ANF detection events become another input into the existing security analytics pipeline.

The 30-day attack report retention is useful both for incident response and for compliance reporting, since it gives you a documented timeline of what was detected and when. For regulated industries, that audit trail is worth almost as much as the detection itself, because being able to demonstrate that you detected, responded, and recovered is often what separates an incident from a reportable breach.

### Performance and operational considerations

The detection runs at the storage layer and is designed to operate without imposing meaningful overhead on volume performance, which is the right design choice. False positives are an inherent risk with any heuristic detection system, and worth understanding in the context of how you'd want the platform to respond. A false positive triggers a snapshot, which is a lightweight operation that doesn't affect the volume's data path, and a notification, which can be acted on or dismissed depending on the operational verdict. The cost of a false positive is therefore low, which is the right place for the trade-off to sit.

For workloads that legitimately produce high-entropy content at speed, such as encrypted backup writes from upstream systems or large compressed archive operations, it's worth understanding how the detection responds and tuning expectations accordingly. The retained attack reports are useful for that tuning conversation, since they give you a documented trail of what triggered the system over time.

## User and group quota reporting (GA)

If you're using individual user or group quotas on NFS, SMB, or dual-protocol volumes, you can now generate quota usage reports and modify quota rules directly in the Azure portal. The visibility includes quota limits, used capacity, and percentage utilisation broken down per target user and quota rule.

### The problem this solves

The previous workflow was a genuine operational friction point. Reporting required mounting the volume from a host with the right credentials, running OS-level tools to gather usage data, and then aggregating that information manually if you needed a view across multiple volumes. In practice, this meant storage administrators often didn't have direct access to the information they were responsible for, which created a split between the team operating the platform and the team able to answer questions about it.

That split had practical consequences. Capacity planning conversations would stall waiting for someone with client-side access to run the relevant tooling. Audit queries would take longer than they should have. Investigations into unexpected utilisation patterns would require coordination across teams that didn't always have aligned priorities. None of these were show-stoppers, but they added up to a steady operational tax that made quota-managed volumes more painful to run than they should have been.

It also made the reporting cadence harder to automate. The natural place for capacity data to surface is in Azure Monitor, Azure Resource Graph, or one of the dashboarding tools that build on those, and host-based tooling didn't fit cleanly into any of them. Aggregating quota data into the broader monitoring picture required custom plumbing, which most organisations either didn't build or built and then struggled to maintain.

### What changes

Bringing this into the portal turns quota management into a first-class operation. Storage administrators can now see and modify quota rules without needing client-side access, generate point-in-time reports for capacity planning conversations, and build a consistent picture of utilisation across volumes. For automation use cases, the portal capability typically implies an underlying control-plane API surface, which opens the door to scripting and dashboarding via Azure Resource Graph, the Azure CLI, PowerShell, or direct REST API calls.

The Azure Resource Graph angle is particularly worth highlighting. ARG queries can aggregate data across subscriptions and resource groups in a way that's difficult to replicate with host-based tooling, which means quota reporting can now be consolidated into the same dashboards that handle the rest of the platform. For organisations running ANF at scale, this is a meaningful improvement in operational visibility.

### Use cases worth considering

The use cases for quotas in ANF are broader than people sometimes assume. Research computing environments often allocate capacity to principal investigators or research groups and need to enforce those allocations to prevent runaway consumption from any single team. EDA environments running large numbers of short-lived simulation jobs benefit from group-level quotas as a mechanism to prevent any single team from consuming the entire shared volume. General-purpose file shares supporting multiple departments or business units use quotas as a soft cost allocation mechanism, where the quota itself doesn't enforce hard limits but provides the basis for chargeback or showback.

Healthcare and life sciences environments with shared analysis volumes use quotas to manage capacity across multiple research projects under a single platform. Financial services environments use them for compliance-driven separation between trading desks or product lines. Higher education environments use them to manage student and faculty allocations. In all of these cases, the previous reporting friction made quotas more painful to operate than they should have been, and this update materially reduces that overhead.

### Dual-protocol considerations

For dual-protocol volumes, quota reporting needs to handle the UID/GID mapping that connects NFS identities to SMB identities, which is the kind of operational detail that's easier to manage from a portal that understands the underlying configuration than from host-based tooling that has to reason about it from outside. The portal-native reporting handles this consistently, which removes another category of friction for environments that bridge Linux and Windows access patterns.

## Secure object REST API access via Azure Key Vault (preview)

The object REST API, the S3-compatible interface introduced for ANF in October 2025, now supports Azure Key Vault for both certificates and access credentials. Certificates can be generated and stored directly in Key Vault and retrieved automatically during bucket creation, and access credentials are managed centrally rather than uploaded manually for each configuration change.

### Why credential management matters

This is a sensible move that brings object access in line with how the rest of the Azure platform expects sensitive material to be handled. Manual certificate uploads and credential management are precisely the kind of operations that end up producing security debt: certificates pasted into runbooks, credentials stored in shared password managers, secrets embedded in scripts that nobody wants to touch in case something breaks. Centralising in Key Vault means certificate rotation can be automated, RBAC-controlled access governs who can retrieve what, audit logging captures every access event, and the secret material itself stops circulating in places it shouldn't be.

For organisations operating under regulatory frameworks that mandate centralised key management, this also resolves a compliance gap. Manual credential handling for object access wasn't necessarily defensible under frameworks that require all sensitive material to live within an approved key management system, and the workaround of treating object credentials as exempt was awkward at best. The native Key Vault integration removes the awkwardness.

### Certificate lifecycle

The certificate lifecycle aspect is particularly worth highlighting. Object access at any meaningful scale generates a non-trivial number of certificates over time, and managing those manually is the kind of work that drifts out of sync with reality. Certificates expire, are forgotten about, get reissued without proper revocation, end up in environments they shouldn't be in. With Key Vault integration, certificate generation, storage, and retrieval all happen in a single managed service, which means rotation can become a scheduled operation rather than a project. Key Vault's existing capabilities around access policies, RBAC, soft delete, and purge protection all apply, which means the certificate material gets the same protection as anything else stored in the vault.

For organisations with strict cryptographic key management requirements, this also brings object access under the same Key Vault policies that govern the rest of the estate, including HSM-backed key storage where that's required. The combination of ANF's customer-managed keys with managed HSM (which went GA in May 2025) and Key Vault integration for the object API gives you a consistent cryptographic posture across the data-at-rest and data-access layers.

### The bigger picture: ANF as a data platform

The broader context matters too, and it's worth pausing on. The object REST API was introduced to bridge ANF data into modern cloud-native consumption patterns. Microsoft Fabric is the most prominent integration target, since Fabric expects S3-compatible access for OneLake shortcut creation, but the same API surface applies to AI Foundry tooling, third-party analytics platforms, and any other workload where S3 semantics are the access pattern of choice.

This matters for the strategic direction of ANF. The traditional ANF use case was high-performance file storage for enterprise workloads: SAP, Oracle, EDA, HPC, VDI. The object API extends ANF into the modern cloud-native consumption pattern without requiring data movement, which is increasingly important as AI/ML pipelines need to consume the same data that operational workloads are generating. For organisations building out AI capabilities on top of existing operational data, the ability to expose that data through an S3-compatible interface without copying or replicating it is genuinely valuable.

As object API usage grows, the credential boundary becomes a more frequently exercised control point. Getting it onto Key Vault now, while the integration patterns are still being established, is the right time to do it. Retrofitting credential management onto an existing object API estate is significantly more painful than building it in from the start.

### Network and access considerations

Object API access has the same network considerations as the underlying ANF service: private endpoints, network security group rules, and virtual network integration all apply. The Key Vault used for credential storage should sit within the same network architecture, which means private endpoint deployment for Key Vault is the natural pattern. For organisations running fully private Azure deployments with no public network exposure, this is the configuration to plan for.

The feature is in preview, so production use should be approached with appropriate caution, but it's worth designing for now if you're standing up new object-API workloads.

## Cool access throughput enhancement for Premium and Ultra (preview)

This one is worth reading carefully if you run Premium or Ultra capacity pools with cool access enabled, because it changes the performance model in a way that materially affects capacity planning and the conversations you have with application teams.

### The previous behaviour and why it created friction

The previous behaviour applied a fixed throughput reduction whenever cool access was enabled on a volume, regardless of how much data was actually tiered. That made the cost-performance trade-off easy to model in a spreadsheet but harder to justify operationally, particularly for workloads where the cool tier ratio fluctuates over time. It also created an awkward conversation with application owners: enabling cool access for cost optimisation reasons came with a performance penalty that didn't necessarily reflect their actual access patterns, and explaining why a volume with 95% hot data was being treated as if it were entirely cool was difficult.

For workloads with predominantly hot data and a long tail of cool reference content, the fixed reduction overstated the impact, and that often led to teams declining to enable cool access even when the economics would have favoured it. The result was a pattern where cool access was used for archival workloads where the performance hit didn't matter and avoided for mixed workloads where it would have mattered, which left value on the table on both sides.

### The new behaviour

The new behaviour calculates maximum throughput dynamically based on the proportion of data tiered to cool storage. Hot data retains its configured performance, and throughput is only adjusted in proportion to what actually sits in the cool tier. The mental model shifts from "enabling cool access costs you X% throughput" to "the cool portion of your volume performs at cool-tier characteristics, the hot portion performs at its full configured rate". That's both a more honest reflection of how the underlying tiering actually works and a more useful basis for capacity planning conversations.

The dynamic calculation also means you don't need to tune anything as access patterns shift over time, which removes a class of operational work that nobody enjoyed doing anyway. The platform tracks the hot/cool ratio and adjusts the throughput envelope automatically, which is the right place for that logic to live.

### Capacity planning implications

For workloads with mixed access patterns, this is a meaningful improvement. Reference datasets paired with active working sets, archives sitting alongside live content, large media libraries with seasonal access patterns, and analytics environments where data ages predictably all benefit from a model that aligns performance with actual data placement rather than a worst-case assumption.

Capacity planning models that currently assume the fixed reduction will need to be revised. The throughput envelope for a given volume now depends on the hot/cool ratio rather than a static derate, which means the "what throughput do I have available" question has a more interesting answer than it used to. For workloads with predictable tiering patterns, this is straightforward to model. For workloads with variable patterns, the modelling gets more complex but more useful, since it reflects the reality of how the volume will perform under different access conditions.

### Monitoring implications

Monitoring should account for the fact that throughput limits are now a moving target tied to data placement, which has implications for how you set thresholds on the relevant metrics. The "throughput limit reached" metric becomes more nuanced, since the limit itself is dynamic. The volume consumed throughput and percentage volume consumed throughput metrics need to be interpreted against the current dynamic limit rather than a static configured limit, which means dashboards that hard-code the configured throughput as a reference line will give misleading results.

The right pattern is probably to monitor the ratio of consumed throughput to currently-applicable limit, which gives you a meaningful signal regardless of how the underlying tiering changes. For organisations running large numbers of cool-access-enabled volumes, this is worth thinking about before the new behaviour is rolled out broadly, since dashboard updates take time and the migration window is a useful opportunity to make those updates.

### Service level scope

The change applies to Premium and Ultra service levels specifically. Standard and Flexible behaviour is unchanged, which is worth being explicit about because it affects how you reason about the platform overall. The Flexible service level has its own approach to the capacity-throughput relationship, where the two are decoupled by design and priced separately, which is a different model from the dynamic adjustment in Premium and Ultra. Both approaches are valid for different use cases, and the platform now offers a meaningful choice between them.

The feature is in preview, so as always, validate the behaviour against your specific workload before rolling out broadly. For production workloads currently running on Premium or Ultra with cool access, it's worth modelling the expected change in throughput envelope before the dynamic behaviour applies, so that any application-side performance considerations are surfaced ahead of time rather than discovered after.

## Summary

The April release continues a clear theme in recent ANF updates: making secure, resilient defaults the path of least resistance, and graduating data protection capabilities from preview into the mainstream. The shape of the platform a year ago was a high-performance shared storage service with enterprise capabilities arranged around it; the shape now is a recognisable data platform, with security, resilience, and operational tooling presented as first-class concerns rather than configuration toggles.

Backup-by-default and ransomware protection going GA are the headline items for anyone responsible for production data, and both reflect a broader industry shift towards security being baked into the platform rather than bolted on afterwards. For organisations building cloud governance frameworks, these are exactly the kind of platform-level controls that make the governance conversation simpler, since the conversation moves from "how do we ensure this is enabled" to "how do we ensure the exceptions are documented".

The cool access throughput change and Key Vault integration for the object API are the ones to watch if you're optimising cost or building modern data platform workloads. The cool access change makes the cost-performance trade-off honest in a way it previously wasn't, which removes a barrier to broader adoption of tiering across mixed workloads. The Key Vault integration aligns object API credential management with the rest of the Azure platform, which is the right position for it to be in as object API usage grows.

The AD and quota reporting changes are the operational improvements that don't make the marketing pages but matter every day for the people running the platform. Both close gaps that have been open longer than they probably should have been, and both reflect a maturing of the platform towards the kind of operational ergonomics that enterprise customers expect as standard.

If you're planning ANF deployments in the coming quarter, the new defaults are worth factoring into provisioning patterns, IaC modules, governance policies, monitoring dashboards, and platform documentation. Several of these features change what "out of the box" means for ANF, and that's worth reflecting in how teams are onboarded and how the platform is described internally. The direction of travel is clear: ANF continues to consolidate enterprise data platform capabilities into the core service, and the gap between what you get by default and what you'd want from a hand-tuned deployment continues to close. For practitioners, that means less time configuring and more time delivering value on top of the platform, which is the right direction for any service to be heading.

For more information on the Azure NetApp Files service, check out the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new) {:target="_blank"} page.
